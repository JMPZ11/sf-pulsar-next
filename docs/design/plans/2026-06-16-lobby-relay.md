# Lobby Relay Implementation Plan


**Goal:** Port-forward-free internet co-op — a transparent UDP relay on the VPS forwards ENet datagrams between host and client by session, so both peers only connect outbound.

**Architecture:** New C# relay daemon (in the Docker compose) cross-forwards UDP by session. PULSAR's host sends `REGISTER`, the client sends `JOIN` + connects its ENet to the relay instead of the host IP. ENet protocol is untouched (relay works below it). Relay is used only when a relay address is configured; otherwise the current direct/LAN path is unchanged.

**Tech Stack:** C# (.NET 8) relay + companion + STDB module; C++/ENet PULSAR DLL; Docker/Caddy on the VPS.

**Spec:** `docs/design/specs/2026-06-16-lobby-relay-design.md`

**Wire protocol (shared contract — keep C# relay and C++ DLL in sync):**
- 8-byte magic `PULSARLY` = `{0x50,0x55,0x4C,0x53,0x41,0x52,0x4C,0x59}`
- Control packet = magic(8) + type(1) + session(16) = **25 bytes**. type `1`=REGISTER, `2`=JOIN.
- Session = 16 raw bytes on the wire; 32 hex chars in JSON/lobby.
- Anything that isn't a 25-byte control packet is forwarded transparently to the other side of the session.

---

## File Structure

- Create `lobby/relay/Relay.csproj` — relay project (net8.0, console).
- Create `lobby/relay/Program.cs` — relay daemon + `--selftest`.
- Create `lobby/relay/Dockerfile` — runtime image.
- Create `lobby/relay/.gitignore` — `bin/ obj/`.
- Modify `lobby/deploy/docker-compose.yml` — add the `relay` service + `PULSAR_RELAY_PORT`.
- Create `include/Network/RelayProtocol.h` — shared protocol constants (C++ side).
- Modify `include/Lobby/LobbyHandoff.h` — add `relayAddr` + `relaySession` to `Handoff`.
- Modify `src/Lobby/LobbyHandoff.cpp` — parse `relay_addr` + `relay_session`.
- Modify `include/Network/NetworkHost.h` — relay fields + helper declarations.
- Modify `src/Network/NetworkHost.cpp` — register/join + relay-mode connect + keepalive.
- Modify `src/Main.cpp` — pass relay addr/session from the handoff into host/client start.
- Modify `lobby/server/spacetimedb/Lib.cs` — `RelaySession` column + `CreateLobby` param.
- Modify `lobby/companion/Program.cs` — generate session, relay addr, handoff/lobbies plumbing.
- Modify `lobby/companion-gui/MainForm.cs` — optional "Relay" field → daemon env.

---

### Task 1: Relay daemon — protocol + forwarding + self-test

**Files:**
- Create: `lobby/relay/Relay.csproj`
- Create: `lobby/relay/Program.cs`
- Create: `lobby/relay/.gitignore`

- [ ] **Step 1: Project file**

`lobby/relay/Relay.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <RootNamespace>PulsarRelay</RootNamespace>
    <AssemblyName>PulsarRelay</AssemblyName>
    <InvariantGlobalization>true</InvariantGlobalization>
  </PropertyGroup>
</Project>
```
`lobby/relay/.gitignore`:
```
bin/
obj/
```

- [ ] **Step 2: Write the daemon + self-test**

`lobby/relay/Program.cs`:
```csharp
using System.Diagnostics;
using System.Net;
using System.Net.Sockets;

namespace PulsarRelay;

// Transparent UDP relay for PULSAR co-op. Pairs a host and a client by a 16-byte
// session and cross-forwards every non-control datagram between them, so neither
// peer needs an inbound/forwarded port. ENet runs unmodified on top.
internal static class Program
{
    // Wire protocol — MUST match include/Network/RelayProtocol.h.
    private static readonly byte[] Magic = "PULSARLY"u8.ToArray(); // 8 bytes
    private const int  SessionBytes = 16;
    private const int  ControlLen   = 8 + 1 + SessionBytes; // 25
    private const byte TypeRegister = 1;
    private const byte TypeJoin     = 2;
    private const long SessionTtlMs = 10 * 60 * 1000;

    private sealed class Session { public IPEndPoint? Host; public IPEndPoint? Client; public long LastSeen; }

    private static int Main(string[] args)
    {
        if (args.Length > 0 && args[0] == "--selftest") return SelfTest();
        int port = int.TryParse(Environment.GetEnvironmentVariable("PULSAR_RELAY_PORT"), out var p) ? p : 7777;
        Run(port);
        return 0;
    }

    private static void Run(int port)
    {
        using var sock = new Socket(AddressFamily.InterNetwork, SocketType.Dgram, ProtocolType.Udp);
        sock.Bind(new IPEndPoint(IPAddress.Any, port));
        Console.WriteLine($"[relay] listening on udp/{port}");
        Loop(sock);
    }

    // Pulled out so the self-test can run the exact same logic on a background thread.
    private static void Loop(Socket sock)
    {
        var sessions = new Dictionary<string, Session>();
        var bySrc    = new Dictionary<IPEndPoint, (string key, bool isHost)>();
        var sw = Stopwatch.StartNew();
        long lastSweep = 0;
        var buf = new byte[2048];
        EndPoint remote = new IPEndPoint(IPAddress.Any, 0);

        while (true)
        {
            int n;
            try { n = sock.ReceiveFrom(buf, ref remote); }
            catch (SocketException) { continue; }   // e.g. ICMP port-unreachable
            long now = sw.ElapsedMilliseconds;
            var src = (IPEndPoint)remote;

            if (IsControl(buf, n, out byte type, out string key))
            {
                if (!sessions.TryGetValue(key, out var s)) { s = new Session(); sessions[key] = s; }
                var ep = new IPEndPoint(src.Address, src.Port);
                if (type == TypeRegister) { s.Host = ep;   bySrc[ep] = (key, true);  }
                else                      { s.Client = ep; bySrc[ep] = (key, false); }
                s.LastSeen = now;
            }
            else if (bySrc.TryGetValue(src, out var owner) && sessions.TryGetValue(owner.key, out var sess))
            {
                sess.LastSeen = now;
                var dst = owner.isHost ? sess.Client : sess.Host;
                if (dst != null) sock.SendTo(buf, n, SocketFlags.None, dst);
            }
            // else: unknown source, no control yet — drop.

            if (now - lastSweep > 60_000) { lastSweep = now; Sweep(sessions, bySrc, now); }
        }
    }

    private static bool IsControl(byte[] b, int n, out byte type, out string key)
    {
        type = 0; key = "";
        if (n != ControlLen) return false;
        for (int i = 0; i < Magic.Length; i++) if (b[i] != Magic[i]) return false;
        type = b[8];
        if (type != TypeRegister && type != TypeJoin) return false;
        key = Convert.ToHexString(b, 9, SessionBytes);
        return true;
    }

    private static void Sweep(Dictionary<string, Session> sessions,
                              Dictionary<IPEndPoint, (string, bool)> bySrc, long now)
    {
        foreach (var kv in sessions.Where(kv => now - kv.Value.LastSeen > SessionTtlMs).ToList())
        {
            if (kv.Value.Host   != null) bySrc.Remove(kv.Value.Host);
            if (kv.Value.Client != null) bySrc.Remove(kv.Value.Client);
            sessions.Remove(kv.Key);
            Console.WriteLine($"[relay] session {kv.Key[..8]}… expired");
        }
    }

    private static byte[] Control(byte type, byte[] session16)
    {
        var b = new byte[ControlLen];
        Array.Copy(Magic, b, 8);
        b[8] = type;
        Array.Copy(session16, 0, b, 9, SessionBytes);
        return b;
    }

    // Self-test: run the relay on an ephemeral port, simulate host+client, and
    // assert a datagram crosses in BOTH directions. Exit 0 on PASS, 1 on FAIL.
    private static int SelfTest()
    {
        var relaySock = new Socket(AddressFamily.InterNetwork, SocketType.Dgram, ProtocolType.Udp);
        relaySock.Bind(new IPEndPoint(IPAddress.Loopback, 0));
        var relayEp = (IPEndPoint)relaySock.LocalEndPoint!;
        new Thread(() => Loop(relaySock)) { IsBackground = true }.Start();

        var session = new byte[SessionBytes];
        for (byte i = 0; i < SessionBytes; i++) session[i] = (byte)(i + 1);

        using var host   = new Socket(AddressFamily.InterNetwork, SocketType.Dgram, ProtocolType.Udp);
        using var client = new Socket(AddressFamily.InterNetwork, SocketType.Dgram, ProtocolType.Udp);
        host.Bind(new IPEndPoint(IPAddress.Loopback, 0));
        client.Bind(new IPEndPoint(IPAddress.Loopback, 0));
        host.ReceiveTimeout = 2000; client.ReceiveTimeout = 2000;

        host.SendTo(Control(TypeRegister, session), relayEp);
        client.SendTo(Control(TypeJoin, session), relayEp);
        Thread.Sleep(100);   // let the relay register both endpoints

        var hello = "HELLO-FROM-HOST"u8.ToArray();
        host.SendTo(hello, relayEp);
        var rbuf = new byte[64];
        EndPoint from = new IPEndPoint(IPAddress.Any, 0);
        try
        {
            int rn = client.ReceiveFrom(rbuf, ref from);
            if (rn != hello.Length || !rbuf.AsSpan(0, rn).SequenceEqual(hello))
            { Console.WriteLine("[selftest] FAIL: host->client payload mismatch"); return 1; }
        }
        catch (SocketException) { Console.WriteLine("[selftest] FAIL: host->client not forwarded"); return 1; }

        var back = "HELLO-FROM-CLIENT"u8.ToArray();
        client.SendTo(back, relayEp);
        try
        {
            int rn = host.ReceiveFrom(rbuf, ref from);
            if (rn != back.Length || !rbuf.AsSpan(0, rn).SequenceEqual(back))
            { Console.WriteLine("[selftest] FAIL: client->host payload mismatch"); return 1; }
        }
        catch (SocketException) { Console.WriteLine("[selftest] FAIL: client->host not forwarded"); return 1; }

        Console.WriteLine("[selftest] PASS");
        return 0;
    }
}
```

- [ ] **Step 3: Run the self-test, verify it PASSES**

Run: `cd lobby/relay && dotnet run -- --selftest`
Expected: prints `[relay] listening on udp/<port>` then `[selftest] PASS`, exit 0.

- [ ] **Step 4: Commit**
```bash
git add lobby/relay/Relay.csproj lobby/relay/Program.cs lobby/relay/.gitignore
git commit -F commit_msg.txt   # see repo convention (temp file, separate remove)
```
Message subject: `feat(lobby/relay): transparent UDP relay daemon + self-test`

---

### Task 2: Relay in Docker compose

**Files:**
- Create: `lobby/relay/Dockerfile`
- Modify: `lobby/deploy/docker-compose.yml`
- Modify: `lobby/deploy/README.md` (firewall: add UDP relay port)

- [ ] **Step 1: Dockerfile (build + runtime)**

`lobby/relay/Dockerfile`:
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY Relay.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish Relay.csproj -c Release -o /app

FROM mcr.microsoft.com/dotnet/runtime:8.0
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "PulsarRelay.dll"]
```

- [ ] **Step 2: Add the relay service to the compose**

In `lobby/deploy/docker-compose.yml`, add under `services:` (after `caddy`):
```yaml
  relay:
    build:
      context: ../relay
    container_name: pulsar-relay
    environment:
      PULSAR_RELAY_PORT: ${PULSAR_RELAY_PORT:-7777}
    ports:
      - "${PULSAR_RELAY_PORT:-7777}:${PULSAR_RELAY_PORT:-7777}/udp"
    restart: unless-stopped
```

- [ ] **Step 3: Validate compose**

Run: `cd lobby/deploy && DOMAIN=x.example.com ACME_EMAIL=a@b.c docker compose config`
Expected: parses without error; `relay` service present with the udp port mapping.

- [ ] **Step 4: Document firewall**

In `lobby/deploy/README.md` "Firewall (ufw)" section, add the line:
```bash
sudo ufw allow 7777/udp     # relay (game traffic, NAT-free)
```

- [ ] **Step 5: Commit**
Message subject: `feat(lobby/relay): dockerize relay + add to compose`

---

### Task 3: PULSAR C++ — relay protocol, handoff fields, register/join

**Files:**
- Create: `include/Network/RelayProtocol.h`
- Modify: `include/Lobby/LobbyHandoff.h:17-24`
- Modify: `src/Lobby/LobbyHandoff.cpp:75-87`
- Modify: `include/Network/NetworkHost.h:21-22,65-117`
- Modify: `src/Network/NetworkHost.cpp:209-294`
- Modify: `src/Main.cpp:1610-1666`

- [ ] **Step 1: Shared protocol header**

`include/Network/RelayProtocol.h`:
```cpp
#pragma once
#include <cstdint>
#include <cstddef>
#include <array>

// Wire contract shared with the C# relay (lobby/relay/Program.cs). Control packet
// = magic(8) + type(1) + session(16) = 25 bytes; anything else is forwarded.
namespace Pulsar::Network::Relay
{
    inline constexpr std::array<uint8_t, 8> kMagic = { 'P','U','L','S','A','R','L','Y' };
    inline constexpr size_t  kSessionBytes = 16;
    inline constexpr size_t  kControlLen   = 8 + 1 + kSessionBytes; // 25
    inline constexpr uint8_t kTypeRegister = 1;
    inline constexpr uint8_t kTypeJoin     = 2;

    // Decode 32 hex chars -> 16 bytes. Returns false on bad length/char.
    inline bool DecodeSession(const std::string& hex, uint8_t out[kSessionBytes])
    {
        if (hex.size() != kSessionBytes * 2) return false;
        auto nyb = [](char c) -> int {
            if (c >= '0' && c <= '9') return c - '0';
            if (c >= 'a' && c <= 'f') return c - 'a' + 10;
            if (c >= 'A' && c <= 'F') return c - 'A' + 10;
            return -1;
        };
        for (size_t i = 0; i < kSessionBytes; ++i) {
            int hi = nyb(hex[i * 2]), lo = nyb(hex[i * 2 + 1]);
            if (hi < 0 || lo < 0) return false;
            out[i] = static_cast<uint8_t>((hi << 4) | lo);
        }
        return true;
    }

    inline std::array<uint8_t, kControlLen> MakeControl(uint8_t type, const uint8_t session[kSessionBytes])
    {
        std::array<uint8_t, kControlLen> b{};
        for (size_t i = 0; i < kMagic.size(); ++i) b[i] = kMagic[i];
        b[8] = type;
        for (size_t i = 0; i < kSessionBytes; ++i) b[9 + i] = session[i];
        return b;
    }
}
```

- [ ] **Step 2: Add relay fields to the Handoff struct**

In `include/Lobby/LobbyHandoff.h`, replace the `Handoff` struct (lines 17-24) with:
```cpp
    struct Handoff
    {
        bool        found   = false;
        std::string role;            // "host" | "client"
        std::string hostIP;          // parsed from "ip:port" (direct/LAN fallback)
        uint16_t    port    = 0;
        uint32_t    version = 0;
        std::string relayAddr;       // "host:port" of the relay; empty = direct mode
        std::string relaySession;    // 32 hex chars; empty = direct mode
    };
```

- [ ] **Step 3: Parse the new handoff fields**

In `src/Lobby/LobbyHandoff.cpp`, inside `ReadHandoff()`, after the `host_addr` parse block (after line 87, before `return h;`), add:
```cpp
        h.relaySession = ExtractStr(j, "relay_session");
        std::string relay = ExtractStr(j, "relay_addr");
        if (!relay.empty()) h.relayAddr = relay;
```

- [ ] **Step 4: NetworkHost.h — relay state + helpers**

In `include/Network/NetworkHost.h`:
- Change `StartHost`/`Connect` decls (lines 21-22) to carry the relay session:
```cpp
        bool StartHost(uint16_t a_port = DEFAULT_PORT);
        bool Connect(const char* a_address, uint16_t a_port = DEFAULT_PORT);
        // Enable relay mode: register/join via the relay instead of a direct
        // peer. a_relayAddr = "host:port", a_sessionHex = 32 hex chars. Call
        // BEFORE StartHost/Connect. No-op if either is empty (direct mode).
        void SetRelay(const std::string& a_relayAddr, const std::string& a_sessionHex);
```
- In the `private:` section (after line 71, near `m_isHost`), add:
```cpp
        // Relay mode (NAT-free). Empty session = direct mode (unchanged path).
        bool        m_relayMode = false;
        ENetAddress m_relayAddress{};
        uint8_t     m_relaySession[16] = {};
        uint64_t    m_relayKeepaliveAt = 0;   // GetTickCount64() of next REGISTER
        void SendRelayControl(uint8_t a_type);  // sends via m_host->socket
```

- [ ] **Step 5: NetworkHost.cpp — SetRelay + register/join + keepalive**

In `src/Network/NetworkHost.cpp`, add the include at the top with the other includes:
```cpp
#include "Network/RelayProtocol.h"
```
Add `SetRelay` and `SendRelayControl` (place just above `StartHost`, before line 209):
```cpp
    void NetworkHost::SetRelay(const std::string& a_relayAddr, const std::string& a_sessionHex)
    {
        m_relayMode = false;
        if (a_relayAddr.empty() || a_sessionHex.empty()) return;
        if (!Relay::DecodeSession(a_sessionHex, m_relaySession)) {
            REX::WARN("[PULSAR] [Relay] bad session hex — falling back to direct"sv);
            return;
        }
        auto colon = a_relayAddr.rfind(':');
        if (colon == std::string::npos) {
            REX::WARN("[PULSAR] [Relay] relay addr needs host:port — direct fallback"sv);
            return;
        }
        std::string host = a_relayAddr.substr(0, colon);
        m_relayAddress = ENetAddress{};
        enet_address_set_host(&m_relayAddress, host.c_str());
        m_relayAddress.port =
            static_cast<uint16_t>(std::strtoul(a_relayAddr.c_str() + colon + 1, nullptr, 10));
        m_relayMode = true;
        REX::INFO("[PULSAR] [Relay] enabled via {}"sv, a_relayAddr);
    }

    void NetworkHost::SendRelayControl(uint8_t a_type)
    {
        if (!m_relayMode || !m_host) return;
        auto pkt = Relay::MakeControl(a_type, m_relaySession);
        ENetBuffer eb{};
        eb.data       = pkt.data();
        eb.dataLength = pkt.size();
        enet_socket_send(m_host->socket, &m_relayAddress, &eb, 1);
    }
```
In `StartHost` (after `m_isHost = true;` at line 229), add:
```cpp
        if (m_relayMode) {
            SendRelayControl(Relay::kTypeRegister);
            m_relayKeepaliveAt = GetTickCount64() + 15000;
            REX::INFO("[PULSAR] [Relay] host registered session"sv);
        }
```
In `DoConnectInternal`, replace the address setup + connect (lines 279-283) with:
```cpp
        ENetAddress address{};
        if (m_relayMode) {
            address = m_relayAddress;
            for (int i = 0; i < 3; ++i) SendRelayControl(Relay::kTypeJoin);  // beat reordering
            REX::INFO("[PULSAR] [Relay] client joined session, connecting via relay"sv);
        } else {
            enet_address_set_host(&address, a_address);
            address.port = a_port;
        }

        m_peer = enet_host_connect(m_host, &address, NUM_CHANNELS, 0);
```
In `Update()`, near the top (after the pending-connect drain block, before the `m_host` null-check), add the host keepalive:
```cpp
        if (m_relayMode && m_isHost && m_host) {
            uint64_t nowMs = GetTickCount64();
            if (nowMs >= m_relayKeepaliveAt) {
                std::lock_guard lk(m_enetMutex);
                SendRelayControl(Relay::kTypeRegister);
                m_relayKeepaliveAt = nowMs + 15000;
            }
        }
```

- [ ] **Step 6: Main.cpp — wire relay from the handoff**

In `src/Main.cpp`, where the handoff is read and host/client started (around lines 1610-1666), set the relay BEFORE `StartHost`/`Connect`. After the handoff fields are applied to `g_config` (after line 1622) add:
```cpp
                if (!ho.relaySession.empty() && !ho.relayAddr.empty()) {
                    network.SetRelay(ho.relayAddr, ho.relaySession);
                    REX::INFO("[PULSAR] [Relay] handoff relay {} session set"sv, ho.relayAddr);
                }
```
(The existing `StartHost(g_config.port)` / `Connect(g_config.hostIP, g_config.port)` calls are unchanged — relay mode is already armed.)

- [ ] **Step 7: Build**

Run: `cmake --build build/vs --config RelWithDebInfo --target PULSAR`
Expected: builds with 0 errors. (`GetTickCount64`/`std::strtoul` already used elsewhere in these files.)

- [ ] **Step 8: Smoke against the relay (no game yet)**

Start the relay locally: `cd lobby/relay && dotnet run` (listens udp/7777).
There's no standalone harness for the DLL; this step is verified in Task 7's
integration. Mark done once the build is clean and the relay is running.

- [ ] **Step 9: Commit**
Message subject: `feat(net): relay-mode register/join in NetworkHost + handoff fields`

---

### Task 4: STDB module — RelaySession on Lobby

**Files:**
- Modify: `lobby/server/spacetimedb/Lib.cs:43-57,130-152`

- [ ] **Step 1: Add the column**

In the `Lobby` struct (lines 43-57), add after `HostAddr`:
```csharp
        public string    RelaySession;  // 32 hex chars (16-byte session); "" = direct/LAN
```

- [ ] **Step 2: Extend CreateLobby**

Replace the `CreateLobby` signature + insert (lines 131-151) with:
```csharp
    public static void CreateLobby(ReducerContext ctx, string name, string hostAddr,
                                   uint maxPlayers, string relaySession)
    {
        name = (name ?? "").Trim();
        if (name.Length < 1 || name.Length > MAX_LOBBY_NAME)
            throw new Exception("lobby name length must be 1.." + MAX_LOBBY_NAME);
        if (maxPlayers < 1 || maxPlayers > MAX_PLAYERS_CAP)
            throw new Exception("maxPlayers must be 1.." + MAX_PLAYERS_CAP);
        if (!IsValidHostAddr(hostAddr))
            throw new Exception("invalid hostAddr (expected host:port)");
        relaySession = (relaySession ?? "").Trim();
        if (relaySession.Length != 0 && relaySession.Length != 32)
            throw new Exception("relaySession must be empty or 32 hex chars");

        ctx.Db.Lobby.Insert(new Lobby
        {
            LobbyId      = 0,
            HostIdentity = ctx.Sender,
            HostAddr     = hostAddr,
            RelaySession = relaySession,
            Name         = name,
            Status       = "open",
            MaxPlayers   = maxPlayers,
            MemberCount  = 0,
            CreatedAt    = ctx.Timestamp,
        });
    }
```

- [ ] **Step 3: Build the module**

Run: `cd lobby/server/spacetimedb && dotnet build`
Expected: builds (wasi-wasm) with 0 errors.

- [ ] **Step 4: Commit**
Message subject: `feat(lobby/module): RelaySession column + CreateLobby param`

> Publish to the live VPS DB is done in Task 7 (needs the running server). Adding
> a column to a live table needs `--delete-data` or a migration — the spec notes
> this; for the test DB a republish with `--delete-data always` is acceptable.

---

### Task 5: Companion — generate session, relay addr, plumbing

**Files:**
- Modify: `lobby/companion/Program.cs` (host cmd, join cmd, WriteHandoff, WriteLobbies, CreateLobby call)

- [ ] **Step 1: Relay addr env + session generator**

Add near the other env reads (top of `Program`, by `Uri`/`ModuleName`):
```csharp
    private static readonly string RelayAddr =
        Environment.GetEnvironmentVariable("PULSAR_RELAY_ADDR") ?? "";

    private static string NewSessionHex()
    {
        var b = new byte[16];
        System.Security.Cryptography.RandomNumberGenerator.Fill(b);
        return Convert.ToHexString(b);   // 32 uppercase hex chars
    }
```

- [ ] **Step 2: WriteHandoff carries relay fields**

Replace `WriteHandoff` (lines 426-435) with:
```csharp
    private static void WriteHandoff(string role, string hostAddr, uint version,
                                     string relayAddr, string relaySession)
    {
        string json =
            "{\n" +
            $"  \"role\": \"{role}\",\n" +
            $"  \"host_addr\": \"{hostAddr}\",\n" +
            $"  \"relay_addr\": \"{relayAddr}\",\n" +
            $"  \"relay_session\": \"{relaySession}\",\n" +
            $"  \"version\": {version}\n" +
            "}\n";
        File.WriteAllText(HandoffPath, json);
    }
```

- [ ] **Step 3: serve host-cmd — generate + pass session**

In `HandleCommandFile`, the `action == "host"` branch (lines 365-375), replace with:
```csharp
            if (action == "host")
            {
                string name = root.TryGetProperty("name", out var n) ? (n.GetString() ?? "Host") : "Host";
                string addr = root.TryGetProperty("addr", out var ad) ? (ad.GetString() ?? "") : "";
                uint max = root.TryGetProperty("max", out var m) ? m.GetUInt32() : 4;
                string session = RelayAddr.Length > 0 ? NewSessionHex() : "";
                c.Reducers.RegisterAccount(name);
                c.Reducers.CreateLobby(name, addr, max, session);
                WriteHandoff("host", addr, 0, RelayAddr, session);
                WriteAck(cmdId, true, null, addr);
                Console.WriteLine($"[companion] serve: hosted '{name}' @ {addr} relay={RelayAddr}");
            }
```

- [ ] **Step 4: serve join-cmd — read the lobby's session**

In the `action == "join"` branch (lines 376-391), replace the body with:
```csharp
            else if (action == "join")
            {
                ulong lobbyId = root.TryGetProperty("lobbyId", out var li) ? li.GetUInt64() : 0;
                string addr   = root.TryGetProperty("addr", out var ad) ? (ad.GetString() ?? "") : "";
                string session = "";
                foreach (var l in c.Db.Lobby.Iter())
                    if (l.LobbyId == lobbyId) { session = l.RelaySession ?? ""; break; }
                var mine = FindMyProfile(c);
                uint version = 0;
                if (mine is not null)
                {
                    File.WriteAllBytes(ProfileInPath, mine.Blob.ToArray());
                    version = mine.Version;
                }
                c.Reducers.JoinLobby(lobbyId);
                WriteHandoff("client", addr, version, RelayAddr, session);
                WriteAck(cmdId, true, null, addr);
                Console.WriteLine($"[companion] serve: joined lobby {lobbyId} relay={RelayAddr}");
            }
```

- [ ] **Step 5: Update the other WriteHandoff/CreateLobby call sites**

The CLI `host` case (line ~278) and CLI `join` case (line ~306) also call
`CreateLobby`/`WriteHandoff`. Update them to compile against the new signatures:
- CLI `host`: `c.Reducers.CreateLobby(name, addr, max, "");` and `WriteHandoff("host", addr, 0, "", "");`
- CLI `join`: `WriteHandoff("client", lobby.HostAddr, version, "", "");`
(CLI paths stay direct-mode; the GUI `serve` path is the relay path.)

- [ ] **Step 6: lobbies.json exposes the session (optional, for the client)**

`WriteLobbies` already serializes lobby fields. The join path reads the session
from `c.Db.Lobby` directly (Step 4), so no JSON change is strictly required.
Skip unless the GUI needs it.

- [ ] **Step 7: Build + regenerate bindings**

The new `CreateLobby` arg + `RelaySession` column change the module bindings.
Run from `lobby/companion`:
```bash
spacetime generate --lang csharp --out-dir module_bindings --project-path ../server/spacetimedb
dotnet build -c Release
```
Expected: bindings regenerate; build 0 errors. (Requires the module from Task 4.)

- [ ] **Step 8: Commit**
Message subject: `feat(lobby/companion): relay session generation + handoff plumbing`

---

### Task 6: GUI — optional Relay field

**Files:**
- Modify: `lobby/companion-gui/MainForm.cs` (mirror the existing Server field)

- [ ] **Step 1: Add the field + pass it as env**

- Add a `private readonly TextBox _relay = new();` next to `_server`.
- In `BuildUi`, add a "Relay (Host:Port)" label + `_relay` on the path rows
  (mirror the `lblServer`/`_server` block), default `_relay.Text = ""`.
- Add `_relay` to the `Controls.AddRange(...)` list and to `SetConnectedState`
  (`_relay.Enabled = !on;`).
- In `ToggleDaemon`, after setting `PULSAR_STDB_URI`, add:
```csharp
            if (_relay.Text.Trim().Length > 0)
                psi.Environment["PULSAR_RELAY_ADDR"] = _relay.Text.Trim();
```

- [ ] **Step 2: Build**

Run: `cd lobby/companion-gui && dotnet build -c Release`
Expected: 0 errors.

- [ ] **Step 3: Commit**
Message subject: `feat(lobby/gui): optional relay address field`

---

### Task 7: Integration — deploy + 2-player NAT-free test

**Files:** none (deploy + manual test)

- [ ] **Step 1: Republish the module to the VPS**
```bash
spacetime publish pulsar-lobby -s pulsar-vps -p lobby/server/spacetimedb --delete-data always -y
```
Expected: publish succeeds; `spacetime sql -s pulsar-vps pulsar-lobby "SELECT * FROM lobby"` shows the `RelaySession` column.

- [ ] **Step 2: Bring up the relay on the VPS**
```bash
cd lobby/deploy && sudo docker compose up -d --build relay
sudo docker compose logs relay   # expect "[relay] listening on udp/7777"
sudo ufw allow 7777/udp
```

- [ ] **Step 3: Publish the GUI/companion build**

Re-publish the self-contained tooling (as before) and copy to the NAS
`deploy\LobbyTools\` so both players have the relay-aware build.

- [ ] **Step 4: Two players, both behind NAT, NO port-forward**

Both set in PulsarLobby: Server `wss://<DOMAIN>`, Relay `<VPS-IP>:7777`.
Host → "Lobby hosten"; Client → select lobby → "Beitreten". Both launch.

- [ ] **Step 5: Verify**

- Relay log: a REGISTER and a JOIN for the same session.
- Host `PULSAR.log`: `[Relay] host registered session`.
- Client `PULSAR.log`: `[Relay] client joined session, connecting via relay`.
- Both: ENet connects → `peer … slot 1` / clone synced (as in the LAN test),
  `loss=0%`, with **no port-forwarding on either side**.

---

## Self-Review

**1. Spec coverage:**
- Wire protocol (magic/control/forward) → Task 1 + RelayProtocol.h (Task 3). ✓
- Relay daemon in compose → Tasks 1-2. ✓
- NetworkHost host-register/client-connect/keepalive → Task 3. ✓
- Lib.cs RelaySession → Task 4. ✓
- Companion/handoff plumbing → Task 5; handoff parse → Task 3. ✓
- GUI relay field → Task 6. ✓
- "Relay only when configured" → empty relayAddr/session ⇒ `m_relayMode=false`/direct (Task 3 SetRelay; Task 5 RelayAddr default ""). ✓
- Hex/16-byte session encoding → `DecodeSession` (Task 3), `NewSessionHex` (Task 5). ✓
- Main-risk (REGISTER over ENet's socket) validated first → relay self-test (Task 1) + build/integration (Tasks 3,7). Note: the socket-send is only fully exercised in Task 7; acceptable since there's no DLL unit harness.

**2. Placeholder scan:** No TBD/TODO; each code step has complete code. Task 3 Step 8 explicitly defers the live exercise to Task 7 (stated, not a placeholder).

**3. Type consistency:** `SetRelay(string,string)`, `relaySession` (32 hex) everywhere, `CreateLobby(name,hostAddr,maxPlayers,relaySession)` matches the companion call, `WriteHandoff(role,hostAddr,version,relayAddr,relaySession)` matches all call sites, handoff JSON keys `relay_addr`/`relay_session` match `ExtractStr` reads. Magic/lengths identical in C# (`Magic`/`ControlLen`) and C++ (`kMagic`/`kControlLen`). ✓
