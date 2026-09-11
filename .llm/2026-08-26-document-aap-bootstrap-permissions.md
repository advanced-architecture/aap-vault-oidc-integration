**Tool:** IBM Bob
**Date:** 2026-08-26

## Prompts

1. "Define the minimum-necessary permissions needed for the AAP credential used in the bootstrap process. Document them in the aap-config readme and link to that information from the main readme"
2. "prepare to close this task"

## Key Changes Made

1. **`examples/aap-config/README.md`**:
   - Added a detailed **Bootstrap Credential Permissions** section specifying the operations performed (`PATCH /api/gateway/v1/settings/`, `POST /api/v2/credential_types/`, `POST /api/v2/credentials/`), the minimum required role (`System Administrator` / `is_superuser`), and recommended security best practices (dedicated service account `sa-aap-vault-bootstrap`).
   - Updated the **Prerequisites** section to reference the new permissions section.

2. **`README.md`**:
   - Added links in the **Prerequisites** and **Usage / Quick Start** (step 2) sections directly navigating to the AAP bootstrap permissions documentation in `examples/aap-config/README.md#bootstrap-credential-permissions`.

## Commit Message Reference

```
docs(aap-config): define minimum-necessary permissions for bootstrap credential

Document the required AAP REST API endpoints and least-privilege role
(System Administrator / is_superuser) needed to enable the OIDC issuer,
create custom credential types, and instantiate credentials. Link to this
guidance from the root README.

Transcript: .llm/2026-08-26-document-aap-bootstrap-permissions.md
```
