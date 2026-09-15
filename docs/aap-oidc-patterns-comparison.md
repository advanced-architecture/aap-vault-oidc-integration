# AAP OIDC Vault Integration — Pattern Comparison

> **Sources:** Red Hat AAP 2.7 documentation, Red Hat Developer articles, community research  
> **Date:** 2026-09-15  
> **Scope:** Architectural comparison of the two OIDC-based Vault integration patterns available in AAP 2.7

---

## Overview

AAP 2.7 supports two distinct patterns for authenticating to Vault using OIDC JWTs.
They share the same underlying token issuance mechanism but differ fundamentally in
**who handles the JWT** and **how secrets reach the playbook**.

| | Pattern 1 — AAP-native secret lookup | Pattern 2 — Playbook-level JWT exchange |
|---|---|---|
| **Who uses the JWT** | AAP control plane (before the job runs) | The playbook itself (during the job run) |
| **JWT passed to EE?** | No | Potentially — see caveat below |
| **Secret accessible in playbook?** | Yes — as an injected extra variable | Yes — after the playbook calls Vault directly |
| **Vault call made by** | AAP's credential plugin layer | `ansible.builtin.uri` or `community.hashi_vault` in the playbook |
| **Officially documented?** | ✅ Yes — Red Hat docs + Developer articles | ❌ No — community-sourced; variable name unconfirmed |
| **Tech Preview (AAP 2.7)?** | Yes | Yes (same feature flag required) |
| **Recommended for production?** | Preferred once confirmed stable | Use with caution — see caveats |

---

## Pattern 1 — AAP-Native Secret Lookup (Officially Documented)

### How it works

AAP's control plane issues the JWT and uses it internally. The JWT never enters
the Execution Environment (EE). Instead:

1. The `HashiCorp Vault Secret Lookup (OIDC)` credential is attached to the job template.
2. At job launch, AAP's credential plugin layer presents the JWT to Vault, receives a
   short-lived Vault token, and fetches the secret.
3. The **fetched secret value** — not the JWT, not the Vault token — is injected into
   the job as an extra variable via a custom credential type.
4. The playbook reads `{{ hashicorp_vault_value }}` (or whatever field name the
   custom credential type defines).

Red Hat documentation explicitly states:
> *"the JWT is only used by the control plane to access HashiCorp Vault and retrieve
> secrets before automation begins executing."*

#todo - add direct reference link

### Credential type injector configuration

#todo - Update this section. Custom credential not accurate for this model. There is a built in credential type, which should be used instead.

```yaml
# Custom credential type input
fields:
  - id: hashicorp_vault_value
    type: string
    label: HashiCorp Vault Value
required:
  - hashicorp_vault_value

# Custom credential type injector
extra_vars:
  hashicorp_vault_value: '{{ hashicorp_vault_value }}'
```

The secret field is wired to the Vault lookup credential using the key icon in the
AAP UI — no hardcoded secret is stored anywhere.


### Playbook usage

#todo - update this section to use built-in Vault OIDC credential type, instead of custom credential type.

```yaml
- name: Fail if secret was not injected
  ansible.builtin.fail:
    msg: "hashicorp_vault_value is not set — check credential attachment"
  when: (hashicorp_vault_value is not defined) or (hashicorp_vault_value | length == 0)

- name: Use the injected secret
  ansible.builtin.debug:
    msg: "Secret length: {{ hashicorp_vault_value | length }} chars"
  # In real use, pass hashicorp_vault_value to the task that needs it
  # and set no_log: true on that task
```

### Example in this repository

[`examples/vault-secret-lookup/demo_vault_secret_lookup.yml`](../examples/vault-secret-lookup/demo_vault_secret_lookup.yml)

### Limitations

- The `HashiCorp Vault Secret Lookup (OIDC)` credential type is a **lookup credential**,
  not a direct job template credential. It cannot be attached to a job template by itself.
  A wrapping custom credential type is always required to bridge the two.
- Each secret field requires its own credential and UI wiring. Fetching multiple
  secrets requires multiple custom credential bindings.

#todo - add direct link to current documentation to cite this limitation

---

## Pattern 2 — Playbook-Level JWT Exchange (Community-Documented)

### How this might work (unconfirmed)

The playbook retrieves the JWT from the EE environment at runtime and uses it to
authenticate to Vault directly, exchanging it for a short-lived Vault client token:

1. AAP injects the JWT into the EE as an environment variable.
2. The playbook captures it: `lookup('env', 'AAP_JWT_TOKEN')`.
3. The playbook POSTs the JWT to Vault's `auth/<mount>/login` endpoint.
4. Vault validates the JWT against AAP's JWKS endpoint, applies the role's
   `bound_claims`, and returns a short-lived Vault token.
5. The playbook uses that Vault token to read secrets directly from the KV API.

### Playbook usage

```yaml
- name: Capture AAP-injected JWT token from environment
  ansible.builtin.set_fact:
    jwt_token: "{{ lookup('env', 'AAP_JWT_TOKEN') }}"

- name: Fail if AAP_JWT_TOKEN is not set
  ansible.builtin.fail:
    msg: >-
      AAP_JWT_TOKEN is not set. Ensure the OIDC feature flag is enabled
      and this playbook runs as an AAP job template.
  when: jwt_token | length == 0

- name: Authenticate to Vault using the AAP JWT
  ansible.builtin.uri:
    url: "{{ vault_addr }}/v1/auth/{{ vault_jwt_mount_path }}/login"
    method: POST
    body_format: json
    body:
      role: "{{ vault_jwt_role_name }}"
      jwt: "{{ jwt_token }}"
    status_code: [200]
  register: vault_login_result
  no_log: true

- name: Read secret from Vault
  ansible.builtin.uri:
    url: "{{ vault_addr }}/v1/{{ vault_secret_path }}"
    method: GET
    headers:
      X-Vault-Token: "{{ vault_login_result.json.auth.client_token }}"
    status_code: [200]
  register: vault_secret
  no_log: true
```

### Examples in this repository

- [`examples/demo-playbook/read_vault_secret.yml`](../examples/demo-playbook/read_vault_secret.yml) — self-managed Vault
- [`examples/hcp-demo-playbook/read_hcp_vault_secret.yml`](../examples/hcp-demo-playbook/read_hcp_vault_secret.yml) — HCP Vault Dedicated

### ⚠️ Critical caveat — `AAP_JWT_TOKEN` is not officially documented

**The environment variable name `AAP_JWT_TOKEN` does not appear in any Red Hat
official documentation** (docs.redhat.com or developers.redhat.com) found as of
the research date above.

Key facts:
- The Red Hat docs describe the JWT as being used by the "control plane" only,
  with no mention of exposing it to the EE.
- The name `AAP_JWT_TOKEN` originates from community sources and was used in this
  repository's playbooks as a best-guess based on AAP naming conventions.
- The HCP demo playbook ([`read_hcp_vault_secret.yml`](../examples/hcp-demo-playbook/read_hcp_vault_secret.yml))
  includes a diagnostic task that runs `env` and dumps all environment variables
  specifically because the actual variable name needs to be confirmed at runtime.
- The feature is Technology Preview in AAP 2.7; the mechanism may change before GA.

**Before relying on Pattern 2 in any environment, run the diagnostic `env` dump
and confirm the exact variable name on your AAP instance.**

---

## Enabling the Feature

Both patterns require the OIDC workload identity feature flag. It is not enabled
by default in AAP 2.7 fresh installs or upgrades.

**RHEL containerized installer** — set in the installation inventory:

```ini
feature_flags:
  FEATURE_OIDC_WORKLOAD_IDENTITY_ENABLED: True
```

**OpenShift operator** — set in the `AnsibleAutomationPlatform` custom resource:

```yaml
apiVersion: aap.ansible.com/v1alpha1
kind: AnsibleAutomationPlatform
metadata:
  name: aap
spec:
  feature_flags:
    FEATURE_OIDC_WORKLOAD_IDENTITY_ENABLED: True
```

Verify the issuer is live after enabling:

```bash
curl -L https://<aap-host>/o/.well-known/openid-configuration/
```

A valid OIDC discovery document confirms the provider is running.

---

## JWT Claims Reference

Every AAP-issued JWT contains these claims, which Vault roles can use in
`bound_claims` and `claim_mappings`:

| Claim | Description |
|---|---|
| `sub` | Composite subject identifier including org and job template |
| `aap_controller_job_id` | Unique job ID |
| `aap_controller_job_name` | Job template name |
| `aap_controller_job_type` | `run`, `cleanup`, etc. |
| `aap_controller_launch_type` | `manual`, `scheduled`, `webhook`, `workflow` |
| `aap_controller_playbook_name` | Playbook filename |
| `aap_controller_launched_by_name` | Username that launched the job |
| `aap_controller_launched_by_id` | User ID |
| `aap_controller_organization_name` | AAP organization name |
| `aap_controller_organization_id` | Organization ID |
| `aap_controller_inventory_name` | Inventory name |
| `aap_controller_inventory_id` | Inventory ID |
| `aap_controller_project_name` | Project name |
| `aap_controller_project_id` | Project ID |
| `aap_controller_job_template_name` | Job template name |
| `aap_controller_job_template_id` | Job template ID |
| `aap_controller_execution_environment_name` | EE name |
| `aap_controller_execution_environment_id` | EE ID |

See [`docs/vault-security-boundaries.md`](vault-security-boundaries.md) for
patterns on using these claims to enforce least-privilege Vault roles.

---

## Decision Guide

```
Do you need a secret in your playbook?
│
├─ Yes ──► Can you use the built-in HashiCorp Vault Secret Lookup (OIDC) credential?
│           │
│           ├─ Yes ──► Use Pattern 1. Attach the lookup credential via the key icon,
│           │          create a wrapping custom credential type, and read
│           │          {{ hashicorp_vault_value }} in the playbook.
│           │          This is the officially documented, recommended path.
│           │
│           └─ No (e.g. you need to read many dynamic paths at runtime,
│                or need the Vault token for multiple operations) ──►
│                Use Pattern 2, but:
│                1. Run the env diagnostic to confirm AAP_JWT_TOKEN on your instance.
│                2. Add no_log: true on every task that touches the JWT or Vault token.
│                3. Treat this as experimental until officially documented by Red Hat.
│
└─ No ───► No credential attachment needed.
```

---

## References

- [AAP 2.7 — OIDC authentication for HashiCorp Vault](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/whats_new-oidc_authentication_for_hashicorp_vault)
- [AAP 2.7 — OIDC credential types for HashiCorp Vault](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/whats_new-oidc_credential_types_for_hashicorp_vault)
- [AAP 2.7 — Claims for workload identity](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/whats_new-claims_for_workload_identity)
- [Just-in-time access to HashiCorp Vault using the Red Hat Ansible Automation Platform OIDC provider](https://developers.redhat.com/articles/2026/08/11/just-in-time-access-to-hashicorp-vault-with-ansible-oidc-provider)
- [HashiCorp Vault — JWT/OIDC Auth Method](https://developer.hashicorp.com/vault/docs/auth/jwt)
