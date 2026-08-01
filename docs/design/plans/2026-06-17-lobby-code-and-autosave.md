# Lobby Join-Code + Auto-Save Implementation Plan


**Goal:** Add a short typable lobby join-code (shortcut to the public browser) and event-based auto-save of the player's character profile (on co-op disconnect + on save/exit).

**Architecture:** STDB stays meta/persistence only; real-time co-op stays on ENet/relay. The join-code is a public `Lobby.Code` column resolved client-side in the companion (no new reducer) and fed into the existing join path. Auto-save reuses the existing F10 capture (`WriteProfileOut`) driven from the game thread, triggered by an atomic flag set on disconnect and on the SFSE save message; the companion `serve` daemon watches `profile-out` and calls `upsert_profile` with a unix-seconds version (monotonic LWW).

**Tech Stack:** C# SpacetimeDB module + ClientSDK, .NET 8 companion, WinForms GUI, C++ SFSE plugin (CommonLibSF), ENet.

**Reference spec:** `docs/design/specs/2026-06-17-lobby-code-and-autosave-design.md`

---

## File Structure

- `lobby/server/spacetimedb/Lib.cs` — add `Lobby.Code` + deterministic generator in `CreateLobby`.
- `lobby/companion/module_bindings/**` — regenerated after the schema change.
- `lobby/companion/Program.cs` — host ack carries `code`; new `joinCode` action; `profile-out` auto-upload watcher.
- `lobby/companion-gui/MainForm.cs` — host code display + Copy; join-by-code field; auto-save toggle + last-saved label.
- `include/Lobby/LobbyHandoff.h`, `src/Lobby/LobbyHandoff.cpp` — `RequestAutoSave()` + `TickAutoSave()`.
- `src/Main.cpp` — call `TickAutoSave()` in the game tick; `RequestAutoSave()` on the SFSE save message.
- `src/Network/NetworkHost.cpp` — `RequestAutoSave()` in `OnDisconnect`.

---

## FEATURE 1 — Lobby Join-Code

### Task 1: Module — `Lobby.Code` column + deterministic generator

**Files:**
- Modify: `lobby/server/spacetimedb/Lib.cs` (Lobby struct ~`79-94`; helpers ~`50-57`; `CreateLobby` ~`213-256`)

- [ ] **Step 1: Add the `Code` column to the public Lobby table**

In the `Lobby` struct, add a field after `MemberCount`:

```csharp
        public uint      MemberCount;   // denormalized joiner count for the browser
        public string    Code;          // short typable join code (browser shortcut)
        public Timestamp CreatedAt;
```

- [ ] **Step 2: Add the code constants + generator helper**

Add near the other constants/helpers (after `IsValidHostAddr`, ~line 57):

```csharp
    private const string CODE_ALPHABET = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789"; // no 0/O/1/I
    private const int    CODE_LEN      = 6;

    // Deterministic (ctx.Rng) join code, unique among OPEN lobbies. Closed lobbies
    // keep their old code (ignored — only open lobbies are joinable).
    private static string GenLobbyCode(ReducerContext ctx)
    {
        for (int attempt = 0; attempt < 6; attempt++)
        {
            var chars = new char[CODE_LEN];
            for (int i = 0; i < CODE_LEN; i++)
                chars[i] = CODE_ALPHABET[ctx.Rng.Next(0, CODE_ALPHABET.Length)];
            string code = new string(chars);
            bool taken = false;
            foreach (var l in ctx.Db.Lobby.Iter())
                if (l.Status == "open" && l.Code == code) { taken = true; break; }
            if (!taken) return code;
        }
        // Astronomically unlikely fallback: a longer code to guarantee uniqueness.
        var ch2 = new char[CODE_LEN + 2];
        for (int i = 0; i < ch2.Length; i++) ch2[i] = CODE_ALPHABET[ctx.Rng.Next(0, CODE_ALPHABET.Length)];
        return new string(ch2);
    }
```

- [ ] **Step 3: Assign the code in `CreateLobby`**

In the `ctx.Db.Lobby.Insert(new Lobby { ... })` call, add `Code = GenLobbyCode(ctx),` right after `MemberCount = 0,`:

```csharp
            MemberCount  = 0,
            Code         = GenLobbyCode(ctx),
            CreatedAt    = ctx.Timestamp,
```

- [ ] **Step 4: Publish locally (schema change → wipe)**

Run: `cd /d/Pulsar/PULSAR/lobby/server/spacetimedb && spacetime publish pulsar-lobby --server local --delete-data -y`
Expected: `Build finished successfully.` then `Updated database with name: pulsar-lobby`.

- [ ] **Step 5: Verify the code is generated and on the public row**

Run:
```bash
cd /d/Pulsar/PULSAR
spacetime call pulsar-lobby register_account '"CodeTester"' --server local
spacetime call pulsar-lobby create_lobby '"CodeLob"' '"127.0.0.1:7777"' 2 '""' '""' --server local
spacetime sql pulsar-lobby "SELECT LobbyId, Name, Code, Status FROM lobby" --server local
```
Expected: one row with `Code` = a 6-character string drawn only from `ABCDEFGHJKLMNPQRSTUVWXYZ23456789` (status will be `closed` after the CLI call disconnects — that's fine; the row + code persist).

- [ ] **Step 6: Commit**

```bash
git add lobby/server/spacetimedb/Lib.cs
git commit -m "feat(lobby/module): add Lobby.Code join code (ctx.Rng, unique among open)"
```

---

### Task 2: Companion — host ack carries code, `joinCode` action, regen bindings

**Files:**
- Regenerate: `lobby/companion/module_bindings/**`
- Modify: `lobby/companion/Program.cs` (`WriteAck` ~`524-532`; serve setup ~`299-339`; `HandleCommandFile` ~`450-522`; serve loop in `Main` ~`197-211`; `WriteLobbies` ~`400-424`)

- [ ] **Step 1: Regenerate the C# bindings (adds `Lobby.Code`)**

Run:
```bash
cd /d/Pulsar/PULSAR/lobby
spacetime generate --lang csharp --out-dir companion/module_bindings --module-path server/spacetimedb
```
Expected: writes `Tables/Lobby.g.cs` etc.; no new table files removed. (If it prompts to delete a no-longer-generated file, leave it; Account is already gone.)

- [ ] **Step 2: Extend `WriteAck` to carry an optional code**

Replace the `WriteAck` method:

```csharp
    private static void WriteAck(string cmdId, bool ok, string? error, string? hostAddr, string? code = null)
    {
        var sb = new StringBuilder("{\"cmdId\":").Append(JsonEsc(cmdId))
            .Append(",\"ok\":").Append(ok ? "true" : "false");
        if (error is not null) sb.Append(",\"error\":").Append(JsonEsc(error));
        if (hostAddr is not null) sb.Append(",\"hostAddr\":").Append(JsonEsc(hostAddr));
        if (code is not null) sb.Append(",\"code\":").Append(JsonEsc(code));
        sb.Append('}');
        try { File.WriteAllText(AckPath, sb.ToString()); } catch { /* best-effort */ }
    }
```

- [ ] **Step 3: Store the connection identity (needed to find the host's own lobby)**

Add a field near the other serve state (after line ~42):

```csharp
    private static Identity s_identity;
```

In the `OnConnect` lambda (Main, ~line 159), capture it:

```csharp
            .OnConnect((c, identity, token) =>
            {
                s_identity = identity;
                SaveToken(token);
```

- [ ] **Step 4: Add deferred host-ack state + resolver (host ack waits for the code)**

Add fields near the join state (after `s_joinDeadline`, ~line 68):

```csharp
    private static string s_hostCmd = "";
    private static long   s_hostDeadline = 0;
```

Add a resolver method next to `ResolveAwaitedJoin`:

```csharp
    // serve: after a CreateLobby commit, the host's lobby (with its generated Code)
    // appears in the local cache; surface the code in the host ack. Times out safely.
    private static void ResolveAwaitedHost(DbConnection c)
    {
        if (s_hostCmd.Length == 0) return;
        foreach (var l in c.Db.Lobby.Iter())
            if (l.HostIdentity == s_identity && l.Status == "open")
            {
                WriteAck(s_hostCmd, true, null, null, l.Code);
                Console.WriteLine($"[companion] serve: hosted '{l.Name}' code={l.Code}");
                s_hostCmd = ""; return;
            }
        if (GetMs() > s_hostDeadline)
        {
            WriteAck(s_hostCmd, true, null, null);   // hosted, code just not visible yet
            Console.WriteLine("[companion] serve: host ack (code unresolved)");
            s_hostCmd = "";
        }
    }
```

- [ ] **Step 5: Call the resolver in the serve loop**

In `Main`, in the `if (cmd == "serve")` block (~line 201-205), add the host resolver:

```csharp
                if (cmd == "serve")
                {
                    lastLobbies = WriteLobbies(conn, lastLobbies);
                    HandleCommandFile(conn);
                    ResolveAwaitedJoin(conn);
                    ResolveAwaitedHost(conn);
                }
```

- [ ] **Step 6: Defer the host ack in `HandleCommandFile` (host branch)**

Replace the host branch's synchronous ack. Find:

```csharp
                c.Reducers.CreateLobby(name, pubAddr, max, session, pwHash);
                WriteHandoff("host", lanAddr, 0, RelayAddr, session);
                WriteAck(cmdId, true, null, null);
                Console.WriteLine($"[companion] serve: hosted '{name}' pw={pwHash.Length>0} relay={RelayAddr}");
```

Replace with:

```csharp
                c.Reducers.CreateLobby(name, pubAddr, max, session, pwHash);
                WriteHandoff("host", lanAddr, 0, RelayAddr, session);
                s_hostCmd = cmdId; s_hostDeadline = GetMs() + 6000;   // ack written by ResolveAwaitedHost (with code)
                Console.WriteLine($"[companion] serve: host requested '{name}' pw={pwHash.Length>0} relay={RelayAddr}");
```

- [ ] **Step 7: Add the `joinCode` action in `HandleCommandFile`**

After the `else if (action == "join")` block and before `else if (action == "register")`, add:

```csharp
            else if (action == "joinCode")
            {
                string code = (root.TryGetProperty("code", out var cc) ? (cc.GetString() ?? "") : "")
                    .Trim().ToUpperInvariant();
                string pwHash = root.TryGetProperty("passwordHash", out var jc) ? (jc.GetString() ?? "") : "";
                ulong lobbyId = 0; string addr = "";
                foreach (var l in c.Db.Lobby.Iter())
                    if (l.Status == "open" && (l.Code ?? "").ToUpperInvariant() == code)
                    { lobbyId = l.LobbyId; addr = l.HostAddr ?? ""; break; }
                if (lobbyId == 0)
                {
                    WriteAck(cmdId, false, "code not found", null);
                    Console.WriteLine($"[companion] serve: joinCode '{code}' not found");
                }
                else
                {
                    var mine = FindMyProfile(c); uint version = 0;
                    if (mine is not null) { File.WriteAllBytes(ProfileInPath, mine.Blob.ToArray()); version = mine.Version; }
                    s_joinCmd = cmdId; s_joinLobby = lobbyId; s_joinDirectAddr = addr; s_joinVersion = version;
                    s_joinAwaitSession = false;
                    c.Reducers.JoinLobby(lobbyId, pwHash);   // reuses OnJoinLobby + ResolveAwaitedJoin
                    Console.WriteLine($"[companion] serve: joinCode '{code}' -> lobby {lobbyId}");
                }
            }
```

- [ ] **Step 8: Include the code in the published lobby list (host can see its own)**

In `WriteLobbies`, add the code to each entry. After the `memberCount` append line, add:

```csharp
              .Append(",\"memberCount\":").Append(l.MemberCount)
              .Append(",\"code\":").Append(JsonEsc(l.Code ?? ""))
              .Append(",\"maxPlayers\":").Append(l.MaxPlayers)
```

- [ ] **Step 9: Build the companion**

Run: `cd /d/Pulsar/PULSAR/lobby/companion && dotnet build -c Release`
Expected: `Build succeeded. 0 Error(s)`.

- [ ] **Step 10: Integration test — host gets a code, joinCode resolves it (single daemon)**

This uses one connected `serve` daemon (the lobby stays open while it runs; joining your own lobby is allowed). Run:

```bash
cd /d/Pulsar/PULSAR/lobby/companion
DIR="$LOCALAPPDATA/PULSAR"; DIR="${DIR//\\//}"
rm -f "$DIR/pulsar_ack.json" "$DIR/pulsar_cmd.json"
PULSAR_STDB_URI=http://127.0.0.1:3000 ./bin/Release/net8.0/PulsarCompanion serve CodeHost &
SERVE=$!; sleep 4
# host
printf '{"action":"host","cmdId":"h1","name":"CodeFlow","max":2,"passwordHash":""}' > "$DIR/pulsar_cmd.json"
sleep 3; echo "HOST ACK:"; cat "$DIR/pulsar_ack.json"; echo
CODE=$(grep -o '"code":"[^"]*"' "$DIR/pulsar_ack.json" | head -1 | sed 's/.*:"//;s/"//')
echo "extracted code=$CODE"
# join by that code
printf '{"action":"joinCode","cmdId":"j1","code":"%s","passwordHash":""}' "$CODE" > "$DIR/pulsar_cmd.json"
sleep 3; echo "JOINCODE ACK:"; cat "$DIR/pulsar_ack.json"; echo
# bogus code
printf '{"action":"joinCode","cmdId":"j2","code":"ZZZZZZ","passwordHash":""}' > "$DIR/pulsar_cmd.json"
sleep 3; echo "BOGUS ACK:"; cat "$DIR/pulsar_ack.json"; echo
kill $SERVE 2>/dev/null
```
Expected: HOST ACK has `"ok":true` + a 6-char `"code"`; JOINCODE ACK `"cmdId":"j1","ok":true`; BOGUS ACK `"cmdId":"j2","ok":false,"error":"code not found"`.

- [ ] **Step 11: Commit**

```bash
git add lobby/companion/Program.cs
git add -A lobby/companion/module_bindings
git commit -m "feat(lobby/companion): host ack returns join code + joinCode action"
```

---

### Task 3: GUI — host code display + Copy, join-by-code field

**Files:**
- Modify: `lobby/companion-gui/MainForm.cs` (controls ~`44-63`; `BuildUi` ~`120-159`; `CheckAck` ~`349-385`; helpers)

- [ ] **Step 1: Declare the new controls**

Add to the control field declarations (after `_join`, ~line 57):

```csharp
    private readonly TextBox _codeBox = new();
    private readonly TextBox _codePw = new();
    private readonly Button  _joinCode = new();
    private readonly Label   _hostCodeLbl = new();
    private readonly TextBox _hostCode = new();
    private readonly Button  _copyCode = new();
```

- [ ] **Step 2: Make room — bump the window + host group height**

In `BuildUi`, change `ClientSize` and the host group size. Replace `ClientSize = new Size(700, 560);` with:

```csharp
        ClientSize = new Size(700, 588);
```

Replace the `grp` creation line with a taller group:

```csharp
        var grp = new GroupBox { Text = "Host a new lobby", Location = new Point(12, 414), Size = new Size(676, 98) };
```

- [ ] **Step 3: Add the host-code row to the group**

After `grp.Controls.AddRange(new Control[] { lblHN, _hostName, lblMax, _max, lblPw, _hostPw, _host });`, add:

```csharp
        _hostCodeLbl.Text = "Invite code:"; _hostCodeLbl.Location = new Point(12, 62); _hostCodeLbl.AutoSize = true;
        _hostCode.Location = new Point(95, 58); _hostCode.Width = 110; _hostCode.ReadOnly = true;
        _hostCode.Font = new Font("Consolas", 11f, FontStyle.Bold); _hostCode.Text = "";
        _copyCode.Location = new Point(215, 57); _copyCode.Size = new Size(70, 24); _copyCode.Text = "Copy";
        _copyCode.Click += (_, _) => { if (_hostCode.Text.Length > 0) { Clipboard.SetText(_hostCode.Text); Log("Invite code copied."); } };
        _hostCodeLbl.Visible = false; _hostCode.Visible = false; _copyCode.Visible = false;
        grp.Controls.AddRange(new Control[] { _hostCodeLbl, _hostCode, _copyCode });
```

- [ ] **Step 4: Add the join-by-code controls (right gutter, under Join)**

After the `_join.Click += (_, _) => DoJoin();` line, add:

```csharp
        var lblOr = new Label { Text = "or by code:", Location = new Point(582, 230), AutoSize = true, Anchor = AnchorTR() };
        _codeBox.Location = new Point(582, 250); _codeBox.Width = 100; _codeBox.CharacterCasing = CharacterCasing.Upper;
        _codeBox.MaxLength = 8; _codeBox.Anchor = AnchorTR();
        _codePw.Location = new Point(582, 276); _codePw.Width = 100; _codePw.UseSystemPasswordChar = true;
        _codePw.Anchor = AnchorTR();   // optional password for code-join
        _joinCode.Location = new Point(582, 302); _joinCode.Size = new Size(100, 28); _joinCode.Text = "Join code";
        _joinCode.Anchor = AnchorTR();
        _joinCode.Click += (_, _) => DoJoinByCode();
```

Then add these three controls to the form's `Controls.AddRange(...)` list (append `lblOr, _codeBox, _codePw, _joinCode` to the array at ~line 152-158):

```csharp
            lblList, _lobbies, _join, lblOr, _codeBox, _codePw, _joinCode, grp, _log
```

- [ ] **Step 5: Implement `DoJoinByCode`**

Add next to `DoJoin`:

```csharp
    private void DoJoinByCode()
    {
        if (!Connected || !_nameClaimed) return;
        var code = _codeBox.Text.Trim();
        if (code.Length < 4) { MessageBox.Show(this, "Enter a lobby code.", "PULSAR Lobby"); return; }
        var json = "{\"action\":\"joinCode\",\"cmdId\":" + Esc(NewId(out var id))
                 + ",\"code\":" + Esc(code) + ",\"passwordHash\":" + Esc(HashPw(_codePw.Text)) + "}";
        Send("joinCode", json, id, _launch.Checked, $"Joining by code {code} ...");
    }
```

- [ ] **Step 6: Show the code on host ack; hide on close; treat joinCode like join in `CheckAck`**

In `CheckAck`, change the `else if (ok)` block to capture the code and show it on host:

```csharp
            else if (ok)
            {
                if (kind == "host")
                {
                    _hosting = true; SetState();
                    string code = Str(root, "code");
                    if (code.Length > 0)
                    {
                        _hostCode.Text = code;
                        _hostCodeLbl.Visible = _hostCode.Visible = _copyCode.Visible = true;
                        Log("Lobby hosted. Invite code: " + code);
                    }
                    else Log("Lobby hosted.");
                }
                else Log(kind == "joinCode" || kind == "join" ? "Joined." : "Done.");
                if (_launchAfterPending) LaunchGame();
            }
            else { Log(kind + " failed: " + err); if (kind == "join" || kind == "joinCode") MessageBox.Show(this, "Wrong password or cannot join.", "PULSAR Lobby"); }
```

In the `else if (kind == "close")` block, hide the code row:

```csharp
            else if (kind == "close")
            {
                _hosting = false;
                _hostCodeLbl.Visible = _hostCode.Visible = _copyCode.Visible = false;
                _hostCode.Text = "";
                SetState(); Log(ok ? "Lobby closed." : "Close failed: " + err);
            }
```

- [ ] **Step 7: Gate the join-code button with state**

In `SetState`, add after the `_join.Enabled` line:

```csharp
        _join.Enabled = c && _nameClaimed && !_hosting;
        _joinCode.Enabled = _codeBox.Enabled = _codePw.Enabled = c && _nameClaimed && !_hosting;
```

- [ ] **Step 8: Build the GUI**

Run: `cd /d/Pulsar/PULSAR/lobby/companion-gui && dotnet build -c Release`
Expected: `Build succeeded. 0 Error(s)`.

- [ ] **Step 9: Commit**

```bash
git add lobby/companion-gui/MainForm.cs
git commit -m "feat(lobby/gui): show host invite code (+Copy) and join-by-code"
```

---

## FEATURE 2 — Auto-Save (event-based)

### Task 4: DLL — `RequestAutoSave` flag + game-thread `TickAutoSave`, disconnect + save hooks

**Files:**
- Modify: `include/Lobby/LobbyHandoff.h`
- Modify: `src/Lobby/LobbyHandoff.cpp` (add includes `<atomic>`; functions after `WriteProfileOut`)
- Modify: `src/Main.cpp` (game tick ~`1120`; SFSE handler ~`869`)
- Modify: `src/Network/NetworkHost.cpp` (`OnDisconnect` ~`1376`; include)

- [ ] **Step 1: Declare the two functions in the header**

Add to `include/Lobby/LobbyHandoff.h` in the `Pulsar::Lobby` namespace (next to `WriteProfileOut`):

```cpp
    // Auto-save: RequestAutoSave() is thread-safe (sets a flag); TickAutoSave()
    // does the actual capture on the game thread (safe-point gated + debounced).
    void RequestAutoSave();
    void TickAutoSave();
```

- [ ] **Step 2: Implement the flag + drain in LobbyHandoff.cpp**

Add `#include <atomic>` to the includes, then append inside the namespace (after `WriteProfileOut`):

```cpp
    static std::atomic<bool> g_autoSaveRequested{ false };
    static uint64_t          s_lastAutoSaveMs = 0;

    void RequestAutoSave()
    {
        g_autoSaveRequested.store(true, std::memory_order_relaxed);
    }

    void TickAutoSave()
    {
        if (!g_autoSaveRequested.exchange(false, std::memory_order_relaxed)) return;

        // Safe-point: only capture when the player is fully spawned (reuse the
        // connect-gate that TickApplyProfile uses). If not ready, re-arm and retry.
        if (!Pulsar::InventorySync::DirectRestore::IsPlayerReadyForConnect()) {
            g_autoSaveRequested.store(true, std::memory_order_relaxed);
            return;
        }

        // Debounce bursts (e.g. several saves in a row). Drop — the next trigger
        // (disconnect or save) will capture the latest state.
        uint64_t now = ::GetTickCount64();
        if (s_lastAutoSaveMs != 0 && now - s_lastAutoSaveMs < 15000) return;
        s_lastAutoSaveMs = now;

        WriteProfileOut();
        REX::INFO("[PULSAR] [LobbyHandoff] auto-save captured profile-out"sv);
    }
```

- [ ] **Step 3: Drain the flag in the game-thread tick**

In `src/Main.cpp`, next to the existing `TickApplyProfile()` call (~line 1120), add:

```cpp
                Pulsar::Lobby::TickApplyProfile();   // Teil 2C: apply staged profile-in once
                Pulsar::Lobby::TickAutoSave();       // auto-save: drain capture request (game thread)
```

- [ ] **Step 4: ~~Trigger auto-save on the SFSE save message~~ — DEFERRED (engine reality)**

VERIFIED during implementation: CommonLibSF's `SFSE::MessagingInterface::MessageType` only exposes
`kPostLoad, kPostPostLoad, kPostDataLoad, kPostPostDataLoad` — **there is no save/exit message**
(SFSE, unlike SKSE, does not broadcast `kSaveGame`). The codebase only *triggers* saves
(`BGSSaveLoadManager`, REL::ID 98640) and has no save-completed hook to piggyback on.

Detouring the core save/load dispatcher (REL::ID 98640) is risky — it runs during save-load
"chaos" that PULSAR already suspends all hooks for (HookGuard), and capturing the profile
mid-serialize could crash. So save/exit capture is **deferred** to a separate, careful follow-up
(hook + post-save timing). The **disconnect trigger (Step 5) fully covers the co-op case**:
both host and client receive `OnDisconnect` when a session ends, capturing the state to carry
forward. Solo-only progress between co-op sessions is the documented gap until the follow-up.

No Main.cpp save-message edit. (The tick drain in Step 3 stays.)

- [ ] **Step 5: Trigger auto-save on co-op disconnect**

In `src/Network/NetworkHost.cpp`, ensure `#include "Lobby/LobbyHandoff.h"` is present near the top. In `OnDisconnect`, right after the existing M.3 capture call (~line 1376):

```cpp
        Pulsar::InventorySync::OnNetworkDisconnect();

        // Lobby auto-save: back up the player's character (loot/credits gained this
        // session). Flag only — actual capture runs on the game thread (TickAutoSave).
        Pulsar::Lobby::RequestAutoSave();
```

- [ ] **Step 6: Build the DLL (standard build)**

Build per the project's standard process (see memory: Build+Deploy / Build sys). Expected: PULSAR.dll compiles with no errors; `RequestAutoSave`/`TickAutoSave` symbols present.

- [ ] **Step 7: Commit**

```bash
git add include/Lobby/LobbyHandoff.h src/Lobby/LobbyHandoff.cpp src/Main.cpp src/Network/NetworkHost.cpp
git commit -m "feat(lobby/dll): event-based auto-save (disconnect + save) -> profile-out"
```

---

### Task 5: Companion — `profile-out` watcher → `upsert_profile` (unix-seconds version)

**Files:**
- Modify: `lobby/companion/Program.cs` (serve setup ~`299-339`; serve loop ~`201-205`; new method near `ResolveAwaitedJoin`)

- [ ] **Step 1: Add the auto-save state + uploader**

Add fields near the other serve state:

```csharp
    private static bool s_autoSave = (Environment.GetEnvironmentVariable("PULSAR_AUTOSAVE") ?? "1") != "0";
    private static long s_profileOutSig = -1;
```

Add the uploader method next to `ResolveAwaitedJoin`:

```csharp
    // serve: watch profile-out (written by the DLL on disconnect/save) and upload it.
    // version = unix seconds so the module's LWW is always monotonic regardless of source.
    private static void AutoUploadProfile(DbConnection c)
    {
        if (!s_autoSave) return;
        try
        {
            if (!File.Exists(ProfileOutPath)) return;
            var fi = new FileInfo(ProfileOutPath);
            long sig = fi.LastWriteTimeUtc.Ticks ^ (fi.Length << 1);
            if (sig == s_profileOutSig) return;
            s_profileOutSig = sig;
            var blob = new List<byte>(File.ReadAllBytes(ProfileOutPath));
            if (blob.Count == 0) return;
            uint version = (uint)DateTimeOffset.UtcNow.ToUnixTimeSeconds();
            c.Reducers.UpsertProfile(blob, version);
            Console.WriteLine($"[companion] serve: auto-saved profile ({blob.Count} bytes) v{version}");
        }
        catch (Exception ex) { Console.WriteLine($"[companion] serve: auto-save error: {ex.Message}"); }
    }
```

- [ ] **Step 2: Seed the signature at serve start (don't re-upload a stale pre-existing file)**

In the `case "serve":` block (Dispatch), after the subscription is set up, add:

```csharp
                // Don't upload a profile-out left over from a previous run; only NEW
                // captures during this session should auto-upload.
                try { if (File.Exists(ProfileOutPath)) { var fi = new FileInfo(ProfileOutPath); s_profileOutSig = fi.LastWriteTimeUtc.Ticks ^ (fi.Length << 1); } } catch { }
                if (!s_autoSave) Console.WriteLine("[companion] serve: auto-save disabled (PULSAR_AUTOSAVE=0)");
```

- [ ] **Step 3: Call the uploader in the serve loop**

In `Main`'s serve block, add `AutoUploadProfile(conn);`:

```csharp
                if (cmd == "serve")
                {
                    lastLobbies = WriteLobbies(conn, lastLobbies);
                    HandleCommandFile(conn);
                    ResolveAwaitedJoin(conn);
                    ResolveAwaitedHost(conn);
                    AutoUploadProfile(conn);
                }
```

- [ ] **Step 4: Build the companion**

Run: `cd /d/Pulsar/PULSAR/lobby/companion && dotnet build -c Release`
Expected: `Build succeeded. 0 Error(s)`.

- [ ] **Step 5: Integration test — profile-out change triggers exactly one upload with a fresh version**

```bash
cd /d/Pulsar/PULSAR/lobby/companion
DIR="$LOCALAPPDATA/PULSAR"; DIR="${DIR//\\//}"
rm -f "$DIR/pulsar_profile_out.json"
PULSAR_STDB_URI=http://127.0.0.1:3000 PULSAR_AUTOSAVE=1 ./bin/Release/net8.0/PulsarCompanion serve SaveTester &
SERVE=$!; sleep 4
printf 'BLOBv1-bytes' > "$DIR/pulsar_profile_out.json"; sleep 3
printf 'BLOBv2-longer-bytes' > "$DIR/pulsar_profile_out.json"; sleep 3
kill $SERVE 2>/dev/null
echo "--- stored profile (owner read) ---"
spacetime sql pulsar-lobby "SELECT Identity, Version FROM profile" --server local
```
Expected: daemon stdout logs `auto-saved profile` twice with increasing `v<unixseconds>`; the `profile` table shows one row whose `Version` is the later unix-seconds value. (Note: the blob is arbitrary bytes here — this only verifies the upload+LWW path, not in-game capture.)

- [ ] **Step 6: Commit**

```bash
git add lobby/companion/Program.cs
git commit -m "feat(lobby/companion): auto-upload profile-out on change (unix-seconds LWW)"
```

---

### Task 6: GUI — auto-save toggle + last-saved label

**Files:**
- Modify: `lobby/companion-gui/MainForm.cs` (`Cfg` ~`32-40`; controls ~`44-63`; `BuildUi`; `ApplyCfgToUi`; `ToggleDaemon` env; `OutputDataReceived`)

- [ ] **Step 1: Add the config field**

In `Cfg`, add:

```csharp
        public bool Advanced { get; set; } = false;
        public bool AutoSave { get; set; } = true;
```

- [ ] **Step 2: Declare the controls**

Add to the control declarations:

```csharp
    private readonly CheckBox _autoSave = new();
    private readonly Label    _lastSaved = new();
```

- [ ] **Step 3: Add the controls in `BuildUi`**

After the `_launch` setup line (~line 114), add:

```csharp
        _autoSave.Location = new Point(95, 160); _autoSave.AutoSize = true; _autoSave.Text = "Auto-save my character";
        _autoSave.CheckedChanged += (_, _) => { _cfg.AutoSave = _autoSave.Checked; SaveCfg(); Log("Auto-save " + (_autoSave.Checked ? "on" : "off") + " (applies on next daemon start)."); };
        _lastSaved.Location = new Point(290, 161); _lastSaved.AutoSize = true; _lastSaved.Text = ""; _lastSaved.ForeColor = Color.Gray;
```

Add both to the form's `Controls.AddRange(...)`:

```csharp
            lblSf, _starfield, _browse, _launch, _autoSave, _lastSaved, _start, _status,
```

> NOTE: the list at ~line 192 (`_lobbies`) and the group at y=414 already moved down via Task 3's ClientSize bump; the `_autoSave`/`_lastSaved` row at y=160 sits in the existing gap below `_launch` (y=138) and above the list label (y=172).

- [ ] **Step 4: Apply config to the checkbox**

In `ApplyCfgToUi`, after `_advanced.Checked = _cfg.Advanced;`:

```csharp
        _autoSave.Checked = _cfg.AutoSave;
```

- [ ] **Step 5: Pass the toggle to the daemon + disable while running**

In `ToggleDaemon`, after `psi.Environment["PULSAR_RELAY_ADDR"] = relay;`:

```csharp
            psi.Environment["PULSAR_AUTOSAVE"] = _autoSave.Checked ? "1" : "0";
```

In `SetState`, add `_autoSave` to the not-connected-enable group:

```csharp
        _starfield.Enabled = !c; _browse.Enabled = !c; _autoSave.Enabled = !c;
```

- [ ] **Step 6: Update the last-saved label from the daemon log**

In `ToggleDaemon`, change the `OutputDataReceived` handler to also catch auto-save lines:

```csharp
            _daemon.OutputDataReceived += (_, e) => { if (e.Data != null) { Log("daemon: " + e.Data); if (e.Data.Contains("subscribed")) BeginInvoke(() => _daemonReady = true); if (e.Data.Contains("auto-saved profile")) BeginInvoke(() => _lastSaved.Text = "Last saved: " + DateTime.Now.ToString("HH:mm")); } };
```

- [ ] **Step 7: Build the GUI**

Run: `cd /d/Pulsar/PULSAR/lobby/companion-gui && dotnet build -c Release`
Expected: `Build succeeded. 0 Error(s)`.

- [ ] **Step 8: Commit**

```bash
git add lobby/companion-gui/MainForm.cs
git commit -m "feat(lobby/gui): auto-save toggle + last-saved indicator"
```

---

## Task 7: Deploy both features together (one wipe)

**Files:** none (build/deploy only)

- [ ] **Step 1: Refresh the self-contained tooling in `_dist/LobbyTools`**

```bash
cd /d/Pulsar/PULSAR
dotnet publish lobby/companion -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true -o lobby/_dist/_pub_companion
dotnet publish lobby/companion-gui -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true -o lobby/_dist/_pub_gui
cp -f lobby/_dist/_pub_gui/PulsarLobby.exe lobby/_dist/LobbyTools/PulsarLobby.exe
cp -f lobby/_dist/_pub_gui/PulsarLobby.pdb lobby/_dist/LobbyTools/PulsarLobby.pdb
cp -f lobby/_dist/_pub_companion/PulsarCompanion.exe lobby/_dist/LobbyTools/companion/PulsarCompanion.exe
cp -f lobby/_dist/_pub_companion/PulsarCompanion.pdb lobby/_dist/LobbyTools/companion/PulsarCompanion.pdb
rm -rf lobby/_dist/_pub_companion lobby/_dist/_pub_gui
```

- [ ] **Step 2: Sync the module to the private deploy repo (VPS pulls this)**

```bash
cp -f /d/Pulsar/PULSAR/lobby/server/spacetimedb/Lib.cs /d/Pulsar/pulsar-lobby-server/lobby/server/spacetimedb/Lib.cs
cd /d/Pulsar/pulsar-lobby-server
git add lobby/server/spacetimedb/Lib.cs
git commit -m "feat: lobby join code (Lobby.Code)"
git push origin HEAD
```

- [ ] **Step 3: Deploy the DLL per the standard process**

Build + copy PULSAR.dll to the Starfield SFSE plugins dir + NAS, per memory Build+Deploy. (Auto-save needs the new DLL in-game.)

- [ ] **Step 4: User-side VPS republish (schema change)**

Document for the user: on the VPS `git pull`, then `spacetime publish pulsar-lobby --delete-data -y` (wipes accounts/lobbies — names re-claimed). No relay/container change this round (relay untouched).

- [ ] **Step 5: Final local smoke test**

Start the GUI (`lobby/_dist/LobbyTools/PulsarLobby.exe`), start the daemon, claim a name, host → confirm an invite code appears + Copy works; in a second instance/identity, "Join code" with that code → joins. (Two GUIs on one machine share the LOCALAPPDATA dir, so this is a quick UI check, not a full two-player test — that's the in-game step.)

- [ ] **Step 6: Update memory + task tracking**

Update `MEMORY.md` lobby line + `project_lobby_system.md` with the join-code + auto-save shipping. Create/close the relevant task.

---

## Self-Review

**Spec coverage:**
- Lobby code: public `Lobby.Code` (Task 1) ✓; client-side resolution, no new reducer (Task 2 joinCode) ✓; host ack carries code (Task 2) ✓; GUI host display+Copy + join-by-code (Task 3) ✓; password orthogonal (`_codePw`, HashPw) ✓.
- Auto-save: DLL capture on disconnect + save/exit, gated + debounced (Task 4) ✓; companion watches profile-out → upsert_profile unix-seconds (Task 5) ✓; GUI toggle + last-saved (Task 6) ✓; restore unchanged (untouched) ✓.
- Deploy together, one wipe (Task 7) ✓. Out-of-scope items (credits/mods/periodic) not implemented ✓.

**Placeholder scan:** No TBD/TODO; every code step has full code; the one verification note (SFSE `kSaveGame` enum name) is an explicit verify-step with the fallback stated, not a placeholder.

**Type consistency:** `WriteAck(..., string? code = null)` used consistently (Task 2 steps 2/4/6, joinCode). `RequestAutoSave()`/`TickAutoSave()` declared (Task 4.1), defined (4.2), called (4.3/4.4/4.5). `s_identity`, `s_hostCmd`, `s_profileOutSig`, `s_autoSave` declared before use. GUI `_codeBox/_codePw/_joinCode/_hostCode/_copyCode/_autoSave/_lastSaved` declared (3.1/6.2) before BuildUi use. `PULSAR_AUTOSAVE` env written by GUI (6.5) and read by companion (5.1). Cmd action string `joinCode` matches GUI sender (3.5) and companion handler (2.7). `Code` column name consistent across module/bindings/companion/WriteLobbies.

## Risks (from spec)
- **SFSE save-message enum name** — verify `kSaveGame`; disconnect-capture covers the co-op case even if exit-capture needs adjustment.
- **Capture mid-teardown** — mitigated by the `IsPlayerReadyForConnect` safe-point gate (re-arm if not ready) and the 15s debounce.
- **Stale profile-out re-upload on daemon start** — mitigated by seeding `s_profileOutSig` to the existing file (Task 5.2).
