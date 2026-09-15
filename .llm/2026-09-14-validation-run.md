# LLM Session Transcript — Validation Run

**Date:** 2026-09-14  
**Model:** Claude (IBM Bob)  
**Topic:** Full validation run of AAP + HCP Vault OIDC integration pattern

---

## Summary

Executed the full `validation-run` plan sub-task against a live AAP 2.7 instance (OCP-hosted, platform-gateway) and an HCP Vault Dedicated cluster. Iterated on the playbooks in real time based on live failure output.

---

## Key findings and fixes during validation

### API path correction
- `/api/v2/` does not exist on AAP 2.7 platform-gateway deployments
- Correct base path: `/api/controller/v2/`
- Updated `aap_api_base` var in `configure_aap_vault_oidc.yml`

### Credential type `kind` field required
- POST to `/api/controller/v2/credential_types/` requires `kind: "cloud"` in the body
- Was missing after an external edit; restored

### Credential type idempotency
- Original code used a conditional `set_fact` based on POST status — unreliable due to Jinja2 integer comparison issues
- Fixed by always doing a GET lookup after the POST and capturing the ID from the GET result

### Credential POST requires `organization`
- `/api/controller/v2/credentials/` POST requires `user`, `team`, or `organization`
- Added org lookup by name (`Default`) and pass `organization` ID in credential POST

### Bootstrap credential approach
- Switched from `CONTROLLER_*` env var lookups to AAP custom credential type injection (`AAP Admin Credential`) for all admin operations
- Switched from `VAULT_TOKEN` env var to `Vault Bootstrap Token` custom credential type for Vault config playbooks
- Same pattern applied to both self-managed and HCP Vault config playbooks

### `community.hashi_vault` collection not available
- Default EE does not include `community.hashi_vault`
- Replaced all `vault_login`, `vault_read`, and `vault_kv2_write` module calls with `ansible.builtin.uri` calls across all config and demo playbooks
- Demo playbooks: JWT login via POST to `/v1/auth/<mount>/login`, secret read via GET with `X-Vault-Token` header
- Config playbooks: KV write via POST to `/v1/<mount>/data/<path>`, KV engine enable via POST to `/v1/sys/mounts/<mount>`

### AAP JWT injection — architecture correction
- `AAP_JWT_TOKEN` env var does not exist in the execution environment
- AAP 2.7 OIDC workload identity works via the built-in `HashiCorp Vault Secret Lookup` credential type, not env var injection
- Added `examples/vault-secret-lookup/` example demonstrating the AAP-native OIDC credential lookup workflow

### `vault_namespace` added to HashiCorp Vault JWT credential type
- Added `vault_namespace` as an optional field to the `HashiCorp Vault JWT` custom credential type
- Injected as extra var — eliminates need to set it per job template for HCP Vault Dedicated

### KV v2 secrets engine enablement
- HCP Vault Dedicated does not pre-enable the `secret` KV v2 engine
- Added `Enable KV v2 secrets engine` task to both Vault config playbooks before the secret write
- Graceful 400-already-enabled handling

### Job template bootstrap
- Added Task 4 to `configure_aap_vault_oidc.yml` to create all six job templates automatically
- Credential attachment uses GET-by-name lookup (not `jt_result`) to handle already-exists case
- Integer serialisation for credential `id` field: `body_format: raw` with hand-built JSON string `'{"id": {{ cred_id | int }} }'` — required because Ansible serialises quoted Jinja2 templates as strings regardless of `| int` filter

---

## Files changed this session

- `examples/aap-config/configure_aap_vault_oidc.yml` — major rework
- `examples/aap-config/vars.yml` — updated
- `examples/aap-config/README.md` — updated with credential setup instructions
- `examples/vault-config/configure_vault_oidc.yml` — env var → credential injection, uri replaces collection modules
- `examples/vault-config/vars.yml` — updated
- `examples/vault-config/README.md` — updated
- `examples/hcp-vault-config/configure_hcp_vault_oidc.yml` — same as above
- `examples/hcp-vault-config/vars.yml` — updated
- `examples/hcp-vault-config/README.md` — updated
- `examples/demo-playbook/read_vault_secret.yml` — replaced collection modules with uri
- `examples/hcp-demo-playbook/read_hcp_vault_secret.yml` — same + diagnostic env task added
- `examples/vault-secret-lookup/` — new directory (configure_vault_secret_lookup.yml, demo_vault_secret_lookup.yml, vars.yml, README.md)
- `demo.md` — updated throughout with correct credential types, troubleshooting entries
- `TODO.md` — items added and completed
- `.github/.agents/plan.md` — validation-run todo items ticked off
- `requirements.yml` — created (deleted externally; not committed)

---

## Commit reference

See commits: `4e286b5` through `3d74e72` (user commits during session) plus upcoming commit for docs/plan/todo updates.
