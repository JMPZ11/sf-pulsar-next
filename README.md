# PULSAR

Co-op multiplayer for Starfield. Play through the game together with a friend.

> ## Looking for someone to take this over
>
> **PULSAR is no longer being developed.** I ran out of time to continue, so
> rather than let it go quiet and disappear, I have published everything.
>
> The full source is in this repository under GPL-3.0: the SFSE plugin, the
> tool, the companion daemon, the relay and the lobby backend. Nothing is held
> back and there is no gatekeeping. Fork it, publish it, rename it, take it in a
> different direction. You do not need to ask me.
>
> It works, and real people have played it. It also has bugs that break co-op in
> specific situations, and nobody is fixing them right now. Both of those are
> true, and [docs/OVERVIEW.md](docs/OVERVIEW.md) tells you which is which before
> you invest any time.
>
> If you want to pick it up, start at [BUILD.md](BUILD.md) and
> [docs/design/](docs/design/).

## Status for players

You can still download and play it. The release on this page works and the
lobby is up.

What changed is that **bug reports no longer lead to fixes from me.** Filing an
issue is still worth doing, because it leaves a record for whoever continues the
project, but please do not wait on a reply. Most known problems are already
listed further down and in [docs/OVERVIEW.md](docs/OVERVIEW.md).

## What you need

* **Starfield, the Steam version.** The mod loads through SFSE, which only supports Steam. The Game Pass and Microsoft Store version is not supported, and the mod will never run on consoles.
* **Starfield 1.16.244.** The plugin resolves engine functions by address, and those addresses are tied to the exact game version. A game update breaks the mod until someone re-resolves them.
* **SFSE (Starfield Script Extender)**: https://sfse.silverlock.org/
* Windows. A friend with the same setup.

## Install

The mod itself is one click. It is bundled inside the tool, so you never download mod files separately and you never copy anything by hand. The only thing you set up yourself is SFSE.

1. Install **SFSE**: drop its files into your Starfield folder, next to the game exe.
2. Start **PULSAR** (Pulsar.exe). If you do not have it yet, grab it from the [Releases](https://github.com/ramisotti13-eng/pulsar-tool/releases) on this page, unzip it anywhere except the Starfield folder, and run it.
3. Open the **Setup** page and click **Install mod**. That one click copies the whole mod into Starfield. Then click **Fix all** so the required game settings are set.

Close Starfield before installing or updating.

## Play together

Both players run the game through the tool, and **both have to be in game with a save loaded** (not sitting in the main menu) for the connection to work.

1. On the **Co-op** page, type your name.
2. One of you clicks **Host** and shares the code that appears. The other clicks **Join** and enters that code. Starfield launches for both of you.
3. Each of you loads a savegame. A fresh save, right after character creation, is the smoothest starting point. Use the same one when you join a lobby.
4. Once the connection is made, the host's current game is sent over to the joiner and loaded, so you both end up in the same place in the same world.

**The save sync in step 4 can take a while, so please be patient.** A Starfield save is several megabytes. It has to transfer to the joiner and then load, and during that time it can look like nothing is happening for a few seconds. That is expected. It is not frozen and it is not a crash, just let it finish.

Even though the joiner loads the host's game, your own character is restored right after, so both of you keep your own level, skills, perks and inventory. You can reset your character any time in the tool under Settings.

Tip: press **F8** in game to regroup at the host after a fast travel, or any time you drift apart.

For a private game without the lobby, use **Play directly**: the host shares an IP and port, and the other joins by IP (over the internet you also forward that port on your router).

## Good to know

A few things that look like problems but usually are not:

* **A character vanishing on a planet is normally not a disconnect.** It is a cell transition: the engine has to reload that character on the other side. Give it a few seconds, or move closer, and it comes back.
* **Lost each other or out of sync? Press F8.** That pulls the client back to the host. It is the manual regroup and it fixes most "where did they go" moments.
* **On a planet surface each of you can fast travel on your own.** The host fast traveling does not drag the client along on purpose. Use F8 afterwards to regroup at the host.
* **The other player is shown as a copy of their character.** A short flicker of that copy when changing cells is cosmetic and known. It does not mean something broke.
* **The first time someone enters a vehicle, the game shows a short tutorial popup.** That popup is local to whoever entered, so the other player will not see it. That is normal.
* **If the tool shows offline, use Reconnect** on the Co-op page. A relay can hiccup for a moment.

## Other mods

PULSAR was never tested alongside other mods. It is technically possible to run
them together, so you are welcome to try, but expect that some combinations will
not work. If something breaks, try again with only PULSAR active before you
conclude it is a PULSAR problem.

## What works

* **2 player co-op.** More is not supported: the engine does not run headless and the networking is built around a single peer. Four players might be reachable with real work, a large server is not. The reasoning is in [docs/OVERVIEW.md](docs/OVERVIEW.md).
* **Host and join by a short code**, or connect directly by IP for LAN and direct play.
* **Your character carries over.** Level, skills, perks, inventory and credits are saved and follow you between sessions and across savegames. Both of you keep your own progress.
* **Shared combat.** Enemies, damage and kills are synced, and both players get the kill XP in a shared fight.
* **Downed and revive.** When you go down you can be picked back up instead of dying.
* **The world stays in sync.** NPC positions, doors, grabbed objects, time and weather, and basic quest progress.
* **Follow into space and on the ground.** The client comes along through takeoff, landing and ground vehicle driving. Some of this is rough.
* **Proximity voice chat.** Push to talk or open mic, and the other player gets quieter with distance.
* **Fast travel and regroup.** Travel on your own, then press F8 to pull yourself to the host.
* **Encrypted connection.** All co-op traffic is encrypted peer to peer, and relays mean you usually do not need to forward a port. The lobby only does matchmaking and never sees game data.
* **Character backup.** Export and import your character to a file, and reset it from Settings whenever you want.

## What does not work

Stated plainly, because you will run into these:

* Station and point-of-interest spawns can diverge in space, leaving the client with a stale position that looks like being stuck inside the ship.
* A mid-session regroup does not always re-apply the client's own character profile.
* Host worn gear can persist as an unusable ghost item on the client.
* The client's gravity can stay disabled after a station or space transition.
* Riding a vehicle together crashes the client on entry.
* Equipment sync is implemented but disabled: applying it to the remote copy crashes its animation graph.

The full list, cosmetic issues included, is in [docs/OVERVIEW.md](docs/OVERVIEW.md).

## Host your own relay (optional, advanced)

You do not need this to play. A relay just passes traffic between two players when a direct connection is hard, and the tool already picks a working one for you automatically.

You might want your own as a private relay for your group, or to serve a region that does not have one yet. The relay source is in this repository under `lobby/relay`, with a Dockerfile. The standalone repository is at https://github.com/ramisotti13-eng/pulsar-relay

A relay only forwards traffic. It never loads your save or touches your character, and because the connection is encrypted it only ever sees encrypted bytes.

## Running your own lobby

The lobby backend is in this repository too, under `lobby/server` and
`lobby/deploy`. No lobby address is compiled into the tool: it reads one from a
config file or an environment variable, and otherwise falls back to a local
stack. That is deliberate, so the project does not stop working if the official
lobby ever goes offline. See [BUILD.md](BUILD.md).

## Source, license, contributing

Everything is here: the SFSE plugin (`src/`, `include/`), the tool
(`lobby/pulsar`), the companion daemon (`lobby/companion`), the relay
(`lobby/relay`) and the lobby backend (`lobby/server`).

* [docs/OVERVIEW.md](docs/OVERVIEW.md) covers the architecture and the honest state of things.
* [BUILD.md](BUILD.md) is the build guide. You must apply the patches in `patches/` after cloning, or the plugin will not work.
* [docs/design/](docs/design/) holds the design documents, including approaches that were tried and abandoned. For anyone picking this up they are the most valuable part of this repository.

Licensed under GPL-3.0, see [LICENSE](LICENSE), with additional attribution
terms in [LICENSE-ADDITIONAL](LICENSE-ADDITIONAL): keep the credit to the
original, mark your version as modified, and do not name your fork plainly
"PULSAR". Everything else the GPL grants you is untouched, commercial use
included.

Third-party components are listed in [THIRD-PARTY.md](THIRD-PARTY.md).

## Disclaimer

Not affiliated with, endorsed by, or connected to Bethesda Softworks or ZeniMax.
Starfield is their trademark. This repository contains no Bethesda assets.

Mods carry risk. Back up your saves.

---

Built by ramisotti13-eng. Thanks to everyone who tested the alpha. I hope
someone takes it further.
