# Combat Sync — Client→Host Damage Forward (V2 Step 1) Implementation Plan


**Goal:** When the CLIENT player damages a host-matched ground enemy, that damage is applied to the HOST's authoritative copy of the same enemy, so the enemy dies on the host and the death mirrors back to the client — i.e. "client hits count."

**Architecture:** Both machines keep simulating combat independently (status quo). NPCSync's matcher already pairs enemies on the client (`g_npcRemap[hostRef] = localRef`, built in the apply loop). The client watches each matched enemy's Health each apply-tick; a *drop* = local-combat damage, which it forwards to the host as `PKT_NPC_DAMAGE{hostRef, delta}`. The host subtracts that from its enemy's Health; the host's existing per-tick capture then broadcasts the reduced Health + `isDead` back through the existing `PKT_NPC_STATE` path, which the client already applies. To stop the existing continuous host→client Health overwrite from "healing" a client-killed enemy and causing double-counting, the client **self-classifies**: the first local Health drop on a matched enemy flips it to `clientEngaged`, after which the client applies only DEATH from the host for that actor (not continuous Health).

**Tech Stack:** C++23, CommonLibSF, MinHook, ENet. Starfield 1.16.236 SFSE plugin. No unit-test framework — verification is in-game log inspection + a two-machine co-op test (project convention).

---

## Why this is the chosen first increment

Probe results (`memory/project_combat_sync.md`) showed dynamic combat enemies have **zero FormID overlap** host↔client and that NPCSync only matches by `effectiveBaseID`+position. The originally-decided "host-authoritative spawning" foundation would mean reviving the dead, crash-prone `PlaceAtMe` trampoline (`if (false)` block, NPCSync.cpp ~2079) + dead anti-dup native suppression (unreachable after the early `return` at NPCSync.cpp:1168). This increment instead reuses three things that **already exist and work**:

1. The matcher's `g_npcRemap[hostRef] = localRef` pairing (NPCSync.cpp ~2065).
2. The `PKT_NPC_STATE` wire format already carries `health`, `healthMax`, `flags` bit0=`isDead` (PacketTypes.h:285-287).
3. The client already applies host Health + death via `SetActorValue` (NPCSync.cpp:2408-2414).

Only the reverse direction (client→host damage) is missing. This plan adds exactly that, measures how reliable the matcher actually is for combat enemies, and defers spawning until proven necessary.

## Key facts grounding the code below (verified this session)

- Apply loop runs **client-only**: `ApplyPendingRemoteState()` returns early `if (IsHostMode())` (NPCSync.cpp:1850).
- In the apply loop body, for each pending update `u`, after match resolution we hold BOTH `u.refFormID` (= host's enemy FormID) and `actor` (= local matched Actor*). The damage check lives right here — no separate reverse map needed.
- Existing Health/death apply block: NPCSync.cpp:2405-2414.
- Health AV is cached in `g_healthAV` (std::atomic<ActorValueInfo*>), populated at the top of the apply loop (~NPCSync.cpp:1927-1932) and reused below as `healthAV`.
- `SafeSetActorValue(RE::Actor*, RE::ActorValueInfo*, float)` SEH wrapper exists (NPCSync.cpp:939).
- Host game-thread entry: `OnGameThreadTick()` (NPCSync.cpp:1566); host-only code begins after `if (!IsHostMode()) return;` (NPCSync.cpp:1572).
- Network send is symmetric: `Pulsar::Network::GetSingleton().Send(buf, len, reliable)` — on the client it routes to the host peer (same call PKT_CLIENT_PROFILE 0x60 client→host uses). `GetSingleton()` returns `NetworkHost&` (NetworkHost.h:136); has `IsConnected()`, `IsHost()`.
- ENet receive callbacks run on the **ENet thread** — must NOT call engine APIs there. Queue + drain on game thread (feedback: ENet thread-safe / no engine calls off game thread). The host damage-apply therefore queues in `OnDamageReceived` and applies in `OnGameThreadTick`.
- Packet dispatch: `NetworkHost::ProcessPacket` if-chain in NetworkHost.cpp; the `PKT_NPC_STATE` block is at NetworkHost.cpp:777-783 (gated `!m_isHost` = client receives). A client→host packet gates on `m_isHost`.
- Next free packet ID: highest in use is `0x95` (PKT_QUEST_STOP). Use `0x96`.

---

## File Structure

| File | Responsibility | Change |
|---|---|---|
| `include/Network/PacketTypes.h` | Wire enum + payload structs | Add `PKT_NPC_DAMAGE = 0x96` + `struct NpcDamageData` + static_assert |
| `include/NPCSync/NPCSync.h` | NPCSync public API | Declare `void OnDamageReceived(const uint8_t*, std::size_t);` |
| `src/NPCSync/NPCSync.cpp` | All NPCSync logic | Client: forward-state map + `SendNpcDamage` + apply-loop branch. Host: `OnDamageReceived` queue + drain in `OnGameThreadTick`. Clear state in `Reset()`. |
| `src/Network/NetworkHost.cpp` | Packet dispatch | Add `PKT_NPC_DAMAGE` → `NPCSync::OnDamageReceived` (host-side) |

No new files. All logic lives next to the code it extends — the apply loop, the host tick, the dispatch — so a reviewer reads one diff per concern.

---

### Task 1: Define the `PKT_NPC_DAMAGE` packet

**Files:**
- Modify: `include/Network/PacketTypes.h` (enum near line 38; struct block near line 290, after the `NPCStateData` static_asserts)

- [ ] **Step 1: Add the enum value**

In the packet-type enum, directly after the `PKT_QUEST_STOP = 0x95,` line, add:

```cpp
        PKT_NPC_DAMAGE       = 0x96,  // Client → Host: client-dealt combat damage to a host-matched NPC. Payload NpcDamageData{hostRefFormID, damage}. Host subtracts `damage` from that NPC's Health (engine processes death at <=0); the reduced Health + isDead echo back via the existing PKT_NPC_STATE path. Combat Sync V2 Step 1.
```

- [ ] **Step 2: Add the payload struct**

Immediately after the `static_assert(sizeof(NPCStateData) == 114, "NPCStateData size");` line (PacketTypes.h ~295), add:

```cpp
#pragma pack(push, 1)
    // PKT_NPC_DAMAGE payload — single client→host combat-damage event.
    // Wire format: [uint8 type][NpcDamageData]. Own framing (no PacketHeader),
    // same style as PKT_NPC_STATE. Sent reliably — deltas are cumulative, a
    // dropped one = lost damage.
    struct NpcDamageData
    {
        uint32_t hostRefFormID;  // The HOST's enemy FormID (key into host g_tracked / LookupByID). Comes from the client's g_npcRemap host-side key.
        float    damage;         // Positive final damage to subtract from the host enemy's Health this event.
    };
    // = 8 bytes
#pragma pack(pop)

    static_assert(sizeof(NpcDamageData) == 8, "NpcDamageData size");
```

- [ ] **Step 3: Build**

Run: `cmake --build <repo>\build\vs --config RelWithDebInfo --target PULSAR`
Expected: builds clean (header-only change; no warnings introduced).

- [ ] **Step 4: Commit**

```bash
git add include/Network/PacketTypes.h
git commit -m "feat(combat): add PKT_NPC_DAMAGE wire type (client->host damage forward)"
```

---

### Task 2: Declare the host receive entry point

**Files:**
- Modify: `include/NPCSync/NPCSync.h`

- [ ] **Step 1: Add the declaration**

In `namespace Pulsar::NPCSync`, next to the existing `void OnPacketReceived(...)` / `void OnCivilianSnapshotReceived(...)` declarations, add:

```cpp
    // Host-side: client forwarded combat damage for a host-matched NPC.
    // Called on the ENet thread — only queues; the actual SetActorValue
    // happens in OnGameThreadTick (engine calls must be on the game thread).
    void OnDamageReceived(const uint8_t* a_data, std::size_t a_length);
```

- [ ] **Step 2: Build**

Run: `cmake --build <repo>\build\vs --config RelWithDebInfo --target PULSAR`
Expected: builds clean (declaration only; unused until Task 4 defines it — but a declaration alone compiles).

- [ ] **Step 3: Commit**

```bash
git add include/NPCSync/NPCSync.h
git commit -m "feat(combat): declare NPCSync::OnDamageReceived"
```

---

### Task 3: Client — detect local combat damage and forward it

This is the core. We add a per-actor forward-state map, a send helper, and rewrite the existing Health/death apply block (NPCSync.cpp:2405-2414) to self-classify engaged enemies.

**Files:**
- Modify: `src/NPCSync/NPCSync.cpp`

- [ ] **Step 1: Add the per-actor forward-state map**

Add near the other client-side static maps (e.g. directly after `g_firstSynced`/`g_firstSyncedMutex`, NPCSync.cpp ~441):

```cpp
    // Combat Sync V2 Step 1 — client-side damage forwarding state, keyed by
    // LOCAL matched actor FormID.
    //   baseline       = the Health value we've already accounted for. Local
    //                    combat only ever lowers Health, so a read below
    //                    baseline = new client-dealt damage to forward.
    //   clientEngaged  = set true the first time we see a local drop. Once set,
    //                    we STOP applying continuous host Health to this actor
    //                    (death only) so the host's lagged "still alive" value
    //                    can't heal a client-killed enemy and cause us to
    //                    re-forward the same damage. Self-classifies enemies
    //                    vs. companions without any host-side flag.
    //   killApplied    = guards the one-shot SetActorValue(0) on the isDead edge.
    struct DamageFwdState {
        float baseline      = -1.0f;   // -1 = uninitialised
        bool  clientEngaged = false;
        bool  killApplied   = false;
    };
    static std::unordered_map<uint32_t, DamageFwdState> g_dmgFwd;
    static std::mutex g_dmgFwdMutex;
```

- [ ] **Step 2: Add the client send helper**

Add as a free function in `namespace Pulsar::NPCSync`, near the other send code (e.g. just above `OnGameThreadTick`, NPCSync.cpp ~1566). It must be reachable from the apply loop:

```cpp
    // Client → host: forward `dmg` of client-dealt damage for the host enemy
    // identified by hostRefFormID (the g_npcRemap key). Reliable: cumulative
    // deltas, a dropped packet = lost damage. Cheap (9 bytes); fires a few
    // times/sec per actively-shot enemy during combat.
    static void SendNpcDamage(uint32_t hostRefFormID, float dmg)
    {
        if (dmg <= 0.0f || hostRefFormID == 0) return;
        auto& net = Pulsar::Network::GetSingleton();
        if (!net.IsConnected()) return;

        using namespace Pulsar::Network;
        uint8_t buf[1 + sizeof(NpcDamageData)];
        buf[0] = PKT_NPC_DAMAGE;
        NpcDamageData d{};
        d.hostRefFormID = hostRefFormID;
        d.damage        = dmg;
        std::memcpy(&buf[1], &d, sizeof(d));
        net.Send(buf, sizeof(buf), /*reliable=*/true);

        static uint64_t s_lastLog = 0;
        uint64_t now = GetTickCount64();
        if (now - s_lastLog > 1000) {
            s_lastLog = now;
            REX::INFO("[COMBAT-FWD] client → host hostRef=0x{:08X} dmg={:.1f}"sv,
                      hostRefFormID, dmg);
        }
    }
```

- [ ] **Step 3: Rewrite the Health/death apply block**

Replace the existing block at NPCSync.cpp:2405-2414:

```cpp
            // Health + Death. Setting health to <= 0 causes the engine to process
            // the death naturally (ragdoll, animation, flags). Reliable packet delivery
            // on the isDead transition guarantees we catch the kill exactly once.
            if (healthAV) {
                float wantHealth = u.isDead ? 0.0f : u.health;
                float curHealth = actor->GetActorValue(*healthAV);
                if (std::abs(curHealth - wantHealth) > 0.5f) {
                    actor->SetActorValue(*healthAV, wantHealth);
                }
            }
```

with:

```cpp
            // Health + Death + client→host damage forward (Combat Sync V2 Step 1).
            if (healthAV) {
                constexpr float kDmgEps = 0.5f;
                float curHealth = actor->GetActorValue(*healthAV);

                DamageFwdState st;
                {
                    std::lock_guard lk(g_dmgFwdMutex);
                    auto& ref = g_dmgFwd[actor->formID];
                    if (ref.baseline < 0.0f) ref.baseline = curHealth;  // first sight
                    st = ref;  // snapshot for use below
                }

                // (1) Detect NEW client-dealt damage = a drop below baseline.
                if (curHealth < st.baseline - kDmgEps) {
                    float delta = st.baseline - curHealth;
                    SendNpcDamage(u.refFormID, delta);
                    std::lock_guard lk(g_dmgFwdMutex);
                    auto& ref = g_dmgFwd[actor->formID];
                    ref.baseline      = curHealth;   // baseline only descends
                    ref.clientEngaged = true;        // self-classify as a combat enemy
                    st = ref;
                }

                // (2) Apply host authority.
                if (st.clientEngaged) {
                    // Death only — do NOT fight the local combat sim by writing
                    // host's (lagged) alive-Health. Apply the kill once on the
                    // isDead edge.
                    if (u.isDead && !st.killApplied) {
                        actor->SetActorValue(*healthAV, 0.0f);
                        std::lock_guard lk(g_dmgFwdMutex);
                        g_dmgFwd[actor->formID].killApplied = true;
                    }
                } else {
                    // Not engaged (companions / NPCs the client hasn't shot):
                    // keep the original continuous host-Health authority, and
                    // track host's value as baseline so a host-driven drop
                    // (host shooting it) is NOT misread as client damage.
                    float wantHealth = u.isDead ? 0.0f : u.health;
                    if (std::abs(curHealth - wantHealth) > 0.5f) {
                        actor->SetActorValue(*healthAV, wantHealth);
                    }
                    std::lock_guard lk(g_dmgFwdMutex);
                    g_dmgFwd[actor->formID].baseline = wantHealth;
                }
            }
```

- [ ] **Step 4: Clear the map in `Reset()`**

In `void Reset()` (NPCSync.cpp:2427), add a scoped clear alongside the other map clears (e.g. after the `g_npcRemap.clear()` block ~2443):

```cpp
        {
            std::lock_guard lock(g_dmgFwdMutex);
            g_dmgFwd.clear();
        }
```

- [ ] **Step 5: Build**

Run: `cmake --build <repo>\build\vs --config RelWithDebInfo --target PULSAR`
Expected: builds clean. If `Pulsar::Network::GetSingleton`/`Send` aren't visible in NPCSync.cpp, confirm the existing `#include` that already backs the `net.Send` calls at NPCSync.cpp:1682/1744 (it is already included — no new include needed).

- [ ] **Step 6: Commit**

```bash
git add src/NPCSync/NPCSync.cpp
git commit -m "feat(combat): client forwards local combat damage to host (self-classifying engage)"
```

---

### Task 4: Host — apply forwarded damage on the game thread

**Files:**
- Modify: `src/NPCSync/NPCSync.cpp`

- [ ] **Step 1: Add the incoming-damage queue**

Add near the host-side tracking statics (e.g. after `g_tracked`/`g_trackedMutex`, NPCSync.cpp ~426):

```cpp
    // Combat Sync V2 Step 1 — host-side queue of client-forwarded damage.
    // Filled on the ENet thread by OnDamageReceived; drained + applied on the
    // game thread in OnGameThreadTick (SetActorValue must run on game thread).
    static std::vector<Pulsar::Network::NpcDamageData> g_pendingDamage;
    static std::mutex g_pendingDamageMutex;
```

- [ ] **Step 2: Define `OnDamageReceived` (ENet thread — queue only)**

Add as a free function in `namespace Pulsar::NPCSync`, near `OnPacketReceived` (NPCSync.cpp ~1780):

```cpp
    void OnDamageReceived(const uint8_t* a_data, std::size_t a_length)
    {
        using namespace Pulsar::Network;
        if (a_length < 1 + sizeof(NpcDamageData)) return;
        NpcDamageData d{};
        std::memcpy(&d, a_data + 1, sizeof(d));
        if (d.damage <= 0.0f || d.hostRefFormID == 0) return;
        std::lock_guard lk(g_pendingDamageMutex);
        g_pendingDamage.push_back(d);
    }
```

- [ ] **Step 3: Drain + apply in `OnGameThreadTick` (host-only region)**

In `OnGameThreadTick()`, after the host-only gate `if (!IsHostMode()) return;` (NPCSync.cpp:1572) and before the periodic-diagnostic block, add:

```cpp
        // Combat Sync V2 Step 1 — apply client-forwarded damage to the host's
        // authoritative enemies. The host's own per-tick capture then broadcasts
        // the reduced Health + isDead back via PKT_NPC_STATE (existing path).
        {
            std::vector<Pulsar::Network::NpcDamageData> dmgBatch;
            {
                std::lock_guard lk(g_pendingDamageMutex);
                dmgBatch.swap(g_pendingDamage);
            }
            if (!dmgBatch.empty()) {
                RE::ActorValueInfo* healthAV = g_healthAV.load(std::memory_order_acquire);
                if (!healthAV) {
                    healthAV = RE::TESForm::LookupByEditorID<RE::ActorValueInfo>("Health");
                    if (healthAV) g_healthAV.store(healthAV, std::memory_order_release);
                }
                for (const auto& d : dmgBatch) {
                    if (!healthAV) break;
                    RE::TESForm* form = RE::TESForm::LookupByID(d.hostRefFormID);
                    if (!form) continue;
                    RE::Actor* actor = form->As<RE::Actor>();
                    if (!actor && form->Is(RE::FormType::kACHR))
                        actor = reinterpret_cast<RE::Actor*>(form);
                    if (!actor || !actor->parentCell) continue;

                    float cur = actor->GetActorValue(*healthAV);
                    float next = cur - d.damage;
                    if (next < 0.0f) next = 0.0f;
                    actor->SetActorValue(*healthAV, next);   // engine processes death at <=0

                    REX::INFO("[COMBAT-FWD] host applied hostRef=0x{:08X} dmg={:.1f} health {:.1f} -> {:.1f}"sv,
                              d.hostRefFormID, d.damage, cur, next);
                }
            }
        }
```

- [ ] **Step 4: Clear the queue in `Reset()`**

In `void Reset()` (NPCSync.cpp:2427), alongside the other clears:

```cpp
        {
            std::lock_guard lock(g_pendingDamageMutex);
            g_pendingDamage.clear();
        }
```

- [ ] **Step 5: Build**

Run: `cmake --build <repo>\build\vs --config RelWithDebInfo --target PULSAR`
Expected: builds clean. (`g_healthAV` is the existing atomic used at NPCSync.cpp:1927; reuse it.)

- [ ] **Step 6: Commit**

```bash
git add src/NPCSync/NPCSync.cpp
git commit -m "feat(combat): host applies client-forwarded damage on game thread"
```

---

### Task 5: Wire the packet into host dispatch

**Files:**
- Modify: `src/Network/NetworkHost.cpp` (dispatch if-chain, after the `PKT_NPC_STATE` block at NetworkHost.cpp:777-783)

- [ ] **Step 1: Add the dispatch branch**

Directly after the `PKT_NPC_STATE` dispatch block (NetworkHost.cpp:783), add:

```cpp
        // PKT_NPC_DAMAGE — client forwarded combat damage for a host-matched
        // NPC (own framing, like PKT_NPC_STATE). Host-only: only the host owns
        // the authoritative enemy. Queues; applied on the game thread.
        if (m_isHost && a_packet->dataLength >= 1 + sizeof(NpcDamageData)
            && a_packet->data[0] == PKT_NPC_DAMAGE) {
            Pulsar::NPCSync::OnDamageReceived(a_packet->data, a_packet->dataLength);
            return;
        }
```

- [ ] **Step 2: Confirm includes**

`NpcDamageData` lives in `Pulsar::Network` (PacketTypes.h, already included here — `NPCStateBatchHeader` is used at NetworkHost.cpp:779) and `Pulsar::NPCSync::OnDamageReceived` is declared in `NPCSync/NPCSync.h` (already included — `Pulsar::NPCSync::OnPacketReceived` is called at NetworkHost.cpp:781). No new includes.

- [ ] **Step 3: Build**

Run: `cmake --build <repo>\build\vs --config RelWithDebInfo --target PULSAR`
Expected: builds clean.

- [ ] **Step 4: Commit**

```bash
git add src/Network/NetworkHost.cpp
git commit -m "feat(combat): dispatch PKT_NPC_DAMAGE to NPCSync on host"
```

---

### Task 6: Deploy + two-machine co-op verification

No unit tests exist for this codebase; verification is log + in-game (project convention). This is the real success gate.

**Files:** none (deploy + test only).

- [ ] **Step 1: Deploy**

Copy `PULSAR.dll` + `PULSAR.pdb` from `<repo>\build\vs\<config>\` to `<Starfield>\Data\SFSE\Plugins`, and copy `PULSAR.dll` to the NAS deploy dir (`\\<NAS>\PULSAR\deploy\Data\SFSE\Plugins`). **Restart BOTH games** (SFSE loads the DLL only at launch).

- [ ] **Step 2: Establish co-op + reach a shared ground fight**

Client joins host (auto-connect; F8 if needed). Both end up on the same ground location with hostile NPCs. Confirm the matcher is pairing enemies — host log should show `[PULSAR] NPCSync: MATCHED hostRef=0x... -> localRef=0x...` lines on the **client** for the enemies.

- [ ] **Step 3: Client shoots a matched enemy; collect logs**

Client fires on one matched enemy until it would die. Then read:
- Client log `\\<NAS>\PULSAR\logs\PULSAR.log` (via `[System.IO.File]::ReadAllLines`).
- Host log `%USERPROFILE%\Documents\My Games\Starfield\SFSE\Logs\PULSAR.log`.

- [ ] **Step 4: Verify the pipeline end-to-end**

Expected log evidence (this is the pass/fail criterion):
1. **Client** emits `[COMBAT-FWD] client → host hostRef=0x... dmg=...` while shooting (throttled to ~1/s).
2. **Host** emits `[COMBAT-FWD] host applied hostRef=0x... dmg=... health A -> B` with `B < A` for the same `hostRef`.
3. On the host machine, the enemy's Health visibly drops and the enemy **dies on the host** from the client's fire (with the host player not shooting it, or barely).
4. On the **client**, once the host enemy reaches 0 / `isDead`, the matched local enemy ends up dead too (death mirrored back through the existing `PKT_NPC_STATE` path).

- [ ] **Step 5: Record the result + known risks**

Update `memory/project_combat_sync.md` with the V2-Step-1 outcome. The two risks to watch and note explicitly:
- **Host kill-on-SetActorValue:** does `SetActorValue(Health, 0)` reliably kill a *live* host AI actor (ragdoll/flags), or does the enemy sit at 0 HP still alive? If the latter, V2-Step-1b = call a proper kill/HandleHealthDamage primitive on the host (needs a short Ghidra RE pass — none done yet).
- **Matcher coverage:** how many of the host's combat enemies actually got a `MATCHED` line on the client? Enemies with no match get no forwarding. The match rate decides whether the heavier host-authoritative *spawning* foundation is needed after all (the originally-decided path), or whether damage-forward over the matcher is enough.

---

## Self-Review

**Spec coverage** (against `memory/project_combat_sync.md` decomposition):
- Sub-feature 3 "Damage forward client→host via g_npcRemap NOT raw FormID" → Tasks 1-5. ✔ (forwarded key = `u.refFormID`, the host-side key of the matcher's remap, never a raw client FormID).
- Sub-feature 2 "Health sync host→client" + 4 "Death sync host→client" → already implemented (NPCSync.cpp:2408); this plan preserves it for non-engaged actors and uses death-only for engaged enemies. ✔
- Sub-feature 1 "host-authoritative spawning" → intentionally deferred (see "Why this is the chosen first increment"); Task 6 Step 5 produces the data that decides whether it's still needed. ✔ (scoped out by design, not a gap).
- Sub-feature 5 "NPCs attack client clone" → out of scope for this increment. ✔

**Placeholder scan:** no TBD/TODO; every code step shows complete code; the only RE-dependent item (host kill primitive) is gated behind a measured Task 6 outcome, not assumed.

**Type consistency:** `NpcDamageData{uint32_t hostRefFormID; float damage;}` defined in Task 1, used identically in `SendNpcDamage` (Task 3), `OnDamageReceived` (Task 4), and dispatch (Task 5). `g_dmgFwd`/`g_dmgFwdMutex`, `g_pendingDamage`/`g_pendingDamageMutex`, `DamageFwdState{baseline,clientEngaged,killApplied}`, `g_healthAV` (existing), `SendNpcDamage`/`OnDamageReceived` names are consistent across all tasks. `PKT_NPC_DAMAGE = 0x96` single source.
