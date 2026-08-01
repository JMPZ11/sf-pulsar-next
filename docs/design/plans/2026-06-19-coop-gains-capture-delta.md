# Co-op Gains-Capture (Delta) Implementation Plan


**Goal:** Co-op gains (looted/crafted weapons+armor, ammo/resource counts, credits earned) persist to the DB-authoritative profile WITHOUT drifting existing items/credits/ammo — completing Phase C DB-authoritative progression.

**Architecture:** Snapshot the live inventory the moment the restore settles (`+10s`, "post-restore baseline" — this INCLUDES the restore's own artifacts and the host-save's credits, deliberately). On capture (disconnect / periodic, dbAuth mode only) compute `delta = live − baseline`; the artifacts and host-credit noise are present in BOTH sides so they cancel, leaving only TRUE session changes. Apply that delta to the frozen DB profile and upload. Existing items keep their exact faithful mods; only real gains/losses move.

**Tech stack:** C++ SFSE plugin, SpacetimeDB JSON profile, existing DirectRestore/ClientProfile/LobbyHandoff. Builds on an earlier commit (faithful mod restore + drift fix).

---

## Background / why delta (not full re-capture)

Full re-capture of the live inventory drifts because the round-trip is faithful ONLY for weapons/mods (fixed in `ddccecf`), NOT for:
- **Credits** — after the host-save bridge the player has the HOST's credits (no Money-API to restore the client's), so a full capture uploads host credits.
- **Stackables** — the restore produces slightly different counts than the DB (`DIFF countMismatch=2`, live 263 vs DB 241): a restore artifact, not a gain. Full capture would re-upload it and drift up each join.

Delta-against-post-restore-baseline cancels both: baseline credits = host credits (cancels), baseline stackable count = 241-ish artifact value (cancels). Only what the player actually changed during the session survives.

## Scope (V1 vs deferred)

- **V1 (this plan):** item gains/losses (new distinct items with faithful mods; stackable count deltas), credit deltas. The headline test: loot a weapon → reload → it's there.
- **Deferred V2:** perk/level/skill-point gains during co-op (level is already deferred per; perks come from leveling). Merged profile keeps the DB baseline's perks/AVs/skillPoints for now.

---

## File structure

- `src/InventorySync/DirectRestore.{h,cpp}` — stash the DB baseline + take the post-restore live snapshot; expose them.
- `src/Profile/ClientProfile.cpp` — `CaptureProfile` gains a dbAuth delta-merge branch.
- `src/Lobby/LobbyHandoff.cpp` — re-enable the post-restore upload (revert the C2 `!IsDbAuthoritative()` gate) now that the merge is delta-safe.
- `src/Main.cpp` — build marker bump (`gains-v1`) for build identification.

---

## Task 1: Stash the DB baseline profile

**Files:** Modify `src/Profile/ClientProfile.cpp` (ApplyProfileData), `src/InventorySync/DirectRestore.{h,cpp}`.

The delta-merge needs the authoritative DB profile (items+credits) as the merge base. ApplyProfileData already has `a_data` (the decoded DB ProfileData). Stash it.

- [ ] **Step 1.1 — DirectRestore: add baseline storage + API**

In `DirectRestore.cpp` near the other Phase C statics (after `g_pendingGhostUnequip`):
```cpp
// Phase C gains — the authoritative DB profile snapshot (items+credits) that
// this dbAuth session restored FROM. Used as the merge base so a gains-capture
// adds only the session delta on top of the frozen DB, never re-uploads the
// (artifact-bearing) live inventory wholesale.
struct BaselineItem { std::uint32_t count; std::vector<std::uint32_t> mods;
                      std::uint16_t sentinel; bool equipped; std::string plugin;
                      std::uint32_t localID; };
std::unordered_map<std::uint32_t, BaselineItem> g_dbBaselineItems;  // formID → item
std::uint32_t g_dbBaselineCredits = 0;
std::atomic<bool> g_dbBaselineValid{ false };
```

Add setter + getters (declared in `DirectRestore.h`):
```cpp
void SeedDbBaseline(const std::vector<...>& items, std::uint32_t credits); // fills the map
bool GetDbBaseline(std::unordered_map<std::uint32_t, BaselineItem>& out, std::uint32_t& credits);
```
Actually expose via a single accessor returning a const ref + valid flag to avoid copies. Define `BaselineItem` in the header so ClientProfile can read it.

- [ ] **Step 1.2 — ClientProfile ApplyProfileData: call SeedDbBaseline**

In `ApplyProfileData`, in the `if (dbAuth)` block (next to `SeedAVsFromProfile`/`SeedPerksFromProfile`), pass the decoded inventory + credits:
```cpp
Pulsar::InventorySync::DirectRestore::SeedDbBaseline(a_data.inventory, a_data.credits);
```
(Map each `ProfileItem` → BaselineItem.)

- [ ] **Step 1.3 — Reset()**: clear `g_dbBaselineItems`, `g_dbBaselineValid=false`, `g_dbBaselineCredits=0`.

- [ ] **Step 1.4 — Build; verify compiles.**

## Task 2: Post-restore live snapshot (the delta reference)

**Files:** Modify `src/InventorySync/DirectRestore.cpp` (TickPostRestoreWatch stage==2 / +10s).

- [ ] **Step 2.1 — Add snapshot storage**
```cpp
// Live inventory the instant the restore settled (+10s). Delta reference: it
// carries the SAME restore artifacts + host credits as a later capture, so
// (capture − snapshot) = pure session gains/losses.
std::unordered_map<std::uint32_t, std::uint32_t> g_postRestoreSnap;  // formID → count
std::uint32_t g_postRestoreSnapCredits = 0;
std::atomic<bool> g_postRestoreSnapValid{ false };
```

- [ ] **Step 2.2 — Take the snapshot at +10s** (TickPostRestoreWatch, stage==2, after the existing O2 re-assert, dbAuth only):
```cpp
if (IsDbAuthoritative()) {
    g_postRestoreSnap.clear();
    player->ForEachInventoryItem([&](const RE::BGSInventoryItem& it) {
        if (it.object) {
            std::uint32_t tot = 0;
            for (auto& s : it.stacks) tot += s.count;   // sum stacks
            g_postRestoreSnap[it.object->GetFormID()] += tot;
        }
        return RE::BSContainer::ForEachResult::kContinue;
    });
    g_postRestoreSnapCredits = <read live credits>;       // reuse ClientProfile's credit read
    g_postRestoreSnapValid.store(true, std::memory_order_release);
    REX::INFO("[PULSAR] [DirectRestore] gains baseline snapshot: {} forms, credits={}",
              g_postRestoreSnap.size(), g_postRestoreSnapCredits);
}
```
Expose `GetPostRestoreSnap()` + `IsPostRestoreSnapValid()` in the header.

- [ ] **Step 2.3 — Build; verify the snapshot log line appears in-game (+10s).**

## Task 3: Delta-merge in CaptureProfile (dbAuth branch)

**Files:** Modify `src/Profile/ClientProfile.cpp` (CaptureProfile).

- [ ] **Step 3.1 — Branch on dbAuth + baseline+snap valid**

At the top of the inventory-building part of `CaptureProfile`, if
`IsDbAuthoritative() && GetDbBaseline(...) && IsPostRestoreSnapValid()`, build the
profile by **delta-merge** instead of raw live capture:

```
merged_items = copy(g_dbBaselineItems)             // frozen DB items (faithful mods)
live = read live inventory (formID → {count, mods, sentinel, equipped})
snap = g_postRestoreSnap

for form in live:
    delta = live[form].count - snap.value(form, 0)
    if form in merged_items:
        merged_items[form].count += delta          // ammo gain/loss on existing
        merged_items[form].equipped = live[form].equipped   // current loadout
    else if delta > 0:
        merged_items[form] = { count=delta, mods=live.mods,   // NEW gain → faithful mods
                               sentinel=live.sentinel, equipped=live.equipped,
                               plugin/localID from form }
for form in snap not in live:                        // fully removed during session
    if form in merged_items: merged_items[form].count -= snap[form]

drop merged_items with count <= 0
merged_credits = g_dbBaselineCredits + (live_credits - g_postRestoreSnapCredits)
```
Emit `data.inventory` from `merged_items`, `data.credits = merged_credits`. Keep
perks/skillPoints/AVs from the DB baseline (V2 handles perk/level gains).

- [ ] **Step 3.2 — Diagnostic log** the delta so the test is verifiable:
```cpp
REX::INFO("[PULSAR] [Profile] gains delta: +{} new items, {} count-changes, "
          "credits {}{:+d}", newItems, countChanges, g_dbBaselineCredits, creditDelta);
```

- [ ] **Step 3.3 — Non-dbAuth path unchanged** (bootstrap full-capture stays as-is).

- [ ] **Step 3.4 — Build; verify compiles.**

## Task 4: Re-enable the post-restore upload

**Files:** Modify `src/Lobby/LobbyHandoff.cpp`.

The C2 drift-fix gated `clientPostRestore` behind `!IsDbAuthoritative()`. Now the
dbAuth capture is delta-safe, so re-enable it.

- [ ] **Step 4.1 — Revert the gate** in BOTH `TickCleanProfileCapture` and `TickAutoSave`:
```cpp
const bool clientPostRestore =
    Pulsar::InventorySync::DirectRestore::RestoreComplete();
```
(Remove `&& !IsDbAuthoritative()`.) Update the comment to reference the delta-merge.

- [ ] **Step 4.2 — Guard:** the delta-merge MUST be active before allowing the upload —
if `IsDbAuthoritative()` but the post-restore snapshot is NOT yet valid (capture
fired before +10s settle), SKIP the upload (re-arm) so we never full-capture in
dbAuth mode. Add to the gate.

- [ ] **Step 4.3 — Build marker → `gains-v1`; build + deploy.**

## Task 5: In-game verification

- [ ] **Step 5.1 — Loot persistence (the headline):**
  1. Client joins (dbAuth, hall's frozen DB). Note baseline snapshot log.
  2. Loot a NEW modded weapon in co-op.
  3. Disconnect → check log `gains delta: +1 new items`. Check DB: hall's profile now has the weapon (item count +1, version bumped).
  4. Reload + join → the weapon is restored faithfully (`modfix[restore] act==exp` for it).
- [ ] **Step 5.2 — No false drift:** join, do NOTHING, disconnect, rejoin twice.
  DB item counts + credits must stay STABLE (delta = 0 → no change). This proves
  the artifact/host-noise cancellation works.
- [ ] **Step 5.3 — Credits:** earn credits in co-op → disconnect → DB credits =
  baseline + earned (NOT host credits, NOT zeroed).
- [ ] **Step 5.4 — Ammo:** spend/gain ammo → DB reflects the delta, doesn't drift on idle joins.

## Task 6: Cleanup + commit

- [ ] **Step 6.1 — Trim** the verbose `modfix[restore]` 3-lines-per-weapon diagnostic to a single summary line (keep act==exp assertion countable). Keep the gains-delta log.
- [ ] **Step 6.2 — Commit** (local master, no push): `feat(progression): Phase C3 — co-op gains-capture (delta-merge)`.
- [ ] **Step 6.3 — Update** `project_m3_player_inventory` memory + `.remember`.

---

## V2 (2026-06-19) — perk + skill-point GAINS capture

**Goal:** co-op skill/perk progression (ranking up a skill, spending/gaining skill
points) persists. Currently the C3 delta-merge keeps perks/skillPoints from the
DB **baseline** (frozen) → gains are lost.

**Key insight:** perks + skill points are RESTORED to hall's values (ApplyPerksReplace
+ skillPts seed), so post-restore LIVE perks/skillPoints = hall's + co-op gains —
NO host poison (unlike level/credits which carry the host value). So capture LIVE
directly (no snapshot/delta needed), guarded against a bad mid-mutation read.

- [ ] **CaptureProfile dbAuth delta-merge** — after building `merged` from the DB
  baseline, override perks+skillPoints with the LIVE capture IF valid:
  `if (data.perks.size() >= g_dbBaseline.perks.size() && !data.perks.empty()) {
   merged.perks = data.perks; merged.skillPoints = data.skillPoints; }`
  (>= baseline count accepts new perks AND rank-ups, same count; rejects a
  loss = a bad SEH-walk read → keep baseline.) Log the perk/skillPts delta.
- [ ] Build marker `gains-v4-skills`; build + deploy; in-game: rank up a skill in
  co-op → disconnect → DB perks/skillPts reflect it → reload → restored. Idle join
  → no perk drift (live==baseline → same).

**DEFERRED to V3 (needs RE, capture-only = low immediate value):**
- **Level + XP:** no clean player-level/XP engine API (`TESActorBaseData::GetLevel`
  is the base template level, unreliable for the player; no XP field exposed).
  playerLevel isn't even captured today (DB=0). Capture-only (apply deferred,
  option A) so low value now. Revisit with proper RE of the player level/XP fields.
- **Skill-challenge progress** (XP bar toward next perk rank), **research/recipes**,
  **Starborn powers** — each needs its own structure RE.

## Open risks / notes

- **Snapshot timing:** if a capture can fire between restore and +10s, Step 4.2's guard prevents a wrong full-capture. Verify the disconnect-capture waits for snapshot-valid.
- **Equipped churn:** merged equipped flags come from live (current loadout). Should round-trip (restore sets, capture reads) — watch for equipped flips on idle joins in Step 5.2.
- **localID/plugin for new items:** new gains need their T34 plugin+localID for cross-machine FormID. Reuse ClientProfile's existing per-item encode (it already does this for the bootstrap).
- **Multi-stack non-stackables:** weapons are count-1 per stack; summing stacks per form is correct for ammo/resources and harmless for weapons.
