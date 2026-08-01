# Lobby Backend Security — Tier 1 Implementation Plan


**Goal:** Close the pre-online read-exposure and transport/at-rest gaps in the PULSAR SpacetimeDB lobby backend, keeping anonymous identity (real accounts = Tier 2/Beta).

**Architecture:** The C# STDB module (`lobby/server/spacetimedb/Lib.cs`) gets per-client read isolation (RLS, with a per-user View as fallback), denormalized member counts so member identities never leave the server, and reducer input caps. The C# companion (`lobby/companion/Program.cs`) reads profiles through the isolated path, gets a configurable (TLS-ready) endpoint, and encrypts its identity token at rest with Windows DPAPI.

**Tech Stack:** SpacetimeDB 2.5 (C# WASM module + C# ClientSDK), .NET 8, Windows DPAPI (`System.Security.Cryptography.ProtectedData`).

**Spec:** `docs/design/specs/2026-06-16-lobby-security-tier1-design.md`

**Conventions for every command below:**
- spacetime CLI is **not** on PATH. Always call it by full path:
  `& "%USERPROFILE%\AppData\Local\SpacetimeDB\spacetime.exe"` (alias `$stdb` below).
- Reducer/table/column names on the wire are **snake_case** (`UpsertProfile` → `upsert_profile`, `MemberCount` → `member_count`).
- `spacetime call` connects→calls→**disconnects** each time → `ClientDisconnected` (`HandleDisconnect`) fires after every call. So a CLI `join_lobby` is auto-undone; persistent membership counts are only observable via the companion. Tests below account for this.
- Module project: `lobby/server/spacetimedb` (`StdbModule.csproj`). Companion: `lobby/companion` (`Companion.csproj`).
- Build/deploy DLL is unaffected here — this plan is C#/module only.

---

### Task 0: Restart local STDB server + baseline publish

**Files:** none (environment setup, no commit)

- [ ] **Step 1: Start the local standalone server (background)**

Run (PowerShell, background):
```powershell
& "%USERPROFILE%\AppData\Local\SpacetimeDB\spacetime.exe" start
```
Expected: server listening on `0.0.0.0:3000`. Leave running.

- [ ] **Step 2: Baseline publish of the current module (clean DB)**

Run:
```powershell
& "%USERPROFILE%\AppData\Local\SpacetimeDB\spacetime.exe" publish pulsar-lobby -s local -p lobby/server/spacetimedb --delete-data=on-conflict -y
```
Expected: `Published to pulsar-lobby` (WARNING/RemoteException lines on stderr are noise).

- [ ] **Step 3: Confirm reducers/tables exist**

Run:
```powershell
& "%USERPROFILE%\AppData\Local\SpacetimeDB\spacetime.exe" sql pulsar-lobby "SELECT * FROM profile" -s local
```
Expected: empty result set, no error (baseline — profile currently public).

---

### Task 1: Profile read-isolation (RLS primary, per-user View fallback)

**Files:**
- Modify: `lobby/server/spacetimedb/Lib.cs` (Profile table + new visibility rule)

**Goal:** A non-owner client may only ever read its own `profile` row. The DB owner (module publisher) still sees all (operator).

- [ ] **Step 1: Add an RLS client-visibility filter for `profile`**

In `Lib.cs`, inside `public static partial class Module`, add (near the top of the class body, after the table declarations):

```csharp
    // Row-Level Security: every client may only see its own profile row.
    // The database owner (module publisher) bypasses this and sees all rows.
    [SpacetimeDB.ClientVisibilityFilter]
    public static readonly Filter PROFILE_RLS =
        new Filter.Sql("SELECT * FROM profile WHERE identity = :sender");
```

Keep `Profile` as `Public = true` (RLS filters rows of a public table; it stays subscribable, but only the caller's row is delivered).

- [ ] **Step 2: Build to confirm the RLS API compiles in STDB 2.5**

Run:
```powershell
& "%USERPROFILE%\AppData\Local\SpacetimeDB\spacetime.exe" build -p lobby/server/spacetimedb
```
Expected: build succeeds.

**Contingency (only if `ClientVisibilityFilter`/`Filter.Sql` does not exist in 2.5 and the build fails on those symbols):** remove the RLS block; make `Profile` **private** (delete `Public = true`) and add a per-user view instead:
```csharp
    [SpacetimeDB.View(Accessor = "MyProfile", Public = true)]
    public static Profile? MyProfile(ViewContext ctx)
    {
        return ctx.Db.Profile.Identity.Find(ctx.Sender) as Profile?;
    }
```
Record which mechanism was used in the commit message (Task 4 depends on it).

- [ ] **Step 3: Publish (keep existing data)**

Run:
```powershell
& "%USERPROFILE%\AppData\Local\SpacetimeDB\spacetime.exe" publish pulsar-lobby -s local -p lobby/server/spacetimedb -y
```
Expected: published, no schema-conflict error (RLS adds a rule, not a column).

- [ ] **Step 4: Seed one profile so isolation is observable**

Run (writes a profile for a throwaway anonymous identity, then that identity disconnects — the row persists):
```powershell
& "%USERPROFILE%\AppData\Local\SpacetimeDB\spacetime.exe" call pulsar-lobby upsert_profile '[1,2,3,4]' '1' -s local --anonymous
```
Expected: no error. Confirm the row exists as **owner**:
```powershell
& "%USERPROFILE%\AppData\Local\SpacetimeDB\spacetime.exe" sql pulsar-lobby "SELECT identity, version FROM profile" -s local
```
Expected: exactly one row (owner SQL bypasses RLS — sees it).

- [ ] **Step 5: Verify isolation — a fresh non-owner client sees zero rows**

Run (a brand-new anonymous identity subscribes; it owns no profile):
```powershell
& "%USERPROFILE%\AppData\Local\SpacetimeDB\spacetime.exe" subscribe pulsar-lobby "SELECT * FROM profile" --anonymous --num-updates 0 -s local
```
Expected: the initial result is an **empty** `profile` set — proving a non-owner cannot read the seeded profile. (Contrast: Step 4's owner SQL showed it.)

- [ ] **Step 6: Commit**

```powershell
git add lobby/server/spacetimedb/Lib.cs
```
Commit message (temp-file pattern, separate remove):
`fix(lobby/security): per-client RLS on profile — non-owner sees only own row`

---

### Task 2: LobbyMember private + denormalized MemberCount

**Files:**
- Modify: `lobby/server/spacetimedb/Lib.cs` (Lobby struct, LobbyMember table, JoinLobby, LeaveLobby, CreateLobby, HandleDisconnect)

**Goal:** Member identities never leave the server; the browser shows a count only.

- [ ] **Step 1: Add `MemberCount` to `Lobby`, make `LobbyMember` private**

In `Lib.cs`, add to the `Lobby` struct (after `MaxPlayers`):
```csharp
        public uint      MemberCount;   // denormalized joiner count for the browser
```
Change the `LobbyMember` table attribute from:
```csharp
    [SpacetimeDB.Table(Accessor = "LobbyMember", Public = true)]
```
to (drop `Public = true` → private; reducers still read it server-side):
```csharp
    [SpacetimeDB.Table(Accessor = "LobbyMember")]
```

- [ ] **Step 2: Initialize and maintain the count in the reducers**

In `CreateLobby`, set the new field in the inserted row:
```csharp
            MaxPlayers   = maxPlayers,
            MemberCount  = 0,
            CreatedAt    = ctx.Timestamp,
```

Replace the body of `JoinLobby` (after the existing full/idempotent checks) so it bumps the count. The full method becomes:
```csharp
    [SpacetimeDB.Reducer]
    public static void JoinLobby(ReducerContext ctx, ulong lobbyId)
    {
        var lobby = ctx.Db.Lobby.LobbyId.Find(lobbyId)
            ?? throw new Exception("lobby not found");
        if (lobby.Status != "open")
            throw new Exception("lobby not open");

        uint count = 0;
        foreach (var m in ctx.Db.LobbyMember.LobbyId.Filter(lobbyId))
        {
            if (m.MemberIdentity == ctx.Sender) return;  // already a member — idempotent
            count++;
        }
        if (count >= lobby.MaxPlayers)
            throw new Exception("lobby full");

        ctx.Db.LobbyMember.Insert(new LobbyMember
        {
            Id             = 0,
            LobbyId        = lobbyId,
            MemberIdentity = ctx.Sender,
            JoinedAt       = ctx.Timestamp,
        });

        lobby.MemberCount = count + 1;
        ctx.Db.Lobby.LobbyId.Update(lobby);
    }
```

Replace `LeaveLobby` so it decrements:
```csharp
    [SpacetimeDB.Reducer]
    public static void LeaveLobby(ReducerContext ctx, ulong lobbyId)
    {
        bool removed = false;
        foreach (var m in ctx.Db.LobbyMember.LobbyId.Filter(lobbyId).ToList())
        {
            if (m.MemberIdentity == ctx.Sender)
            {
                ctx.Db.LobbyMember.Id.Delete(m.Id);
                removed = true;
            }
        }
        if (removed && ctx.Db.Lobby.LobbyId.Find(lobbyId) is Lobby lobby)
        {
            lobby.MemberCount = lobby.MemberCount > 0 ? lobby.MemberCount - 1 : 0;
            ctx.Db.Lobby.LobbyId.Update(lobby);
        }
    }
```

Replace `HandleDisconnect` so host-close zeroes the count and a member-disconnect decrements its lobby:
```csharp
    [SpacetimeDB.Reducer(ReducerKind.ClientDisconnected)]
    public static void HandleDisconnect(ReducerContext ctx)
    {
        // Lobbies this identity HOSTS: drop members, close, zero count.
        foreach (var lobby in ctx.Db.Lobby.HostIdentity.Filter(ctx.Sender).ToList())
        {
            foreach (var m in ctx.Db.LobbyMember.LobbyId.Filter(lobby.LobbyId).ToList())
                ctx.Db.LobbyMember.Id.Delete(m.Id);

            var closed = lobby;
            closed.Status      = "closed";
            closed.MemberCount = 0;
            ctx.Db.Lobby.LobbyId.Update(closed);
        }

        // This identity's memberships in OTHER lobbies: remove + decrement each.
        foreach (var m in ctx.Db.LobbyMember.MemberIdentity.Filter(ctx.Sender).ToList())
        {
            ulong lid = m.LobbyId;
            ctx.Db.LobbyMember.Id.Delete(m.Id);
            if (ctx.Db.Lobby.LobbyId.Find(lid) is Lobby lb)
            {
                lb.MemberCount = lb.MemberCount > 0 ? lb.MemberCount - 1 : 0;
                ctx.Db.Lobby.LobbyId.Update(lb);
            }
        }
    }
```

- [ ] **Step 3: Publish with data reset (schema changed — new column)**

Run:
```powershell
& "%USERPROFILE%\AppData\Local\SpacetimeDB\spacetime.exe" publish pulsar-lobby -s local -p lobby/server/spacetimedb --delete-data=on-conflict -y
```
Expected: published.

- [ ] **Step 4: Verify `lobby` exposes `member_count` and `lobby_member` is private**

Run:
```powershell
& "%USERPROFILE%\AppData\Local\SpacetimeDB\spacetime.exe" call pulsar-lobby create_lobby '"Test"' '"127.0.0.1:7777"' '4' -s local --anonymous
& "%USERPROFILE%\AppData\Local\SpacetimeDB\spacetime.exe" sql pulsar-lobby "SELECT name, max_players, member_count, status FROM lobby" -s local
```
Expected: one row, `member_count = 0`, `status = closed` (the anon creator disconnected → HandleDisconnect closed it — correct behavior; member_count column present).

Run (non-owner cannot read the member table at all):
```powershell
& "%USERPROFILE%\AppData\Local\SpacetimeDB\spacetime.exe" subscribe pulsar-lobby "SELECT * FROM lobby_member" --anonymous --num-updates 0 -s local
```
Expected: an error or empty/denied result — `lobby_member` is no longer publicly subscribable. (Owner `spacetime sql` can still read it.)

- [ ] **Step 5: Commit**

```powershell
git add lobby/server/spacetimedb/Lib.cs
```
Commit message: `feat(lobby/security): private lobby_member + denormalized member_count`

---

### Task 3: Reducer input hardening (caps + validation)

**Files:**
- Modify: `lobby/server/spacetimedb/Lib.cs` (constants + validation in 4 reducers)

- [ ] **Step 1: Add limit constants + a host_addr validator**

In `Lib.cs`, at the top of `public static partial class Module`:
```csharp
    private const int  MAX_BLOB_BYTES  = 262144; // 256 KB profile blob cap
    private const int  MAX_NAME_LEN    = 32;     // account display name
    private const int  MAX_LOBBY_NAME  = 48;     // lobby name
    private const int  MAX_ADDR_LEN    = 48;     // "host:port"
    private const uint MAX_PLAYERS_CAP = 8;      // co-op upper bound

    private static bool IsValidHostAddr(string addr)
    {
        if (string.IsNullOrEmpty(addr) || addr.Length > MAX_ADDR_LEN) return false;
        int colon = addr.LastIndexOf(':');
        if (colon <= 0 || colon == addr.Length - 1) return false;
        if (!uint.TryParse(addr.Substring(colon + 1), out var port)) return false;
        return port >= 1 && port <= 65535;
    }
```

- [ ] **Step 2: Validate in `RegisterAccount`**

At the start of `RegisterAccount`, before the Find:
```csharp
        displayName = (displayName ?? "").Trim();
        if (displayName.Length < 1 || displayName.Length > MAX_NAME_LEN)
            throw new Exception("displayName length must be 1.." + MAX_NAME_LEN);
```

- [ ] **Step 3: Validate in `UpsertProfile`**

At the start of `UpsertProfile`, before the Find:
```csharp
        if (blob.Count > MAX_BLOB_BYTES)
            throw new Exception("profile blob exceeds " + MAX_BLOB_BYTES + " bytes");
```

- [ ] **Step 4: Validate in `CreateLobby`**

At the start of `CreateLobby`, before the Insert:
```csharp
        name = (name ?? "").Trim();
        if (name.Length < 1 || name.Length > MAX_LOBBY_NAME)
            throw new Exception("lobby name length must be 1.." + MAX_LOBBY_NAME);
        if (maxPlayers < 1 || maxPlayers > MAX_PLAYERS_CAP)
            throw new Exception("maxPlayers must be 1.." + MAX_PLAYERS_CAP);
        if (!IsValidHostAddr(hostAddr))
            throw new Exception("invalid hostAddr (expected host:port)");
```

- [ ] **Step 5: Validate in `UpdateLobbyStatus`**

After the host check in `UpdateLobbyStatus`:
```csharp
        if (status != "open" && status != "in_game" && status != "closed")
            throw new Exception("invalid status");
```

- [ ] **Step 6: Publish (no schema change → keep data)**

Run:
```powershell
& "%USERPROFILE%\AppData\Local\SpacetimeDB\spacetime.exe" publish pulsar-lobby -s local -p lobby/server/spacetimedb -y
```
Expected: published.

- [ ] **Step 7: Verify rejection + acceptance**

Run (oversized name and bad addr must be rejected; valid must pass):
```powershell
& "%USERPROFILE%\AppData\Local\SpacetimeDB\spacetime.exe" call pulsar-lobby create_lobby '"ThisLobbyNameIsWayTooLongToBeAcceptedByTheValidator___over48"' '"127.0.0.1:7777"' '4' -s local --anonymous
& "%USERPROFILE%\AppData\Local\SpacetimeDB\spacetime.exe" call pulsar-lobby create_lobby '"OK"' '"not-an-addr"' '4' -s local --anonymous
& "%USERPROFILE%\AppData\Local\SpacetimeDB\spacetime.exe" call pulsar-lobby create_lobby '"OK"' '"127.0.0.1:7777"' '99' -s local --anonymous
```
Expected: all three error (check `spacetime logs pulsar-lobby -n 20 -s local` for the thrown messages).
```powershell
& "%USERPROFILE%\AppData\Local\SpacetimeDB\spacetime.exe" call pulsar-lobby create_lobby '"OK"' '"127.0.0.1:7777"' '4' -s local --anonymous
```
Expected: no error.

- [ ] **Step 8: Commit**

```powershell
git add lobby/server/spacetimedb/Lib.cs
```
Commit message: `feat(lobby/security): reducer input caps (blob/name/maxPlayers/hostAddr/status)`

---

### Task 4: Companion reads via the isolated path + round-trip verify

**Files:**
- Modify: `lobby/companion/module_bindings/**` (regenerated)
- Modify: `lobby/companion/Program.cs` — **only if** Task 1 used the View fallback

- [ ] **Step 1: Regenerate client bindings against the new schema**

Run:
```powershell
& "%USERPROFILE%\AppData\Local\SpacetimeDB\spacetime.exe" generate --lang csharp --out-dir lobby/companion/module_bindings --module-path lobby/server/spacetimedb
```
Expected: bindings regenerated (Lobby now has `MemberCount`; RLS is server-side so the `Profile` table type is unchanged; View fallback would add a `MyProfile` accessor).

- [ ] **Step 2: Adjust the companion read path IF the View fallback was used**

If Task 1 used **RLS** (primary): no Program.cs change — the existing `SELECT * FROM profile` subscription now returns only the caller's row, and `FindMyProfile` still works. Skip to Step 3.

If Task 1 used the **View** fallback: in `Program.cs`, change the two profile subscriptions from `"SELECT * FROM profile"` to read the view. Replace the `pull-profile` subscription query:
```csharp
                    .Subscribe(new[] { "SELECT * FROM my_profile" });
```
and the `join` subscription's profile query likewise (`"SELECT * FROM lobby"`, `"SELECT * FROM my_profile"`), and change `FindMyProfile` to read the view rows:
```csharp
    private static Profile? FindMyProfile(DbConnection c)
    {
        foreach (var p in c.Db.MyProfile.Iter())
            return p;            // the view yields only the caller's row
        return null;
    }
```

- [ ] **Step 3: Build the companion**

Run:
```powershell
dotnet build lobby/companion/Companion.csproj -c Release
```
Expected: build succeeds.

- [ ] **Step 4: Verify push→pull round-trip still MATCHES (own profile readable)**

Run:
```powershell
$dir = "$env:LOCALAPPDATA\PULSAR"
[IO.File]::WriteAllBytes("$dir\rt_out.bin", [byte[]](1,2,3,4,5,6,7,8))
dotnet run --project lobby/companion/Companion.csproj -c Release -- register "RTUser"
dotnet run --project lobby/companion/Companion.csproj -c Release -- push-profile "$dir\rt_out.bin" 7
dotnet run --project lobby/companion/Companion.csproj -c Release -- pull-profile "$dir\rt_in.bin"
```
Then compare:
```powershell
if ((Get-FileHash "$dir\rt_out.bin").Hash -eq (Get-FileHash "$dir\rt_in.bin").Hash) { "MATCH" } else { "MISMATCH" }
```
Expected: `MATCH` (the companion's own identity reads its own profile through the isolated path).

- [ ] **Step 5: Commit**

```powershell
git add lobby/companion/module_bindings
```
(plus `lobby/companion/Program.cs` if the View fallback edited it)
Commit message: `chore(lobby/companion): regenerate bindings for isolated profile read`

---

### Task 5: Companion configurable endpoint (TLS-ready)

**Files:**
- Modify: `lobby/companion/Program.cs:21-22` (Uri/ModuleName constants → env-overridable)

- [ ] **Step 1: Make Uri + module name environment-overridable**

In `Program.cs`, replace:
```csharp
    private const string Uri = "http://127.0.0.1:3000";
    private const string ModuleName = "pulsar-lobby";
```
with:
```csharp
    private static readonly string Uri =
        Environment.GetEnvironmentVariable("PULSAR_STDB_URI") ?? "http://127.0.0.1:3000";
    private static readonly string ModuleName =
        Environment.GetEnvironmentVariable("PULSAR_STDB_DB") ?? "pulsar-lobby";
```
(For the VPS this becomes `PULSAR_STDB_URI=wss://<host>` / `https://<host>`; the SDK upgrades `https`→`wss`.)

- [ ] **Step 2: Build**

Run:
```powershell
dotnet build lobby/companion/Companion.csproj -c Release
```
Expected: build succeeds.

- [ ] **Step 3: Verify the override is honored**

Run (point at a bogus endpoint → connect error names that endpoint, proving it's used):
```powershell
$env:PULSAR_STDB_URI = "http://127.0.0.1:39999"
dotnet run --project lobby/companion/Companion.csproj -c Release -- list
Remove-Item Env:\PULSAR_STDB_URI
```
Expected: a connect ERROR/timeout referencing `127.0.0.1:39999` (not the default 3000).

- [ ] **Step 4: Commit**

```powershell
git add lobby/companion/Program.cs
```
Commit message: `feat(lobby/companion): env-configurable STDB endpoint (TLS-ready)`

---

### Task 6: Companion — DPAPI-encrypt the identity token at rest

**Files:**
- Modify: `lobby/companion/Companion.csproj` (add DPAPI package)
- Modify: `lobby/companion/Program.cs` (`LoadToken`/`SaveToken`)

- [ ] **Step 1: Add the DPAPI package**

In `Companion.csproj`, inside the existing `<ItemGroup>`:
```xml
    <PackageReference Include="System.Security.Cryptography.ProtectedData" Version="8.0.0" />
```

- [ ] **Step 2: Encrypt on save, decrypt on load (plaintext-migration fallback)**

In `Program.cs`, add the using at the top:
```csharp
using System.Security.Cryptography;
```
Replace `LoadToken`/`SaveToken` with:
```csharp
    private const string TokenMarker = "DPAPI1:";   // prefix for encrypted token files

    private static string? LoadToken()
    {
        if (!File.Exists(TokenPath)) return null;
        var raw = File.ReadAllText(TokenPath).Trim();
        if (raw.Length == 0) return null;
        if (!raw.StartsWith(TokenMarker)) return raw;   // legacy plaintext → migrate on next save
        try
        {
            var cipher = Convert.FromBase64String(raw.Substring(TokenMarker.Length));
            var plain  = ProtectedData.Unprotect(cipher, null, DataProtectionScope.CurrentUser);
            return System.Text.Encoding.UTF8.GetString(plain);
        }
        catch { return null; }   // unreadable → treat as no token (re-auth)
    }

    private static void SaveToken(string token)
    {
        try
        {
            var plain  = System.Text.Encoding.UTF8.GetBytes(token);
            var cipher = ProtectedData.Protect(plain, null, DataProtectionScope.CurrentUser);
            File.WriteAllText(TokenPath, TokenMarker + Convert.ToBase64String(cipher));
        }
        catch { /* best-effort */ }
    }
```

- [ ] **Step 3: Build**

Run:
```powershell
dotnet build lobby/companion/Companion.csproj -c Release
```
Expected: build succeeds (package restored).

- [ ] **Step 4: Verify the token file is ciphertext and identity is stable**

Run:
```powershell
Remove-Item "$env:LOCALAPPDATA\PULSAR\companion.token" -ErrorAction SilentlyContinue
dotnet run --project lobby/companion/Companion.csproj -c Release -- register "DpapiUser"
Get-Content "$env:LOCALAPPDATA\PULSAR\companion.token"
```
Expected: file content starts with `DPAPI1:` followed by Base64 (not a raw JWT). Run `register` again:
```powershell
dotnet run --project lobby/companion/Companion.csproj -c Release -- register "DpapiUser2"
```
Expected: `connected identity=…` shows the **same** identity as the first run (token decrypted + reused).

- [ ] **Step 5: Commit**

```powershell
git add lobby/companion/Companion.csproj lobby/companion/Program.cs
```
Commit message: `feat(lobby/companion): DPAPI-encrypt identity token at rest`

---

### Task 7: VPS TLS deployment notes

**Files:**
- Create: `lobby/DEPLOY.md`

- [ ] **Step 1: Write the deploy/TLS doc**

Create `lobby/DEPLOY.md`:
```markdown
# PULSAR Lobby Backend — VPS Deployment (Tier 1)

## Components
- SpacetimeDB standalone (`spacetime start`) — the module host.
- TLS terminator in front of it (clients must use wss/https).

## TLS (required before going online)
SpacetimeDB speaks ws/http on :3000. Put a TLS reverse proxy in front so
clients connect over wss. Example (Caddy):

    lobby.example.com {
        reverse_proxy 127.0.0.1:3000
    }

Caddy auto-provisions a Let's Encrypt cert; it upgrades http→https and
proxies the WebSocket transparently. (nginx with a `proxy_pass` +
`Upgrade`/`Connection` headers and a manual cert is equivalent.)

## Publish to the VPS server
    spacetime server add pulsar-vps --url https://lobby.example.com
    spacetime publish pulsar-lobby -s pulsar-vps -p lobby/server/spacetimedb -y

## Companion configuration
Point the companion at the TLS endpoint via environment:
    PULSAR_STDB_URI = wss://lobby.example.com   (or https://… ; SDK upgrades to wss)
    PULSAR_STDB_DB  = pulsar-lobby

## Security posture (Tier 1)
- Per-client RLS on `profile` (or per-user `MyProfile` view) — clients read
  only their own profile; the module-owner identity is the operator.
- `lobby_member` is private; the browser sees `member_count` only.
- Reducer input caps: blob ≤ 256 KB, name ≤ 32, lobby name ≤ 48, maxPlayers ≤ 8,
  host_addr validated, status whitelisted.
- Companion identity token encrypted at rest (Windows DPAPI, CurrentUser).
- Identity is still anonymous/machine-bound. Real accounts (OIDC) = Tier 2 / Beta.

## Operator notes
- The publishing identity owns the DB and can read all tables — protect that
  machine's `cli.toml`.
- Back up `AppData\Local\SpacetimeDB\data` (or the VPS data dir) for profile durability.
```

- [ ] **Step 2: Commit**

```powershell
git add lobby/DEPLOY.md
```
Commit message: `docs(lobby): VPS TLS deployment + Tier-1 security posture notes`

---

## Notes on commits
Use the project commit convention: write the message to `commit_msg.txt`, then
`git commit -F commit_msg.txt`, then a **separate** `Remove-Item commit_msg.txt`.
Never `git add -A`/`.`; stage only the listed paths. Never stage `extern/CommonLibSF`
or `scripts/PulsarFurnSync.pex`. No `git push`.
