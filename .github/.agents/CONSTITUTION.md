# Engineering Constitution — Solo Static Site

> A streamlined adaptation of the [full Engineering Constitution](CONSTITUTION.md) for a single developer working on a static site with LLM assistance.  
> The full constitution remains the authoritative reference. This document removes rules that only apply to teams, CI pipelines, or backend systems, and relaxes processes that would create overhead without benefit for a solo project.
---

## Table of Contents

1. [Principles](#1-principles)
2. [Git Workflow](#2-git-workflow)
3. [Definition of Done](#3-definition-of-done)
4. [Code Quality & Testing](#4-code-quality--testing)
5. [Enforcement](#5-enforcement)
6. [Amendments](#6-amendments)

---

## 1. Principles

When rules conflict or a situation is not covered, apply these principles.

| # | Principle | What it means in practice |
|---|-----------|--------------------------|
| P1 | **Clarity over cleverness** | Code is read far more than it is written. Prefer clear, human-readable solutions. |
| P2 | **Small, incremental steps** | Prefer many small, isolated changes over large sweeping ones. |
| P4 | **Do not change what works** | Only make changes that fix a bug, add a feature, or demonstrably improve readability, performance, or maintainability. |
| P5 | **Automate the boring** | If a check can be run by a script, script it. Write deterministic scripts for repeated workflows. |
| P6 | **Fail fast, loudly** | Defects must surface as early as possible — in the editor or a local build, not after deploy. |
| P7 | **Prefer existing over new** | Use a proven library, theme, or tool before writing new code. Document the justification in a commit when you choose to build instead. |
| P9 | **Cite sources and credit authors** | Any code derived or copied from an external source (docs, blogs, Stack Overflow, AI) must have the source URL and author credited in a comment at the point of use. |
| P10 | **Manage licenses actively** | Before adding a dependency or copying external code: identify its license and verify it is compatible with your project. Keep a simple `LICENSES.md` inventory. |
| P11 | **Test first where practical** | For logic-bearing code, write a failing test before writing implementation. For purely presentational markup, a visual check is sufficient. |
| P12 | **Script common workflows** | Any workflow performed more than twice must be scripted and committed to the repository. Build, deploy, and lint must never rely on manual steps. |
| P13 | **Record LLM Contribution** | Keep detailed records of LLM interactions used to produce code or content. Save transcripts alongside the contribution — commit them to `.llm/` or link them in the relevant commit message. |
| P14 | **Exclude or anonymize customer information** | No customer names, identifiers, contact details, or any data that could identify a customer may be committed to the repository, rendered in the portal, or included in any data file, template, transcript, or test fixture. Use anonymized placeholders (e.g. `customer-a`, `acme-example`) in all examples and seed data. |
| P15 | **Preserve existing data** | Data is our most precious asset. Scripts and tools must never silently overwrite or delete data that was authored by a human. Carry forward existing values on re-runs; use additive operations (append, merge) over destructive ones (overwrite, truncate); require explicit confirmation or a dedicated flag before any destructive operation. |
| P16 | **Use the local clock for dates** | Never infer the current date from training data or conversation context. Always run `date +%Y-%m-%d` to get today's date before naming files, writing timestamps, or filling date fields. |

> **P3 note:** You are the sole owner, but you are still responsible for the end user experience.  
> **P8 note:** Prefer established patterns (12-factor config, component-based structure). If you deviate, note the reason in a commit message rather than a formal ADR.

---

## 2. Git Workflow

### 2.1 Branch Model

Use a simple trunk model. Direct commits to `main` are acceptable for trivial changes (typos, content updates). Use a branch for anything that involves logic, structure, or dependency changes.

```
main          ← deployable at all times
  └── feature/<short-description>
  └── fix/<short-description>
  └── chore/<short-description>
```

- There are no ticket ID requirements — use a short descriptive slug.
- Branches should be short-lived. Merge or delete within a day or two.

### 2.2 Commit Conventions

Use **Conventional Commits** format (`conventionalcommits.org`):

```
<type>(<optional scope>): <short imperative summary>

[optional body — explain WHY, not WHAT]
```

- Subject line ≤ 72 characters, present tense, no trailing period.
- If LLM tools were used, note it in the commit body with a transcript reference per P13: e.g., `Generated with Claude — transcript: .llm/2024-01-15-nav-refactor.md`
- Each commit should be **atomic** — the site builds on its own at every commit.
- When working from a plan, **commit at the end of each sub-task** before starting the next. Each sub-task's todo list includes the commit message to use as its final step.

### 2.3 LLM Artifact Storage

Store LLM transcripts in a `.llm/` directory at the repository root. Name files by date and topic:

```
.llm/
  2024-01-15-navigation-refactor.md
  2024-02-03-seo-meta-tags.md
```

Each transcript file should record: the tool used, the date, the prompt(s), and the response(s) that produced committed code or content.

### 2.4 Working with Bob

See [`WORKFLOW.md`](../WORKFLOW.md) for the step-by-step guide to the daily commit loop, working with Bob, recording LLM transcripts, and what the pre-commit hook checks.

---

## 3. Definition of Done

A piece of work is **Done** only when **all** of the following are true:

- [ ] The goal you set out to achieve is met and works in a local build.
- [ ] The site builds without errors (`lint`, `build` scripts pass).
- [ ] Relevant documentation (README, inline comments) has been updated if the change affects how someone would understand or run the project.
- [ ] All new or modified dependencies are recorded in `LICENSES.md` (see P10).
- [ ] If LLM tools were used, the interaction transcript is saved in `.llm/` and referenced in the commit message per P13.
- [ ] The change is committed to `main` (directly or via merged branch) and the site deploys successfully.

> **Open questions — Definition of Done**
> - Should visual changes require a screenshot saved alongside the commit or LLM transcript?
> - Is a successful local build sufficient, or should a staging deploy also be required before marking work done?
> - What counts as "significant content change" that triggers a link check — every page edit, or only structural/navigation changes?
> - Should the DoD checklist be enforced via a git commit template or PR description template to make it harder to skip?

---

## 4. Code Quality & Testing

### 4.1 Local Quality Checks

Run these before every commit (automate via pre-commit hook where possible):

| Check | Rule |
|---|---|
| **Formatting** | All files pass the configured formatter. |
| **Linting** | Zero lint errors. |
| **Build** | The site builds successfully (`npm run build` or equivalent). |
| **Secrets scan** | No API keys, tokens, or credentials committed. |

### 4.2 Testing

For a static site, testing is lighter than a full application:

| Type | When required |
|---|---|
| **Logic unit tests** | Required for any JavaScript/TypeScript functions that compute or transform data. |
| **Visual check** | Required for any layout or styling change — open the page in a browser before committing. |
| **Link check** | Run a link checker before publishing significant content changes. |

- Tests must assert **behavior**, not implementation details.
- No `sleep` or arbitrary delays in any test or build script.

### 4.3 Code Health

- **Formatter:** Prettier (`npm run format`). Config in `.prettierrc`. Runs on `*.js` and `*.json`; YAML and Nunjucks files are excluded (hand-authored style preserved).
- **Linter:** ESLint (`npm run lint`). Config in `eslint.config.js`. Root files use ESM; `scripts/` uses CommonJS.
- **No TODO without a note.** `TODO` and `FIXME` comments must include a brief description of what needs to be done and when (a ticket ID is not required, but a reason is).
- **No dead code.** Unused files, components, or styles must be deleted.
- **No magic numbers or strings.** Non-obvious literals should be named constants.
- **File length** should stay under 300 lines for template/component files. Split when a file becomes hard to navigate.

> **Open questions — Code Quality & Coding Conventions**
> - Are there Nunjucks/template-specific linting tools worth adopting for `.njk` files?
> - Should `_data/` files have a defined schema (e.g., JSON Schema or Zod) to catch malformed content early?
> - What is the rule for CSS organisation — utility classes, BEM, something else? Should a style guide section be added here?
> - Is 300 lines the right file-length limit for this project's template complexity, or should it be adjusted?
> - Should there be a naming convention for template partials, data files, and layout files (e.g., kebab-case everywhere)?

---

## 5. Enforcement

For a solo project, enforcement is self-imposed. The goal is to make good habits automatic.

| Layer | What is enforced | How |
|---|---|---|
| **Bob Mode** | Planning vs. implementation separation | Bob Plan mode for task breakdown and design; Bob Agent mode for all code changes |
| **Editor** | Formatting, lint, type errors | ESLint + Prettier (configs committed) |
| **Pre-commit hook** | Format check (staged files only), lint, build, secrets scan | Plain shell script — `scripts/hooks/pre-commit.sh`, installed via `npm install` |
| **commit-msg hook** | LLM transcript reminder | `scripts/hooks/commit-msg.sh` — non-blocking reminder on `.njk`/`.js`/`_data/` commits |
| **Self-review** | Correctness, LLM disclosure, DoD checklist | Read your own diff before merging or pushing to `main` |
| **Deploy check** | Site builds and renders correctly | Verify the live site after every deploy |

When you catch yourself skipping a check, script it so you cannot skip it again (P5, P12).

> **Open questions — Git Workflow & Conventions**
> - Should branch names follow a stricter pattern (e.g., include a date prefix) for easier history navigation?
> - When collaborators are added, should squash-merge or merge commits be the standard strategy?

---

## 6. Amendments

This is your document. Amend it freely as the project evolves.

- Record the change and your rationale in the commit message.
- If a rule stops making sense for your project, remove it — but note why.
- If you start collaborating with others, revisit the [full constitution](CONSTITUTION.md) and re-introduce the sections that apply.

---

*Last updated: see `git log -- constitution-simple.md`*
