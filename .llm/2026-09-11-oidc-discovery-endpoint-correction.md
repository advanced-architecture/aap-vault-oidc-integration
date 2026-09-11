# LLM Transcript — OIDC Discovery Endpoint Correction

**Date:** 2026-09-11
**Engine:** Claude (IBM Bob)
**Session topic:** Live validation of AAP bootstrap — OIDC endpoint correction

---

## Context

During live bootstrap validation against a real AAP 2.7 instance, the following failures and discoveries were made:

1. `GET /api/v2/ping/` → 404 (wrong endpoint for this deployment)
2. `GET /api/gateway/v1/ping/` → 200 (correct connectivity check endpoint confirmed)
3. `PATCH /api/gateway/v1/settings/` → 405 Method Not Allowed (endpoint is GET/HEAD only)
4. `GET /api/v2/settings/system/` → fails (not applicable for this topology)
5. `curl -L ${CONTROLLER_HOST}/o/.well-known/openid-configuration/` → valid OIDC discovery document (correct endpoint confirmed)

Key insight: On this AAP 2.7 platform-gateway deployment, the OIDC issuer is **already active by default** — no manual enablement step is required. The correct way to verify is via the standard OIDC discovery endpoint at `/o/.well-known/openid-configuration/`.

---

## Changes Made

### `examples/aap-config/configure_aap_vault_oidc.yml`
- Removed `Enable AAP OIDC issuer via Gateway settings API` task (the `PATCH /api/gateway/v1/settings/` call)
- Renumbered comments: Task 2 → Task 1, Task 3 → Task 2
- Updated header comment to document verification via `curl -L ${CONTROLLER_HOST}/o/.well-known/openid-configuration/`

### `examples/aap-config/README.md`
- Removed "Enable OIDC Issuer" row from Bootstrap Credential Permissions table (now 2 operations, not 3)
- Updated description from "three" to "two" administrative operations
- Replaced stale "OIDC Issuer Enablement Endpoint" warning with a clean "Note — OIDC Issuer" section documenting the correct verification command

### `demo.md`
- Added pre-flight OIDC verification block to Step 2 with `curl -L ${CONTROLLER_HOST}/o/.well-known/openid-configuration/`
- Removed stale `⚠️ OIDC issuer endpoint` warning from Step 2 Verify section
- Updated troubleshooting `AAP_JWT_TOKEN not set` row to reference correct check command
- Removed invalid "Step 2 fails at OIDC PATCH task" troubleshooting row
- Updated `aap_oidc_discovery_url` extra vars examples from `/api/gateway/v1/jwt/` → `/o`

### `README.md`
- Updated Integration Highlights table: AAP OIDC Issuer URL now points to `/o/.well-known/openid-configuration/` with correct description

### `examples/vault-config/vars.yml`
- Updated `aap_oidc_discovery_url` comment and placeholder value from `/api/gateway/v1/jwt/` to `/o`

### `examples/hcp-vault-config/vars.yml`
- Same update as vault-config/vars.yml

### `examples/vault-config/README.md`
- Updated variables table and `How to Run` command to use correct issuer URL pattern
- `How to Run` command now uses shell substitution to extract `issuer` from the discovery document

### `examples/hcp-vault-config/README.md`
- Same updates as vault-config/README.md

### `.github/.agents/plan.md`
- Updated critical alignment points table: AAP OIDC discovery URL corrected
- Updated Todo List item 2 to reference correct verification command

---

## Commit Reference

`fix(aap-config): correct OIDC discovery endpoint and remove stale issuer enablement task`
