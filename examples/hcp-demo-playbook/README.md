# HCP Demo Playbook — End-to-End HCP Vault Dedicated Secret Retrieval via AAP OIDC JWT

This playbook (`read_hcp_vault_secret.yml`) is the HCP Vault Dedicated variant of the [self-managed demo](../demo-playbook/README.md). It demonstrates the same OIDC JWT workload-identity flow — AAP injects a short-lived JWT token into each job run; the playbook uses that token to authenticate to **HCP Vault Dedicated** and retrieve a secret — **no static credentials required**.

The only code-level difference from the self-managed demo is the addition of `namespace: "{{ vault_namespace }}"` on the `vault_login` and `vault_read` tasks, targeting the `admin` namespace required by HCP Vault Dedicated.

---

## Prerequisites

- **AAP 2.7+** — OIDC issuer enabled and configured (see [`examples/aap-config/README.md`](../aap-config/README.md)) — identical setup to self-managed Vault.
- **HCP Vault Dedicated cluster** — JWT auth method, policy, role, and demo secret configured using [`examples/hcp-vault-config/README.md`](../hcp-vault-config/README.md).
- **`community.hashi_vault` collection ≥ 6.x** installed in the execution environment used by the AAP job template.
- Network reachability from AAP execution nodes to the HCP Vault Dedicated cluster public endpoint (port `8200`).

> **This playbook cannot be run locally** with `ansible-playbook`. The `AAP_JWT_TOKEN` environment variable is only injected by AAP at job-run time.

---

## The One Difference from Self-Managed

> **`vault_namespace: admin` is required.**
>
> HCP Vault Dedicated clusters use Vault namespaces. The root namespace is named `admin`. Both `vault_login` and `vault_read` tasks in this playbook pass `namespace: "{{ vault_namespace }}"` so all API calls are correctly scoped. The default value is `admin` — change it only if your cluster uses a custom child namespace.

---

## How to Import into AAP

1. **Create a Project** in AAP pointing to this repository (SCM type: Git, SCM URL: `<your repo URL>`).
2. **Create a Job Template** with the following settings:
   - **Playbook:** `examples/hcp-demo-playbook/read_hcp_vault_secret.yml`
   - **Inventory:** `localhost` (or any inventory — the playbook runs on `localhost`)
   - **Execution Environment:** an EE that includes `community.hashi_vault >= 6.x`
3. **Attach the credential** created in the aap-config step (type: *HashiCorp Vault JWT*) to the Job Template.
4. **Set extra vars** — at minimum, override `vault_addr` with your HCP Vault Dedicated cluster URL:
   ```yaml
   vault_addr: "https://<cluster-id>.vault.<region>.hashicorp.cloud:8200"
   vault_namespace: "admin"
   ```
5. **Launch the job** and review the output.

---

## Expected Output

A successful run prints a debug message confirming the secret was retrieved and listing the secret keys (never the values):

```
TASK [Confirm secret retrieval (keys only — values are not printed)] ***********
ok: [localhost] => {
    "msg": "Secret retrieved successfully from HCP Vault Dedicated path secret/data/aap-demo/config. Keys found: ['username', 'password']"
}
```

---

## Variables

| Variable | Default | Description |
|---|---|---|
| `vault_addr` | `https://<cluster-id>.vault.<region>.hashicorp.cloud:8200` | HCP Vault Dedicated cluster URL (no trailing slash) |
| `vault_jwt_mount_path` | `jwt` | JWT auth method mount path in Vault |
| `vault_jwt_role_name` | `aap-automation` | JWT role name in Vault |
| `vault_secret_path` | `secret/data/aap-demo/config` | KV v2 path to read (`<mount>/data/<path>`) |
| `vault_namespace` | `admin` | Vault namespace — must be `admin` for HCP Vault Dedicated root namespace |

---

## Note on `AAP_JWT_TOKEN`

`AAP_JWT_TOKEN` is the expected environment variable name through which AAP 2.7 injects the per-job OIDC JWT token into the execution environment. **Confirm the exact variable name during live testing** against your AAP instance — it may differ depending on configuration or future AAP releases. If the name differs, update the `set_fact` task in `read_hcp_vault_secret.yml` accordingly.
