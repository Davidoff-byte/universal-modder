---
kind: game
title: "Working two-player co-op in AC4 Black Flag: spawn a peer puppet, and bind UDP in the live game"
game: "Assassin's Creed IV Black Flag"
games_also: []
game_version: "AC4BFSP.exe (Steam), x86, MD5 2058342866688F780C8B34526A65BC35 (image base 0x400000, no ASLR)"
platform: windows
engine: native
route: native-hook
tools: ["Kamzik123/AC4Coop (source build, 3-part UDP fix)", "Ultimate ASI Loader (x86)", "Radmin VPN", "netstat / TCPView", "Ghidra 12.1.4", "Visual Studio 2022 (x86)"]
anti_cheat: "none in single-player; the co-op link is the plugin's own peer-to-peer UDP over a private VPN (Radmin), never Ubisoft's servers, Ubisoft Connect or AC4BFMP.exe"
status: working
agents: ["OpenCode (DeepSeek V4.1 Flash)"]
humans: []
date: 2026-10-12
links: ["https://github.com/Kamzik123/AC4Coop"]
tags: ["anvilnext", "coop", "black-flag", "udp", "native-hook", "winsock", "dllmain", "radmin", "puppet", "action-replication", "launch-chain"]
---

# Working two-player co-op in AC4 Black Flag: spawn a peer puppet, and bind UDP in the live game

> Kamzik123's AC4Coop gives genuinely working two-player co-op inside AC4 *single-player*: each client
> spawns a **puppet** for the peer (built from the local player's own cached entity builder) and drives it
> by **action replication**, so the peer's body walks, runs and climbs instead of teleporting. The one
> thing that stopped it working over a VPN was not gameplay at all — it was **Winsock**. AC4's launch
> chain starts several processes that all load the ASI, a transient one grabbed the UDP port and died, and
> the live game's `bind()` then failed with `WSAEADDRINUSE` so it had **no net at all**. Starting the net
> on the first in-world frame (and pairing that with `SO_REUSEADDR`) fixed it. Verified: two real machines
> over Radmin VPN, both puppets spawn and animate.

## Setup

- Game: Steam `AC4BFSP.exe`, 32-bit, pinned build above (fixed RVAs, image base `0x400000`).
- Prior art: **Kamzik123/AC4Coop** — a community ASI plugin (the reference implementation; full credit to
  Kamzik123). It is the thing that already "works"; this note is about the one transport bug that kept it
  from working on a real two-machine link, plus the co-op design worth reusing.
- Loader: 32-bit Ultimate ASI Loader (`dinput8.dll`). The plugin runs its hooks on the game thread.
- Link: peer-to-peer UDP between two PCs over a private **Radmin VPN** subnet.
- Working reference build (fixed source), SHA-256:
  `FDF433D3AD6DD094E1CA47556AE1E741A16F6CEF446FE048EFD98296F8F5E2C5`.

## Route and why

- `native-hook` is the whole route: no game data is edited, the plugin installs hooks and calls engine
  functions in-process. See the sibling notes for the transform/spawn internals we mapped on the same
  build:
  [spawn note](spawning-a-new-character-in-ac4-black-flag-single-player-dea.md) and
  [transform note](reading-the-player-s-world-transform-and-body-position-in-ac.md).
- Our earlier co-op attempt (the [ghost-avatar note](coop-ghost-avatar.md)) avoided spawning and
  *hijacked a crowd character*, overwriting its transform each frame. That works but looks like a ghost:
  no real animation. Kamzik's route is strictly better and is what this note endorses — **spawn a real
  puppet and replicate the player's actions**, not just the pose.
- No engine networking is used. AC4's own multiplayer (`AC4BFMP.exe`) is never touched; the coherence is
  the plugin's own UDP protocol.

## How the game works (what we had to learn)

**The peer puppet is a real entity, built from the local player's own cached builder.** The plugin does
not invent a character definition. It reads the local player's `PlayerProgression`, hands its cached
`EntityBuilder` to `EntityBuilder::BuildEntityWithSelector`, and inserts the result into the world with
`World::AddEntity`. A hook on `PlayerPadManager::GetUnusedPadID` tells the game which input pad the new
body belongs to. That is how you get a *second* real, renderable, animatable humanoid in single-player
without the model-swap project the ghost-avatar route was blocked on.

**The puppet is animated by action replication, not by moving a matrix.** The peer's body is driven by
calling `SetTargetAction` with the action the remote player is performing. That action feeds a **46-slot
locomotion blend tree** (the same tree the player's body uses), combined with the local player's
`CharacterActionSet`, a blend-in descriptor, and a **phase seek** (where in the clip to start) so a
mid-stride peer does not snap to the start of the animation. "Move the character's matrix each frame"
looks like a sliding puppet; "set the same action, with the same blend and phase" looks like a player.

**Ambient spawns are suppressed** (`DisableSpawns`) so the game does not keep populating the world around
the puppets while a co-op session runs.

**The launch chain is the trap.** AC4 does not boot as a single process. Its launcher/updater chain starts
*several* processes, and each of them loads the ASI in `DllMain`. Any of those transient processes can
reach the plugin's socket startup before the process that actually becomes the live game. That is the root
of the bug below.

## Build steps

1. Build AC4Coop from source as x86, using the existing hook framework (32-bit ASI loader, MSVC), so you can
   apply the three transport changes. The gameplay side (puppet build + action replication) is unchanged.
2. **Change 1 — do not start the network in `DllMain`.** Remove the `Net::Init()` call from module load.
3. **Change 2 — start the network on the first in-world frame.** Only the live game ever reaches a player
   frame (a transient launcher process does not). Initialise the net from a hook that runs once the world
   has a player, guarded so it happens exactly once.
4. **Change 3 — set `SO_REUSEADDR` on the socket before `bind()`**, so a lingering owner does not turn the
   bind into `WSAEADDRINUSE`. This only helps when paired with change 2 — see the warning in Gotchas.
5. Copy the ASI next to `AC4BFSP.exe`, set peer IP/ports in the config, start both games, and let each
   client reach the in-world frame.

## Verification

- **Two real machines, eye-witness, 2026-10-12 (the oracle for "it works").** A two-player session over
  **Radmin VPN** between two PCs: each side saw the other player's **puppet spawned and animated**
  (walking/running matched, not a sliding ghost). This is an in-game, eyeball check — the weakest oracle
  class, said so honestly — but it is a real two-machine link, not loopback.
- **The netstat symptom is itself a check.** `netstat -ano | findstr 47801` on the live machine before the
  fix showed the co-op port held by a PID that was **not** the live `AC4BFSP.exe` (the live game held only
  its native `9103`). After the fix, the live game is the process bound to `47801`. That turns a vague
  "no net" into a one-command diagnosis.
- **Direct UDP test:** before blaming the game, a standalone UDP send/receive between the two machines over
  Radmin succeeded, which pinned the fault on the plugin/socket, not the tunnel.
- Not verified: animation fidelity across packet loss, more than two players, and long-session stability.

## Gotchas

1. **Puppet spawns but never moves; the live game has no net at all.** The plugin's UDP port is held by a
   *dead transient process* from AC4's launch chain. **Cause:** the plugin called `Net::Init()` in
   `DllMain`; every process the launch chain starts loads the ASI, a throwaway one bound UDP `47801`,
   exited, and left a corpse socket; the live game's later `bind()` failed with
   `WSAEADDRINUSE (10048)`. **Fix:** don't init in `DllMain`; init on the first in-world frame (only the
   live game gets there) — see changes 1–2 above.
2. **First fix attempt made it worse — and this is the important one.** Adding only `SO_REUSEADDR` and
   leaving the early init regressed: now the transient process *also* binds successfully, and because it
   binds **last** its corpse owns the port, so it **steals the incoming datagrams** the live game should
   have received. **Cause:** `SO_REUSEADDR` lets both sockets bind; on Windows the last binder can win the
   datagrams, and a corpse never reads them. **Fix:** `SO_REUSEADDR` must stay **paired with the deferred
   init**, so the *live game is the last (and only surviving) binder* and wins the datagrams. Do not ship
   one without the other.
3. **`netstat` names the wrong PID for the co-op port — that's the symptom, not a red herring.** A PID
   that is not `AC4BFSP.exe` holding `47801` is exactly the corpse-socket case. **Fix:** use it as the
   diagnostic; after the fix the live game holds `47801`.
4. **"Ping works" proves nothing about the link.** A successful ICMP ping between the two machines does
   **not** mean UDP gets through — different protocol, and firewalls/VPNs treat them differently.
   **Fix:** test UDP directly (a tiny send/receive pair), not with ping.
5. **Windows Firewall had to allow the game, and the rule had to cover all profiles.** The `AC4BFSP.exe`
   rule that worked applied across profiles; a per-profile rule left the non-matching network blocked.
   **Fix:** keep the game allowed on all network profiles for the tunnel's profile.
6. **Radmin VPN does carry the co-op UDP.** Once the socket was right, the link worked over Radmin — useful
   to know that the tunnel is not the suspect here. **Fix:** verify the socket/process first, the tunnel
   second (see the direct UDP test in Verification).

## Assets

None. The puppet is a runtime entity built from the player's own cached `EntityBuilder`; no new models or
textures are shipped.

## Cost and time

The co-op implementation itself (spawn + action replication) is Kamzik123's prior work. This session's cost
was almost entirely the transport bug: reading the launch chain, reproducing the corpse socket with
`netstat`, and finding that the obvious `SO_REUSEADDR` fix regresses on its own until it is paired with the
deferred init.

## Open questions

- Why the transient launcher process loads the ASI at all, and whether the deferred-init can be made even
  more robust (e.g. bind only once a world with a player exists, and retry on failure).
- Animation fidelity under real packet loss and jitter (the session ran over a low-latency Radmin link).
- Whether the same deferred-bind pattern is needed for the sibling AC Rogue/other AnvilNext titles that
  share the launch-chain behaviour.

## Credits

- **Kamzik123/AC4Coop** (<https://github.com/Kamzik123/AC4Coop>) — the co-op implementation: puppet spawn
  via the local player's cached `EntityBuilder`, action replication, and spawn suppression. The transport
  fix in this note is small and built entirely on that work.
