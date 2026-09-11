# AAP Configuration — HashiCorp Vault JWT Credential Setup

This example configures Ansible Automation Platform (AAP) 2.7 to support secretless authentication with HashiCorp Vault using JWT/OIDC workload identity. It enables the AAP OIDC issuer and creates the custom credential type and credential instance required by the demo playbook.

---

## Prerequisites

- **Ansible** 2.14 or later installed on the machine running this playbook.
- **AAP 2.7** instance with an account having the required permissions (see [Bootstrap Credential Permissions](#bootstrap-credential-permissions) below).
- No additional Ansible collections are required — all tasks use the `ansible.builtin.uri` module from Ansible core.
  > If you prefer a declarative approach using the `ansible.controller` (formerly `awx.awx`) collection, the collection can be installed with `ansible-galaxy collection install ansible.controller`. The playbook as written uses `uri` tasks for portability, so the collection is optional.
- Network reachability from the machine running this playbook to the AAP API (`https://<aap_host>`).

---

## Bootstrap Credential Permissions

The bootstrap playbook (`configure_aap_vault_oidc.yml`) executes three administrative operations via the AAP REST API. To adhere to the principle of least privilege, the user account or service account associated with the bootstrap credential needs only the following minimum-necessary permissions:

| Operation / Task | Endpoint & HTTP Method | Minimum AAP Role / Permission Required | Purpose |
|---|---|---|---|
| **Enable OIDC Issuer** | `PATCH /api/gateway/v1/settings/` (or `PATCH /api/v2/settings/system/`) | **System Administrator** (`is_superuser: true` / Platform Gateway Settings Administrator) | Modifies global platform gateway / system settings to enable per-job JWT issuance. |
| **Create / Look up Credential Type** | `POST /api/v2/credential_types/`<br>`GET /api/v2/credential_types/` | **System Administrator** (`is_superuser: true`) | Defines the custom `HashiCorp Vault JWT` credential type, its schema inputs, and injectors. |
| **Create Credential Instance** | `POST /api/v2/credentials/` | **System Administrator** (or **Credential Admin** / **Organization Admin** for the target Organization) | Instantiates the `Vault JWT - <role_name>` credential record referencing the custom credential type. |

### Minimum Role Summary

- **System Administrator (Superuser):** Required during the initial bootstrap run because modifying system/gateway settings and creating global custom credential types are privileged platform-level operations in AAP.
- **Dedicated Service Account Recommendation:** In production environments, create a dedicated bootstrap service account (e.g., `sa-aap-vault-bootstrap`) assigned Superuser permissions solely for running bootstrap configuration playbooks, rather than using personal or interactive administrator credentials.

---

## Variables

Edit `vars.yml` before running. All variables are required unless noted.

| Variable | Default | Description |
|---|---|---|
| `tower_host` | *(injected)* | AAP Controller hostname — injected by the built-in AAP platform credential; do not set in vars |
| `tower_username` | *(injected)* | AAP admin username — injected by the built-in AAP platform credential |
| `tower_password` | *(injected)* | AAP admin password — injected by the built-in AAP platform credential |
| `tower_verify_ssl` | *(injected)* | TLS validation flag — injected by the built-in AAP platform credential |
| `vault_addr` | `https://vault.example.com:8200` | Full Vault server URL (no trailing slash) |
| `vault_jwt_mount_path` | `jwt` | JWT auth method mount path in Vault |
| `vault_jwt_role_name` | `aap-automation` | JWT role name in Vault that AAP jobs authenticate against |
| `vault_secret_path` | `secret/data/aap-demo/config` | KV secret path used in the demo job template extra vars |

---

## How to Run as an AAP Job Template

This is the bootstrap step — run it once before running any other config playbooks. It uses AAP's built-in platform credential so no custom credential type is needed to get started.

1. In AAP, go to **Resources → Credentials → Add** and create a credential of type **Red Hat Ansible Automation Platform** with:
   - **Host:** `https://<your-aap-host>`
   - **Username:** your AAP admin username
   - **Password:** your AAP admin password
   - **Verify SSL:** set appropriately for your environment
2. Create a job template with:
   - **Playbook:** `examples/aap-config/configure_aap_vault_oidc.yml`
   - **Credentials:** attach the **Red Hat Ansible Automation Platform** credential created above
3. In the job template **Extra Variables** field, supply the non-sensitive values:
   ```yaml
   vault_addr: "https://<your-vault-host>:8200"
   vault_jwt_mount_path: "jwt"
   vault_jwt_role_name: "aap-automation"
   ```
4. Launch the job template. AAP admin credentials are injected securely — they never appear in Extra Variables or the job log.

After this job completes, the following credential types will exist in AAP and can be used by the other config playbooks:
- **Vault Bootstrap Token** — for `examples/vault-config/`
- **HCP Vault Bootstrap Token** — for `examples/hcp-vault-config/`
- **HashiCorp Vault JWT** — for the demo playbooks
- **AAP Admin Credential** — for future re-runs of this playbook without the built-in credential

---

## What It Configures

- **Enables the AAP OIDC issuer** — sends a `PATCH` to the AAP Gateway settings API to enable `ANSIBLE_CONTROLLER_GATEWAY_ENABLED`. This allows AAP to mint per-job JWT tokens that Vault can validate.
- **Creates a custom credential type** named `HashiCorp Vault JWT` with three input fields (`vault_addr`, `vault_jwt_mount`, `vault_role`) and an injector that exposes those values as Ansible extra vars inside job runs.
- **Creates a credential instance** of the new type, populated with the Vault connection details from `vars.yml`, ready to be attached to any AAP job template.

---

## Note — OIDC Issuer Enablement Endpoint

> ⚠️ **Verify during testing.** The OIDC issuer enablement task calls `PATCH /api/gateway/v1/settings/` with the setting key `ANSIBLE_CONTROLLER_GATEWAY_ENABLED: true`. This path is correct for AAP 2.7 deployments that include the platform gateway component.
>
> On **standalone controller-only** deployments (without the gateway), the correct endpoint may instead be `/api/v2/settings/system/` with a different setting key. Confirm the exact endpoint and key against your live AAP 2.7 instance and update the playbook task accordingly.
>
> Reference: [AAP 2.7 — OIDC Authentication for HashiCorp Vault](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/whats_new-oidc_authentication_for_hashicorp_vault)

---

## Next Steps

Once this playbook has run successfully, proceed to configure Vault:

- **Self-managed Vault** → run `../vault-config/configure_vault_oidc.yml` (see [`../vault-config/README.md`](../vault-config/README.md))
- **HCP Vault Dedicated** → run `../hcp-vault-config/configure_hcp_vault_oidc.yml` (see [`../hcp-vault-config/README.md`](../hcp-vault-config/README.md))

The `Vault JWT - aap-automation` credential created by this playbook is used by both the self-managed and HCP demo playbooks.
