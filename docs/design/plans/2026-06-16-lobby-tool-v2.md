# Lobby Tool v2 Implementation Plan


**Goal:** Make `PulsarLobby.exe` English, idiot-proof, and DDoS-safe — unique claimed player names, one Server field with auto-relay + history, no public host IP, and QoL guardrails.

**Architecture:** Module enforces unique `DisplayName` and stores no host IP in relay mode. Companion gains a `register` command (with ack) and stops transmitting the host IP. The GUI is rewritten: one Server field (auto-derives relay), persisted config/history (`lobbygui.json`), a claim-name flow, no IP column, and connect→claim→host/join gating.

**Tech Stack:** C# (.NET 8 WinForms + console + WASM module), SpacetimeDB 2.5.0.

**Spec:** `docs/design/specs/2026-06-16-lobby-tool-v2-design.md`

---

## File Structure
- Modify `lobby/server/spacetimedb/Lib.cs` — unique DisplayName, RegisterAccount reject, CreateLobby host-IP blanking.
- Modify `lobby/companion/Program.cs` — `register` cmd + ack, host IP not transmitted, `lobbies.json` without IP, LAN auto-detect.
- Rewrite `lobby/companion-gui/MainForm.cs` — the v2 UI.
- (Deploy) publish self-contained → NAS `LobbyTools`, push private repo.

---

### Task 1: Module — unique names + no public host IP

**Files:** Modify `lobby/server/spacetimedb/Lib.cs`

- [ ] **Step 1: Make DisplayName unique**

In the `Account` struct, change `public string DisplayName;` to:
```csharp
        [Unique]
        public string    DisplayName;
```

- [ ] **Step 2: Reject taken names in RegisterAccount**

Replace the body of `RegisterAccount` (after the length check, before the `if (...Find(ctx.Sender))`) by inserting the uniqueness check:
```csharp
        displayName = (displayName ?? "").Trim();
        if (displayName.Length < 1 || displayName.Length > MAX_NAME_LEN)
            throw new Exception("displayName length must be 1.." + MAX_NAME_LEN);

        // Unique: reject a name already held by a DIFFERENT identity (your own rename is fine).
        if (ctx.Db.Account.DisplayName.Find(displayName) is Account taken && taken.Identity != ctx.Sender)
            throw new Exception("name taken");
```
(The rest of RegisterAccount — the upsert by identity — stays unchanged.)

- [ ] **Step 3: Blank host IP in relay mode (CreateLobby)**

In `CreateLobby`, replace the `relaySession` validation + the `IsValidHostAddr` check with:
```csharp
        relaySession = (relaySession ?? "").Trim();
        if (relaySession.Length != 0 && relaySession.Length != 32)
            throw new Exception("relaySession must be empty or 32 hex chars");
        // Relay mode → never store a public host IP (DDoS-safe). Direct/LAN mode needs one.
        if (relaySession.Length == 0)
        {
            if (!IsValidHostAddr(hostAddr))
                throw new Exception("invalid hostAddr (expected host:port)");
        }
        else
        {
            hostAddr = "";
        }
```
(The `ctx.Db.Lobby.Insert(...)` below already uses `hostAddr`/`relaySession`.)

- [ ] **Step 4: Build + republish locally**
```bash
cd lobby/server/spacetimedb && dotnet build
"/c/Users/r/AppData/Local/SpacetimeDB/spacetime.exe" publish pulsar-lobby -s local -p . --delete-data -y
```
Expected: build 0 errors; publish succeeds.

- [ ] **Step 5: Verify uniqueness (two identities)**
```bash
spacetime call -s local pulsar-lobby register_account '"Alice"'                      # first claim ok
# second identity claiming "Alice" must fail; check via logs/sql that only one Alice exists
spacetime sql -s local pulsar-lobby "SELECT DisplayName FROM Account"
```
Expected: a duplicate claim throws "name taken" (reducer error in `spacetime logs`).

- [ ] **Step 6: Commit** — `feat(lobby/module): unique player names + no public host IP in relay`

---

### Task 2: Companion — register command, no host IP

**Files:** Modify `lobby/companion/Program.cs`

- [ ] **Step 1: Pending-register field + LAN helper**

Add near the other static fields (under `RelayAddr`):
```csharp
    private static string s_pendingRegister = "";

    private static string GuessLanAddr()
    {
        try
        {
            foreach (var ip in System.Net.Dns.GetHostAddresses(System.Net.Dns.GetHostName()))
                if (ip.AddressFamily == System.Net.Sockets.AddressFamily.InterNetwork &&
                    !System.Net.IPAddress.IsLoopback(ip))
                    return ip + ":7777";
        }
        catch { /* fall through */ }
        return "127.0.0.1:7777";
    }
```

- [ ] **Step 2: register-result handler in the serve dispatch**

In the `case "serve":` block (after the `Subscribe(...)` call), add a one-time reducer handler:
```csharp
                c.Reducers.OnRegisterAccount += (ectx, dn) =>
                {
                    if (s_pendingRegister.Length == 0) return;
                    var cmd = s_pendingRegister; s_pendingRegister = "";
                    bool ok = ectx.Event.Status is SpacetimeDB.Status.Committed;
                    WriteAck(cmd, ok, ok ? null : "name taken or invalid", null);
                    Console.WriteLine($"[companion] serve: register '{dn}' -> {(ok ? "ok" : "REJECTED")}");
                };
```

- [ ] **Step 3: Handle the register command**

In `HandleCommandFile`, add a branch before the `else` (unknown action):
```csharp
            else if (action == "register")
            {
                string name = root.TryGetProperty("name", out var nm) ? (nm.GetString() ?? "") : "";
                s_pendingRegister = cmdId;             // ack written by OnRegisterAccount
                c.Reducers.RegisterAccount(name);
                Console.WriteLine($"[companion] serve: register requested '{name}'");
            }
```

- [ ] **Step 4: host — don't transmit the IP; auto-detect for direct fallback**

Replace the `action == "host"` branch body with:
```csharp
            if (action == "host")
            {
                string name = root.TryGetProperty("name", out var n) ? (n.GetString() ?? "Host") : "Host";
                uint max = root.TryGetProperty("max", out var m) ? m.GetUInt32() : 4;
                string session = RelayAddr.Length > 0 ? NewSessionHex() : "";
                string lanAddr = GuessLanAddr();              // local, for the direct/LAN handoff only
                string pubAddr = session.Length > 0 ? "" : lanAddr;  // never publish an IP in relay mode
                c.Reducers.RegisterAccount(name);             // idempotent (name already claimed)
                c.Reducers.CreateLobby(name, pubAddr, max, session);
                WriteHandoff("host", lanAddr, 0, RelayAddr, session);
                WriteAck(cmdId, true, null, null);
                Console.WriteLine($"[companion] serve: hosted '{name}' relay={RelayAddr}");
            }
```

- [ ] **Step 5: join — read addr+session server-side (GUI sends only lobbyId)**

Replace the `action == "join"` branch body with:
```csharp
            else if (action == "join")
            {
                ulong lobbyId = root.TryGetProperty("lobbyId", out var li) ? li.GetUInt64() : 0;
                string addr = "", session = "";
                foreach (var l in c.Db.Lobby.Iter())
                    if (l.LobbyId == lobbyId) { session = l.RelaySession ?? ""; addr = l.HostAddr ?? ""; break; }
                var mine = FindMyProfile(c);
                uint version = 0;
                if (mine is not null)
                {
                    File.WriteAllBytes(ProfileInPath, mine.Blob.ToArray());
                    version = mine.Version;
                }
                c.Reducers.JoinLobby(lobbyId);
                WriteHandoff("client", addr, version, RelayAddr, session);
                WriteAck(cmdId, true, null, null);
                Console.WriteLine($"[companion] serve: joined lobby {lobbyId} relay={RelayAddr}");
            }
```

- [ ] **Step 6: lobbies.json without host IP**

In `WriteLobbies`, remove the `hostAddr` field from the JSON builder (delete the `.Append(",\"hostAddr\":")...` line). Keep lobbyId, name, status, memberCount, maxPlayers.

- [ ] **Step 7: Regenerate bindings + build**
```bash
cd lobby/companion
"/c/Users/r/AppData/Local/SpacetimeDB/spacetime.exe" generate --lang csharp --out-dir module_bindings --module-path ../server/spacetimedb
dotnet build -c Release
```
Expected: 0 errors. (Account binding now has the unique DisplayName; no signature change to reducers.)

- [ ] **Step 8: Commit** — `feat(lobby/companion): register cmd + ack, no host IP, lobbies without IP`

---

### Task 3: GUI rewrite (English, server field, claim, persistence, no IP)

**Files:** Rewrite `lobby/companion-gui/MainForm.cs` (whole file below).

- [ ] **Step 1: Replace MainForm.cs with the v2 implementation**

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Drawing;
using System.IO;
using System.Linq;
using System.Text;
using System.Text.Json;
using System.Windows.Forms;

namespace PulsarCompanionGui;

// PULSAR Lobby — v2 front-end over the companion `serve` daemon (JSON protocol in
// %LOCALAPPDATA%\PULSAR). One Server field auto-derives the relay; player names are
// claimed (unique); no host IP is ever shown. Settings persist in lobbygui.json.
internal sealed class MainForm : Form
{
    private readonly string _sharedDir, _lobbiesPath, _cmdPath, _ackPath, _cfgPath;

    private Process? _daemon;
    private string _lastLobbiesJson = "";
    private string _lastAckCmdId = "";
    private string _pendingCmdId = "";
    private string _pendingKind = "";      // host | join | register
    private bool _launchAfterPending;
    private bool _nameClaimed;
    private bool _hosting;
    private Cfg _cfg = new();

    private sealed class Cfg
    {
        public string PlayerName { get; set; } = "Player";
        public List<string> Servers { get; set; } = new();
        public List<string> Relays { get; set; } = new();
        public string StarfieldPath { get; set; } = "";
        public bool AutoLaunch { get; set; } = true;
        public bool Advanced { get; set; } = false;
    }

    private readonly TextBox _name = new();
    private readonly Button _claim = new();
    private readonly Label _nameStatus = new();
    private readonly ComboBox _server = new();
    private readonly CheckBox _advanced = new();
    private readonly Label _relayLabel = new();
    private readonly ComboBox _relay = new();
    private readonly TextBox _starfield = new();
    private readonly Button _browse = new();
    private readonly CheckBox _launch = new();
    private readonly Button _start = new();
    private readonly Label _status = new();
    private readonly ListView _lobbies = new();
    private readonly Button _join = new();
    private readonly TextBox _hostName = new();
    private readonly NumericUpDown _max = new();
    private readonly Button _host = new();
    private readonly TextBox _log = new();
    private readonly System.Windows.Forms.Timer _timer = new();

    public MainForm()
    {
        _sharedDir = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData), "PULSAR");
        Directory.CreateDirectory(_sharedDir);
        _lobbiesPath = Path.Combine(_sharedDir, "pulsar_lobbies.json");
        _cmdPath     = Path.Combine(_sharedDir, "pulsar_cmd.json");
        _ackPath     = Path.Combine(_sharedDir, "pulsar_ack.json");
        _cfgPath     = Path.Combine(_sharedDir, "lobbygui.json");

        LoadCfg();
        BuildUi();
        ApplyCfgToUi();
        SetState();

        _timer.Interval = 1000;
        _timer.Tick += (_, _) => Poll();
        _timer.Start();
        FormClosing += (_, _) => StopDaemon();
        Log("Ready. Start the daemon, then claim your name.");
    }

    private void BuildUi()
    {
        Text = "PULSAR Lobby";
        ClientSize = new Size(700, 560);
        MinimumSize = new Size(640, 500);
        Font = new Font("Segoe UI", 9f);

        var lblName = new Label { Text = "Player name:", Location = new Point(12, 16), AutoSize = true };
        _name.Location = new Point(95, 12); _name.Width = 160;
        _claim.Location = new Point(265, 11); _claim.Size = new Size(90, 26); _claim.Text = "Claim";
        _claim.Click += (_, _) => DoClaim();
        _nameStatus.Location = new Point(365, 16); _nameStatus.AutoSize = true; _nameStatus.Text = "not claimed";
        _nameStatus.ForeColor = Color.Gray;

        var lblServer = new Label { Text = "Server:", Location = new Point(12, 50), AutoSize = true };
        _server.Location = new Point(95, 46); _server.Width = 360; _server.Anchor = AnchorAll();
        _server.DropDownStyle = ComboBoxStyle.DropDown;
        _advanced.Location = new Point(465, 47); _advanced.AutoSize = true; _advanced.Text = "Advanced"; _advanced.Anchor = AnchorTR();
        _advanced.CheckedChanged += (_, _) => { _cfg.Advanced = _advanced.Checked; RelayVisible(_advanced.Checked); SaveCfg(); };

        _relayLabel.Text = "Relay:"; _relayLabel.Location = new Point(12, 80); _relayLabel.AutoSize = true;
        _relay.Location = new Point(95, 76); _relay.Width = 360; _relay.Anchor = AnchorAll();
        _relay.DropDownStyle = ComboBoxStyle.DropDown;

        var lblSf = new Label { Text = "Starfield:", Location = new Point(12, 112), AutoSize = true };
        _starfield.Location = new Point(95, 108); _starfield.Width = 470; _starfield.Anchor = AnchorAll();
        _browse.Location = new Point(573, 107); _browse.Size = new Size(90, 24); _browse.Text = "Browse"; _browse.Anchor = AnchorTR();
        _browse.Click += (_, _) => { using var d = new OpenFileDialog { Filter = "EXE|*.exe" }; if (d.ShowDialog(this) == DialogResult.OK) { _starfield.Text = d.FileName; SaveCfg(); } };
        _launch.Location = new Point(95, 138); _launch.AutoSize = true; _launch.Text = "Launch Starfield after host/join";

        _start.Location = new Point(380, 134); _start.Size = new Size(130, 28); _start.Text = "Start daemon"; _start.Anchor = AnchorTR();
        _start.Click += (_, _) => ToggleDaemon();
        _status.Location = new Point(516, 140); _status.AutoSize = true; _status.Text = "disconnected"; _status.ForeColor = Color.Firebrick; _status.Anchor = AnchorTR();

        var lblList = new Label { Text = "Open lobbies:", Location = new Point(12, 172), AutoSize = true };
        _lobbies.Location = new Point(12, 192); _lobbies.Size = new Size(560, 210);
        _lobbies.View = View.Details; _lobbies.FullRowSelect = true; _lobbies.GridLines = true;
        _lobbies.MultiSelect = false; _lobbies.HideSelection = false; _lobbies.Anchor = AnchorTLBR();
        _lobbies.Columns.Add("Name", 360);
        _lobbies.Columns.Add("Players", 80);
        _lobbies.Columns.Add("Status", 100);
        _lobbies.DoubleClick += (_, _) => DoJoin();
        _join.Location = new Point(582, 192); _join.Size = new Size(100, 30); _join.Text = "Join"; _join.Anchor = AnchorTR();
        _join.Click += (_, _) => DoJoin();

        var grp = new GroupBox { Text = "Host a new lobby", Location = new Point(12, 414), Size = new Size(676, 70) };
        grp.Anchor = AnchorBLR();
        var lblHN = new Label { Text = "Lobby name:", Location = new Point(12, 28), AutoSize = true };
        _hostName.Location = new Point(95, 24); _hostName.Width = 230; _hostName.Text = "My Lobby";
        var lblMax = new Label { Text = "Max:", Location = new Point(345, 28), AutoSize = true };
        _max.Location = new Point(385, 24); _max.Width = 50; _max.Minimum = 2; _max.Maximum = 8; _max.Value = 4;
        _host.Location = new Point(470, 22); _host.Size = new Size(180, 30); _host.Text = "Host lobby";
        _host.Click += (_, _) => DoHostToggle();
        grp.Controls.AddRange(new Control[] { lblHN, _hostName, lblMax, _max, _host });

        _log.Location = new Point(12, 492); _log.Size = new Size(676, 60);
        _log.Multiline = true; _log.ReadOnly = true; _log.ScrollBars = ScrollBars.Vertical;
        _log.BackColor = Color.FromArgb(245, 245, 245); _log.Anchor = AnchorBLR();

        Controls.AddRange(new Control[]
        {
            lblName, _name, _claim, _nameStatus,
            lblServer, _server, _advanced, _relayLabel, _relay,
            lblSf, _starfield, _browse, _launch, _start, _status,
            lblList, _lobbies, _join, grp, _log
        });
    }

    private static AnchorStyles AnchorAll()  => AnchorStyles.Top | AnchorStyles.Left | AnchorStyles.Right;
    private static AnchorStyles AnchorTR()   => AnchorStyles.Top | AnchorStyles.Right;
    private static AnchorStyles AnchorTLBR() => AnchorStyles.Top | AnchorStyles.Bottom | AnchorStyles.Left | AnchorStyles.Right;
    private static AnchorStyles AnchorBLR()  => AnchorStyles.Bottom | AnchorStyles.Left | AnchorStyles.Right;

    // ── config ────────────────────────────────────────────────────────────────
    private void LoadCfg()
    {
        try { if (File.Exists(_cfgPath)) _cfg = JsonSerializer.Deserialize<Cfg>(File.ReadAllText(_cfgPath)) ?? new(); }
        catch { _cfg = new(); }
    }
    private void SaveCfg()
    {
        try { File.WriteAllText(_cfgPath, JsonSerializer.Serialize(_cfg, new JsonSerializerOptions { WriteIndented = true })); }
        catch { /* best-effort */ }
    }
    private void ApplyCfgToUi()
    {
        _name.Text = _cfg.PlayerName;
        _server.Items.Clear(); _server.Items.AddRange(_cfg.Servers.ToArray());
        if (_cfg.Servers.Count > 0) _server.Text = _cfg.Servers[0];
        _relay.Items.Clear(); _relay.Items.AddRange(_cfg.Relays.ToArray());
        if (_cfg.Relays.Count > 0) _relay.Text = _cfg.Relays[0];
        _starfield.Text = _cfg.StarfieldPath.Length > 0 ? _cfg.StarfieldPath : GuessStarfieldExe();
        _launch.Checked = _cfg.AutoLaunch;
        _advanced.Checked = _cfg.Advanced;
        RelayVisible(_cfg.Advanced);
    }
    private void RelayVisible(bool on) { _relayLabel.Visible = on; _relay.Visible = on; }
    private static void Remember(List<string> list, string v)
    {
        v = v.Trim(); if (v.Length == 0) return;
        list.RemoveAll(x => string.Equals(x, v, StringComparison.OrdinalIgnoreCase));
        list.Insert(0, v);
        if (list.Count > 8) list.RemoveRange(8, list.Count - 8);
    }

    // ── daemon ────────────────────────────────────────────────────────────────
    private void ToggleDaemon()
    {
        if (_daemon is { HasExited: false }) { StopDaemon(); return; }
        var exe = GuessCompanionExe();
        if (!File.Exists(exe)) { MessageBox.Show(this, "PulsarCompanion.exe not found:\n" + exe, "PULSAR Lobby"); return; }

        var serverHost = ServerHost();
        if (serverHost.Length == 0) { MessageBox.Show(this, "Enter a server (e.g. my.host or wss://my.host).", "PULSAR Lobby"); return; }
        var name = _name.Text.Trim().Length > 0 ? _name.Text.Trim() : "Player";
        var stdbUri = serverHost.Contains("://") ? serverHost : "wss://" + serverHost;
        var relay   = _advanced.Checked && _relay.Text.Trim().Length > 0 ? _relay.Text.Trim() : HostOnly(serverHost) + ":7777";

        try
        {
            var psi = new ProcessStartInfo
            {
                FileName = exe, Arguments = "serve \"" + name + "\"",
                WorkingDirectory = Path.GetDirectoryName(exe) ?? _sharedDir,
                UseShellExecute = false, CreateNoWindow = true,
                RedirectStandardOutput = true, RedirectStandardError = true, RedirectStandardInput = true,
            };
            psi.Environment["PULSAR_STDB_URI"]  = stdbUri;
            psi.Environment["PULSAR_RELAY_ADDR"] = relay;
            _daemon = new Process { StartInfo = psi, EnableRaisingEvents = true };
            _daemon.OutputDataReceived += (_, e) => { if (e.Data != null) Log("daemon: " + e.Data); };
            _daemon.ErrorDataReceived  += (_, e) => { if (e.Data != null) Log("daemon! " + e.Data); };
            _daemon.Exited += (_, _) => BeginInvoke(() => { _nameClaimed = false; SetState(); Log("Daemon stopped."); });
            _daemon.Start(); _daemon.BeginOutputReadLine(); _daemon.BeginErrorReadLine();

            _cfg.PlayerName = name; Remember(_cfg.Servers, serverHost);
            if (_advanced.Checked) Remember(_cfg.Relays, relay);
            _cfg.StarfieldPath = _starfield.Text.Trim(); _cfg.AutoLaunch = _launch.Checked; SaveCfg(); ApplyCfgToUi();
            SetState();
            Log($"Daemon started (server={stdbUri}, relay={relay}). Now claim your name.");
        }
        catch (Exception ex) { Log("Start failed: " + ex.Message); }
    }

    private void StopDaemon()
    {
        try { if (_daemon is { HasExited: false }) { _daemon.Kill(true); _daemon.WaitForExit(2000); } } catch { }
        _daemon = null; _nameClaimed = false; _hosting = false;
    }

    private bool Connected => _daemon is { HasExited: false };

    private void SetState()
    {
        bool c = Connected;
        _start.Text = c ? "Stop daemon" : "Start daemon";
        _status.Text = c ? "connected" : "disconnected";
        _status.ForeColor = c ? Color.ForestGreen : Color.Firebrick;
        _name.Enabled = !c; _server.Enabled = !c; _advanced.Enabled = !c; _relay.Enabled = !c;
        _starfield.Enabled = !c; _browse.Enabled = !c;
        _claim.Enabled = c && !_nameClaimed;
        _join.Enabled = c && _nameClaimed && !_hosting;
        _host.Enabled = c && _nameClaimed;
        _host.Text = _hosting ? "Close lobby" : "Host lobby";
        if (_nameClaimed) { _nameStatus.Text = "you are '" + _name.Text.Trim() + "'"; _nameStatus.ForeColor = Color.ForestGreen; }
        else { _nameStatus.Text = c ? "claim your name" : "not claimed"; _nameStatus.ForeColor = Color.Gray; }
    }

    // ── actions ───────────────────────────────────────────────────────────────
    private void DoClaim()
    {
        if (!Connected) return;
        var name = _name.Text.Trim();
        if (name.Length < 1 || name.Length > 32) { MessageBox.Show(this, "Name must be 1..32 characters.", "PULSAR Lobby"); return; }
        Send("register", "{\"action\":\"register\",\"cmdId\":" + Esc(NewId(out var id)) + ",\"name\":" + Esc(name) + "}", id, false,
             $"Claiming '{name}' ...");
    }

    private void DoHostToggle()
    {
        if (_hosting) { _hosting = false; SetState(); Log("Lobby closed (stop the daemon to fully release)."); return; }
        if (!Connected || !_nameClaimed) return;
        var ln = _hostName.Text.Trim();
        if (ln.Length == 0) { MessageBox.Show(this, "Enter a lobby name.", "PULSAR Lobby"); return; }
        var json = "{\"action\":\"host\",\"cmdId\":" + Esc(NewId(out var id)) + ",\"name\":" + Esc(ln) + ",\"max\":" + (int)_max.Value + "}";
        Send("host", json, id, _launch.Checked, $"Hosting '{ln}' ...");
    }

    private void DoJoin()
    {
        if (!Connected || !_nameClaimed) return;
        if (_lobbies.SelectedItems.Count == 0 || _lobbies.SelectedItems[0].Tag is not ulong lobbyId)
        { MessageBox.Show(this, "Select a lobby first.", "PULSAR Lobby"); return; }
        var json = "{\"action\":\"join\",\"cmdId\":" + Esc(NewId(out var id)) + ",\"lobbyId\":" + lobbyId + "}";
        Send("join", json, id, _launch.Checked, $"Joining lobby {lobbyId} ...");
    }

    private void Send(string kind, string json, string cmdId, bool launchAfter, string logMsg)
    {
        try { _pendingKind = kind; _pendingCmdId = cmdId; _launchAfterPending = launchAfter; File.WriteAllText(_cmdPath, json); Log(logMsg); }
        catch (Exception ex) { Log("command write failed: " + ex.Message); _pendingCmdId = ""; }
    }

    // ── poll ──────────────────────────────────────────────────────────────────
    private void Poll() { RefreshLobbies(); CheckAck(); }

    private void RefreshLobbies()
    {
        string json;
        try { json = File.Exists(_lobbiesPath) ? File.ReadAllText(_lobbiesPath) : "[]"; } catch { return; }
        if (json == _lastLobbiesJson) return;
        _lastLobbiesJson = json;
        ulong sel = _lobbies.SelectedItems.Count > 0 && _lobbies.SelectedItems[0].Tag is ulong s ? s : 0;
        _lobbies.BeginUpdate(); _lobbies.Items.Clear();
        try
        {
            using var doc = JsonDocument.Parse(json);
            foreach (var e in doc.RootElement.EnumerateArray())
            {
                ulong id = e.TryGetProperty("lobbyId", out var v) ? v.GetUInt64() : 0;
                string nm = Str(e, "name"), st = Str(e, "status");
                long mc = e.TryGetProperty("memberCount", out var m) ? m.GetInt64() : 0;
                long mx = e.TryGetProperty("maxPlayers", out var x) ? x.GetInt64() : 0;
                var it = new ListViewItem(nm);
                it.SubItems.Add($"{mc}/{mx}");
                it.SubItems.Add(st);
                it.Tag = id;
                if (id == sel) it.Selected = true;
                _lobbies.Items.Add(it);
            }
            if (_lobbies.Items.Count == 0) { var it = new ListViewItem("No open lobbies — host one!"); it.ForeColor = Color.Gray; _lobbies.Items.Add(it); }
        }
        catch (Exception ex) { Log("lobbies parse error: " + ex.Message); }
        _lobbies.EndUpdate();
    }

    private void CheckAck()
    {
        if (!File.Exists(_ackPath)) return;
        string text; try { text = File.ReadAllText(_ackPath); } catch { return; }
        try
        {
            using var doc = JsonDocument.Parse(text);
            var root = doc.RootElement;
            string cmdId = Str(root, "cmdId");
            if (cmdId.Length == 0 || cmdId == _lastAckCmdId) return;
            _lastAckCmdId = cmdId;
            if (cmdId != _pendingCmdId) return;
            bool ok = root.TryGetProperty("ok", out var o) && o.GetBoolean();
            string err = Str(root, "error");
            string kind = _pendingKind; _pendingCmdId = ""; _pendingKind = "";
            if (kind == "register")
            {
                _nameClaimed = ok;
                if (ok) Log("Name claimed.");
                else { Log("Claim rejected: " + err); MessageBox.Show(this, "Name taken — choose another.", "PULSAR Lobby"); }
                SetState();
            }
            else if (ok)
            {
                if (kind == "host") { _hosting = true; SetState(); }
                Log(kind == "host" ? "Lobby hosted." : "Joined.");
                if (_launchAfterPending) LaunchGame();
            }
            else Log(kind + " failed: " + err);
            _launchAfterPending = false;
        }
        catch { /* partial write; retry */ }
    }

    private void LaunchGame()
    {
        var exe = _starfield.Text.Trim();
        if (!File.Exists(exe)) { Log("Starfield not found: " + exe); return; }
        try { Process.Start(new ProcessStartInfo { FileName = exe, UseShellExecute = true, WorkingDirectory = Path.GetDirectoryName(exe) ?? "" }); Log("Starfield launched."); }
        catch (Exception ex) { Log("Launch failed: " + ex.Message); }
    }

    // ── helpers ───────────────────────────────────────────────────────────────
    private string ServerHost() => _server.Text.Trim();
    private static string HostOnly(string s)
    {
        var h = s; int i = h.IndexOf("://"); if (i >= 0) h = h.Substring(i + 3);
        int slash = h.IndexOf('/'); if (slash >= 0) h = h.Substring(0, slash);
        int colon = h.IndexOf(':'); if (colon >= 0) h = h.Substring(0, colon);
        return h;
    }
    private void Log(string m) { if (InvokeRequired) { BeginInvoke(() => Log(m)); return; } _log.AppendText($"[{DateTime.Now:HH:mm:ss}] {m}{Environment.NewLine}"); }
    private static string Str(JsonElement e, string p) => e.TryGetProperty(p, out var v) && v.ValueKind == JsonValueKind.String ? (v.GetString() ?? "") : "";
    private static string Esc(string s) => "\"" + (s ?? "").Replace("\\", "\\\\").Replace("\"", "\\\"") + "\"";
    private static string NewId(out string id) { id = Guid.NewGuid().ToString("N"); return id; }

    private static string GuessCompanionExe()
    {
        var baseDir = AppContext.BaseDirectory;
        foreach (var rel in new[] { "PulsarCompanion.exe", Path.Combine("companion", "PulsarCompanion.exe") })
        { var c = Path.Combine(baseDir, rel); if (File.Exists(c)) return c; }
        var dir = baseDir;
        for (int i = 0; i < 8 && dir != null; i++)
        {
            foreach (var cfg in new[] { "Release", "Debug" })
            { var c = Path.Combine(dir, "companion", "bin", cfg, "net8.0", "PulsarCompanion.exe"); if (File.Exists(c)) return c; }
            dir = Directory.GetParent(dir)?.FullName;
        }
        return "PulsarCompanion.exe";
    }

    private static string GuessStarfieldExe()
    {
        foreach (var p in new[]
        {
            @"<Starfield>\sfse_loader.exe",
            @"C:\Program Files (x86)\Steam\steamapps\common\Starfield\sfse_loader.exe",
            @"C:\Program Files (x86)\Steam\steamapps\common\Starfield\Starfield.exe",
        })
            if (File.Exists(p)) return p;
        return @"<Starfield>\sfse_loader.exe";
    }
}
```

- [ ] **Step 2: Build the GUI**

Run: `cd lobby/companion-gui && dotnet build -c Release`
Expected: 0 errors.

- [ ] **Step 3: Commit** — `feat(lobby/gui): v2 — English, claim flow, server+relay history, no IP, guardrails`

---

### Task 4: Deploy + publish

- [ ] **Step 1: Publish self-contained + copy to NAS**
```bash
cd lobby
dotnet publish companion-gui/CompanionGui.csproj -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true -o _dist/LobbyTools
dotnet publish companion/Companion.csproj -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true -o _dist/LobbyTools/companion
cp -f _dist/LobbyTools/PulsarLobby.exe "<NAS>/deploy/LobbyTools/PulsarLobby.exe"
cp -f _dist/LobbyTools/companion/PulsarCompanion.exe "<NAS>/deploy/LobbyTools/companion/PulsarCompanion.exe"
```

- [ ] **Step 2: Sync the private repo (module changed)**
Copy `lobby/server/spacetimedb/Lib.cs` into the export `<local>/pulsar-lobby-server/lobby/server/spacetimedb/Lib.cs`, commit, `git push origin main`.

- [ ] **Step 3: VPS republish reminder (user)**
On the VPS: `git pull` then `spacetime publish pulsar-lobby -s pulsar-vps -p lobby/server/spacetimedb --delete-data -y` (unique DisplayName is a schema change → needs the reset). Both players use the v2 `LobbyTools`.

---

## Self-Review

**1. Spec coverage:** English (Task 3) ✓; unique name + claim (Task 1 module, Task 2 register cmd, Task 3 claim flow) ✓; one Server field + auto-relay + Advanced + history/persistence (Task 3 + lobbygui.json) ✓; hide host IP (Task 1 blank, Task 2 no transmit + lobbies.json, Task 3 no column) ✓; QoL guardrails/auto-detect/status/empty-state (Task 3 SetState/Guess/RefreshLobbies) ✓.

**2. Placeholder scan:** No TBD/TODO; full code per step. The `ectx.Event.Status is SpacetimeDB.Status.Committed` API is verified at Task 2 build; if the SDK exposes it differently, the build error pinpoints it (the only API-uncertain line).

**3. Type consistency:** cmd JSON keys (`action`,`cmdId`,`name`,`lobbyId`,`max`) match companion `HandleCommandFile`; `register` ack handled in both; GUI join sends only `lobbyId` (companion reads addr/session server-side); `CreateLobby(name,hostAddr,max,relaySession)` 4-arg matches Task 1; `WriteHandoff(role,hostAddr,version,relayAddr,relaySession)` matches existing signature; `Cfg` property names consistent across Load/Save/Apply.
