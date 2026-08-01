# Co-op Revive (Inc4) Implementation Plan


**Goal:** Let a single downed co-op player be brought back up — automatically when combat ends, or by a partner holding Activate with a healing item.

**Architecture:** The revive is always EXECUTED on the downed player's own machine via one shared `ReviveLocalPlayer(hpFraction)` primitive (generalized from the existing F10 debug revive). Way B (combat-end) is detected locally via `Actor::IsInCombat()`. Way A (medipack) is detected on the partner's machine (proximity to the downed clone + Activate-hold + healing item consumed) and triggers the downed machine over a new `PKT_PLAYER_REVIVE 0x98` packet.

**Tech Stack:** C++23, SFSE, CommonLibSF, MinHook, ENet. Module: `src/Coop/CoopMortality.cpp`. Spec: `docs/design/specs/2026-05-30-coop-revive-inc4-design.md`.

**Testing note (read first):** This is a Starfield engine-mod DLL — there is **no unit-test framework**. "Verification" for every task = (1) the build succeeds, and (2) an in-game co-op test where you watch for the exact `[DOWNED]` log lines named in the task. Build + deploy + restart-both-games is the established loop. Logs: host = `%USERPROFILE%\Documents\My Games\Starfield\SFSE\Logs\PULSAR.log` (accumulates — filter by timestamp); client = `\\<NAS>\PULSAR\logs\PULSAR.log`.

**Build command:** `cmake --build <repo>\build\vs --config RelWithDebInfo --target PULSAR`
**Deploy:** copy `build\vs\RelWithDebInfo\PULSAR.dll` + `PULSAR.pdb` → `<Starfield>\Data\SFSE\Plugins`, and `PULSAR.dll` → `\\<NAS>\PULSAR\deploy\Data\SFSE\Plugins`, then restart BOTH games.
**Commit rules:** `git add` specific paths only (never `extern/CommonLibSF`). No `git push`.

---

## File Structure

- `include/Network/PacketTypes.h` — add `PKT_PLAYER_REVIVE = 0x98` and `struct PlayerReviveData`.
- `src/Coop/CoopMortality.cpp` — all revive logic: shared `ReviveLocalPlayer`, Way B poll, Way A partner detection, `SendRevive`, `OnReviveReceived`, healing-item helpers.
- `src/Network/NetworkHost.cpp` — dispatch `PKT_PLAYER_REVIVE` to `Mortality::OnReviveReceived`.
- Read-only dependency: `include/RemotePlayer/RemotePlayerManager.h` — accessor for the downed clone's `RE::Actor*` (for position/proximity).

Existing helpers in `CoopMortality.cpp` to reuse (already present, verified this session):
`SafeClearDeathState(player)`, `SafeReadMaxHealth(player, av, out)`, `SafeRaiseHealthTo(player, av, target)`, `SafeReadLifeStateBits(player, bits)`, `SendDownedState(bool)`, `HealthAV()`, `g_localDowned`, `g_remoteDowned`, `kDownedFloor`.

---

## Task 1: Extract the shared `ReviveLocalPlayer` primitive (refactor F10)

**Files:**
- Modify: `src/Coop/CoopMortality.cpp` (the F10 block in `OnGameThreadTick`, ~lines 404-429)

This is a pure refactor: pull the F10 revive body into a reusable function so Way A and Way B call the same code. Behavior must stay identical for F10.

- [ ] **Step 1: Add the `ReviveLocalPlayer` function**

Add this static function just above `OnGameThreadTick` (after `SafeReadMaxHealth`, ~line 347):

```cpp
    // Shared revive primitive — used by F10 debug revive, Way A (medipack), and
    // Way B (combat-end). Runs on the downed player's OWN machine.
    //   1. leave the death life-state (clear +0xF8 death bits)
    //   2. restore Health to a_hpFraction * permanentMax (floor fallback)
    //   3. clear downed flags + notify the peer (downed=0)
    static void ReviveLocalPlayer(RE::PlayerCharacter* a_player,
                                  RE::ActorValueInfo*  a_av,
                                  float                a_hpFraction,
                                  const char*          a_reason)
    {
        if (!a_player || !a_av) return;

        g_localDowned = false;
        g_remoteDowned.store(false, std::memory_order_relaxed);
        SendDownedState(false);                 // tell the peer we are back up
        SafeClearDeathState(a_player);          // leave the death life-state

        float maxHealth = 0.0f;
        if (SafeReadMaxHealth(a_player, a_av, maxHealth) && maxHealth > 0.0f) {
            const float restoreHP = maxHealth * a_hpFraction;
            SafeRaiseHealthTo(a_player, a_av, restoreHP);
            REX::INFO("[DOWNED] revive ({}) -> health={:.1f} ({:.0f}% max)"sv,
                      a_reason, restoreHP, a_hpFraction * 100.0f);
        } else {
            SafeRaiseHealthTo(a_player, a_av, kDownedFloor);
            REX::INFO("[DOWNED] revive ({}) -> health={:.1f} (max-read failed)"sv,
                      a_reason, kDownedFloor);
        }
    }
```

- [ ] **Step 2: Replace the F10 body to call it**

In `OnGameThreadTick`, replace the inner F10 revive block (the part inside `if (g_localDowned) { ... }` under the rising-edge `if (keyNow && !s_reviveKeyDown)`) with a single call. The new block:

```cpp
            if (keyNow && !s_reviveKeyDown) {
                // Rising edge
                if (g_localDowned) {
                    ReviveLocalPlayer(player, av, 0.5f, "F10");
                }
            }
            s_reviveKeyDown = keyNow;
```

- [ ] **Step 3: Build**

Run: `cmake --build <repo>\build\vs --config RelWithDebInfo --target PULSAR`
Expected: `PULSAR.vcxproj -> ...\PULSAR.dll`, no errors. (Pre-existing C4505 warnings from other files are fine.)

- [ ] **Step 4: Commit**

```bash
git add src/Coop/CoopMortality.cpp
git commit -m "refactor(coop): extract ReviveLocalPlayer from F10 (Inc4)"
```

---

## Task 2: Way B — combat-end auto-revive

**Files:**
- Modify: `src/Coop/CoopMortality.cpp` (`OnGameThreadTick`; add a SEH `IsInCombat` helper)

When the local player is downed and the engine reports the player is no longer in combat for a debounced window, auto-revive at 25%.

- [ ] **Step 1: Add a SEH-isolated combat-state read**

`Actor::IsInCombat()` is the CommonLibSF virtual at vtable index 0x16C (verified in `extern/CommonLibSF/include/RE/A/Actor.h:231`). Wrap it in SEH like the other engine reads. Add near `SafeReadLifeStateBits` (~line 99):

```cpp
    // SEH: read the engine combat-state flag (Actor vtable 0x16C).
    static bool SafeIsInCombat(RE::PlayerCharacter* a_this, bool& a_inCombat)
    {
        __try {
            a_inCombat = a_this->IsInCombat();
            return true;
        } __except (EXCEPTION_EXECUTE_HANDLER) { return false; }
    }
```

- [ ] **Step 2: Add the Way-B poll in `OnGameThreadTick`**

Insert this block right AFTER the post-load/revive recovery block (the one that clears stale downed when life-state reads alive) and BEFORE the HP-floor clamp block:

```cpp
        // ----------------------------------------------------------------
        // Way B — combat-end auto-revive. While downed, once the engine
        // reports the player is no longer in combat for kCombatClearMs
        // continuously (debounced against combat-flag flicker), stand back up
        // at 25% HP. IsInCombat() is the Actor virtual vt+0x16C.
        // ----------------------------------------------------------------
        if (g_localDowned) {
            static std::uint64_t s_combatClearSinceMs = 0;  // 0 = not currently clear
            bool inCombat = true;
            const bool ok = SafeIsInCombat(player, inCombat);
            const std::uint64_t nowMs = ::GetTickCount64();
            constexpr std::uint64_t kCombatClearMs = 1500;

            if (ok && !inCombat) {
                if (s_combatClearSinceMs == 0) s_combatClearSinceMs = nowMs;
                if (nowMs - s_combatClearSinceMs >= kCombatClearMs) {
                    s_combatClearSinceMs = 0;
                    ReviveLocalPlayer(player, av, 0.25f, "combat-over");
                }
            } else {
                s_combatClearSinceMs = 0;  // still in combat (or read failed) → reset timer
            }
        }
```

- [ ] **Step 3: Build**

Run: `cmake --build <repo>\build\vs --config RelWithDebInfo --target PULSAR`
Expected: builds clean.

- [ ] **Step 4: Deploy + in-game test**

Deploy DLL+PDB, restart both games. In a co-op fight, let ONE player drop to 0 HP (go downed), then kill/clear all enemies so combat ends.
Expected log on the downed player's machine:
- `[DOWNED] entered downed (lifeState=0x20000, reload suppressed)` when downed.
- After enemies are cleared (~1.5 s): `[DOWNED] revive (combat-over) -> health=... (25% max)`.
- Player stands up; no reload, no crash.

**If `IsInCombat()` stays true while downed (no auto-revive fires):** the player's combat flag is unreliable in the masked-death state. Apply the fallback in Step 5; otherwise skip Step 5.

- [ ] **Step 5 (only if Step 4 fallback needed): nearby-hostile combat check**

Replace the `SafeIsInCombat(player, ...)` signal with "any nearby living actor is in combat". NPCSync already reads `BOOL_BITS::kSearchingInCombat` (`1 << 14`) and the combat target at actor `+0x298` (see `src/NPCSync/NPCSync.cpp:810-865`). Add a helper that walks `RE::ProcessLists::GetSingleton()->highActorHandles`, and for each living actor within ~60 m of the player checks `actor->boolBits.all(RE::Actor::BOOL_BITS::kSearchingInCombat)` OR a non-zero combat target; SEH-wrapped. Treat "no such actor" as combat-over. Use this boolean in place of `!inCombat`.

```cpp
    // SEH fallback: is any living actor near the player still in combat?
    // kCombatScanUnits ~= the combat area radius in game units (tunable; ~100 m).
    static constexpr float kCombatScanUnits = 8000.0f;
    static bool SafeAnyNearbyHostileInCombat(RE::PlayerCharacter* a_player, bool& a_any)
    {
        __try {
            a_any = false;
            auto* pl = RE::ProcessLists::GetSingleton();
            if (!pl) return false;
            const RE::NiPoint3 pp = a_player->GetPosition();
            const float maxSq = kCombatScanUnits * kCombatScanUnits;
            for (auto& h : pl->highActorHandles) {
                auto sp = h.get();
                RE::Actor* act = sp.get();
                if (!act || act == a_player || act->IsDead()) continue;
                const RE::NiPoint3 ap = act->GetPosition();
                const float dx = ap.x - pp.x, dy = ap.y - pp.y, dz = ap.z - pp.z;
                if (dx*dx + dy*dy + dz*dz > maxSq) continue;
                if (act->boolBits.all(RE::Actor::BOOL_BITS::kSearchingInCombat)) { a_any = true; break; }
            }
            return true;
        } __except (EXCEPTION_EXECUTE_HANDLER) { return false; }
    }
```

Then in the Way-B block use: `const bool ok = SafeAnyNearbyHostileInCombat(player, anyCombat); ... if (ok && !anyCombat) { ... }`. (Verify `ProcessLists.h` / `highActorHandles` field name against CommonLibSF before building; adjust the iteration to the actual container type.)

- [ ] **Step 6: Commit**

```bash
git add src/Coop/CoopMortality.cpp
git commit -m "feat(coop): Way B combat-end auto-revive (Inc4)"
```

---

## Task 3: `PKT_PLAYER_REVIVE 0x98` packet + receive path

**Files:**
- Modify: `include/Network/PacketTypes.h`
- Modify: `src/Coop/CoopMortality.cpp` (`SendRevive`, `OnReviveReceived`)
- Modify: `src/Network/NetworkHost.cpp` (dispatch)

This wires the network trigger before Way A's detection (so Way A in Task 4 only needs to call `SendRevive`).

- [ ] **Step 1: Add the packet type + payload**

In `include/Network/PacketTypes.h`, after the `PKT_PLAYER_DOWNED = 0x97` line:

```cpp
        PKT_PLAYER_REVIVE    = 0x98,  // Partner → downed peer: revive request (medpack path). Authority stays with the downed machine: it runs ReviveLocalPlayer on receipt. Payload PlayerReviveData{playerSlot}. Coop Downed & Revive  Feature A, Inc4.
```

After the `PlayerDownedData` struct + its `static_assert` (~line 316-327):

```cpp
    // PKT_PLAYER_REVIVE payload — a partner asks the downed player to get up.
#pragma pack(push, 1)
    struct PlayerReviveData {
        uint8_t playerSlot;   // 0 = host, 1 = client — the DOWNED player being revived
        uint8_t pad[3];
    };
#pragma pack(pop)
    static_assert(sizeof(PlayerReviveData) == 4, "PlayerReviveData size");
```

- [ ] **Step 2: Implement `SendRevive` + `OnReviveReceived` in `CoopMortality.cpp`**

Add `SendRevive` next to `SendDownedState` (~line 57):

```cpp
    // Tell the downed peer to get back up (medpack revive). Reliable.
    static void SendRevive(std::uint8_t a_targetSlot)
    {
        auto& net = Pulsar::Network::GetSingleton();
        if (!net.IsConnected()) return;
        using namespace Pulsar::Network;
        std::uint8_t buf[1 + sizeof(PlayerReviveData)];
        buf[0] = PKT_PLAYER_REVIVE;
        PlayerReviveData d{};
        d.playerSlot = a_targetSlot;
        std::memcpy(&buf[1], &d, sizeof(d));
        net.Send(buf, sizeof(buf), /*reliable=*/true);
        REX::INFO("[DOWNED] sent revive request to slot {}"sv, a_targetSlot);
    }
```

Replace the empty `OnReviveReceived` stub (~line 463) with:

```cpp
    void OnReviveReceived(const std::uint8_t* a_data, std::size_t a_len)
    {
        using namespace Pulsar::Network;
        if (!a_data || a_len < 1 + sizeof(PlayerReviveData)) return;
        PlayerReviveData d{};
        std::memcpy(&d, a_data + 1, sizeof(d));

        // Only act if WE are the targeted (downed) player and actually downed.
        auto& net = Pulsar::Network::GetSingleton();
        const std::uint8_t mySlot = net.IsHost() ? 0 : 1;
        if (d.playerSlot != mySlot) return;
        if (!g_localDowned) {
            REX::INFO("[DOWNED] revive request ignored (not downed)"sv);
            return;
        }

        auto* player = RE::PlayerCharacter::GetSingleton();
        auto* av     = HealthAV();
        if (player && av) ReviveLocalPlayer(player, av, 0.5f, "medpack");
    }
```

- [ ] **Step 3: Dispatch in `NetworkHost.cpp`**

Find the `PKT_PLAYER_DOWNED` dispatch block in `OnReceive` (added in Inc3, calls `Mortality::OnDownedReceived`). Add an identical block right after it (bidirectional, NOT host-gated):

```cpp
    if (a_packet->dataLength >= 1 + sizeof(PlayerReviveData)
        && a_packet->data[0] == PKT_PLAYER_REVIVE) {
        Pulsar::Coop::Mortality::OnReviveReceived(a_packet->data, a_packet->dataLength);
        return;
    }
```

(`CoopMortality.h` is already included in `NetworkHost.cpp` from Inc3.)

- [ ] **Step 4: Build**

Run: `cmake --build <repo>\build\vs --config RelWithDebInfo --target PULSAR`
Expected: builds clean.

- [ ] **Step 5: Commit**

```bash
git add include/Network/PacketTypes.h src/Coop/CoopMortality.cpp src/Network/NetworkHost.cpp
git commit -m "feat(coop): PKT_PLAYER_REVIVE 0x98 + receive path (Inc4)"
```

---

## Task 4: Healing-item detection + consume helpers

**Files:**
- Modify: `src/Coop/CoopMortality.cpp`

The partner must (a) know it holds at least one healing item and (b) consume exactly one when reviving. Resolve the detection method first, then implement count + consume.

- [ ] **Step 1: Resolve the healing-item detection method**

Two acceptable approaches — pick based on what CommonLibSF exposes (check `extern/CommonLibSF/include/RE/A/AlchemyItem.h` and `RE/M/MagicItem.h` for an effects array + a way to read each effect's restored ActorValue):

- **Preferred (ID-free):** iterate the player's inventory; an item counts as "healing" if it is an `RE::AlchemyItem` whose effect list contains an effect that restores the `Health` ActorValue.
- **Fallback (ID table):** a fixed `static constexpr std::uint32_t kHealingBaseIDs[]` of the base FormIDs for Med Pack / Trauma Pack / Emergency Kit. Resolve the IDs by temporarily logging the player's aid items: in `OnGameThreadTick`, behind a one-shot key (e.g. F11), dump each inventory entry's `form->GetFormID()` + name; read the log, fill the table, remove the dump.

Document the chosen method in a comment. If unsure after reading the headers, implement the **Fallback** (it is guaranteed concrete) and leave a `// TODO(polish): switch to effect-based detection` note.

- [ ] **Step 2: Implement `SafeCountHealingItems` + `SafeConsumeOneHealingItem`**

Using the engine inventory API (`RE::TESObjectREFR::GetInventory()` returns a map of `TESBoundObject* -> {count, unique_ptr<InventoryEntryData>}`; verify the exact signature in `extern/CommonLibSF/include/RE/T/TESObjectREFR.h` before building). SEH-wrap both. Sketch:

```cpp
    // SEH: count healing items in the actor's inventory.
    static bool SafeCountHealingItems(RE::Actor* a_actor, int& a_count)
    {
        __try {
            a_count = 0;
            auto inv = a_actor->GetInventory();
            for (auto& [obj, data] : inv) {
                const std::int32_t cnt = data.first;
                if (cnt <= 0 || !obj) continue;
                if (IsHealingForm(obj)) a_count += cnt;   // IsHealingForm = chosen method from Step 1
            }
            return true;
        } __except (EXCEPTION_EXECUTE_HANDLER) { return false; }
    }

    // SEH: remove exactly one healing item. Returns true if one was removed.
    static bool SafeConsumeOneHealingItem(RE::Actor* a_actor)
    {
        __try {
            auto inv = a_actor->GetInventory();
            for (auto& [obj, data] : inv) {
                if (data.first <= 0 || !obj) continue;
                if (IsHealingForm(obj)) {
                    a_actor->RemoveItem(obj, 1, RE::ITEM_REMOVE_REASON::kRemove,
                                        nullptr, nullptr);  // verify RemoveItem signature
                    return true;
                }
            }
            return false;
        } __except (EXCEPTION_EXECUTE_HANDLER) { return false; }
    }
```

`IsHealingForm(RE::TESBoundObject*)` implements the Step-1 method (effect scan or ID-table membership). Place these above `OnGameThreadTick`.

- [ ] **Step 3: Build**

Run: `cmake --build <repo>\build\vs --config RelWithDebInfo --target PULSAR`
Expected: builds clean. (Resolve the exact `GetInventory` / `RemoveItem` signatures against the CommonLibSF headers — adjust the loop/types to compile.)

- [ ] **Step 4: Commit**

```bash
git add src/Coop/CoopMortality.cpp
git commit -m "feat(coop): healing-item detect + consume helpers (Inc4)"
```

---

## Task 5: Way A — partner proximity + Activate-hold → revive

**Files:**
- Modify: `src/Coop/CoopMortality.cpp` (`OnGameThreadTick`)
- Read-only: `include/RemotePlayer/RemotePlayerManager.h` (downed clone `RE::Actor*`)

On the partner's machine: when a peer is downed and we are near its clone, holding Activate, with a healing item → consume one and send the revive.

- [ ] **Step 1: Resolve the downed clone's position accessor**

Read `include/RemotePlayer/RemotePlayerManager.h`. There is a remote clone actor pointer (`m_remoteActor`, nulled by `ClearStaleClonePtrs` per the L.7 memory). Identify the public getter for the remote player's `RE::Actor*` (or its world position). If only a private member exists, add a minimal public accessor `RE::Actor* GetRemoteActor() const;` (and its definition) — this is the one allowed touch to RemotePlayerManager. Document which accessor you used.

- [ ] **Step 2: Add Way-A tunables**

Near the top of the `Pulsar::Coop::Mortality` namespace (by `kDownedFloor`, ~line 35):

```cpp
    static constexpr float        kReviveRadius      = 250.0f;  // game units (~2.5 m)
    static constexpr std::uint64_t kReviveHoldMs     = 2000;    // hold Activate this long
    static constexpr int          kActivateVK        = 0x45;    // 'E' (MVP; rebind = polish)
```

- [ ] **Step 3: Add the Way-A poll in `OnGameThreadTick`**

Insert AFTER the Way-B block. This runs on the HELPER (we are up, a peer is down):

```cpp
        // ----------------------------------------------------------------
        // Way A — medipack revive (helper side). If a peer is downed and we
        // are NOT downed, and we hold Activate within kReviveRadius of the
        // downed clone with >=1 healing item, consume one and send the revive.
        // ----------------------------------------------------------------
        if (!g_localDowned && g_remoteDowned.load(std::memory_order_relaxed)) {
            static std::uint64_t s_holdSinceMs = 0;
            const std::uint64_t nowMs = ::GetTickCount64();

            RE::Actor* clone = Pulsar::RemotePlayer::RemotePlayerManager::GetSingleton()
                                   .GetRemoteActor();   // accessor from Step 1
            bool inRange = false;
            if (clone) {
                __try {
                    const RE::NiPoint3 pp = player->GetPosition();
                    const RE::NiPoint3 cp = clone->GetPosition();
                    const float dx = cp.x - pp.x, dy = cp.y - pp.y, dz = cp.z - pp.z;
                    inRange = (dx*dx + dy*dy + dz*dz) <= (kReviveRadius * kReviveRadius);
                } __except (EXCEPTION_EXECUTE_HANDLER) { inRange = false; }
            }

            const bool activateHeld = (::GetAsyncKeyState(kActivateVK) & 0x8000) != 0;

            if (inRange && activateHeld) {
                if (s_holdSinceMs == 0) s_holdSinceMs = nowMs;
                if (nowMs - s_holdSinceMs >= kReviveHoldMs) {
                    s_holdSinceMs = 0;
                    int healing = 0;
                    if (SafeCountHealingItems(player, healing) && healing > 0) {
                        if (SafeConsumeOneHealingItem(player)) {
                            auto& net = Pulsar::Network::GetSingleton();
                            const std::uint8_t downedSlot = net.IsHost() ? 1 : 0; // the OTHER player
                            SendRevive(downedSlot);
                            REX::INFO("[DOWNED] medpack revive triggered for slot {} (1 healing item consumed)"sv,
                                      downedSlot);
                        }
                    } else {
                        REX::INFO("[DOWNED] revive hold complete but no healing item"sv);
                    }
                }
            } else {
                s_holdSinceMs = 0;   // released or out of range → reset hold timer
            }
        }
```

Add `#include "RemotePlayer/RemotePlayerManager.h"` to the includes if not present.

- [ ] **Step 4: Build**

Run: `cmake --build <repo>\build\vs --config RelWithDebInfo --target PULSAR`
Expected: builds clean.

- [ ] **Step 5: Deploy + in-game test (the full revive loop)**

Deploy DLL+PDB, restart both games. In a co-op fight:
1. Let ONE player drop to 0 HP (downed). The other stays up and stays in combat (so Way B does NOT fire).
2. The standing partner (with a Med/Trauma/Emergency item) walks onto the downed player and holds `E` ~2 s.

Expected logs:
- Helper machine: `[DOWNED] medpack revive triggered for slot N (1 healing item consumed)` and `[DOWNED] sent revive request to slot N`.
- Downed machine: `[DOWNED] revive (medpack) -> health=... (50% max)`; player stands up; 1 healing item gone from the helper's inventory; no crash.

- [ ] **Step 6: Commit**

```bash
git add src/Coop/CoopMortality.cpp include/RemotePlayer/RemotePlayerManager.h
git commit -m "feat(coop): Way A medpack revive (proximity + Activate-hold) (Inc4)"
```

---

## Final integration test

- [ ] **Way B:** one player downed → clear all enemies → downed player auto-stands at 25% (no item).
- [ ] **Way A:** one player downed (combat ongoing) → partner holds E with a healing item → downed player stands at 50%, one item consumed.
- [ ] **Both-down still game-over:** both players downed at the same time before any revive → both load last save, host does NOT crash (V13m), clean post-load recovery.
- [ ] **No false revives:** standing near a downed clone WITHOUT a healing item, or while still in combat, does not revive.
- [ ] After all pass, update memory: mark Inc4 confirmed in `project_coop_downed_revive.md` + `MEMORY.md`; note the final healing-item method + clone accessor used.
