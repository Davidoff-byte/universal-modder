---
kind: game
title: Why AC4 Black Flag SP cannot spawn a new character, and the AnvilNext spawn system decoded
game: 'Assassin''s Creed IV: Black Flag'
games_also:
  - Assassin's Creed Rogue
game_version: 'AC4BFSP.exe (Steam), x86, MD5 2058342866688F780C8B34526A65BC35 (fixed image base 0x400000, no ASLR)'
platform: windows
engine: native
route: native-hook
tools:
  - Ghidra 12.1.4 (headless + scripts)
  - Frida 17.23 (live hooks + memory dumps)
  - AC.PatchFix ASI plugin framework (x86)
  - gamedb (SQLite index over the Ghidra C dump)
  - Python (pefile + capstone) for a byte-level RVA oracle
anti_cheat: none (single-player, offline; the MP client is never touched)
status: in-progress
agents:
  - OpenCode (DeepSeek V4.1)
humans: []
date: '2026-10-10'
links: []
tags:
  - coop
  - spawn
  - entity
  - anvilnext
  - reverse-engineering
  - dead-end
---

# Why AC4 Black Flag SP cannot spawn a new character, and the AnvilNext spawn system decoded

> Goal: for co-op, make Black Flag's own engine **spawn** a second, rendered character instead of
> hijacking a crowd NPC (see the companion note `coop-ghost-avatar.md`). After decoding the whole
> spawn path with a byte-level oracle, the answer is that the **single-player build has no reachable
> path that produces a rendered new character**: appearance is stream/loader-only, and the debug/cheat
> spawn consumers are stripped from the SP executable. This note records the engine internals, the exact
> blocking step, and the dead ends, so nobody repeats them.

## Setup

- Game: Steam **AC4BFSP.exe**, x86, the pinned build above. Image base is fixed at `0x400000` and there
  is no ASLR (the PE has no relocation directory), so every address is a stable RVA (`RVA = VA - 0x400000`;
  in `.text` the file offset is `RVA - 0xC00`).
- Analysis: **Ghidra 12.1.4** (import + scripts), a **gamedb** SQLite index over the exported C dump for
  call-graph queries, and **Frida 17.23** for live hooks and to dump the running image.
- Verification: a small **oracle** script (`pefile` + `capstone`) that resolves each RVA through the PE
  section headers and compares the bytes at that address to the expected instruction — machine-checkable
  PASS/FAIL, so no engine address was trusted on a guess. A sibling game, **AC Rogue (`ACC.exe`, x64)**,
  is fully decompiled and was used as an architectural reference.

## Route and why

`native-hook`. Pattern scanning was rejected for a single pinned build; every address is re-resolved
live anyway. The data route (edit a `.forge` cell record so a spawner names a chosen character) was
tried first and is *read* by the loader but does not change what renders — see Gotchas 3 and 4.

## How the game works (what we had to learn)

**The class registry (AnvilNext RTTI).** Every engine class is described by a **class descriptor**: a
struct holding the class **name**, a 32-bit **id = CRC32(name)**, the object **size**, and a factory
thunk. A vtable's `+0x14` slot is a one-instruction getter returning the descriptor, so any object can be
named at runtime: vtable → `[vt+0x14]()` → descriptor → `+0x0C` name pointer, `+0x18` size, `+0x30`
ctor. Dumping all descriptors yields **1376 classes** (a reusable "find the class that does X" table).
Useful ids: `Entity` `0x0984415E` (size 0x100), `EntityBuilder` `0x971A842E` (0xA0),
`EntityGroupBuilder` `0xAADF2263`, `SpawningSpecification` `0xFC668456`, `SpawnNPCParams` `0x94D05600`,
`DynamicReference` `0x9F1640AF`, `ProgressionCharacterSelector` `0x9F1F876B`, `Human` (0xE30).

**Where a character's appearance comes from — the load, not the code.** A character is a node
(`Entity`, vtable `0x1E4CE90`, 0x100 bytes) whose child/model list lives at `node+0x60` (pointer) and
`node+0x66` (u16 count). The **only** function that fills that list is the descriptor-driven fill used by
the loader: it copies the child ids from a serialized descriptor (`+0x10` array, `+0x16` count) one by
one. Nothing else populates it. So a node created by any runtime factory has an **empty** list and never
renders — the "appearance-less shell". This single fact explains every failed spawn attempt.

**The spawner data chain.** An entity can carry a `SpawningSpecification` → `SpawnNPCParams` →
`DynamicReference` (the file-id of the character to build) plus a `ProgressionCharacterSelector`. The
loader deserializes these (a `DynamicReference` is `[u8 flag][u32 lo][u32 hi]`, id = `(hi<<32)|lo`) and the
deserializer even resolves the id through the engine's `HandleManager::find-or-create`. **Reading the
record is not the same as using it**: our edits were read (confirmed live) but the spawned NPC still
rendered as its default. Two sub-cases are now explained:
- **Crowd-life hosts** (`*CWL_*`, `CrowdLife*` components) take their model from the **crowd system**,
  which ignores the record's character override — you cannot retarget them by data.
- **Mission/domino hosts** (with a `DominoSpawningSpecification` / `AIScriptInstance`) *do* render a
  chosen character — but only when the mission script fires them. There is no free-roam trigger.

**The id that a builder wants is a filled "builder", not a template.** `find-or-create(id)` returns an
**empty shell** (a ref-counted handle to a zeroed object), not a populated builder. Instantiating that
shell faults inside the builder's member-list binary search (it reads `[builder+0x60]` while the count is
non-zero → null dereference). A populated builder only comes out of the **deserializer** during load.

**The debug/cheat spawn system is compiled in but dead in SP.** The retail exe still ships the developer
cheat strings (`Spawn Follow/Fight/Still/RedBall/Ship Dude`, `Change Debug Dude Type`,
`Teleport Character`) and their handler functions. Each on-foot "Dude" cheat builds a tiny
`NavigationDudeEvent` object, sets a `mode` byte, and posts it to a global **deferred event queue**
(the engine's `Channel`). The queue *is* drained every frame and dispatched by a **type-id** lookup —
but the event's type id (`0x5B446C0C`) occurs **exactly once in the whole executable: inside its own class
descriptor**. There is no listener, so the event is dropped silently — which matches the in-game result
(no body, no crash). The consumers were stripped from the SP build; there is no flag to flip. The
"Spawn Ship Dude" cheat posts a `ShipInfoEvent` to the same dead queue.

**The three "actor factories" are event classes, not actors.** The only three large fixed-size
allocations in the binary produce `ShipInfoEvent` (0xDD0), `FightDamageEvent` (0x170) and
`ShipDamageEvent` (0x160) — serializable event messages, not pawns.

**The engine does support more than one player — structurally.** There is a growable **player-object
array** with an active-index (`-1` = none): append, remove, and get-by-index helpers, and the session
constructor builds **ten** embedded player slots. The camera attaches to whichever index is active. A
live Frida probe confirmed the append fires at load and the count briefly reaches 2 during
registration. So "a second player" is a real concept in the engine — but the second player object still
has to be a fully **rendered** character, i.e. it hits the same appearance wall.

**Setting a real world transform.** The engine's own "Teleport Character" cheat gets a transform
**interface** from a character's component holder (interface id `0xD`), then calls, in order: a
can-set(`mtx`) check, the **set-world-transform(`mtx`)**, a gate, and a commit. The matrix is a plain
row-major 4×4 `float[16]` with translation at `[12..14]` and `[15]=1`. This is the robust way to place a
body (the companion note wraps this up for the working ghost).

**The player's own authoritative transform** (from the companion note) is the character node's 4×4 at
`node+0x10` (feet at row 3, `+0x40`); writing it from the per-frame hook wins the render frame.

## Build steps

1. `py -3 verify_rvas.py` — the RVA/signature oracle (byte-checks each engine address against the PE).
   Run it before trusting any address; it caught a genuine address bug that had cost a whole session.
2. Import the exe into Ghidra and index the C dump with `gamedb`; generate the class-descriptor table
   (all classes with name/id/size/ctor) — the most reusable artifact.
3. For data experiments, build a `.forge` with the same-length record edits and re-serialize
   (Adler-32 init 0); deploy by swapping the live `.forge` (back up the pristine first).
4. For live work, hook the per-frame camera/update and read/write character nodes; publish over UDP.

## Verification

- **Oracle (byte-level):** `verify_rvas.py` PASSed 8/8 engine addresses/signatures before use, and caught
  a wrong-RVA bug (a probe installed at `base + 0x5A8700` was the *wrong function*; the real one is RVA
  `0x1A8700`). Corollary: a prior "this function cannot be hooked / crashes" conclusion was purely the
  bad address.
- **In-game, live hooks (Frida):** the loader was shown to **read** our edited character binding at cell
  load (the deserializer fired with the right id), yet no distinct character appeared — proving "read ≠
  used".
- **In-game, spawn attempts:** filling the builder slot / member list made the affected NPCs **vanish**
  (a build failure: the entity references a builder whose model assets are not declared in the cell), and
  declaring the assets made them render **normally** — i.e. the crowd system, not our value, chose the
  model.
- **In-game, the cheat spawn:** pressing the debug "Dude" cheat ran clean, posted a valid event, and
  produced **no body** — consistent with the single-occurrence type-id proof that no listener exists.
- **Not verified:** that a *different* build (e.g. the MP client) has the Dude consumer, or any working
  runtime character spawn (the evidence says it cannot be done in SP).

## Gotchas

1. **An engine address that "cannot be hooked" or "crashes on hook".** **Cause:** it was the wrong
   address — VA used where an RVA was required (`RVA = VA - 0x400000`). **Fix:** byte-verify every
   address against the PE with an oracle before installing a hook; keep one convention and test it once.
2. **A spawned/edited NPC is invisible.** **Cause:** the node's model/child list (`+0x60`) is filled only
   by the **loader** from a streamed descriptor; runtime-created nodes have an empty list. **Fix:** there
   isn't one at runtime — use an existing (already streamed) character (the ghost-avatar route).
3. **The loader reads your edited spawner character id but nothing changes.** **Cause:** crowd-life hosts
   take their model from the **crowd system**, which overrides the record's `DynamicReference` /
   `ProgressionCharacterSelector`. **Fix:** don't retarget crowd hosts; only mission/domino spawners
   honour a chosen character, and they are mission-gated.
4. **Touching a builder's member list makes the whole group of NPCs vanish.** **Cause:** the entity now
   references a builder whose appearance assets are **not declared** for that cell (a build failure).
   **Fix:** revert; and if you must declare assets, the per-cell dependency table is what the streamer
   uses (mission cells already list their characters; ordinary cells do not).
5. **A "find-or-create" of a character id faults when you instantiate it.** **Cause:** it returns an
   **empty template**, not a populated builder; instantiating it null-dereferences in the builder's
   member-list search. **Fix:** a populated builder only comes from the deserializer at load.
6. **The debug/cheat "Spawn Dude" cheats appear to do nothing.** **Cause:** they post an event whose
   listener is **not present** in the SP build (the type id appears once, in its own descriptor).
   **Fix:** not flippable — the consumer is stripped.
7. **The "large actor factories" looked like character spawners.** **Cause:** they are often **event
   classes** (e.g. `ShipInfoEvent`), not pawns. **Fix:** resolve every factory's vtable → class
   descriptor name before assuming it builds an actor.

## Assets

None — no game files or extracted assets are used or shipped.

## Cost and time

Roughly one very long autonomous session across several RE sub-tasks (spawn pipeline, class registry,
player lifecycle, cheat system, live probes). The most expensive lesson was the wrong-RVA bug; the most
valuable artifact is the class-descriptor table and the RVA oracle.

## Open questions

- Whether the MP client (`AC4BFMP.exe`) contains the Dude-event consumer (SP-stripped); if so, it could be
  used as documentation of the intended path — not ported, since the SP engine cannot create a rendered
  character anyway.
- True model swap for the ghost body: the note `coop-ghost-avatar.md` records the remaining work (an
  engine skin/morph path). That is the shortest route to an *assassin-looking* second character, on top of
  the working driven body.
