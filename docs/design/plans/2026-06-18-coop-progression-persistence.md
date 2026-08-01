# Co-op Progression Persistence Implementation Plan


**Goal:** The client's full progression — including items/loot gained during a co-op session — persists across logout/login and is restored on the next join regardless of which save file is loaded, with item mods intact.

**Architecture:** The DB profile becomes the single source of truth for client progression. It is continuously captured from the client's *confirmed-own* live inventory (own save when solo, AND post-restore during co-op once host items are wiped). On join it is restored DB-authoritatively. Mods (stored in the profile as the M.2 mod array) are re-applied on restore so DB-restored items are not naked base forms.

**Tech Stack:** SFSE C++ plugin (1.16.236), CommonLibSF, STDB (`pulsar-lobby`) + companion JSON handoff. No unit-test framework for engine interactions — every task is verified by build → deploy → in-game → log/DB inspection (the project's standard loop).

**Why this is needed (the core problem):** During co-op the bridge makes the client RUN the host's save, so any save created then (incl. the engine's leave-co-op auto-save) is the *host's* character. The client's own save is the pre-co-op (stale) state. So **no save file holds the client's post-co-op progress** — only the DB can. "Load your own save" loses the session's gains; that is unacceptable. The DB must capture gains during play and restore them on any join.

**Current state (already shipped, this is the foundation):**
- Save-authoritative restore for the client's OWN save (live pre-bridge capture preserves items+mods+AVs; in-game confirmed clean).
- `DirectRestore`: `g_cache` (restore source), `SeedCacheFromProfile` (DB seed = fallback/base-forms), `CaptureClientState` (live walk WITH ExtraData), `LockCache`/`UnlockCache` (bridge window), `g_bridgedThisSession` (sticky), `g_loadedSaveIsOwn` (foreign/host-save detection via Health-base signature), perk replace + `FUN_1419577c0` derived-AV refresh.
- DB capture (`Lobby::TickCleanProfileCapture` + `TickAutoSave`) is FROZEN during co-op (`HasBridgedThisSession()` gate) — this is exactly what blocks gains from persisting. The DB profile stores items WITH their mod arrays + AVs (verified via online DB export: banso12 = 17 items + full mods + o2 105 + hp 200).
- Restore is DB-authoritative ONLY for a detected foreign/host save (`g_loadedSaveIsOwn==false`); otherwise save-authoritative.

**Key risk acknowledged up front:** flipping to DB-authoritative restore BEFORE mods are re-applied would regress the working own-save mod fidelity (DB seed = base forms). Therefore mod re-apply (Phase B) MUST land before the DB-authoritative flip (Phase C).

---

## Phase A — Capture co-op gains into the DB (no restore change yet)

Make the DB profile track the client's gains as they play. This is purely additive — the restore path is unchanged, so there is no regression risk; the DB simply becomes current.

### Task A1: `g_restoreComplete` state in DirectRestore

**Files:**
- Modify: `src/InventorySync/DirectRestore.cpp` (flag near `g_loadedSaveIsOwn`; set in `RestoreClientState` end + `LockCache`)
- Modify: `include/InventorySync/DirectRestore.h` (declare `RestoreComplete()`)

- [ ] **Step 1: Add the flag + accessor**

```cpp
// DirectRestore.cpp, anonymous namespace, near g_loadedSaveIsOwn:
// True once RestoreClientState has finished a bridge restore — i.e. the live
// inventory is now the CLIENT's items (host items were wiped), so it is safe to
// capture to the DB. Cleared in LockCache (next bridge re-merges host state).
std::atomic<bool> g_restoreComplete{ false };
```

```cpp
// DirectRestore.cpp, near HasBridgedThisSession():
bool RestoreComplete() { return g_restoreComplete.load(std::memory_order_acquire); }
```

```cpp
// DirectRestore.h, near HasBridgedThisSession():
// True once a bridge RestoreClientState has completed this session — the live
// inventory is the client's own (host items wiped), safe to capture to the DB.
bool RestoreComplete();
```

- [ ] **Step 2: Set it at the end of `RestoreClientState`** (after the perk-derived AV refresh line), and clear it in `LockCache()`:

```cpp
// end of RestoreClientState, after SafeRefreshDerivedAVs(player):
g_restoreComplete.store(true, std::memory_order_release);
```
```cpp
// LockCache(), alongside g_bridgedThisSession.store(true,...):
g_restoreComplete.store(false, std::memory_order_release);   // bridge merges host state again
```
And clear it in `Reset()` (disconnect): `g_restoreComplete.store(false, ...)`.

- [ ] **Step 3: Build + deploy + verify** the log shows the existing `RestoreClientState V4.1` line still fires and nothing regresses (own-save join still clean). No behaviour change yet.

```
"/c/Program Files/CMake/bin/cmake" --build build/vs --config RelWithDebInfo --target PULSAR
```

- [ ] **Step 4: Commit** `feat(progression): A1 — g_restoreComplete state (post-restore = client-owned live)`

### Task A2: Allow DB capture post-restore (the gains)

**Files:**
- Modify: `src/Lobby/LobbyHandoff.cpp` (`TickCleanProfileCapture`, `TickAutoSave` gates)

- [ ] **Step 1: Loosen the `TickCleanProfileCapture` gate** so it ALSO captures post-restore (co-op gains). The live inventory is the client's in two cases: own save pre-bridge, OR post-restore.

Replace:
```cpp
        if (Pulsar::InventorySync::DirectRestore::HasBridgedThisSession()) return;
        if (!Pulsar::InventorySync::DirectRestore::LoadedSaveIsOwn()) return;
```
with:
```cpp
        // Capture only when the live inventory is the CLIENT's own:
        //   (a) own save, before any bridge (solo / pre-join), OR
        //   (b) post-restore during co-op — RestoreClientState wiped host items,
        //       so the live inventory is the client's items + the loot gained
        //       this session. This is how co-op gains reach the DB.
        // Never while a bridge is mid-flight (the lock check inside the capturer
        // handles that), never from a foreign save before its restore.
        const bool ownPreBridge =
            !Pulsar::InventorySync::DirectRestore::HasBridgedThisSession() &&
            Pulsar::InventorySync::DirectRestore::LoadedSaveIsOwn();
        const bool clientPostRestore =
            Pulsar::InventorySync::DirectRestore::RestoreComplete();
        if (!ownPreBridge && !clientPostRestore) return;
```

- [ ] **Step 2: Same loosening for `TickAutoSave`** (disconnect capture): allow it when `RestoreComplete()` (post-co-op disconnect now captures the client's gains, not host poison, because host items were wiped). Replace the `HasBridgedThisSession()` SKIP with the same `ownPreBridge || clientPostRestore` condition.

- [ ] **Step 3: Build + deploy + in-game verify.** Join co-op, pick up a distinct item, wait ~120s (or disconnect). Expect log: `clean profile-capture (periodic)` (or `auto-save captured`) firing DURING co-op, and the captured item count > the join count. Then export the DB profile (online `spacetime sql` / the server export path) and confirm the new item is present.

- [ ] **Step 4: Commit** `feat(progression): A2 — capture co-op gains to DB post-restore`

**Phase A exit criteria:** the DB profile reflects items gained during co-op (verified by DB export growing across a session). Restore behaviour unchanged.

---

## Phase B — Re-apply item mods on DB restore 

Without this, a DB-authoritative restore yields naked base-form items (engine rolls random mods). The mod data IS in the profile (M.2 array) and in `CachedStack::extra` (ExtraData) — the gap is re-application.

### Task B1: RE — why the M.2 mod-apply corrupts; find the correct attach path ✅ DONE (2026-06-18)

**RE outcome (Ghidra, this session):**

- **Why the M.2 inventory primitives corrupt / no-op.** `SafeRestoreModArrayOnInventoryItem` /
  `SafeReplaceModArrayOnInventoryItem` (LootSync) walk the item's `kObjectInstance` (0x33)
  ExtraData and OVERWRITE existing array slots in cache-order, clamped to
  `commonSlots = min(localCount, cachedCount)`. Two failure modes:
  1. **base form (DB seed)** → `localCount==0` / `localArray==null` → `commonSlots==0` → writes
     NOTHING, just sets `modCount=0`. This is the observed `modsApplied=0` → suit naked → O2=0.
  2. **proc-rolled form** → engine placed mods in slots that don't line up with our cache order →
     wrong mod in wrong slot ("epic item turns grey"). The primitive CANNOT allocate/grow and
     does NOT respect the mod's target slot.

- **The correct engine path** (decompiled, 1.16.236):
  Papyrus `AttachMod` handler `FUN_14206d840` dispatches REFR-vs-inventory:
  - REFR branch → `FUN_140402cf0` → `FUN_140403910` (= our `g_attachMod`, REL::ID 40716). Works on
    a REFR's OWN `+0xC8` ExtraData — correct for a spawned world item (F.2 drop), WRONG target for
    an inventory item (would attach to the actor, not the item).
  - inventory branch → `FUN_14206d6b0(owner, uint16 uniqueID, mod, flag, 1)` → `FUN_1412e1a20`
    (attach-by-INDEX, the crafting path) → **`FUN_14063de80(ctx, entry, &status)`** = the per-entry
    attach core. **RVA `0x63de80`.**
  - `FUN_14063de80`: when `ctx+0x21 != 0` (attach mode) it FINDS-or-ALLOCATES the entry's
    `kObjectInstance` (`FUN_14063f7b0`, size 0x30, vtable `BGSObjectInstanceExtra`, sentinel
    init 0xffff), GROWS the array via `FUN_140404300`, then ADDS the mod via `FUN_1404041f0`
    (picks the CORRECT slot from the mod template). This is exactly allocate+grow+correct-slot —
    the two things the M.2 overwrite primitive lacks.
  - **ctx ABI** (param_3 of `FUN_1412e1a20`, param_1 of `FUN_14063de80`), 0x30 bytes zeroed:
    `+0x00 owner REFR* · +0x08 mod TESForm* · +0x10 scratch(0) · +0x18 &statusByte ·
     +0x20 flag(1) · +0x21 attach-mode (MUST be 1) · +0x22 scratch(0)`. `statusByte` init 1.
  - `param_2` = the inventory ENTRY pointer (`*(invList+0x30) + index*0x28`); the engine navigates
    to the stack's ExtraData itself.
  - **Why `FUN_14063de80` not the full `FUN_1412e1a20`:** the wrapper adds a critical section,
    `BGSInventoryListEvent` dispatch, item-value recompute, and an async `BSService` service-command
    branch (gated on `GetCurrentThreadId()` — only off the game thread, but still extra machinery).
    `FUN_14063de80` is the bare attach: no critsec, no event, no async — minimal crash surface in
    the fragile post-bridge window. The subsequent `DirectEquipForm` unequip→equip re-reads
    `instanceData`, so the inventory event isn't required for the equip bonus (suit O2) to apply.

### Task B2: Restore items WITH their mods (native attach, restore-only) ✅ IMPLEMENTED (2026-06-18)

**Files:**
- `include/WorldSync/LootSync.h` — declare `AttachItemModsNative`.
- `src/WorldSync/LootSync.cpp` — resolve `FUN_14063de80` (RVA 0x63de80) in `Init`; add
  `SafeAttachModNativeToInventoryItem` (file-static) + public `AttachItemModsNative`.
- `src/InventorySync/DirectRestore.cpp` — call `AttachItemModsNative` (not `ApplyItemMods`) on the
  restore path; flip `kApplyModsOnRestore = true`.

**Decisions / guard-rails:**
- **New function, NOT a change to `ApplyItemMods`.** `ApplyItemMods` (overwrite primitive) still
  serves its other callers — RemotePlayerManager clone-equip (fragile anim-graph) and
  NPCSync corpse-loot (, untested). Changing those is out of scope and risky; restore gets a
  dedicated native path → minimal blast radius.
- Modded items in the restore set are almost always count-1 (weapons/armor/suit), so the inventory
  index is stable across per-mod attach calls. Re-resolve the entry by formID before each attach
  defensively. Duplicate same-base forms → key-by-formID ambiguity (accepted for Alpha; modded gear
  is typically unique).
- SEH-wrap every native call. Gate behind `kApplyModsOnRestore` (already present) so it can be
  disabled instantly if it regresses.
- After attaching, stamp the source sentinel (+0x28 tier) so item rendering matches; optional, one
  SEH write.

- [x] **Step 1:** `SeedItem`/`SeedCacheFromProfile`/`CachedItem` already carry `mods`+`modSentinel` (Phase B plumbing).
- [x] **Step 2:** Implement `AttachItemModsNative` + wire into restore; flip `kApplyModsOnRestore`.
- [ ] **Step 3:** Build + deploy + in-game verify: load the HOST save (FOREIGN) → DB restore → modded weapons show CORRECT mods (not random/grey) AND the suit regains O2.
- [ ] **Step 4: Commit** `fix(inventory): native engine mod-attach on DB restore`

**Phase B exit criteria:** items restored from the DB seed carry their correct mods (own-save fidelity preserved; foreign/DB restore no longer base-form, suit O2 back).

---

## Phase C — DB-authoritative restore (gains survive any loaded save)

**User-Modell (2026-06-19, verbatim):** "client lädt sein game, sachen landen von dem save in der DB; danach wenn DB gefüllt ist wird IMMER aus der DB geladen → egal welche Lobby gejoint wird, behält man Items, lvl, progress." → **leere DB = Bootstrap aus dem Save; gefüllte DB = DB ist autoritativ.**

### Key constraint discovered (2026-06-19) — items+AVs+perks are COUPLED

The post-restore upload (`CaptureProfile`) captures the **whole** live player state. So if the
restore leaves AVs/perks coming from a *foreign* (host) save while only items come from the DB, the
next upload writes host AVs/perks back into the DB → poison. **Therefore DB-authoritative must cover
items + AVs + perks together**, or not at all. The upside: once everything restores from the DB, the
upload is idempotent (live == DB + gains), which **removes the dependence on the fragile hp-based
own/foreign detection entirely** — the DB is the source of truth regardless of which save is loaded.

**Gap found:** perks/skills are NOT in the DB profile today. `ProfileData` has items + level-number +
credits + health/carry/o2 bases only. Perks/skills travel via the loaded save (`DirectRestore`
`g_perkSnap`, captured pre-bridge). So Phase C requires **adding perks + skill-points to the DB blob
first** (C0), then flipping the restore (C1). Skills in Starfield ARE perks (the skill tree), so the
perk array covers skills. Level/XP set is deferred (engine XP is a separate hard problem; perks +
skill-points are the real "progress").

### Task C0: Add perks + skill-points to the DB profile (additive, safe) ✅ IMPLEMENTED (2026-06-19)

**Files:** `include/Profile/ClientProfile.h` (`ProfilePerk` + `ProfileData.perks` + `skillPoints`),
`src/Profile/ClientProfile.cpp` (`CaptureProfile` walks `player->perks` + reads skill-points @ +0xF60;
`EncodeProfile`/`DecodeProfile` round-trip them; self-test covers them).

- [x] Add `ProfilePerk { formID, rank, plugin, localID }`, `ProfileData.perks`, `ProfileData.skillPoints`.
- [x] `CaptureProfile`: walk perks (T34-encode each), read skill-points (+0xF60).
- [x] `EncodeProfile`/`DecodeProfile`: round-trip perks (CSV-of-records) + skillPoints; extend `RunSelfTest`.

This is purely additive: it enriches the uploaded blob; restore behaviour is unchanged. The captured
perks are unused until C1 flips the restore, so there is no behaviour risk and nothing to test in-game
yet (verify via the load-time self-test PASS line).

### Task C1: Flip restore to DB-authoritative (behind `dbAuthoritative` INI toggle, default OFF)

**Files:** `src/InventorySync/DirectRestore.cpp`, `src/Profile/ClientProfile.cpp`,
`src/Lobby/LobbyHandoff.cpp`, `src/Config.{h,cpp}` (toggle).

- [ ] **g_dbAuthoritative** = (toggle ON) AND (DB seed non-empty). Set in `SeedCacheFromProfile`/seed path.
- [ ] **Restore source:** when `g_dbAuthoritative`, the DB seed (items + mods via, AVs, perks,
      skill-points) is the restore source. `CaptureClientState` skips ALL live capture (items+perks+AVs)
      so the DB seed survives the bridge (this is the existing foreign-skip, re-gated from "hp-foreign"
      to "DB non-empty"). Seed AVs + perks from the DB into the restore (new: `SeedAVsFromProfile`,
      `SeedPerksFromProfile`).
- [ ] **Bootstrap:** when DB empty, capture the live own save (as today) → restore it → upload fills the
      DB. After that the DB is non-empty → authoritative.
- [ ] **Upload gating (no stale-overwrite):** when `g_dbAuthoritative`, upload ONLY post-restore (live =
      DB + gains → DB grows). Never pre-bridge (a stale loaded save must not overwrite the DB). When DB
      empty (bootstrap), pre-bridge upload is allowed (fills the DB).
- [ ] Build + deploy. **Test (toggle ON):** bootstrap from a clean DB with own save → DB fills; then load
      ANY save (stale own / host autosave) → items+mods+perks+AVs restored from DB. The 3h-coop scenario.
- [ ] **Commit** `feat(progression): C — DB-authoritative restore (gains survive any save)`

**Clean-DB note:** banso's existing DB profile holds residual pre-fix items; for a clean bootstrap test
either reset that identity's profile once (STDB) or accept its current content. Profile is keyed by
STDB **identity**, not display name — renaming does not yield a fresh profile.

**Phase C exit criteria:** load ANY save → items + co-op gains + mods + perks/skills + AVs restored from
the DB; the upload keeps the DB current+clean; no dependence on hp-based own/foreign detection.

---

## Self-Review (run before handing off)

1. **Spec coverage:** Phase A = gains captured; Phase B = mods intact; Phase C = restored on any save + AVs. The core requirement ("don't lose 3h of co-op") is met by A+C; B+C2 are fidelity.
2. **Sequencing:** B (mods) MUST precede C (DB-authoritative flip) or own-save mod fidelity regresses — encoded above.
3. **Poison safety:** every capture is gated on `ownPreBridge || RestoreComplete` and never mid-bridge (`g_cacheLocked`) — host items are wiped before any post-restore capture, so they can't reach the DB.
4. **Type consistency:** flags `g_restoreComplete`/`g_loadedSaveIsOwn`/`g_bridgedThisSession` + accessors `RestoreComplete()`/`LoadedSaveIsOwn()`/`HasBridgedThisSession()` used consistently.

## Open questions / risks
- **B1 (mod-apply corruption)** — RESOLVED. Root cause = the M.2 inventory primitive only
  overwrites existing slots (can't allocate/grow, ignores slot compatibility). Fix = the engine's
  own per-entry attach `FUN_14063de80` (RVA 0x63de80). In-game test pending (Step 3). If the native
  attach crashes in the post-bridge window, fall back to `kApplyModsOnRestore=false` (base forms,
  gains still persist) — the toggle isolates the risk.
- **Cross-machine** (joining from a PC without the save): DB-authoritative restore makes this work for the first time (the DB provides the loadout) — a bonus of Phase C.
- **Stale-own-save detection** is sidestepped by making restore DB-authoritative always (the DB is the latest by construction via continuous capture).
