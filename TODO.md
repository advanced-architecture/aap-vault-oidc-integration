# TODO

Outstanding tasks and ideas that don't yet belong to an active plan sub-task.
When a task is picked up for implementation, move it into `.github/.agents/plan.md` as a full sub-task.

---

## Open tasks

- [ ] **fix-bobignore** — manually add `*.env` to `.bobignore` to block local secret files from Bob's context. Bob cannot edit this file itself (self-protected). One line: `*.env        # local secret files — keep out of Bob context`

---

## Ideas / future work

- [ ] Cross-reference `ansible_oidc_vault` project plan.md in the daily workflow so Bob picks up the right plan for the right repo automatically.
- [ ] Add `scripts/smoke-test.sh` preflight once the `ansible_oidc_vault` scaffold is in place.
- [ ] Evaluate whether `.llm/` transcripts should be `.gitignore`d (tradeoff: keeps repo clean vs. loses provenance history).
- [ ] Add a `requirements.yml` at the repo root declaring `community.hashi_vault >= 6.x` so EE builders and local runners have a machine-readable collection dependency.
- [ ] Confirm `AAP_JWT_TOKEN` env var name against a live AAP 2.7 instance during validation-run; update playbook comments if the name differs.

---

*Updated by Bob during session — see `.llm/` for transcript.*
