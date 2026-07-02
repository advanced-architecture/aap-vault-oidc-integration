# HCP Vault Dedicated Configuration Playbook

This playbook configures the HCP Vault Dedicated side of the AAP OIDC/JWT integration. It is the HCP Vault Dedicated variant of [`../vault-config/`](../vault-config/README.md) — the task structure is identical, with two additions: an `X-Vault-Namespace` header on every Vault API call and a `namespace` parameter on the KV v2 write task. Run this once (or idempotently re-run) before the HCP demo playbook can authenticate to HCP Vault Dedicated using JWT workload identity.

---

## Prerequisites

- **Ansible** >= 2.14
- **`community.hashi_vault` collection** >= 6.x:
  ```bash
  ansible-galaxy collection install community.hashi_vault
  ```
- **An HCP account** with an active HCP Vault Dedicated cluster
- **An HCP service principal token** with the **Contributor** role on the Vault cluster:
  1. Log in to the [HCP portal](https://portal.cloud.hashicorp.com)
  2. Navigate to **Access Control** → **Service Principals**
  3. Select your service principal (or create one)
  4. Click **Generate token** and copy the token — it is shown only once
- **Network access** from the machine running this playbook to the HCP Vault Dedicated cluster public endpoint (`vault_addr`)

---

## Key HCP Differences

The following table summarises what differs from the self-managed Vault configuration in [`../vault-config/`](../vault-config/README.md):

| Variable | HCP Vault Dedicated | Self-Managed Vault |
|---|---|---|
| `vault_namespace` | **Required**, default `admin` — every API call must target a namespace | Not used |
| `vault_token` | HCP service principal token (Contributor role) — **not a root token** | Bootstrap admin/root token |
| `vault_addr` | `https://<cluster-id>.vault.<region>.hashicorp.cloud:8200` | `https://vault.example.com:8200` (operator-defined) |

---

## Variables

All variables are defined in [`vars.yml`](vars.yml). Override any of them at run time with `-e key=value`.

| Variable | Description | Required | Default |
|---|---|---|---|
| `vault_addr` | HCP Vault Dedicated cluster URL | ✅ | `https://<cluster-id>.vault.<region>.hashicorp.cloud:8200` |
| `vault_token` | HCP service principal token | ✅ | `CHANGEME` |
| `vault_namespace` | Vault namespace (HCP root namespace) | ✅ | `admin` |
| `aap_oidc_discovery_url` | AAP 2.7 OIDC discovery URL (`/api/gateway/v1/jwt/`) | ✅ | `https://aap.example.com/api/gateway/v1/jwt/` |
| `vault_jwt_mount_path` | Mount path for the Vault JWT auth method | ✅ | `jwt` |
| `vault_jwt_role_name` | Name of the JWT role Vault creates for AAP jobs | ✅ | `aap-automation` |
| `vault_policy_name` | Name of the Vault ACL policy attached to the JWT role | ✅ | `aap-automation-policy` |
| `vault_secret_path` | Full KV v2 path the policy grants read access to | ✅ | `secret/data/aap-demo/config` |
| `vault_kv_mount` | KV v2 engine mount name (no `/data/` segment) | ✅ | `secret` |
| `vault_kv_secret_path` | Secret path within the KV mount (no mount prefix) | ✅ | `aap-demo/config` |
| `jwt_token_ttl` | TTL for Vault tokens issued to AAP jobs | ✅ | `300s` |
| `jwt_bound_audience` | Required `aud` claim value in the AAP JWT | ✅ | `https://<cluster-id>.vault.<region>.hashicorp.cloud:8200` |

---

## How to Run

Update `vars.yml` with your environment values, then run:

```bash
ansible-playbook configure_hcp_vault_oidc.yml
# or pass the HCP service principal token directly without storing it in vars.yml:
ansible-playbook configure_hcp_vault_oidc.yml -e vault_token=<your-hcp-sp-token>
```

---

## What It Configures

The playbook creates the following resources in the `admin` namespace of your HCP Vault Dedicated cluster:

- **JWT auth method** mounted at `vault_jwt_mount_path` (default: `jwt`), configured to trust AAP 2.7 as the OIDC issuer using the JWKS endpoint fetched from `aap_oidc_discovery_url`
- **ACL policy** (`vault_policy_name`) granting `read` and `list` on `vault_secret_path` and its children, scoped to the `admin` namespace
- **JWT role** (`vault_jwt_role_name`) binding the `aud` claim to `jwt_bound_audience`, attaching the ACL policy, and enforcing a short token TTL
- **Sample KV v2 secret** at `vault_kv_secret_path` containing a `username` and `password` for use by the HCP demo playbook
