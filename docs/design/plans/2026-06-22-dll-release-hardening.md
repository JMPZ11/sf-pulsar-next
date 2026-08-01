# PULSAR.dll Release Hardening (IP-theft protection)

> Applied to EVERY public release (the `pulsar-releases` GitHub repo). Goal: raise the
> cost of stealing the reverse-engineered netcode/sync/engine work. Full prevention of
> native-DLL reversing is impossible — this is cost-raising, Tier 0+1 (free, no AV risk).
> VMProtect/Themida (Tier 2) is a LATER, optional decision.

**Threat:** someone reverses the shipped DLL and lifts our code/RE work into their own project.
**Non-goals (now):** anti-cheat (belongs server-side in the lobby/STDB), anti-piracy, VMProtect.

## Iron rule: Dev build stays FULL, only Release is hardened
- **Dev (RelWithDebInfo):** full `REX::INFO` logging + PDB + symbols — unchanged, this is our crash-dump workflow (`dump_stack.py`, NAS logs). NEVER ship this.
- **Release-hardened:** no PDB shipped, symbols stripped, INFO logs compiled OUT, strings encrypted, minimal exports. This is what goes to players/GitHub.
- Switch = a compile define `PULSAR_HARDENED` (set only in the hardened Release config).

---

## Task 1 — Hardened Release build config (CMake / MSVC)
**Files:** `CMakeLists.txt` (+ a `cmake/Hardened.cmake` optional).
- Add a hardened configuration (reuse `Release`, or a custom `ReleaseHardened`) that:
  - `target_compile_definitions(PULSAR PRIVATE PULSAR_HARDENED=1)`
  - Optimisation on (`/O2 /Gy /Gw`), `/Zi` OFF; linker `/DEBUG:NONE` → **no PDB emitted**.
  - `/OPT:REF /OPT:ICF` (fold/strip unused).
  - No `/MAP`. Ensure no `.pdb` in the output dir for this config.
- **Acceptance:** `build/vs/ReleaseHardened/PULSAR.dll` exists, **no `PULSAR.pdb`** beside it; `dumpbin /headers` shows no debug dir (or only minimal). Keep the dev PDB private (never in the release zip).

## Task 2 — Minimal exports (.def)
**Files:** `PULSAR.def` (new), `CMakeLists.txt` (link `/DEF:` for the hardened config).
- SFSE/CommonLibSF plugins export a single version-data symbol — **CONFIRM the exact name** for our CommonLibSF (`dumpbin /exports build/vs/RelWithDebInfo/PULSAR.dll` — look for `SFSEPlugin_Version` / `SFSEPlugin_Load` / `SFSEPluginVersionData`). Export ONLY that in the `.def`.
- **Acceptance:** `dumpbin /exports` on the hardened DLL lists only the SFSE-required export(s); plugin still loads in-game (SFSE finds the version data).

## Task 3 — Compile-out INFO logging in hardened builds (kills the biggest leak)
Our `REX::INFO(...)` strings literally name functions/RVAs/behaviour and are the #1 reversing aid (our own `disasm_strings.py` lives off them). They must be COMPILED OUT (not just runtime-silenced) so the literals never reach the binary.
**Files:** new `include/Log.h`, then migrate call sites; PCH include.
- `Log.h`:
  ```cpp
  #ifdef PULSAR_HARDENED
  #  define PLOG_INFO(...) ((void)0)        // literal never emitted
  #  define PLOG_WARN(...) ((void)0)        // (or keep WARN minimal+encrypted if wanted)
  #else
  #  define PLOG_INFO(...) REX::INFO(__VA_ARGS__)
  #  define PLOG_WARN(...) REX::WARN(__VA_ARGS__)
  #endif
  ```
- Migration: replace `REX::INFO(` → `PLOG_INFO(` and `REX::WARN(` → `PLOG_WARN(`. Large mechanical change (hundreds of sites) — do it in a scripted sweep, build after.
  - **Priority order if doing incrementally:** first the most revealing logs (anything printing RVAs, `FUN_…`, function names, offsets, internal state) — e.g. CoopMortality, FlightSync, ShipCombatSync, NPCSync, Config. Benign ones later.
- Keep a few hard ERROR logs (encrypted strings, Task 4) so a crashing release still leaves a breadcrumb.
- **Acceptance:** `strings PULSAR.dll | grep -iE "FUN_|0x14|hull=0|RVA|ship destruction"` returns ~nothing on the hardened build; dev build unchanged.

## Task 4 — Compile-time string encryption (remaining literals)
**Files:** vendor a header-only MSVC-compatible obfuscator (`xorstr` / JustasMasiulis `xorstr`, or `adamyaxley/obfuscate` `AY_OBFUSCATE`), e.g. `extern/xorstr/xorstr.hpp`; wrap surviving sensitive literals.
- Wrap: the build tag, remaining error strings, any file paths / identifying constants, the watermark (Task 5). Usage `xorstr_("...")` → encrypted bytes in `.rdata`, decrypted at runtime.
- Only needed for strings that survive Task 3; keep it targeted (don't encrypt everything → bloat).
- **Acceptance:** the wrapped strings don't appear in `strings PULSAR.dll`.

## Task 5 — Watermark / canary (provenance)
**Files:** a small `src/Watermark.cpp` referenced so the linker keeps it.
- Embed a hidden, encrypted unique build/author marker (a GUID + "PULSAR © <author>"), documented PRIVATELY (not in the public repo). If our code surfaces elsewhere, the marker proves origin.
- **Acceptance:** marker present (decryptable by us) but not plaintext-visible.

## Task 6 — License / EULA + release packaging
**Files:** `LICENSE` (all-rights-reserved / no-reuse-no-derivative EULA — decide exact text), `tools/package_release.ps1` (new).
- `package_release.ps1`: builds the **hardened** config, asserts (no PDB, minimal exports via dumpbin, strings-grep clean), then zips **DLL + .pex scripts + PULSAR.esp + LICENSE + README/install note** for the `pulsar-releases` upload. NEVER include the PDB or any `_dist`/source.
- This script = the "every release is secured this way" guarantee the user asked for.
- **Acceptance:** running it produces a clean release zip; manual spot-check of contents.

---

## Order
1 (build config) → 2 (.def) → 3 (log compile-out, the big win) → 4 (string crypto) → 5 (watermark) → 6 (license + packaging). Each builds + the hardened DLL must still **load + run in-game on both machines** (user test) before it's the release path.

## Later (Tier 2, separate decision)
VMProtect/Themida selective virtualization of crown-jewel functions (netcode/sync/RE wrappers). Caveats: paid tool, AV/anti-cheat false-positives (packed self-injecting DLL looks like a cheat), SFSE-load + perf risk → heavy 2-machine testing. Only if a real theft threat materialises.

## Notes
- Source stays PRIVATE (main repo local, never pushed) — the only attack surface is the shipped binary; Tier 0+1 hardens exactly that.
- Don't over-invest client-side: cheating is a SERVER-authoritative problem (lobby/STDB), not a DLL-protection problem.
