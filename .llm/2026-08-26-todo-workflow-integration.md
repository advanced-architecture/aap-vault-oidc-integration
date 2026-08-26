**Tool:** IBM Bob (Agent Mode)
**Date:** 2026-08-26

## Prompts

> Create a document in this project as an ongoing todo list. Update the workflow to add to this todo list when outstanding or new tasks are identified.

## Commands Run

```
date +%Y-%m-%d
# → 2026-08-26
```

## Responses / Code Produced

### TODO.md — created
Root-level `TODO.md` with two sections:
- **Open tasks** — immediate actionable items (seed: fix-bobignore manual step carried from previous session)
- **Ideas / future work** — lower-priority items not yet scoped into a plan

### agents.md — modified (two additions)
1. **Step 1, item 3** — added "Read Open Tasks: read `TODO.md`" with `read_file` Bob tool callout; renumbered existing step 3 → 4.
2. **Step 4 DoD** — added "Capture new work" checklist item instructing Bob to append to `TODO.md` using `apply_diff` whenever new tasks surface during a session.

## Files Produced or Modified

- `TODO.md` — created (new)
- `.github/.agents/agents.md` — modified (Step 1 + Step 4)
- `.llm/2026-08-26-todo-workflow-integration.md` — created (this file)

## Commit Reference

`chore(agents): add TODO.md and wire into daily workflow`
