---
name: handoff
description: Write the current session to one portable markdown file so a fresh agent can pick the work up. Use when the user types /handoff or asks to hand off, transfer, or continue this session elsewhere — a different harness, directory, machine, colleague, or a forked side task. Not a summary for yourself: if nothing is travelling, keep the session or compact instead.
---

# Handoff

Compact this conversation into **one markdown file** that a fresh agent reads to continue the work. The
value is **portability, not compression**: you need a file only when the work has to *travel*.

The user triggers this (`/handoff <note>`); the note says what the next session is *for*, and you write the
document for that. Do not fire it unbidden.

## Is this the right move?

| Situation | Handoff? | Why |
|---|---|---|
| Swapping harness (Claude → Codex → OpenCode) | **Yes** | The new harness cannot see this context. |
| Moving to a different directory, repo or machine | **Yes** | The new agent has no session to inherit. |
| Sending the work to a colleague | **Yes** | They need something they can read cold. |
| Forking a side task mid-work | **Yes** | You keep going; a second agent takes the fork. |
| Same task, same place, you stay in the loop | No | Continue, or compact. |
| Everything behind you is disposable | No | Clear. |

All of these turn the conversation (a primary source) into a summary (a secondary source). Continuing is the
only move that doesn't, so rule it out first.

## Where the file goes

The OS **temporary directory**, never the workspace — a handoff is a transit document, not an artifact to
maintain. Find it rather than assuming:

- Windows: `%TEMP%`. In OpenCode that is usually `%LOCALAPPDATA%\Temp\opencode`.
- macOS / Linux: `$TMPDIR` (often `/private/tmp`), else `/tmp`.

Windows paths are long and differ per machine; expect to try more than one. **Report the full path back to
the user** so they can keep it before moving on.

Temp is volatile. If the next session is not starting within the hour, or starts under a different harness
that may clear temp, also write a durable copy (the repo's scratch area, or the mod working folder) and
point at that instead. The same goes for anything the document *points at*: a reference to a temp file is a
reference the next agent cannot follow.

## Build it

1. **Locate temp** and pick a name (`handoff-<slug>.md`). If the destination is durable, choose it now.
2. **Write down everything already on disk** — never copy it:
   - plan → `MODDING_PLAN.md`
   - journal → `MODLOG.md`
   - safety rules → `skills/mod-any-game/references/safety.md`
   - engine route → `skills/mod-any-game/references/engines/<engine>.md`
   - field notes → `knowledge/...` (and `um kb search`)
   - issues, PRs, commits, diffs → path or URL
   Referencing by path keeps the file small and keeps settled detail in one place instead of two that drift.
3. **Write the live thread**: what is in flight, why, and the next concrete action. Use the template in
   [`references/handoff-document.md`](references/handoff-document.md).
4. **Name the suggested skills** the next agent should reach for (see below).
5. **Redact secrets before writing.** No API keys, tokens, passwords, `.env` values, or `FAL_KEY`. No
   decompiled dumps or game files — those must never leave the machine anyway.
6. **Read it back once and downgrade anything you only assumed.** "X isn't built", "Y is done" written as
   fact becomes a false premise for everything after it. The next agent treats this document as a contract
   and will not re-check it.
7. **Give the user the path**, and how to use it: open the fresh session and point it at the file.

## Suggested skills to name

Pick only what the remaining work needs, and say why for each. Candidates from this toolkit:
`mod-any-game` (the loop), `game-recon`, `reverse-engineering`, `gamedb-cli` (navigate a decompiled tree),
`fal-assets` / `asset-pipeline` (art), `game-automation` (drive and screenshot the game), `mashup-mods`,
`showcase-video`, `publish-mod`, `share-field-notes` (write the field note and open the PR).

## Handing it over

Point the new session at the **path**; do not paste the summary into a command. A summary containing
backticks or `$(...)` is silently truncated when interpolated into `claude "<summary>"`, so the new agent
starts with a quietly incomplete brief. In OpenCode, just name the file and ask the agent to read it.

## It's working if

- The document is a small fraction of the conversation, and plans, issues and diffs appear as paths and
  URLs rather than copied text.
- You can read it cold, with the original session closed, and know the next action.
- The fresh agent starts working instead of asking the user to re-explain the setup.
- In the fork case, the original session is still untouched when the user returns to it.
- The suggested-skills section names the skill you would have reached for yourself.
- Nothing in it is a key, a token, or a password.
