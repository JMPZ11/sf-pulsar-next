# Client Progression Stage 1 (Skills + Perks) — Implementation Plan


**Goal:** A joining client keeps their own perks (which include skill ranks) across a co-op session — captured before the host-save bridge, re-applied additively after it.

**Architecture:** Extend the proven `DirectRestore` flow (the authoritative item/O2 restore) with a perk snapshot captured pre-bridge (in `CaptureClientState`) and re-applied post-bridge (in `RestoreClientState`), mirroring the existing O2-ActorValue capture/apply pattern. Same machine ⇒ raw FormIDs are authoritative; the client's own save on disk is the cross-session store. No STDB schema change, no new path.

**Tech Stack:** C++ SFSE plugin (CommonLibSF), Ghidra (RVA verification), ENet co-op bridge.

**Reference spec:** `docs/design/specs/2026-06-17-client-progression-perks-design.md`

**Scope note (refined during planning):** Stage 1 = the **same-session join restore** (DirectRestore). It fully delivers "client doesn't lose perks when joining" on the same machine (the client's own save keeps their perks across sessions). **Cloud-blob perk persistence is DEFERRED to Stage 1b** because (a) it only adds value cross-machine, which needs accounts/OIDC anyway, and (b) the auto-save  runs **post-bridge** on disconnect, so naively capturing perks there would store the *host's* perks — Stage 1b must feed the cloud blob from DirectRestore's pre-bridge snapshot, not a fresh read.

---

## File Structure
- Modify: `src/InventorySync/DirectRestore.cpp` — perk snapshot globals, capture in `CaptureClientState`, additive apply in `RestoreClientState`, perk diff logging in `TickPostRestoreWatch`, `kApplyPerksOnRestore` flag, verified `AddPerk` relocation. (Single file — the whole feature lives where the item restore lives.)
- Reference only (no change): `extern/CommonLibSF/include/RE/A/Actor.h` (`perks`@0x360, `perkArrayLock`@0x36C, `Perks{BSTArray<PerkRankData>* perkRanks}`), `extern/CommonLibSF/include/RE/P/PerkRankData.h` (`{BGSPerk* perk@0; int8 currentRank@8}`).

---

## Task 1: RE — verify `Actor::AddPerk` RVA + perk layout (GATING)

**Files:** none (investigation; produces verified constants used by Task 3)

This task is the gate: Task 3 cannot apply perks safely without a Ghidra-verified `AddPerk`. If it can't be verified, Task 3 falls back to capture-only (no apply) and the feature ships as "perks captured + logged, apply deferred".

- [ ] **Step 1: Find the AddPerk candidate RVA from the version DB**

The Papyrus native `Actor.AddPerk` exists (`BSScript__NativeFunction*_Actor_void_BGSPerk_bool_`). Use the project's version-lib reader to list candidates and the Ghidra project (NOT the raw exe — see memory `feedback_use_ghidra_project`) to confirm. Run the existing helper:

Run: `python read_versionlib.py` (workflow per `reference_ghidra_workflow`) and locate the engine function the `Actor.AddPerk` native binds to. Note the REL::ID and RVA.
Expected: a single function id whose decompile takes `(Actor* this, BGSPerk* perk, int rank)`-shaped args and writes into the actor's perk array.

- [ ] **Step 2: Verify the function signature + semantics in Ghidra**

Decompile the candidate (`DecompileAt.java` against the Ghidra project). Confirm:
- It receives the actor + a `BGSPerk*` + an integer rank.
- Whether rank is **absolute** (sets to rank) or **incremental** (adds one rank). Record which — Task 3's apply depends on it.
- It updates `perks->perkRanks` and applies perk entry-point effects (not just array insert).

Expected: signature + semantics documented in the task notes (e.g. "REL::ID(NNNNN), absolute rank, applies entries").

- [ ] **Step 3: Confirm the live-player perk layout**

Cross-check against CommonLibSF (already read): `Actor::perks` at +0x360, `perkArrayLock` at +0x36C, `Perks::perkRanks` is `BSTArray<PerkRankData>*`, `PerkRankData{BGSPerk* perk@0; int8 currentRank@8}`. Confirm these offsets match the Ghidra struct for 1.16.236 (CLibSF offsets are routinely stale — verify, per `feedback_commonlibsf_verify`).
Expected: offsets confirmed OR corrected values recorded.

- [ ] **Step 4: Record results**

Write the verified `AddPerk` REL::ID, its rank semantics, and confirmed offsets into a short note in the reference memory (`reference_*` ghidra note) and at the top of the Task 3 code comment. No commit (no code yet).

---

## Task 2: Perk snapshot type + pre-bridge capture

**Files:**
- Modify: `src/InventorySync/DirectRestore.cpp` (anonymous-namespace globals near line 80; `CaptureClientState` end, near line 530)

- [ ] **Step 1: Add the perk snapshot globals**

In the anonymous namespace (after `g_savedO2Max`, ~line 80):

```cpp
        // Stage-1 progression — perks captured PRE-bridge (the client's own),
        // re-applied additively post-bridge. Mirrors g_savedO2Max: only written
        // while captures are unlocked (normal play), so it survives the bridge.
        struct PerkSnap { std::uint32_t formID = 0; std::int8_t rank = 0; };
        std::vector<PerkSnap> g_perkSnap;
        std::mutex            g_perkMu;
```

- [ ] **Step 2: Capture perks at the end of `CaptureClientState`**

Inside `CaptureClientState`, after the O2 snapshot block (before the closing brace ~line 531), add a SEH-guarded walk of the live perk array (capture is skipped while the cache is locked, so this only records the client's own pre-bridge perks):

```cpp
        // Capture the client's own perks (pre-bridge). SEH-guarded like the
        // inventory walk; the player->perks array is read under no assumption
        // it's non-null. Stored only when unlocked (this function already
        // early-returns when g_cacheLocked), so it holds across the bridge.
        {
            std::vector<PerkSnap> perks;
            __try {
                auto* pk = player->perks;
                if (pk && pk->perkRanks) {
                    for (const auto& prd : *pk->perkRanks) {
                        if (prd.perk) perks.push_back({ prd.perk->formID, prd.currentRank });
                    }
                }
            } __except (EXCEPTION_EXECUTE_HANDLER) {
                perks.clear();
            }
            if (!perks.empty()) {
                std::lock_guard lk(g_perkMu);
                g_perkSnap = std::move(perks);
            }
            REX::INFO("[PULSAR] [DirectRestore] perk capture: {} perks"sv,
                      []{ std::lock_guard lk(g_perkMu); return g_perkSnap.size(); }());
        }
```

Add the include for the perk types at the top with the other RE includes (near line 13):

```cpp
#include <RE/B/BGSPerk.h>
```

- [ ] **Step 3: Build the DLL**

Run: `"/c/Program Files/CMake/bin/cmake" --build build/vs --config RelWithDebInfo --target PULSAR`
Expected: `PULSAR.vcxproj -> ...\PULSAR.dll`, no errors. (Capture-only at this point — safe to ship even before apply.)

- [ ] **Step 4: Commit**

```bash
git add src/InventorySync/DirectRestore.cpp
git commit -m "feat(progression): capture client perks pre-bridge (DirectRestore)"
```

---

## Task 3: Additive post-bridge perk apply

**Files:**
- Modify: `src/InventorySync/DirectRestore.cpp` (apply relocation near the `DirectEquipForm` block ~line 86; apply call inside `RestoreClientState` after the inventory/O2 restore; `kApplyPerksOnRestore` flag)

> Uses the verified `AddPerk` REL::ID + rank semantics from Task 1. The code below assumes **absolute** rank; if Task 1 found it's incremental, loop the call `rank` times instead of passing the rank. If Task 1 could NOT verify the RVA safely, SKIP this task's apply call — leave `kApplyPerksOnRestore = false` and ship capture-only.

- [ ] **Step 1: Add the flag + the verified AddPerk relocation**

Near the top of the namespace (with the other `REL::Relocation` helpers, ~line 86), using the **Task 1 verified id** (placeholder `AVERIFIED_ID` must be replaced with the real value from Task 1):

```cpp
        // Stage-1 — apply perks additively post-bridge. Kill-switch mirrors the
        // ApplyProfile kApply* flags. REL::ID + rank-semantics verified in Ghidra
        // (Task 1: see reference note). Absolute-rank signature assumed below.
        constexpr bool kApplyPerksOnRestore = true;
        using AddPerkFn = void(*)(RE::Actor*, RE::BGSPerk*, std::int32_t);
        static REL::Relocation<AddPerkFn> s_addPerk{ REL::ID(/*AVERIFIED_ID*/ 0) };
```

- [ ] **Step 2: Add an additive perk-apply helper**

After `RestoreClientState`'s helpers (or just above `RestoreClientState`), add:

```cpp
        // Returns the player's current rank for a perk form, or -1 if absent.
        static std::int32_t LivePerkRank(RE::PlayerCharacter* a_player, std::uint32_t a_formID)
        {
            std::int32_t rank = -1;
            __try {
                auto* pk = a_player->perks;
                if (pk && pk->perkRanks) {
                    for (const auto& prd : *pk->perkRanks) {
                        if (prd.perk && prd.perk->formID == a_formID) { rank = prd.currentRank; break; }
                    }
                }
            } __except (EXCEPTION_EXECUTE_HANDLER) { rank = -1; }
            return rank;
        }

        // SEH-isolated call (C2712: __try cannot share a frame with C++ unwinding
        // objects — the same trap Task 2 hit; ApplyPerksAdditive has a vector).
        static bool SafeAddPerk(AddPerkFn a_fn, RE::Actor* a_actor, RE::BGSPerk* a_perk, std::int32_t a_rank)
        {
            __try { a_fn(a_actor, a_perk, a_rank); return true; }
            __except (EXCEPTION_EXECUTE_HANDLER) { return false; }
        }

        // Additive: add only the client's perks the live player lacks (or has at
        // a lower rank). Never removes host perks (Stage 1). Same-machine raw
        // FormID; a form that doesn't resolve here is skipped (logged).
        static void ApplyPerksAdditive(RE::PlayerCharacter* a_player)
        {
            if (!kApplyPerksOnRestore || s_addPerk.address() == 0) return;
            std::vector<PerkSnap> snap;
            { std::lock_guard lk(g_perkMu); snap = g_perkSnap; }
            if (snap.empty()) return;

            int applied = 0, skipped = 0, missing = 0;
            for (const auto& ps : snap) {
                auto* form = RE::TESForm::LookupByID(ps.formID);
                if (!form || form->GetFormType() != RE::FormType::kPERK) { ++missing; continue; }
                auto* perk = static_cast<RE::BGSPerk*>(form);
                const std::int32_t live = LivePerkRank(a_player, ps.formID);
                if (live >= ps.rank) { ++skipped; continue; }
                if (SafeAddPerk(*s_addPerk, a_player, perk, ps.rank)) ++applied;
                else REX::WARN("[PULSAR] [DirectRestore] AddPerk faulted form=0x{:08X}"sv, ps.formID);
            }
            REX::INFO("[PULSAR] [DirectRestore] perk apply: +{} applied, {} already-had, {} unresolved"sv,
                      applied, skipped, missing);
        }
```

Add the FormType include near the top includes (~line 13):

```cpp
#include <RE/F/FormTypes.h>
```

- [ ] **Step 3: Call the apply at the end of `RestoreClientState`**

At the end of `RestoreClientState` (after the inventory re-equip / O2 re-apply, before the function returns), add:

```cpp
        ApplyPerksAdditive(player);
```

- [ ] **Step 4: Log the post-restore perk count in `TickPostRestoreWatch`**

In `TickPostRestoreWatch` (~line 917), where it logs the +5s/+10s inventory diff, also log the live perk count so the in-game test can compare pre-bridge vs post-restore:

```cpp
            // Stage-1 progression verification — live perk count.
            if (auto* pc = RE::PlayerCharacter::GetSingleton()) {
                int n = 0;
                __try { if (pc->perks && pc->perks->perkRanks) n = (int)pc->perks->perkRanks->size(); }
                __except (EXCEPTION_EXECUTE_HANDLER) { n = -1; }
                REX::INFO("[PULSAR] [DirectRestore] perk watch: live perks={}"sv, n);
            }
```

- [ ] **Step 5: Build the DLL**

Run: `"/c/Program Files/CMake/bin/cmake" --build build/vs --config RelWithDebInfo --target PULSAR`
Expected: builds clean, no errors.

- [ ] **Step 6: Commit**

```bash
git add src/InventorySync/DirectRestore.cpp
git commit -m "feat(progression): additive post-bridge perk apply (verified AddPerk)"
```

---

## Task 4: Deploy + in-game verification

**Files:** none (build/deploy/test)

- [ ] **Step 1: Deploy the DLL (local + NAS)**

```bash
cp -f build/vs/RelWithDebInfo/PULSAR.dll /d/Games/Starfield/Data/SFSE/Plugins/PULSAR.dll
cp -f build/vs/RelWithDebInfo/PULSAR.dll <NAS>/deploy/Data/SFSE/Plugins/PULSAR.dll
```
(Forward-slash UNC per the Bash-UNC trap in `feedback_deploy`.)

- [ ] **Step 2: In-game test — perks survive the join (user)**

Host + client (client has known perks the host lacks — e.g. a few skill ranks). Client joins. In the client log (`...My Games\Starfield\SFSE\Logs\PULSAR.log`) confirm, in order:
- `[DirectRestore] perk capture: N perks` (pre-bridge, N = client's perk count)
- `[DirectRestore] perk apply: +X applied, Y already-had, Z unresolved`
- `[DirectRestore] perk watch: live perks=M` at +5s/+10s, with M ≥ the client's pre-bridge N.
Expected: client retains their own perks after the bridge; no crash. Verify the specific known perks are present in-game (skill menu).

- [ ] **Step 3: No-regression check**

Same session: confirm inventory + O2 still restore correctly (existing `[DirectRestore]` cached/restored logs unchanged; items present, O2 bar full). Expected: no change to item/O2 behavior.

- [ ] **Step 4: If apply destabilizes — fall back**

If AddPerk crashes or misbehaves in-game: set `kApplyPerksOnRestore = false` (Task 3 Step 1), rebuild, redeploy → ships capture-only (perks logged, not applied) while the apply is re-RE'd. Commit the flag flip with a note.

---

## Self-Review

**Spec coverage:**
- "Keep own perks across join" → Tasks 2 (capture pre-bridge) + 3 (additive apply) ✓.
- "Extend DirectRestore, not a new path" → all changes in DirectRestore.cpp ✓.
- "Skills = perks (skill ranks)" → captured via the perk array ✓.
- "Additive" → `ApplyPerksAdditive` only adds, never removes ✓.
- "Ghidra-verify AddPerk" → Task 1 gates Task 3 ✓.
- "kApplyPerksOnRestore kill-switch + capture-only fallback" → Task 3 Step 1 + Task 4 Step 4 ✓.
- "No STDB schema change / no --delete-data" → no module/companion change ✓.
- Cloud-blob persistence → explicitly deferred to Stage 1b with the post-bridge-timing rationale ✓ (scope note).

**Placeholder scan:** The only placeholder is `AVERIFIED_ID` in Task 3 Step 1 — intentional, filled by Task 1's verified RVA (Task 3 is explicitly gated on Task 1). All other steps have complete code + exact build/deploy commands.

**Type consistency:** `PerkSnap{formID,rank}`, `g_perkSnap`/`g_perkMu`, `kApplyPerksOnRestore`, `s_addPerk`, `LivePerkRank`, `ApplyPerksAdditive` used consistently across Tasks 2–3. Offsets/fields (`player->perks->perkRanks`, `PerkRankData.perk/.currentRank`) match the verified CommonLibSF layout and Task 1's confirmation.

## Risks
- **AddPerk RVA safety** — gated by Task 1; capture-only fallback if unverifiable.
- **Live-apply stability** — perks are stat modifiers (lower risk than equip/anim per project history); SEH-guarded per-perk; kill-switch.
- **Rank semantics** (absolute vs incremental) — resolved in Task 1; Task 3 code note covers both.
- **FormID resolution** — same-machine raw FormID; unresolved forms skipped+logged (cross-machine handled later with the T34 plugin map / OIDC).
