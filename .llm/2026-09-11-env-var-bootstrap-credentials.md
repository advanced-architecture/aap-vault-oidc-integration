# LLM Transcript — 2026-09-11 — env-var-bootstrap-credentials

**Date:** 2026-09-11
**Engine:** Claude (IBM Bob)
**Topic:** Switch config playbook bootstrap credentials from AAP custom credential types to environment variables

---

## Context

Review of the project identified that the three config playbooks (`vault-config`, `hcp-vault-config`, `aap-config`) were designed around AAP custom credential type injection (Sub-Task 11). The decision was made to simplify: bootstrap credentials are sourced from environment variables instead, making the playbooks runnable locally or from CI without needing AAP pre-configured.

## Changes Made

### `examples/vault-config/configure_vault_oidc.yml`
- Added `connection: local`
- Added `vars:` block: `vault_token: "{{ lookup('env', 'VAULT_TOKEN') }}"`
- Added preflight `fail` task if `VAULT_TOKEN` is not set

### `examples/hcp-vault-config/configure_hcp_vault_oidc.yml`
- Added `connection: local`
- Added `vars:` block: `vault_token: "{{ lookup('env', 'VAULT_TOKEN') }}"`
- Added preflight `fail` task if `VAULT_TOKEN` is not set

### `examples/aap-config/configure_aap_vault_oidc.yml`
- Replaced `tower_host/username/password/verify_ssl` with `CONTROLLER_HOST/USERNAME/PASSWORD/VERIFY_SSL` env var lookups
- Added preflight `fail` loop over the three required `CONTROLLER_*` vars
- Removed Tasks 3–5 (`Vault Bootstrap Token`, `HCP Vault Bootstrap Token`, `AAP Admin Credential` credential type creation) — no longer needed
- Fixed credential type ID lookup to be idempotent: GET on 400 to retrieve existing type ID
- Made credential instance creation idempotent with 400 guard
- Fixed duplicate "Task 2" comment header (renumbered to Task 3)

### `vars.yml` comment blocks
- `examples/vault-config/vars.yml`: updated comment to reference `VAULT_TOKEN` env var
- `examples/hcp-vault-config/vars.yml`: updated comment to reference `VAULT_TOKEN` env var
- `examples/aap-config/vars.yml`: updated comment block to reference `CONTROLLER_*` env vars

### READMEs
- `examples/vault-config/README.md`: replaced AAP credential attachment instructions with `export VAULT_TOKEN=...` + `ansible-playbook` command; fixed stale `s.CHANGEME` in variables table
- `examples/hcp-vault-config/README.md`: same pattern; fixed stale `CHANGEME` in variables table
- `examples/aap-config/README.md`: replaced credential attachment instructions with `export CONTROLLER_*=...` + `ansible-playbook` command; removed mention of three now-deleted credential types

### `demo.md`
- Step 2 (Configure AAP): removed credential attachment; updated Verify checklist to only list `HashiCorp Vault JWT` type
- Step 3 Path A: replaced credential attachment with `VAULT_TOKEN` env var note
- Step 3 Path B: replaced credential attachment with `VAULT_TOKEN` env var note

## Commit
`refactor(config): switch bootstrap credentials to environment variables`
