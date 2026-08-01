# Pulsar — Unified Control Tool — Implementation Plan (overnight autonomous build)

**Goal:** Rebuild the lobby companion into one modern "Pulsar" desktop tool that controls everything: DB (Online official / Local), relay master-list (pick by ping/region, no IP), one-click local self-host, and an update checker.

**Architecture (LOCKED — see memory project_pulsar_tool.md):**
1. Official Online DB (STDB) = character home; relays = interchangeable brokers; local DB only for offline/LAN.
2. Relays self-register into an STDB `relay` table + heartbeat → GUI master-list (client-side ping, region, verified badge). Per-relay API key.
3. Updates from a PUBLIC `pulsar-releases` GitHub repo (tool + mod), checked on startup.
4. Local hosting = one-click bundled full stack (spacetime binary + module + relay).
5. Relay stays open-source; security via E2E-encrypted relayed traffic + verified badge + DB/master-list gatekeeper + per-relay API key.

**UI decision:** NEW Avalonia + FluentAvalonia (Fluent dark, MVVM) project `lobby/pulsar` → assembly `Pulsar.exe`. Keep the old WinForms `lobby/companion-gui` intact as fallback. Reuse `lobby/companion` daemon (file-handoff in %LOCALAPPDATA%\PULSAR), `lobby/relay`, `lobby/server/spacetimedb`.

**Tech:** .NET 8 (match daemon), Avalonia 11 + FluentAvalonia, CommunityToolkit.Mvvm. dotnet 10 SDK present; spacetime.exe present.

**Constraints / governance:** main repo LOCAL only (never push). Never stage extern/CommonLibSF, scripts/PulsarFurnSync.pex, lobby/_dist. German output. 8. Commit only working/compiling increments.

---

## What needs the USER (morning, cannot do autonomously)
- Visual sign-off on the look ("modern/aufgeräumt") — I can build+compile but not see it.
- Create public `pulsar-releases` GitHub repo + upload first releases.
- VPS deploy (spacetime publish — schema change for `relay` table wipes data) + relay redeploy.
- Per-relay API keys / official-DB registration secret.
- In-game 2-machine end-to-end test (only real validation).
- E2E-crypto needs a 2-machine test before public relay opening.

---

## Phase 1 — Pulsar Foundation (Avalonia shell) [BUILD FIRST]
**Files (create):** `lobby/pulsar/Pulsar.csproj`, `App.axaml(.cs)`, `Program.cs`, `ViewModels/*`, `Views/*`, `Services/DaemonBridge.cs`, `Services/Config.cs`.
- Avalonia + FluentAvalonia app, Fluent **dark** theme, single window + side-nav (Play / Servers / Host / Settings / About).
- **Connection model:** DB selector "Online (official)" (default) vs "Local". Online = `PULSAR_STDB_URI=wss://<official>` ; Local = launch local stack (Phase 4) → `http://127.0.0.1:3000`.
- **First-run onboarding:** name claim → pick DB (default Online) → done. Minimal, idiot-proof.
- **DaemonBridge service:** port the WinForms file-handoff (write `pulsar_cmd.json`, poll `pulsar_ack.json` / `pulsar_lobbies.json`, start/stop the `companion serve` daemon process, env vars). Pure C#, reused from MainForm.cs logic.
- Port host / join / join-by-code / claim-name flows into MVVM commands.
- **Acceptance:** `dotnet build lobby/pulsar` succeeds; app runs; can start daemon + host/join (same behavior as WinForms). COMMIT.

## Phase 2 — Relay Master-List
**Files:** `lobby/server/spacetimedb/Lib.cs` (+ `Relay` table + `RegisterRelay`/`RelayHeartbeat`/cleanup reducers + public list), `lobby/relay/Program.cs` (self-register loop + API key + region/name env), `lobby/companion/Program.cs` (subscribe `relay`, expose list to GUI via `pulsar_relays.json`), regen `module_bindings`, `lobby/pulsar` Servers page (list + client-side UDP ping + region + verified badge + "Connect").
- Relay env: `PULSAR_RELAY_NAME`, `PULSAR_RELAY_REGION`, `PULSAR_RELAY_APIKEY`, `PULSAR_RELAY_PUBLIC_HOST`. Heartbeat ~15s; module prunes stale (>60s) on read/timer.
- GUI pings each relay's UDP `--probe`-style RTT; sort by ping; verified flag from DB.
- **Acceptance:** module builds; daemon publishes relay list; GUI shows it. COMMIT. (Live verify = user/VPS.)

## Phase 5 — Update Checker [small, do early after P1]
**Files:** `lobby/pulsar/Services/UpdateChecker.cs` + a startup banner.
- On launch, GET `https://api.github.com/repos/<org>/pulsar-releases/releases/latest`; compare tool version (assembly) + mod version (a `mod_version` asset/tag) vs local (`pulsar_handoff`/a local version file). Non-blocking banner "New version: … [Download]". Configurable repo in Settings.
- **Acceptance:** builds; handles offline gracefully; shows banner against a configurable repo. COMMIT.

## Phase 4 — One-Click Local Stack
**Files:** `lobby/pulsar/Services/LocalStack.cs`, bundle plan for `spacetime.exe` + prebuilt module wasm + `PulsarRelay.exe`.
- "Host locally / LAN" → spin up: `spacetime start` (local) → publish module (or use prebuilt) → start local relay → point daemon at `http://127.0.0.1:3000` + local relay. Stop cleanly on exit.
- Binary bundling: reference spacetime.exe path (configurable; bundle in release zip). Code written; actual bundling = release packaging (user).
- **Acceptance:** code builds; starts/stops local processes on this machine. COMMIT.

## Phase 3 — E2E Encryption (relay traffic) [design + scaffold; needs 2-machine test]
- Verify ENet encryption status in the DLL (likely none). Design: symmetric session key derived at lobby create/join (shared via STDB grant, never through the relay), applied as an ENet packet transform (compress/encrypt callback) on both peers. Relay only sees ciphertext.
- Build scaffolding + doc; full enable gated behind a 2-machine test (user). COMMIT design + scaffold.

---

## Execution order: P1 → P5 → P2 → P4 → P3. Commit each working increment locally (no push). Morning: write handoff in this file's "STATUS" + a chat report.

## STATUS (live)
- 2026-06-22 night: plan written.
- **Phase 2 backend DONE + committed :** STDB Relay table + RegisterRelay/RelayHeartbeat/PruneRelays, relay self-register (PULSAR_RELAY_REGISTER=1), daemon writes pulsar_relays.json. All compiles. (GUI Servers-page still TODO — needs Phase 1.)
- **Phase 3 finding (E2E):** ENet traffic is currently PLAINTEXT (no enet_host_compress / no encrypt / no checksum). Hook points: encrypt payload before `enet_packet_create` (NetworkHost.cpp ~659-661 Send + ~735-736 large-data ch1) and decrypt after receive in the enet_host_service loop. Per-session symmetric key shared via the STDB RelayGrant (never through the relay). Full impl gated behind a 2-machine test (user).
- **Phase 1 (Avalonia GUI shell) DONE + committed :** lobby/pulsar, FluentAvalonia 2.4, MVVM, dark, side-nav, DaemonBridge ported. Builds clean.
- **Phases 2-GUI + 5 + 4 building in background** (Servers master-list page w/ ICMP ping + Connect; Update-checker banner vs pulsar-releases; One-Click Local-Stack spacetime+relay process mgmt). Awaiting completion → verify build → commit.
- REMAINING for the human (morning): visual sign-off; create public pulsar-releases repo (+ fix UpdateRepo default org); VPS `spacetime publish` (schema wipe for Relay table) + relay operators set PULSAR_RELAY_* envs + API keys; 2-machine in-game test; Phase 3 E2E full enable (design done, hook points noted).
