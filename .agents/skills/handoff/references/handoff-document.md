# Handoff document template

Fill this in and write it to the OS temp directory. Delete any section that does not apply; keep it short.
Anything already written down goes in **Artifacts** as a path or URL, never copied.

```markdown
# Handoff: <the work in one line>

- Written: <date>, from <harness> in <directory>
- For: <what the next session is for, from the user's note>
- Resume with: read this file, then start at "Next actions"

## State
<What is in flight and why. The live thread, in a few sentences — not a log of the session.>

## Next actions
1. <the concrete first thing to do>
2. <then>
3. <then>

## Open questions / blockers
- <what is undecided or stuck, and what would unblock it>

## Facts (verified)
- <versions, paths, commands that actually worked>

## Assumptions (not verified)
- <things believed but not checked — the next agent should verify these, not trust them>

## Artifacts (referenced, not copied)
| What | Where |
|---|---|
| Plan | MODDING_PLAN.md |
| Journal | MODLOG.md |
| Field note | knowledge/games/<game>/<note>.md |
| Engine route | skills/mod-any-game/references/engines/<engine>.md |
| Code / commits / PR | <path or URL> |
| Logs / saves / screenshots | <path or URL> |

## Suggested skills
- `<skill>` — <why the next agent should reach for it>

## Redaction check
- [ ] No API keys, tokens, passwords or `.env` values
- [ ] No game files, extracted assets or decompiled code
- [ ] Unverified claims are under "Assumptions", not "Facts"
```
