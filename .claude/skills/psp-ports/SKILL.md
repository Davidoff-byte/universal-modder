---
name: psp-ports
description: Make a game run on real PlayStation Portable hardware (or PPSSPP). Covers the PSP hardware ceiling, the five routes — emulation, source port, decompilation, static recompilation, clean reimplementation — the pspdev toolchain and EBOOT.PBP packaging, testing on PC then on device, asset conversion, and the no-ROMs/no-assets publishing rules. Use when the user asks to port, rewrite or recompile a game for PSP, make a game playable on PSP, or asks whether a given game can run on PSP.
---

# PSP ports

Getting a game onto the PSP is usually **not** a rewrite. Pick the lightest route that delivers the idea, and
rule emulation out first — most "make X playable on PSP" requests are already solved by a homebrew emulator.

The PSP is fixed hardware: two 333 MHz **MIPS32 R4000** cores, **32 MB RAM** (64 MB on later models),
**2 MB VRAM**, a 166 MHz GPU with **no shaders**, 480×272 screen. MIPS32 matches PS1/N64/PS2 code and the
VFPU is a bonus, but 2 MB of VRAM and a fixed-function GPU are the wall. Modern 3D engines do not fit.

## Start here

The user may invoke this with a game and a target (`/psp-ports <game>`). Do the setup yourself, then propose a
route:

1. **Target.** Real hardware, or PPSSPP only? PPSSPP emulates PSP specs, so you still build to PSP limits —
   the target changes the *testing* loop, not the ceiling. Confirm the model too (32 vs 64 MB RAM).
2. **Ownership.** The user must own the game and dump it themselves. Never download ROMs or ISOs. If the only
   way forward needs a downloaded ROM/ISO, stop and say so.
3. **Fingerprint the game.** Platform of origin, engine, source availability, age, 2D/3D. Use `game-recon`
   and `um scan` if the PC build is installed.
4. **Search before building.** `um kb search "<game or engine>"`, then the web: `"<game> PSP port"`,
   `"<game> decomp"`, `"<game> recomp"`, `"<engine> PSP"`. A working port may already exist.
5. **Pick a route, lightest first** (table below). Make sure route 0 is genuinely ruled out before route 1.
6. **Write a port plan** from [`references/port-plan.md`](references/port-plan.md) and keep a `MODLOG.md`.
   Detail on each route and the toolchain is in [`references/routes.md`](references/routes.md).

## Route chooser

| Route | Rewrite effort | Realistic for |
|---|---|---|
| **0. Emulation** — run an emulator, change nothing | none | NES, SNES, Genesis/Sega CD, GB/GBC/GBA, PS1 (POPS), arcade, ScummVM adventures, Doom/Quake-family engines, N64 (partial) |
| **1. Source port / engine reimplementation** | low–medium | Any game with an open engine or a clean-room reimplementation (SDL-based ports well) |
| **2. Decompilation → cross-compile to MIPS** | high | N64/PS1 titles with a matching decomp — the code is already MIPS |
| **3. Static recompilation** | high | Games a recompiler supports (N64Recomp, PS2Recomp, psxrecomp/RecompOne, DolRecomp) |
| **4. Clean reimplementation / demake** | very high | 2D and simple games; a full AAA rebuild is a multi-month project |

**Route 0 first.** A homebrew emulator already makes thousands of titles playable; "porting" one of those is
wasted work. **Route 1 is the sweet spot** for PC games. Routes 2–3 exploit the fact that PSP is MIPS, like
the source consoles; the CPU is the easy half — the GPU runtime is the hard half (see `references/routes.md`).
Route 4 is `mashup-mods` Pattern 4: read the user's own install for assets, use `fal-assets` for new art.

## Toolchain and build

- Install **pspdev** (psp-gcc, PSPSDK, psp-packages), or use the `pspdev/pspdev:latest` Docker image.
  Export `PSPDEV` and put `$PSPDEV/bin` on `PATH`.
- Build with the `psp-cmake` toolchain, or `make` with `psp-gcc`. Link `pspgu`/`pspgl`, `pspaudio`, VFPU,
  and the ported SDL/SDL2, zlib, libpng, freetype, vorbis/opus libraries.
- Output is an **`EBOOT.PBP`** (`mksfo` + `pack-pbp`, or `ebootsigner` for signed homebrew).
- Commands, package names and a minimal `psp-cmake` example are in `references/routes.md`.

## Test loop

1. **PPSSPP on the PC** for fast iteration — run, screenshot, savestates, memory. Drive it the way
   `game-automation` drives a PC game (windowed, fixed size, screenshot; see that skill for input safety).
2. **Real hardware** for the final check, via CFW/HEN. Use `psplinkusb` (`usbhostfs` + `pspsh`, optionally
   `psp-gdb`) to read logs and catch crashes on the device. Confirm frame rate and RAM headroom on the
   32 MB model if that is the target.
3. Record findings in `MODLOG.md`. If the work has to move to another harness or directory, use the
   `handoff` skill.

## Assets and publishing

- **Never ship game files, extracted assets or decompiled source.** Ship code, patches and converters that
  transform the *user's own* files at install time; regenerate anything you cannot ship with `fal-assets`.
- `um publish check --game <install>` before any release. Keep it non-commercial.
- Credit the engine/port you built on and disclose AI use honestly. Some recomp/decomp communities ban AI
  contributions — read the project's rules first. Publishers DMCA ports even without assets.

## Effort, honestly

Route 0 is minutes. Route 1 is days to weeks. A full matching decompilation can take **billions of tokens**
(see `skills/mod-any-game/references/engines/retro-decomp.md`). Route 4 is a project, not a session. Say
which one the user is actually signing up for before starting.

## It's working if

- Route 0 was ruled out with evidence, not assumed away.
- The plan names the exact engine/decomp/recomp and its current release, not a version from memory.
- The first milestone is a trivial thing on screen (a triangle, a menu, one room) before content work.
- It runs in PPSSPP *and* boots on hardware before you call it done.
- Nothing in the build or repo contains game data, extracted assets or retail offsets.
