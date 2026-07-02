# HCP Vault Dedicated Configuration Playbook

This playbook is the HCP Vault Dedicated variant of [`examples/vault-config/`](../vault-config/README.md). It configures an HCP Vault Dedicated cluster for AAP OIDC JWT workload-identity authentication — enabling the JWT auth method, creating a policy and role, and seeding the demo secret.

The playbook is structurally identical to the self-managed version with exactly two HCP-specific additions: the `X-Vault-Namespace` header on every API call, and the `namespace` parameter on the KV write task.

---

## Prerequisites

- **Ansible >= 2.14**
- **`community.hashi_vault` collection >= 6.x:**
  ```bash
  ansible-galaxy collection install community.hashi_vault
  ```
- **An HCP Vault Dedicated cluster** — the cluster must be running and its public endpoint reachable from the machine running this playbook (port `8200`).
- **An HCP service principal token** with the Contributor role on the Vault cluster:
  1. Log in to the [HCP portal](https://portal.cloud.hashicorp.com)
  2. Navigate to your organisation → **Access Control (IAM)** → **Service Principals**
  3. Create or select a service principal and assign it the **Contributor** role on your Vault cluster
  4. Generate a client secret and exchange it for a token using the HCP API, or use the `hcp` CLI: `hcp auth login --client-id=<id> --client-secret=<secret>`
- Network reachability from the machine running this playbook to `https://<cluster-id>.vault.<region>.hashicorp.cloud:8200`

> **Note:** HCP Vault Dedicated does not expose a root token. Use the service principal token described above — it has sufficient permissions to enable auth methods and write policies.

---

## Key HCP Differences

| Setting | Self-Managed Vault | HCP Vault Dedicated |
|---|---|---|
| `vault_token` | Vault root or admin token | HCP service principal token |
| `vault_addr` | `https://<your-host>:8200` | `https://<cluster-id>.vault.<region>.hashicorp.cloud:8200` |
| `vault_namespace` | Not required | Required — set to `admin` |
| Vault API surface | Full | Full — `sys/auth`, `sys/policies`, KV v2 all supported |

---

## Variables

Edit `vars.yml` before running. All variables are required.

| Variable | Default | Description |
|---|---|---|
| `vault_addr` | `https://<cluster-id>.vault.<region>.hashicorp.cloud:8200` | HCP Vault Dedicated cluster URL (no trailing slash) |
| `vault_token` | `CHANGEME` | HCP service principal token — **change before use** |
| `vault_namespace` | `admin` | Vault namespace — root namespace on HCP Vault Dedicated |
| `aap_oidc_discovery_url` | `https://aap.example.com/api/gateway/v1/jwt/` | AAP 2.7 OIDC discovery URL |
| `vault_jwt_mount_path` | `jwt` | JWT auth method mount path |
| `vault_jwt_role_name` | `aap-automation` | JWT role name |
| `vault_policy_name` | `aap-automation-policy` | Vault ACL policy name |
| `vault_secret_path` | `secret/data/aap-demo/config` | Full KV v2 path the policy grants read access to |
| `vault_kv_mount` | `secret` | KV v2 engine mount name |
| `vault_kv_secret_path` | `aap-demo/config` | Secret path within the KV mount |
| `jwt_token_ttl` | `300s` | TTL for Vault tokens issued to AAP jobs |
| `jwt_bound_audience` | `https://<cluster-id>.vault.<region>.hashicorp.cloud:8200` | Required `aud` claim in the AAP JWT — set to your cluster URL |

---

## How to Run

```bash
# Edit variables for your environment
vi vars.yml

# Run the playbook (or pass the token at the command line)
ansible-playbook configure_hcp_vault_oidc.yml
ansible-playbook configure_hcp_vault_oidc.yml -e vault_token=<your-service-principal-token>
```

---

## What It Configures

All resources are created inside the `admin` namespace of your HCP Vault Dedicated cluster:

- **JWT auth method** mounted at `vault_jwt_mount_path` (default: `jwt`), configured to trust AAP 2.7 as the OIDC issuer
- **ACL policy** (`vault_policy_name`) granting `read` and `list` on `vault_secret_path` and its children
- **JWT role** (`vault_jwt_role_name`) binding the `aud` claim, attaching the ACL policy, and enforcing a short token TTL
- **Sample KV v2 secret** at `vault_kv_secret_path` containing a `username` and `password` for use by the HCP demo playbook
