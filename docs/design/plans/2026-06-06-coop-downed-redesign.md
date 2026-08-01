# Co-op Downed & Revive — Keep-Alive Redesign Implementation Plan


**Goal:** Make a co-op player who would die enter a stable, frozen, *alive* downed state (no ragdoll, no fall-through) that recovers on combat-end / F10, with both-down → vanilla reload.

**Architecture:** Never let the player die. Hook the life-state setter (REL::ID 100787) to swallow the engine's player death-set so the life-state stays `alive` (0); on the game thread clamp HP > 0 (engine stops retrying) and lock player controls (frozen). Revive = unfreeze + restore HP. Both-down = force real death → vanilla reload (V13m trap already neutralized in StateProbe).

**Tech Stack:** C++23, SFSE, CommonLibSF, MinHook. Single module `src/Coop/CoopMortality.cpp`. No unit-test framework for engine behavior — verification is "compiles" + the in-game manual test (Task 7).

**Spec:** `docs/design/specs/2026-06-06-coop-downed-redesign-design.md`

**Build:** `cmake --build <repo>\build\vs --config RelWithDebInfo --target PULSAR`
**Deploy:** copy `build\vs\RelWithDebInfo\PULSAR.dll`+`.pdb` → `<Starfield>\Data\SFSE\Plugins`, and `PULSAR.dll` → `\\<NAS>\PULSAR\deploy\Data\SFSE\Plugins`, then restart BOTH games.
**Commit msgs:** end with . Write the message to a temp file with `-Encoding ascii` and `git commit -F` (PS 5.1 `utf8` adds a BOM). Never `git add -A` — stage only the named files; `extern/CommonLibSF` must stay unstaged.

**Starting point:** HEAD `0484cbf` (the broken bleedout build). This plan rips out the bleedout/Resurrect/death-mask code and replaces it with the keep-alive model.

---

## File Structure

- **`src/Coop/CoopMortality.cpp`** — the only file changed. Holds: the life-state setter hook (entry), the control-lock helper, the game-thread tick (HP clamp + control-lock edge + Way B + F10 + both-down), `ReviveLocalPlayer`, net-sync glue.
- No new files. `PKT_PLAYER_DOWNED 0x97` / `PlayerDownedData` already exist in `include/Network/PacketTypes.h` and are dispatched in `src/Network/NetworkHost.cpp` — unchanged.

---

## Task 1: RE — player control lock/unlock helper

**Files:**
- Modify: `src/Coop/CoopMortality.cpp` (add `SafeSetPlayerControlsEnabled`)
- Reference: `<Starfield>\Data\SFSE\Plugins\versionlib-1-16-236-0.bin`

**Why:** CommonLibSF exposes no control-disable; PULSAR has none. This is the one open RE. Frozen = no movement/attack/activate while downed.

- [ ] **Step 1: Local-first lookup (no agent yet).** Find the engine function behind the Papyrus `Game.DisablePlayerControls` / `EnablePlayerControls` natives. Run, in order:
  - `Grep` CommonLibSF `IDs.h`/`IDs_RTTI.h` for `Controls`, `UserEvents`, `ControlMap`.
  - `python local RE notes "<Starfield>\Data\SFSE\Plugins\versionlib-1-16-236-0.bin" <id>` for any candidate IDs found.
  - In the Ghidra project, string-search `DisablePlayerControls` / `EnablePlayerControls` and follow the Papyrus native handler to the engine function (same discovery-trail pattern documented in `reference_activate_re.md`).
  - Only if local tools cannot resolve it, dispatch **ONE** Ghidra agent (never on a guessed address; do the versionlib lookup first). See the token-budget rule.

- [ ] **Step 2: Add the helper (resolved RVA path).** Once the function `void __fastcall DisablePlayerControls(args...)` / its enable counterpart is identified, add near the other `Safe*` helpers in `CoopMortality.cpp`:

```cpp
// Resolved in InstallHooks(): the engine's player-controls toggle (see Task 1 RE).
using SetControlsFn = void(__fastcall*)(/* exact ABI from RE */);
static SetControlsFn s_disablePlayerControls = nullptr;
static SetControlsFn s_enablePlayerControls  = nullptr;

// SEH: freeze (a_enabled=false) / unfreeze (true) the local player's controls.
// Returns 1 on success, 0 if SEH caught (then the caller logs and we are no worse
// than before). MUST run on the game thread.
static int SafeSetPlayerControlsEnabled(bool a_enabled)
{
    __try {
        // Exact call shape filled from the RE in Step 1, e.g. disable movement +
        // fighting + activate + (camera left ON so the downed player can look).
        if (a_enabled) { if (s_enablePlayerControls)  s_enablePlayerControls(/*...*/); }
        else           { if (s_disablePlayerControls) s_disablePlayerControls(/*...*/); }
        return 1;
    } __except (EXCEPTION_EXECUTE_HANDLER) { return 0; }
}
```

- [ ] **Step 3: Resolve in InstallHooks().** Add to `InstallHooks()` (after the setter hook): resolve `s_disablePlayerControls`/`s_enablePlayerControls` from their RVAs and `REX::INFO` whether each is non-null.

- [ ] **Step 4: FALLBACK if RE not found within a reasonable local effort.** Implement `SafeSetPlayerControlsEnabled` as a stopgap with NO new RE: on disable, `SetActorValue(SpeedMult, 0)` (immobilize) and force the weapon sheathed; on enable, restore `SpeedMult` to its prior value. Log `[DOWNED] control-lock FALLBACK (SpeedMult)`. Document in the function comment that this is an MVP stopgap (does not block camera/attack) pending the proper control RE.

- [ ] **Step 5: Build.**

Run: `cmake --build <repo>\build\vs --config RelWithDebInfo --target PULSAR`
Expected: compiles; log lines for the resolved/fallback control path.

- [ ] **Step 6: Commit.**

```
git add src/Coop/CoopMortality.cpp
git commit -F <ascii msg file>   # "feat(coop): player control-lock helper for downed "
```

---

## Task 2: Life-state setter hook → keep-alive (swallow death, NO bleedout)

**Files:**
- Modify: `src/Coop/CoopMortality.cpp` — `Hooked_SetLifeState`

**Why:** Entry trigger. Keep the player `alive` by swallowing the engine's death-set instead of redirecting to bleedout (bleedout flips → through floor, proven dead-end).

- [ ] **Step 1: Replace the hook body.** Replace the current `Hooked_SetLifeState` (the bleedout-redirect version) with:

```cpp
static void __fastcall Hooked_SetLifeState(RE::Actor* a_this, std::uint32_t a_newState)
{
    if (Pulsar::Network::GetSingleton().IsConnected() && SafeIsLocalPlayer(a_this)) {
        const bool isDeath    = (a_newState == kDeathEnum1 || a_newState == kDeathEnum2);
        const bool bothDowned = g_remoteDowned.load(std::memory_order_relaxed);
        if (isDeath && !bothDowned) {
            // KEEP-ALIVE: swallow the engine's player death-set entirely. We do NOT
            // call orig, so the life-state is left untouched (stays alive / 0).
            // No bleedout, no death, no ragdoll. Game-thread tick then clamps HP > 0
            // (engine stops retrying) and locks controls (frozen). First edge only:
            // flag + notify the peer.
            if (!g_localDowned.exchange(true)) {
                SendDownedState(true);
                REX::INFO("[DOWNED] entered downed (keep-alive; death-set swallowed)"sv);
            }
            return;  // swallow
        }
        // bothDowned: let death through (falls to orig below) -> real death -> reload.
    }
    if (s_origSetLifeState) s_origSetLifeState(a_this, a_newState);
}
```

- [ ] **Step 2: Build.** Run the build command. Expected: compiles.
- [ ] **Step 3: Commit.** `"fix(coop): keep-alive entry - swallow player death-set, no bleedout "`

---

## Task 3: Game-thread — control-lock on downed edge + HP clamp

**Files:**
- Modify: `src/Coop/CoopMortality.cpp` — `OnGameThreadTick`

**Why:** The setter hook runs on worker threads; `ModActorValue` and the control-lock must run on the game thread. Lock controls once on the false→true transition; clamp HP every frame while downed.

- [ ] **Step 1: Add an "entry applied" latch + the block.** Near the top of `OnGameThreadTick`'s downed handling, add (uses the existing `player`, `av`, `health` locals):

```cpp
// Apply the downed side-effects on the game thread exactly once per entry:
// lock controls (frozen). The setter hook (worker thread) set g_localDowned.
static bool s_controlsLocked = false;
if (g_localDowned && !s_controlsLocked) {
    s_controlsLocked = true;
    SafeSetPlayerControlsEnabled(false);
    REX::INFO("[DOWNED] controls locked (frozen)"sv);
}
if (!g_localDowned && s_controlsLocked) {
    s_controlsLocked = false;  // revive path already re-enabled controls; just clear the latch
}

// HP clamp: while downed keep Health > 0 so the engine stops issuing death-sets.
if (g_localDowned && health < kDownedFloor) {
    SafeRaiseHealthTo(player, av, kDownedFloor);
}
```

- [ ] **Step 2: Re-enable the clamp region.** Remove the "HP-floor clamp DISABLED" comment block left from the bleedout attempt (the clamp now lives in Step 1).

- [ ] **Step 3: Build.** Expected: compiles.
- [ ] **Step 4: Commit.** `"feat(coop): lock controls on downed edge + HP clamp on game thread "`

---

## Task 4: `ReviveLocalPlayer` → unfreeze + restore HP (drop life-state/Resurrect)

**Files:**
- Modify: `src/Coop/CoopMortality.cpp` — `ReviveLocalPlayer`

**Why:** The player was never dead (kept alive), so revive needs no Resurrect / life-state change — just restore HP and re-enable controls.

- [ ] **Step 1: Replace the body.** Replace the current `ReviveLocalPlayer` body with:

```cpp
static void ReviveLocalPlayer(RE::PlayerCharacter* a_player,
                              RE::ActorValueInfo*  a_av,
                              float                a_hpFraction,
                              const char*          a_reason)
{
    if (!a_player || !a_av) return;

    g_localDowned = false;
    SendDownedState(false);

    // Re-enable controls (we froze them on entry). Player was kept alive the whole
    // time -> no life-state change / Resurrect needed.
    SafeSetPlayerControlsEnabled(true);

    float maxHealth = 0.0f;
    float restoreHP = kDownedFloor;
    const bool gotMax = (SafeReadMaxHealth(a_player, a_av, maxHealth) && maxHealth > 0.0f);
    if (gotMax) restoreHP = maxHealth * a_hpFraction;
    SafeRaiseHealthTo(a_player, a_av, restoreHP);

    REX::INFO("[DOWNED] revive ({}) -> health={:.1f}{}"sv,
              a_reason, restoreHP, gotMax ? "" : " (max-read failed)");
}
```

- [ ] **Step 2: Build.** Expected: compiles.
- [ ] **Step 3: Commit.** `"fix(coop): revive = unfreeze + restore HP, no life-state change "`

---

## Task 5: Both-down → game-over (force death + reload)

**Files:**
- Modify: `src/Coop/CoopMortality.cpp` — `OnGameThreadTick` (both-down block) and/or the retained death-tick hook

**Why:** With keep-alive, the player never reaches death, so the reload can't arm. When both are down we must force real death so the vanilla reload fires (V13m StateProbe neutralizes the relaunch trap).

- [ ] **Step 1: Add the both-down block** in `OnGameThreadTick` (after the entry/clamp block):

```cpp
// Both players downed -> controlled game-over: stop keeping ourselves alive, force
// real death via the original setter (bypasses the keep-alive swallow), and clear
// the engine load counter every frame so the imminent reload avoids the relaunch+
// int3 trap (V13m). The setter hook's !bothDowned condition also lets death through.
if (g_localDowned && g_remoteDowned.load(std::memory_order_relaxed)) {
    SafeClearLoadCounter();
    SafeSetLifeStateRaw(player, kDeathEnum1);  // orig setter -> real death -> reload arms
    static bool s_loggedBoth = false;
    if (!s_loggedBoth) { s_loggedBoth = true; REX::INFO("[DOWNED] BOTH downed -> forcing death for game-over reload"sv); }
}
```

- [ ] **Step 2: Decide the death-tick hook's fate.** With keep-alive, single-down never reaches death bits, so the death-tick masking is dormant. Keep the hook installed (REL::ID 102812) as a safety net (it still calls orig plain for the forced both-down death, letting the reload arm) but remove its now-unreachable single-down entry/masking branches if they add confusion. Minimal change: leave the hook as-is if it still compiles and the both-down forced death reaches the vanilla reload; do NOT spend effort here unless the both-down test fails.

- [ ] **Step 3: Build.** Expected: compiles.
- [ ] **Step 4: Commit.** `"feat(coop): both-down forces death for vanilla reload under keep-alive "`

---

## Task 6: Remove dead bleedout/Resurrect code

**Files:**
- Modify: `src/Coop/CoopMortality.cpp`

**Why:** DRY/clarity — the bleedout pivot left unused code that now warns (C4505) and confuses.

- [ ] **Step 1: Delete unused symbols.** Remove: `SafeClearDeathState` (unused), `g_reviveGuardUntilMs` (unused after Task 2/4), `kBleedoutEnum` if no longer referenced, and any stale "bleedout"/"resurrect" comments. Keep `SafeIsLocalPlayer`, `SafeSetLifeStateRaw`, `kAliveEnum`/`kDeathEnum1`/`kDeathEnum2`, `kDownedFloor`. Leave `SafeIsInCombat` (still referenced by Way B docs) or delete if truly unused.
- [ ] **Step 2: Build.** Expected: compiles with NO C4505 warnings for the deleted symbols.
- [ ] **Step 3: Commit.** `"refactor(coop): drop dead bleedout/resurrect code from downed "`

---

## Task 7: Build, deploy, in-game test (manual)

**Files:** none (deploy + test)

- [ ] **Step 1: Build + deploy.** Build, copy DLL+PDB → game Plugins, DLL → NAS deploy, restart BOTH games.
- [ ] **Step 2: Test — stable downed.** One player takes lethal damage → player **frozen, alive, NOT through the floor**, controls locked; partner sees the clone standing still. Log: a SINGLE `entered downed (keep-alive...)` (no per-frame spam) + `controls locked`.
- [ ] **Step 3: Test — combat-end revive.** Kill all hostiles → Way B fires → `revive (combat-over)` → controls return, player moves. Clone on partner screen moves again.
- [ ] **Step 4: Test — F10 revive.** Down with one enemy still alive (Way B holds off) → press F10 → `revive (F10)` → unfreeze.
- [ ] **Step 5: Test — both-down.** Both players down → `BOTH downed -> forcing death` → both load last save, no host crash.
- [ ] **Step 6: Regression.** Enemies/NPCs still die normally (setter hook is player-only).
- [ ] **Step 7: Update memory.** Record the result in `project_coop_downed_revive.md` + `MEMORY.md` + `.remember/remember.md`.

---

## Self-Review

**Spec coverage:** Entry trigger=Task 2; HP clamp=Task 3; control-lock=Task 1+3; revive=Task 4; both-down=Task 5; net-sync=existing (unchanged, noted); cleanup=Task 6; testing=Task 7. All spec sections covered. MVP-only (medpack/pose deferred per spec) — no tasks for those (correct).

**Placeholder scan:** The only "fill from RE" is Task 1 Step 2's ABI — unavoidable (the RVA/signature is the deliverable of Step 1) and bounded by a concrete fallback (Step 4). Not a hidden placeholder.

**Type consistency:** `SafeSetPlayerControlsEnabled(bool)`, `SafeSetLifeStateRaw(Actor*, uint32)`, `SafeRaiseHealthTo(Actor*, AVInfo*, float)`, `g_localDowned` (atomic<bool>), enums `kAliveEnum/kDeathEnum1/kDeathEnum2`, `kDownedFloor` — used consistently across Tasks 1–6.
