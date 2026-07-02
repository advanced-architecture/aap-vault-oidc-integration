# AAP Configuration — HashiCorp Vault JWT Credential Setup

This example configures Ansible Automation Platform (AAP) 2.7 to support secretless authentication with HashiCorp Vault using JWT/OIDC workload identity. It enables the AAP OIDC issuer and creates the custom credential type and credential instance required by the demo playbook.

---

## Prerequisites

- **Ansible** 2.14 or later installed on the machine running this playbook.
- **AAP 2.7** instance with admin access. The admin account credentials are required to call the AAP REST API.
- No additional Ansible collections are required — all tasks use the `ansible.builtin.uri` module from Ansible core.
  > If you prefer a declarative approach using the `ansible.controller` (formerly `awx.awx`) collection, the collection can be installed with `ansible-galaxy collection install ansible.controller`. The playbook as written uses `uri` tasks for portability, so the collection is optional.
- Network reachability from the machine running this playbook to the AAP API (`https://<aap_host>`).

---

## Variables

Edit `vars.yml` before running. All variables are required unless noted.

| Variable | Default | Description |
|---|---|---|
| `aap_host` | `aap.example.com` | AAP Controller hostname or IP (no protocol, no trailing slash) |
| `aap_username` | `admin` | AAP admin username |
| `aap_password` | `CHANGEME` | AAP admin password — **change before use** |
| `aap_validate_certs` | `true` | Validate TLS certificates when calling the AAP API. Set to `false` for self-signed certs in lab environments. |
| `vault_addr` | `https://vault.example.com:8200` | Full Vault server URL (no trailing slash) |
| `vault_jwt_mount_path` | `jwt` | JWT auth method mount path in Vault |
| `vault_jwt_role_name` | `aap-automation` | JWT role name in Vault that AAP jobs authenticate against |
| `vault_secret_path` | `secret/data/aap-demo/config` | KV secret path used in the demo job template extra vars |

---

## How to Run

```bash
# 1. Edit variables for your environment
vi vars.yml

# 2. Run the playbook
ansible-playbook configure_aap_vault_oidc.yml
```

To disable TLS verification in a lab environment:

```bash
ansible-playbook configure_aap_vault_oidc.yml -e aap_validate_certs=false
```

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

Once this playbook has run successfully:

1. Complete the Vault-side configuration using `../vault-config/configure_vault_oidc.yml`.
2. Attach the `Vault JWT - <role>` credential to your AAP job template.
3. Run the end-to-end demo from `../demo-playbook/read_vault_secret.yml` to verify the full JWT authentication flow.
