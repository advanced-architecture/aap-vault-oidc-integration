# LLM Transcript — aap-admin-builtin-credential planning session

**Date:** 2026-09-15
**Engine:** IBM Bob (Claude Sonnet via Bob agent mode)
**Workspace:** `/Users/joani.delaporte/Bob/aap-vault-oidc-integration`

---

## Session goals

1. Investigate whether the built-in **Red Hat Ansible Automation Platform** credential type could replace the custom "AAP Admin Credential" credential type.
2. Determine whether reading from the built-in type's `env` injector (rather than its `extra_vars` injector) provides automatic log scrubbing for `CONTROLLER_PASSWORD`.
3. Write the plan sub-task and close the TODO item.

---

## Key findings

### Built-in credential type injector schema (provided by user from live AAP instance)

The built-in **Red Hat Ansible Automation Platform** credential type injects **both** `extra_vars` and `env` vars:

```yaml
extra_vars:
  aap_hostname: '{{host}}'
  aap_username: '{{username}}'
  aap_password: '{{password}}'
  aap_token: '{{oauth_token}}'
  aap_request_timeout: '{{request_timeout}}'
  aap_validate_certs: '{{verify_ssl}}'
env:
  TOWER_HOST: '{{host}}'
  TOWER_USERNAME: '{{username}}'
  TOWER_PASSWORD: '{{password}}'
  TOWER_VERIFY_SSL: '{{verify_ssl}}'
  TOWER_OAUTH_TOKEN: '{{oauth_token}}'
  CONTROLLER_HOST: '{{host}}'
  CONTROLLER_USERNAME: '{{username}}'
  CONTROLLER_PASSWORD: '{{password}}'
  CONTROLLER_VERIFY_SSL: '{{verify_ssl}}'
  CONTROLLER_OAUTH_TOKEN: '{{oauth_token}}'
  CONTROLLER_REQUEST_TIMEOUT: '{{request_timeout}}'
  AAP_HOSTNAME: '{{host}}'
  AAP_USERNAME: '{{username}}'
  AAP_PASSWORD: '{{password}}'
  AAP_VALIDATE_CERTS: '{{verify_ssl}}'
  AAP_TOKEN: '{{oauth_token}}'
  AAP_REQUEST_TIMEOUT: '{{request_timeout}}'
```

### Why env over extra_vars

AAP adds the values of credential fields marked `secret: true` (password, token) to the ansible-runner's `no_log` word list when injected via `env`. This means the password is automatically replaced with `********` in all task output and verbose play-vars dumps — not just at the `url_password` HTTP level (which `ansible.builtin.uri` already handles). The `extra_vars` path does not trigger this runner-level scrubbing.

### Affected files

- `examples/aap-config/configure_aap_vault_oidc.yml` — `aap_api_base` var, guard task (11 `url_username`/`url_password`/`validate_certs` occurrences)
- `examples/vault-secret-lookup/configure_vault_secret_lookup.yml` — same pattern (3 uri tasks, 9 occurrences)
- `examples/aap-config/README.md` — Bootstrap Credential Setup section, Variables table, How to Run

### Implementation note: validate_certs

`CONTROLLER_VERIFY_SSL` is a boolean field in the credential type but env vars are always strings. The correct Ansible pattern is:

```yaml
validate_certs: "{{ lookup('env', 'CONTROLLER_VERIFY_SSL') | default(true, true) }}"
```

`default(true, true)` treats an empty string (unset env var) as `true`, which is the safe default.

---

## Changes made this session

- **`TODO.md`** — three new items added and one closed:
  - Added `aap-credential-hide-token-uri` (env injector fix for `vault_token` in `X-Vault-Token` headers)
  - Closed `secure-vault-token-logs` (superseded by `aap-credential-hide-token-uri`)
  - Added `aap-admin-builtin-credential` (built-in credential type + env lookup)
  - Closed `aap-admin-builtin-credential` (plan written to plan.md)

- **`.github/.agents/plan.md`** — new sub-task `aap-admin-builtin-credential` added with full intent, expected outcomes, changeset table, and implementation notes.

---

## Upcoming commit

```
docs(plan): add aap-admin-builtin-credential sub-task and close TODO

Replace custom AAP Admin Credential type with built-in Red Hat Ansible
Automation Platform type. Plan written to .github/.agents/plan.md with
full changeset: configure_aap_vault_oidc.yml and
configure_vault_secret_lookup.yml switch to lookup('env', 'CONTROLLER_*')
for automatic password log scrubbing; README.md updated accordingly.

Transcript: .llm/2026-09-15-aap-admin-builtin-credential.md
```
