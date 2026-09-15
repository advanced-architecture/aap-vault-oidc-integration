# TODO

Outstanding tasks and ideas that don't yet belong to an active plan sub-task.
When a task is picked up for implementation, move it into `.github/.agents/plan.md` as a full sub-task.

---

## Open tasks

- [x] **fix-bobignore** — manually add `*.env` to `.bobignore` to block local secret files from Bob's context. Bob cannot edit this file itself (self-protected). One line: `*.env        # local secret files — keep out of Bob context`

---

## Open tasks

- [ ] **secure-vault-token-logs** — `vault_token` appears in plain text in AAP job logs via the `X-Vault-Token` header on `ansible.builtin.uri` tasks in both Vault config playbooks. Add `no_log: true` to all `uri` tasks that include the token header, or investigate whether `uri` scrubs `X-Vault-Token` automatically when `url_password` no_log behaviour applies.
- [x] **aap-bootstrap-templates** — add tasks to `configure_aap_vault_oidc.yml` to create the AAP job templates (Vault config, HCP Vault config, demo, HCP demo) as part of the bootstrap, so they don't need to be created manually in the UI.

---

## Ideas / future work

- [ ] Cross-reference `ansible_oidc_vault` project plan.md in the daily workflow so Bob picks up the right plan for the right repo automatically.
- [ ] Add `scripts/smoke-test.sh` preflight once the `ansible_oidc_vault` scaffold is in place.
- [ ] Evaluate whether `.llm/` transcripts should be `.gitignore`d (tradeoff: keeps repo clean vs. loses provenance history).
- [x] Add a `requirements.yml` at the repo root declaring `community.hashi_vault >= 6.x` so EE builders and local runners have a machine-readable collection dependency.
- [x] Confirm `AAP_JWT_TOKEN` env var name against a live AAP 2.7 instance during validation-run; update playbook comments if the name differs. **Result:** `AAP_JWT_TOKEN` does not exist; AAP 2.7 OIDC workload identity uses the built-in `HashiCorp Vault Secret Lookup` credential type — not env var injection. Playbook architecture corrected; `examples/vault-secret-lookup/` added.
- [ ] **vault-sync-playbook** — Implement the `tasks/vault_sync_aap_project.yml` idempotent role-provisioning task (documented in `docs/vault-security-boundaries.md` §7) as a reusable task file or standalone playbook under `examples/vault-config/`.
- [ ] **templated-policy** — Add `aap-project-policy.hcl` (with `AUTH_JWT_ACCESSOR` templating) as a variable in `examples/vault-config/configure_vault_oidc.yml` so the Vault role uses a templated policy by default instead of a static one.
- [ ] **env-isolation-module** — Consider adding a Terraform module under `examples/terraform/` implementing the env-parameterised `vault_jwt_auth_backend_role` from `docs/vault-security-boundaries.md` §4 (dev/staging/prod glob pattern).

---

*Updated by Bob during session — see `.llm/` for transcript.*
