# Proximity Voice Chat Implementation Plan


**Goal:** In-game proximity voice between the two co-op players over the existing encrypted P2P/relay link, with a Push-to-Talk mode (bindable key), an Open-Mic mode, and a per-user "always audible" override, all configured from a new VoiceChat tab in the tool.

**Architecture:** The Companion (C#) owns the whole audio stack (NAudio capture/render, Concentus Opus codec, jitter buffer, PTT gate) and exchanges opaque Opus frames with the DLL over a localhost UDP loopback. The DLL (C++) wraps each mic frame as `PKT_VOICE` and ships it over `NetworkHost::EncSend` on a new unreliable ENet channel; for inbound frames it computes a proximity gain from both players' in-process positions/cells and prepends it before handing the frame back to the Companion. The tool (C#, Avalonia) writes `pulsar_voice.json`; the Companion is the single reader and forwards the three gain-relevant values to the DLL over the loopback.

**Tech Stack:** C++23 (SFSE plugin, Winsock loopback, ENet), C# net8 (Companion daemon + Avalonia tool), NuGet `NAudio` 2.2.1, NuGet `Concentus` 1.1.7 (classic array API), xUnit for pure-logic tests.

**Status:** Post-Beta. This plan is written now; execution waits until the open co-op-breaking bugs (///) are closed. Tracks task.

---

## Loopback protocol (shared contract, referenced by many tasks)

One fixed UDP port on `127.0.0.1`. The DLL binds it; the Companion binds an ephemeral port, sends a HELLO once, and the DLL learns the Companion's address from `recvfrom`. All multi-byte fields little-endian, `#pragma pack(1)` / C# sequential.

Message tags (first byte):

| Tag  | Name        | Direction         | Payload after tag                                  |
|------|-------------|-------------------|----------------------------------------------------|
| 0x10 | HELLO       | Companion -> DLL  | (none) - registers the Companion endpoint          |
| 0x01 | AUDIO (mic) | Companion -> DLL  | `[2B seq][1B flags][opus...]`                       |
| 0x04 | SETTINGS    | Companion -> DLL  | `[4B dNear float][4B dFar float][1B alwaysAudible]` |
| 0x01 | AUDIO (peer)| DLL -> Companion  | `[4B gain float][2B seq][1B flags][opus...]`        |
| 0x02 | SESSION_UP  | DLL -> Companion  | (none) - ENet peer connected                        |
| 0x03 | SESSION_DOWN| DLL -> Companion  | (none) - ENet peer disconnected                     |

Note the AUDIO tag `0x01` is reused in both directions but the layouts differ: the peer->Companion audio has a 4-byte gain prefix, the mic->DLL audio does not. Direction is unambiguous because each side only ever receives one of the two.

Fixed constants (define once in `include/VoiceBridge/VoiceProtocol.h`, mirror in C# `VoiceFraming`):
- `kLoopbackPort = 27900`
- `kMaxOpusBytes = 400` (48 kHz mono 20 ms VOIP frame is well under this)
- ENet voice channel index = `2`

---

## File Structure

**DLL (C++)**
- `include/VoiceBridge/VoiceProtocol.h` (new) - loopback tags, port, constants. Pure header, shared by DLL code.
- `include/VoiceBridge/ProximityGain.h` (new) - pure `ComputeProximityGain(...)` free function. Header-only so a one-off test can include it.
- `include/VoiceBridge/VoiceBridge.h` (new) - `VoiceBridge` static class: `Init/Shutdown/Service/OnVoicePacket/OnPeerUp/OnPeerDown`.
- `src/VoiceBridge/VoiceBridge.cpp` (new) - Winsock loopback socket, framing, calls `ComputeProximityGain`, calls `NetworkHost::SendVoiceFrame`.
- `include/Network/PacketTypes.h` (modify) - add `PKT_VOICE = 0xA4` + `VoiceFrameHeader`.
- `include/Network/NetworkHost.h` (modify) - `NUM_CHANNELS 2 -> 3`; declare `void SendVoiceFrame(const uint8_t*, size_t)`.
- `src/Network/NetworkHost.cpp` (modify) - implement `SendVoiceFrame`; dispatch `PKT_VOICE` in `OnReceive`; call `VoiceBridge::OnPeerUp/OnPeerDown` from the ENet connect/disconnect handlers.
- `src/Main.cpp` (modify) - `VoiceBridge::Init()` after `EventTrace::Init`; `VoiceBridge::Service()` inside the existing network loop.

**Companion (C#)** - namespace `Pulsar.Companion.Voice`, folder `lobby/companion/Voice/`
- `VoiceFraming.cs` (new) - pure static pack/parse of loopback + wire frames. Testable.
- `JitterBuffer.cs` (new) - reorder-by-seq / drop-on-stall buffer. Testable.
- `VoiceConfig.cs` (new) - `pulsar_voice.json` record + load + file-change watch. Testable (parse).
- `OpusCodec.cs` (new) - thin Concentus encode/decode wrapper.
- `VoiceBridgeClient.cs` (new) - localhost UDP to the DLL (HELLO, send mic, recv peer/session, send settings).
- `PttHook.cs` (new) - `WH_KEYBOARD_LL` global keyboard hook, exposes `IsKeyDown(vk)`.
- `VoiceEngine.cs` (new) - orchestrates capture->gate->encode->bridge and bridge->jitter->decode->gain->render; lifecycle on SESSION_UP/DOWN.
- `Program.cs` (modify) - construct/start `VoiceEngine` in the `serve` loop; dispose on exit.
- `PulsarCompanion.csproj` (modify) - add NAudio + Concentus.

**Companion tests (C#)**
- `lobby/companion.tests/Companion.Tests.csproj` (new) - xUnit, references the companion project.
- `lobby/companion.tests/VoiceFramingTests.cs`, `JitterBufferTests.cs`, `VoiceConfigTests.cs` (new).

**Tool (C#, Avalonia)** - `lobby/pulsar/`
- `ViewModels/VoiceChatViewModel.cs` (new) - device enumeration (NAudio), bound properties, key capture, writes `pulsar_voice.json`.
- `Views/VoiceChatView.axaml` (+ `.axaml.cs`) (new) - the tab UI.
- `Views/MainWindow.axaml` (modify) - nav entry.
- `ViewModels/MainWindowViewModel.cs` (modify) - nav wiring.
- `Pulsar.csproj` (modify) - add NAudio (device list only).

**DLL one-off test**
- `tests/voice/test_proximity_gain.cpp` (new) - standalone asserts for the gain curve (no DLL test harness exists; compiled ad hoc).

**Config contract (`pulsar_voice.json`, `%LOCALAPPDATA%\PULSAR\`)**
```json
{
  "enabled": true,
  "mode": "ptt",
  "pttKey": 84,
  "alwaysAudible": false,
  "alwaysAudibleKey": 0,
  "inputDevice": "",
  "outputDevice": "",
  "inputVolume": 1.0,
  "outputVolume": 1.0,
  "dNear": 600.0,
  "dFar": 4000.0
}
```
`pttKey`/`alwaysAudibleKey` are Win32 virtual-key codes (0 = unbound). `inputDevice`/`outputDevice` are NAudio `MMDevice.ID` strings ("" = system default).

---

# Phase 0 - Transport spike (DLL only, no audio)

Proves Companion -> DLL -> peer -> DLL -> Companion end to end with dummy bytes.

### Task 1: Add PKT_VOICE and the wire header

**Files:**
- Modify: `include/Network/PacketTypes.h` (enum block ends at the line with `PKT_NPC_SPAWN = 0xA3,` inside `enum PacketType`)

- [ ] **Step 1: Add the packet id**

In `include/Network/PacketTypes.h`, directly after the `PKT_NPC_SPAWN = 0xA3,` line and before the closing `};` of `enum PacketType`, add:

```cpp
        PKT_VOICE            = 0xA4,  // Either side -> peer: one Opus voice frame on unreliable ENet channel 2. Wire: [1B type][2B seq][1B flags][opus]. Framed/handled by VoiceBridge, never reaches the PacketReader switch. Proximity Voice .
```

- [ ] **Step 2: Add the frame header struct**

After the `static_assert(sizeof(NpcSpawnData) ...` block near the end of the namespace, add:

```cpp
#pragma pack(push, 1)
    // PKT_VOICE payload header — precedes the raw Opus blob on the ENet wire.
    struct VoiceFrameHeader
    {
        uint16_t seq;    // sender monotonic sequence, for jitter-buffer reorder/drop
        uint8_t  flags;  // bit 0 = talking (VAD/PTT open); reserved otherwise
    };
#pragma pack(pop)
    static_assert(sizeof(VoiceFrameHeader) == 3, "VoiceFrameHeader size");
```

- [ ] **Step 3: Build the DLL (dev config) to confirm it compiles**

Run: `cmake --build build/vs --config RelWithDebInfo --target PULSAR`
Expected: builds with no new errors.

- [ ] **Step 4: Commit**

```bash
git add include/Network/PacketTypes.h
git commit -m "feat(voice): add PKT_VOICE packet id + VoiceFrameHeader"
```

### Task 2: Bump ENet channel count and add the send API

**Files:**
- Modify: `include/Network/NetworkHost.h:16` (`NUM_CHANNELS`) and the public method list
- Modify: `src/Network/NetworkHost.cpp` (near the existing `EncSend` at line 689)

- [ ] **Step 1: Bump the channel count**

In `include/Network/NetworkHost.h`, change line 16 from:

```cpp
    constexpr uint32_t NUM_CHANNELS = 2;
```
to:
```cpp
    constexpr uint32_t NUM_CHANNELS = 3;  // 0 = control/state, 1 = bulk (saves), 2 = voice (unreliable). Voice bump protocol change, both peers must match.
```

- [ ] **Step 2: Declare the public voice send wrapper**

In `include/Network/NetworkHost.h`, in the `public:` section next to `Send(...)`, add:

```cpp
        // Send one already-framed voice payload ([VoiceFrameHeader][opus]) to the
        // peer on channel 2, unreliable. Prepends PKT_VOICE and encrypts via EncSend.
        // No-op if not connected. Called from VoiceBridge on the network thread.
        void SendVoiceFrame(const uint8_t* a_payload, size_t a_len);
```

- [ ] **Step 3: Implement it**

In `src/Network/NetworkHost.cpp`, immediately after the `EncSend` function (ends around line 720), add:

```cpp
    void NetworkHost::SendVoiceFrame(const uint8_t* a_payload, size_t a_len)
    {
        if (!m_peer || a_len == 0 || a_len > 1024) return;  // no peer / sanity bound
        // [1B PKT_VOICE][payload]
        std::vector<uint8_t> buf(1 + a_len);
        buf[0] = static_cast<uint8_t>(PKT_VOICE);
        std::memcpy(buf.data() + 1, a_payload, a_len);
        EncSend(2, buf.data(), buf.size(), /*reliable=*/false);
    }
```

If the member peer handle is not named `m_peer`, grep `NetworkHost.h` for the `ENetPeer*` member and use that name. If a connected-state helper already exists (grep `IsConnected`), prefer `if (!IsConnected()) return;`.

- [ ] **Step 4: Build**

Run: `cmake --build build/vs --config RelWithDebInfo --target PULSAR`
Expected: builds clean.

- [ ] **Step 5: Commit**

```bash
git add include/Network/NetworkHost.h src/Network/NetworkHost.cpp
git commit -m "feat(voice): NUM_CHANNELS 3 + NetworkHost::SendVoiceFrame on channel 2"
```

### Task 3: VoiceProtocol constants + VoiceBridge skeleton (loopback socket)

**Files:**
- Create: `include/VoiceBridge/VoiceProtocol.h`
- Create: `include/VoiceBridge/VoiceBridge.h`
- Create: `src/VoiceBridge/VoiceBridge.cpp`

- [ ] **Step 1: Write the protocol constants header**

`include/VoiceBridge/VoiceProtocol.h`:

```cpp
#pragma once
#include <cstdint>

namespace Pulsar::Voice
{
    constexpr uint16_t kLoopbackPort = 27900;   // DLL binds 127.0.0.1:27900
    constexpr int      kVoiceChannel = 2;       // ENet channel for PKT_VOICE
    constexpr size_t   kMaxOpusBytes = 400;     // 48kHz mono 20ms VOIP frame ceiling

    enum LoopbackTag : uint8_t
    {
        kTagAudio       = 0x01,  // Companion<->DLL audio frame (layout differs by direction)
        kTagSessionUp   = 0x02,  // DLL->Companion: ENet peer connected
        kTagSessionDown = 0x03,  // DLL->Companion: ENet peer disconnected
        kTagSettings    = 0x04,  // Companion->DLL: {float dNear; float dFar; uint8 alwaysAudible}
        kTagHello       = 0x10,  // Companion->DLL: register endpoint
    };
}
```

- [ ] **Step 2: Write the VoiceBridge interface**

`include/VoiceBridge/VoiceBridge.h`:

```cpp
#pragma once
#include <cstdint>
#include <cstddef>

namespace Pulsar::Voice
{
    // Owns the localhost UDP loopback to the Companion. Serviced on the network
    // thread. All entry points are noexcept and self-guarding: any failure logs
    // and disables voice without touching the rest of the DLL.
    class VoiceBridge
    {
    public:
        static void Init();       // bind loopback socket (idempotent)
        static void Shutdown();   // close socket
        static void Service();    // drain inbound loopback (mic frames, settings, hello)

        // Called from NetworkHost::OnReceive when a PKT_VOICE arrives (already
        // decrypted). a_data/a_len is [VoiceFrameHeader][opus] (no type byte).
        static void OnVoicePacket(const uint8_t* a_data, size_t a_len) noexcept;

        // Called from NetworkHost ENet connect/disconnect handlers.
        static void OnPeerUp() noexcept;
        static void OnPeerDown() noexcept;
    };
}
```

- [ ] **Step 3: Write the socket skeleton (bind + Service drain + hello capture)**

`src/VoiceBridge/VoiceBridge.cpp`:

```cpp
#include "VoiceBridge/VoiceBridge.h"
#include "VoiceBridge/VoiceProtocol.h"
#include "Log.h"

#include <winsock2.h>
#include <ws2tcpip.h>
#include <atomic>
#include <cstring>

#pragma comment(lib, "ws2_32.lib")

namespace Pulsar::Voice
{
    namespace
    {
        SOCKET             g_sock = INVALID_SOCKET;
        sockaddr_in        g_companionAddr{};       // learned from HELLO
        std::atomic<bool>  g_haveCompanion{ false };

        // gain-calc settings pushed by the Companion (kTagSettings)
        std::atomic<float> g_dNear{ 600.0f };
        std::atomic<float> g_dFar{ 4000.0f };
        std::atomic<bool>  g_alwaysAudible{ false };

        void SendToCompanion(const uint8_t* data, int len)
        {
            if (g_sock == INVALID_SOCKET || !g_haveCompanion.load()) return;
            ::sendto(g_sock, reinterpret_cast<const char*>(data), len, 0,
                     reinterpret_cast<sockaddr*>(&g_companionAddr), sizeof(g_companionAddr));
        }
    }

    void VoiceBridge::Init()
    {
        if (g_sock != INVALID_SOCKET) return;
        WSADATA wsa;
        if (::WSAStartup(MAKEWORD(2, 2), &wsa) != 0) { PLOG_WARN("[VOICE] WSAStartup failed"); return; }
        g_sock = ::socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
        if (g_sock == INVALID_SOCKET) { PLOG_WARN("[VOICE] socket() failed"); return; }
        u_long nb = 1; ::ioctlsocket(g_sock, FIONBIO, &nb);  // non-blocking
        sockaddr_in addr{}; addr.sin_family = AF_INET;
        addr.sin_port = htons(kLoopbackPort);
        ::inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr);
        if (::bind(g_sock, reinterpret_cast<sockaddr*>(&addr), sizeof(addr)) != 0) {
            PLOG_WARN("[VOICE] bind 127.0.0.1:{} failed (voice disabled)", kLoopbackPort);
            ::closesocket(g_sock); g_sock = INVALID_SOCKET; return;
        }
        PLOG_INFO("[VOICE] loopback bound on 127.0.0.1:{}", kLoopbackPort);
    }

    void VoiceBridge::Shutdown()
    {
        if (g_sock != INVALID_SOCKET) { ::closesocket(g_sock); g_sock = INVALID_SOCKET; }
        g_haveCompanion.store(false);
    }

    void VoiceBridge::Service()
    {
        if (g_sock == INVALID_SOCKET) return;
        uint8_t buf[1500];
        sockaddr_in from{}; int fromLen = sizeof(from);
        for (;;)
        {
            int n = ::recvfrom(g_sock, reinterpret_cast<char*>(buf), sizeof(buf), 0,
                               reinterpret_cast<sockaddr*>(&from), &fromLen);
            if (n <= 0) break;  // WSAEWOULDBLOCK when drained
            const uint8_t tag = buf[0];
            if (tag == kTagHello) {
                g_companionAddr = from; g_haveCompanion.store(true);
                PLOG_INFO("[VOICE] companion registered");
            } else if (tag == kTagAudio && n > 1) {
                // mic frame [1B tag][VoiceFrameHeader][opus] -> strip tag, ship to peer
                extern void SendVoiceToPeer(const uint8_t*, size_t);  // Task 5
                SendVoiceToPeer(buf + 1, static_cast<size_t>(n - 1));
            } else if (tag == kTagSettings && n >= 1 + 9) {
                float dn, df; std::memcpy(&dn, buf + 1, 4); std::memcpy(&df, buf + 5, 4);
                g_dNear.store(dn); g_dFar.store(df); g_alwaysAudible.store(buf[9] != 0);
            }
        }
    }

    void VoiceBridge::OnPeerUp() noexcept
    {
        const uint8_t m = kTagSessionUp; SendToCompanion(&m, 1);
    }
    void VoiceBridge::OnPeerDown() noexcept
    {
        const uint8_t m = kTagSessionDown; SendToCompanion(&m, 1);
    }

    // OnVoicePacket + the settings getters are completed in Task 4 / Task 18.
    void VoiceBridge::OnVoicePacket(const uint8_t* a_data, size_t a_len) noexcept
    {
        // Phase 0: no gain yet — prepend gain=1.0 and forward to Companion.
        if (a_len == 0 || a_len > 1200) return;
        uint8_t out[1500];
        out[0] = kTagAudio;
        float gain = 1.0f; std::memcpy(out + 1, &gain, 4);
        std::memcpy(out + 5, a_data, a_len);
        SendToCompanion(out, static_cast<int>(5 + a_len));
    }

    // exposed to VoiceProtocol getters for Task 18
    float  VoiceBridge_dNear()         { return g_dNear.load(); }
    float  VoiceBridge_dFar()          { return g_dFar.load(); }
    bool   VoiceBridge_alwaysAudible() { return g_alwaysAudible.load(); }
}
```

- [ ] **Step 4: Register the new source dir with CMake (glob already recurses)**

The `CMakeLists.txt` `GLOB_RECURSE` on `src/*.cpp` and `include/*.h` picks up the new files. Re-run configure so the glob refreshes:

Run: `cmake --build build/vs --config RelWithDebInfo --target PULSAR`
Expected: builds; if the new files are not compiled, run `cmake --preset vs` (or your configure preset) once, then rebuild.

- [ ] **Step 5: Commit**

```bash
git add include/VoiceBridge/ src/VoiceBridge/
git commit -m "feat(voice): VoiceBridge loopback socket skeleton + protocol constants"
```

### Task 4: Route PKT_VOICE + peer up/down through NetworkHost

**Files:**
- Modify: `src/Network/NetworkHost.cpp` (the `OnReceive` custom-handler block; the ENet connect + disconnect handlers)

- [ ] **Step 1: Include the bridge header**

At the top of `src/Network/NetworkHost.cpp`, with the other includes, add:

```cpp
#include "VoiceBridge/VoiceBridge.h"
```

- [ ] **Step 2: Dispatch PKT_VOICE before the PacketReader switch**

In `OnReceive`, find the custom-handler block that runs after decryption and before the `PacketReader` switch (grep for other early-return custom handlers). Add:

```cpp
        if (type == PKT_VOICE) {
            // payload after the 1-byte type = [VoiceFrameHeader][opus]
            Pulsar::Voice::VoiceBridge::OnVoicePacket(data + 1, len - 1);
            return;
        }
```

Use whatever local variables that block already exposes for the decrypted buffer + length (match the neighbouring handlers exactly; they already name the type byte and payload).

- [ ] **Step 3: Fire session up/down from the ENet handlers**

In the ENet `ENET_EVENT_TYPE_CONNECT` handler (host accept + client connect success paths), after the connection is fully established, add:

```cpp
        Pulsar::Voice::VoiceBridge::OnPeerUp();
```

In `OnDisconnect` (around line 1742) add near the top:

```cpp
        Pulsar::Voice::VoiceBridge::OnPeerDown();
```

- [ ] **Step 4: Build**

Run: `cmake --build build/vs --config RelWithDebInfo --target PULSAR`
Expected: builds clean.

- [ ] **Step 5: Commit**

```bash
git add src/Network/NetworkHost.cpp
git commit -m "feat(voice): route PKT_VOICE to VoiceBridge + peer up/down signals"
```

### Task 5: Outbound send + wire Init/Service into Main

**Files:**
- Modify: `src/VoiceBridge/VoiceBridge.cpp` (implement the `SendVoiceToPeer` referenced in Task 3)
- Modify: `src/Main.cpp` (Init after EventTrace; Service in the network loop)

- [ ] **Step 1: Implement SendVoiceToPeer via NetworkHost**

In `src/VoiceBridge/VoiceBridge.cpp`, add near the top includes:

```cpp
#include "Network/NetworkHost.h"
```

and, inside `namespace Pulsar::Voice`, add the free function the Service loop calls:

```cpp
    void SendVoiceToPeer(const uint8_t* payload, size_t len)
    {
        // payload = [VoiceFrameHeader][opus]; hand to NetworkHost which prepends PKT_VOICE
        Pulsar::Network::NetworkHost::GetSingleton()->SendVoiceFrame(payload, len);
    }
```

Confirm the singleton accessor name against `NetworkHost.h:182` (grep `GetSingleton`); adjust if it is a reference-returning `GetSingleton()`.

- [ ] **Step 2: Init the bridge at plugin load**

In `src/Main.cpp`, find the `EventTrace::Init` call in `SFSE_PLUGIN_LOAD`. Immediately after it add:

```cpp
    Pulsar::Voice::VoiceBridge::Init();
```

and add `#include "VoiceBridge/VoiceBridge.h"` to the includes.

- [ ] **Step 3: Service the bridge on the network thread**

In `src/Main.cpp`, find the network loop (grep `NetworkLoop` / the thread body that calls `network.Update()` each tick). After the `Update()` call in that loop add:

```cpp
        Pulsar::Voice::VoiceBridge::Service();
```

- [ ] **Step 4: Build the dev DLL**

Run: `cmake --build build/vs --config RelWithDebInfo --target PULSAR`
Expected: builds clean.

- [ ] **Step 5: Commit**

```bash
git add src/VoiceBridge/VoiceBridge.cpp src/Main.cpp
git commit -m "feat(voice): outbound SendVoiceToPeer + Init/Service wired into Main"
```

### Task 6: Dummy round-trip verification (no audio yet)

**Files:**
- Create (temporary): `tests/voice/loopback_ping.py`

- [ ] **Step 1: Write a tiny loopback pinger**

`tests/voice/loopback_ping.py` — sends HELLO, then one fake mic frame, and prints whatever the DLL loopback returns:

```python
import socket, struct, time
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.bind(("127.0.0.1", 0))
s.settimeout(3.0)
DLL = ("127.0.0.1", 27900)
s.sendto(bytes([0x10]), DLL)                       # HELLO
# fake mic frame: tag 0x01 + VoiceFrameHeader{seq=1,flags=1} + 8 dummy opus bytes
frame = bytes([0x01]) + struct.pack("<HB", 1, 1) + b"\xAA" * 8
s.sendto(frame, DLL)
print("sent; waiting for peer echo (needs a connected 2nd instance)...")
try:
    while True:
        data, _ = s.recvfrom(1500)
        print("recv tag", hex(data[0]), "len", len(data))
except socket.timeout:
    print("no echo (expected if no peer connected)")
```

- [ ] **Step 2: Manual two-instance test**

Deploy the dev DLL to the game (`bash deploy.sh` builds+copies) on both test machines, start a co-op session so the ENet peer connects. On the host machine run:

Run: `python tests/voice/loopback_ping.py`
Expected: `PULSAR.log` on both machines shows `[VOICE] loopback bound on 127.0.0.1:27900`, `[VOICE] companion registered`, and after peer connect the Companion receives a `tag 0x02` SESSION_UP. The fake mic frame travels host -> peer; the peer machine's loopback (if a pinger runs there too) receives `tag 0x01 len 13`.

Because the real Companion is not sending yet, the practical Phase-0 check is: both logs show the socket bound + SESSION_UP fired on connect and SESSION_DOWN on disconnect, and the Python pinger's frame reaches the peer's `OnVoicePacket` (add a temporary `PLOG_INFO("[VOICE] rx peer frame len {}", a_len)` at the top of `OnVoicePacket`, verify, then remove).

- [ ] **Step 3: Remove the temp log line and commit the test script**

```bash
git add tests/voice/loopback_ping.py
git commit -m "test(voice): loopback ping script for Phase 0 transport spike"
```

---

# Phase 1 - Audio loop, local (Companion, pure logic + local self-loop)

### Task 7: Add NuGet deps + create the test project

**Files:**
- Modify: `lobby/companion/PulsarCompanion.csproj`
- Create: `lobby/companion.tests/Companion.Tests.csproj`

- [ ] **Step 1: Add NAudio + Concentus to the Companion**

In `lobby/companion/PulsarCompanion.csproj`, inside an `<ItemGroup>`:

```xml
    <PackageReference Include="NAudio" Version="2.2.1" />
    <PackageReference Include="Concentus" Version="1.1.7" />
```

- [ ] **Step 2: Create the xUnit test project**

`lobby/companion.tests/Companion.Tests.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <IsPackable>false</IsPackable>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.11.1" />
    <PackageReference Include="xunit" Version="2.9.2" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.8.2" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\companion\PulsarCompanion.csproj" />
  </ItemGroup>
</Project>
```

- [ ] **Step 3: Restore + build the test project (empty, just proves wiring)**

Run: `dotnet build lobby/companion.tests/Companion.Tests.csproj`
Expected: build succeeds (0 tests).

- [ ] **Step 4: Commit**

```bash
git add lobby/companion/PulsarCompanion.csproj lobby/companion.tests/
git commit -m "build(voice): NAudio+Concentus deps + Companion.Tests xUnit project"
```

### Task 8: VoiceFraming (pure pack/parse) - TDD

**Files:**
- Create: `lobby/companion/Voice/VoiceFraming.cs`
- Create: `lobby/companion.tests/VoiceFramingTests.cs`

- [ ] **Step 1: Write the failing test**

`lobby/companion.tests/VoiceFramingTests.cs`:

```csharp
using Pulsar.Companion.Voice;
using Xunit;

public class VoiceFramingTests
{
    [Fact]
    public void MicFrame_RoundTrips_SeqFlagsOpus()
    {
        var opus = new byte[] { 1, 2, 3, 4 };
        var wire = VoiceFraming.PackMicFrame(seq: 7, flags: 1, opus, opus.Length);
        Assert.Equal(VoiceFraming.TagAudio, wire[0]);

        bool ok = VoiceFraming.TryParsePeerFrame(
            // simulate DLL adding a gain prefix: [tag][gain][seq][flags][opus]
            BuildPeerWire(gain: 0.5f, seq: 7, flags: 1, opus),
            out float gain, out ushort seq, out byte flags, out byte[] payload);

        Assert.True(ok);
        Assert.Equal(0.5f, gain, 3);
        Assert.Equal((ushort)7, seq);
        Assert.Equal((byte)1, flags);
        Assert.Equal(opus, payload);
    }

    private static byte[] BuildPeerWire(float gain, ushort seq, byte flags, byte[] opus)
    {
        var b = new byte[1 + 4 + 3 + opus.Length];
        b[0] = VoiceFraming.TagAudio;
        System.BitConverter.GetBytes(gain).CopyTo(b, 1);
        System.BitConverter.GetBytes(seq).CopyTo(b, 5);
        b[7] = flags;
        opus.CopyTo(b, 8);
        return b;
    }
}
```

- [ ] **Step 2: Run it to confirm it fails**

Run: `dotnet test lobby/companion.tests/Companion.Tests.csproj --filter VoiceFramingTests`
Expected: FAIL (VoiceFraming does not exist).

- [ ] **Step 3: Implement VoiceFraming**

`lobby/companion/Voice/VoiceFraming.cs`:

```csharp
using System;

namespace Pulsar.Companion.Voice;

// Pure pack/parse for the localhost loopback frames. Mirrors the C++
// include/VoiceBridge/VoiceProtocol.h contract. No sockets, no state.
public static class VoiceFraming
{
    public const byte TagAudio       = 0x01;
    public const byte TagSessionUp   = 0x02;
    public const byte TagSessionDown = 0x03;
    public const byte TagSettings    = 0x04;
    public const byte TagHello       = 0x10;
    public const int  LoopbackPort   = 27900;

    // Companion -> DLL mic frame: [tag][seq:2][flags:1][opus...]
    public static byte[] PackMicFrame(ushort seq, byte flags, byte[] opus, int opusLen)
    {
        var b = new byte[1 + 2 + 1 + opusLen];
        b[0] = TagAudio;
        BitConverter.GetBytes(seq).CopyTo(b, 1);
        b[3] = flags;
        Array.Copy(opus, 0, b, 4, opusLen);
        return b;
    }

    // Companion -> DLL settings: [tag][dNear:4][dFar:4][alwaysAudible:1]
    public static byte[] PackSettings(float dNear, float dFar, bool alwaysAudible)
    {
        var b = new byte[1 + 4 + 4 + 1];
        b[0] = TagSettings;
        BitConverter.GetBytes(dNear).CopyTo(b, 1);
        BitConverter.GetBytes(dFar).CopyTo(b, 5);
        b[9] = (byte)(alwaysAudible ? 1 : 0);
        return b;
    }

    public static byte[] Hello() => new[] { TagHello };

    // DLL -> Companion peer audio: [tag][gain:4][seq:2][flags:1][opus...]
    public static bool TryParsePeerFrame(byte[] buf, out float gain, out ushort seq,
                                         out byte flags, out byte[] payload)
    {
        gain = 0; seq = 0; flags = 0; payload = Array.Empty<byte>();
        if (buf.Length < 8 || buf[0] != TagAudio) return false;
        gain  = BitConverter.ToSingle(buf, 1);
        seq   = BitConverter.ToUInt16(buf, 5);
        flags = buf[7];
        payload = new byte[buf.Length - 8];
        Array.Copy(buf, 8, payload, 0, payload.Length);
        return true;
    }
}
```

- [ ] **Step 4: Run the test to confirm it passes**

Run: `dotnet test lobby/companion.tests/Companion.Tests.csproj --filter VoiceFramingTests`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lobby/companion/Voice/VoiceFraming.cs lobby/companion.tests/VoiceFramingTests.cs
git commit -m "feat(voice): VoiceFraming loopback pack/parse (TDD)"
```

### Task 9: JitterBuffer - TDD

**Files:**
- Create: `lobby/companion/Voice/JitterBuffer.cs`
- Create: `lobby/companion.tests/JitterBufferTests.cs`

- [ ] **Step 1: Write the failing test**

`lobby/companion.tests/JitterBufferTests.cs`:

```csharp
using Pulsar.Companion.Voice;
using Xunit;

public class JitterBufferTests
{
    [Fact]
    public void ReordersOutOfOrderFrames()
    {
        var jb = new JitterBuffer(targetDepth: 2, capacity: 8);
        jb.Push(2, new byte[] { 2 });
        jb.Push(1, new byte[] { 1 });
        jb.Push(3, new byte[] { 3 });
        Assert.Equal((byte)1, jb.Pop()![0]);
        Assert.Equal((byte)2, jb.Pop()![0]);
        Assert.Equal((byte)3, jb.Pop()![0]);
    }

    [Fact]
    public void DropsStaleFramesBelowPlayhead()
    {
        var jb = new JitterBuffer(targetDepth: 1, capacity: 8);
        jb.Push(5, new byte[] { 5 });
        Assert.Equal((byte)5, jb.Pop()![0]);
        jb.Push(4, new byte[] { 4 });     // older than playhead -> dropped
        Assert.Null(jb.Pop());
    }

    [Fact]
    public void ReturnsNullWhenBelowTargetDepth()
    {
        var jb = new JitterBuffer(targetDepth: 3, capacity: 8);
        jb.Push(1, new byte[] { 1 });
        Assert.Null(jb.Pop());            // still filling
    }
}
```

- [ ] **Step 2: Run it to confirm it fails**

Run: `dotnet test lobby/companion.tests/Companion.Tests.csproj --filter JitterBufferTests`
Expected: FAIL (JitterBuffer missing).

- [ ] **Step 3: Implement JitterBuffer**

`lobby/companion/Voice/JitterBuffer.cs`:

```csharp
using System.Collections.Generic;

namespace Pulsar.Companion.Voice;

// Seq-ordered playout buffer. Pop() returns the next in-order frame once the
// buffer has filled to targetDepth; drops frames older than the playhead and
// caps memory at capacity (oldest evicted). Not thread-safe; the VoiceEngine
// pushes and pops from one playout thread.
public sealed class JitterBuffer
{
    private readonly SortedDictionary<ushort, byte[]> _frames = new();
    private readonly int _targetDepth;
    private readonly int _capacity;
    private bool _started;
    private ushort _playhead;

    public JitterBuffer(int targetDepth, int capacity)
    {
        _targetDepth = targetDepth;
        _capacity = capacity;
    }

    public void Push(ushort seq, byte[] frame)
    {
        if (_started && SeqOlderOrEqual(seq, _playhead)) return;   // stale, drop
        _frames[seq] = frame;
        while (_frames.Count > _capacity)
        {
            var oldest = default(ushort);
            foreach (var k in _frames.Keys) { oldest = k; break; }
            _frames.Remove(oldest);
        }
    }

    public byte[]? Pop()
    {
        if (_frames.Count < (_started ? 1 : _targetDepth)) return null;
        ushort next = default;
        foreach (var k in _frames.Keys) { next = k; break; }
        _frames.Remove(next);
        _started = true;
        _playhead = next;
        return _frames.Count >= 0 ? Frame(next) : null;
    }

    private byte[] Frame(ushort seq) => _lastPopped;
    private byte[] _lastPopped = System.Array.Empty<byte>();

    private static bool SeqOlderOrEqual(ushort a, ushort b)
        => (short)(a - b) <= 0;   // wrap-safe comparison
}
```

Note: the `Pop` above must actually return the removed frame. Replace the body of `Pop` with the correct version:

```csharp
    public byte[]? Pop()
    {
        if (_frames.Count < (_started ? 1 : _targetDepth)) return null;
        ushort next = default;
        byte[] frame = System.Array.Empty<byte>();
        foreach (var kv in _frames) { next = kv.Key; frame = kv.Value; break; }
        _frames.Remove(next);
        _started = true;
        _playhead = next;
        return frame;
    }
```

Delete the placeholder `Frame`/`_lastPopped` members. Keep `SeqOlderOrEqual`.

- [ ] **Step 4: Run the test to confirm it passes**

Run: `dotnet test lobby/companion.tests/Companion.Tests.csproj --filter JitterBufferTests`
Expected: PASS (all 3).

- [ ] **Step 5: Commit**

```bash
git add lobby/companion/Voice/JitterBuffer.cs lobby/companion.tests/JitterBufferTests.cs
git commit -m "feat(voice): seq-ordered jitter buffer (TDD)"
```

### Task 10: VoiceConfig (pulsar_voice.json) - TDD

**Files:**
- Create: `lobby/companion/Voice/VoiceConfig.cs`
- Create: `lobby/companion.tests/VoiceConfigTests.cs`

- [ ] **Step 1: Write the failing test**

`lobby/companion.tests/VoiceConfigTests.cs`:

```csharp
using System.IO;
using Pulsar.Companion.Voice;
using Xunit;

public class VoiceConfigTests
{
    [Fact]
    public void ParsesJson_WithDefaultsForMissingFields()
    {
        var path = Path.GetTempFileName();
        File.WriteAllText(path, "{ \"enabled\": true, \"mode\": \"ptt\", \"pttKey\": 84 }");
        var cfg = VoiceConfig.Load(path);
        Assert.True(cfg.Enabled);
        Assert.Equal("ptt", cfg.Mode);
        Assert.Equal(84, cfg.PttKey);
        Assert.Equal(600f, cfg.DNear);       // default
        Assert.Equal(4000f, cfg.DFar);       // default
        Assert.False(cfg.AlwaysAudible);     // default
        File.Delete(path);
    }

    [Fact]
    public void MissingFile_ReturnsDisabledDefault()
    {
        var cfg = VoiceConfig.Load(Path.Combine(Path.GetTempPath(), "does_not_exist_voice.json"));
        Assert.False(cfg.Enabled);
    }
}
```

- [ ] **Step 2: Run it to confirm it fails**

Run: `dotnet test lobby/companion.tests/Companion.Tests.csproj --filter VoiceConfigTests`
Expected: FAIL (VoiceConfig missing).

- [ ] **Step 3: Implement VoiceConfig**

`lobby/companion/Voice/VoiceConfig.cs`:

```csharp
using System;
using System.IO;
using System.Text.Json;
using System.Text.Json.Serialization;

namespace Pulsar.Companion.Voice;

public sealed class VoiceConfig
{
    [JsonPropertyName("enabled")]        public bool   Enabled { get; set; } = false;
    [JsonPropertyName("mode")]           public string Mode { get; set; } = "ptt";     // "ptt" | "open"
    [JsonPropertyName("pttKey")]         public int    PttKey { get; set; } = 0;
    [JsonPropertyName("alwaysAudible")]  public bool   AlwaysAudible { get; set; } = false;
    [JsonPropertyName("alwaysAudibleKey")] public int  AlwaysAudibleKey { get; set; } = 0;
    [JsonPropertyName("inputDevice")]    public string InputDevice { get; set; } = "";
    [JsonPropertyName("outputDevice")]   public string OutputDevice { get; set; } = "";
    [JsonPropertyName("inputVolume")]    public float  InputVolume { get; set; } = 1.0f;
    [JsonPropertyName("outputVolume")]   public float  OutputVolume { get; set; } = 1.0f;
    [JsonPropertyName("dNear")]          public float  DNear { get; set; } = 600f;
    [JsonPropertyName("dFar")]           public float  DFar { get; set; } = 4000f;

    public static string DefaultPath =>
        Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
                     "PULSAR", "pulsar_voice.json");

    public static VoiceConfig Load(string path)
    {
        try
        {
            if (!File.Exists(path)) return new VoiceConfig();
            return JsonSerializer.Deserialize<VoiceConfig>(File.ReadAllText(path)) ?? new VoiceConfig();
        }
        catch { return new VoiceConfig(); }
    }
}
```

Verify `DefaultPath` matches the folder the Companion already writes `pulsar_lobbies.json` into (grep the Companion for `pulsar_lobbies.json`); if that code uses a different base dir, reuse the exact same base-dir helper here so tool and Companion agree.

- [ ] **Step 4: Run the test to confirm it passes**

Run: `dotnet test lobby/companion.tests/Companion.Tests.csproj --filter VoiceConfigTests`
Expected: PASS (both).

- [ ] **Step 5: Commit**

```bash
git add lobby/companion/Voice/VoiceConfig.cs lobby/companion.tests/VoiceConfigTests.cs
git commit -m "feat(voice): pulsar_voice.json config model + loader (TDD)"
```

### Task 11: OpusCodec wrapper + local capture/render self-loop

**Files:**
- Create: `lobby/companion/Voice/OpusCodec.cs`
- Create (temporary probe): `lobby/companion/Voice/VoiceSelfLoopProbe.cs`

- [ ] **Step 1: Implement the Opus wrapper**

`lobby/companion/Voice/OpusCodec.cs`:

```csharp
using Concentus.Enums;
using Concentus.Structs;

namespace Pulsar.Companion.Voice;

// 48kHz mono 20ms frames (960 samples). Classic Concentus array API.
public sealed class OpusCodec
{
    public const int SampleRate = 48000;
    public const int Channels   = 1;
    public const int FrameSamples = 960;   // 20ms @ 48kHz

    private readonly OpusEncoder _enc;
    private readonly OpusDecoder _dec;

    public OpusCodec()
    {
        _enc = new OpusEncoder(SampleRate, Channels, OpusApplication.OPUS_APPLICATION_VOIP);
        _enc.Bitrate = 24000;
        _dec = new OpusDecoder(SampleRate, Channels);
    }

    // pcm: FrameSamples shorts -> returns opus byte length written into outBuf.
    public int Encode(short[] pcm, byte[] outBuf)
        => _enc.Encode(pcm, 0, FrameSamples, outBuf, 0, outBuf.Length);

    // opus[0..len) -> FrameSamples shorts into outPcm. Returns samples decoded.
    public int Decode(byte[] opus, int len, short[] outPcm)
        => _dec.Decode(opus, 0, len, outPcm, 0, FrameSamples, false);
}
```

If `dotnet build` reports the Concentus API differs (2.x uses `OpusCodecFactory` + `Span`), pin `Concentus` to `1.1.7` in the csproj (Task 7) so this classic API resolves; do not rewrite to the Span API.

- [ ] **Step 2: Write a temporary self-loop probe (mic -> opus -> decode -> speakers)**

`lobby/companion/Voice/VoiceSelfLoopProbe.cs`:

```csharp
using System;
using NAudio.Wave;

namespace Pulsar.Companion.Voice;

// Temporary: run with `companion voiceprobe`. Captures the default mic, encodes
// to Opus, decodes, and plays back so you hear yourself with a short delay.
// Confirms NAudio + Concentus work on the target machine before wiring the net.
public static class VoiceSelfLoopProbe
{
    public static void Run()
    {
        var codec = new OpusCodec();
        var fmt = new WaveFormat(OpusCodec.SampleRate, 16, OpusCodec.Channels);
        var play = new BufferedWaveProvider(fmt) { DiscardOnBufferOverflow = true };
        using var outp = new WaveOutEvent();
        outp.Init(play); outp.Play();

        using var cap = new WaveInEvent { WaveFormat = fmt, BufferMilliseconds = 20 };
        var opus = new byte[OpusCodec.kMaxOpus];
        var pcmIn = new short[OpusCodec.FrameSamples];
        var pcmOut = new short[OpusCodec.FrameSamples];
        cap.DataAvailable += (_, e) =>
        {
            int samples = e.BytesRecorded / 2;
            if (samples < OpusCodec.FrameSamples) return;
            Buffer.BlockCopy(e.Buffer, 0, pcmIn, 0, OpusCodec.FrameSamples * 2);
            int n = codec.Encode(pcmIn, opus);
            codec.Decode(opus, n, pcmOut);
            var outBytes = new byte[OpusCodec.FrameSamples * 2];
            Buffer.BlockCopy(pcmOut, 0, outBytes, 0, outBytes.Length);
            play.AddSamples(outBytes, 0, outBytes.Length);
        };
        cap.StartRecording();
        Console.WriteLine("voiceprobe: talk into the mic; you should hear yourself. Enter to stop.");
        Console.ReadLine();
        cap.StopRecording();
    }
}
```

Add `public const int kMaxOpus = 400;` to `OpusCodec`. Add a `case "voiceprobe": VoiceSelfLoopProbe.Run(); return;` branch to the Companion command dispatch in `Program.cs` (temporary).

- [ ] **Step 3: Build + run the probe**

Run: `dotnet run --project lobby/companion -- voiceprobe`
Expected: you hear your own voice echoed back with ~40-60 ms delay, no crash. Confirms capture, Opus round-trip, and render all work.

- [ ] **Step 4: Remove the temporary probe branch (keep OpusCodec)**

Delete `VoiceSelfLoopProbe.cs` and the `voiceprobe` dispatch branch. Keep `OpusCodec.cs`.

- [ ] **Step 5: Commit**

```bash
git add lobby/companion/Voice/OpusCodec.cs
git commit -m "feat(voice): Concentus Opus codec wrapper (verified via self-loop probe)"
```

---

# Phase 2 - Full wire (Companion audio over the DLL loopback to the peer)

### Task 12: VoiceBridgeClient (localhost UDP to the DLL)

**Files:**
- Create: `lobby/companion/Voice/VoiceBridgeClient.cs`

- [ ] **Step 1: Implement the client**

`lobby/companion/Voice/VoiceBridgeClient.cs`:

```csharp
using System;
using System.Net;
using System.Net.Sockets;
using System.Threading;

namespace Pulsar.Companion.Voice;

// Owns the Companion side of the localhost loopback. Sends HELLO on start,
// sends mic frames + settings, and raises events for peer audio + session
// state pushed by the DLL. One background receive thread.
public sealed class VoiceBridgeClient : IDisposable
{
    private readonly UdpClient _udp;
    private readonly IPEndPoint _dll = new(IPAddress.Loopback, VoiceFraming.LoopbackPort);
    private readonly Thread _rx;
    private volatile bool _run = true;

    public event Action<float, ushort, byte, byte[]>? PeerAudio;   // gain, seq, flags, opus
    public event Action? SessionUp;
    public event Action? SessionDown;

    public VoiceBridgeClient()
    {
        _udp = new UdpClient(new IPEndPoint(IPAddress.Loopback, 0));  // ephemeral
        _udp.Send(VoiceFraming.Hello(), 1, _dll);
        _rx = new Thread(RxLoop) { IsBackground = true, Name = "voice-rx" };
        _rx.Start();
    }

    public void SendMic(ushort seq, byte flags, byte[] opus, int opusLen)
    {
        var f = VoiceFraming.PackMicFrame(seq, flags, opus, opusLen);
        try { _udp.Send(f, f.Length, _dll); } catch { }
    }

    public void SendSettings(float dNear, float dFar, bool alwaysAudible)
    {
        var f = VoiceFraming.PackSettings(dNear, dFar, alwaysAudible);
        try { _udp.Send(f, f.Length, _dll); } catch { }
    }

    private void RxLoop()
    {
        var any = new IPEndPoint(IPAddress.Any, 0);
        while (_run)
        {
            byte[] buf;
            try { buf = _udp.Receive(ref any); }
            catch { if (!_run) break; continue; }
            if (buf.Length == 0) continue;
            switch (buf[0])
            {
                case VoiceFraming.TagAudio:
                    if (VoiceFraming.TryParsePeerFrame(buf, out var g, out var s, out var fl, out var p))
                        PeerAudio?.Invoke(g, s, fl, p);
                    break;
                case VoiceFraming.TagSessionUp:   SessionUp?.Invoke();   break;
                case VoiceFraming.TagSessionDown: SessionDown?.Invoke(); break;
            }
        }
    }

    public void Dispose()
    {
        _run = false;
        try { _udp.Close(); } catch { }
        try { _rx.Join(500); } catch { }
    }
}
```

- [ ] **Step 2: Build**

Run: `dotnet build lobby/companion/PulsarCompanion.csproj`
Expected: builds clean.

- [ ] **Step 3: Commit**

```bash
git add lobby/companion/Voice/VoiceBridgeClient.cs
git commit -m "feat(voice): VoiceBridgeClient localhost loopback to the DLL"
```

### Task 13: PttHook (global WH_KEYBOARD_LL)

**Files:**
- Create: `lobby/companion/Voice/PttHook.cs`

- [ ] **Step 1: Implement the low-level keyboard hook**

`lobby/companion/Voice/PttHook.cs`:

```csharp
using System;
using System.Collections.Concurrent;
using System.Diagnostics;
using System.Runtime.InteropServices;
using System.Threading;

namespace Pulsar.Companion.Voice;

// Global low-level keyboard hook (works while Starfield has focus). Tracks the
// down/up state of any watched virtual-key. Runs its own message-pump thread
// because WH_KEYBOARD_LL requires a message loop.
public sealed class PttHook : IDisposable
{
    private const int WH_KEYBOARD_LL = 13;
    private const int WM_KEYDOWN = 0x0100, WM_KEYUP = 0x0101,
                      WM_SYSKEYDOWN = 0x0104, WM_SYSKEYUP = 0x0105;

    private readonly ConcurrentDictionary<int, bool> _down = new();
    private readonly Thread _pump;
    private IntPtr _hook = IntPtr.Zero;
    private LowLevelKeyboardProc? _proc;   // keep alive
    private volatile bool _run = true;

    public PttHook()
    {
        _pump = new Thread(Pump) { IsBackground = true, Name = "voice-ptt" };
        _pump.Start();
    }

    public bool IsKeyDown(int vk) => vk != 0 && _down.TryGetValue(vk, out var d) && d;

    private void Pump()
    {
        _proc = HookProc;
        using var mod = Process.GetCurrentProcess().MainModule!;
        _hook = SetWindowsHookEx(WH_KEYBOARD_LL, _proc, GetModuleHandle(mod.ModuleName), 0);
        while (_run && GetMessage(out var msg, IntPtr.Zero, 0, 0) > 0) { /* pump */ }
        if (_hook != IntPtr.Zero) UnhookWindowsHookEx(_hook);
    }

    private IntPtr HookProc(int nCode, IntPtr wParam, IntPtr lParam)
    {
        if (nCode >= 0)
        {
            int vk = Marshal.ReadInt32(lParam);
            int msg = wParam.ToInt32();
            if (msg == WM_KEYDOWN || msg == WM_SYSKEYDOWN) _down[vk] = true;
            else if (msg == WM_KEYUP || msg == WM_SYSKEYUP) _down[vk] = false;
        }
        return CallNextHookEx(_hook, nCode, wParam, lParam);
    }

    public void Dispose()
    {
        _run = false;
        PostThreadMessage((uint)_pump.ManagedThreadId, 0x0012 /*WM_QUIT*/, IntPtr.Zero, IntPtr.Zero);
        try { _pump.Join(500); } catch { }
    }

    private delegate IntPtr LowLevelKeyboardProc(int nCode, IntPtr wParam, IntPtr lParam);
    [DllImport("user32.dll", SetLastError = true)]
    private static extern IntPtr SetWindowsHookEx(int idHook, LowLevelKeyboardProc lpfn, IntPtr hMod, uint dwThreadId);
    [DllImport("user32.dll", SetLastError = true)]
    private static extern bool UnhookWindowsHookEx(IntPtr hhk);
    [DllImport("user32.dll")]
    private static extern IntPtr CallNextHookEx(IntPtr hhk, int nCode, IntPtr wParam, IntPtr lParam);
    [DllImport("user32.dll")]
    private static extern int GetMessage(out MSG lpMsg, IntPtr hWnd, uint min, uint max);
    [DllImport("user32.dll")]
    private static extern bool PostThreadMessage(uint idThread, uint msg, IntPtr wParam, IntPtr lParam);
    [DllImport("kernel32.dll", CharSet = CharSet.Auto)]
    private static extern IntPtr GetModuleHandle(string name);
    [StructLayout(LayoutKind.Sequential)]
    private struct MSG { public IntPtr hwnd; public uint message; public IntPtr wParam, lParam; public uint time; public int ptX, ptY; }
}
```

Note: `PostThreadMessage` needs the native thread id, not `ManagedThreadId`. If the WM_QUIT does not stop the pump in testing, store the native id via `GetCurrentThreadId()` inside `Pump()` and post to that instead. This only affects clean shutdown, not function.

- [ ] **Step 2: Build**

Run: `dotnet build lobby/companion/PulsarCompanion.csproj`
Expected: builds clean.

- [ ] **Step 3: Manual smoke test**

Add a temporary `case "ptttest":` branch that constructs `PttHook`, then loops printing `IsKeyDown(84)` (the `T` key) 10x/sec for 10 seconds.
Run: `dotnet run --project lobby/companion -- ptttest`
Expected: prints `True` while T is held (even if another window has focus), `False` otherwise. Remove the branch after.

- [ ] **Step 4: Commit**

```bash
git add lobby/companion/Voice/PttHook.cs
git commit -m "feat(voice): global WH_KEYBOARD_LL PTT hook"
```

### Task 14: VoiceEngine (orchestration)

**Files:**
- Create: `lobby/companion/Voice/VoiceEngine.cs`

- [ ] **Step 1: Implement the engine**

`lobby/companion/Voice/VoiceEngine.cs`:

```csharp
using System;
using System.Buffers.Binary;
using NAudio.CoreAudioApi;
using NAudio.Wave;

namespace Pulsar.Companion.Voice;

// Wires capture -> gate -> Opus -> loopback, and loopback -> jitter -> Opus ->
// gain -> render. Active only between SessionUp and SessionDown. One instance
// for the whole Companion lifetime; Start/Stop toggle the audio devices.
public sealed class VoiceEngine : IDisposable
{
    private readonly VoiceBridgeClient _bridge;
    private readonly PttHook _ptt;
    private readonly OpusCodec _codec = new();
    private readonly JitterBuffer _jitter = new(targetDepth: 4, capacity: 32);
    private VoiceConfig _cfg;

    private WaveInEvent? _capture;
    private WaveOutEvent? _render;
    private BufferedWaveProvider? _playout;
    private ushort _seq;
    private bool _active;

    private readonly byte[] _opus = new byte[OpusCodec.kMaxOpus];
    private readonly short[] _pcmIn = new short[OpusCodec.FrameSamples];
    private readonly short[] _pcmOut = new short[OpusCodec.FrameSamples];

    public VoiceEngine(VoiceConfig cfg)
    {
        _cfg = cfg;
        _bridge = new VoiceBridgeClient();
        _ptt = new PttHook();
        _bridge.SessionUp   += Start;
        _bridge.SessionDown += Stop;
        _bridge.PeerAudio   += OnPeerAudio;
        // push initial proximity settings to the DLL
        _bridge.SendSettings(_cfg.DNear, _cfg.DFar, _cfg.AlwaysAudible);
    }

    public void ApplyConfig(VoiceConfig cfg)
    {
        _cfg = cfg;
        _bridge.SendSettings(cfg.DNear, cfg.DFar, cfg.AlwaysAudible);
        if (_active) { Stop(); if (cfg.Enabled) Start(); }   // device / mode change
    }

    private void Start()
    {
        if (_active || !_cfg.Enabled) return;
        var fmt = new WaveFormat(OpusCodec.SampleRate, 16, OpusCodec.Channels);
        _playout = new BufferedWaveProvider(fmt) { DiscardOnBufferOverflow = true };
        _render = new WaveOutEvent();
        _render.Init(_playout); _render.Play();
        _capture = new WaveInEvent { WaveFormat = fmt, BufferMilliseconds = 20,
                                     DeviceNumber = ResolveInputDevice() };
        _capture.DataAvailable += OnMicData;
        _capture.StartRecording();
        _active = true;
        Console.WriteLine("[companion] voice: session up");
    }

    private void Stop()
    {
        if (!_active) return;
        try { _capture?.StopRecording(); _capture?.Dispose(); } catch { }
        try { _render?.Stop(); _render?.Dispose(); } catch { }
        _capture = null; _render = null; _playout = null; _active = false;
        Console.WriteLine("[companion] voice: session down");
    }

    private bool MicOpen()
        => _cfg.Mode == "open" || (_cfg.PttKey != 0 && _ptt.IsKeyDown(_cfg.PttKey));

    private void OnMicData(object? _, WaveInEventArgs e)
    {
        if (!MicOpen()) return;
        int samples = e.BytesRecorded / 2;
        if (samples < OpusCodec.FrameSamples) return;
        Buffer.BlockCopy(e.Buffer, 0, _pcmIn, 0, OpusCodec.FrameSamples * 2);
        ApplyGainInPlace(_pcmIn, _cfg.InputVolume);
        int n = _codec.Encode(_pcmIn, _opus);
        _bridge.SendMic(_seq++, flags: 1, _opus, n);
    }

    private void OnPeerAudio(float gain, ushort seq, byte flags, byte[] opus)
        => _jitter.Push(seq, opus);   // decode happens on the render pull below

    // NAudio pulls from _playout on its own thread; we feed it from a timer-like
    // pump. Simplest robust approach: decode+enqueue on each peer frame arrival
    // through the jitter buffer's steady drain.
    public void PumpPlayout(float lastGain)
    {
        if (!_active || _playout == null) return;
        var frame = _jitter.Pop();
        if (frame == null) return;
        int got = _codec.Decode(frame, frame.Length, _pcmOut);
        if (got <= 0) return;
        ApplyGainInPlace(_pcmOut, lastGain * _cfg.OutputVolume);
        var outBytes = new byte[OpusCodec.FrameSamples * 2];
        Buffer.BlockCopy(_pcmOut, 0, outBytes, 0, outBytes.Length);
        _playout.AddSamples(outBytes, 0, outBytes.Length);
    }

    private static void ApplyGainInPlace(short[] pcm, float gain)
    {
        if (Math.Abs(gain - 1.0f) < 0.001f) return;
        for (int i = 0; i < pcm.Length; i++)
            pcm[i] = (short)Math.Clamp((int)(pcm[i] * gain), short.MinValue, short.MaxValue);
    }

    private int ResolveInputDevice()
    {
        if (string.IsNullOrEmpty(_cfg.InputDevice)) return 0;
        for (int i = 0; i < WaveInEvent.DeviceCount; i++)
            if (WaveInEvent.GetCapabilities(i).ProductName.Contains(_cfg.InputDevice, StringComparison.OrdinalIgnoreCase))
                return i;
        return 0;
    }

    public void Dispose() { Stop(); _ptt.Dispose(); _bridge.Dispose(); }
}
```

Design note on the playout pull: `PumpPlayout` must be called steadily (every ~20 ms) with the gain from the most recent peer frame. Store the last gain in `OnPeerAudio` (add a `private volatile float _lastGain = 1f;` set there, and call `PumpPlayout(_lastGain)` from a 20 ms `System.Threading.Timer` started in `Start` / stopped in `Stop`). Wire that timer in this task.

- [ ] **Step 2: Add the 20 ms playout timer**

In `Start()`, after `_active = true;` add:

```csharp
        _playTimer = new System.Threading.Timer(_ => PumpPlayout(_lastGain), null, 0, 20);
```

Add fields `private System.Threading.Timer? _playTimer;` and `private volatile float _lastGain = 1f;`, set `_lastGain = gain;` at the top of `OnPeerAudio`, and in `Stop()` add `_playTimer?.Dispose(); _playTimer = null;`.

- [ ] **Step 3: Build**

Run: `dotnet build lobby/companion/PulsarCompanion.csproj`
Expected: builds clean.

- [ ] **Step 4: Commit**

```bash
git add lobby/companion/Voice/VoiceEngine.cs
git commit -m "feat(voice): VoiceEngine capture/encode/loopback + jitter/decode/render"
```

### Task 15: Companion lifecycle wiring

**Files:**
- Modify: `lobby/companion/Program.cs` (the `serve` loop around line 282-300)

- [ ] **Step 1: Construct the engine in serve**

In `Program.cs`, before the `while (!s_done)` serve loop (near line 282), add:

```csharp
        VoiceEngine? voice = null;
        VoiceConfig voiceCfg = VoiceConfig.Load(VoiceConfig.DefaultPath);
        if (voiceCfg.Enabled)
        {
            voice = new VoiceEngine(voiceCfg);
            Console.WriteLine("[companion] voice: engine constructed (waiting for peer)");
        }
        var voiceCfgWatch = new System.IO.FileSystemWatcher(
            System.IO.Path.GetDirectoryName(VoiceConfig.DefaultPath)!,
            System.IO.Path.GetFileName(VoiceConfig.DefaultPath))
            { EnableRaisingEvents = true, NotifyFilter = System.IO.NotifyFilters.LastWrite };
        voiceCfgWatch.Changed += (_, __) =>
        {
            var c = VoiceConfig.Load(VoiceConfig.DefaultPath);
            if (voice == null && c.Enabled) voice = new VoiceEngine(c);
            else voice?.ApplyConfig(c);
        };
```

Add `using Pulsar.Companion.Voice;` at the top of `Program.cs`.

- [ ] **Step 2: Dispose on exit**

After the serve loop exits (near the existing `conn.Disconnect()` at line 311), add:

```csharp
        try { voiceCfgWatch.Dispose(); voice?.Dispose(); } catch { }
```

- [ ] **Step 3: Build**

Run: `dotnet build lobby/companion/PulsarCompanion.csproj`
Expected: builds clean.

- [ ] **Step 4: Commit**

```bash
git add lobby/companion/Program.cs
git commit -m "feat(voice): construct+lifecycle VoiceEngine in companion serve loop"
```

### Task 16: End-to-end open-mic voice test (2 machines, no proximity)

- [ ] **Step 1: Prep a minimal config on both machines**

Write `%LOCALAPPDATA%\PULSAR\pulsar_voice.json` on both machines:

```json
{ "enabled": true, "mode": "open", "alwaysAudible": true, "outputVolume": 1.0, "inputVolume": 1.0 }
```

(`alwaysAudible: true` bypasses proximity so this test is pure transport/audio.)

- [ ] **Step 2: Deploy dev DLL + run**

Run: `bash deploy.sh` (both machines), start co-op, join.
Expected: both `PULSAR.log` show `[VOICE] loopback bound`, `companion registered`, `SESSION_UP` after connect. Companion console shows `voice: session up`.

- [ ] **Step 3: Talk test**

Speak on machine A.
Expected: machine B hears A's voice with low latency (well under a Discord relay hop, since direct P2P). Both directions work. Enabling PTT mode + a key binding gates the mic to key-held.

- [ ] **Step 4: No code change; record the result in the plan checklist**

If audio is choppy, note it for Phase 5 jitter tuning (Task 27). If it works, proceed to proximity.

---

# Phase 3 - Proximity gain (DLL)

### Task 17: ProximityGain pure function + one-off test

**Files:**
- Create: `include/VoiceBridge/ProximityGain.h`
- Create: `tests/voice/test_proximity_gain.cpp`

- [ ] **Step 1: Write the pure gain function**

`include/VoiceBridge/ProximityGain.h`:

```cpp
#pragma once
#include <algorithm>
#include <cmath>
#include <cstdint>

namespace Pulsar::Voice
{
    // Returns 0..1. alwaysAudible short-circuits to 1. Cross-cell (different
    // non-zero cell ids) => 0. Otherwise linear falloff: full <= dNear, silent
    // >= dFar. Both positions are world units (~70 units per metre).
    inline float ComputeProximityGain(const float local[3], const float peer[3],
                                      uint32_t localCell, uint32_t peerCell,
                                      bool alwaysAudible, float dNear, float dFar)
    {
        if (alwaysAudible) return 1.0f;
        if (localCell != 0 && peerCell != 0 && localCell != peerCell) return 0.0f;
        const float dx = local[0] - peer[0];
        const float dy = local[1] - peer[1];
        const float dz = local[2] - peer[2];
        const float dist = std::sqrt(dx * dx + dy * dy + dz * dz);
        if (dFar <= dNear) return dist <= dNear ? 1.0f : 0.0f;
        const float g = (dFar - dist) / (dFar - dNear);
        return std::clamp(g, 0.0f, 1.0f);
    }
}
```

- [ ] **Step 2: Write the standalone test**

`tests/voice/test_proximity_gain.cpp`:

```cpp
#include "../../include/VoiceBridge/ProximityGain.h"
#include <cassert>
#include <cmath>
#include <cstdio>
using namespace Pulsar::Voice;

int main()
{
    float a[3] = {0,0,0};
    float near[3] = {100,0,0};      // within dNear
    float mid[3]  = {2300,0,0};     // between 600 and 4000
    float far[3]  = {5000,0,0};     // beyond dFar

    assert(ComputeProximityGain(a, near, 1, 1, false, 600, 4000) == 1.0f);
    assert(ComputeProximityGain(a, far,  1, 1, false, 600, 4000) == 0.0f);
    float g = ComputeProximityGain(a, mid, 1, 1, false, 600, 4000);
    assert(g > 0.0f && g < 1.0f);
    // cross-cell mute
    assert(ComputeProximityGain(a, near, 1, 2, false, 600, 4000) == 0.0f);
    // always-audible overrides distance + cell
    assert(ComputeProximityGain(a, far,  1, 2, true, 600, 4000) == 1.0f);
    std::printf("ProximityGain: all asserts passed\n");
    return 0;
}
```

- [ ] **Step 3: Compile + run the one-off test**

Run (from repo root, VS dev shell):
`cl /std:c++17 /EHsc tests\voice\test_proximity_gain.cpp /Fe:test_gain.exe && test_gain.exe`
Expected: prints `ProximityGain: all asserts passed`, exit 0.

- [ ] **Step 4: Commit**

```bash
git add include/VoiceBridge/ProximityGain.h tests/voice/test_proximity_gain.cpp
git commit -m "feat(voice): ProximityGain pure function + standalone test"
```

### Task 18: Wire ProximityGain into OnVoicePacket

**Files:**
- Modify: `src/VoiceBridge/VoiceBridge.cpp` (`OnVoicePacket`)

- [ ] **Step 1: Read local + peer position/cell and compute gain**

Replace the Phase-0 body of `VoiceBridge::OnVoicePacket` with:

```cpp
    void VoiceBridge::OnVoicePacket(const uint8_t* a_data, size_t a_len) noexcept
    {
        if (a_len == 0 || a_len > 1200) return;
        float gain = 1.0f;
        __try
        {
            float local[3] = {0,0,0}, peer[3] = {0,0,0};
            uint32_t localCell = 0, peerCell = 0;

            auto* pc = RE::PlayerCharacter::GetSingleton();
            if (pc) {
                // reuse the SAME read as the position broadcast (raw +0x8C/90/94)
                auto* base = reinterpret_cast<const uint8_t*>(pc);
                std::memcpy(local, base + 0x8C, sizeof(local));
                if (auto* cell = pc->parentCell) localCell = cell->formID;
            }
            auto& rpm = Pulsar::RemotePlayer::GetRemotePlayerManager();
            const auto& tp = rpm.m_targetPosData;     // RemotePositionData
            peer[0] = tp.x; peer[1] = tp.y; peer[2] = tp.z;
            peerCell = rpm.GetRemoteCellID();

            gain = ComputeProximityGain(local, peer, localCell, peerCell,
                                        VoiceBridge_alwaysAudible(),
                                        VoiceBridge_dNear(), VoiceBridge_dFar());
        }
        __except (EXCEPTION_EXECUTE_HANDLER) { gain = VoiceBridge_alwaysAudible() ? 1.0f : 0.0f; }

        uint8_t out[1500];
        out[0] = kTagAudio;
        std::memcpy(out + 1, &gain, 4);
        std::memcpy(out + 5, a_data, a_len);
        SendToCompanion(out, static_cast<int>(5 + a_len));
    }
```

Add the includes at the top of `VoiceBridge.cpp`:
```cpp
#include "VoiceBridge/ProximityGain.h"
#include "RemotePlayer/RemotePlayerManager.h"
// plus the CommonLibSF PlayerCharacter/TESObjectCELL headers already used elsewhere
```

Confirm the exact `RemotePositionData` field names (`x/y/z`) in `RemotePlayerManager.h` around line 250; if they are named `posX`/`posY`/`posZ` or nested, adjust. Confirm the namespace of `GetRemotePlayerManager()` (grep — it may be `Pulsar::RemotePlayer::` or just `Pulsar::`). Confirm the local read matches the position-broadcast read in `Main.cpp` (use the identical offset/helper so coordinate spaces match; if the broadcast uses a helper like `GetPlayerPos`, call that instead of the raw memcpy).

- [ ] **Step 2: Build the dev DLL**

Run: `cmake --build build/vs --config RelWithDebInfo --target PULSAR`
Expected: builds clean.

- [ ] **Step 3: Commit**

```bash
git add src/VoiceBridge/VoiceBridge.cpp
git commit -m "feat(voice): compute proximity gain on inbound frames"
```

### Task 19: Confirm the settings path drives the DLL

**Files:**
- (no new files) - verifies Task 3's `kTagSettings` handler + Task 14's `SendSettings`

- [ ] **Step 1: Add a temporary diagnostic**

In `VoiceBridge::Service`, inside the `kTagSettings` branch, add:

```cpp
                PLOG_INFO("[VOICE] settings dNear={} dFar={} always={}", dn, df, buf[9] != 0);
```

- [ ] **Step 2: Build + run with a config that has alwaysAudible=false**

Set `pulsar_voice.json` `alwaysAudible=false`, `dNear=600`, `dFar=4000`. Deploy, start co-op.
Expected: `PULSAR.log` shows `[VOICE] settings dNear=600 dFar=4000 always=0` shortly after the Companion starts (the engine sends settings on construction).

- [ ] **Step 3: Remove the diagnostic, commit**

```bash
git add src/VoiceBridge/VoiceBridge.cpp
git commit -m "chore(voice): verified settings loopback drives DLL gain params"
```

### Task 20: In-game proximity verification

- [ ] **Step 1: Distance falloff**

Two players in the same cell, `alwaysAudible=false`. Player A talks; player B walks away.
Expected: A's voice gets quieter as distance grows past dNear, silent past dFar.

- [ ] **Step 2: Cell mute**

A stays inside; B walks out through a door to a different cell.
Expected: A's voice goes silent for B (cross-cell), and returns when B comes back.

- [ ] **Step 3: Global toggle**

Set `alwaysAudible=true` (edit config, save; the FileSystemWatcher pushes new settings live).
Expected: full volume regardless of distance or cell.

- [ ] **Step 4: No code change**

This is a verification-only task. If falloff feels wrong, tune `dNear`/`dFar` defaults in `VoiceConfig.cs` and `ProximityGain` callers; re-verify.

---

# Phase 4 - Settings UI (VoiceChat tab in the tool)

### Task 21: VoiceChatViewModel

**Files:**
- Create: `lobby/pulsar/ViewModels/VoiceChatViewModel.cs`
- Modify: `lobby/pulsar/Pulsar.csproj` (add NAudio for device enumeration)

- [ ] **Step 1: Add NAudio to the tool**

In `lobby/pulsar/Pulsar.csproj`, inside an `<ItemGroup>`:

```xml
    <PackageReference Include="NAudio" Version="2.2.1" />
```

- [ ] **Step 2: Implement the view model**

`lobby/pulsar/ViewModels/VoiceChatViewModel.cs` (follows the existing VM pattern; if the project uses CommunityToolkit.Mvvm `[ObservableProperty]`, match SettingsViewModel; here shown with plain `ViewModelBase` + setters):

```csharp
using System;
using System.Collections.ObjectModel;
using System.IO;
using System.Linq;
using System.Text.Json;
using NAudio.Wave;

namespace Pulsar.ViewModels;

public class VoiceChatViewModel : ViewModelBase
{
    public ObservableCollection<string> InputDevices { get; } = new();

    private bool _enabled;
    public bool Enabled { get => _enabled; set { _enabled = value; OnChanged(); Save(); } }

    private bool _pushToTalk = true;   // true = ptt, false = open-mic
    public bool PushToTalk { get => _pushToTalk; set { _pushToTalk = value; OnChanged(); Save(); } }

    private int _pttKey;               // Win32 VK
    public int PttKey { get => _pttKey; set { _pttKey = value; OnChanged(); Save(); } }
    public string PttKeyLabel => _pttKey == 0 ? "Click to bind" : KeyName(_pttKey);

    private bool _alwaysAudible;
    public bool AlwaysAudible { get => _alwaysAudible; set { _alwaysAudible = value; OnChanged(); Save(); } }

    private string _inputDevice = "";
    public string InputDevice { get => _inputDevice; set { _inputDevice = value ?? ""; OnChanged(); Save(); } }

    private double _inputVolume = 1.0;
    public double InputVolume { get => _inputVolume; set { _inputVolume = value; OnChanged(); Save(); } }
    private double _outputVolume = 1.0;
    public double OutputVolume { get => _outputVolume; set { _outputVolume = value; OnChanged(); Save(); } }

    private double _dNear = 600, _dFar = 4000;
    public double DNear { get => _dNear; set { _dNear = value; OnChanged(); Save(); } }
    public double DFar  { get => _dFar;  set { _dFar = value;  OnChanged(); Save(); } }

    private static string ConfigPath => Path.Combine(
        Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
        "PULSAR", "pulsar_voice.json");

    public VoiceChatViewModel()
    {
        for (int i = 0; i < WaveInEvent.DeviceCount; i++)
            InputDevices.Add(WaveInEvent.GetCapabilities(i).ProductName);
        Load();
    }

    private void OnChanged() => OnPropertyChanged(string.Empty);

    private void Load()
    {
        try
        {
            if (!File.Exists(ConfigPath)) return;
            using var doc = JsonDocument.Parse(File.ReadAllText(ConfigPath));
            var r = doc.RootElement;
            _enabled = Get(r, "enabled", false);
            _pushToTalk = Get(r, "mode", "ptt") == "ptt";
            _pttKey = Get(r, "pttKey", 0);
            _alwaysAudible = Get(r, "alwaysAudible", false);
            _inputDevice = Get(r, "inputDevice", "");
            _inputVolume = Get(r, "inputVolume", 1.0);
            _outputVolume = Get(r, "outputVolume", 1.0);
            _dNear = Get(r, "dNear", 600.0);
            _dFar = Get(r, "dFar", 4000.0);
            OnChanged();
        }
        catch { }
    }

    public void Save()
    {
        try
        {
            Directory.CreateDirectory(Path.GetDirectoryName(ConfigPath)!);
            var obj = new
            {
                enabled = _enabled,
                mode = _pushToTalk ? "ptt" : "open",
                pttKey = _pttKey,
                alwaysAudible = _alwaysAudible,
                inputDevice = _inputDevice,
                inputVolume = (float)_inputVolume,
                outputVolume = (float)_outputVolume,
                dNear = (float)_dNear,
                dFar = (float)_dFar
            };
            File.WriteAllText(ConfigPath, JsonSerializer.Serialize(obj,
                new JsonSerializerOptions { WriteIndented = true }));
        }
        catch { }
    }

    private static bool   Get(JsonElement r, string k, bool d)   => r.TryGetProperty(k, out var v) ? v.GetBoolean() : d;
    private static int    Get(JsonElement r, string k, int d)    => r.TryGetProperty(k, out var v) ? v.GetInt32()   : d;
    private static double Get(JsonElement r, string k, double d) => r.TryGetProperty(k, out var v) ? v.GetDouble()  : d;
    private static string Get(JsonElement r, string k, string d) => r.TryGetProperty(k, out var v) ? (v.GetString() ?? d) : d;

    private static string KeyName(int vk) => ((char)vk).ToString();  // refined in Task 23
}
```

Match the actual base-class change-notification method (`OnPropertyChanged`) to whatever `ViewModelBase.cs` exposes; grep it and align.

- [ ] **Step 3: Build the tool**

Run: `dotnet build lobby/pulsar/Pulsar.csproj`
Expected: builds clean.

- [ ] **Step 4: Commit**

```bash
git add lobby/pulsar/ViewModels/VoiceChatViewModel.cs lobby/pulsar/Pulsar.csproj
git commit -m "feat(voice): VoiceChatViewModel writes pulsar_voice.json + device list"
```

### Task 22: VoiceChatView.axaml

**Files:**
- Create: `lobby/pulsar/Views/VoiceChatView.axaml`
- Create: `lobby/pulsar/Views/VoiceChatView.axaml.cs`

- [ ] **Step 1: Write the view**

`lobby/pulsar/Views/VoiceChatView.axaml` (mirrors the KeybindsView card/style structure):

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:Pulsar.ViewModels"
             x:Class="Pulsar.Views.VoiceChatView"
             x:DataType="vm:VoiceChatViewModel">
  <Design.DataContext><vm:VoiceChatViewModel /></Design.DataContext>
  <ScrollViewer Padding="32,28">
    <StackPanel Spacing="16" MaxWidth="620" HorizontalAlignment="Left">
      <TextBlock Classes="PageTitle" Text="Voice Chat" />
      <TextBlock Classes="PageSubtitle"
                 Text="Proximity voice over the co-op connection. Louder up close, muted across cells. Turn on 'Always audible' to just talk." />

      <Border Classes="Card">
        <StackPanel Spacing="12">
          <ToggleSwitch Header="Enable voice chat" IsChecked="{Binding Enabled}" />
          <ToggleSwitch Header="Push-to-Talk (off = open mic)" IsChecked="{Binding PushToTalk}" />
          <Grid ColumnDefinitions="*,Auto">
            <TextBlock Grid.Column="0" Text="PTT key" VerticalAlignment="Center" />
            <Button Grid.Column="1" Name="PttBindButton" Content="{Binding PttKeyLabel}" MinWidth="140" />
          </Grid>
          <ToggleSwitch Header="Always audible (ignore distance / cells)" IsChecked="{Binding AlwaysAudible}" />
        </StackPanel>
      </Border>

      <Border Classes="Card">
        <StackPanel Spacing="12">
          <TextBlock Classes="SectionHeader" Text="Devices + levels" />
          <ComboBox HorizontalAlignment="Stretch" ItemsSource="{Binding InputDevices}"
                    SelectedItem="{Binding InputDevice}" PlaceholderText="Default microphone" />
          <TextBlock Text="Mic volume" />
          <Slider Minimum="0" Maximum="2" Value="{Binding InputVolume}" />
          <TextBlock Text="Speaker volume" />
          <Slider Minimum="0" Maximum="2" Value="{Binding OutputVolume}" />
        </StackPanel>
      </Border>

      <Border Classes="Card">
        <StackPanel Spacing="12">
          <TextBlock Classes="SectionHeader" Text="Proximity range" />
          <TextBlock Text="Full volume within (units)" />
          <Slider Minimum="100" Maximum="2000" Value="{Binding DNear}" />
          <TextBlock Text="Silent beyond (units)" />
          <Slider Minimum="1000" Maximum="10000" Value="{Binding DFar}" />
        </StackPanel>
      </Border>
    </StackPanel>
  </ScrollViewer>
</UserControl>
```

- [ ] **Step 2: Write the code-behind**

`lobby/pulsar/Views/VoiceChatView.axaml.cs`:

```csharp
using Avalonia.Controls;
using Avalonia.Markup.Xaml;

namespace Pulsar.Views;

public partial class VoiceChatView : UserControl
{
    public VoiceChatView() => AvaloniaXamlLoader.Load(this);
}
```

- [ ] **Step 3: Build the tool**

Run: `dotnet build lobby/pulsar/Pulsar.csproj`
Expected: builds clean; XAML compiles.

- [ ] **Step 4: Commit**

```bash
git add lobby/pulsar/Views/VoiceChatView.axaml lobby/pulsar/Views/VoiceChatView.axaml.cs
git commit -m "feat(voice): VoiceChat tab view"
```

### Task 23: PTT key capture (reuse Keybinds pattern)

**Files:**
- Modify: `lobby/pulsar/Views/VoiceChatView.axaml.cs`
- Modify: `lobby/pulsar/ViewModels/VoiceChatViewModel.cs` (`KeyName`)

- [ ] **Step 1: Capture the next key press into PttKey**

Read how `KeybindsView.axaml.cs` captures a key (it listens for a `KeyDown` after the bind button is clicked and maps `Avalonia.Input.Key` to the stored code). Mirror it in `VoiceChatView.axaml.cs`:

```csharp
using Avalonia.Controls;
using Avalonia.Input;
using Avalonia.Interactivity;
using Avalonia.Markup.Xaml;
using Pulsar.ViewModels;

namespace Pulsar.Views;

public partial class VoiceChatView : UserControl
{
    private bool _binding;
    public VoiceChatView()
    {
        AvaloniaXamlLoader.Load(this);
        this.FindControl<Button>("PttBindButton")!.Click += OnBindClick;
        AddHandler(KeyDownEvent, OnKeyDown, RoutingStrategies.Tunnel);
    }

    private void OnBindClick(object? s, RoutedEventArgs e) => _binding = true;

    private void OnKeyDown(object? s, KeyEventArgs e)
    {
        if (!_binding || DataContext is not VoiceChatViewModel vm) return;
        vm.PttKey = KeyInterop.VirtualKeyFromKey(e.Key);   // Avalonia Key -> Win32 VK
        _binding = false;
        e.Handled = true;
    }
}
```

If `KeyInterop.VirtualKeyFromKey` is not available in the Avalonia version used, copy the small `Key`->VK switch that `KeybindsView` already uses (it must convert similarly to write `PULSAR.ini`). Reuse that exact mapping so both tabs agree.

- [ ] **Step 2: Human-readable key label**

Replace `KeyName` in `VoiceChatViewModel` with the same VK->name map KeybindsViewModel uses (grep it). At minimum:

```csharp
    private static string KeyName(int vk) => vk switch
    {
        0 => "Click to bind",
        >= 0x41 and <= 0x5A => ((char)vk).ToString(),          // A-Z
        >= 0x70 and <= 0x7B => "F" + (vk - 0x6F),              // F1-F12
        0x20 => "Space",
        _ => "Key " + vk
    };
```

and ensure `PttKeyLabel` raises change when `PttKey` changes (already via `OnChanged()`).

- [ ] **Step 3: Build**

Run: `dotnet build lobby/pulsar/Pulsar.csproj`
Expected: builds clean.

- [ ] **Step 4: Commit**

```bash
git add lobby/pulsar/Views/VoiceChatView.axaml.cs lobby/pulsar/ViewModels/VoiceChatViewModel.cs
git commit -m "feat(voice): PTT key capture + readable key label"
```

### Task 24: Add the VoiceChat tab to the nav

**Files:**
- Modify: `lobby/pulsar/Views/MainWindow.axaml`
- Modify: `lobby/pulsar/ViewModels/MainWindowViewModel.cs`

- [ ] **Step 1: Find how tabs are registered**

Grep `MainWindow.axaml` for the existing nav entries (e.g. how `KeybindsView` / `SettingsView` are listed - likely a `NavigationView` / `ListBox` of items plus a content selector). Note the exact pattern (icon + label + a selected-page enum or DataTemplate).

- [ ] **Step 2: Add the nav item + page**

Following that exact pattern, add a "Voice Chat" entry pointing at `VoiceChatView` / `VoiceChatViewModel`. If `MainWindowViewModel` holds page VMs as properties (grep for `KeybindsViewModel` usage there), add:

```csharp
    public VoiceChatViewModel VoiceChat { get; } = new();
```

and register the matching `DataTemplate` (in `App.axaml` or `MainWindow.axaml` where the other `*ViewModel -> *View` templates live) so `VoiceChatViewModel` resolves to `VoiceChatView`. Match the neighbouring templates exactly.

- [ ] **Step 3: Build + run the tool**

Run: `dotnet run --project lobby/pulsar`
Expected: a "Voice Chat" tab appears; clicking it shows the settings page; toggling values writes `%LOCALAPPDATA%\PULSAR\pulsar_voice.json` (verify the file updates).

- [ ] **Step 4: Commit**

```bash
git add lobby/pulsar/Views/MainWindow.axaml lobby/pulsar/ViewModels/MainWindowViewModel.cs
git commit -m "feat(voice): register VoiceChat tab in the tool nav"
```

### Task 25: Live-settings verification (tool <-> companion <-> DLL)

- [ ] **Step 1: Full-stack live test**

Run the tool + start co-op. In the VoiceChat tab: toggle Enable, switch PTT/open, bind a key, drag the range sliders, toggle Always audible.
Expected: the Companion's FileSystemWatcher (Task 15) reloads on each save; `voice: session up/down` and settings changes take effect without restarting the DLL or the game. Bound PTT key gates the mic live. Range slider changes shift the falloff (verify by walking).

- [ ] **Step 2: No code change - record results**

Note any lag in the watcher (Windows can fire multiple `Changed` events; if it double-applies, add a 250 ms debounce in the Task 15 handler). Fix inline if observed.

---

# Phase 5 - Polish

### Task 26: Version bump + protocol gate

**Files:**
- Modify: `include/Plugin.h` (Version), `CMakeLists.txt:7` (VERSION), `lobby/pulsar/Pulsar.csproj` (Version/Assembly/File), `lobby/companion/PulsarCompanion.csproj` (Version)
- Modify: the handshake version check in `src/Network/NetworkHost.cpp` (the PKT_HANDSHAKE handler) if it hard-gates on version

- [ ] **Step 1: Bump all four version strings in lockstep**

Pick the next release version (e.g. `0.3.0` because this adds a protocol channel). Update:
- `include/Plugin.h`: `REL::Version{ 0, 3, 0, 0 }`
- `CMakeLists.txt` line 7: `VERSION 0.3.0`
- `lobby/pulsar/Pulsar.csproj`: `<Version>`, `<AssemblyVersion>0.3.0.0</AssemblyVersion>`, `<FileVersion>0.3.0.0</FileVersion>`
- `lobby/companion/PulsarCompanion.csproj`: `<Version>0.3.0</Version>`

- [ ] **Step 2: Ensure NUM_CHANNELS mismatch cannot half-connect**

The `NUM_CHANNELS 3` change means a 0.3.0 client and a pre-0.3.0 client negotiate different channel counts. ENet uses the lower of the two peers' channel counts, so an old peer would simply never receive channel-2 traffic (voice silently absent) but the rest still works. Confirm the `HandshakeData` version check in `NetworkHost.cpp` already rejects mismatched majors/minors if you want a hard gate; if it only warns, leave voice as a soft-degrade (old peer = no voice, no crash). Document the chosen behaviour in a code comment at the `NUM_CHANNELS` definition.

- [ ] **Step 3: Build both configs**

Run: `cmake --build build/vs --config RelWithDebInfo --target PULSAR`
Run: `cmake --build build/vs --config Release --target PULSAR`
Expected: both build; the Release post-build `verify_hardening.ps1` passes.

- [ ] **Step 4: Commit**

```bash
git add include/Plugin.h CMakeLists.txt lobby/pulsar/Pulsar.csproj lobby/companion/PulsarCompanion.csproj src/Network/NetworkHost.cpp
git commit -m "chore(voice): version bump 0.3.0 + document NUM_CHANNELS protocol change"
```

### Task 27: Noise gate + jitter tuning

**Files:**
- Modify: `lobby/companion/Voice/VoiceEngine.cs` (`OnMicData` gate; jitter depth)

- [ ] **Step 1: Add a simple RMS noise gate for open-mic**

In `VoiceEngine.OnMicData`, before encoding, in open-mic mode drop frames below an energy threshold so background hiss does not stream constantly:

```csharp
        if (_cfg.Mode == "open")
        {
            long sum = 0;
            for (int i = 0; i < OpusCodec.FrameSamples; i++) sum += (long)_pcmIn[i] * _pcmIn[i];
            double rms = Math.Sqrt(sum / (double)OpusCodec.FrameSamples);
            if (rms < 350) return;   // silence gate; tune 200-600 to taste
        }
```

(PTT mode is already gated by the key, so the RMS gate only applies to open-mic.)

- [ ] **Step 2: Tune jitter depth from the Phase 2 result**

If Task 16 audio was choppy, raise the `JitterBuffer` `targetDepth` in the `VoiceEngine` field init from 4 to 6 (adds ~40 ms latency, smoother). If it was smooth, drop to 3 for lower latency. Pick one, comment the choice.

- [ ] **Step 3: Rebuild + quick two-machine listen test**

Run: `dotnet build lobby/companion/PulsarCompanion.csproj`
Expected: builds; in-game the open-mic no longer streams during silence, and audio is smooth.

- [ ] **Step 4: Commit**

```bash
git add lobby/companion/Voice/VoiceEngine.cs
git commit -m "feat(voice): open-mic noise gate + jitter depth tuning"
```

### Task 28: Full test pass + hardened build

- [ ] **Step 1: Run the C# unit tests**

Run: `dotnet test lobby/companion.tests/Companion.Tests.csproj`
Expected: all VoiceFraming / JitterBuffer / VoiceConfig tests PASS.

- [ ] **Step 2: Run the C++ gain test**

Run: `cl /std:c++17 /EHsc tests\voice\test_proximity_gain.cpp /Fe:test_gain.exe && test_gain.exe`
Expected: `ProximityGain: all asserts passed`.

- [ ] **Step 3: Build both DLL configs + confirm hardening**

Run: `cmake --build build/vs --config Release --target PULSAR`
Expected: `verify_hardening.ps1` post-build passes (no leak strings; the only `[PULSAR]`/`[VOICE]` strings that survive are DIAG-allowlisted). If any `[VOICE]` INFO/WARN string trips the hardening scanner, confirm it is compiled out under `PULSAR_HARDENED` (INFO/WARN are stripped in Release per `include/Log.h`) or add it to the curated allowlist.

- [ ] **Step 4: Final in-game acceptance (both machines, hardened DLL)**

Deploy the Release DLL + published tool/companion. Verify: enable voice, PTT with a bound key, hear each other, walk apart -> quieter, cross a cell -> muted, toggle Always audible -> full volume. Disconnect -> voice stops cleanly (SESSION_DOWN).

- [ ] **Step 5: Commit any final tuning + update task**

```bash
git add -A
git commit -m "feat(voice): proximity voice chat complete "
```

---

## Self-Review notes (author)

- **Spec coverage:** every spec section maps to tasks - Companion audio (7-16, 27), DLL VoiceBridge transport (1-6), proximity gain + global toggle (17-20), VoiceChat tab + PTT bind + config (21-25), wire protocols (1, 8, loopback contract section), version bump/hardening (26, 28). The lifecycle refinement (session up/down over loopback) and settings-forwarding refinement are folded into Tasks 3/4/14/15/19.
- **Deferred verifications (not placeholders):** three RE anchors are intentionally re-confirmed at implementation time because RE facts drift - `RemotePositionData` field names, the `GetRemotePlayerManager()` namespace, and the exact local-position read used by the broadcast. Each has a concrete grep instruction in the owning task (18).
- **Type consistency:** loopback tags, `VoiceFraming` constants, and `VoiceProtocol.h` share one contract; `OpusCodec.FrameSamples/kMaxOpus`, `VoiceConfig` field names, and the JSON keys are used identically across Companion, tool, and tests.
- **Adaptation to this codebase:** the DLL has no unit-test harness, so pure logic is tested standalone (gain) and integration is verified via `PULSAR.log` diagnostics + two-machine in-game runs, matching the project's established verification style. Pure C# logic uses real xUnit tests.
