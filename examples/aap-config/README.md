# AAP Configuration — HashiCorp Vault JWT Credential Setup

This example configures Ansible Automation Platform (AAP) 2.7 to support secretless authentication with HashiCorp Vault using JWT/OIDC workload identity. It creates the custom credential type and credential instance required by the demo playbook.

---

## Prerequisites

- **AAP 2.7** instance with an account having the required permissions (see [Bootstrap Credential Permissions](#bootstrap-credential-permissions) below).
- No additional Ansible collections are required — all tasks use the `ansible.builtin.uri` module from Ansible core.
- An **AAP Admin Credential** (custom credential type) created and attached to this job template — see [Bootstrap Credential Setup](#bootstrap-credential-setup) below.
- The feature flag for OIDC Authentication must be turned on (see [Enable OIDC Feature Flag](#enable-oidc-feature-flag) below).

---

## Enable OIDC Feature Flag

The AAP Cluster must have the OIDC Feature Flag turned on. This feature flag is exposed via the yaml defining the Resource in the OpenShift cluster where AAP is running. There are a variety of ways to navigate to this resource definition. I found it via this path: OCP platform console -> Ecosystem -> Installed Operators -> Ansible Automation Platform -> (scroll to the right on the top bar ... Details, YAML, Subscription, Events, All instances, Automation Controller Backup, Automation Controller Restore, Automation Controller, ... until you get to Ansible Automation Platform) -> click on the name of your AAP; mine is `sandbox-aap` (if you cannot see it, change the "Show operands in" setting) -> YAML tab. Once there, use the guidance from the link below. I only needed to add the following into the `spec:` section.

```yaml
 feature_flags:
    FEATURE_OIDC_WORKLOAD_IDENTITY_ENABLED: True
```

Extra helpful reading: [Just in Time Access to HashiCorp Vault](https://developers.redhat.com/articles/2026/08/11/just-in-time-access-to-hashicorp-vault-with-ansible-oidc-provider?source=sso#configure_ansible_access)

You can confirm the feature is turned on with this check:

% curl -sk -u "${CONTROLLER_USERNAME}:${CONTROLLER_PASSWORD}" \
  "${CONTROLLER_HOST}/o/.well-known/openid-configuration/" | python3 -m json.tool

Which will return something like the following result, if it is successful.

``` json
{
    "issuer": "https://controller_url/o",
    "authorization_endpoint": "https://controller_url/o/authorize/",
    "token_endpoint": "https://controller_url/o/token/",
    "userinfo_endpoint": "https://controller_url/o/userinfo/",
    "jwks_uri": "https://controller_url/o/.well-known/jwks.json",
    "scopes_supported": [
        "read",
        "write",
        "aap_controller_automation_job",
        "openid",
        "roles"
    ],
    "response_types_supported": [
        "code",
        "token",
        "id_token",
        "id_token token",
        "code token",
        "code id_token",
        "code id_token token"
    ],
    "subject_types_supported": [
        "public"
    ],
    "id_token_signing_alg_values_supported": [
        "RS256",
        "HS256"
    ],
    "token_endpoint_auth_methods_supported": [
        "client_secret_post",
        "client_secret_basic"
    ],
    "claims_supported": [
        "aap_controller_launched_by_id",
        "aap_controller_organization_name",
        "exp",
        "aap_controller_project_id",
        "given_name",
        "aap_controller_organization_id",
        "aap_controller_job_template_name",
        "sub",
        "aap_controller_instance_group_name",
        "name",
        "aud",
        "preferred_username",
        "aap_controller_inventory_name",
        "aap_controller_launch_type",
        "aap_controller_instance_group_id",
        "aap_controller_unified_job_template_id",
        "aap_controller_execution_environment_name",
        "aap_controller_job_template_id",
        "aap_controller_job_id",
        "jti",
        "email",
        "aap_system_role",
        "aap_controller_job_type",
        "aap_controller_inventory_id",
        "aap_controller_unified_job_template_name",
        "aap_organizations",
        "iss",
        "aap_controller_execution_environment_id",
        "aap_controller_job_name",
        "aap_controller_playbook_name",
        "aap_controller_project_name",
        "aap_teams",
        "iat",
        "family_name",
        "aap_controller_launched_by_name"
    ],
    "end_session_endpoint": "https://controller_url/o/logout/",
    "revocation_endpoint": "https://controller_url/o/revoke_token/",
    "code_challenge_methods_supported": [
        "S256",
        "plain"
    ]
```

## Bootstrap Credential Permissions

The bootstrap playbook (`configure_aap_vault_oidc.yml`) executes two administrative operations via the AAP REST API. To adhere to the principle of least privilege, the user account or service account associated with the bootstrap credential needs only the following minimum-necessary permissions:

| Operation / Task | Endpoint & HTTP Method | Minimum AAP Role / Permission Required | Purpose |
|---|---|---|---|
| **Create / Look up Credential Type** | `POST /api/controller/v2/credential_types/`<br>`GET /api/controller/v2/credential_types/` | **System Administrator** (`is_superuser: true`) | Defines the custom `HashiCorp Vault JWT` credential type, its schema inputs, and injectors. |
| **Create Credential Instance** | `POST /api/controller/v2/credentials/` | **System Administrator** (or **Credential Admin** / **Organization Admin** for the target Organization) | Instantiates the `Vault JWT - <role_name>` credential record referencing the custom credential type. |

### Minimum Role Summary

- **System Administrator (Superuser):** Required during the initial bootstrap run because creating global custom credential types is a privileged platform-level operation in AAP.
- **Dedicated Service Account Recommendation:** In production environments, create a dedicated bootstrap service account (e.g., `sa-aap-vault-bootstrap`) assigned Superuser permissions solely for running bootstrap configuration playbooks, rather than using personal or interactive administrator credentials.

---

## Bootstrap Credential Setup

The playbook reads AAP admin credentials from extra vars injected by a custom AAP credential type. Create this once before running the job template.

### 1 — Create the custom credential type

In AAP → **Resources → Credential Types → Add**:

| Field | Value |
|---|---|
| **Name** | `AAP Admin Credential` |
| **Kind** | `Cloud` |

**Input Configuration:**
```yaml
fields:
  - id: controller_host
    type: string
    label: Controller Host URL
  - id: controller_username
    type: string
    label: Controller Username
  - id: controller_password
    type: string
    label: Controller Password
    secret: true
  - id: controller_verify_ssl
    type: boolean
    label: Verify SSL
required:
  - controller_host
  - controller_username
  - controller_password
```

**Injector Configuration:**
```yaml
extra_vars:
  controller_host: '{{ controller_host }}'
  controller_username: '{{ controller_username }}'
  controller_password: '{{ controller_password }}'
  controller_verify_ssl: '{{ controller_verify_ssl }}'
```

### 2 — Create a credential instance

In AAP → **Resources → Credentials → Add**:

| Field | Value |
|---|---|
| **Name** | `AAP Admin - <your-instance-name>` |
| **Credential Type** | `AAP Admin Credential` |
| **Controller Host URL** | `https://<your-aap-host>` |
| **Controller Username** | your AAP admin username |
| **Controller Password** | your AAP admin password |
| **Verify SSL** | checked (uncheck only to skip TLS validation) |

### 3 — Attach to the job template

When creating the job template for `configure_aap_vault_oidc.yml`, add this credential under **Credentials**.

---

## Variables

Edit `vars.yml` before running. All variables are required unless noted.

| Variable | Default | Description |
|---|---|---|
| `controller_host` | *(credential injector)* | AAP platform-gateway base URL (e.g. `https://<aap-host>`). API calls route to `/api/controller/v2/`. Injected by AAP Admin Credential. |
| `controller_username` | *(credential injector)* | AAP admin username — injected by AAP Admin Credential |
| `controller_password` | *(credential injector)* | AAP admin password — injected by AAP Admin Credential |
| `controller_verify_ssl` | *(credential injector)* | TLS validation flag — injected by AAP Admin Credential |
| `vault_addr` | `https://vault.example.com:8200` | Full Vault server URL (no trailing slash) |
| `vault_jwt_mount_path` | `jwt` | JWT auth method mount path in Vault |
| `vault_jwt_role_name` | `aap-automation` | JWT role name in Vault that AAP jobs authenticate against |
| `vault_secret_path` | `secret/data/aap-demo/config` | KV secret path used in the demo job template extra vars |

---

## How to Run

Create the job template in AAP with the `AAP Admin Credential` attached, supply the Vault extra vars, and launch:

```yaml
vault_addr: "https://<your-vault-host>:8200"
vault_jwt_mount_path: "jwt"
vault_jwt_role_name: "aap-automation"
```

After this playbook completes, the **HashiCorp Vault JWT** credential type and a matching credential instance will exist in AAP, ready to be attached to the demo job templates.

---

## What It Configures

- **Creates a custom credential type** named `HashiCorp Vault JWT` with three input fields (`vault_addr`, `vault_jwt_mount`, `vault_role`) and an injector that exposes those values as Ansible extra vars inside job runs.
- **Creates a credential instance** of the `HashiCorp Vault JWT` type, populated with the Vault connection details from `vars.yml`, ready to be attached to any AAP job template.
- **Creates a custom credential type** named `Vault Bootstrap Token` with a single secret `vault_token` field and an `env` injector (`VAULT_TOKEN`) — used by the Vault config job templates. The credential *instance* (which contains the actual token) must be created manually after this playbook runs.

---

## Credential Injectors & Secret Masking in AAP

In AAP, custom credential types define how secrets and inputs are exposed to job execution environments:

- **`extra_vars` injector:** Injected values become regular Ansible variables. These are **not** added to AAP's runner-level `no_log` string-scrubbing word list. Consequently, expressions like `X-Vault-Token: "{{ vault_token }}"` in `ansible.builtin.uri` tasks print the plain-text Vault token in job output / standard out if verbosity is enabled.
- **`env` injector:** Any credential field marked `secret: true` and mapped via `env: { VAULT_TOKEN: "{{ vault_token }}" }` causes the AAP runner to add the secret value to its runtime scrubber. Any occurrence of that secret value across all stdout/stderr streams (including within serialized headers of HTTP requests) is automatically masked with `********`.

**Documentation Reference:**
- [Red Hat Ansible Automation Platform — Custom Credential Types & Injectors](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/using_automation_execution/custom-credential-types#custom_credential_types)
- [Ansible Runner Secret Masking / Passwords Documentation](https://ansible-runner.readthedocs.io/en/latest/intro/#passwords-and-sensitive-data) (describes the runner-level environment variable scrubber and wordlist masking mechanism)

---

## Note — OIDC Issuer

The AAP OIDC issuer is enabled by default on AAP 2.7 platform-gateway deployments — no manual enablement step is required. Verify it is active before running the Vault configuration playbook:

```bash
curl -L ${CONTROLLER_HOST}/o/.well-known/openid-configuration/
```

A valid OIDC discovery document (containing `issuer`, `jwks_uri`, `token_endpoint`, etc.) confirms the issuer is live. If this returns a 404, check that the OIDC workload identity feature is available on your AAP instance.

---

## Next Steps

Once this playbook has run successfully, proceed to configure Vault:

- **Self-managed Vault** → run `../vault-config/configure_vault_oidc.yml` (see [`../vault-config/README.md`](../vault-config/README.md))
- **HCP Vault Dedicated** → run `../hcp-vault-config/configure_hcp_vault_oidc.yml` (see [`../hcp-vault-config/README.md`](../hcp-vault-config/README.md))

The `Vault JWT - aap-automation` credential created by this playbook is used by both the self-managed and HCP demo playbooks.
