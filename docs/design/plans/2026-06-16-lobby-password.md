# Lobby Password Implementation Plan


**Goal:** Real, non-bypassable optional lobby password — the relay session is private and released only after a password-verified join.

**Architecture:** RelaySession moves from the public Lobby table into a private `LobbyRelay` table (+password hash); a per-user `MyRelay` view releases the session only to the host or verified members. Companion hashes client-side and the join becomes async (read the session from `my_relay` after `JoinLobby` verifies the password). Relay/DLL protocol unchanged.

**Tech Stack:** C# (.NET 8 WASM module + console + WinForms), SpacetimeDB 2.5.0.

**Spec:** `docs/design/specs/2026-06-16-lobby-password-design.md`

---

### Task 1: Module — private session + password verify

**Files:** Modify `lobby/server/spacetimedb/Lib.cs`

- [ ] **Step 1: Lobby — drop RelaySession, add HasPassword**

In `Lobby`, replace `public string RelaySession; ... public string Name;` region — remove the RelaySession line and add HasPassword:
```csharp
        public string    HostAddr;     // "ip:port" (direct/LAN only; "" in relay)
        public bool      HasPassword;  // public flag → browser shows a lock
        public string    Name;
```

- [ ] **Step 2: Private LobbyRelay table + RelayInfo type + MyRelay view**

Add after the `Lobby` struct (before `LobbyMember`):
```csharp
    [SpacetimeDB.Table(Accessor = "LobbyRelay")]   // PRIVATE — session + pw hash never reach clients directly
    public partial struct LobbyRelay
    {
        [PrimaryKey]
        public ulong  LobbyId;
        public string Session;   // relay session (32 hex) or "" for direct lobbies
        public string Hash;      // password hash (64 hex) or "" for open lobbies
    }

    [SpacetimeDB.Type]
    public partial struct RelayInfo { public ulong LobbyId; public string Session; }
```
Add after the `MyProfile` view:
```csharp
    // Per-user: a member (or the host) of a lobby may read its relay session.
    // Non-members never see it → a password actually gates relay access.
    [SpacetimeDB.View(Accessor = "MyRelay", Public = true)]
    public static List<RelayInfo> MyRelay(ViewContext ctx)
    {
        var result = new List<RelayInfo>();
        foreach (var lr in ctx.Db.LobbyRelay.Iter())
        {
            if (lr.Session.Length == 0) continue;
            bool allowed = false;
            if (ctx.Db.Lobby.LobbyId.Find(lr.LobbyId) is Lobby lob && lob.HostIdentity == ctx.Sender)
                allowed = true;
            else
                foreach (var m in ctx.Db.LobbyMember.LobbyId.Filter(lr.LobbyId))
                    if (m.MemberIdentity == ctx.Sender) { allowed = true; break; }
            if (allowed) result.Add(new RelayInfo { LobbyId = lr.LobbyId, Session = lr.Session });
        }
        return result;
    }
```

- [ ] **Step 3: CreateLobby — password hash + private relay row**

Replace `CreateLobby` (signature + body) with:
```csharp
    [SpacetimeDB.Reducer]
    public static void CreateLobby(ReducerContext ctx, string name, string hostAddr,
                                   uint maxPlayers, string relaySession, string passwordHash)
    {
        name = (name ?? "").Trim();
        if (name.Length < 1 || name.Length > MAX_LOBBY_NAME)
            throw new Exception("lobby name length must be 1.." + MAX_LOBBY_NAME);
        if (maxPlayers < 1 || maxPlayers > MAX_PLAYERS_CAP)
            throw new Exception("maxPlayers must be 1.." + MAX_PLAYERS_CAP);
        relaySession = (relaySession ?? "").Trim();
        if (relaySession.Length != 0 && relaySession.Length != 32)
            throw new Exception("relaySession must be empty or 32 hex chars");
        passwordHash = (passwordHash ?? "").Trim();
        if (passwordHash.Length != 0 && passwordHash.Length != 64)
            throw new Exception("passwordHash must be empty or 64 hex chars");
        if (relaySession.Length == 0)
        {
            if (!IsValidHostAddr(hostAddr))
                throw new Exception("invalid hostAddr (expected host:port)");
        }
        else
        {
            hostAddr = "";   // relay mode: never publish a host IP
        }

        var row = ctx.Db.Lobby.Insert(new Lobby
        {
            LobbyId      = 0,
            HostIdentity = ctx.Sender,
            HostAddr     = hostAddr,
            HasPassword  = passwordHash.Length > 0,
            Name         = name,
            Status       = "open",
            MaxPlayers   = maxPlayers,
            MemberCount  = 0,
            CreatedAt    = ctx.Timestamp,
        });
        if (relaySession.Length > 0 || passwordHash.Length > 0)
            ctx.Db.LobbyRelay.Insert(new LobbyRelay { LobbyId = row.LobbyId, Session = relaySession, Hash = passwordHash });
    }
```

- [ ] **Step 4: JoinLobby — verify password**

In `JoinLobby`, after the `if (lobby.Status != "open") throw ...;` line, add the password check (signature also gains `passwordHash`):
```csharp
    public static void JoinLobby(ReducerContext ctx, ulong lobbyId, string passwordHash)
    {
        var lobby = ctx.Db.Lobby.LobbyId.Find(lobbyId)
            ?? throw new Exception("lobby not found");
        if (lobby.Status != "open")
            throw new Exception("lobby not open");
        if (lobby.HasPassword)
        {
            passwordHash = (passwordHash ?? "").Trim();
            if (ctx.Db.LobbyRelay.LobbyId.Find(lobbyId) is not LobbyRelay lr || lr.Hash != passwordHash)
                throw new Exception("wrong password");
        }
```
(The rest of JoinLobby — member count, insert, MemberCount update — is unchanged.)

- [ ] **Step 5: HandleDisconnect — drop the LobbyRelay row**

In `HandleDisconnect`, inside the `foreach (var lobby in ctx.Db.Lobby.HostIdentity.Filter(ctx.Sender).ToList())` loop, after closing the lobby, add:
```csharp
            ctx.Db.LobbyRelay.LobbyId.Delete(lobby.LobbyId);
```

- [ ] **Step 6: Build + republish + CLI test**
```bash
cd lobby/server/spacetimedb && dotnet build
"/c/Users/r/AppData/Local/SpacetimeDB/spacetime.exe" publish pulsar-lobby -s local -p . --delete-data -y
```
Expected: 0 errors; publish OK. (Functional join/verify is tested end-to-end in Task 2.)

- [ ] **Step 7: Commit** — `feat(lobby/module): private relay session + lobby password verify`

---

### Task 2: Companion — hash, async verified join

**Files:** Modify `lobby/companion/Program.cs`

- [ ] **Step 1: HashPw + pending-join state**

Add near the other static helpers:
```csharp
    private const string PW_SALT = "PULSAR-lobby-v1:";
    private static string HashPw(string pw)
    {
        pw = (pw ?? "").Trim();
        if (pw.Length == 0) return "";
        using var sha = System.Security.Cryptography.SHA256.Create();
        var h = sha.ComputeHash(System.Text.Encoding.UTF8.GetBytes(PW_SALT + pw));
        return Convert.ToHexString(h);   // 64 hex
    }

    // Async join state: JoinLobby verifies the password, then we read the session
    // from the my_relay view (relay mode) before writing the handoff.
    private static string  s_joinCmd = "";       // cmdId awaiting a join result
    private static ulong   s_joinLobby = 0;
    private static string  s_joinDirectAddr = ""; // non-empty = direct lobby (no session needed)
    private static uint    s_joinVersion = 0;
    private static bool    s_joinAwaitSession = false;
    private static long    s_joinDeadline = 0;    // GetTickCount64 deadline for the session
```

- [ ] **Step 2: Subscribe my_relay + OnJoinLobby handler (serve dispatch)**

In `case "serve":`, change the Subscribe to include my_relay:
```csharp
                c.SubscriptionBuilder()
                    .OnApplied(_ => Console.WriteLine("[companion] serve subscribed (lobby + my_profile + my_relay)"))
                    .Subscribe(new[] { "SELECT * FROM lobby", "SELECT * FROM my_profile", "SELECT * FROM my_relay" });
```
And after the existing `OnRegisterAccount` handler, add the join-result handler:
```csharp
                c.Reducers.OnJoinLobby += (ectx, lid, pwh) =>
                {
                    if (s_joinCmd.Length == 0 || lid != s_joinLobby) return;
                    if (ectx.Event.Status is not SpacetimeDB.Status.Committed)
                    {
                        WriteAck(s_joinCmd, false, "wrong password", null);
                        Console.WriteLine($"[companion] serve: join {lid} REJECTED");
                        s_joinCmd = ""; return;
                    }
                    if (s_joinDirectAddr.Length > 0)   // direct lobby — no relay session
                    {
                        WriteHandoff("client", s_joinDirectAddr, s_joinVersion, RelayAddr, "");
                        WriteAck(s_joinCmd, true, null, null);
                        Console.WriteLine($"[companion] serve: joined {lid} (direct)");
                        s_joinCmd = "";
                    }
                    else { s_joinAwaitSession = true; s_joinDeadline = GetMs() + 6000; }
                };
```
Add a small time helper near HashPw:
```csharp
    private static long GetMs() => Environment.TickCount64;
```

- [ ] **Step 3: serve loop — resolve the session for an awaited join**

In `Main`, inside the `if (cmd == "serve")` loop body (after `HandleCommandFile(conn);`), add:
```csharp
                    ResolveAwaitedJoin(conn);
```
Add the method:
```csharp
    private static void ResolveAwaitedJoin(DbConnection c)
    {
        if (!s_joinAwaitSession) return;
        string session = "";
        foreach (var r in c.Db.MyRelay.Iter())
            if (r.LobbyId == s_joinLobby) { session = r.Session; break; }
        if (session.Length > 0)
        {
            WriteHandoff("client", "", s_joinVersion, RelayAddr, session);
            WriteAck(s_joinCmd, true, null, null);
            Console.WriteLine($"[companion] serve: joined {s_joinLobby} (relay session resolved)");
            s_joinAwaitSession = false; s_joinCmd = "";
        }
        else if (GetMs() > s_joinDeadline)
        {
            WriteAck(s_joinCmd, false, "session unavailable", null);
            Console.WriteLine($"[companion] serve: join {s_joinLobby} — session timeout");
            s_joinAwaitSession = false; s_joinCmd = "";
        }
    }
```

- [ ] **Step 4: host — pass the password hash**

In the `action == "host"` branch, read the password and pass its hash:
```csharp
                string name = root.TryGetProperty("name", out var n) ? (n.GetString() ?? "Host") : "Host";
                uint max = root.TryGetProperty("max", out var m) ? m.GetUInt32() : 4;
                string pwHash = HashPw(root.TryGetProperty("password", out var pw) ? (pw.GetString() ?? "") : "");
                string session = RelayAddr.Length > 0 ? NewSessionHex() : "";
                string lanAddr = GuessLanAddr();
                string pubAddr = session.Length > 0 ? "" : lanAddr;
                c.Reducers.RegisterAccount(name);
                c.Reducers.CreateLobby(name, pubAddr, max, session, pwHash);
                WriteHandoff("host", lanAddr, 0, RelayAddr, session);
                WriteAck(cmdId, true, null, null);
                Console.WriteLine($"[companion] serve: hosted '{name}' pw={(pwHash.Length>0)} relay={RelayAddr}");
```

- [ ] **Step 5: join — async, verified**

Replace the `action == "join"` branch body with:
```csharp
                ulong lobbyId = root.TryGetProperty("lobbyId", out var li) ? li.GetUInt64() : 0;
                string pwHash = HashPw(root.TryGetProperty("password", out var jp) ? (jp.GetString() ?? "") : "");
                string addr = "";
                foreach (var l in c.Db.Lobby.Iter())
                    if (l.LobbyId == lobbyId) { addr = l.HostAddr ?? ""; break; }
                var mine = FindMyProfile(c);
                uint version = 0;
                if (mine is not null) { File.WriteAllBytes(ProfileInPath, mine.Blob.ToArray()); version = mine.Version; }
                s_joinCmd = cmdId; s_joinLobby = lobbyId; s_joinDirectAddr = addr; s_joinVersion = version;
                s_joinAwaitSession = false;
                c.Reducers.JoinLobby(lobbyId, pwHash);   // ack written by OnJoinLobby / ResolveAwaitedJoin
                Console.WriteLine($"[companion] serve: join requested {lobbyId}");
```

- [ ] **Step 6: lobbies.json — add hasPassword (still no IP/session)**

In `WriteLobbies`, add a `hasPassword` field:
```csharp
              .Append(",\"status\":").Append(JsonEsc(l.Status))
              .Append(",\"hasPassword\":").Append(l.HasPassword ? "true" : "false")
              .Append(",\"memberCount\":").Append(l.MemberCount)
```

- [ ] **Step 7: Update CLI join call site**

The CLI `join` case calls `c.Reducers.JoinLobby(lobbyId)` (1-arg) — update to `c.Reducers.JoinLobby(lobbyId, "")` and its `WriteHandoff(...)` already passes "" for relay; it reads `lobby.HostAddr` for addr. Leave CLI direct-mode behaviour; just fix the arity.

- [ ] **Step 8: Regen + build**
```bash
cd lobby/companion
"/c/Users/r/AppData/Local/SpacetimeDB/spacetime.exe" generate --lang csharp --out-dir module_bindings --module-path ../server/spacetimedb
dotnet build -c Release
```
Expected: 0 errors (CreateLobby now 5-arg, JoinLobby 2-arg, c.Db.MyRelay present).

- [ ] **Step 9: Local end-to-end test (incl. regression)**
Start `serve` (relay + fresh token). Then:
  - register → ack ok.
  - host with `password:"secret"` → `hasPassword=true` in lobbies.json; `spacetime sql` shows Lobby.HasPassword=true and a LobbyRelay row.
  - join the lobby with wrong password → ack `{ok:false,"wrong password"}`.
  - **Regression:** host an OPEN lobby (no password); join it → ack ok + handoff has a relay_session (from my_relay). This proves the moved-session flow still works.

- [ ] **Step 10: Commit** — `feat(lobby/companion): client-side pw hash + async verified join`

---

### Task 3: GUI — password field, lock column, join prompt

**Files:** Modify `lobby/companion-gui/MainForm.cs`

- [ ] **Step 1: Host password field**

Add `private readonly TextBox _hostPw = new();` with the controls. In the host GroupBox (`grp`), add after `_max`:
```csharp
        var lblPw = new Label { Text = "Password:", Location = new Point(445, 28), AutoSize = true };
        _hostPw.Location = new Point(510, 24); _hostPw.Width = 130; _hostPw.UseSystemPasswordChar = true;
```
Add `lblPw, _hostPw` to `grp.Controls.AddRange(...)`, and widen `_host`/move it down if needed (place `_host` at `new Point(470, 22)` stays; move the password row — set group height to 100 and put password on a second line at y=60 if cramped). Simplify: set `grp.Size = new Size(676, 100)`, `_host.Location = new Point(470, 62)`, and password at `Location = new Point(12, 64)` label + `_hostPw` at `new Point(95, 60)`.

- [ ] **Step 2: Lock column + per-row hasPassword**

In `BuildUi`, add a column: `_lobbies.Columns.Add("PW", 40);` (after Status). Replace the `_lobbies.Tag`/selection model: store a small record. Add:
```csharp
    private readonly record struct LobbyRow(ulong Id, bool HasPw);
```
In `RefreshLobbies`, set `it.Tag = new LobbyRow(id, hasPw);` where `bool hasPw = e.TryGetProperty("hasPassword", out var hp) && hp.GetBoolean();` and add a subitem `it.SubItems.Add(hasPw ? "🔒" : "");`. Update the selection-restore + `DoJoin` to read `LobbyRow` from Tag (and `RefreshLobbies`'s `sel` lookup).

- [ ] **Step 3: Join password prompt**

Add a minimal modal prompt:
```csharp
    private static string? PromptPassword(IWin32Window owner)
    {
        using var f = new Form { Text = "Lobby password", FormBorderStyle = FormBorderStyle.FixedDialog,
            StartPosition = FormStartPosition.CenterParent, ClientSize = new Size(300, 96), MinimizeBox = false, MaximizeBox = false };
        var tb = new TextBox { Location = new Point(12, 28), Width = 276, UseSystemPasswordChar = true };
        var lbl = new Label { Text = "Enter password:", Location = new Point(12, 8), AutoSize = true };
        var ok = new Button { Text = "OK", DialogResult = DialogResult.OK, Location = new Point(132, 60), Size = new Size(70, 26) };
        var cancel = new Button { Text = "Cancel", DialogResult = DialogResult.Cancel, Location = new Point(214, 60), Size = new Size(74, 26) };
        f.Controls.AddRange(new Control[] { lbl, tb, ok, cancel }); f.AcceptButton = ok; f.CancelButton = cancel;
        return f.ShowDialog(owner) == DialogResult.OK ? tb.Text : null;
    }
```

- [ ] **Step 4: Wire password into host + join**

In `DoHostToggle`, include the password in the host cmd:
```csharp
        var json = "{\"action\":\"host\",\"cmdId\":" + Esc(NewId(out var id)) + ",\"name\":" + Esc(ln) + ",\"max\":" + (int)_max.Value + ",\"password\":" + Esc(_hostPw.Text) + "}";
```
In `DoJoin`, read the row, prompt if needed, include the password:
```csharp
        if (_lobbies.SelectedItems.Count == 0 || _lobbies.SelectedItems[0].Tag is not LobbyRow row)
        { MessageBox.Show(this, "Select a lobby first.", "PULSAR Lobby"); return; }
        string pw = "";
        if (row.HasPw) { var p = PromptPassword(this); if (p is null) return; pw = p; }
        var json = "{\"action\":\"join\",\"cmdId\":" + Esc(NewId(out var id)) + ",\"lobbyId\":" + row.Id + ",\"password\":" + Esc(pw) + "}";
        Send("join", json, id, _launch.Checked, $"Joining lobby {row.Id} ...");
```
(In `CheckAck`, the `join failed: wrong password` path already logs the error; add a `MessageBox.Show(this, "Wrong password.", "PULSAR Lobby");` when `kind=="join" && !ok`.)

- [ ] **Step 5: Build** — `cd lobby/companion-gui && dotnet build -c Release` → 0 errors.

- [ ] **Step 6: Commit** — `feat(lobby/gui): lobby password — host field, lock marker, join prompt`

---

### Task 4: Deploy
- [ ] Publish self-contained GUI + companion → `_dist/LobbyTools` → copy to NAS (as before).
- [ ] Copy `Lib.cs` to the private repo export, commit, `git push origin main`.
- [ ] User: VPS `git pull` + `spacetime publish ... --delete-data -y` (schema change).

---

## Self-Review
**1. Spec coverage:** private session (Task 1 LobbyRelay + MyRelay) ✓; HasPassword flag (Task 1) ✓; client-side hash (Task 2 HashPw) ✓; async verified join (Task 2 OnJoinLobby + ResolveAwaitedJoin) ✓; lobbies.json hasPassword (Task 2) ✓; GUI field/lock/prompt (Task 3) ✓; HandleDisconnect cleanup (Task 1) ✓; regression open-lobby (Task 2 Step 9) ✓.
**2. Placeholder scan:** complete code per step; no TBD.
**3. Type consistency:** `CreateLobby(name,hostAddr,max,relaySession,passwordHash)` 5-arg used by companion host; `JoinLobby(lobbyId,passwordHash)` 2-arg used by companion + CLI; `OnJoinLobby(ectx,lid,pwh)` matches; `RelayInfo{LobbyId,Session}` matches `c.Db.MyRelay.Iter()`; cmd JSON `password` key produced by GUI + read by companion; `LobbyRow{Id,HasPw}` consistent across RefreshLobbies/DoJoin.
