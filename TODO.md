# TODO

Outstanding tasks and ideas that don't yet belong to an active plan sub-task.
When a task is picked up for implementation, move it into `.github/.agents/plan.md` as a full sub-task.

---

## Open tasks

- [x] **fix-bobignore** — manually add `*.env` to `.bobignore` to block local secret files from Bob's context. Bob cannot edit this file itself (self-protected). One line: `*.env        # local secret files — keep out of Bob context`

---

## Open tasks

- [x] **secure-vault-token-logs** — superseded by **aap-credential-hide-token-uri** below, which addresses the same problem via the correct AAP-native mechanism (env injector + scrub list) rather than `no_log`.
- [x] **aap-admin-builtin-credential** — plan written to `.github/.agents/plan.md` (sub-task `aap-admin-builtin-credential`). Implementation pending: switch `configure_aap_vault_oidc.yml` and `configure_vault_secret_lookup.yml` from `{{ controller_* }}` extra_vars to `lookup('env', 'CONTROLLER_*')`; update `examples/aap-config/README.md` to document the built-in credential type instead of the custom one.
- [ ] **aap-credential-hide-token-uri** — `vault_token` appears in plain text in AAP job output inside `X-Vault-Token` headers on every `uri` task in `configure_vault_oidc.yml` and `configure_hcp_vault_oidc.yml`. **Root cause:** the existing "Vault Bootstrap Token" custom credential type uses an `extra_vars` injector — extra vars are Ansible variables, not env vars, and are NOT added to AAP's runtime scrub list. **Fix:** change the injector to `env: { VAULT_TOKEN: "{{ vault_token }}" }`. AAP adds every env var sourced from a credential with `secret: true` to the runner's `no_log` word list before job launch, so the token value is automatically replaced with `********` in all log output regardless of verbosity — including inside URI headers. The playbooks then read the value via `lookup('env', 'VAULT_TOKEN')` instead of `{{ vault_token }}`. Update: (1) the `Vault Bootstrap Token` credential type definition in `configure_aap_vault_oidc.yml` (change `extra_vars` → `env`); (2) every `X-Vault-Token: "{{ vault_token }}"` header in `configure_vault_oidc.yml` and `configure_hcp_vault_oidc.yml` (use `lookup('env', 'VAULT_TOKEN')`); (3) the guard task that checks `vault_token is defined`; (4) the inline comments in both playbooks describing the credential injector. Note: the built-in `HashiCorp Vault Secret Lookup` credential type is not applicable here — it is a plugin/lookup credential for runtime OIDC secret retrieval, not for injecting a bootstrap token into `uri` tasks.
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
