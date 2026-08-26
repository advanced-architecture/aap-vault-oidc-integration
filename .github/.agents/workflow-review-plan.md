---
name: Agentic Workflow Review & Streamlining
overview: >
  Assess the current `.github/.agents/` workflow for Bob compatibility gaps and
  friction points, then apply targeted improvements so every Bob session starts,
  executes, and closes with zero ambiguity. Changes are additive and non-destructive.
todos:
  - id: align-plan-format
    content: >
      Migrate plan.md from mixed frontmatter+prose to Bob's native plan schema
      (subtask blocks with Intent / Expected Outcomes / Todo List / Relevant
      Context / Status) so Bob's built-in plan-tracking can operate it directly.
    status: pending
  - id: fix-bobignore
    content: >
      Add a *.env pattern to .bobignore to block config/.env secret files from
      Bob's context. Do NOT list .llm/ — bobignore blocks read_file and
      write_file entirely, which would prevent Bob from writing transcripts.
      Transcripts stay readable/writable; sensitive files stay blocked.
    status: pending
  - id: create-agents-root-symlink
    content: >
      Create a root-level AGENTS.md symlink pointing to
      .github/.agents/agents.md so Bob's automatic AGENTS.md auto-load picks up
      the 4-step cycle without manual file navigation.
    status: pending
  - id: add-session-opener
    content: >
      Add a "Session Start Checklist" section to agents.md with exact Bob tool
      calls: run date, read plan.md, run git status — removing guesswork about
      how Step 1 is performed.
    status: pending
  - id: add-transcript-automation-note
    content: >
      Document in agents.md that Bob should use write_file (not a shell command)
      to create .llm/ transcripts, since Bob's file tools do not require
      additional user permissions.
    status: pending
  - id: update-constitution-dod
    content: >
      Add a "Bob Mode" row to the Enforcement Layers table in CONSTITUTION.md
      confirming that Bob Plan mode is used for planning and Bob Agent mode for
      implementation, aligning the constitution with Bob's actual mode system.
    status: pending
isProject: false
---

# Agentic Workflow Review & Streamlining Plan

## Overview

This plan captures **targeted, minimal changes** to `.github/.agents/` that remove
Bob-compatibility friction discovered during the workflow review. No existing
content is removed; all changes are additive refinements.

---

## Review Findings

### What Is Working Well

| Area | Evidence |
|---|---|
| Governance files exist | `CONSTITUTION.md`, `agents.md`, `bob-instructions.md`, `plan.md` all present |
| 4-step daily cycle | `agents.md` §2 defines Plan → Execute → Transcript → DoD clearly |
| LLM transcript system | `.llm/` directory, `TEMPLATE.md`, and first session recorded |
| Secrets discipline | P14 enforced; all examples use placeholder domains |
| Data preservation rule | P15 enforced; additive-only operations required |
| Conventional Commits | Commit format with `Transcript:` footer defined |
| `.bobignore` present | `bob-task*` excluded |
| Security-first examples | `no_log: true` on all credential tasks |

### Compatibility Gaps & Friction Points

| Gap | Impact | Target Sub-task |
|---|---|---|
| `plan.md` uses ad-hoc YAML frontmatter, not Bob's subtask schema | Bob cannot track task state natively; must parse prose table | `align-plan-format` |
| `.bobignore` only excludes `bob-task*` | `*.env` files (secrets) could leak into context; `.llm/` must stay accessible so Bob can write transcripts | `fix-bobignore` |
| No root-level `AGENTS.md` confirmed | Bob's auto-load of `AGENTS.md` may not trigger; user must manually navigate to `.github/.agents/agents.md` | `create-agents-root-symlink` |
| Step 1 says "run `date`" but does not specify Bob tool | Bob must decide whether to use `execute_command` or `read_file`; creates ambiguity | `add-session-opener` |
| Step 3 says "write transcript" but does not specify Bob tool | Bob might reach for shell tools unnecessarily | `add-transcript-automation-note` |
| `CONSTITUTION.md` Enforcement Layers does not mention Bob modes | Bob Plan / Bob Agent mode distinction is undocumented; agent might use wrong mode | `update-constitution-dod` |

---

## Sub-Tasks

---

### Sub-task 1: `align-plan-format`

**Intent:** Rewrite `plan.md` to use Bob's native subtask schema (Intent / Expected
Outcomes / Todo List / Relevant Context / Status blocks) so Bob can read, update,
and reason over the task list without parsing a custom prose format.

**Expected Outcomes:**
- `plan.md` retains all existing scope decisions, architecture details, and code examples.
- Each of the 6 original tasks (`scaffold-repo`, `vault-scripts`, `scqa-readme`,
  `aap-docs`, `example-playbook`, `validation-run`) appears as a properly structured
  subtask block.
- Bob can toggle `status: pending → done` using a single `search_and_replace` call.

**Todo List:**
1. Read the current `plan.md` in full.
2. Keep the YAML frontmatter but replace the inline `todos` list with expanded subtask
   blocks directly in the body.
3. For each of the 6 tasks, write: **Intent**, **Expected Outcomes**, **Todo List**
   (ordered steps), **Relevant Context** (file paths), **Status**.
4. Preserve all prose sections (architecture, code examples, validation plan, out-of-scope).
5. Verify the file is under 300 lines or split prose into a separate `plan-context.md`.

**Relevant Context:**
- [`/.github/.agents/plan.md`](.github/.agents/plan.md) — current plan file
- Bob `create-plan` skill — defines the subtask schema target

**Status:** `[ ] pending`

---

### Sub-task 2: `fix-bobignore`

**Intent:** Add `*.env` to `.bobignore` to block secret-bearing env files from
Bob's context. Do **not** list `.llm/` — per Bob docs, `bobignore` fully blocks
`read_file` and `write_file` on matched paths, which would prevent Bob from
writing transcripts entirely. The `.llm/` directory must remain accessible.

**Expected Outcomes:**
- `.bobignore` contains: `bob-task*` and `*.env`.
- `config/*.env` and any local `.env` files are excluded from Bob's context.
- `.llm/` transcripts remain readable and writable by Bob.

**Todo List:**
1. Read current `.bobignore` (currently one line: `bob-task*`).
2. Append `*.env` with an inline comment explaining the rationale.
3. Confirm no legitimate non-secret source files match `*.env`.

**Relevant Context:**
- [`.bobignore`](.bobignore) — current file

**Status:** `[ ] pending`

---

### Sub-task 3: `create-agents-root-symlink`

**Intent:** Create a root-level `AGENTS.md` as a **symlink** to
`.github/.agents/agents.md` so Bob's automatic `AGENTS.md` auto-load picks up
the full 4-step daily cycle at session start. A symlink keeps a single source
of truth — editing `.github/.agents/agents.md` automatically updates what Bob
sees, with no duplicated content to maintain.

**Expected Outcomes:**
- `AGENTS.md` at repo root resolves as a symlink to `.github/.agents/agents.md`.
- Bob auto-loads the 4-step cycle at every new session without manual navigation.
- No content duplication between root and `.github/.agents/`.

**Todo List:**
1. Confirm `AGENTS.md` does not already exist at repo root.
2. Run `ln -s .github/.agents/agents.md AGENTS.md` from repo root.
3. Verify the symlink resolves: `ls -la AGENTS.md`.
4. Stage the symlink: `git add AGENTS.md`.

**Relevant Context:**
- [`.github/.agents/agents.md`](.github/.agents/agents.md) — symlink target (source of truth)
- Bob AGENTS.md auto-load behavior (loads root `AGENTS.md` automatically at session start)

**Status:** `[ ] pending`

---

### Sub-task 4: `add-session-opener`

**Intent:** Add a concrete "Session Start Checklist" to `agents.md` that specifies
the exact Bob tool to use for each Step 1 action, eliminating ambiguity about
whether to use `execute_command` or file tools.

**Expected Outcomes:**
- `agents.md` §2 Step 1 includes a numbered checklist with Bob tool names in backticks.
- Example: "Use `execute_command` with `date +%Y-%m-%d`", "Use `read_file` on
  `.github/.agents/plan.md`", "Use `execute_command` with `git status`".

**Todo List:**
1. Read `agents.md` §2 Step 1.
2. Insert a "Bob Tool Reference" aside beneath each step bullet.
3. Ensure no existing instructions are removed.

**Relevant Context:**
- [`.github/.agents/agents.md`](.github/.agents/agents.md:31) — Step 1 section (lines ~31–33)

**Status:** `[ ] pending`

---

### Sub-task 5: `add-transcript-automation-note`

**Intent:** Document in `agents.md` Step 3 that Bob should use `write_file` (not
`execute_command`) to create `.llm/` transcript files, since Bob's native file
tools require no additional shell permissions.

**Expected Outcomes:**
- Step 3 of `agents.md` contains a "Bob Tool Reference" note: "Use `write_file`
  with full transcript content. Do not use shell `cat` or `tee` commands."

**Todo List:**
1. Read `agents.md` §2 Step 3.
2. Append the tool reference note inline below the existing bullet.

**Relevant Context:**
- [`.github/.agents/agents.md`](.github/.agents/agents.md:42) — Step 3 section (lines ~42–44)

**Status:** `[ ] pending`

---

### Sub-task 6: `update-constitution-dod`

**Intent:** Add a "Bob Mode" row to the Enforcement Layers table in `CONSTITUTION.md`
confirming that Bob Plan mode is used for planning and Bob Agent mode for
implementation, so the constitution remains the authoritative reference for all
layers of quality control.

**Expected Outcomes:**
- The Enforcement Layers table in `CONSTITUTION.md` includes a row for "Bob Mode"
  with description "Plan mode for task planning; Agent mode for implementation".

**Todo List:**
1. Read the Enforcement Layers table in `CONSTITUTION.md`.
2. Add a new row: `| Bob Mode | Planning vs. implementation separation | Bob Plan mode → Bob Agent mode |`.
3. Do not modify any other rows or surrounding prose.

**Relevant Context:**
- [`.github/.agents/CONSTITUTION.md`](.github/.agents/CONSTITUTION.md) — Enforcement Layers table

**Status:** `[ ] pending`

---

## Recommended Daily Workflow (After Changes)

```
Session Start
  → Bob auto-loads AGENTS.md (root)
  → execute_command: date +%Y-%m-%d
  → read_file: .github/.agents/plan.md
  → execute_command: git status

Pick Next Pending Sub-task
  → Research relevant files (read_file / grep / FindSymbol)
  → Implement minimal change (apply_diff / write_file)

Record Transcript
  → write_file: .llm/YYYY-MM-DD-<topic>.md

DoD Check + Commit
  → No secrets, no hardcoded values, no TODOs
  → apply_diff: plan.md status pending → done
  → execute_command: git commit
```
