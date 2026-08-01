# Co-op Downed & Revive (Feature A) Implementation Plan


**Goal:** When a co-op player would die, they instead enter a *downed* (bleedout) state that their partner can revive; the downed/revive state is mirrored onto the player's clone, and both-downed = real game-over.

**Architecture:** Each player is authoritative for their OWN health/downed/revive (separate HP pools, no cross-machine player-damage forwarding). The engine's essential-actor *bleedout* is repurposed as the downed state: at 0 HP an essential actor collapses instead of triggering game-over. Local downed-enter → `PKT_PLAYER_DOWNED` → the remote machine puts the player's clone in a downed pose. Partner activates the downed clone → `PKT_PLAYER_REVIVE` → on the downed player's own machine bleedout ends + HP is restored. Reuses the Combat-Sync infra (SEH-wrapped engine primitive via `REL::Offset`, packet + `NetworkHost` dispatch, game-thread drain, NAS logging).

**Tech Stack:** C++23, SFSE, CommonLibSF (`RE::Actor`/`RE::PlayerCharacter`), MinHook, ENet. Engine = Starfield 1.16.236. No unit-test harness (project convention) — every task is verified by build + deploy + two-machine co-op observation via the NAS/host logs.

---

## 🔄 HP-FLOOR PIVOT (2026-05-27) — Essential gate FAILED, plan revised

The Task 1 probe  tested in co-op: **the Essential/Bleedout approach is dead for the player.** The runtime `boolFlags::kEssential` is cleared by the engine, and the player at 0 HP never enters `kDead`/`kInBleedoutAnimation` — it lingers at negative HP for 2–6 s, then a *separate* game-over system reloads. Bleedout is an NPC-only mechanic. (Full evidence: `memory/project_coop_downed_revive.md`.) A prerequisite crash was also found + fixed: `RemotePlayerManager::ProcessPending` made unguarded `NotifyAnimationGraphImpl` virtual calls on a stale clone .

**User-approved pivot → Fallback C: HP-floor + custom downed state machine.** Revised increment list (replaces Tasks 1–2's essential/RE parts; Tasks 3/5/6/7 packet+sync+revive+both-downed structure still applies):

- **Inc 1 — HP-floor core (NEW GATE):** per-tick, when local player Health ≤ threshold → enter custom `g_localDowned`, clamp Health to a positive floor each tick so the game-over system never fires. Debug key to manually revive. Gate Q: does clamping reliably prevent the reload? (in-game test)
- **Inc 2 — incapacitation:** down/freeze the player without death (disable controls + knockdown anim, or `BOOL_BITS::kParalyzed`) — experiment.
- **Inc 3 — sync:** `PKT_PLAYER_DOWNED` (0x97) broadcast + receiver + clone downed pose.
- **Inc 4 — revive:** restore partial HP + clear flag + unfreeze; revive interaction (activate downed clone → `PKT_PLAYER_REVIVE` 0x98 → owner self-revives).
- **Inc 5 — both-downed → real game-over** (let it proceed) → rejoin via save-bridge. (Crash fix 4e89277 covers the clone-invalidation on this path.)

Everything below this banner is the original Essential-based plan, kept for reference. Follow the increment list above instead.

---

## ⚠️ Hard gate up front (read before starting)

The **entire Essential/Bleedout architecture hinges on Task 1**: does flagging the *player* `kEssential` produce a bleedout/downed state instead of Starfield's vanilla game-over? Players are not normally essential, so this is unverified. **Task 1 is a throwaway-able empirical probe whose only job is to answer that question.**

- **If Task 1 shows bleedout/collapse (no game-over):** the approach holds → continue to Task 2+.
- **If Task 1 shows game-over anyway:** STOP. Do not build Tasks 3–7 as written. Record the finding in `memory/project_coop_downed_revive.md` and re-plan with a fallback:
  - **Fallback B:** Ghidra-RE the player death / game-over trigger function and hook it (suppress + force bleedout-equivalent state).
  - **Fallback C:** HP-floor (clamp player health to 1 and drive a fully custom downed state machine, no engine bleedout).

Do not write speculative code for Tasks 3–7 before Task 1's gate passes.

---

## File structure

| File | Responsibility |
|---|---|
| `include/Coop/CoopMortality.h` | **NEW** — public API for the mortality module: `Init()`, `OnGameThreadTick()`, `OnDownedReceived(...)`, `OnReviveReceived(...)`. One namespace `Pulsar::Coop::Mortality`. |
| `src/Coop/CoopMortality.cpp` | **NEW** — the whole feature: essential-set, downed detection, send/recv, clone pose apply, revive interaction, both-downed handler, engine-primitive wrappers (`SafeReviveActor`). Single focused file, ~mirrors `NPCSync.cpp`'s structure (statics + tick drain + SEH-wrapped primitives). |
| `include/Network/PacketTypes.h` | **MODIFY** — add `PKT_PLAYER_DOWNED = 0x97`, `PKT_PLAYER_REVIVE = 0x98` + two payload structs. |
| `src/Network/NetworkHost.cpp` | **MODIFY** — dispatch the two new packet types to `CoopMortality`. |
| `src/Main.cpp` | **MODIFY** — `Init()` call (~line 837) + `OnGameThreadTick()` call (~line 1326). |
| `CMakeLists.txt` | **MODIFY only if not globbed** — add the new source. |
| `local RE notes | **NEW (Task 2)** — Ghidra script to RE the revive/clear-bleedout primitive (template: `FindKillNative.java`). |

Single new module file because all six responsibilities share the same state (which player is downed, the cached Health AV, the clone pointer) and the same game-thread tick — splitting would create cross-file shared statics. If `CoopMortality.cpp` grows past ~600 lines during execution, split the engine-primitive wrappers into `CoopMortalityPrimitives.cpp`.

---

## Task 1: Discovery probe — make the local player essential & observe (THE GATE)

**Files:**
- Create: `include/Coop/CoopMortality.h`
- Create: `src/Coop/CoopMortality.cpp`
- Modify: `src/Main.cpp:837` (Init), `src/Main.cpp:1326` (tick)
- Modify: `CMakeLists.txt` (only if sources are not auto-globbed — check first)

- [ ] **Step 1: Create the module header**

`include/Coop/CoopMortality.h`:
```cpp
#pragma once
#include <cstdint>
#include <cstddef>

namespace Pulsar::Coop::Mortality
{
    void Init();
    void OnGameThreadTick();   // game thread, ~1000x/sec; gated internally

    // ENet-thread receivers (queue only, applied on the game thread).
    void OnDownedReceived(const std::uint8_t* a_data, std::size_t a_length);
    void OnReviveReceived(const std::uint8_t* a_data, std::size_t a_length);
}
```

- [ ] **Step 2: Create the probe implementation**

`src/Coop/CoopMortality.cpp` — Task 1 is probe-only: set the local player essential once, then log health + flags whenever health is low. No packets yet.
```cpp
#include "PCH.h"
#include "Coop/CoopMortality.h"
#include "Network/NetworkHost.h"
#include "NASLog.h"

#include <RE/A/Actor.h>
#include <RE/P/PlayerCharacter.h>
#include <RE/A/ActorValueInfo.h>
#include <RE/T/TESDataHandler.h>

#ifndef WIN32_LEAN_AND_MEAN
#define WIN32_LEAN_AND_MEAN
#endif
#include <windows.h>

using namespace std::string_view_literals;

namespace Pulsar::Coop::Mortality
{
    static bool                   g_inited = false;
    static bool                   g_essentialSet = false;
    static RE::ActorValueInfo*    g_healthAV = nullptr;

    static RE::ActorValueInfo* HealthAV()
    {
        if (!g_healthAV) {
            if (auto* dh = RE::TESDataHandler::GetSingleton())
                g_healthAV = dh->LookupForm<RE::ActorValueInfo>(0x000002D4, "Starfield.esm"sv);
            // Fallback by editor id if the hardcoded form lookup misses.
            if (!g_healthAV)
                g_healthAV = RE::TESForm::LookupByEditorID<RE::ActorValueInfo>("Health"sv);
        }
        return g_healthAV;
    }

    // Set the runtime per-instance essential flag (BOOL_FLAGS::kEssential = 1<<18 @ 0x378).
    // This is the per-instance flag; if the probe shows it does NOT cause bleedout,
    // Step 6 will additionally try the base-data flag.
    static int SafeSetEssential(RE::Actor* a_actor, bool a_on)
    {
        __try {
            if (a_on) a_actor->boolFlags.set(RE::Actor::BOOL_FLAGS::kEssential);
            else      a_actor->boolFlags.reset(RE::Actor::BOOL_FLAGS::kEssential);
            return 1;
        } __except (EXCEPTION_EXECUTE_HANDLER) { return 0; }
    }

    void Init()
    {
        if (g_inited) return;
        g_inited = true;
        REX::INFO("[DOWNED] CoopMortality probe initialized"sv);
    }

    void OnGameThreadTick()
    {
        // Probe runs only in an active co-op session (host or client).
        auto& net = Pulsar::Network::GetSingleton();
        if (!net.IsConnected()) return;

        auto* player = RE::PlayerCharacter::GetSingleton();
        if (!player) return;

        if (!g_essentialSet) {
            int ok = SafeSetEssential(player, true);
            g_essentialSet = true;
            bool isEss = false;
            __try { isEss = player->boolFlags.all(RE::Actor::BOOL_FLAGS::kEssential); }
            __except (EXCEPTION_EXECUTE_HANDLER) {}
            REX::INFO("[DOWNED] set player essential ok={} flagReadsEssential={}"sv, ok, isEss);
        }

        // Log player state only when health is low (avoid log spam).
        auto* av = HealthAV();
        if (!av) return;
        float health = 0.0f, dead = 0.0f, bleedFlag = 0.0f;
        __try {
            health = player->AsActorValueOwner()->GetActorValue(*av);
            dead   = player->IsDead() ? 1.0f : 0.0f;
            // boolBits dump for the downed/bleedout transition discovery.
            bleedFlag = player->boolBits.underlying();
        } __except (EXCEPTION_EXECUTE_HANDLER) { return; }

        if (health <= 20.0f) {
            static uint64_t s_lastLog = 0;
            uint64_t now = ::GetTickCount64();
            if (now - s_lastLog > 250) {  // 4 Hz while low
                s_lastLog = now;
                REX::INFO("[DOWNED-PROBE] health={:.1f} dead={} boolBits=0x{:08X} boolFlags=0x{:08X}"sv,
                          health, dead, bleedFlag, (uint32_t)player->boolFlags.underlying());
            }
        }
    }

    // Stubs — wired in Task 3+. Present so the header compiles.
    void OnDownedReceived(const std::uint8_t*, std::size_t) {}
    void OnReviveReceived(const std::uint8_t*, std::size_t) {}
}
```
> **Note on API names:** verify against the installed CommonLibSF before building — `boolFlags.underlying()`, `boolBits.underlying()`, `AsActorValueOwner()`, and `LookupForm<ActorValueInfo>` are the expected `REX::TEnumSet` / `RE::Actor` shapes, but per `memory/feedback_commonlibsf_verify` confirm exact spellings in `extern/CommonLibSF/include/RE/A/Actor.h` and `RE/A/ActorValueOwner.h`. If `boolFlags.underlying()` isn't exposed, cast via `static_cast<std::uint32_t>(player->boolFlags.get())` or read the raw member. The Combat-Sync code in `NPCSync.cpp` already reads Health via a cached AV — copy its exact accessor pattern if these differ.

- [ ] **Step 3: Wire Init + tick into Main.cpp**

In `src/Main.cpp`, add to the init block (after `Pulsar::InventorySync::Init();` ~line 837):
```cpp
        Pulsar::Coop::Mortality::Init();
```
And in the game-thread tick, immediately after `Pulsar::NPCSync::OnGameThreadTick();` (~line 1326):
```cpp
                Pulsar::Coop::Mortality::OnGameThreadTick();
```
Add `#include "Coop/CoopMortality.h"` to the Main.cpp include block alongside the other module headers.

- [ ] **Step 4: Ensure the source is in the build**

Check `CMakeLists.txt` for how sources are collected. Run:
```
Grep pattern="GLOB|file\\(|add_library|target_sources|\\.cpp" path="<repo>\\CMakeLists.txt"
```
If sources are globbed (e.g. `file(GLOB_RECURSE ... *.cpp)`), no edit needed. If listed explicitly, add `src/Coop/CoopMortality.cpp`.

- [ ] **Step 5: Build, deploy, and run the empirical gate test**

Build:
```
cmake --build <repo>\build\vs --config RelWithDebInfo --target PULSAR
```
Expected: compiles clean, no warnings in `CoopMortality.cpp`.

Deploy (per project convention) DLL+PDB → `<Starfield>\Data\SFSE\Plugins` and DLL → `\\<NAS>\PULSAR\deploy\Data\SFSE\Plugins`, then restart BOTH games (SFSE loads the DLL only at launch).

**In-game gate test (ask the user to run — needs a live session):**
1. Start a co-op session (host + client connected).
2. Take the local player to 0 HP (let an NPC kill you, or `player.damageav health 1000` in console).
3. Observe: does the player **collapse/bleed out (no game-over screen)**, or does the **vanilla "you died / load" game-over** appear?

Read logs: client `\\<NAS>\PULSAR\logs\PULSAR.log`, host `%USERPROFILE%\Documents\My Games\Starfield\SFSE\Logs\PULSAR.log`. Look for `[DOWNED] set player essential ok=1 flagReadsEssential=1` and the `[DOWNED-PROBE]` lines as health crosses 0 — note the `boolBits` value at the moment of collapse (that's the bleedout/downed bit we'll read in Task 4).

- [ ] **Step 6: Evaluate the gate (decision, not code)**

- **Bleedout observed → GATE PASSES.** Record in `memory/project_coop_downed_revive.md`: which flag (runtime `boolFlags` vs needs base-data), and the `boolBits` value seen at collapse. Continue to Task 2.
- **If the runtime flag did NOT cause bleedout:** before failing the gate, try the base-data essential flag (one more probe build): set `player->GetActorRuntimeData().*`/`GetActorBase()`'s `actorData.actorBaseFlags.set(RE::ACTOR_BASE_DATA::Flag::kEssential)` (the flag `IsEssential()` reads). Re-test. If THAT causes bleedout, record it and continue.
- **Game-over either way → GATE FAILS.** Stop, record the finding, and re-plan per the Hard-Gate fallbacks (B: hook death fn; C: HP-floor + custom state machine). Do not proceed to Task 3.

- [ ] **Step 7: Commit the probe**
```
git add include/Coop/CoopMortality.h src/Coop/CoopMortality.cpp src/Main.cpp
git commit -m "feat(coop): downed-revive probe — player essential + bleedout gate "
```
(Add `CMakeLists.txt` to the commit only if you edited it. Never `git add -A` — the `extern/CommonLibSF` submodule must stay unstaged.)

---

## Task 2: RE the bleedout-state read + revive primitive (Ghidra) — GATED on Task 1

> Only start this once Task 1's gate has passed. The goal: a reliable way to (a) read "is this actor downed/bleeding out", and (b) end bleedout + restore the actor to a standing, alive state (the revive). Model the whole effort on how the Combat-Sync kill primitive `FUN_141fcb950` was found (`memory/project_combat_sync.md`, scripts in `local RE notes).

**Files:**
- Create: `local RE notes (template: `local RE notes)
- Read: `extern/CommonLibSF/include/RE/A/Actor.h` (Resurrect is vtable slot 0x139 per memory — confirm), `RE/A/ActorValueOwner.h` (RestoreActorValue = slot 09)

- [ ] **Step 1: Confirm the downed-state read from Task 1's evidence**

From Task 1's `[DOWNED-PROBE]` logs, identify the bit in `boolBits` (or a dedicated bleedout flag) that flips at collapse. If a clean `boolBits` bit exists, that's the read — no RE needed for detection. If not, search CommonLibSF for a bleedout accessor:
```
Grep pattern="BleedingOut|BleedOut|GetKnockState|knockState|IsDowned|kUnconscious" path="<repo>\\extern\\CommonLibSF\\include\\RE" output_mode="content" -n=true
```
Record the chosen detection method in memory.

- [ ] **Step 2: Try the clean CommonLibSF revive path first (Resurrect)**

Per `memory/project_combat_sync.md`, `Actor::Resurrect` is vtable slot **0x139**. A revive may be as simple as: clamp/restore Health AV to a partial value (`RestoreActorValue(kDamage, Health, +amount)` — slot 09 inverse of the kill path) + clear the bleedout/dead bit. Read `Actor.h` to confirm the `Resurrect` signature. If a virtual `Resurrect(bool resetInventory)` exists and works on a bled-out (not truly dead) actor, prefer it — no raw RVA needed.

- [ ] **Step 3: If no clean API works, RE the revive primitive with Ghidra**

Write `local RE notes (copy `FindKillNative.java`, swap the Papyrus string). Search for the Papyrus native that ends bleedout — candidates: `"Resurrect"`, `"StartDeferredKill"`/`"EndDeferredKill"`, `"SetUnconscious"`, `"ResetHealthAndLimbs"`. Find the string → xref → registrar → handler → inner worker, exactly like the kill chain (`FUN_141fcb950`). Run headless:
```
"<local>/ghidra_12.0.4_PUBLIC/support/analyzeHeadless.bat" "the local Ghidra project" StarfieldRE -process Starfield.exe -readOnly -noanalysis -scriptPath <local>/tools -postScript FindReviveNative.java
```
Reverse-lookup the RVA → REL::ID with `read_versionlib.py`; if no REL::ID exists, use `REL::Offset{<rva>}` like `SafeKillActor`.

- [ ] **Step 4: Add the SEH-wrapped revive wrapper to CoopMortality.cpp**

Mirror `SafeKillActor` (from `NPCSync.cpp`). Example shape (fill the resolved primitive from Step 2/3):
```cpp
    static int SafeReviveActor(RE::Actor* a_actor)
    {
        __try {
            // Preferred clean path (confirmed in Step 2): restore HP + clear downed.
            auto* av = HealthAV();
            float maxH = a_actor->AsActorValueOwner()->GetPermanentActorValue(*av);
            float cur  = a_actor->AsActorValueOwner()->GetActorValue(*av);
            float want = maxH * 0.35f;                 // partial revive
            if (cur < want)
                a_actor->AsActorValueOwner()->RestoreActorValue(
                    RE::ACTOR_VALUE_MODIFIER::kDamage, *av, want - cur);
            // clear bleedout/dead bit (exact bit/call from Task 2 Step 1/3)
            a_actor->boolBits.reset(RE::Actor::BOOL_BITS::kDead);
            return 1;
        } __except (EXCEPTION_EXECUTE_HANDLER) { return 0; }
    }
```
> The exact clear-bleedout call is whatever Task 2 resolved. If a raw RVA was needed, declare it like `static REL::Relocation<ReviveFn> g_reviveFn{ REL::Offset{ 0x... } };` and call it inside the `__try`.

- [ ] **Step 5: Commit the RE findings + wrapper**
```
git add src/Coop/CoopMortality.cpp local RE notes
git commit -m "feat(coop): RE revive primitive + SafeReviveActor "
```
Also record the RVA/method in `memory/project_coop_downed_revive.md`.

---

## Task 3: Packet types + dispatch

**Files:**
- Modify: `include/Network/PacketTypes.h` (after `PKT_NPC_DAMAGE = 0x96` and after the `NpcDamageData` struct block)
- Modify: `src/Network/NetworkHost.cpp` (dispatch block ~line 783, alongside the `PKT_NPC_DAMAGE` handler)

- [ ] **Step 1: Add the packet IDs**

In `include/Network/PacketTypes.h`, after the `PKT_NPC_DAMAGE = 0x96,` line:
```cpp
        PKT_PLAYER_DOWNED    = 0x97,  // Either side → peer: local player entered/left the downed (bleedout) state. Payload PlayerDownedData. Authority = sender's own machine. Coop Downed & Revive  Feature A.
        PKT_PLAYER_REVIVE    = 0x98,  // Either side → peer: instruct the (downed) peer to revive — sent when the local player activates the downed clone. Payload PlayerReviveData.
```

- [ ] **Step 2: Add the payload structs**

After the `static_assert(sizeof(NpcDamageData) == 9, ...)` block in `PacketTypes.h`:
```cpp
#pragma pack(push, 1)
    // PKT_PLAYER_DOWNED — sender's own downed-state transition (own-machine authority).
    struct PlayerDownedData
    {
        uint8_t  downed;   // 1 = entered downed/bleedout, 0 = left it (revived or stood up)
        float    x, y, z;  // sender position at the transition (so the clone is posed in place)
    };
    // = 13 bytes
#pragma pack(pop)
    static_assert(sizeof(PlayerDownedData) == 13, "PlayerDownedData size");

#pragma pack(push, 1)
    // PKT_PLAYER_REVIVE — partner instructs the downed peer to revive.
    struct PlayerReviveData
    {
        uint8_t  reserved;  // padding / future flags; 0 for now
    };
    // = 1 byte
#pragma pack(pop)
    static_assert(sizeof(PlayerReviveData) == 1, "PlayerReviveData size");
```

- [ ] **Step 3: Dispatch both packets in NetworkHost.cpp**

In `src/Network/NetworkHost.cpp`, alongside the existing `PKT_NPC_DAMAGE` dispatch (~line 783). Note these are NOT host-only (`PKT_NPC_DAMAGE` is host-only; mortality packets flow both directions):
```cpp
            if (dataLength >= 1 + sizeof(PlayerDownedData) && data[0] == PKT_PLAYER_DOWNED) {
                Pulsar::Coop::Mortality::OnDownedReceived(data, dataLength);
                return;
            }
            if (dataLength >= 1 + sizeof(PlayerReviveData) && data[0] == PKT_PLAYER_REVIVE) {
                Pulsar::Coop::Mortality::OnReviveReceived(data, dataLength);
                return;
            }
```
Add `#include "Coop/CoopMortality.h"` to `NetworkHost.cpp` if not already present.

- [ ] **Step 4: Build to verify wiring**
```
cmake --build <repo>\build\vs --config RelWithDebInfo --target PULSAR
```
Expected: clean build (receivers are still stubs — fine).

- [ ] **Step 5: Commit**
```
git add include/Network/PacketTypes.h src/Network/NetworkHost.cpp
git commit -m "feat(coop): PKT_PLAYER_DOWNED/REVIVE packets + dispatch "
```

---

## Task 4: Downed detection + broadcast (own-machine authority)

**Files:**
- Modify: `src/Coop/CoopMortality.cpp`

- [ ] **Step 1: Add the downed-detection state + send helpers**

Replace the Task-1 probe `OnGameThreadTick` body. Add statics near the top of the namespace:
```cpp
    static bool   g_localDowned = false;   // edge-tracked: is the local player currently downed?
    static float  g_lowHealthThreshold = 0.0f;

    static bool IsLocalDowned(RE::Actor* a_player)
    {
        // Detection method confirmed in Task 2 Step 1. Default: essential actor at
        // <=0 effective health while not truly dead == bleeding out / downed.
        __try {
            auto* av = HealthAV();
            if (!av) return false;
            float h = a_player->AsActorValueOwner()->GetActorValue(*av);
            bool trulyDead = a_player->IsDead();
            return (h <= 0.0f) && !trulyDead;   // refine with the boolBits bit from Task 2 if available
        } __except (EXCEPTION_EXECUTE_HANDLER) { return false; }
    }

    static void SendDowned(bool a_downed, RE::Actor* a_player)
    {
        Network::PlayerDownedData d{};
        d.downed = a_downed ? 1 : 0;
        __try {
            auto pos = a_player->GetPosition();
            d.x = pos.x; d.y = pos.y; d.z = pos.z;
        } __except (EXCEPTION_EXECUTE_HANDLER) { d.x = d.y = d.z = 0.0f; }

        uint8_t buf[1 + sizeof(Network::PlayerDownedData)];
        buf[0] = Network::PKT_PLAYER_DOWNED;
        std::memcpy(buf + 1, &d, sizeof(d));
        Pulsar::Network::GetSingleton().Send(buf, sizeof(buf), true);
        REX::INFO("[DOWNED] local -> peer downed={}"sv, (int)d.downed);
    }
```

- [ ] **Step 2: Edge-trigger downed enter/exit in the tick**

In `OnGameThreadTick`, after the essential-set block, replace the probe logging with:
```cpp
        auto* av = HealthAV();
        if (!av) return;
        bool downedNow = IsLocalDowned(player);
        if (downedNow != g_localDowned) {
            g_localDowned = downedNow;
            SendDowned(downedNow, player);   // edge-triggered, one packet per transition
        }
```

- [ ] **Step 3: Build, deploy, test the send half**

Build + deploy + restart both games. In co-op, take one player to 0 HP. **Expected logs:** the downed player logs `[DOWNED] local -> peer downed=1`; on revive/stand it logs `downed=0`. (Clone apply is Task 5 — the partner won't see a pose yet, only the log proves the send.)

- [ ] **Step 4: Commit**
```
git add src/Coop/CoopMortality.cpp
git commit -m "feat(coop): downed detection + edge-triggered broadcast "
```

---

## Task 5: Clone downed apply (receive)

**Files:**
- Modify: `src/Coop/CoopMortality.cpp`

- [ ] **Step 1: Queue received downed packets (ENet thread → game thread)**

Add a thread-safe pending buffer + the receiver:
```cpp
    struct PendingDowned { uint8_t downed; float x, y, z; };
    static std::vector<PendingDowned> g_pendingDowned;
    static std::mutex                 g_pendingDownedMutex;

    void OnDownedReceived(const std::uint8_t* a_data, std::size_t a_length)
    {
        if (a_length < 1 + sizeof(Network::PlayerDownedData)) return;
        Network::PlayerDownedData d{};
        std::memcpy(&d, a_data + 1, sizeof(d));
        std::lock_guard lk(g_pendingDownedMutex);
        g_pendingDowned.push_back({ d.downed, d.x, d.y, d.z });
    }
```
Add `#include <vector>`, `#include <mutex>`, `#include <cstring>` and `#include "RemotePlayer/RemotePlayerManager.h"` to the file.

- [ ] **Step 2: Apply the clone pose on the game thread**

In `OnGameThreadTick`, after the local-detection block, drain the queue and pose the remote clone. Use the RE'd downed pose from Task 2 — default approach: drive the clone into a bleedout via `SafeSetEssential(clone,true)` + force its health to 0 (collapse) for downed, and `SafeReviveActor(clone)` for stand-up:
```cpp
        std::vector<PendingDowned> work;
        { std::lock_guard lk(g_pendingDownedMutex); work.swap(g_pendingDowned); }
        if (!work.empty()) {
            auto* clone = Pulsar::RemotePlayer::GetRemotePlayerManager().GetRemoteActor();
            for (auto& w : work) {
                if (!clone) { REX::INFO("[DOWNED] recv downed={} but no clone yet"sv, (int)w.downed); continue; }
                if (w.downed) {
                    SafeSetEssential(clone, true);
                    __try {
                        auto* av = HealthAV();
                        if (av) clone->AsActorValueOwner()->SetActorValue(*av, 0.0f);  // collapse pose
                    } __except (EXCEPTION_EXECUTE_HANDLER) {}
                    REX::INFO("[DOWNED] clone posed downed at %.0f,%.0f,%.0f"sv, w.x, w.y, w.z);
                } else {
                    SafeReviveActor(clone);
                    REX::INFO("[DOWNED] clone revived/stood up"sv);
                }
            }
        }
```
> The clone-pose mechanism is the most RE-sensitive piece. If forcing health-to-0 doesn't reliably collapse the clone (clones have AI suppressed — see `feedback_coop_no_combat_clone`), Task 2 Step 4 should have established a ragdoll/down-anim path (e.g. the `SafeKillActor`-style ragdoll used for dead NPCs in `LocomotionApply.cpp` TickLerp, but reversible). Slot that in here.

- [ ] **Step 3: Build, deploy, test the mirror**

Build + deploy + restart both. In co-op, take Player A to 0 HP → **expected:** Player B sees A's clone collapse/downed at A's position; B's log shows `[DOWNED] clone posed downed`. When A is revived, B sees the clone stand (`clone revived/stood up`).

- [ ] **Step 4: Commit**
```
git add src/Coop/CoopMortality.cpp
git commit -m "feat(coop): mirror downed/revive onto remote clone "
```

---

## Task 6: Revive interaction (activate downed clone → revive owner)

**Files:**
- Modify: `src/Coop/CoopMortality.cpp`

- [ ] **Step 1: Track whether the remote clone is currently downed**

Add a static set/flag updated in the Task-5 apply loop:
```cpp
    static bool g_remoteCloneDowned = false;   // set true when we pose the clone downed, false on revive
```
Set it in the Task-5 loop: `g_remoteCloneDowned = (w.downed != 0);`

- [ ] **Step 2: Detect the local player activating the downed clone**

In `OnGameThreadTick`, when `g_remoteCloneDowned` is true, check proximity + an activate keypress (reuse the project's activate-detection if available; otherwise a hold-key in range). Minimal proximity + key approach:
```cpp
        if (g_remoteCloneDowned) {
            auto* clone = Pulsar::RemotePlayer::GetRemotePlayerManager().GetRemoteActor();
            if (clone) {
                float dist2 = 1e9f;
                __try {
                    auto pp = player->GetPosition(); auto cp = clone->GetPosition();
                    float dx = pp.x-cp.x, dy = pp.y-cp.y, dz = pp.z-cp.z;
                    dist2 = dx*dx + dy*dy + dz*dz;
                } __except (EXCEPTION_EXECUTE_HANDLER) {}
                static bool s_revKeyDown = false;
                bool dn = (GetAsyncKeyState(0x45) & 0x8000) != 0; // VK_E (activate)
                if (dist2 < (200.0f*200.0f) && dn && !s_revKeyDown) {
                    s_revKeyDown = true;
                    Network::PlayerReviveData r{}; r.reserved = 0;
                    uint8_t buf[1 + sizeof(r)]; buf[0] = Network::PKT_PLAYER_REVIVE;
                    std::memcpy(buf + 1, &r, sizeof(r));
                    Pulsar::Network::GetSingleton().Send(buf, sizeof(buf), true);
                    REX::INFO("[DOWNED] local activated downed clone -> sent REVIVE"sv);
                } else if (!dn) { s_revKeyDown = false; }
            }
        }
```
> If the project already has an activate-on-ref hook (`memory/reference_activate_re.md`, REL::ID 63380) that fires when the player presses E on a specific REFR, prefer hooking that and filtering for the clone's FormID — cleaner than a raw key+distance check. Note this in execution.

- [ ] **Step 3: Handle the revive on the downed player's own machine**

Implement the receiver (the downed player receives the instruction and revives THEMSELVES — own-machine authority):
```cpp
    void OnReviveReceived(const std::uint8_t* a_data, std::size_t a_length)
    {
        if (a_length < 1 + sizeof(Network::PlayerReviveData)) return;
        // Queue a self-revive flag; applied on the game thread.
        std::lock_guard lk(g_pendingDownedMutex);
        g_selfReviveRequested = true;
    }
```
Add `static bool g_selfReviveRequested = false;` and, in the tick (game thread), apply it:
```cpp
        bool reviveMe = false;
        { std::lock_guard lk(g_pendingDownedMutex); reviveMe = g_selfReviveRequested; g_selfReviveRequested = false; }
        if (reviveMe && g_localDowned) {
            SafeReviveActor(player);
            g_localDowned = false;
            SendDowned(false, player);   // broadcast the cleared state so the partner's clone stands
            REX::INFO("[DOWNED] self-revived on partner's request"sv);
        }
```

- [ ] **Step 4: Build, deploy, full revive loop test**

Build + deploy + restart both. Co-op: A goes down → B walks to A's downed clone, presses E in range → **expected:** A is revived on A's machine (`self-revived`), A stands with partial HP; B's clone of A stands (`clone revived/stood up`). The full down→revive loop works end to end.

- [ ] **Step 5: Commit**
```
git add src/Coop/CoopMortality.cpp
git commit -m "feat(coop): revive interaction — activate downed clone revives owner "
```

---

## Task 7: Both-downed → real game-over

**Files:**
- Modify: `src/Coop/CoopMortality.cpp`

- [ ] **Step 1: Track the remote downed state for the both-downed check**

`g_remoteCloneDowned` (Task 6) already tracks the partner's downed state, and `g_localDowned` tracks ours. Add the both-downed handler in the tick:
```cpp
        if (g_localDowned && g_remoteCloneDowned) {
            // Both players are down at once -> real game-over. Clear essential so the
            // engine's vanilla death/game-over proceeds (-> both reload -> rejoin via
            // the existing save-bridge). One-shot.
            static bool s_gameOverFired = false;
            if (!s_gameOverFired) {
                s_gameOverFired = true;
                SafeSetEssential(player, false);
                __try {
                    auto* av = HealthAV();
                    if (av) player->AsActorValueOwner()->SetActorValue(*av, 0.0f);
                } __except (EXCEPTION_EXECUTE_HANDLER) {}
                REX::INFO("[DOWNED] BOTH downed -> clearing essential, real game-over"sv);
            }
        }
```
> Resetting `s_gameOverFired` on session reconnect is handled by module reload (DLL is per-launch). If a `Reset()` path is added later, clear it there.

- [ ] **Step 2: Build, deploy, both-downed test**

Build + deploy + restart both. Co-op: take BOTH players down close together → **expected:** vanilla game-over/reload fires for the local player (and symmetrically on the peer, since each machine sees both downed); both reload and rejoin via the existing save-bridge. Log shows `[DOWNED] BOTH downed -> ... real game-over`.

- [ ] **Step 3: Commit + update memory**
```
git add src/Coop/CoopMortality.cpp
git commit -m "feat(coop): both-downed -> real game-over "
```
Update `memory/project_coop_downed_revive.md` to SHIPPED with the commit hashes, the confirmed essential/bleedout mechanism, and the revive primitive RVA/method. Mark Task complete. Note Feature B (clone aggro-draw) as the next follow-up spec.

---

## Self-review (against the spec `docs/design/specs/2026-05-25-coop-downed-revive-design.md`)

- **Death intercept (set kEssential):** Task 1 (probe) + Task 4 (permanent set in tick). ✓
- **Downed detection (edge-triggered):** Task 4 Step 2. ✓
- **Downed sync (send):** Task 4 (`SendDowned`). ✓
- **Clone downed apply (recv):** Task 5. ✓
- **Revive interaction:** Task 6. ✓
- **Both-downed handler:** Task 7. ✓
- **Packets PKT_PLAYER_DOWNED/REVIVE (0x97/0x98):** Task 3. ✓
- **Own-machine authority / no cross-machine player damage:** detection reads only the LOCAL player; clone is posed only from the peer's own broadcast — no host-side player-damage forwarding. ✓ (extends `feedback_coop_no_combat_clone`.)
- **RE discovery first (gate):** Task 1 + Task 2, with explicit fallbacks. ✓
- **Out of scope honored:** no Feature B (aggro-draw), no continuous HP-bar sync, no downed-polish (revive bar/timer/prompts). ✓

**Type/name consistency:** `SafeSetEssential`, `SafeReviveActor`, `HealthAV()`, `IsLocalDowned`, `SendDowned`, `g_localDowned`, `g_remoteCloneDowned`, `g_pendingDowned`/`g_pendingDownedMutex`, `g_selfReviveRequested`, `PlayerDownedData`(13B)/`PlayerReviveData`(1B), `PKT_PLAYER_DOWNED=0x97`/`PKT_PLAYER_REVIVE=0x98` — used consistently across tasks.

**Known RE-dependent values (produced in Task 2, consumed later):** the exact bleedout-state read bit (Task 4 `IsLocalDowned`), the clear-bleedout call inside `SafeReviveActor` (Task 5/6), and the clone-collapse-vs-ragdoll pose mechanism (Task 5). These are discovered constants, not placeholders — Task 2 is responsible for resolving them, exactly as the Combat-Sync kill primitive was.
