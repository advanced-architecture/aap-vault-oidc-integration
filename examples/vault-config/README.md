# Vault Configuration Playbook

This playbook configures the HashiCorp Vault side of the AAP OIDC/JWT integration. It must be run once (or idempotently re-run) before the AAP demo playbook can authenticate to Vault using JWT workload identity.

---

## Prerequisites

- **Ansible** >= 2.14
- **`community.hashi_vault` collection** >= 6.x:
  ```bash
  ansible-galaxy collection install community.hashi_vault
  ```
- **Network access** from the machine running this playbook to the Vault server (`vault_addr`)
- **A Vault bootstrap token** with permissions to:
  - Enable auth methods (`sys/auth`)
  - Write policies (`sys/policies/acl`)
  - Write secrets to the KV v2 mount

In the case where a Vault server is needed, a Vault Dedicated cluster can be configured on portal.cloud.hashicorp.com/services/vault/clusters in a few clicks.  

This repo was tested with a Vault Dedicated cluster with this process:
1. Go to portal.cloud.hashicorp.com
1. Create project: "vault-ansible-oidc"
1. Click `Vault Dedicated` tile: Get started with Vault Dedicated
2. Click `Start from scratch`
3.  Accept the defaults (AWS, Development tier, Extra Small, default network HVN)
4. Configure cluster ID to be unique within your HCP organization. I used "vault-cluster-ansible-oidc".
5. Choose `Start from scratch`
6. Enable reporting (currently beta)
---

## Variables

All variables are defined in [`vars.yml`](vars.yml). Override any of them at run time with `-e key=value`.

| Variable | Description | Required | Default |
|---|---|---|---|
| `vault_addr` | Vault server address (no trailing slash) | ✅ | `https://vault.example.com:8200` |
| `vault_token` | Bootstrap admin token for initial configuration | ✅ | `s.CHANGEME` |
| `aap_oidc_discovery_url` | AAP 2.7 OIDC discovery URL (`/api/gateway/v1/jwt/`) | ✅ | `https://aap.example.com/api/gateway/v1/jwt/` |
| `vault_jwt_mount_path` | Mount path for the Vault JWT auth method | ✅ | `jwt` |
| `vault_jwt_role_name` | Name of the JWT role Vault creates for AAP jobs | ✅ | `aap-automation` |
| `vault_policy_name` | Name of the Vault ACL policy attached to the JWT role | ✅ | `aap-automation-policy` |
| `vault_secret_path` | Full KV v2 path the policy grants read access to | ✅ | `secret/data/aap-demo/config` |
| `vault_kv_mount` | KV v2 engine mount name (no `/data/` segment) | ✅ | `secret` |
| `vault_kv_secret_path` | Secret path within the KV mount (no mount prefix) | ✅ | `aap-demo/config` |
| `jwt_token_ttl` | TTL for Vault tokens issued to AAP jobs | ✅ | `300s` |
| `jwt_bound_audience` | Required `aud` claim value in the AAP JWT | ✅ | `https://vault.example.com:8200` |

---

## How to Run as an AAP Job Template

> **Prerequisite:** Run `examples/aap-config/configure_aap_vault_oidc.yml` first. It creates the `Vault Bootstrap Token` credential type in AAP that this playbook requires.

1. In AAP, go to **Resources → Credentials → Add** and create a credential of type **Vault Bootstrap Token**. Enter your Vault admin token in the `Vault Bootstrap Token` field.
2. Create a job template (or update the existing one) with:
   - **Playbook:** `examples/vault-config/configure_vault_oidc.yml`
   - **Credentials:** attach the `Vault Bootstrap Token` credential created above
3. In the job template **Extra Variables** field, supply the environment-specific non-sensitive values:
   ```yaml
   vault_addr: "https://<your-vault-host>:8200"
   aap_oidc_discovery_url: "https://<your-aap-host>/api/gateway/v1/jwt/"
   jwt_bound_audience: "https://<your-vault-host>:8200"
   ```
4. Launch the job template. The `vault_token` is injected securely from the credential — it never appears in the job log or Extra Variables.

---

## What It Configures

The playbook creates the following resources in Vault:

- **JWT auth method** mounted at `vault_jwt_mount_path` (default: `jwt`), configured to trust AAP 2.7 as the OIDC issuer using the AAP JWKS endpoint fetched from `aap_oidc_discovery_url`
- **ACL policy** (`vault_policy_name`) granting `read` and `list` on `vault_secret_path` and its children
- **JWT role** (`vault_jwt_role_name`) binding the `aud` claim to `jwt_bound_audience`, attaching the ACL policy, and enforcing a short token TTL
- **Sample KV v2 secret** at `vault_kv_secret_path` containing a `username` and `password` for use by the demo playbook

## Next Steps

Once this playbook has run successfully:

1. Configure the AAP side (if not already done) using [`../aap-config/`](../aap-config/README.md).
2. Run the end-to-end demo from [`../demo-playbook/read_vault_secret.yml`](../demo-playbook/README.md) as an AAP job template to verify the full JWT authentication flow.
