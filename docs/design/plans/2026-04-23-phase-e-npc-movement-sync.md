# Phase E — NPC Movement-Controller Animation-Sync Implementation Plan


**Goal:** NPCs on client move fluidly (walk/run/sprint animations play natively, no teleport-look) driven by host-sampled velocity/direction. Position-snapping only for drift correction.

**Architecture:** Host samples `Speed` / `Direction` graph-vars + sprint-state from each tracked NPC and sends them alongside position at 30 Hz in extended `PKT_NPC_STATE` entries. Client applies via the same primitives used for the player-clone: `ForceSetGraphFloat`, `ForceSetGraphInt`, Havok `setLinearVelocity` via CharController vtable[79]. `SetPosition` fires only when positional drift exceeds 2 m. AI stays suppressed (existing `SetRestrained` + Aggression=0).

**Tech stack:** C++ (SFSE plugin, CommonLibSF), ENet transport, hand-rolled SEH wrappers for Havok writes, MinHook for engine entry-points.

**RE foundation:** T41 (memory `reference_npc_movement_agents.md`) confirmed no single native `SetDesiredVelocity(actor, v)` primitive exists — Starfield uses class-based MovementAgent + Handler + Planner, not callable externally. Phase E therefore reuses existing player-clone primitives.

---

## Task 0: Probe NPC CharController layout

**Why:** Before writing Havok velocity to NPCs we must confirm they have the same `middleHigh+0x458 = bhkCharRigidBodyController` layout as Player. Some NPCs (ships, creatures) may use `hknpBody` only.

**Files:**
- Modify: `src/NPCSync/NPCSync.cpp:~873` (inside `CaptureFromActor`)

- [ ] **Step 1: Add one-shot CharController probe in `CaptureFromActor`**

Insert right after the `actorState2` read (line ~892) — one-shot per unique base, capped at 5 logs:

```cpp
// Phase E probe: log CharController layout for first 5 unique base IDs.
static std::unordered_set<uint32_t> s_probedBases;
static std::atomic<int> s_probedCount{0};
if (baseFormID != 0 && s_probedCount.load() < 5) {
    bool insert = false;
    {
        std::lock_guard lk(g_trackedMutex);
        insert = s_probedBases.insert(baseFormID).second;
    }
    if (insert) {
        __try {
            auto u = reinterpret_cast<uintptr_t>(a_actor);
            uintptr_t aiProc = *reinterpret_cast<uintptr_t*>(u + 0x228);
            uintptr_t mh = aiProc ? *reinterpret_cast<uintptr_t*>(aiProc + 0x08) : 0;
            void* cc = mh ? *reinterpret_cast<void**>(mh + 0x458) : nullptr;
            void** vt = cc ? *reinterpret_cast<void***>(cc) : nullptr;
            uintptr_t bw = cc ? *reinterpret_cast<uintptr_t*>(
                reinterpret_cast<uintptr_t>(cc) + 0x20) : 0;
            REX::INFO("[PULSAR] E-probe base=0x{:08X} actor=0x{:08X} cc={} vt={} body={}"sv,
                baseFormID, a_actor->formID,
                (void*)cc, (void*)vt, (void*)bw);
            s_probedCount.fetch_add(1);
        } __except (EXCEPTION_EXECUTE_HANDLER) {
            REX::WARN("[PULSAR] E-probe SEH for base=0x{:08X}"sv, baseFormID);
        }
    }
}
```

- [ ] **Step 2: Build + deploy**

```bash
cmake --build build --config Release && cp build/Release/PULSAR.dll "<NAS>\\deploy\\Data\\SFSE\\Plugins\\" && cp build/Release/PULSAR.dll "<Starfield>/Data/SFSE/Plugins/"
```

- [ ] **Step 3: Run host + client, check logs**

Start both sides, host walks past 5 different NPC types (Sarah, random proc-gen enemy, etc.). Verify log line appears for each:
- Expected: `cc=0x... vt=0x... body=0x...` — all three non-null on at least the common humanoid NPCs.
- Expected: `vt` matches known `bhkCharRigidBodyController::vftable` VA (RVA 0x4D6B430 per VehicleSync memory — verify with `disasm_at.py`).

- [ ] **Step 4: Commit probe**

```bash
git add src/NPCSync/NPCSync.cpp
git commit -m "debug: Phase E probe CharController layout on tracked NPCs"
```

If probe shows CharController is absent / different vtable on certain NPC types, note which and gate locomotion writes in Task 7 accordingly.

---

## Task 1: Extend wire format — add speed / direction / moveFlags to NPCStateData

**Files:**
- Modify: `include/Network/PacketTypes.h:219-236`

- [ ] **Step 1: Add fields to `NPCStateData`**

Insert between `rotZ` (line ~228) and `health` (line ~229):

```cpp
float    speed;         // Graph "Speed" — mirrors player-clone sync
float    direction;     // Graph "Direction" — body-relative heading
```

After `flags` add:

```cpp
uint8_t  moveFlags;     // bit 0 = isMoving, bit 1 = isSprinting
                        // (bit 2+ reserved; weaponDrawn stays in `flags`)
```

- [ ] **Step 2: Update size comment + static_assert**

Change the size comment (line ~232) from:

```cpp
// = 4+4+4+4 + 64 + 12+4+4+4 + 1 = 105 bytes
```

to:

```cpp
// = 16 (IDs) + 64 (plugin) + 12 (pos) + 4 (rotZ) + 8 (speed+dir) + 8 (health×2) + 2 (flags+moveFlags) = 114 bytes
```

Change the static_assert (line ~236):

```cpp
static_assert(sizeof(NPCStateData) == 114, "NPCStateData size");
```

- [ ] **Step 3: Build to catch any struct-layout fallout**

```bash
cmake --build build --config Release 2>&1 | tail -40
```

Expected: clean build. If any other file reads NPCStateData with a literal size (`== 105` anywhere), fix it.

- [ ] **Step 4: Grep for stale size references**

```bash
grep -rnE "sizeof\(NPCStateData\)|== 105" include/ src/
```

Expected: only the updated lines. Any `105` literal in NPC context → bump to 114.

- [ ] **Step 5: Commit**

```bash
git add include/Network/PacketTypes.h
git commit -m "feat: Phase E — extend NPCStateData with speed/direction/moveFlags"
```

---

## Task 2: Extend host-side TrackedNPC + client-side PendingUpdate mirrors

**Files:**
- Modify: `src/NPCSync/NPCSync.cpp` — two structs at ~line 100-120 (TrackedNPC) and ~line 1135-1149 (PendingUpdate)

- [ ] **Step 1: Locate TrackedNPC struct**

```bash
grep -n "struct TrackedNPC" src/NPCSync/NPCSync.cpp
```

- [ ] **Step 2: Add fields to TrackedNPC**

After the existing `rotZ` field add:

```cpp
float    speed = 0.0f;
float    direction = 0.0f;
uint8_t  moveFlags = 0;
```

- [ ] **Step 3: Add same fields to PendingUpdate (line ~1149)**

After `rotZ` in PendingUpdate add:

```cpp
float    speed;
float    direction;
uint8_t  moveFlags;
```

- [ ] **Step 4: Build to verify struct compiles everywhere**

```bash
cmake --build build --config Release 2>&1 | tail -30
```

- [ ] **Step 5: Commit**

```bash
git add src/NPCSync/NPCSync.cpp
git commit -m "feat: Phase E — mirror speed/direction/moveFlags in TrackedNPC + PendingUpdate"
```

---

## Task 3: Host — sample Speed / Direction / Sprint per tracked NPC

**Why:** Reuse the same graph-var read path that `Main.cpp::FillAnimationData` uses for the real Player — works on any Actor.

**Files:**
- Modify: `src/NPCSync/NPCSync.cpp:~892-998` (inside `CaptureFromActor`)
- Reference-only: `src/Main.cpp:~145-190` (how the player samples vars)

- [ ] **Step 1: Locate existing graph-var read primitive**

```bash
grep -n "GetGraphVariableImplFloat\|GetGraphVariableFloat" src/Main.cpp src/RemotePlayer/RemotePlayerManager.cpp
```

Find the exact function Main.cpp calls to read `"Speed"` from the Player's anim graph — it's either a free function or a vtable call via an Actor pointer.

- [ ] **Step 2: Add reads in CaptureFromActor**

After the `weaponDrawn` line (~892) add:

```cpp
// Phase E: sample graph vars for locomotion sync
float speed = 0.0f, direction = 0.0f;
int32_t sprintState = 0;
{
    // Reuse Main.cpp's graph-var-read primitive. Walk actor's anim graph
    // manager the same way the Player does — same API works on any Actor.
    RE::BSFixedString sName("Speed"), dName("Direction"), spName("iSyncSprintState");
    if (auto* amgr = a_actor->GetAnimationGraphManager()) {  // adjust to actual API
        amgr->GetGraphVariableFloat(sName, speed);
        amgr->GetGraphVariableFloat(dName, direction);
        amgr->GetGraphVariableInt(spName, sprintState);
    }
}
uint8_t moveFlags = 0;
if (speed > 5.0f) moveFlags |= 0x1;
if (sprintState != 0) moveFlags |= 0x2;
```

(If `GetAnimationGraphManager` is not the exact API, grep `src/Main.cpp` for the actual read path and mirror it.)

- [ ] **Step 3: Store into TrackedNPC**

Extend the assignment block at ~line 994:

```cpp
entry.posX = posX; entry.posY = posY; entry.posZ = posZ;
entry.rotZ = rotZ;
entry.speed = speed;           // NEW
entry.direction = direction;   // NEW
entry.moveFlags = moveFlags;   // NEW
entry.health = health; entry.healthMax = healthMax;
```

- [ ] **Step 4: Build + deploy + sanity-check log**

Add a temporary diag log 1×/3s:

```cpp
if (doDiag) {
    REX::INFO("[PULSAR] E-host NPC 0x{:08X} speed={:.1f} dir={:.2f} mf=0x{:02X}"sv,
        a_actor->formID, speed, direction, moveFlags);
}
```

Deploy, connect. Expected: values update as NPC walks (speed > 0 when moving), zero when idle.

- [ ] **Step 5: Remove diag, commit**

```bash
git add src/NPCSync/NPCSync.cpp
git commit -m "feat: Phase E — host samples Speed/Direction/Sprint from tracked NPCs"
```

---

## Task 4: Host — pack new fields into `PKT_NPC_STATE` batch

**Files:**
- Modify: `src/NPCSync/NPCSync.cpp:~1098-1120` (batch build loop in `OnGameThreadTick`)

- [ ] **Step 1: Extend the entry-copy block**

Find the loop that copies `TrackedNPC` fields into each `NPCStateData&` entry (between lines ~1100 and ~1130). After the existing `rotZ` and `flags` assignments add:

```cpp
entries[i].speed = e.speed;
entries[i].direction = e.direction;
entries[i].moveFlags = e.moveFlags;
```

- [ ] **Step 2: Build**

```bash
cmake --build build --config Release 2>&1 | tail -20
```

- [ ] **Step 3: Commit**

```bash
git add src/NPCSync/NPCSync.cpp
git commit -m "feat: Phase E — pack speed/direction/moveFlags into PKT_NPC_STATE"
```

---

## Task 5: Client — unpack new fields in `OnPacketReceived`

**Files:**
- Modify: `src/NPCSync/NPCSync.cpp:1184-1200` (the `PendingUpdate` build loop)

- [ ] **Step 1: Extend the unpack loop**

Between `u.rotZ = e.rotZ;` (line 1194) and `u.health = e.health;` (line 1195) insert:

```cpp
u.speed = e.speed;
u.direction = e.direction;
u.moveFlags = e.moveFlags;
```

- [ ] **Step 2: Build**

```bash
cmake --build build --config Release 2>&1 | tail -20
```

- [ ] **Step 3: Commit**

```bash
git add src/NPCSync/NPCSync.cpp
git commit -m "feat: Phase E — client unpacks speed/direction/moveFlags from packet"
```

---

## Task 6: Client — extract shared `ApplyLocomotion` helper

**Why:** RemotePlayerManager already has the exact primitive we need (graph-var writes + CharController velocity + lerp rotation). Extract it so NPCSync can call the same code without re-implementing the SEH wrappers.

**Files:**
- Create: `include/NPCSync/LocomotionApply.h`
- Create: `src/NPCSync/LocomotionApply.cpp`
- Reference: `src/RemotePlayer/RemotePlayerManager.cpp` for the Havok velocity + graph-var primitives

- [ ] **Step 1: Create header**

```cpp
#pragma once
#include <cstdint>
#include <RE/N/NiPoint3.h>

namespace RE { class Actor; }

namespace Pulsar::NPCSync {

    struct LocomotionInput {
        RE::NiPoint3 targetPos;
        float targetRotZ;
        float speed;
        float direction;
        uint8_t moveFlags;   // bit 0 = moving, bit 1 = sprinting
        bool   weaponDrawn;
    };

    // Apply one frame of host-driven locomotion to a non-owned Actor.
    // Writes graph vars (Speed, Direction, iSyncIdleLocomotion, iSyncSprintState)
    // Writes Havok setLinearVelocity if CharController is attached
    // Lerps rotZ toward target
    // Snaps SetPosition only if |actor.pos - target| > drift threshold
    // Returns true if at least one write landed.
    bool ApplyLocomotion(RE::Actor* actor, const LocomotionInput& in);

}  // namespace Pulsar::NPCSync
```

- [ ] **Step 2: Create implementation**

```cpp
#include "PCH.h"
#include "NPCSync/LocomotionApply.h"
#include "NASLog.h"
#include <RE/A/Actor.h>
#include <Windows.h>
#include <cmath>

namespace Pulsar::NPCSync {

    // Drift threshold: above this, SetPosition snap. Below, let Havok integrate velocity.
    static constexpr float kDriftThresholdSq = 2.0f * 2.0f;

    // Port of SafeHavokBodySetLinearVelocityImpl from RemotePlayerManager — kept
    // local so this module has no link-time dependency on RemotePlayer.
    static void SafeHavokBodySetLinearVelocity(RE::Actor* actor, float vx, float vy, float vz)
    {
        __try {
            auto u = reinterpret_cast<uintptr_t>(actor);
            uintptr_t aiProc = *reinterpret_cast<uintptr_t*>(u + 0x228);
            if (!aiProc) return;
            uintptr_t mh = *reinterpret_cast<uintptr_t*>(aiProc + 0x08);
            if (!mh) return;
            void* cc = *reinterpret_cast<void**>(mh + 0x458);
            if (!cc) return;
            uintptr_t bw = *reinterpret_cast<uintptr_t*>(
                reinterpret_cast<uintptr_t>(cc) + 0x20);
            if (!bw) return;
            uintptr_t inner = *reinterpret_cast<uintptr_t*>(bw + 0x20);
            if (!inner) return;
            auto** vt = *reinterpret_cast<void***>(cc);
            if (!vt) return;

            alignas(16) float vel4[4] = { vx, vy, vz, 0.0f };
            using SetVelFn = void(__fastcall*)(void*, float*);
            reinterpret_cast<SetVelFn>(vt[0x278 / 8])(cc, vel4);
        } __except (EXCEPTION_EXECUTE_HANDLER) {}
    }

    // Write graph-var via actor's AnimationGraphManager. Must match the primitive
    // Main.cpp uses to READ these vars — if they're readable, the setter works too.
    static void SetGraphFloat(RE::Actor* actor, const char* name, float val)
    {
        __try {
            RE::BSFixedString v(name);
            if (auto* amgr = actor->GetAnimationGraphManager()) {
                amgr->SetGraphVariableFloat(v, val);
            }
        } __except (EXCEPTION_EXECUTE_HANDLER) {}
    }

    static void SetGraphInt(RE::Actor* actor, const char* name, int32_t val)
    {
        __try {
            RE::BSFixedString v(name);
            if (auto* amgr = actor->GetAnimationGraphManager()) {
                amgr->SetGraphVariableInt(v, val);
            }
        } __except (EXCEPTION_EXECUTE_HANDLER) {}
    }

    static bool SafeSetPosition(RE::Actor* actor, const RE::NiPoint3& target, float rotZ)
    {
        __try {
            RE::NiPoint3A tgt(target.x, target.y, target.z);
            actor->SetPosition(tgt, true);
            actor->data.angle.z = rotZ;
            return true;
        } __except (EXCEPTION_EXECUTE_HANDLER) {
            return false;
        }
    }

    bool ApplyLocomotion(RE::Actor* actor, const LocomotionInput& in)
    {
        if (!actor) return false;

        // 1. Graph vars — drives Walk/Run/Sprint blend on the NPC's own anim graph
        SetGraphFloat(actor, "Speed", in.speed);
        SetGraphFloat(actor, "Direction", in.direction);
        SetGraphInt(actor, "iSyncIdleLocomotion", (in.moveFlags & 0x1) ? 1 : 0);
        SetGraphInt(actor, "iSyncSprintState", (in.moveFlags & 0x2) ? 1 : 0);

        // 2. Havok velocity — world-space vel from speed + heading
        if (in.speed > 1.0f) {
            const float headingRad = actor->data.angle.z + in.direction;
            const float vx = std::cos(headingRad) * in.speed;
            const float vy = std::sin(headingRad) * in.speed;
            SafeHavokBodySetLinearVelocity(actor, vx, vy, 0.0f);
        } else {
            SafeHavokBodySetLinearVelocity(actor, 0.0f, 0.0f, 0.0f);
        }

        // 3. Drift correction — SetPosition only if fallen behind
        const float dx = actor->data.location.x - in.targetPos.x;
        const float dy = actor->data.location.y - in.targetPos.y;
        const float dz = actor->data.location.z - in.targetPos.z;
        const float driftSq = dx * dx + dy * dy + dz * dz;
        if (driftSq > kDriftThresholdSq) {
            SafeSetPosition(actor, in.targetPos, in.targetRotZ);
        } else {
            // Lerp rotZ only — lightweight, no position write
            const float cur = actor->data.angle.z;
            const float delta = in.targetRotZ - cur;
            actor->data.angle.z = cur + delta * 0.2f;
        }

        return true;
    }

}
```

- [ ] **Step 3: Add to CMakeLists**

```bash
grep -n "NPCSync" CMakeLists.txt
```

Add `src/NPCSync/LocomotionApply.cpp` to the source list where the other NPCSync sources are.

- [ ] **Step 4: Build**

```bash
cmake --build build --config Release 2>&1 | tail -30
```

Expected: clean build. If `GetAnimationGraphManager` / `SetGraphVariableFloat` aren't the exact APIs, fix to match the actual CommonLibSF declarations — cross-check against how `ForceSetGraphFloatImpl` works in `src/RemotePlayer/RemotePlayerManager.cpp`. May need to copy that implementation verbatim (direct vtable call) rather than going through CommonLib's cpp wrapper.

- [ ] **Step 5: Commit**

```bash
git add include/NPCSync/LocomotionApply.h src/NPCSync/LocomotionApply.cpp CMakeLists.txt
git commit -m "feat: Phase E — shared LocomotionApply helper (graph + velocity + drift-snap)"
```

---

## Task 7: Client — replace the Phase D v14 throttled-SetPosition block

**Files:**
- Modify: `src/NPCSync/NPCSync.cpp:1643-1685` (the Phase D v14 "Throttled SetPosition" block)

- [ ] **Step 1: Add include**

At the top of `src/NPCSync/NPCSync.cpp` add:

```cpp
#include "NPCSync/LocomotionApply.h"
```

- [ ] **Step 2: Replace lines 1643-1685**

The current block computes `shouldSend` from time + position-delta and calls `SafeNpcSetPosition`. Replace with:

```cpp
// Phase E: host-velocity-driven locomotion. Calls SetPosition only when
// positional drift exceeds the threshold inside ApplyLocomotion.
Pulsar::NPCSync::LocomotionInput li{};
li.targetPos    = RE::NiPoint3(u.posX, u.posY, u.posZ);
li.targetRotZ   = u.rotZ;
li.speed        = u.speed;
li.direction    = u.direction;
li.moveFlags    = u.moveFlags;
li.weaponDrawn  = u.weaponDrawn;
Pulsar::NPCSync::ApplyLocomotion(actor, li);
```

Keep the existing `has3DNow` guard above (lines 1661-1672) — ApplyLocomotion still requires loaded 3D to be meaningful.

- [ ] **Step 3: Keep SYNC-DIAG diagnostic intact**

The `doDiag` block (1687-1709) stays — it's the rate-limited drift measurement. Useful to know whether our new path actually tracks.

- [ ] **Step 4: Build**

```bash
cmake --build build --config Release 2>&1 | tail -30
```

- [ ] **Step 5: Commit**

```bash
git add src/NPCSync/NPCSync.cpp
git commit -m "feat: Phase E — apply host-velocity-driven locomotion on client NPCs"
```

---

## Task 8: Smoke test on one walking NPC

**Files:** none — runtime test.

- [ ] **Step 1: Deploy**

```bash
cp build/Release/PULSAR.dll "<NAS>\\deploy\\Data\\SFSE\\Plugins\\" && cp build/Release/PULSAR.dll "<Starfield>/Data/SFSE/Plugins/"
```

- [ ] **Step 2: Run**

Start host, let client connect. Host walks to a POI with visible NPCs (e.g. a spaceport with civilians). Watch one NPC the host can see walking.

- [ ] **Step 3: Observe on client**

Expected:
- NPC walks smoothly (anim graph plays Walk cycle, legs move).
- No teleport jumps.
- Direction-changes are smooth (small heading lerps visible).
- When NPC starts running on host → client NPC transitions Walk→Run via iSyncSprintState.

- [ ] **Step 4: Check logs for drift-snap frequency**

```bash
grep -c "SafeSetPosition\|SYNC-DIAG" "<Starfield>/Data/SFSE/Plugins/PULSAR.log"
```

Expected: SafeSetPosition fires rarely (drift >2m is uncommon for walking NPCs). If firing every frame → velocity integration broken, revisit Task 6 Step 2.

- [ ] **Step 5: Commit if stable, or revert block and diagnose**

If visibly working:

```bash
git tag phase-e-smoke-ok
```

If broken:
- NPC stuck in T-pose → graph-var writes didn't take. Check `GetAnimationGraphManager` → real vtable path.
- NPC rubber-bands → velocity integration overshoots; lower 0.2 lerp or cap `speed`.
- Client crash → see Task 9.

---

## Task 9: Handle edge cases (crash triage)

**Files:** varies depending on observed failure.

- [ ] **Step 1: If `Phase E probe` (Task 0) showed CharController absent on some NPCs**

Gate the velocity write. Edit `LocomotionApply.cpp::ApplyLocomotion`:

```cpp
// Skip velocity write when CC not attached; fall through to drift-snap.
// SafeHavokBodySetLinearVelocity already SEH-guards but saves a null-chain.
```

No code change needed — the SEH wrapper already bails on null CC. Just verify with a warning-once log.

- [ ] **Step 2: If graph-var writes crash**

Switch the `SetGraphFloat` helper to the direct-vtable pattern used by RemotePlayerManager:

```cpp
// Reference: RemotePlayerManager.cpp::ForceSetGraphFloatImpl
// Uses BSAnimationGraphManager vtable slot XX directly via SEH — avoids
// CommonLibSF's cpp wrapper which may have incorrect struct offsets.
```

Copy the ForceSetGraphIntDirect / ForceSetGraphFloatDirect implementations from RemotePlayerManager.cpp verbatim.

- [ ] **Step 3: If `iSyncSprintState` doesn't transition Walk→Run**

Sprint graph may depend on additional vars — look at `Main.cpp` for what else the Player sends (`iSyncIdleLocomotion`, Speed>300 threshold, etc.). Add whichever ints transition the NPC graph into its Run state.

- [ ] **Step 4: Commit whatever fix lands**

```bash
git commit -am "fix: Phase E edge case — <symptom>"
```

---

## Task 10: Tune + document

**Files:**
- Modify: `src/NPCSync/LocomotionApply.cpp` (thresholds)
- Modify: `docs/PULSAR_Technical.md` (Phase E section)

- [ ] **Step 1: Tune drift threshold based on smoke-test logs**

If drift-snap fires too often (more than 1× per 2s per NPC) → raise to `3.0 * 3.0`. If NPCs visibly drift >2m → lower to `1.0 * 1.0`.

- [ ] **Step 2: Tune rotation lerp**

Current: 0.2 per tick (~0.2 × 30 Hz = 6× convergence/sec). If NPCs rotate jerkily → drop to 0.1. If too slow and NPCs aim wrong way → raise to 0.3.

- [ ] **Step 3: Document in `docs/PULSAR_Technical.md`**

Add a Phase E section after Phase D:
- Protocol extension (speed/direction/moveFlags)
- Shared LocomotionApply primitive
- Drift threshold constants (for future co-op tuning)
- RE note: no native SetDesiredVelocity primitive — player-clone primitives reused

- [ ] **Step 4: Final commit + tag**

```bash
git add src/NPCSync/LocomotionApply.cpp docs/PULSAR_Technical.md
git commit -m "feat: Phase E — tuned + documented NPC movement sync"
git tag phase-e-shipped
```

---

## Risks + mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| NPC AnimGraphManager API differs from Player's | Medium | Task 9 Step 2 falls back to direct-vtable pattern from RemotePlayerManager |
| CharController absent on creatures/spaceships | Medium | Task 0 probe reveals; Task 9 Step 1 gates velocity writes |
| `Direction` semantics body-relative vs world | Low-Medium | Same global BSFixedString → same engine consumer → same convention (T41 confirmed) |
| 30 Hz Havok velocity writes race with render thread | Medium | Existing `SafeHavokBodySetPositionImpl` + attach-gate pattern handles this for player — same gates in Task 6 |
| Drift-snap crashes on no-3D NPCs | Low | Existing `has3DNow` guard above the apply block retained |

## Verification matrix

Before declaring Phase E done:
- [ ] 3+ simultaneous tracked NPCs walk smoothly on client (no teleports ≥1 min)
- [ ] Sprinting NPC triggers Walk→Run animation transition
- [ ] Direction changes: NPC rotates body smoothly as host-NPC turns
- [ ] Host-side crash rate unchanged vs pre-Phase-E baseline
- [ ] Client-side crash rate unchanged vs pre-Phase-E baseline
- [ ] SetPosition drift-snap fires ≤1× per NPC per 2s (normal walking)
