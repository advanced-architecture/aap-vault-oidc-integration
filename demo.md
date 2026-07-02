# Demo Guide — Testing the AAP + Vault OIDC Integration

This guide covers the end-to-end steps to test this integration pattern. All three playbooks in this repo are run as **AAP Job Templates** — there is no local `ansible-playbook` execution.

Follow the stages in order. Each stage depends on the previous one being complete.

---

## Prerequisites

Before starting, confirm:

- This repository is accessible to AAP as a **Git SCM source**
- You have **AAP 2.7** admin access
- You have a **self-managed HashiCorp Vault 1.9+** instance with a bootstrap token that can enable auth methods, write policies, and write KV secrets
- The AAP execution environment used by the job templates includes **`community.hashi_vault >= 6.x`**
- Network reachability:
  - AAP execution nodes → Vault API (port `8200`)
  - Vault server → AAP OIDC/JWKS endpoint (`https://<aap-host>/api/gateway/v1/jwks/`)

---

## Step 1 — Add This Repo as an AAP Project

1. In the AAP web UI go to **Resources → Projects → Add**
2. Set:
   - **Name:** `AAP Vault OIDC Integration Pattern`
   - **SCM Type:** `Git`
   - **SCM URL:** URL of this repository
3. Click **Save** and wait for the sync to show a green status before continuing

---

## Step 2 — Configure Vault

Run the Vault configuration playbook as an AAP job template. It enables the JWT auth method, creates the policy and role, and writes the demo secret.

### Create the Job Template

1. Go to **Resources → Templates → Add → Add job template**
2. Set:

   | Field | Value |
   |---|---|
   | **Name** | `1 - Configure Vault OIDC` |
   | **Job Type** | `Run` |
   | **Inventory** | Any inventory that resolves `localhost` |
   | **Project** | `AAP Vault OIDC Integration Pattern` |
   | **Playbook** | `examples/vault-config/configure_vault_oidc.yml` |
   | **Execution Environment** | An EE with `community.hashi_vault >= 6.x` |

3. In the **Extra Variables** field, supply your environment values (these override the placeholders in `vars.yml`):

   ```yaml
   vault_addr: "https://<your-vault-host>:8200"
   vault_token: "<your-bootstrap-token>"
   aap_oidc_discovery_url: "https://<your-aap-host>/api/gateway/v1/jwt/"
   jwt_bound_audience: "https://<your-vault-host>:8200"
   ```

   > The remaining variables (`vault_jwt_mount_path`, `vault_jwt_role_name`, `vault_policy_name`, etc.) can be left at their defaults unless you need to change mount paths or names.

4. Click **Save**, then **Launch**

### Verify

After the job completes successfully, confirm in Vault:

- `vault auth list` shows `jwt/` is enabled
- `vault read auth/jwt/config` shows your AAP OIDC discovery URL
- `vault kv get secret/aap-demo/config` returns the demo secret

---

## Step 3 — Configure AAP

Run the AAP configuration playbook as a job template. It enables the OIDC issuer and creates the Vault JWT credential type and credential instance.

### Create the Job Template

1. Go to **Resources → Templates → Add → Add job template**
2. Set:

   | Field | Value |
   |---|---|
   | **Name** | `2 - Configure AAP Vault Credential` |
   | **Job Type** | `Run` |
   | **Inventory** | Any inventory that resolves `localhost` |
   | **Project** | `AAP Vault OIDC Integration Pattern` |
   | **Playbook** | `examples/aap-config/configure_aap_vault_oidc.yml` |
   | **Execution Environment** | Default EE (no extra collections required) |

3. In the **Extra Variables** field:

   ```yaml
   aap_host: "<your-aap-host>"
   aap_username: "admin"
   aap_password: "<your-aap-admin-password>"
   aap_validate_certs: false   # set to true if using valid TLS certs
   vault_addr: "https://<your-vault-host>:8200"
   ```

4. Click **Save**, then **Launch**

### Verify

After the job completes, confirm in the AAP web UI:

- **Resources → Credential Types** — a type named `HashiCorp Vault JWT` exists
- **Resources → Credentials** — a credential named `Vault JWT - aap-automation` exists

> ⚠️ **OIDC issuer endpoint.** The playbook calls `PATCH /api/gateway/v1/settings/` to enable the AAP OIDC issuer. This is correct for AAP 2.7 platform-gateway deployments. On standalone controller-only deployments the endpoint may differ — see the [AAP 2.7 OIDC docs](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/whats_new-oidc_authentication_for_hashicorp_vault) and update the task if your job fails here.

---

## Step 4 — Run the End-to-End Demo

Run the demo playbook as an AAP job template. It uses the AAP-injected JWT token to authenticate to Vault and retrieve the demo secret — no static credentials.

### Create the Job Template

1. Go to **Resources → Templates → Add → Add job template**
2. Set:

   | Field | Value |
   |---|---|
   | **Name** | `3 - Demo: Vault Secret via OIDC JWT` |
   | **Job Type** | `Run` |
   | **Inventory** | Any inventory that resolves `localhost` |
   | **Project** | `AAP Vault OIDC Integration Pattern` |
   | **Playbook** | `examples/demo-playbook/read_vault_secret.yml` |
   | **Execution Environment** | An EE with `community.hashi_vault >= 6.x` |
   | **Credentials** | Add `Vault JWT - aap-automation` (type: *HashiCorp Vault JWT*) |

3. In the **Extra Variables** field (only if your values differ from the defaults in `vars.yml`):

   ```yaml
   vault_addr: "https://<your-vault-host>:8200"
   ```

4. Click **Save**, then **Launch**

### Expected Output

A successful run looks like this:

```
TASK [Capture AAP-injected JWT token from environment] *************************
ok: [localhost]

TASK [Fail if AAP_JWT_TOKEN is not set] ****************************************
skipping: [localhost]

TASK [Authenticate to Vault using the AAP JWT token] ***************************
ok: [localhost]

TASK [Read secret from Vault] **************************************************
ok: [localhost]

TASK [Confirm secret retrieval (keys only — values are not printed)] ***********
ok: [localhost] => {
    "msg": "Secret retrieved successfully from secret/data/aap-demo/config. Keys found: ['username', 'password']"
}
```

> The `vault_login` and `vault_read` tasks use `no_log: true` — secret values are never printed to the job output.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `AAP_JWT_TOKEN environment variable is not set` | OIDC issuer not enabled, or playbook run locally | Confirm Step 3 completed and the job is running as an AAP job template |
| Vault returns `401 permission denied` on login | `aud` claim mismatch | Confirm `jwt_bound_audience` in the Vault config job's extra vars exactly matches the `vault_addr` value; inspect the raw JWT at [jwt.io](https://jwt.io) |
| Vault returns `403 permission denied` on secret read | Policy path mismatch | Confirm `vault_secret_path` in the Vault config job's extra vars matches the path the demo playbook reads |
| Step 3 playbook fails at the OIDC issuer `PATCH` task | Wrong endpoint for your AAP topology | See the note in Step 3 above |
| Vault config playbook fails saying mount already exists | JWT auth method was partially configured previously | Run `vault auth disable jwt` and re-run the Step 2 job template |
