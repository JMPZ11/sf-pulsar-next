# P2P-Primary Co-op (NAT Hole-Punching) Implementation Plan


**Goal:** Make direct P2P the primary co-op transport; the relay becomes rendezvous (endpoint exchange) + hole-punch coordinator + fallback, so game traffic normally flows host↔client directly (no VPS load).

**Architecture:** Direct-first, relay-fallback. Relay sends each peer the other's public endpoint (new `PEER` control msg). Both peers hole-punch on the ENet socket; client `enet_host_connect`s directly to the host. If the direct path doesn't establish within a timeout, connect via the relay (today's path). The `PEER` datagram is caught by ENet's `host->intercept` callback (same socket ENet uses → correct NAT mapping). `PULSAR_FORCE_RELAY=1` forces the proven relay-always path.

**Tech Stack:** C++ (ENet, CommonLibSF SFSE plugin), C# (.NET relay), shared `RelayProtocol.h`.

**Reference spec:** `docs/design/specs/2026-06-17-p2p-hole-punching-design.md`

**Verified facts:** ENet `ENetInterceptCallback = int(ENetHost*, ENetEvent*)`; raw datagram in `host->receivedData[receivedDataLength]`, source `host->receivedAddress`; return 1=consume, 0=pass. `ENetAddress.host` = network-order IPv4 uint32, `.port` = host-order uint16. NetworkHost members: `m_host`, `m_peer`, `m_relayMode`, `m_relayAddress`, `m_relaySession[16]`, `m_relayKeepaliveAt`, `SendRelayControl`, `DoConnectInternal`, `Update`, `OnConnect`.

---

## File Structure
- `include/Network/RelayProtocol.h` — add `kTypePeer`, `kPeerLen`, `MakePeer`, `ParsePeer`, `IsControlMagic`.
- `lobby/relay/Program.cs` — send `PEER` to both peers once paired; `--selftest` asserts it.
- `include/Network/NetworkHost.h` — rendezvous state + members.
- `src/Network/NetworkHost.cpp` — intercept callback, rendezvous/punch/direct/fallback state machine, FORCE_RELAY, keepalive-stop-on-direct.

---

## Task 1: Protocol — `PEER` message in `RelayProtocol.h`

**Files:** Modify `include/Network/RelayProtocol.h`

- [ ] **Step 1: Add the type, length, and helpers**

After `kTypeJoin` (line 15) add:
```cpp
    inline constexpr uint8_t kTypePeer = 3;
    inline constexpr size_t  kPeerLen  = kControlLen + 6; // 25 + IPv4(4) + port(2) = 31
```

After `MakeControl` (before the closing `}` of the namespace) add:
```cpp
    // PEER (relay -> peer): control header + peer's public endpoint.
    // ip = 4 bytes network order (as observed by the relay); port = 2 bytes big-endian.
    inline std::array<uint8_t, kPeerLen> MakePeer(const uint8_t session[kSessionBytes],
                                                  uint32_t ipNetOrder, uint16_t portHostOrder)
    {
        std::array<uint8_t, kPeerLen> b{};
        for (size_t i = 0; i < kMagic.size(); ++i) b[i] = kMagic[i];
        b[8] = kTypePeer;
        for (size_t i = 0; i < kSessionBytes; ++i) b[9 + i] = session[i];
        // ipNetOrder is already in network byte order; copy as-is.
        b[25] = static_cast<uint8_t>(ipNetOrder & 0xFF);
        b[26] = static_cast<uint8_t>((ipNetOrder >> 8) & 0xFF);
        b[27] = static_cast<uint8_t>((ipNetOrder >> 16) & 0xFF);
        b[28] = static_cast<uint8_t>((ipNetOrder >> 24) & 0xFF);
        b[29] = static_cast<uint8_t>((portHostOrder >> 8) & 0xFF); // big-endian
        b[30] = static_cast<uint8_t>(portHostOrder & 0xFF);
        return b;
    }

    // Cheap classify for the ENet intercept: magic + a known control type.
    inline bool IsControlMagic(const uint8_t* d, size_t n)
    {
        if (n < kControlLen) return false;
        for (size_t i = 0; i < kMagic.size(); ++i) if (d[i] != kMagic[i]) return false;
        return d[8] == kTypeRegister || d[8] == kTypeJoin || d[8] == kTypePeer;
    }

    // Parse a PEER datagram. Returns false unless it's a well-formed PEER.
    // Out: ipNetOrder (for ENetAddress.host), portHostOrder (for ENetAddress.port).
    inline bool ParsePeer(const uint8_t* d, size_t n, uint32_t& ipNetOrder, uint16_t& portHostOrder)
    {
        if (n < kPeerLen || !IsControlMagic(d, n) || d[8] != kTypePeer) return false;
        ipNetOrder = static_cast<uint32_t>(d[25]) | (static_cast<uint32_t>(d[26]) << 8) |
                     (static_cast<uint32_t>(d[27]) << 16) | (static_cast<uint32_t>(d[28]) << 24);
        portHostOrder = static_cast<uint16_t>((static_cast<uint16_t>(d[29]) << 8) | d[30]);
        return true;
    }
```

- [ ] **Step 2: Commit** (header compiles via consumers; no standalone test)
```bash
git add include/Network/RelayProtocol.h
git commit -m "feat(net): RelayProtocol PEER message (kTypePeer + MakePeer/ParsePeer/IsControlMagic)"
```

---

## Task 2: Relay sends `PEER` once a session is paired

**Files:** Modify `lobby/relay/Program.cs`

- [ ] **Step 1: Add the PEER constant + builder**

Add the type constant near `TypeJoin`:
```csharp
    private const byte TypePeer = 3;
```

Add a builder + sender (near `Control(...)`):
```csharp
    // PEER (relay -> peer): 25-byte control + peer endpoint (4-byte IPv4 net order + 2-byte port BE).
    private static byte[] PeerMsg(byte[] session16, IPEndPoint peer)
    {
        var b = new byte[ControlLen + 6];
        Array.Copy(Magic, b, 8);
        b[8] = TypePeer;
        Array.Copy(session16, 0, b, 9, SessionBytes);
        var ip = peer.Address.MapToIPv4().GetAddressBytes(); // 4 bytes, network order
        Array.Copy(ip, 0, b, 25, 4);
        b[29] = (byte)((peer.Port >> 8) & 0xFF); // big-endian
        b[30] = (byte)(peer.Port & 0xFF);
        return b;
    }
```

- [ ] **Step 2: Send PEER to both when the pair is complete**

In `Loop`, after the control-handling block updates `s.Host`/`s.Client` (right before `s.LastSeen = now;`), add:
```csharp
                // Rendezvous: once both endpoints are known, tell each peer the
                // other's public endpoint so they can hole-punch + connect directly.
                // Re-sent on every control packet (cheap, beats loss) until TTL.
                if (s.Host != null && s.Client != null)
                {
                    var toHost = PeerMsg(b.AsSpan(9, SessionBytes).ToArray(), s.Client);
                    var toClient = PeerMsg(b.AsSpan(9, SessionBytes).ToArray(), s.Host);
                    sock.SendTo(toHost, s.Host);
                    sock.SendTo(toClient, s.Client);
                }
```
(`b` is the receive buffer; `b.AsSpan(9, SessionBytes).ToArray()` is this session's 16-byte id — the control just parsed.)

> NOTE: `IsControl` returns the parsed key but the raw session bytes live at `b[9..25]`. If `b` isn't in scope there, reuse `Convert.FromHexString(key)` instead — `key` is the hex session already in scope. Prefer `Convert.FromHexString(key)` for clarity:
> `var sess = Convert.FromHexString(key); var toHost = PeerMsg(sess, s.Client); ...`

- [ ] **Step 3: Extend `--selftest` to assert PEER delivery**

In `SelfTest`, after the host REGISTER + client JOIN + `Thread.Sleep(100)`, before the payload test, add a receive-and-check that each side got a PEER carrying the other's loopback endpoint:
```csharp
        // Each side should receive a PEER with the other's endpoint.
        var pbuf = new byte[64];
        EndPoint pfrom = new IPEndPoint(IPAddress.Any, 0);
        host.SendTo(Control(TypeRegister, session), relayEp);   // re-trigger pairing
        client.SendTo(Control(TypeJoin, session), relayEp);
        Thread.Sleep(100);
        try
        {
            int pn = host.ReceiveFrom(pbuf, ref pfrom);
            if (pn != 31 || pbuf[8] != TypePeer) { Console.WriteLine("[selftest] FAIL: host did not get PEER"); return 1; }
        }
        catch (SocketException) { Console.WriteLine("[selftest] FAIL: host PEER not delivered"); return 1; }
```
(Client side analogous; keep it to host-side if the data-forward asserts below still consume their own sockets cleanly — order the PEER check before the HELLO payload sends so the buffers don't interleave.)

- [ ] **Step 4: Build + run the relay self-test**

Run: `cd /d/Pulsar/PULSAR/lobby/relay && dotnet run -c Release -- --selftest`
Expected: `[selftest] PASS` (PEER delivered + both-way forward still works).

- [ ] **Step 5: Commit**
```bash
git add lobby/relay/Program.cs
git commit -m "feat(relay): send PEER endpoint to both peers once paired (rendezvous)"
```

---

## Task 3: NetworkHost — intercept, rendezvous, punch, direct-connect, fallback

**Files:** Modify `include/Network/NetworkHost.h`, `src/Network/NetworkHost.cpp`

- [ ] **Step 1: Add rendezvous members to the header**

In `include/Network/NetworkHost.h`, after `uint64_t m_relayKeepaliveAt = 0;` add:
```cpp
        // P2P hole-punch rendezvous (direct-first, relay-fallback).
        enum class Rz { kNone, kAwaitPeer, kDirectConnecting, kDoneDirect, kDoneRelay };
        Rz          m_rz = Rz::kNone;
        uint64_t    m_rzDeadline = 0;
        bool        m_forceRelay = false;   // PULSAR_FORCE_RELAY
        ENetAddress m_directAddress{};      // peer's punched public endpoint
        void        PunchTo(const ENetAddress& a_ep, int a_count);  // raw datagrams via m_host->socket
```

- [ ] **Step 2: Add the intercept callback + peer-endpoint handoff (cpp, file-local)**

Near the top of `NetworkHost.cpp` (after includes, in an anonymous namespace or as static), add:
```cpp
    // Set by the ENet intercept when a PEER arrives; read by Update().
    static std::atomic<bool>     g_rzPeerKnown{ false };
    static std::atomic<uint32_t> g_rzPeerIp{ 0 };    // network order
    static std::atomic<uint16_t> g_rzPeerPort{ 0 };  // host order

    // ENet calls this for every raw datagram before protocol processing.
    // Consume our control messages (return 1); pass everything else (return 0).
    static int ENET_CALLBACK RelayIntercept(ENetHost* a_host, ENetEvent* /*a_event*/)
    {
        const auto* d = a_host->receivedData;
        const size_t n = a_host->receivedDataLength;
        if (!Pulsar::Network::Relay::IsControlMagic(d, n)) return 0;
        uint32_t ip; uint16_t port;
        if (Pulsar::Network::Relay::ParsePeer(d, n, ip, port)) {
            g_rzPeerIp.store(ip, std::memory_order_relaxed);
            g_rzPeerPort.store(port, std::memory_order_relaxed);
            g_rzPeerKnown.store(true, std::memory_order_release);
        }
        return 1; // consume (stray REGISTER/JOIN echoes too)
    }
```

- [ ] **Step 3: Implement `PunchTo` + read `PULSAR_FORCE_RELAY`**

Add to `NetworkHost.cpp`:
```cpp
    void NetworkHost::PunchTo(const ENetAddress& a_ep, int a_count)
    {
        if (!m_host) return;
        const uint8_t one = 0x50; // 'P' — content irrelevant; sending opens our NAT mapping
        ENetBuffer eb{ 1, const_cast<uint8_t*>(&one) };
        for (int i = 0; i < a_count; ++i) enet_socket_send(m_host->socket, &a_ep, &eb, 1);
    }
```
In `SetRelay` (after `m_relayMode = true;`), read the kill-switch:
```cpp
        char* fr = nullptr; size_t frn = 0;
        if (_dupenv_s(&fr, &frn, "PULSAR_FORCE_RELAY") == 0 && fr) { m_forceRelay = (fr[0] == '1'); free(fr); }
```

- [ ] **Step 4: Set the intercept after every `enet_host_create`**

After `m_host = enet_host_create(...)` in BOTH the host path (~line 257) and the client path (`DoConnectInternal` ~line 312), add:
```cpp
        if (m_host && m_relayMode && !m_forceRelay) m_host->intercept = &RelayIntercept;
```

- [ ] **Step 5: Client — defer the direct connect until PEER (rewrite the relay branch of `DoConnectInternal`)**

Replace the relay branch in `DoConnectInternal` (lines 319-322, the `if (m_relayMode) { address = m_relayAddress; ... JOIN ... }`) so that, when P2P is enabled, it sends JOIN and enters rendezvous instead of connecting to the relay immediately:
```cpp
        ENetAddress address{};
        if (m_relayMode && !m_forceRelay) {
            // P2P: announce to the relay, then wait for PEER (Update drives the
            // punch + direct connect). Do NOT connect to the relay yet.
            g_rzPeerKnown.store(false, std::memory_order_release);
            for (int i = 0; i < 3; ++i) SendRelayControl(Relay::kTypeJoin);
            m_rz = Rz::kAwaitPeer;
            m_rzDeadline = GetTickCount64() + 2000;   // kPeerWaitMs
            m_isHost = false;
            m_lastAddress = a_address; m_lastPort = a_port;
            REX::INFO("[PULSAR] [Relay] client: rendezvous, awaiting peer endpoint"sv);
            return true;
        } else if (m_relayMode) {  // FORCE_RELAY — today's path verbatim
            address = m_relayAddress;
            for (int i = 0; i < 3; ++i) SendRelayControl(Relay::kTypeJoin);
            REX::INFO("[PULSAR] [Relay] client joined session, connecting via relay (forced)"sv);
        } else {
            enet_address_set_host(&address, a_address);
            address.port = a_port;
        }

        m_peer = enet_host_connect(m_host, &address, NUM_CHANNELS, 0);
        if (!m_peer) { REX::WARN("Failed to initiate connection"sv); return false; }
        m_isHost = false; m_lastAddress = a_address; m_lastPort = a_port;
        REX::INFO("[PULSAR] Connecting..."sv);
        return true;
```

- [ ] **Step 6: Client — drive punch / direct-connect / fallback in `Update()`**

In `NetworkHost::Update()`, near the top (after the host keepalive block), add the client rendezvous driver:
```cpp
        // P2P client rendezvous state machine.
        if (m_relayMode && !m_forceRelay && !m_isHost && m_rz != Rz::kNone &&
            m_rz != Rz::kDoneDirect && m_rz != Rz::kDoneRelay)
        {
            const uint64_t now = GetTickCount64();
            std::lock_guard lk(m_enetMutex);
            if (m_rz == Rz::kAwaitPeer) {
                if (g_rzPeerKnown.load(std::memory_order_acquire)) {
                    m_directAddress.host = g_rzPeerIp.load(std::memory_order_relaxed);
                    m_directAddress.port = g_rzPeerPort.load(std::memory_order_relaxed);
                    PunchTo(m_directAddress, 5);                          // kPunchCount
                    m_peer = enet_host_connect(m_host, &m_directAddress, NUM_CHANNELS, 0);
                    m_rz = Rz::kDirectConnecting;
                    m_rzDeadline = now + 3000;                            // kDirectMs
                    REX::INFO("[PULSAR] [Relay] punch + direct connect to peer"sv);
                } else if (now >= m_rzDeadline) {
                    m_peer = enet_host_connect(m_host, &m_relayAddress, NUM_CHANNELS, 0);
                    m_rz = Rz::kDoneRelay;
                    REX::INFO("[PULSAR] [Relay] no peer endpoint — relay fallback"sv);
                }
            } else if (m_rz == Rz::kDirectConnecting) {
                if (m_connectionEstablished.load(std::memory_order_acquire)) {
                    m_rz = Rz::kDoneDirect;
                    REX::INFO("[PULSAR] [Relay] DIRECT P2P established"sv);
                } else if (now >= m_rzDeadline) {
                    if (m_peer) { enet_peer_reset(m_peer); m_peer = nullptr; }
                    m_peer = enet_host_connect(m_host, &m_relayAddress, NUM_CHANNELS, 0);
                    m_rz = Rz::kDoneRelay;
                    REX::INFO("[PULSAR] [Relay] direct timed out — relay fallback"sv);
                }
            }
        }
```
(`m_connectionEstablished` is the existing flag set in `OnConnect`. Re-punch is unnecessary; the first burst opens the mapping for the connect handshake.)

- [ ] **Step 7: Host — punch toward the client; stop keepalive on a direct connection**

In `Update()`'s host keepalive block (the `if (m_relayMode && m_isHost && m_host)`), add a one-shot punch when the peer endpoint is learned:
```cpp
        if (m_relayMode && !m_forceRelay && m_isHost && m_host &&
            g_rzPeerKnown.load(std::memory_order_acquire) && m_rz == Rz::kNone)
        {
            ENetAddress cep{}; cep.host = g_rzPeerIp.load(); cep.port = g_rzPeerPort.load();
            { std::lock_guard lk(m_enetMutex); PunchTo(cep, 5); }
            m_rz = Rz::kAwaitPeer; // mark punched (reuse enum; host has no further states here)
            REX::INFO("[PULSAR] [Relay] host punched toward client"sv);
        }
```
In `OnConnect` (NetworkHost.cpp ~line 740), after `m_peer = a_peer;`, distinguish direct vs relayed and stop the host keepalive when direct:
```cpp
        if (m_relayMode && m_isHost) {
            const bool viaRelay = (a_peer->address.host == m_relayAddress.host &&
                                   a_peer->address.port == m_relayAddress.port);
            if (!viaRelay) {
                m_relayKeepaliveAt = UINT64_MAX;  // stop REGISTER → relay session idles out
                REX::INFO("[PULSAR] [Relay] host: DIRECT peer — keepalive stopped"sv);
            } else {
                REX::INFO("[PULSAR] [Relay] host: peer via relay (fallback)"sv);
            }
        }
```

- [ ] **Step 8: Build the DLL**

Run: `"/c/Program Files/CMake/bin/cmake" --build build/vs --config RelWithDebInfo --target PULSAR`
Expected: `PULSAR.vcxproj -> ...PULSAR.dll`, no errors.

- [ ] **Step 9: Commit**
```bash
git add include/Network/NetworkHost.h src/Network/NetworkHost.cpp
git commit -m "feat(net): P2P hole-punch — intercept + rendezvous/punch/direct, relay fallback, FORCE_RELAY"
```

---

## Task 4: Local verification (no NAT)

**Files:** none

- [ ] **Step 1: Relay self-test** — `cd lobby/relay && dotnet run -c Release -- --selftest` → `PASS` (PEER + forward).
- [ ] **Step 2: Relay live probe** — `dotnet run -c Release -- --probe 127.0.0.1:7777` against a locally-run relay → `PASS` (forward path intact).
- [ ] **Step 3: FORCE_RELAY regression (manual reasoning + build):** confirm that with `PULSAR_FORCE_RELAY=1` the code path is byte-identical to today (intercept not set, `DoConnectInternal` takes the `else if (m_relayMode)` branch = old behavior). This is the public-alpha safety valve.
- [ ] **Step 4: Loopback-direct path (in-game, single machine is fine):** host + client on the same PC via the relay. Since both are loopback, the relay reports loopback endpoints and the direct connect targets loopback → exercises intercept + punch + direct-connect + the `DIRECT P2P established` log without a real NAT. Confirm the client log shows `DIRECT P2P established` (not `relay fallback`) and co-op works.

---

## Task 5: Deploy + real-NAT validation

**Files:** none

- [ ] **Step 1: Deploy DLL** (local + NAS, forward-slash UNC):
```bash
cp -f build/vs/RelWithDebInfo/PULSAR.dll /d/Games/Starfield/Data/SFSE/Plugins/PULSAR.dll
cp -f build/vs/RelWithDebInfo/PULSAR.dll <NAS>/deploy/Data/SFSE/Plugins/PULSAR.dll
```
- [ ] **Step 2: Sync + push relay to the private deploy repo** (VPS pulls it):
```bash
cp -f lobby/relay/Program.cs /d/Pulsar/pulsar-lobby-server/lobby/relay/Program.cs
cp -f include/Network/RelayProtocol.h /d/Pulsar/pulsar-lobby-server/  # only if the repo carries it; else skip
cd /d/Pulsar/pulsar-lobby-server && git add -A && git commit -m "feat(relay): PEER rendezvous for P2P hole-punch" && git push origin HEAD
```
- [ ] **Step 3: USER VPS** — `git pull` + `docker compose up -d --build` (rebuild relay container with the PEER change). No STDB schema change.
- [ ] **Step 4: USER in-game (two different networks):** host + client. Confirm client log: `punch + direct connect` → `DIRECT P2P established`. Play; then **stop the relay container mid-session** → co-op survives (proves no game traffic on the VPS). Then test fallback: `PULSAR_FORCE_RELAY=1` on the client → connects via relay, plays.

---

## Self-Review

**Spec coverage:** PEER protocol (Task 1) ✓; relay sends PEER when paired + selftest (Task 2) ✓; intercept callback (T3.2) ✓; punch (T3.3) ✓; client rendezvous/direct/fallback (T3.5–6) ✓; host punch + keepalive-stop + direct/relay distinction (T3.7) ✓; FORCE_RELAY kill-switch (T3.3–5) ✓; local tests incl. force-relay regression + loopback-direct (Task 4) ✓; deploy + real-NAT + relay-kill survival + fallback test (Task 5) ✓. Out-of-scope items untouched.

**Placeholder scan:** No TBD/TODO. Constants are concrete (kTypePeer=3, kPeerLen=31, 2000/3000 ms, punch count 5). The one NOTE (relay session-bytes source `b` vs `Convert.FromHexString(key)`) gives the exact resolution, not a placeholder.

**Type consistency:** `kTypePeer`/`MakePeer`/`ParsePeer`/`IsControlMagic` match between header (T1) and C++ intercept (T3.2). `g_rzPeerIp/Port/Known` written in intercept (T3.2), read in Update (T3.6–7). `Rz` enum + `m_rz`/`m_rzDeadline`/`m_directAddress`/`m_forceRelay`/`PunchTo` declared (T3.1) and used (T3.3–7). `ENetAddress.host`=net-order, `.port`=host-order consistent (MakePeer/ParsePeer/intercept/PunchTo). Relay `TypePeer=3`/`PeerMsg` match the C++ wire format (4-byte net-order IP + 2-byte BE port).

## Risks
- **Symmetric NAT (~20–30%)** → direct fails → relay fallback (expected, logged).
- **Real punch only testable cross-NAT** — local proves protocol/fallback/loopback-direct; T5.4 is the true validation.
- **Intercept runs on the ENet thread** — callback only does parse + atomic store (no locks/alloc).
- **`m_host->socket` send during rendezvous** — already used for REGISTER/JOIN, same pattern.
