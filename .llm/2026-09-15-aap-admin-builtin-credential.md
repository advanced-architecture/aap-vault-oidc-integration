# LLM Session Transcript — 2026-09-15

**Topic:** `aap-admin-builtin-credential`
**Engine:** IBM Bob (claude-sonnet-4-5)
**Session date:** 2026-09-15

---

## Objective

Replace the custom "AAP Admin Credential" credential type with the built-in
**Red Hat Ansible Automation Platform** credential type across all AAP-admin-facing
playbooks, and switch all controller credential references from `{{ controller_* }}`
extra vars to `lookup('env', 'CONTROLLER_*')`.

---

## Files changed

### `examples/aap-config/configure_aap_vault_oidc.yml`
- Updated header comment: documents `CONTROLLER_*` env vars and built-in credential type
- `aap_api_base` var now uses `lookup('env', 'CONTROLLER_HOST')`
- Guard task: loop items changed to `CONTROLLER_HOST/USERNAME/PASSWORD`; `when` changed
  from `vars[item] is not defined` to `lookup('env', item) | length == 0`
- All 11 `uri` task occurrences of `url_username`, `url_password`, `validate_certs`
  changed to `lookup('env', 'CONTROLLER_*')` / `| default('true', true) | bool`

### `examples/vault-secret-lookup/configure_vault_secret_lookup.yml`
- Same pattern applied to all 3 `uri` tasks; header comment and guard task updated

### `examples/aap-config/README.md`
- Prerequisites bullet updated: references built-in type instead of custom type
- Bootstrap Credential Setup section replaced: 3-step custom type creation (with YAML
  snippets) condensed to 2 steps — create instance of built-in type, attach to template
- Variables table updated: `controller_*` rows replaced with `CONTROLLER_*` env var rows
- How to Run updated: references built-in credential instead of custom type

### `examples/aap-config/vars.yml`
- Comment updated: references `CONTROLLER_*` env vars and built-in credential type

### `examples/vault-secret-lookup/vars.yml`
- Comment updated: same

### `examples/vault-secret-lookup/README.md`
- Credentials table row updated: `AAP Admin Credential` → `Red Hat Ansible Automation Platform`

### `demo.md`
- Line 60: credential attachment instruction updated to built-in type + env var names
- Line 252: same update for Step 5a prerequisites

---

## Key decisions

- `validate_certs` filter chain: `(lookup('env', 'CONTROLLER_VERIFY_SSL') | default('true', true)) | bool`
  — `default(value, boolean=True)` treats empty string as missing and falls back to `'true'`;
  `| bool` coerces the resulting string to a proper boolean for the `uri` module.
- `CONTROLLER_*` is the canonical modern name; `TOWER_*` and `AAP_*` are aliases also
  injected by the built-in type but not referenced here.
- `.llm/` transcripts and `plan.md` historical text retain "AAP Admin Credential" mentions
  deliberately — they are provenance records, not active code.

---

## Upcoming commit

```
feat(aap-config): switch admin credential from custom type to built-in AAP type

Replace the custom "AAP Admin Credential" extra_vars injector with the
built-in "Red Hat Ansible Automation Platform" credential type, and read
all controller credentials via lookup('env', 'CONTROLLER_*').

CONTROLLER_PASSWORD is now in AAP's runner scrub list (env injector with
secret:true), preventing leakage in verbose play-vars output.

Transcript: .llm/2026-09-15-aap-admin-builtin-credential.md
```
