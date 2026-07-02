# Demo Playbook — End-to-End Vault Secret Retrieval via AAP OIDC JWT

This playbook (`read_vault_secret.yml`) demonstrates the complete OIDC JWT workload-identity flow: AAP injects a short-lived JWT token into each job run; the playbook uses that token to authenticate to Vault and retrieve a secret — **no static credentials required**.

---

## Prerequisites

- **AAP 2.7+** — OIDC issuer enabled and configured (see [`examples/aap-config/README.md`](../aap-config/README.md)).
- **HashiCorp Vault 1.19+** — JWT auth method enabled and JWT role created (see [`examples/vault-config/README.md`](../vault-config/README.md)).
- **`community.hashi_vault` collection ≥ 6.x** installed in the execution environment used by the AAP job template.
- Network reachability between the AAP execution environment and the Vault server on the configured port (default `8200`).

> **This playbook cannot be run locally** with `ansible-playbook`. The `AAP_JWT_TOKEN` environment variable is only injected by AAP at job-run time.

---

## How to Import into AAP

1. **Create a Project** in AAP pointing to this repository (SCM type: Git, SCM URL: `<your repo URL>`).
2. **Create a Job Template** with the following settings:
   - **Playbook:** `examples/demo-playbook/read_vault_secret.yml`
   - **Inventory:** `localhost` (or any inventory — the playbook runs on `localhost`)
   - **Execution Environment:** an EE that includes `community.hashi_vault >= 6.x`
3. **Attach the credential** created in the aap-config step (type: *HashiCorp Vault JWT*) to the Job Template.
4. **Set extra vars** (optional) — override any variable from `vars.yml` directly in the Job Template's *Extra Variables* field if the defaults do not match your environment. For example:
   ```yaml
   vault_addr: "https://vault.internal.example.com:8200"
   vault_secret_path: "secret/data/myteam/config"
   ```
5. **Launch the job** and review the output.

---

## Expected Output

A successful run prints a debug message confirming the secret was retrieved and listing the secret keys (never the values):

```
TASK [Confirm secret retrieval (keys only — values are not printed)] ***********
ok: [localhost] => {
    "msg": "Secret retrieved successfully from secret/data/aap-demo/config. Keys found: ['username', 'password']"
}
```

---

## Variables

| Variable | Default | Description |
|---|---|---|
| `vault_addr` | `https://vault.example.com:8200` | Vault server URL (no trailing slash) |
| `vault_jwt_mount_path` | `jwt` | JWT auth method mount path in Vault |
| `vault_jwt_role_name` | `aap-automation` | JWT role name in Vault |
| `vault_secret_path` | `secret/data/aap-demo/config` | KV v2 path to read (`<mount>/data/<path>`) |

---

## Note on `AAP_JWT_TOKEN`

`AAP_JWT_TOKEN` is the expected environment variable name through which AAP 2.7 injects the per-job OIDC JWT token into the execution environment. **Confirm the exact variable name during live testing** against your AAP instance — it may differ depending on configuration or future AAP releases. If the name differs, update the `set_fact` task in `read_vault_secret.yml` accordingly.
