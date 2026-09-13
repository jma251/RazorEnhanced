# What is on `dev` and not yet on `release/1.0`

A record of every change made on the `dev` branch, what evidence it rests
on, and whether it has been tested in real play. Nothing here should be
merged into `release/1.0` until its row says it was tested — `release/1.0`
is what players download, and 1.0.0.13 is the reminder of what shipping an
untested change costs.

Dev builds are versioned `1.0.0.149NN` and published as a prerelease on the
`dev` tag. Players on the normal download never see them.

**Legend for "Verified"**

| | |
|---|---|
| **fact** | follows from the code or a data file in this repo, not from a comment or an assumption |
| **built** | compiles in CI; says nothing about whether it behaves correctly |
| **played** | actually exercised in game |

---

## Tooling and release plumbing

| Change | Commit | Verified |
|---|---|---|
| Dev build lane — separate workflow, prerelease on the `dev` tag, refuses to run on `release/1.0` or publish to `latest` | `1558d237` | played |
| Dev builds update themselves from the dev manifest; the URL is stamped in at build time, never committed, so merging dev cannot move players onto dev builds | `204a11a6` | played |
| Packet logger can record Razor's own outgoing packets (`PacketLogger.ListenPacketPath("RazorToServer", True)`), off by default | `a072c5a1`, `54686255` | played |

Before this, Razor could not see the packets it sent itself. Two days of the
bandage investigation were spent reasoning about packets nobody could
observe.

## Scripting API

| Change | Commit | Verified |
|---|---|---|
| `Player.IsCasting`, `Player.CastingSpell` — built from the cast command on the wire, so they work on OSI as well as ClassicUO | `2b9dbc09` | built |
| Movement tracking: `IsMoving`, `IsRunning`, `IsWalking`, `StepDuration`, `LastStepDelay`. Pace-aware — 400 ms foot walk, 200 ms foot run, 200 ms mounted walk, 100 ms mounted run, taken from the running bit in the direction byte | `2b9dbc09` | built |
| 21 properties the engine already tracked but never exposed: `TithingPoints`, `Race`, the `Max*Resistance` set, `DamageMin`/`DamageMax`, `LastSpell`/`LastSkill`/`LastObject`, `Expansion`, `Season`, light levels, `SpeechHue` | `7cb8fc34` | built |
| `Player.GetStatStatus` — was switching on the wrong enum and could not return a correct answer | `7cb8fc34` | built |
| Null and torn-read hardening on the above (x86 build, so 64-bit fields need `Interlocked`) | `f86e7712` | built |
| `CUO.Follow` / `FollowOff` / `Following` — used `GetField` against what is a property, so they had been dead for years | `af789794` | built |
| `Misc.GetContPosition` on ClassicUO — walks `UIManager.Gumps` for a container or grid-container gump; returns `(0,0)` rather than throwing when there is none | `d705051f` | built |
| Cast end is taken from the player's target response, not a countdown | `e227fda5`, `c60e996e` | built |
| Spell timeouts load lazily instead of from a static constructor | `b11712f5` | built |

## Bug fixes found by reading the code

| Change | Commit | Verified |
|---|---|---|
| Journal instance list never shed dead entries. The sweep tested `TryGetTarget(out el) && el == null`, which cannot be true — `TryGetTarget` returns true only when the target is alive, and then `el` is not null. Every journal a script created left a stub for the life of the process, and `Enqueue` walked all of them on every journal line. Also dropped the empty `~Journal()` left behind when that logic moved | `89c0c2b2` | fact, built |
| `Spells.WaitCastComplete` kept its `CountdownEvent` in one static field, so two scripts casting at once stranded the first until its timeout. Now one event per call, all of them signalled. `CountdownEvent` has no finalizer, so it also leaked a handle per cast; it is disposed now, and removed from the list first so the signalling path can never reach a disposed one | `aac75c9f` | fact, built |
| `PathFindTo`'s `z` was documented as a coordinate and then dropped. `Route` has no Z field; the pathfinder reads each tile's height from the map itself. Comment corrected, no behaviour change | `dd8babd6` | fact |

## Bandage agent

| Change | Commit | Verified |
|---|---|---|
| The agent registers its answer before the cursor exists (`QueueAutoTarget`), with an expected target flag, so it can only ever answer its own cursor and never one the player is holding | `ebc14ab6` | built |

This does **not** solve the cursor problem, and nothing here can. Testing
established that the server cancels any held cursor 90–250 ms after a
bandage, by both the targetless `0xBF 0x2C` path and the double-click path.
TazUO sends the identical packet, so there is nothing to copy from it.

---

## Tried and withdrawn

Kept here so the same ground is not covered twice.

| Idea | Why it was dropped |
|---|---|
| Start the cast clock on spoken power words (`1c4c542d`, reverted `a84038b7`) | Cause and effect backwards. Typing the words is just speech; it starts no cast |
| `Target.KeepCursorOnServerCancel` (`b43fcf55`, removed `77892681`) | Blocking the server's cancel leaves a cursor that is drawn but does nothing when clicked. Tested and confirmed cosmetic |
| Deferring a bandage until the cursor was free (`8ff90b8e`, reverted `f2ad3350`) | Bandages must never wait |
| Faster Casting prediction for `CastingTimeLeft` (`b768d281`, stripped `c60e996e`) | The FC caps for necromancy, mysticism, spellweaving, mastery, druid and cleric were extrapolated, not known. `Player.FasterCasting` and `FasterCastRecovery` are exposed, so a script can do its own shard's arithmetic correctly instead |
| Narrowing `WaitForTargetOrFizzle` to sound `0x5C` (`00db6d5f`, reverted `82245567`) | Packet `0x54` is broadcast to everyone in range, so the right sound id still cannot tell your fizzle from someone else's nearby. Correct less often is not correct. The file is byte-identical to before |
| "`Target.SelfQueued()` is broken because it fires immediately" | Not a bug. `forceQ` means "queue even when the global QueueTargets setting is off, and do not announce it" — the fire-now branch is the point of a hotkey. Read the name, not the code |

## Known, unchanged, deliberately

- **`m_QueueTarget` is a single slot** shared by hotkey queueing and agent
  auto-targeting (`Razor/Core/Targeting.cs`). An agent queueing a target
  overwrites a player's pending one, and the reverse. Real, structural, and
  not something to change casually.
- **ClassicUO disconnect stops nothing.** `ClassicUO.OnDisconnected` only
  sets `Connected = false`. OSI's logout stops autoloot, scavenger, bandage
  heal, organizer, dress, restock and all scripts; ClassicUO does none of
  it, and `World.Player` is never nulled there, so scripts keep running
  against stale data. Changing it is a behaviour decision, not a bug fix,
  and a shard transfer looks like a disconnect.
- **62 empty `catch {}` blocks** out of 273. Some are right — a diagnostic
  must not break a packet send. Sweeping them all is not a safe change; the
  useful version is a log line in the handful on paths actually used.
- **`Player`'s unguarded property getters.** 78 read `World.Player.X` with
  no null check, but `World.Player` is assigned in exactly two places —
  at login, and to null on OSI logout, which happens after scripts are
  already told to stop. On ClassicUO it is never nulled at all. Not worth
  78 edits to a file this size.
