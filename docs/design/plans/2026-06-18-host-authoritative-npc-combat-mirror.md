# Host-Authoritative NPC Combat-Mirror (Shared Aggro) — Implementation Plan

> **Status:** BETA feature. Alpha ships with the current local-AI combat model
> (`npcCombatAiLive=1`, per-player aggro). This plan adds *shared aggro*: both
> sides see the same enemy target + fire at the same player.

**Goal:** The host's NPC combat decisions (who each enemy targets + when it fires)
become authoritative; the client mirrors them, so an enemy shooting the client is
visible on the host and vice-versa.

**Why it works:** On the host BOTH players exist as actors — the host player AND
the client's clone (remote-player avatar). The host enemy AI already aggros both.
That host-side target is the single source of truth; the client just reproduces it.

**Core model:** capture host enemy combat target → map to the client-side entity →
force the client's matched enemy to aim/engage it. Damage stays host-authoritative
(client clone takes the hits on the host → forwarded to the client player via the
existing downed/health path).

---

## Pre-req findings (already known this session)
- Fire anim is AI-driven; can't be poked into the graph (FIRE_RE/RE2/RE3 — see
  `reference_locomotion_graph_vars.md`). So the enemy's *own AI* must run to fire →
  this plan keeps `npcCombatAiLive` (AI runs) and adds target-steering on top.
- `g_npcRemap` (host enemy formID ↔ local matched formID) is solid (remap=9 in
  tests). RemotePlayer gives the client-clone↔host-player mapping.
- Soft position sync (factor 0.2) already keeps positions aligned — keep it.

---

## Phase 0 — RE the combat target field (BLOCKER, do first)

**Files:** `local RE notes (new), Ghidra headless.

What we need: given a host `RE::Actor*`, read **which actor it is currently fighting
/ aiming at** (an `ObjectRefHandle` or `Actor*`).

- [ ] Check CommonLibSF first: `Actor`/`AIProcess`/`CombatController` for a combat-
      target accessor (e.g. a `currentCombatTarget` handle). If it exists + resolves
      on 1.16.236, skip Ghidra. **Verify the offset in Ghidra regardless** (CLibSF
      offsets are often stale — see `feedback_commonlibsf_verify`).
- [ ] If absent: Ghidra RE. Path candidates:
      `Actor + 0x228 (AIProcess/currentProcess) → middleHigh (+0x08) → combat data`.
      The CombatController/CombatTarget lives off middleHigh. Find the getter by
      searching the combat-AI functions that read the target handle; cross-check
      against the `StartCombat`/`SetCombatTarget` writers.
- [ ] Deliverable: a verified offset chain + a SEH-wrapped `SafeReadNpcCombatTarget
      (actor) -> uint32_t targetFormID` (0 = no target).
- [ ] Also note `StartCombat(actor, target)` engine fn (for Phase 3 Level-B) — it's
      a Papyrus native (`Actor.StartCombat`), find its RVA / CLibSF binding.

**Risk:** if the target is an `ObjectRefHandle`, resolve it to an Actor → formID
(handle→ref lookup, SEH-guarded). Handles can be stale; treat 0/invalid as "no
target".

---

## Phase 1 — Host capture + protocol

**Files:** `include/Network/PacketTypes.h`, `src/NPCSync/NPCSync.cpp`,
`include/NPCSync/LocomotionApply.h`.

- [ ] `NPCStateData`: add `uint32_t combatTargetFormID;` → struct 114→118 bytes;
      update the `static_assert(sizeof(NPCStateData)==118)`. (Both sides rebuild —
      we control both binaries.)
- [ ] Host capture (NPCSync.cpp, near the weaponDrawn/fire-state sample ~1540):
      `entry.combatTargetFormID = SafeReadNpcCombatTarget(a_actor);`
      Add `combatTargetFormID` to `TrackedNPC`.
- [ ] Plumb: `dst.combatTargetFormID = src.combatTargetFormID` (send loop),
      `u.combatTargetFormID = e.combatTargetFormID` (recv), `LocomotionInput`
      gets a `combatTargetFormID` field, `li.combatTargetFormID = u...`,
      `LerpTarget.combatTargetFormID = in...` (store like graphSpeed).

**Verify:** `[COMBAT-DBG]` log on host: `enemy=0x.. target=0x..` shows the host
enemy's target (host player 0x14 / client-clone formID / another NPC).

---

## Phase 2 — Client target mapping

**Files:** `src/NPCSync/LocomotionApply.cpp` (TickLerp combat-live branch).

Map the received host-side `combatTargetFormID` to a client-side actor:
- [ ] host player `0x00000014` → **the host's clone on the client** (RemotePlayer
      `GetRemoteActor()`).
- [ ] the client-clone-on-host formID → **the client's own player** (`0x14`).
      (Need the client-clone formID; RemotePlayer knows the clone refID it spawned
      on the host — sync/store it, or recognize via base 0x7.)
- [ ] another host NPC formID → the matched local NPC (`g_npcRemap` **reverse**
      lookup: value→key, build a reverse map or scan).
- [ ] 0 / unresolved → no forced target (let local AI pick).
- [ ] Deliverable: `uint32_t MapHostTargetToLocal(uint32_t hostTargetFormID)`.

---

## Phase 3 — Client steering (two levels, ship Level A first)

**Files:** `src/NPCSync/LocomotionApply.cpp` (combat-live branch).

**Level A — cosmetic aim (quick visible win):**
- [ ] In the combat-live branch, resolve the mapped target's world position; force
      the enemy's facing (rotZ) — and pitch if a fire is active — toward it each
      frame instead of reading the AI's own facing. → both sides see the enemy
      *pointing at* the same player. Reuses `SafeNpcWriteDataLoc` rotZ write.
- [ ] Keep fire driven by the local AI (it still fires); aim now matches the host.

**Level B — true target (real shared aggro):**
- [ ] Force the engine combat target: `StartCombat(enemy, mappedTarget)` (or write
      the combat-target field) when it differs from the AI's current target, so the
      AI genuinely engages the synced player. Throttle (don't call every frame —
      only on target change).
- [ ] Gate behind the existing `npcCombatAiLive` (or a sub-toggle `npcCombatMirror`)
      so it's revertible.

**Risk:** forcing the target may fight the local AI's own pick → oscillation.
Mitigate: only force on change + a short cooldown; let the AI run between.

---

## Phase 4 — Damage reconciliation (prevent death-desync regression)

**Files:** `src/Coop/CoopMortality.cpp`, `src/NPCSync/NPCSync.cpp`.

- [ ] Confirm: with mirrored aggro, the host's enemies damage the **client's clone**
      on the host → that must forward to the client player (downed/health path). The
      earlier "dies host-side, invisible client-side" bug came from the clone NOT
      taking damage. Verify the clone IS hit + the forward fires.
- [ ] Decide the client-player damage source of truth: host-clone-damage-forward
      (host authoritative) vs local-enemy-damage (current AI-live). Pick ONE to
      avoid double-counting. Recommended: host-authoritative (clone damage forward)
      since aggro is now host-authoritative — but only if the forward is solid.

---

## Phase 5 — In-game test + tuning

- [ ] Both players (Sottle host / banso client) in an outpost fight together AND
      apart. Verify: (a) host sees enemies firing at the client + vice-versa,
      (b) no death-desync / invisible, (c) positions still synced + smooth,
      (d) no double-damage (enemy HP sane), (e) no crash on the proc planet.
- [ ] Tune: soft-sync factor (0.2), StartCombat throttle, aim-lerp rate.
- [ ] Remove the dead fire-poke scaffold (NotifyFireEvent / AttackWinStart /
      bAimActive / iState=17 — all skipped under combat-live) + diag spam.

---

## Open risks / unknowns
1. **Combat-target offset (Phase 0)** — the whole feature blocks on this RE. If the
   target is hard to read safely, Level A (aim-only via the host's facing rotZ,
   which we already capture) is a fallback that needs NO new field — just steer to
   the host-sent rotZ during combat instead of the AI's facing.
2. **Death-desync regression** (Phase 4) — the reason Alpha uses local-AI. Must be
   re-validated before shipping the mirror.
3. **StartCombat side-effects** — forcing combat AI may break the NPC's package /
   cause it to path oddly. Level A avoids this.
4. Proc-planet vehicle-entry crash (separate, host-side `ACCESS VIOLATION read -1`)
   is unrelated but should be fixed before heavy proc-planet combat testing.

## Recommended order
Phase 0 (RE) → Phase 1 (capture+protocol) → Phase 3 Level A (aim-only, no Phase 2
NPC-target needed for player-aim) → test → Phase 2 + Phase 3 Level B (true target)
→ Phase 4 → Phase 5. Level A alone already gives "both see the enemy aiming at the
same player" — the visible win — at low risk.
