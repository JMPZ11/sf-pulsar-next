# Self-Host on Own IP + Local Character Backup Implementation Plan


**Goal:** Let a player self-host a co-op session on their own IP (LAN + internet), joinable by direct-IP invite, with self-host strictly private; plus a local character backup (auto rolling + export/import). See spec `docs/design/specs/2026-06-23-self-host-own-ip-and-local-backup-design.md`.

**Architecture:** Relay-only network exposure (DB stays host-local); direct-IP invite bypasses cloud lobby; backup operates on the existing JSON-v2 profile blob.

**Tech Stack:** C# / .NET 8 / Avalonia 11 / CommunityToolkit.Mvvm. Build: `dotnet build lobby/pulsar/Pulsar.csproj -c Release`.

---

## Status 2026-06-23 — all tasks code-complete + build-verified (tool 0/0, companion 0/0)

- **Task 1** NetworkAddress — DONE, unit-tested (18/18 in dist/selfhosttest).
- **Task 2** Companion direct verbs (hostDirect/joinDirect) — DONE, companion builds 0/0.
- **Task 3** Tool UI invite + Join-by-IP (DaemonBridge.HostDirect/JoinDirect, PlayViewModel, PlayView "Play directly" card) — DONE, tool builds 0/0. NEEDS: user GUI + 2-machine acceptance.
- **Task 4** Internet self-host — covered inline (the "Host over the internet" checkbox uses the public IP + states the UDP 7777 port-forward). Live reachability probe deferred (needs a helper endpoint) - documented.
- **Task 5** CharacterBackup — DONE, unit-tested.
- **Task 6** Backup wiring (auto-backup on each daemon auto-save) + Export/Import UI in Settings — DONE, tool builds 0/0. LIMITATION: import reuses the auto-upload path, so it uploads to the DB only while connected with auto-save on (both default while playing). NEEDS: user acceptance.
- **Task 7** Bundling — license check DONE (SpacetimeDB BSL 1.1 permits redistribution + one local instance per user; ship its LICENSE/NOTICE). LocalStack bundled-binary + bundled-module locators DONE (tool 0/0); relay start made non-fatal (direct self-host needs no relay). package_release.ps1 bundles spacetime + module + license (NOT run here; executes at packaging on a machine with the binary).

NEEDS USER: GUI/visual acceptance, real 2-machine direct-IP join (LAN + internet+port-forward), export/import round-trip in-game, and a clean-machine package smoke after running package_release.ps1.

---

### Task 1: NetworkAddress service (LAN IP + invite parse) [pure logic] — DONE (build-verified 2026-06-23)

**Files:**
- Create: `lobby/pulsar/Services/NetworkAddress.cs`

- [ ] **Step 1: LAN IP detection.** `GetLanIpv4()` enumerates `NetworkInterface.GetAllNetworkInterfaces()`, picks an Up, non-loopback, non-virtual adapter with a private IPv4 (192.168/10/172.16-31), skips APIPA (169.254). Returns null if none.
- [ ] **Step 2: Invite parse/format.** `string FormatInvite(string ip, int port)` returns `pulsar://join/<ip>:<port>`. `bool TryParseInvite(string text, out string ip, out int port)` accepts `pulsar://join/ip:port` AND bare `ip:port`; validates IPv4/hostname + port range.
- [ ] **Step 3: Public IP (async).** `Task<string?> GetPublicIpAsync(ct)` does a single HTTPS GET to a plain text "what is my IP" endpoint with a 3s timeout; returns null on failure. Cache the result in the instance.
- [ ] **Step 4: Build.** `dotnet build lobby/pulsar/Pulsar.csproj -c Release`. Expected: 0 errors.

> NOTE: Tasks 2-3 revised per the spec UPDATE 2026-06-23 to the "pure direct" model.
> The DLL handoff already supports a direct connect (`host_addr`, empty `relay_addr`/
> `relay_session`); no relay and no session are needed for self-host. The invite is just
> `ip:port`.

### Task 2: Companion direct host/join verbs (bypass the DB lobby)

**Files:**
- Modify: `lobby/companion/Program.cs`

- [ ] **Step 1: host-direct.** A serve command (or CLI verb) that stages the host's own profile and writes handoff `role=host` with an empty relay/session (DLL StartHosts on the game port). No CreateLobby, nothing written to the DB lobby table. Returns an ack.
- [ ] **Step 2: join-direct <ip:port>.** Pulls the joiner's own profile (their current DB, online or local) -> `profile-in`, writes handoff `role=client, host_addr=<ip:port>`, empty relay/session. No JoinLobby/DB resolve. Validates ip:port. Returns an ack.
- [ ] **Step 3:** `dotnet build lobby/companion/PulsarCompanion.csproj -c Release`. Expected: 0 errors. Headless smoke: run the verb, confirm the handoff JSON has role + host_addr and empty relay fields.

### Task 3: Tool UI - reachable-IP invite (host) + Join-by-IP (joiner)

**Files:**
- Modify: `lobby/pulsar/ViewModels/PlayViewModel.cs`
- Modify: `lobby/pulsar/Views/PlayView.axaml`
- Modify: `lobby/pulsar/Services/DaemonBridge.cs` (wire the two direct verbs)

- [ ] **Step 1: DaemonBridge.** Add `HostDirect(launchAfter)` and `JoinDirect(string ip, int port, launchAfter)` that write the new companion commands (mirrors existing Host/Join).
- [ ] **Step 2: Host invite.** When self-hosting, compute the reachable address via `NetworkAddress` (LAN by default; public IP in an expander when internet self-host is on), expose `InviteString = FormatInvite(ip, gamePort)` + a Copy command.
- [ ] **Step 3: Join by IP.** `JoinIpText` + `JoinByIpCommand`: `NetworkAddress.TryParseInvite` -> `DaemonBridge.JoinDirect` -> launch. No cloud-lobby lookup.
- [ ] **Step 4:** Build green. Manual acceptance (user): host shows invite, joiner pastes it, connects + plays.

### Task 4: Internet self-host (Advanced) + reachability hint

**Files:**
- Modify: `lobby/pulsar/ViewModels/SettingsViewModel.cs` (or PlayViewModel Advanced section)
- Modify: `lobby/pulsar/Views/*` (Advanced panel)

- [ ] **Step 1:** Advanced toggle "Host over the internet (port-forward required)" with a one-line warning. When on, the invite uses the public IP.
- [ ] **Step 2:** Show detected public IP + a port-forward line ("forward UDP <port> to this PC"). Reachability live-probe deferred to a later step (needs a helper endpoint); ship the manual checklist now.
- [ ] **Step 3:** Build green.

### Task 5: CharacterBackup service [pure logic] — DONE (build-verified 2026-06-23)

**Files:**
- Create: `lobby/pulsar/Services/CharacterBackup.cs`

- [ ] **Step 1: Rolling backup.** `BackupBlob(string identity, string blob)` writes `%LOCALAPPDATA%\PULSAR\backups\<identity>\char-<unixtime>.json`, then prunes to the newest 10. Identity = STDB identity short id, or "local".
- [ ] **Step 2: Export.** `Export(string blob, string destPath, Header h)` writes a `.pulsarchar` = JSON `{ header:{name,level,ts,mode}, blob:"..." }`.
- [ ] **Step 3: Import.** `TryReadExport(string path, out Header h, out string blob)` parses + validates a `.pulsarchar`.
- [ ] **Step 4:** Build green.

### Task 6: Wire backup into auto-save + Backup UI

**Files:**
- Modify: `lobby/companion/Program.cs` (call BackupBlob on each profile-out upload) OR `lobby/pulsar` watcher
- Create/Modify: a Backup view + ViewModel (or a section in Settings)

- [ ] **Step 1:** On each profile blob produced (same hook as auto-save), call CharacterBackup.BackupBlob.
- [ ] **Step 2:** UI: "Export character" (file save dialog), "Import character" (file open dialog -> show header -> confirm -> UpsertProfile via existing path). Retention shown/configurable (default 10).
- [ ] **Step 3:** Build green. Manual acceptance (user): export, wipe/local, import, character restored.

### Task 7: Bundling for distribution

**Files:**
- Modify: `tools/package_release.ps1`
- Modify: `lobby/pulsar/Services/LocalStack.cs` (bundled-binary locator first)

- [ ] **Step 0 (prerequisite):** Verify SpacetimeDB redistribution license. If not permitted -> fall back to a guided one-time download on the Setup page (like SFSE) instead of bundling. Record the decision here.
- [ ] **Step 1:** Lay out `spacetime(.exe)` + prebuilt module + `PulsarRelay.exe` next to the tool in the release ZIP.
- [ ] **Step 2:** LocalStack locates the bundled binary first, then Settings path, then dev walk-up. Resolve the `spacetime publish` flag TODO against the bundled CLI (or ship a prebuilt module to skip publish).
- [ ] **Step 3:** Build green; package smoke (user, on a clean machine).
