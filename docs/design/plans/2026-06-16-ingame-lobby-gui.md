# In-Game Lobby GUI (Teil 3) Implementation Plan


**Goal:** A hotkey-opened, Starfield-native Scaleform menu to browse / host / join lobbies, front-ending the existing STDB/companion persistence.

**Architecture:** The PULSAR DLL hosts a custom `RE::IMenu` subclass (`LobbyMenu`) rendered by a `.swf`. The companion becomes a local `serve` daemon: it stays connected to STDB, writes the live lobby list to `pulsar_lobbies.json`, and watches `pulsar_cmd.json` for host/join requests. The DLL renders the list into the movie, writes commands on button clicks, and drives the ENet connect directly once the companion stages the profile.

**Tech Stack:** SFSE C++ plugin, CommonLibSF Scaleform (`IMenu`/`UI`/`SWFToCodeFunctionHandler`/`ScaleformGFxValue`), a `.swf` (ActionScript), the .NET 8 companion.

**Specs:** `docs/design/specs/2026-06-16-ingame-lobby-gui-design.md` (+ depends on the shipped `2026-06-16-lobby-security-tier1` for the secure read path).

**Resolved engine IDs (1.16.236 versionlib — CommonLibSF ships these as `{0}`):**
| Symbol | REL::ID | RVA |
|---|---|---|
| `UI::Singleton` | 937580 | 0x1405FE1900 |
| `UI::RegisterMenu` | 80375 | 0x1411A74E0 |
| `UI::IsMenuOpen` | 130475 | 0x142547180 |
| `IMenu::dtor` | 187216 | 0x1437CB520 |
| `IMenu::LoadMovie` | 187240 | 0x1437D1DD0 |
| `IMenu::ProcessMessage` | 187247 | 0x1437D1F90 |
| `UIMessageQueue::Singleton` | 937897 | (resolved in CLibSF) |
| `UIMessageQueue::AddMessage` | 130659 | (resolved in CLibSF) |

**Conventions:** Build `cmake --build build/vs --config RelWithDebInfo --target PULSAR`. Deploy DLL+PDB → `<Starfield>\Data\SFSE\Plugins` and DLL → `\\<NAS>\PULSAR\deploy\Data\SFSE\Plugins`. `.swf` → `<Starfield>\Data\Interface\`. Host log `%USERPROFILE%\Documents\My Games\Starfield\SFSE\Logs\PULSAR.log`. Never edit `extern/CommonLibSF` (submodule, never staged). Commit via `commit_msg.txt` + `git commit -F` + separate `Remove-Item`. No push.

---

### Task 1: PULSAR-local menu ID header + semantic verify

**Files:**
- Create: `src/Lobby/LobbyMenuIDs.h`
- Tool: `local RE notes, Ghidra project `the local Ghidra project/StarfieldRE.gpr`

- [ ] **Step 1: Disasm-verify that ID 80375 (0x1411A74E0) is really the menu registrar**

Run:
```bash
python "local RE notes" "<Starfield>\Starfield.exe" 11A74E0 160
```
Expected: a small function that takes a `BSFixedString` name + a creator and inserts into a hashmap (the `menuMap`). If it clearly is NOT a registrar, STOP and re-RE (search xrefs to a known menu name string instead). Record the confirmation.

- [ ] **Step 2: Create the local ID header**

Create `src/Lobby/LobbyMenuIDs.h`:
```cpp
#pragma once
#include "REL/Relocation.h"

// CommonLibSF ships these REL::IDs as {0} (unresolved). They ARE valid in the
// 1.16.236 versionlib (verified via read_versionlib.py + disasm). Defined here
// locally so we never edit the submodule.
namespace Pulsar::Lobby::EngineID
{
    inline constexpr REL::ID UI_RegisterMenu{ 80375 };   // 0x1411A74E0
    inline constexpr REL::ID IMenu_dtor{ 187216 };       // 0x1437CB520
    inline constexpr REL::ID IMenu_LoadMovie{ 187240 };  // 0x1437D1DD0
    inline constexpr REL::ID IMenu_ProcessMessage{ 187247 }; // 0x1437D1F90
}
```

- [ ] **Step 3: Commit**

```
git add src/Lobby/LobbyMenuIDs.h
```
Message: `feat(lobby/gui): PULSAR-local menu engine IDs (CLibSF ships them {0})`

---

### Task 2: .swf authoring spike — minimal movie + C++ callback

**Files:**
- Create: `lobby/interface/pulsar_lobby.swf` (+ source)
- Deploy: `<Starfield>\Data\Interface\pulsar_lobby.swf`

**Goal:** Prove we can author a Scaleform `.swf` that Starfield loads and that calls back into C++. This is the only remaining true unknown — do it before building the full UI.

- [ ] **Step 1: Pick the toolchain**

Use JPEXS Free Flash Decompiler (ffdec, open-source) — decompile an existing Bethesda HUD `.swf` (extract from `Data/Interface/*.ba2` via `<local>\extractor\bae.exe`) to confirm Starfield's ActionScript version (expect AS2/AS3 Scaleform). Record the AS version.

- [ ] **Step 2: Build a minimal movie**

Author `pulsar_lobby.swf` with: a `TextField` (status), one button. On button press, call the registered code function (Scaleform `ExternalInterface`/`fscommand`-style bridge that `SWFToCodeFunctionHandler` receives). Stage size 1280x720, transparent bg.

- [ ] **Step 3: Verify the menu loads + the callback fires (in-game, with Task 3/4 scaffolding)**

Deploy the `.swf`, register a stub menu (Task 3 minimal), open via hotkey, click the button → expect a `[LobbyMenu] callback: <name>` line in PULSAR.log. If the movie doesn't load, check `GetRootPath()` returns the correct `Interface/pulsar_lobby.swf` and the file is under `Data/Interface/`.

- [ ] **Step 4: Commit**

```
git add lobby/interface/pulsar_lobby.swf
```
Message: `feat(lobby/gui): minimal Scaleform lobby movie + C++ callback bridge`

---

### Task 3: `LobbyMenu` IMenu subclass

**Files:**
- Create: `src/Lobby/LobbyMenu.h`, `src/Lobby/LobbyMenu.cpp`

- [ ] **Step 1: Declare the menu**

`src/Lobby/LobbyMenu.h`: subclass `RE::IMenu`, `SF_MENU_NAME("LobbyMenu")`, implement the pure virtuals `GetName()` (return "LobbyMenu"), `GetRootPath()` (return "Interface/pulsar_lobby.swf"), `GetUnk05()`. Override `ProcessMessage` to handle `kShow`/`kHide`. Constructor: set flags (`ShowCursor`, pause flags), `LoadMovie(...)` via the local `IMenu_LoadMovie` relocation if the default thunk is `{0}`.

- [ ] **Step 2: Register Scaleform → C++ callbacks**

Implement `SWFToCodeFunctionHandler::Call` (or register handlers) for `OnHost`, `OnJoin(lobbyId)`, `OnRefresh`, `OnClose`. Each logs and (Task 6) writes `pulsar_cmd.json`. `OnClose` requests menu close via `UIMessageQueue::AddMessage(MENU_NAME, kHide)`.

- [ ] **Step 3: Push data into the movie**

Add `SetLobbies(const std::vector<LobbyRow>&)` that builds a `ScaleformGFxValue` array and invokes the movie's AS setter; and `SetStatus(string)`. Guard against `uiMovie == nullptr`.

- [ ] **Step 4: Build**

Run: `cmake --build build/vs --config RelWithDebInfo --target PULSAR`
Expected: compiles.

- [ ] **Step 5: Commit**

`git add src/Lobby/LobbyMenu.h src/Lobby/LobbyMenu.cpp` — Message: `feat(lobby/gui): LobbyMenu IMenu subclass (movie load + AS callbacks)`

---

### Task 4: Register the menu + hotkey to open it

**Files:**
- Modify: `src/Main.cpp`

- [ ] **Step 1: Register at SFSE data-loaded**

In the SFSE message handler (kDataLoaded / post-load), call `RE::UI::GetSingleton()->RegisterMenu<Pulsar::Lobby::LobbyMenu>("LobbyMenu")` guarded by `IsMenuRegistered`. If CLibSF's `RegisterMenu` thunk crashes (its `ID::UI::RegisterMenu` is `{0}`), call the engine function directly via a local relocation using `EngineID::UI_RegisterMenu`.

- [ ] **Step 2: Hotkey opens the menu**

In the existing input/tick path, on a chosen key (e.g. F9), `RE::UIMessageQueue::GetSingleton()->AddMessage("LobbyMenu", RE::UI_MESSAGE_TYPE::kShow, nullptr)`. Toggle with `kHide` if `UI::IsMenuOpen("LobbyMenu")`.

- [ ] **Step 3: Build + in-game smoke**

Build, deploy, launch: press F9 → menu opens (movie visible), F9 again → closes. Confirms registration + lifecycle. Commit: `feat(lobby/gui): register LobbyMenu + F9 hotkey`

---

### Task 5: Companion `serve` daemon

**Files:**
- Modify: `lobby/companion/Program.cs`

- [ ] **Step 1: Add the `serve` command**

Long-running (like `host`): connect, subscribe `SELECT * FROM lobby` + `my_profile`. On any lobby change, write `%LOCALAPPDATA%\PULSAR\pulsar_lobbies.json` = `[{lobbyId,name,status,hostAddr,memberCount,maxPlayers}]` (only `status=="open"`).

- [ ] **Step 2: Watch the command file**

In the FrameTick loop, poll `pulsar_cmd.json`; on appearance parse `{action,lobbyId?,name?,addr?,max?}`, run the reducer (`create_lobby`+keep-alive for host; `join_lobby`+pull-profile→`profile-in` for join), write `pulsar_ack.json` `{cmdId,ok,error?,hostAddr?}`, delete the cmd file.

- [ ] **Step 3: Verify headless**

Run `serve`, write a `pulsar_cmd.json` for create + for join → assert `lobbies.json` updates, `profile-in`/`ack` written, reducers logged. Commit: `feat(lobby/companion): serve daemon (lobby list + command watch)`

---

### Task 6: DLL ↔ files (render list, write commands)

**Files:**
- Modify: `src/Lobby/LobbyMenu.cpp`, `src/Lobby/LobbyHandoff.{h,cpp}`

- [ ] **Step 1: Read lobbies.json → movie**

On menu show + on a refresh tick, read `pulsar_lobbies.json`, parse rows, call `SetLobbies(...)`. If the file is missing/stale, `SetStatus("Companion offline")`.

- [ ] **Step 2: Write commands on click**

`OnHost`/`OnJoin` write `pulsar_cmd.json` with a fresh `cmdId`; show `SetStatus("…")`; poll `pulsar_ack.json` for the matching `cmdId` (timeout → status error). Commit: `feat(lobby/gui): render lobby list + emit host/join commands`

---

### Task 7: Wire join/host → ENet connect

**Files:**
- Modify: `src/Lobby/LobbyHandoff.{h,cpp}`, `src/Main.cpp`

- [ ] **Step 1: On join ack, connect**

When `pulsar_ack.json` reports `ok` for a join (with `hostAddr` + `profile-in` staged), apply profile via the existing `TickApplyProfile` gate and initiate the ENet client connect to `hostAddr` directly (DLL is live — no relaunch). Host path: set host role. Commit: `feat(lobby/gui): drive ENet connect from in-game join/host`

---

### Task 8: Full in-game test

- [ ] **Step 1:** Companion `serve` running. Host: F9 → Host → confirm lobby appears in a second client's `serve` list. Join from the second client: F9 → select → Join → profile applies, ENet connects. F10 → profile-out → companion push. Verify `[LobbyMenu]`/`[LobbyHandoff]` log lines. Record results in the spec / a memory.

---

## Notes
- Verify every locally-defined engine ID against Ghidra before trusting it (CLibSF offsets are stale for 1.16.236 — see [[reference_ghidra_workflow]]).
- ImGui is NOT an option (D3D12 second-frame hard crash — [[feedback_imgui_crash]]).
- Stage only listed paths; never `git add -A`; never stage `extern/CommonLibSF` or `scripts/PulsarFurnSync.pex`.
