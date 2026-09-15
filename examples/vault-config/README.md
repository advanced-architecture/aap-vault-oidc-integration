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
- **A Vault bootstrap token** with permissions to enable auth methods (`sys/auth`), write policies (`sys/policies/acl`), and write KV v2 secrets — injected via the **Vault Bootstrap Token** AAP credential (see [Bootstrap Credential Setup](#bootstrap-credential-setup) below)

In the case where a Vault server is needed, a Vault Dedicated cluster can be configured on portal.cloud.hashicorp.com/services/vault/clusters in a few clicks.  

This repo was tested with a Vault Dedicated cluster with this process:

1. Go to portal.cloud.hashicorp.com
1. Create project: "vault-ansible-oidc"
1. Click `Vault Dedicated` tile: Get started with Vault Dedicated
1. Click `Start from scratch`
1. Accept the defaults (AWS, Development tier, Extra Small, default network HVN)
1. Configure cluster ID to be unique within your HCP organization. I used "vault-cluster-ansible-oidc".
1. Choose `Start from scratch`
1. Enable reporting (currently beta)

---

## Variables

All variables are defined in [`vars.yml`](vars.yml). Override any of them at run time with `-e key=value`.

| Variable | Description | Required | Default |
|---|---|---|---|
| `vault_addr` | Vault server address (no trailing slash) | ✅ | `https://vault.example.com:8200` |
| `vault_token` | Bootstrap admin token — injected by the Vault Bootstrap Token credential | ✅ | *(credential injector)* |
| `aap_oidc_discovery_url` | AAP 2.7 OIDC issuer URL — use the `issuer` value from `curl -L <aap-host>/o/.well-known/openid-configuration/` | ✅ | `https://aap.example.com/o` |
| `vault_jwt_mount_path` | Mount path for the Vault JWT auth method | ✅ | `jwt` |
| `vault_jwt_role_name` | Name of the JWT role Vault creates for AAP jobs | ✅ | `aap-automation` |
| `vault_policy_name` | Name of the Vault ACL policy attached to the JWT role | ✅ | `aap-automation-policy` |
| `vault_secret_path` | Full KV v2 path the policy grants read access to | ✅ | `secret/data/aap-demo/config` |
| `vault_kv_mount` | KV v2 engine mount name (no `/data/` segment) | ✅ | `secret` |
| `vault_kv_secret_path` | Secret path within the KV mount (no mount prefix) | ✅ | `aap-demo/config` |
| `jwt_token_ttl` | TTL for Vault tokens issued to AAP jobs | ✅ | `300s` |
| `jwt_bound_audience` | Required `aud` claim value in the AAP JWT | ✅ | `https://vault.example.com:8200` |

---

## Bootstrap Credential Setup

The playbook receives the Vault bootstrap token via the **Vault Bootstrap Token** custom credential type. Create this once — it is shared by both the self-managed and HCP Vault config playbooks.

### 1 — Create the custom credential type

In AAP → **Resources → Credential Types → Add**:

| Field | Value |
|---|---|
| **Name** | `Vault Bootstrap Token` |
| **Kind** | `Cloud` |

**Input Configuration:**
```yaml
fields:
  - id: vault_token
    type: string
    label: Vault Token
    secret: true
required:
  - vault_token
```

**Injector Configuration:**
```yaml
env:
  VAULT_TOKEN: '{{ vault_token }}'
```

### 2 — Create a credential instance

In AAP → **Resources → Credentials → Add**:

| Field | Value |
|---|---|
| **Name** | `Vault Bootstrap Token - <your-cluster>` |
| **Credential Type** | `Vault Bootstrap Token` |
| **Vault Token** | your Vault admin/bootstrap token |

### 3 — Attach to the job template

When creating the job template for `configure_vault_oidc.yml`, add this credential under **Credentials**.

## How to Run

Create the job template in AAP with the `Vault Bootstrap Token` credential attached, supply the required extra vars, and launch:

```yaml
vault_addr: "https://<your-vault-host>:8200"
aap_oidc_discovery_url: "https://<your-aap-host>/o"
jwt_bound_audience: "https://<your-vault-host>:8200"
```

The remaining variables (`vault_jwt_mount_path`, `vault_jwt_role_name`, `vault_policy_name`, etc.) default to sensible values in `vars.yml`.

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
