# .llm — LLM Transcript Storage

Transcripts of LLM interactions that produced committed code or content are stored here, per Constitution §2.3 and P13.

## Naming convention

```
YYYY-MM-DD-<short-topic-slug>.md
```

Examples:
```
2026-07-21-git-workflow-setup.md
2026-08-03-gap-page-filters.md
```

## Required fields

Each transcript file must record:

| Field | Description |
|---|---|
| **Tool** | The LLM tool used (e.g. IBM Bob, Claude, ChatGPT) |
| **Date** | ISO date of the session (YYYY-MM-DD) |
| **Prompts** | The prompts or instructions given |
| **Responses** | The responses or code produced that was committed |
| **Files produced** | List of files created or modified as a result |

See `TEMPLATE.md` for a copy-paste starting point.

## Referencing a transcript in a commit

Add a line to the commit message body:

```
transcript: .llm/YYYY-MM-DD-<topic>.md
```

The `commit-msg` hook will remind you if you forget on commits that touch `.njk`, `.js`, or `_data/` files.
