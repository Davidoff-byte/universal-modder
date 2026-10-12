# PSP porting: routes, toolchain, testing

Deep reference for the `psp-ports` skill. Versions and project names move; check current releases rather than
trusting the names below.

## The hardware ceiling

| | PSP-1000 | PSP-2000/3000/Go/E1000 |
|---|---|---|
| CPU | 222–333 MHz MIPS32 R4000 ("Allegrex"), 2 cores (main + Media Engine) | same |
| RAM | 32 MB | 64 MB |
| VRAM | 2 MB (eDRAM) | 2 MB |
| GPU | 166 MHz fixed-function, **no vertex/pixel shaders** | same |
| Display | 480×272, 24-bit | 480×272 |
| Vector unit | VFPU (MIPS FPU extension; used heavily by games) | same |
| Media | UMD / Memory Stick PRO Duo | MS Pro Duo; Go is 16 GB flash + M2 |

What this means:

- The CPU is MIPS32 — the same ISA family as PS1, N64 and PS2. Decompiled/recompiled code from those
  consoles translates cleanly. This is why PSP is a good retarget for retro recomps.
- 32/64 MB is *generous* next to an N64 (4 MB) or PS1 (2 MB), so most retro data fits.
- 2 MB VRAM and a fixed-function GPU are the hard limits. Any renderer that assumes shaders, large
  framebuffers or modern texture formats must be rewritten or downgraded.

## Route 0 — emulation (change nothing)

Prefer this whenever it covers the game. Historically available on CFW/homebrew (names are examples, not a
verified current list):

- **Nintendo:** NES (FCEUltra/nesterJ-style), SNES (Snes9xTYL family), GB/GBC (MasterBoy/RIN), GBA (`gpSP`,
  TempGBA), N64 (DaedalusX64 — partial, many games imperfect).
- **Sega:** Genesis/Mega Drive + Sega CD + Master System (`PicoDrive`).
- **Sony:** PS1 via **POPS** (Sony's own PS1 emulator, run through CFW). PSP games run natively.
- **Arcade:** CPS1/CPS2 and Neo Geo ports of MAME.
- **ScummVM:** official PSP port — covers hundreds of point-and-click adventures.
- **DOSBox:** exists but slow; only simple DOS games.
- **RetroArch:** has a PSP port with a limited core set.

If the game is one of these, the "port" is: get the emulator onto the Memory Stick, add the user's own dump.

## Route 1 — source port / engine reimplementation

The sweet spot for PC games. Look for an open engine or a clean-room reimplementation, then port that.

Engineering notes:

- **SDL-based engines port well.** pspdev ships SDL 1.2 and SDL2 plus SDL_image/mixer/ttf, zlib, libpng,
  freetype, vorbis/opus; a project already on SDL often builds with the `psp-cmake` toolchain and light
  platform glue.
- **Filesystem:** the Memory Stick path is `ms0:/` (and `ef0:/` on the Go). Rewrite hardcoded absolute paths
  and PC save locations.
- **Input:** the PSP has a d-pad, analog nub, four face buttons, L/R, Start/Select — map PC mouse/keyboard
  controls down to that; add a virtual cursor where a game needs a mouse.
- **Audio:** `pspaudio`/SDL audio; reduce mixing rate and channel count.
- **Memory:** watch the 2 MB VRAM; downscale textures and framebuffers to 480×272.

Known PSP source ports to point at (verify current): Doom/Quake/Duke Nukem 3D/Wolfenstein/Heretic/Hexen/
Descent family, **OpenLara** (Tomb Raider 1–2), **DevilutionX** (Diablo 1), **OpenTTD**, **CorsixTH**
(Theme Hospital), **Cannonball** (OutRun), **Commander Genius** (Keen), **REminiscence** (Flashback),
**OpenTyrian**, **Aleph One** (Marathon). Community ports of the **Super Mario 64** decomp exist; the
Ocarina of Time decomp (`Ship of Harkinian`) has community retarget efforts.

## Route 2 — decompilation, cross-compiled to MIPS

Match a decompilation to the original ROM, then cross-compile its C to PSP.

- The original consoles are MIPS (N64, PS1), so the logic port is mostly mechanical. Swap the platform layer:
  - N64 `libultra`/RSP/RDP → PSPSDK + `pspgu` (or `pspgl`);
  - PS1 libgs/libpad/libcd → PSPSDK equivalents;
  - audio, timers, controllers, CD/file access → PSP.
- Replace asset loading with a converter that runs on the **user's own dump** (never commit assets).
- Expect renderer work: N64 RDP combines/dispatchers and PS1 GPU semi-transparency do not map one-to-one to
  `pspgu`. Render to the 480×272 target and budget for the 2 MB VRAM.
- Decompilation itself is the expensive part; see the agent loop and budget in
  `skills/mod-any-game/references/engines/retro-decomp.md` (a large decomp can take billions of tokens).

## Route 3 — static recompilation

Recompilers turn machine code into C linked against a runtime that reimplements the console:

- N64: N64Recomp + N64ModernRuntime (RT64 renderer)
- PS2: PS2Recomp
- PS1: psxrecomp / RecompOne
- GameCube/Wii: DolRecomp

Because the source is MIPS and PSP is MIPS, instruction translation is favorable. The work is in the
**runtime**: the graphics runtime (RDP/RSP, PS2 GS, PS1 GPU) must be rewritten for `pspgu`, and the OS/audio
runtime for PSPSDK. Treat the recomp as an oracle for a clean reimplementation when the runtime port stalls.

## Route 4 — clean reimplementation / demake

`mashup-mods` Pattern 4: write an idiomatic engine for PSP that reads the user's own game files. Reuse
`asset-pipeline` to re-fit art to 480×272 and `fal-assets` for anything you cannot ship. This is the most
work and the most control; it is realistic for 2D and simple games, not for a full 3D AAA title.

## Toolchain

```sh
# Install (or use the docker image pspdev/pspdev:latest)
# Prebuilt releases: https://github.com/pspdev/pspdev/releases
export PSPDEV=~/pspdev
export PATH="$PATH:$PSPDEV/bin"

# Library management
psp-pacman -S <pkg>            # install ported libraries/packages
```

Minimal `psp-cmake` build:

```cmake
cmake_minimum_required(VERSION 3.13)
project(hello C)
include($ENV{PSPDEV}/psp/psp.cmake)   # provides the toolchain + EBOOT rules
add_executable(hello main.c)
target_link_libraries(hello pspgu pspdebug)
```

Result: `EBOOT.PBP`. Signing (`ebootsigner`) helps on some CFW; modern CFW runs unsigned homebrew.

Structure of a homebrew app on the stick:

```
ms0:/PSP/GAME/<title>/
  EBOOT.PBP
  assets/...
```

## Test loop

1. **PPSSPP** (PC/phone): fastest iteration. Run, savestate, screenshot, inspect memory. Automate it like
   any windowed PC app per the `game-automation` skill (fixed window, screenshots, input only while focused).
2. **Real hardware:** CFW/HEN. Use **`psplinkusb`** — `usbhostfs` + `pspsh` for a shell and file transfer,
   and `psp-gdb` for debugging. Read the crash/exception log rather than guessing.
3. If 32 MB is a target, test on a PSP-1000-or-equivalent RAM profile, not just PPSSPP defaults.

## Assets, legality, publishing

- The user must own the game and dump it themselves. **Never download ROMs or ISOs**, and never build on
  leaked source.
- Ship **code, patches, diffs and converters**; never game files, extracted assets or decompiled dumps.
  `um publish check --game <install>` before release.
- Read each recomp/decomp project's contribution rules — some ban AI-generated work. Disclose AI use.
- Publishers issue takedowns for ports even without assets (there have been real cases). Keep it
  non-commercial and don't monetize.

## Effort budget

| Route | Typical effort |
|---|---|
| 0 Emulation | minutes to hours |
| 1 Source port | days to weeks |
| 2 Decompilation | weeks to months; a full decomp is billions of tokens |
| 3 Static recomp | weeks to months, dominated by the GPU/OS runtime |
| 4 Reimplementation | a project; scope to 2D/simple games |
