# Inventory/Progression Sync Unification (A) — Implementation Plan

> **Goal:** ONE authoritative, DB-backed client profile is the single source of
> truth for the client's inventory + equipment + progression. Kill the poisonable
> per-bridge live re-capture (Bug A) and the additive dual-apply (Bug A join), so
> "host items leak onto the client" and "no visible weapon on join" can't happen.

**Driver:** user insight — items/skills already persist in the online DB; the local
"two systems fighting" (DB-fed `ClientProfile` additive + non-DB `DirectRestore`
live-capture replace) is the mess. Unify on the DB profile as the single source.

---

## Findings that shape the design (verified 2026-06-18)
- **DirectRestore is NOT DB-connected** — pure live-capture (`CaptureClientState`
  walks the live inventory). It is the **poisonable** path: during a bridge the
  client briefly holds host items, a capture grabs them → cache poisoned → restore
  re-adds host items (Bug A vicious cycle).
- **`RestoreClientState` is already a correct strict replace** (unequip host gear →
  wipe all → re-add cache → re-equip → O2/perks/skills/health). The bug is its
  SOURCE (poisonable cache), not its apply.
- **BFCB chunk-filter (M.3) is unreliable** — diag `playerHits=0 bfcbDrops=0
  bfcbInner playerCalls=0`. So we CANNOT just "trust the filter + drop the restore."
  The restore is load-bearing → fix its source, keep the filter only as first-line.
- **`ClientProfile` (DB-fed) is incomplete** vs DirectRestore: it captures inventory
  + equip + O2, but NOT perks / skill points / health-base / carryweight-base. For a
  single source it must capture everything.
- **`ClientProfile::ApplyProfileData` is ADDITIVE** (top-up `need = count - have`,
  never removes) → host/extra items survive (Bug A join).
- Credits are known-broken (AddObjectToContainer doesn't drive money) — out of scope,
  keep current least-broken handling.

---

## Target architecture
```
            STDB (online DB)  ← authoritative cross-session store
                 │  (lobby companion)
        pulsar_profile_in.json / out.json
                 │
   ┌─────────────▼──────────────┐
   │  ClientProfile  (the ONE     │  full state: inventory+extras, equipped set,
   │  authoritative client state) │  perks, skill points, O2/health/carryweight
   └─────────────┬──────────────┘
     seed at join │   update at SAFE points (NOT during a bridge)
                  ▼
       Apply engine (one path, REPLACE):
        wipe → re-add from profile → re-equip(+verify) → AVs → 3D rebuild
        (= DirectRestore's proven apply mechanics, fed from the profile)
```
- **Single source:** the in-memory authoritative `ClientProfile`, seeded from the DB
  at join, updated from the live inventory ONLY when clean (never mid-bridge).
- **Single apply:** the proven DirectRestore wipe+replace+equip+AV+3D mechanics,
  but reading the profile, not a separate poisonable cache.
- **Filter stays** as first-line (drop host BFCB) but is no longer relied on.

---

## Phase 1 — Make ClientProfile the COMPLETE state
**Files:** `src/Profile/ClientProfile.cpp/.h`.
- [ ] Extend `ProfileData` + `CaptureProfile` to also capture: perks (the additive
      perk set DirectRestore uses), skill points, Health base, CarryWeight base,
      O2 max. Mirror DirectRestore's capture logic (it already reads these).
- [ ] Keep per-item ExtraData where possible (durability/mods) — or accept base
      forms for v1 (mods already deferred via).
- [ ] Verify the JSON serialize/deserialize round-trips the new fields.
- [ ] **Test:** capture → log shows inventory+equip+perks+skills+AVs counts.

## Phase 2 — Make ApplyProfileData a strict REPLACE
**Files:** `src/Profile/ClientProfile.cpp`.
- [ ] Change apply from additive top-up to replace: wipe items not in the profile +
      set exact counts (reuse DirectRestore's `RawRemoveExceptKeepSet` + per-stack
      add with held ExtraData). Re-equip with the verify+retry (already added 12be9fa)
      + 3D rebuild. Apply perks/skills/AVs.
- [ ] This makes ClientProfile's apply identical to DirectRestore's apply → they can
      become ONE function (extract a shared `ApplyClientState(profile)`).
- [ ] **Test:** join with a profile → inventory EXACTLY matches profile (no host
      extras), weapon visible.

## Phase 3 — Single authoritative live profile (kill the poisonable re-capture)
**Files:** `src/InventorySync/DirectRestore.cpp`, `src/WorldSync/WorldSync.cpp`,
`src/Profile/ClientProfile.cpp`.
- [ ] Seed the authoritative profile at join from the DB (`pulsar_profile_in`).
- [ ] Update it from the LIVE inventory ONLY at safe points: NOT during a bridge
      (the lock already gates this), and only when `!IsPreBridgeSnapshotPending` and
      no recent bridge. Prefer event-driven (on inventory-change) or a clean periodic
      that verifies it's not mid-transition.
- [ ] On bridge (`OnSaveReceived`): do NOT re-capture from the (possibly host-tainted)
      live inventory. Use the already-held authoritative profile as the restore
      source. Remove `CaptureClientState`'s role as the per-bridge source.
- [ ] `RestoreClientState` reads the authoritative profile (not `g_cache`) → wipe +
      re-add + equip + AVs. Delete the poisonable `g_cache`/`LockCache` re-capture
      machinery once the profile is the source.
- [ ] **Test:** host loads a DIFFERENT save mid-session → client keeps ITS OWN
      inventory (no host items). Repeat F8 several times — no drift, no poisoning.

## Phase 4 — Collapse the two systems into one
**Files:** `src/Profile/ClientProfile.cpp`, `src/InventorySync/DirectRestore.cpp`,
`src/Lobby/LobbyHandoff.cpp`.
- [ ] One capture (`CaptureClientState`/`CaptureProfile` merged), one apply
      (`ApplyClientState`), one source (the authoritative profile).
- [ ] LobbyHandoff join-apply and the bridge-restore both call the SAME apply.
- [ ] Remove dead/duplicate paths. Keep the BFCB filter as first-line defense only.
- [ ] **Test:** full matrix (below).

## Phase 5 — DB write-back
**Files:** `src/Lobby/LobbyHandoff.cpp`, lobby companion.
- [ ] The authoritative live profile is written to `pulsar_profile_out.json` →
      lobby → DB at safe points + on disconnect (already exists via WriteProfileOut;
      ensure it serializes the FULL profile from Phase 1).

## Phase 6 — Test matrix (in-game, Sottle host / banso client)
- [ ] Join: client inventory = its own (not host's), weapon visible, perks/skills/
      health/O2 intact.
- [ ] Mid-session host F8 / host loads DIFFERENT save: client keeps own inventory,
      no host items, no drift over repeated bridges.
- [ ] Pick up items mid-session → bridge → kept. Disconnect → reconnect → persisted
      (DB round-trip).
- [ ] No save-load crash (this subsystem has UAF history — keep HookGuard::Suspend).

---

## Risks
1. **Crash-prone subsystem** — save-load/bridge has documented UAF history
   (HookGuard::Suspend, the V4.4 freeze on mass engine-path remove). Keep the
   suspend window; do NOT mass-mutate inventory mid-bridge; reuse the proven
   `RawRemoveExceptKeepSet` + held-ExtraData add (no engine-path negative-count loop).
2. **Live-update timing** — the authoritative profile must never update from a
   host-tainted live inventory. Gate strictly on the bridge lock + a settle window.
3. **Profile completeness** — if Phase 1 misses a field, that progression is lost on
   bridge. Validate each captured field against DirectRestore's current set.
4. **Credits** — stays broken (separate money-API RE). Don't regress it.
5. **Migration** — do it incrementally (Phase 1-2 are safe + independently testable);
   only Phase 3 removes the live re-capture (the risky cut) — test heavily there.

## Order
Phase 1 (complete profile) → Phase 2 (replace-apply) → test → Phase 3 (profile as
bridge source, the Bug-A cut) → test hard → Phase 4 (collapse) → Phase 5 (DB) →
Phase 6. Phases 1-2 are low-risk + testable alone; Phase 3 is the real fix.
