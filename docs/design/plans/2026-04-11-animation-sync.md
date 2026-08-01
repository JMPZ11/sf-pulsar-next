# Animation Sync Implementation Plan


**Goal:** Make the remote companion (Vasco) play correct walk/run/idle animations driven by the other player's movement, eliminating the "ghost gliding" effect.

**Architecture:** Two-layer approach: (1) Hook Actor::SetPosition via vtable patching to block the engine from moving remote Vasco on its own, eliminating jitter. (2) Read animation variables (Speed, Direction, iSyncSprintState etc.) from the sending player, piggyback them onto position packets, and write them into Vasco's behavior graph on the receiving side. The hook ensures animations drive visuals only, not position.

**Tech Stack:** CommonLibSF (RE::Actor vtable index 0x137 for SetPosition), existing ENet networking, BSAnimationGraph vtable for SetGraphVariableFloat/Int/Bool (already probed in CompanionManager.cpp).

**Known Starfield Animation Variables (from extracted .agx game data):**
- Float: `Speed`, `Direction`, `DirectionSmoothed`, `LookHeadingOffset`, `CurrentSpeed`, `fControllerXRaw`
- Int: `iSyncIdleLocomotion`, `iSyncSprintState`, `iSyncRelaxReadySneak`, `iSyncGravity`, `iSyncSwimState`, `iSyncSighted`
- Bool: `IsCompletelyStopped`, `bPlayerMoveStartActive`, `bNoFootIK`

---

### Task 1: SetPosition VTable Hook — Block Engine Movement on Remote Companion

**Why this is first:** Without this hook, the engine fights our position updates every frame causing jitter. Animation variables made this WORSE in previous attempts because the engine used them to move the actor. This hook is the prerequisite for everything else.

**Files:**
- Modify: `include/Companion/CompanionManager.h`
- Modify: `src/Companion/CompanionManager.cpp`

- [ ] **Step 1: Add hook infrastructure to CompanionManager.h**

Add a static flag and the original function pointer storage:

```cpp
// In CompanionManager class, add to private section after m_wasMoving:

// SetPosition hook — blocks engine from moving remote companion
static inline bool s_allowSetPosition = false;
static inline void* s_originalSetPosition = nullptr;
static void __fastcall HookedSetPosition(RE::Actor* a_this, const RE::NiPoint3& a_pos, bool a_updateCharController);

void InstallSetPositionHook();
void RemoveSetPositionHook();
```

- [ ] **Step 2: Implement the hook in CompanionManager.cpp**

Add the hook function and install/remove methods. The hook checks: is this our companion? If yes, only allow if `s_allowSetPosition` is true (set by our code). If it's any other actor, always pass through.

```cpp
void __fastcall CompanionManager::HookedSetPosition(RE::Actor* a_this, const RE::NiPoint3& a_pos, bool a_updateCharController)
{
    // Always allow non-companion actors through
    auto& mgr = GetCompanionManager();
    if (a_this != mgr.m_companion || s_allowSetPosition) {
        // Call original
        using OrigFn = void(__fastcall*)(RE::Actor*, const RE::NiPoint3&, bool);
        reinterpret_cast<OrigFn>(s_originalSetPosition)(a_this, a_pos, a_updateCharController);
    }
    // else: blocked — engine tried to move our remote companion
}

void CompanionManager::InstallSetPositionHook()
{
    if (!m_companion || s_originalSetPosition) return;

    // Get vtable pointer from companion instance
    auto** vtable = *reinterpret_cast<void***>(m_companion);

    // Save original function at index 0x137 (Actor::SetPosition)
    s_originalSetPosition = vtable[0x137];

    // Overwrite vtable entry with our hook
    DWORD oldProtect;
    VirtualProtect(&vtable[0x137], sizeof(void*), PAGE_READWRITE, &oldProtect);
    vtable[0x137] = reinterpret_cast<void*>(&HookedSetPosition);
    VirtualProtect(&vtable[0x137], sizeof(void*), oldProtect, &oldProtect);

    REX::INFO("[PULSAR] SetPosition hook installed on companion vtable[0x137]"sv);
}

void CompanionManager::RemoveSetPositionHook()
{
    if (!m_companion || !s_originalSetPosition) return;

    auto** vtable = *reinterpret_cast<void***>(m_companion);

    DWORD oldProtect;
    VirtualProtect(&vtable[0x137], sizeof(void*), PAGE_READWRITE, &oldProtect);
    vtable[0x137] = s_originalSetPosition;
    VirtualProtect(&vtable[0x137], sizeof(void*), oldProtect, &oldProtect);

    s_originalSetPosition = nullptr;
    REX::INFO("[PULSAR] SetPosition hook removed"sv);
}
```

- [ ] **Step 3: Wire up the hook in DoDisableAI and DoEnableAI**

In `DoDisableAI()`, after `m_aiDisabled = true;` add:

```cpp
InstallSetPositionHook();
```

In `DoEnableAI()`, before `m_companion = nullptr;` add:

```cpp
RemoveSetPositionHook();
```

- [ ] **Step 4: Use the flag in DoSetPosition**

Wrap the existing SetPosition call so the hook lets it through:

```cpp
void CompanionManager::DoSetPosition()
{
    s_allowSetPosition = true;

    RE::NiPoint3A pos(m_targetPosData.x, m_targetPosData.y, m_targetPosData.z);
    m_companion->SetPosition(pos, true);

    s_allowSetPosition = false;

    // Rotation
    m_companion->data.angle.x = m_targetPosData.rotX;
    m_companion->data.angle.y = m_targetPosData.rotY;
    m_companion->data.angle.z = m_targetPosData.rotZ;
}
```

- [ ] **Step 5: Build and test**

Run: `cmake --build build/vs --config Release`
Expected: Clean build, no errors.

Deploy DLL to both Host and Client. Test:
1. Connect client to host
2. Vasco should follow position updates with **no jitter**
3. If Vasco still jitters, the vtable index 0x137 may be wrong — check the log for "SetPosition hook installed" message

- [ ] **Step 6: Commit**

```bash
git add include/Companion/CompanionManager.h src/Companion/CompanionManager.cpp
git commit -m "feat: hook Actor::SetPosition to block engine movement on remote companion"
```

---

### Task 2: Animation Variable Helpers — SetGraphVarInt and SetGraphVarBool

**Why:** We already have `SetGraphVarFloat` (probes vtable to find the function). We need `Int` and `Bool` variants for state variables like `iSyncSprintState` (int) and `IsCompletelyStopped` (bool). The existing `GetGraphVariableImplInt` and `GetGraphVariableImplBool` are at known vtable indices (0x13, 0x14) on IAnimationGraphManagerHolder.

**Files:**
- Modify: `src/Companion/CompanionManager.cpp`

- [ ] **Step 1: Add SetGraphVarInt helper**

Add after the existing `SetGraphVarFloat` function (around line 232):

```cpp
static bool SetGraphVarInt(RE::Actor* a_actor, const char* a_name, int32_t a_val)
{
    if (!a_actor) return false;

    RE::BSFixedString varName(a_name);
    int32_t testVal = 0;
    if (!a_actor->GetGraphVariableImplInt(varName, testVal)) return false;

    RE::BSTSmartPointer<RE::BSAnimationGraphManager> graphMgr;
    if (!a_actor->GetAnimationGraphManagerImpl(graphMgr) || !graphMgr) return false;
    if (graphMgr->graphs.size() == 0) return false;

    auto& graph = graphMgr->graphs[graphMgr->activeGraph];
    if (!graph) return false;

    auto** vtable = *reinterpret_cast<void***>(graph.get());

    // SetVariableInt is typically at the same relative offset as SetVariableFloat
    // but for int type. Probe nearby indices.
    static int s_foundIndex = -1;
    if (s_foundIndex >= 0) {
        using SetVarFn = bool(*)(void*, const void*, int32_t);
        reinterpret_cast<SetVarFn>(vtable[s_foundIndex])(graph.get(), &varName, a_val);
        return true;
    }

    // Probe: set a unique value, read back, verify
    for (int idx = 0x19; idx <= 0x2C; idx++) {
        __try {
            using SetVarFn = bool(*)(void*, const void*, int32_t);
            reinterpret_cast<SetVarFn>(vtable[idx])(graph.get(), &varName, 99999);
        } __except(1) { continue; }

        int32_t check = 0;
        a_actor->GetGraphVariableImplInt(varName, check);
        if (check == 99999) {
            s_foundIndex = idx;
            REX::INFO("[PULSAR] SetGraphVarInt FOUND at vtable[0x{:02X}]!"sv, idx);
            // Set the actual value
            using SetVarFn = bool(*)(void*, const void*, int32_t);
            reinterpret_cast<SetVarFn>(vtable[idx])(graph.get(), &varName, a_val);
            return true;
        }
    }

    return false;
}
```

- [ ] **Step 2: Add SetGraphVarBool helper**

```cpp
static bool SetGraphVarBool(RE::Actor* a_actor, const char* a_name, bool a_val)
{
    if (!a_actor) return false;

    RE::BSFixedString varName(a_name);
    bool testVal = false;
    if (!a_actor->GetGraphVariableImplBool(varName, testVal)) return false;

    RE::BSTSmartPointer<RE::BSAnimationGraphManager> graphMgr;
    if (!a_actor->GetAnimationGraphManagerImpl(graphMgr) || !graphMgr) return false;
    if (graphMgr->graphs.size() == 0) return false;

    auto& graph = graphMgr->graphs[graphMgr->activeGraph];
    if (!graph) return false;

    auto** vtable = *reinterpret_cast<void***>(graph.get());

    static int s_foundIndex = -1;
    if (s_foundIndex >= 0) {
        using SetVarFn = bool(*)(void*, const void*, bool);
        reinterpret_cast<SetVarFn>(vtable[s_foundIndex])(graph.get(), &varName, a_val);
        return true;
    }

    for (int idx = 0x19; idx <= 0x2C; idx++) {
        __try {
            using SetVarFn = bool(*)(void*, const void*, bool);
            reinterpret_cast<SetVarFn>(vtable[idx])(graph.get(), &varName, true);
        } __except(1) { continue; }

        bool check = false;
        a_actor->GetGraphVariableImplBool(varName, check);
        if (check == true) {
            s_foundIndex = idx;
            REX::INFO("[PULSAR] SetGraphVarBool FOUND at vtable[0x{:02X}]!"sv, idx);
            using SetVarFn = bool(*)(void*, const void*, bool);
            reinterpret_cast<SetVarFn>(vtable[idx])(graph.get(), &varName, a_val);
            return true;
        }
    }

    return false;
}
```

- [ ] **Step 3: Build**

Run: `cmake --build build/vs --config Release`
Expected: Clean build.

- [ ] **Step 4: Commit**

```bash
git add src/Companion/CompanionManager.cpp
git commit -m "feat: add SetGraphVarInt and SetGraphVarBool helpers for animation sync"
```

---

### Task 3: Extend Position Packet with Animation Data

**Why:** Instead of a separate animation packet, we piggyback animation variables onto the existing position packet. This keeps them in sync and avoids extra network overhead. We add the key locomotion variables: Speed (float), Direction (float), and state flags (int/bool packed into a single uint16).

**Files:**
- Modify: `include/Network/PacketTypes.h`

- [ ] **Step 1: Replace PositionData with extended version**

Replace the existing `PositionData` struct:

```cpp
// PKT_POSITION payload — position + animation variables piggybacked
struct PositionData
{
    // Position
    float x;
    float y;
    float z;
    float rotX;
    float rotY;
    float rotZ;

    // Animation variables (piggybacked for sync)
    float speed;          // "Speed" behavior graph variable
    float direction;      // "Direction" behavior graph variable
    uint16_t animFlags;   // Packed state flags:
                          //   bits 0-1: iSyncIdleLocomotion (0=idle, 1=locomotion)
                          //   bits 2-3: iSyncSprintState (0=no sprint, 1=sprinting)
                          //   bits 4-6: iSyncRelaxReadySneak (0=relaxed,1=ready,2=sneak,3=alert)
                          //   bit 7:    IsCompletelyStopped
                          //   bit 8:    bPlayerMoveStartActive
};
```

No other files need changes yet — the struct is just bigger, existing Write/Read calls will pick up the new fields automatically since they use `sizeof(T)`.

- [ ] **Step 2: Build**

Run: `cmake --build build/vs --config Release`
Expected: Clean build. The new fields are uninitialized (zero) until Task 4 fills them.

- [ ] **Step 3: Commit**

```bash
git add include/Network/PacketTypes.h
git commit -m "feat: extend PositionData with animation variables for locomotion sync"
```

---

### Task 4: Read Animation Variables from Player and Send

**Why:** On the sending side, we read the player's current animation state and pack it into the position packet. Both host and client do this.

**Files:**
- Modify: `src/Network/NetworkHost.cpp` (host sends in Update)
- Modify: `src/Main.cpp` (client sends in ClientSendPosition)

- [ ] **Step 1: Add helper to read and pack animation variables from player**

Add this free function at the top of `NetworkHost.cpp`, after the includes:

```cpp
static void FillAnimationData(RE::Actor* a_actor, PositionData& a_pos)
{
    if (!a_actor) return;

    RE::BSFixedString speedVar("Speed");
    RE::BSFixedString dirVar("Direction");
    RE::BSFixedString idleLocoVar("iSyncIdleLocomotion");
    RE::BSFixedString sprintVar("iSyncSprintState");
    RE::BSFixedString relaxVar("iSyncRelaxReadySneak");
    RE::BSFixedString stoppedVar("IsCompletelyStopped");
    RE::BSFixedString moveStartVar("bPlayerMoveStartActive");

    a_actor->GetGraphVariableImplFloat(speedVar, a_pos.speed);
    a_actor->GetGraphVariableImplFloat(dirVar, a_pos.direction);

    int32_t idleLoco = 0, sprint = 0, relax = 0;
    bool stopped = false, moveStart = false;

    a_actor->GetGraphVariableImplInt(idleLocoVar, idleLoco);
    a_actor->GetGraphVariableImplInt(sprintVar, sprint);
    a_actor->GetGraphVariableImplInt(relaxVar, relax);
    a_actor->GetGraphVariableImplBool(stoppedVar, stopped);
    a_actor->GetGraphVariableImplBool(moveStartVar, moveStart);

    a_pos.animFlags =
        (static_cast<uint16_t>(idleLoco & 0x3))       |
        (static_cast<uint16_t>(sprint & 0x3)   << 2)   |
        (static_cast<uint16_t>(relax & 0x7)    << 4)   |
        (static_cast<uint16_t>(stopped ? 1 : 0) << 7)  |
        (static_cast<uint16_t>(moveStart ? 1 : 0) << 8);
}
```

- [ ] **Step 2: Call FillAnimationData in host's Update position send**

In `NetworkHost::Update()`, find the block where the host sends position (around line 98-103). Change:

```cpp
PositionData pos{ loc.x, loc.y, loc.z, ang.x, ang.y, ang.z };
```

to:

```cpp
PositionData pos{};
pos.x = loc.x; pos.y = loc.y; pos.z = loc.z;
pos.rotX = ang.x; pos.rotY = ang.y; pos.rotZ = ang.z;
FillAnimationData(player, pos);
```

- [ ] **Step 3: Add same helper to Main.cpp and call in ClientSendPosition**

Add the same `FillAnimationData` function (or move it to a shared header if preferred — for now, duplicate is fine since it's small).

Add includes at top of Main.cpp:

```cpp
#include <RE/B/BSFixedString.h>
#include <RE/I/IAnimationGraphManagerHolder.h>
```

Add the `FillAnimationData` function (same code as Step 1) before `ClientSendPosition`.

In `ClientSendPosition()`, change:

```cpp
Pulsar::Network::PositionData posData{};
posData.x = pos.x;
posData.y = pos.y;
posData.z = pos.z;
posData.rotX = ang.x;
posData.rotY = ang.y;
posData.rotZ = ang.z;
```

to:

```cpp
Pulsar::Network::PositionData posData{};
posData.x = pos.x;
posData.y = pos.y;
posData.z = pos.z;
posData.rotX = ang.x;
posData.rotY = ang.y;
posData.rotZ = ang.z;
FillAnimationData(player, posData);
```

- [ ] **Step 4: Build**

Run: `cmake --build build/vs --config Release`
Expected: Clean build.

- [ ] **Step 5: Commit**

```bash
git add src/Network/NetworkHost.cpp src/Main.cpp
git commit -m "feat: read player animation variables and pack into position packets"
```

---

### Task 5: Apply Animation Variables to Remote Companion

**Why:** On the receiving side, after updating the companion's position, we also apply the animation variables from the packet. Combined with the SetPosition hook from Task 1, the engine will play the correct animation without moving the actor.

**Files:**
- Modify: `include/Companion/CompanionManager.h`
- Modify: `src/Companion/CompanionManager.cpp`

- [ ] **Step 1: Extend CompanionPositionData to carry animation data**

In `CompanionManager.h`, modify `CompanionPositionData`:

```cpp
struct CompanionPositionData
{
    float x, y, z;
    float rotX, rotY, rotZ;

    // Animation (filled from network packet)
    float    speed = 0.0f;
    float    direction = 0.0f;
    uint16_t animFlags = 0;
};
```

- [ ] **Step 2: Update RequestSetPosition to accept animation data**

Change the method signature in both `.h` and `.cpp`:

Header:
```cpp
void RequestSetPosition(float x, float y, float z, float rotX, float rotY, float rotZ,
                        float speed = 0.0f, float direction = 0.0f, uint16_t animFlags = 0);
```

Implementation:
```cpp
void CompanionManager::RequestSetPosition(float x, float y, float z, float rotX, float rotY, float rotZ,
                                          float speed, float direction, uint16_t animFlags)
{
    std::lock_guard lock(m_posMutex);
    m_pendingPosData = { x, y, z, rotX, rotY, rotZ, speed, direction, animFlags };
    m_pendingPosition = true;
}
```

- [ ] **Step 3: Apply animation variables in DoSetPosition**

Add animation application after setting position and rotation:

```cpp
void CompanionManager::DoSetPosition()
{
    s_allowSetPosition = true;

    RE::NiPoint3A pos(m_targetPosData.x, m_targetPosData.y, m_targetPosData.z);
    m_companion->SetPosition(pos, true);

    s_allowSetPosition = false;

    // Rotation
    m_companion->data.angle.x = m_targetPosData.rotX;
    m_companion->data.angle.y = m_targetPosData.rotY;
    m_companion->data.angle.z = m_targetPosData.rotZ;

    // Animation variables
    SetGraphVarFloat(m_companion, "Speed", m_targetPosData.speed);
    SetGraphVarFloat(m_companion, "Direction", m_targetPosData.direction);

    // Unpack state flags
    int32_t idleLoco   = m_targetPosData.animFlags & 0x3;
    int32_t sprint     = (m_targetPosData.animFlags >> 2) & 0x3;
    int32_t relax      = (m_targetPosData.animFlags >> 4) & 0x7;
    bool    stopped    = (m_targetPosData.animFlags >> 7) & 0x1;
    bool    moveStart  = (m_targetPosData.animFlags >> 8) & 0x1;

    SetGraphVarInt(m_companion, "iSyncIdleLocomotion", idleLoco);
    SetGraphVarInt(m_companion, "iSyncSprintState", sprint);
    SetGraphVarInt(m_companion, "iSyncRelaxReadySneak", relax);
    SetGraphVarBool(m_companion, "IsCompletelyStopped", stopped);
    SetGraphVarBool(m_companion, "bPlayerMoveStartActive", moveStart);
}
```

- [ ] **Step 4: Update callers in NetworkHost.cpp to pass animation data**

In `NetworkHost::OnReceive`, PKT_POSITION handler, change the `RequestSetPosition` call:

```cpp
mgr.RequestSetPosition(pos.x, pos.y, pos.z, pos.rotX, pos.rotY, pos.rotZ,
                        pos.speed, pos.direction, pos.animFlags);
```

- [ ] **Step 5: Build**

Run: `cmake --build build/vs --config Release`
Expected: Clean build.

- [ ] **Step 6: Commit**

```bash
git add include/Companion/CompanionManager.h src/Companion/CompanionManager.cpp src/Network/NetworkHost.cpp
git commit -m "feat: apply animation variables to remote companion from position packets"
```

---

### Task 6: Integration Test & Debug Logging

**Why:** This is a complex feature touching vtable hooks, animation graphs, and networking. We need targeted logging to diagnose issues during live testing on two PCs.

**Files:**
- Modify: `src/Companion/CompanionManager.cpp`

- [ ] **Step 1: Add debug logging for animation variable application**

Add rate-limited debug logging in `DoSetPosition` (every 120 frames = ~2 seconds):

```cpp
// At the end of DoSetPosition, after all SetGraphVar calls:
static int s_debugCounter = 0;
if (++s_debugCounter >= 120) {
    s_debugCounter = 0;
    REX::INFO("[PULSAR] AnimSync: Speed={:.1f} Dir={:.1f} IdleLoco={} Sprint={} Relax={} Stopped={} MoveStart={}"sv,
        m_targetPosData.speed, m_targetPosData.direction,
        idleLoco, sprint, relax, stopped ? 1 : 0, moveStart ? 1 : 0);
}
```

- [ ] **Step 2: Add debug logging for hook activity**

Add a rate-limited counter in HookedSetPosition to confirm the hook is working:

```cpp
void __fastcall CompanionManager::HookedSetPosition(RE::Actor* a_this, const RE::NiPoint3& a_pos, bool a_updateCharController)
{
    auto& mgr = GetCompanionManager();
    if (a_this != mgr.m_companion || s_allowSetPosition) {
        using OrigFn = void(__fastcall*)(RE::Actor*, const RE::NiPoint3&, bool);
        reinterpret_cast<OrigFn>(s_originalSetPosition)(a_this, a_pos, a_updateCharController);
    } else {
        // Blocked — log periodically
        static int s_blockCount = 0;
        if (++s_blockCount >= 300) {
            REX::INFO("[PULSAR] SetPosition hook blocked {} engine calls"sv, s_blockCount);
            s_blockCount = 0;
        }
    }
}
```

- [ ] **Step 3: Build and deploy**

Run: `cmake --build build/vs --config Release`

Deploy to:
- Host: Copy DLL to Starfield SFSE plugins folder
- NAS: Copy DLL to `\\<NAS>\PULSAR\`
- Client (Laptop): User copies manually from NAS

- [ ] **Step 4: Live test on two PCs**

Test checklist:
1. Host starts, Client connects — both see Vasco
2. **Jitter test**: Walk around — Vasco should NOT jitter (hook blocks engine)
3. **Animation test**: Walk — Vasco should play walk animation
4. **Sprint test**: Sprint — Vasco should play sprint animation
5. **Stop test**: Stand still — Vasco should play idle animation
6. **Check logs**: Verify "AnimSync" log shows changing Speed/Direction values
7. **Check logs**: Verify "SetPosition hook blocked" messages appear (confirms hook works)

- [ ] **Step 5: Commit**

```bash
git add src/Companion/CompanionManager.cpp
git commit -m "feat: add debug logging for animation sync and SetPosition hook"
```

---

### Task 7: Cleanup — Remove Unused Code

**Why:** CompanionManager.cpp still has unused functions from earlier experiments: `SetCharControllerVelocity`, `TryCallVtable` (replaced by inline __try), the `hkVector4` struct. Clean up after confirming animation sync works.

**Files:**
- Modify: `src/Companion/CompanionManager.cpp`
- Modify: `include/Companion/CompanionManager.h`

- [ ] **Step 1: Remove unused code from CompanionManager.cpp**

Remove:
- `struct hkVector4` (lines 27-30)
- `SetCharControllerVelocity` function (lines 37-48)
- `TryCallVtable` function (lines 179-189) — replaced by inline `__try` in each SetGraphVar helper
- Unused includes: `RE/N/NiAVObject.h`, `RE/N/NiTransform.h` (if not used elsewhere)
- Remove CharController velocity zeroing in `DoEnableAI` (lines 140-144) since we no longer use velocity-driven movement

- [ ] **Step 2: Remove unused fields from CompanionManager.h**

Remove:
- `m_wasMoving` (line 47) — no longer used

- [ ] **Step 3: Build**

Run: `cmake --build build/vs --config Release`
Expected: Clean build, no warnings about removed code.

- [ ] **Step 4: Commit**

```bash
git add include/Companion/CompanionManager.h src/Companion/CompanionManager.cpp
git commit -m "refactor: remove unused velocity and vtable probe code from CompanionManager"
```
