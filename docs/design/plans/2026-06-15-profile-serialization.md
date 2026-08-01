# Profil-Serialisierung (Teil 1) Implementation Plan


**Goal:** Build a storage-agnostic profile codec — `SerializeProfile() → blob` and `ApplyProfile(blob)` — that captures the player's inventory (incl. mods), equip-state, credits, and O2 into an opaque versioned JSON blob and applies it back, cross-machine-stable via T34 FormID encoding.

**Architecture:** Extend the existing `Pulsar::Profile` module (`src/Profile/ClientProfile.{cpp,h}`) **additively** (old fields/paths keep compiling). Reuse four battle-tested in-repo pieces: `PluginLookup` (T34 FormID↔plugin), `Loot::ReadItemMods`/`ApplyItemMods` (mod arrays), `DirectRestore` apply patterns (`AddObjectToContainer` diff, Oxygen AV get/set), and the existing JSON writer/reader. No new module, no rewrite of the live connect-send path.

**Tech Stack:** C++23, CommonLibSF, SFSE plugin (in-process Starfield 1.16.236 DLL). Logging via `REX::INFO`/`REX::WARN`. Build: `cmake --build build/vs --config RelWithDebInfo --target PULSAR`.

**Testing reality:** PULSAR has **no unit-test runner** (it's an in-process game DLL). The "test" idiom is: (a) a **load-time self-check** for pure-logic code (encode/decode round-trip on a synthetic struct, logs `PASS`/`FAIL`), and (b) a **`GetAsyncKeyState` hotkey self-test** for engine-touching code (capture→encode→decode→apply round-trip in-game, logs a diff). Both follow existing patterns: load-time logging in `src/Main.cpp`, hotkey polling at `src/Main.cpp:1130/1236/1347`.

**Reference reading before starting:**
- `include/Profile/ClientProfile.h` + `src/Profile/ClientProfile.cpp` — current schema, JSON writer/reader, `SnapshotFromPlayer`.
- `include/NPCSync/PluginLookup.h` — T34 encode/decode signatures.
- `include/WorldSync/LootSync.h:61-64` — `ReadItemMods`/`ApplyItemMods` signatures.
- `src/InventorySync/DirectRestore.cpp` — the **apply patterns to copy verbatim**: `AddObjectToContainer` diff (~671-736), Oxygen AV get/set (~516-526, 736-751), credits note (form `0x0000000F`, ~678).
- `docs/design/specs/2026-06-15-profile-serialization-design.md` — the spec this plan implements.

**v1 scope reminders (from spec):** inventory(+mods)+equip+credits+O2 only. Apply is **additive convergence** (add missing items up to count; set credits/O2 exact; re-equip; apply mods) — does NOT remove non-profile items in v1 (the live host-item-stripping stays DirectRestore's job). Mod arrays are stored **raw `uint32`** (correct for same-machine persistence; cross-machine mod-form encoding deferred pending RE of the mod-array entry semantics — the item form itself IS T34-encoded). Known **** limit: mod-slot clamp can grey epics with more mods than locally allocated.

---

## File Structure

- **Modify** `include/Profile/ClientProfile.h` — extend `ProfileItem` + `ProfileData`; declare new functions.
- **Modify** `src/Profile/ClientProfile.cpp` — v2 JSON writer/reader; `CaptureProfile`, `ApplyProfileData`, `EncodeProfile`, `DecodeProfile`, `SerializeProfile`, `ApplyProfile`, `RunSelfTest`.
- **Modify** `src/Main.cpp` — call `RunSelfTest()` once at load; wire an `F8`-style hotkey for the in-game capture→apply round-trip self-test.

No new files (follows the additive, single-module-responsibility pattern).

---

## Task 1: Extend the data model (additive, non-breaking)

**Files:**
- Modify: `include/Profile/ClientProfile.h`

- [ ] **Step 1: Extend `ProfileItem` and `ProfileData`, declare new functions**

Replace the contents of `include/Profile/ClientProfile.h` with:

```cpp
#pragma once

#include <cstdint>
#include <span>
#include <string>
#include <vector>

namespace Pulsar::Profile {

    struct ProfileItem {
        // --- existing fields (kept; used by the live connect-send path) ---
        uint32_t formID = 0;     // current-machine FormID (resolved id)
        int32_t  count  = 0;

        // --- v2 additions ---
        std::string plugin;                 // T34: owning plugin filename ("" if base/unresolved)
        uint32_t    localID  = 0;           // T34: plugin-local id
        bool        equipped = false;       // equip-state captured inline
        std::vector<uint32_t> mods;         // M.2 mod array (raw, same-machine)
        uint16_t    modSentinel = 0;        // M.2 mod sentinel
    };

    struct ProfileData {
        // --- existing fields (kept) ---
        std::string playerName;
        uint32_t    playerLevel  = 0;
        std::string raceName;
        uint16_t    gender       = 0;
        uint32_t    weaponFormID = 0;       // legacy; still written by old path
        std::vector<uint32_t> armorFormIDs; // legacy; still written by old path
        std::vector<ProfileItem> inventory;

        // --- v2 additions ---
        uint32_t version = 2;
        uint32_t credits = 0;               // form 0x0000000F count
        float    o2      = 0.0f;            // Oxygen ActorValue
    };

    // ---- existing API (unchanged behaviour) ----
    bool Save(const ProfileData& a_data);
    bool Load(const std::string& a_playerName, ProfileData& a_out);
    ProfileData SnapshotFromPlayer();   // alias to CaptureProfile after Task 3

    // ---- v2 codec API (the contract Teil 2 consumes) ----
    ProfileData CaptureProfile();                                   // Task 3
    std::vector<uint8_t> EncodeProfile(const ProfileData&);         // Task 2
    bool DecodeProfile(std::span<const uint8_t>, ProfileData& out); // Task 2
    bool ApplyProfileData(const ProfileData&);                      // Task 4
    std::vector<uint8_t> SerializeProfile();                        // Task 5 = Encode(Capture())
    bool ApplyProfile(std::span<const uint8_t>);                    // Task 5 = ApplyProfileData(Decode())

    // ---- dev self-test (load-time codec check + hotkey-driven apply check) ----
    bool RunSelfTest();                  // Task 2: synthetic Encode→Decode round-trip, logs PASS/FAIL
    void RunInGameRoundTripTest();       // Task 5: Capture→Encode→Decode→Apply in-game, logs diff
}
```

- [ ] **Step 2: Build to verify the header compiles (no impl yet — linker will fail later, that's fine)**

Run: `cmake --build build/vs --config RelWithDebInfo --target PULSAR`
Expected: **compiles**; may fail at **link** with unresolved `EncodeProfile`/`DecodeProfile`/`CaptureProfile`/`ApplyProfileData`/`SerializeProfile`/`ApplyProfile`/`RunSelfTest`/`RunInGameRoundTripTest`. Compile errors in `ClientProfile.h` consumers (e.g. `Main.cpp`) = a real problem to fix now. Unresolved-symbol link errors = expected, ignore until Task 5.

- [ ] **Step 3: Commit**

```
git add include/Profile/ClientProfile.h
git commit -F commit_msg.txt   # message: "feat(profile): v2 data model — encoded forms, equip, mods, credits, o2"
```
(Use the repo convention: write the message into `commit_msg.txt`, commit with `-F`, then `Remove-Item commit_msg.txt` as a **separate** command. Never `git add -A`.)

---

## Task 2: JSON v2 codec + load-time self-check

**Files:**
- Modify: `src/Profile/ClientProfile.cpp`
- Modify: `src/Main.cpp` (call `RunSelfTest()` at load)

The blob is the **bytes of a JSON v2 document**. Mods are serialized as a space-separated decimal string so the existing hand-rolled per-object reader (`ExtractString`/`ExtractUInt` over each `{...}`) can parse them without a nested-array parser.

- [ ] **Step 1: Write the failing self-check first**

At the bottom of `src/Profile/ClientProfile.cpp` (inside `namespace Pulsar::Profile`, before the closing brace), add:

```cpp
    bool RunSelfTest()
    {
        // Build a synthetic profile with every v2 field populated.
        ProfileData a;
        a.version    = 2;
        a.playerName = "SelfTest";
        a.credits    = 12345;
        a.o2         = 87.5f;
        ProfileItem it;
        it.formID      = 0x0001A2B3;
        it.count       = 3;
        it.plugin      = "Starfield.esm";
        it.localID     = 0x1A2B3;
        it.equipped    = true;
        it.mods        = { 111u, 222u, 333u };
        it.modSentinel = 7;
        a.inventory.push_back(it);

        std::vector<uint8_t> blob = EncodeProfile(a);
        ProfileData b;
        const bool decoded = DecodeProfile(blob, b);

        const bool ok =
            decoded &&
            b.version == a.version &&
            b.playerName == a.playerName &&
            b.credits == a.credits &&
            (b.o2 > a.o2 - 0.01f && b.o2 < a.o2 + 0.01f) &&
            b.inventory.size() == 1 &&
            b.inventory[0].count == it.count &&
            b.inventory[0].plugin == it.plugin &&
            b.inventory[0].localID == it.localID &&
            b.inventory[0].equipped == it.equipped &&
            b.inventory[0].mods == it.mods &&
            b.inventory[0].modSentinel == it.modSentinel;

        if (ok) {
            REX::INFO("[PROFILE-TEST] codec round-trip PASS"sv);
        } else {
            REX::WARN("[PROFILE-TEST] codec round-trip FAIL (decoded={}, items={})"sv,
                      decoded, b.inventory.size());
        }
        return ok;
    }
```

Also add temporary empty stubs so it links and the test can FAIL meaningfully:

```cpp
    std::vector<uint8_t> EncodeProfile(const ProfileData&) { return {}; }
    bool DecodeProfile(std::span<const uint8_t>, ProfileData&) { return false; }
```

- [ ] **Step 2: Wire the self-check to run at load, build, run, verify it FAILS**

In `src/Main.cpp`, find the existing startup banner `REX::INFO("[PULSAR]` log in the load/init path and add immediately after it:

```cpp
        Pulsar::Profile::RunSelfTest();
```
(`#include "Profile/ClientProfile.h"` is already present at `src/Main.cpp:42`.)

Run: `cmake --build build/vs --config RelWithDebInfo --target PULSAR`, deploy, launch.
Expected: log shows `[PROFILE-TEST] codec round-trip FAIL` (stubs return empty/false).

- [ ] **Step 3: Implement `EncodeProfile` (replace the stub)**

```cpp
    static void AppendModsCSV(std::string& out, const std::vector<uint32_t>& mods)
    {
        for (size_t i = 0; i < mods.size(); ++i) {
            if (i) out += ' ';
            out += std::to_string(mods[i]);
        }
    }

    std::vector<uint8_t> EncodeProfile(const ProfileData& a_data)
    {
        std::string j;
        j += "{\n";
        j += "  \"version\": " + std::to_string(a_data.version) + ",\n";
        j += "  \"playerName\": \"" + EscapeJSON(a_data.playerName) + "\",\n";
        j += "  \"playerLevel\": " + std::to_string(a_data.playerLevel) + ",\n";
        j += "  \"raceName\": \"" + EscapeJSON(a_data.raceName) + "\",\n";
        j += "  \"gender\": " + std::to_string(a_data.gender) + ",\n";
        j += "  \"credits\": " + std::to_string(a_data.credits) + ",\n";
        j += "  \"o2\": " + std::to_string(a_data.o2) + ",\n";
        j += "  \"inventory\": [\n";
        for (size_t i = 0; i < a_data.inventory.size(); ++i) {
            const auto& it = a_data.inventory[i];
            std::string modsCsv;
            AppendModsCSV(modsCsv, it.mods);
            j += "    {\"formID\": " + std::to_string(it.formID);
            j += ", \"count\": " + std::to_string(it.count);
            j += ", \"plugin\": \"" + EscapeJSON(it.plugin) + "\"";
            j += ", \"localID\": " + std::to_string(it.localID);
            j += ", \"equipped\": " + std::string(it.equipped ? "1" : "0");
            j += ", \"modSentinel\": " + std::to_string(it.modSentinel);
            j += ", \"mods\": \"" + modsCsv + "\"}";
            if (i + 1 < a_data.inventory.size()) j += ",";
            j += "\n";
        }
        j += "  ]\n}\n";
        return std::vector<uint8_t>(j.begin(), j.end());
    }
```

- [ ] **Step 4: Implement `DecodeProfile` (replace the stub)**

```cpp
    static std::vector<uint32_t> ParseModsCSV(const std::string& csv)
    {
        std::vector<uint32_t> out;
        size_t p = 0;
        while (p < csv.size()) {
            while (p < csv.size() && (csv[p] == ' ' || csv[p] == ',')) ++p;
            if (p >= csv.size()) break;
            out.push_back(static_cast<uint32_t>(std::strtoul(csv.c_str() + p, nullptr, 10)));
            while (p < csv.size() && csv[p] != ' ' && csv[p] != ',') ++p;
        }
        return out;
    }

    bool DecodeProfile(std::span<const uint8_t> a_blob, ProfileData& a_out)
    {
        if (a_blob.empty()) return false;
        std::string json(reinterpret_cast<const char*>(a_blob.data()), a_blob.size());

        a_out = ProfileData{};
        a_out.version    = ExtractUInt(json, "version");
        a_out.playerName = ExtractString(json, "playerName");
        a_out.playerLevel= ExtractUInt(json, "playerLevel");
        a_out.raceName   = ExtractString(json, "raceName");
        a_out.gender     = static_cast<uint16_t>(ExtractUInt(json, "gender"));
        a_out.credits    = ExtractUInt(json, "credits");
        {
            auto pos = json.find("\"o2\"");
            if (pos != std::string::npos) {
                pos = json.find(':', pos);
                if (pos != std::string::npos) a_out.o2 = std::strtof(json.c_str() + pos + 1, nullptr);
            }
        }

        // Parse the inventory array of {...} objects (reuse the existing per-object scan).
        auto invPos = json.find("\"inventory\"");
        if (invPos != std::string::npos) {
            auto arrStart = json.find('[', invPos);
            auto arrEnd   = json.rfind(']');
            if (arrStart != std::string::npos && arrEnd != std::string::npos && arrEnd > arrStart) {
                std::string arr = json.substr(arrStart + 1, arrEnd - arrStart - 1);
                size_t p = 0;
                while (p < arr.size()) {
                    auto objStart = arr.find('{', p);
                    if (objStart == std::string::npos) break;
                    auto objEnd = arr.find('}', objStart);
                    if (objEnd == std::string::npos) break;
                    std::string obj = arr.substr(objStart, objEnd - objStart + 1);
                    ProfileItem it;
                    it.formID      = ExtractUInt(obj, "formID");
                    it.count       = ExtractInt(obj, "count");
                    it.plugin      = ExtractString(obj, "plugin");
                    it.localID     = ExtractUInt(obj, "localID");
                    it.equipped    = ExtractUInt(obj, "equipped") != 0;
                    it.modSentinel = static_cast<uint16_t>(ExtractUInt(obj, "modSentinel"));
                    it.mods        = ParseModsCSV(ExtractString(obj, "mods"));
                    if (it.count != 0) a_out.inventory.push_back(it);
                    p = objEnd + 1;
                }
            }
        }
        return true;
    }
```

Note: `EscapeJSON`, `ExtractString`, `ExtractUInt`, `ExtractInt` already exist in this file (lines ~22-131). `<string>` and the `sv` literal (`using namespace std::string_view_literals;`) are already in scope at the top of the file.

- [ ] **Step 5: Build, run, verify the self-check now PASSES**

Run: `cmake --build build/vs --config RelWithDebInfo --target PULSAR`, deploy, launch.
Expected: log shows `[PROFILE-TEST] codec round-trip PASS`.

- [ ] **Step 6: Commit**

```
git add src/Profile/ClientProfile.cpp src/Main.cpp
git commit -F commit_msg.txt   # "feat(profile): JSON v2 codec + load-time round-trip self-check"
```

---

## Task 3: `CaptureProfile` — full capture (T34, equip, mods, credits, O2)

**Files:**
- Modify: `src/Profile/ClientProfile.cpp`

- [ ] **Step 1: Add includes for the reuse primitives**

At the top of `src/Profile/ClientProfile.cpp`, after the existing includes, add:

```cpp
#include "NPCSync/PluginLookup.h"
#include "WorldSync/LootSync.h"
```

- [ ] **Step 2: Implement `CaptureProfile`**

Add inside the namespace:

```cpp
    static constexpr uint32_t kCreditsFormID = 0x0000000F;  // see DirectRestore.cpp:678

    ProfileData CaptureProfile()
    {
        ProfileData data;
        data.version = 2;

        auto* player = RE::PlayerCharacter::GetSingleton();
        if (!player) {
            REX::WARN("[PROFILE] CaptureProfile: no player"sv);
            return data;
        }

        // Build the set of equipped object FormIDs first.
        std::vector<uint32_t> equippedForms;
        player->ForEachEquippedItem([&](const RE::BGSInventoryItem& item) -> RE::BSContainer::ForEachResult {
            if (item.object) equippedForms.push_back(item.object->GetFormID());
            return RE::BSContainer::ForEachResult::kContinue;
        });
        auto isEquipped = [&](uint32_t fid) {
            for (uint32_t e : equippedForms) if (e == fid) return true;
            return false;
        };

        player->ForEachInventoryItem([&](const RE::BGSInventoryItem& a_item) -> RE::BSContainer::ForEachResult {
            if (!a_item.object) return RE::BSContainer::ForEachResult::kContinue;
            const uint32_t fid = a_item.object->GetFormID();
            int32_t total = 0;
            for (auto& stack : a_item.stacks) total += static_cast<int32_t>(stack.count);
            if (total <= 0) return RE::BSContainer::ForEachResult::kContinue;

            // Credits go into their own field, not the item list.
            if (fid == kCreditsFormID) {
                data.credits = static_cast<uint32_t>(total);
                return RE::BSContainer::ForEachResult::kContinue;
            }

            ProfileItem it;
            it.formID   = fid;
            it.count    = total;
            it.equipped = isEquipped(fid);

            // T34-encode the item form so it resolves on another machine / load order.
            char pluginBuf[260] = {};
            if (Pulsar::NPCSync::PluginLookup::FormIDToPluginName(fid, pluginBuf, sizeof(pluginBuf))) {
                it.plugin  = pluginBuf;
                it.localID = Pulsar::NPCSync::PluginLookup::FormIDToLocalID(fid);
            }

            // Mods (raw uint32 array + sentinel) via the M.2 primitive.
            Pulsar::WorldSync::Loot::ReadItemMods(player, fid, it.mods, it.modSentinel);

            data.inventory.push_back(std::move(it));
            return RE::BSContainer::ForEachResult::kContinue;
        });

        // O2: copy the EXACT Oxygen-AV read used in DirectRestore.cpp (~line 516/586/1001):
        //   look up the "Oxygen" ActorValueInfo, then player->GetActorValue(*av).
        // Use that identical lookup+read here and assign to data.o2.
        // (Left as a direct copy of the working pattern to avoid a wrong AV-lookup API.)

        REX::INFO("[PROFILE] Captured: {} items, credits={}, o2={}, ({} equipped)"sv,
                  data.inventory.size(), data.credits, data.o2, equippedForms.size());
        return data;
    }
```

For the O2 read in the marked spot, open `src/InventorySync/DirectRestore.cpp` around line 516/1001 and copy the identical "Oxygen" `ActorValueInfo` lookup + `player->GetActorValue(*av)` call, assigning the result to `data.o2`. Do not invent an AV-lookup API — reuse the one that already works in that file.

- [ ] **Step 3: Point `SnapshotFromPlayer` at `CaptureProfile`**

Replace the existing `SnapshotFromPlayer()` body (lines ~201-239) with:

```cpp
    ProfileData SnapshotFromPlayer()
    {
        return CaptureProfile();
    }
```

This keeps the existing connect-send caller (`Main.cpp:774`, `Main.cpp:1530`) working — it still reads `weaponFormID`/`armorFormIDs`/`inventory[].formID`, all still populated (weapon/armor are now in `inventory` with `equipped=true`; if the old path specifically needs `weaponFormID`/`armorFormIDs`, also set them at the end of `CaptureProfile` from the equipped set — add that only if a build/runtime check shows the old path regressing).

- [ ] **Step 4: Build, deploy, verify capture logs**

Run: `cmake --build build/vs --config RelWithDebInfo --target PULSAR`, deploy, launch, load a save.
Expected: on the next `SnapshotFromPlayer` call (connect or save hotkey at `Main.cpp:1530`), log shows `[PROFILE] Captured: N items, credits=…, o2=…` with plausible non-zero values.

- [ ] **Step 5: Commit**

```
git add src/Profile/ClientProfile.cpp
git commit -F commit_msg.txt   # "feat(profile): CaptureProfile — T34 forms, equip, mods, credits, O2"
```

---

## Task 4: `ApplyProfileData` — additive convergence + mods + equip + credits + O2

**Files:**
- Modify: `src/Profile/ClientProfile.cpp`

Apply is **additive** in v1: for each profile item, resolve its FormID and add the missing count; set credits exactly; set O2; re-equip equipped items; apply mods. It does **not** remove non-profile items.

- [ ] **Step 1: Implement `ApplyProfileData`**

```cpp
    // Resolve a profile item to a current-machine FormID: prefer T34 (plugin+localID),
    // fall back to the stored raw formID (correct on the same machine).
    static uint32_t ResolveItemFormID(const ProfileItem& it)
    {
        if (!it.plugin.empty()) {
            uint32_t fid = Pulsar::NPCSync::PluginLookup::ResolveCrossPlugin(it.plugin.c_str(), it.localID);
            if (fid != 0) return fid;
        }
        return it.formID;  // same-machine fallback
    }

    static int32_t CurrentCount(RE::PlayerCharacter* player, uint32_t formID)
    {
        int32_t total = 0;
        player->ForEachInventoryItem([&](const RE::BGSInventoryItem& a_item) -> RE::BSContainer::ForEachResult {
            if (a_item.object && a_item.object->GetFormID() == formID) {
                for (auto& s : a_item.stacks) total += static_cast<int32_t>(s.count);
                return RE::BSContainer::ForEachResult::kStop;
            }
            return RE::BSContainer::ForEachResult::kContinue;
        });
        return total;
    }

    bool ApplyProfileData(const ProfileData& a_data)
    {
        auto* player = RE::PlayerCharacter::GetSingleton();
        if (!player) {
            REX::WARN("[PROFILE] ApplyProfileData: no player"sv);
            return false;
        }

        // Suppress M.2 inventory broadcast while we mutate (same guard DirectRestore uses).
        const bool prevReplay = Pulsar::WorldSync::Loot::SetReplayingFromPacket(true);

        int added = 0, modsApplied = 0, equipped = 0, skipped = 0;
        for (const auto& it : a_data.inventory) {
            const uint32_t fid = ResolveItemFormID(it);
            if (fid == 0) { ++skipped;
                REX::WARN("[PROFILE] apply: unresolved form plugin={} local=0x{:X}"sv, it.plugin, it.localID);
                continue;
            }
            const int32_t have = CurrentCount(player, fid);
            const int32_t need = it.count - have;
            if (need > 0) {
                // Copy the EXACT AddObjectToContainer call used in DirectRestore.cpp (~671-736):
                //   LookupByID(fid) -> TESBoundObject* -> player->AddObjectToContainer(obj, nullptr, need, nullptr)
                // Reuse that working call here with count = need.
                ++added;
            }
            if (!it.mods.empty()) {
                Pulsar::WorldSync::Loot::ApplyItemMods(player, fid, it.mods, it.modSentinel);
                ++modsApplied;
            }
            if (it.equipped) {
                // Re-equip: reuse the equip helper pattern already used elsewhere (e.g. EquipSync.cpp
                // / the engine ActorEquipManager::EquipObject path). Equip the resolved bound object.
                ++equipped;
            }
        }

        // Credits: converge form 0x0000000F to data.credits exactly (delta add/remove via
        // the same AddObjectToContainer call; negative count removes, per DirectRestore header).
        {
            const int32_t have = CurrentCount(player, kCreditsFormID);
            const int32_t delta = static_cast<int32_t>(a_data.credits) - have;
            if (delta != 0) {
                // AddObjectToContainer(LookupByID(0x0000000F) as TESBoundObject*, nullptr, delta, nullptr)
            }
        }

        // O2: copy the EXACT Oxygen-AV set used in DirectRestore.cpp (~736-751):
        //   look up "Oxygen" ActorValueInfo, player->SetActorValue(*av, a_data.o2).

        Pulsar::WorldSync::Loot::SetReplayingFromPacket(prevReplay);
        REX::INFO("[PROFILE] Applied: +{} items, {} mods, {} equipped, {} skipped"sv,
                  added, modsApplied, equipped, skipped);
        return skipped == 0;
    }
```

For the three marked engine calls (`AddObjectToContainer` for items, credits delta, and `SetActorValue("Oxygen")`), copy the identical working calls from `src/InventorySync/DirectRestore.cpp` at the cited line ranges — including how it resolves `RE::TESForm::LookupByID(fid)` to a `TESBoundObject*`. For the re-equip, copy the equip-object call already used in `src/RemotePlayer/EquipSync.cpp`. This keeps every engine ABI call identical to code proven on 1.16.236.

- [ ] **Step 2: Build to verify it compiles and links**

Run: `cmake --build build/vs --config RelWithDebInfo --target PULSAR`
Expected: compiles + links clean (the marked calls are now filled with the copied working code).

- [ ] **Step 3: Commit**

```
git add src/Profile/ClientProfile.cpp
git commit -F commit_msg.txt   # "feat(profile): ApplyProfileData — additive converge, mods, equip, credits, O2"
```

---

## Task 5: Convenience wrappers + in-game round-trip self-test

**Files:**
- Modify: `src/Profile/ClientProfile.cpp`
- Modify: `src/Main.cpp` (hotkey wiring)

- [ ] **Step 1: Implement the wrappers and the in-game self-test**

```cpp
    std::vector<uint8_t> SerializeProfile()
    {
        return EncodeProfile(CaptureProfile());
    }

    bool ApplyProfile(std::span<const uint8_t> a_blob)
    {
        ProfileData d;
        if (!DecodeProfile(a_blob, d)) {
            REX::WARN("[PROFILE] ApplyProfile: decode failed"sv);
            return false;
        }
        return ApplyProfileData(d);
    }

    void RunInGameRoundTripTest()
    {
        ProfileData before = CaptureProfile();
        std::vector<uint8_t> blob = EncodeProfile(before);
        ProfileData after;
        const bool decoded = DecodeProfile(blob, after);
        const bool match =
            decoded &&
            after.inventory.size() == before.inventory.size() &&
            after.credits == before.credits;
        REX::INFO("[PROFILE-TEST] in-game round-trip {}: items {}->{}, credits {}->{}, blob={} bytes"sv,
                  match ? "PASS" : "FAIL",
                  before.inventory.size(), after.inventory.size(),
                  before.credits, after.credits, blob.size());
        // Apply the decoded blob back onto the live player (idempotent: additive converge to same counts).
        ApplyProfileData(after);
    }
```

- [ ] **Step 2: Wire a hotkey for the in-game test in `src/Main.cpp`**

Follow the existing `GetAsyncKeyState` edge-detect pattern (see `src/Main.cpp:1130`). In the same per-frame tick block, add (using `VK_F10` as a dev-only test key):

```cpp
                {
                    static bool s_prevProfileTest = false;
                    const bool now = (GetAsyncKeyState(VK_F10) & 0x8000) != 0;
                    if (now && !s_prevProfileTest) {
                        Pulsar::Profile::RunInGameRoundTripTest();
                    }
                    s_prevProfileTest = now;
                }
```

- [ ] **Step 3: Build, deploy, run the in-game test**

Run: `cmake --build build/vs --config RelWithDebInfo --target PULSAR`, deploy to **both** host (`<Starfield>\Data\SFSE\Plugins\PULSAR.dll`) and NAS (`<NAS>/deploy/Data/SFSE/Plugins/PULSAR.dll`), launch, load a save, press **F10**.
Expected log: `[PROFILE-TEST] in-game round-trip PASS: items N->N, credits C->C, blob=… bytes` followed by `[PROFILE] Applied: …`. Verify in-game that inventory/credits/O2 are unchanged (idempotent additive apply) and equipped gear stays equipped.

- [ ] **Step 4: Manual restore test (the real goal)**

In-game: note credits, drop/sell a few items, press **F10**. Expected: the captured-then-applied profile re-adds the missing items (additive), credits/O2 restored to the captured values, mods preserved on epics (subject to). Confirm via `[PROFILE] Applied: +K items…` and visual inventory.

- [ ] **Step 5: Commit**

```
git add src/Profile/ClientProfile.cpp src/Main.cpp
git commit -F commit_msg.txt   # "feat(profile): Serialize/ApplyProfile wrappers + F10 in-game round-trip self-test"
```

---

## Done criteria

- `[PROFILE-TEST] codec round-trip PASS` at load.
- `[PROFILE-TEST] in-game round-trip PASS` on F10, with `ApplyProfileData` idempotent.
- Manual restore (drop items → F10) re-adds items with mods, restores credits + O2, keeps equip-state.
- Existing connect-send path (`Main.cpp` `ClientProfileHeader`) still compiles and runs unchanged.
- `SerializeProfile`/`ApplyProfile` exist as the blob contract for Teil 2.

## Known limitations carried forward (documented, not bugs)

- **Additive apply** (no removal of non-profile items) — full mirroring deferred; live host-item stripping stays DirectRestore's job.
- **Mod arrays stored raw** — correct same-machine; cross-machine mod-form portability deferred pending RE of mod-array entry semantics (item form itself IS T34-encoded).
- **mod-slot clamp** — epics with more mods than locally allocated can grey; needs the array-realloc fix.
- **Duplicate-form mods ambiguity** — `ReadItemMods`/`ApplyItemMods` key by FormID, so two stacks of the same form with different mods can't be distinguished; acceptable for v1 (modded items are typically unique weapons/armor).
- **Appearance / perks / skills / other AVs** — out of v1 scope per spec.
```
