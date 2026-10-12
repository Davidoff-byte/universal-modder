---
kind: game
title: Spawning a real second player in AC4 Black Flag (the working route, and why the obvious ones fail)
game: 'Assassin''s Creed IV: Black Flag'
games_also: []
game_version: 'AC4BFSP.exe (Steam), x86, image base 0x400000, no ASLR'
platform: windows
engine: native
route: native-hook
tools:
  - AC4Coop (reference implementation) - https://github.com/Kamzik123/AC4Coop
  - AnvilToolkit - https://github.com/Kamzik123/AnvilToolkit
  - Ghidra 12.1.4 + a gamedb index over the export
  - a byte-level RVA oracle (pefile + capstone), custom Frida harnesses
anti_cheat: none (single-player, offline; the MP client is never touched)
status: working
agents:
  - OpenCode (DeepSeek V4.1)
humans:
  - Kamzik123 (AC4Coop / AnvilToolkit - the spawn method and the engine class map)
date: '2026-10-11'
links:
  - https://github.com/Kamzik123/AC4Coop
  - https://github.com/Kamzik123/AnvilToolkit
tags:
  - coop
  - spawn
  - anvilnext
  - black-flag
  - player-progression
---

# Spawning a real second player in AC4 Black Flag (the working route, and why the obvious ones fail)

> In AC4 Black Flag single-player you **can** spawn a second, fully-rendered player body: build it from
> **the local player's own cached EntityBuilder**, not from any spawner. Someone else worked this out
> first — **Kamzik123's [`AC4Coop`](https://github.com/Kamzik123/AC4Coop)** is the reference, and the
> method below is his (port/credit him, not this note). This note records the route, the engine facts
> around it, and the many *near-miss* routes that do **not** work, so the next person skips them.

## Credit

- **The spawn route and the engine class map are Kamzik123's** (AC4Coop; he also wrote AnvilToolkit).
  His `src/Scimitar/` headers are reversed engine classes with real field offsets.
- The verification (every address byte-checked against the retail x86 exe, the near-miss dead ends, the
  class registry, the render model) is from our own RE; the actual *idea* is his.

## Setup

- Steam **`AC4BFSP.exe`**, x86, fixed image base `0x400000`, **no ASLR** (so absolute addresses are
  stable; **RVA = VA − 0x400000** and in `.text` the file offset is `RVA − 0xC00`).
- An ASI plugin (Ultimate ASI Loader) for the engine calls; the reference is `AC4Coop.asi`.
- **Careful:** AC4Coop's code calls its engine literals as **absolute VAs** (a `Hook::CallXMethod`
  helper). If you copy them into an RVA-based hook framework, subtract `0x400000`.

## Route and why (the working spawn: build from the player's own cached builder)

```
world  = World::GetCurrentWorld()          // cdecl, VA 0x0054D4D0
player = world->GetPlayer()                // thiscall ECX=world, VA 0x0070E810
selector  = *(void**)0x04DEEF90            // PlayerProgression: cached RowSelector
builderHC = *(void**)0x04DEEF94            // PlayerProgression: cached EntityBuilder handle container
builder   = deref(builderHC)               // the container's object; if released, reload by its id
DuplicationManager::Begin(1,0)             // cdecl, VA 0x00A2D5A0
dupSel = ((RowSelector*)selector)->Duplicate(0,0)   // VA 0x007A3BD0
DuplicationManager::End()                  // cdecl, VA 0x00A2F320   (run even on fault)
rc = ((EntityBuilder*)builder)->BuildEntityWithSelector(&entity, dupSel)  // thiscall, VA 0x005FAC00
g_blockReg = entity;  world->AddEntity(entity)      // thiscall, VA 0x0070E860
```

Key points:
- **The builder is the player's, cached when the player spawned.** There is no need to construct or
  populate one — `PlayerProgression` already holds it. This is the single fact that makes it work.
- **The builder object is usually released** after the player spawns; the *handle container* keeps its
  resource id, so reload by id: `HandleManager::CreateReference` (VA 0x00A0BE50) →
  `LoadingThread::AddLoadTask_Callback` (VA 0x00A2A6C0) → `LoadingThread::GetWaitTasks` (VA
  0x00A278F0), then poll the handle's object.
- `BuildEntityWithSelector` **consumes** the selector, so duplicate a fresh one first, inside an active
  `DuplicationManager::Begin/End` scope (outside it, `Duplicate` null-derefs).
- **Register it, then bind the pad:** hook `PlayerPadManager::GetUnusedPadID` (VA 0x00744CF0) and, while
  your "this is the puppet" flag is set, force an empty pad slot. Without this the spawned body enrolls
  onto the real player's pad/gamer and **you lose your inventory**.

## How the rest of the engine reads (what we learned the hard way)

- **A node renders only with a render record.** The per-frame graphics pass consumes a per-entity record
  reached through `Entity+0xB0` (an interface whose `+0x24` is a kind); a factory-created or `memcpy`
  clone of a node has `+0xB0 == 0` → invisible. `Entity::addVisual` (VA 0x00528300) appends a `Visual`
  to `+0x60`. (`+0xAC` is *not* a graphic pointer — it's the high half of a 64-bit flag word.)
- **Class registry:** every class has a ClassInfo descriptor (name, `id = CRC32(name)`, size, ctor).
  A vtable's `+0x14` getter returns it — enough to name any object at runtime and to find "the class
  that does X". (We dumped all 1376 for this exe.)
- **Data-driven spawners read but don't retarget:** the loader reads a cell record's
  `SpawningSpecification → SpawnNPCParams → DynamicReference` and resolves the character id, but
  crowd-life hosts take their model from the crowd system (they ignore the override), and only
  mission/domino spawners honour a chosen character — and they're mission-gated.
- **The debug/cheat "Spawn Dude" system is inert in SP:** the cheats post a `NavigationDudeEvent` whose
  type id occurs exactly once in the exe (its own descriptor) → no subscriber → dropped.

## Verification

- **Byte-level oracle:** every address above was verified against the retail exe with a `pefile` +
  `capstone` script (resolve RVA via the PE sections, compare the bytes to the expected instruction).
  This caught a real class of bug: copy-pasting AC4Coop's VAs where RVAs are required.
- **In game (2026-10-11):** the log shows `builderID = 0x2C0A3FCE0` (Edward) → `rc=0 entity=0x… added=1`
  → `pad forced to empty slot 1 (player pad kept)` → **two Edwards rendered on the Havana dock**.
- **Not verified here:** full animation replication (the reference's action/blend/phase sync is his and
  is the mature way; we only mirrored bone transforms).

## Gotchas

1. **You cannot spawn a character by creating an entity from a spawner/file-id.** The file-id resolve
   returns an empty template (`+0x60` NULL) and instantiating it faults. **Fix:** use the player's cached
   builder (above).
2. **A memory clone of the player node renders nothing.** `Entity::clone` copies the child/rig list but
   not `+0x5C` (scene) or `+0xB0` (render record). Don't go down this path.
3. **Copying AC4Coop's addresses as RVAs.** Its literals are VAs; subtract `0x400000`.
4. **The second player steals your pad/gamer (inventory loss).** **Fix:** the `GetUnusedPadID` hook
   forcing an empty slot while the puppet is being enrolled.
5. **`RowSelector::Duplicate` null-derefs outside `DuplicationManager::Begin/End`.** Wrap it, and always
   `End` (use `__try/__finally`) or the scope corrupts every later duplication.
6. **The builder is often released.** `reloaded=0` on one load, `=1` on another — handle both (reload by
   id when the object isn't readable).
7. **`+0xAC == 0` on a clone is not "missing graphic".** It's a flag word; the real gate is `+0xB0`.
8. **A spawned body that "glides" has no walk children.** The spawned body's character node reads
   `c14+0x134 == 0` (a null walk-child pointer), so a per-frame bone mirror writes to nothing — the
   body translates but the legs never cycle. **Fix:** drive a body that actually carries a skeleton
   (the crowd-hijack or `mod16` source), or stop mirroring bones and replicate **actions**
   (`SetTargetAction` + blend tree) instead.

## Assets

None — no game files or extracted assets are shipped. Everything is code + addresses.

## Open questions

- Animation replication parity with the reference (action alias / blend tree / blend-in descriptor /
  phase seek). That is the reference's strength and where a fresh implementation should look first.
- Whether the spawned body survives world travel cleanly (the reference respawns it across travel).

## Related notes

- `coop-ghost-avatar.md` — the earlier "drive a hijacked crowd NPC" body (works, wrong model). This
  note supersedes it for the *body source*.
- `why-ac4-black-flag-sp-cannot-spawn-a-new-character-...md` — **superseded on its conclusion** (see the
  dated correction there).

---

## Update 2026-10-12 — the body spawns, but it has no walk skeleton; animate by action replication

The spawn above is real, but a fresh spawned body is **not** animation-complete on its own, and that is
where the "it glides" reports come from.

- **The spawned body has no walk-child skeleton.** On the body we built, the character node's
  walk-child pointer reads **`c14+0x134 == 0`** (null). With no walk children there is nothing for a
  bone-mirror to write to, so a per-frame bone copy is a *silent no-op*: the body's world matrix moves
  (it follows your position) but the legs never cycle, so it slides/glides instead of walking.
  **Symptom → cause → fix:** gliding body → `c14+0x134 == 0`, no walk children → use a body that *has*
  a skeleton, or stop mirroring bones and replicate actions instead.
- **The crowd body is the source that *does* animate this way.** The hijack route in
  [`coop-ghost-avatar.md`](coop-ghost-avatar.md) drives a crowd character, which *does* have a
  skeleton, so the mirror visibly animates it. The same held for the **`mod16` forge** body we tried:
  it renders **and** animates through the mirror, because it carries walk children. So "spawn from the
  cached builder" and "get a rig that animates" are two separate problems — the cached builder gets you
  the *identity* (a real Edward), while the crowd/`mod16` source gets you the *skeleton*.
- **The correct animation route is action replication, not bone writes.** Instead of copying bone
  transforms, drive the body's **action** and let the engine's own blend tree interpolate:
  `SetTargetAction` picks the action, `ActivateCharacterActionSet` makes the action set live, the
  `SimpleAnimBlender<46>` blend tree blends it over time, and `GetCurrentAction` /
  `SetCurrentActionRatio` read and seek the blend phase. That is exactly what the reference does, and
  it is why the reference looks right where a bone mirror looks like sliding. Addresses we recorded
  (x86, image base `0x400000`, **absolute VAs** — subtract `0x400000` for an RVA):

  | What | VA |
  |---|---|
  | per-frame hook point | `0x12FAACD` |
  | `SetTargetAction` | `0x6625D0` |
  | `GetCurrentAction` | `0x666C80` |
  | `SetCurrentActionRatio` | `0x652640` |
  | `ActivateCharacterActionSet` | `0x651840` |

  plus the `SimpleAnimBlender<46>` blend tree the actions feed into. **The method is Kamzik123's**
  (`AC4Coop`) — action replication is his approach and the mature one.
- **Verified vs not, honestly.** Verified live (2026-10-11/12): the cached-builder spawn renders a
  second Edward; the spawned body's walk-child pointer is null (`c14+0x134 == 0`); the crowd-hijack body
  and the `mod16` forge body both render **and** animate through the bone mirror. **Not verified by
  us:** end-to-end action replication on the spawned body — the addresses above are *recorded*, not
  successfully driven from here, and it is unconfirmed whether `SetTargetAction` alone suffices or
  `ActivateCharacterActionSet` must be called first. Treat the table as a starting point, not a recipe.
