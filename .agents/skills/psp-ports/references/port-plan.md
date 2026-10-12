# PSP port plan template

Write this next to the mod working folder (or the repo root) and keep a `MODLOG.md` alongside it. Replace the
angle brackets; delete sections that do not apply.

```markdown
# <Game> → PSP port plan

- Game: <title>, <platform of origin>, <year>
- Owned / dumped by the user: <yes — how>
- Target: <real PSP model(s) | PPSSPP | both>
- Engine: <engine + how known>
- Source available: <open engine / decomp / recomp / none>
- Route: <0 emulation | 1 source port | 2 decompilation | 3 recompilation | 4 reimplementation>
- Why this route: <one line, and why the lighter routes were ruled out>
- Prior art: <existing PSP port / decomp / recomp, with URL and current release>

## Hardware budget
- RAM: <fits in 32 MB? need 64 MB model?>
- VRAM: <textures/framebuffers fit in 2 MB?>
- Renderer: <what replaces the original GPU path — pspgu/pspgl/software>
- Controls: <PC controls → PSP buttons + nub mapping>

## Milestones
1. Toolchain builds the pspdev sample; `EBOOT.PBP` runs in PPSSPP.
2. <trivial thing on screen: triangle / menu / one room>.
3. <first real subsystem: renderer / audio / input / save>.
4. <first playable slice> — boots on hardware.
5. <content complete for the stated scope>.

## Dependencies and tools
- pspdev <version>, psp-pacman packages: <...>
- Existing projects reused (credit + read their rules): <...>
- Assets: converted from the user's dump at install time; new art via fal. Nothing shipped.

## Risks
- <e.g. RDP→pspgu combine semantics; 2 MB VRAM; frame rate on 222 MHz>

## Publishing
- Ship: code / patch / converter only.
- Check: `um publish check --game <install>`.
- AI disclosure and credit: <...>
```
