# Demo Guide — Testing the AAP + Vault OIDC Integration

This guide covers the end-to-end steps to test this integration pattern. **All playbooks in this repo are run as AAP Job Templates** — there is no local `ansible-playbook` execution.

Follow the steps in order. Each step depends on the previous one being complete. Steps 1–3 are shared between both Vault deployment models. Step 4 branches into a self-managed Vault path or an HCP Vault Dedicated path.

---

## Prerequisites

Before starting, confirm:

- This repository is accessible to AAP as a **Git SCM source**
- You have **AAP 2.7** admin access
- You have either:
  - A **self-managed HashiCorp Vault 1.19+** instance with a bootstrap token that can enable auth methods, write policies, and write KV secrets, **or**
  - An **HCP Vault Dedicated** cluster and an HCP service principal token with the Contributor role
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

## Step 2 — Configure AAP

Run the AAP configuration playbook as a job template. It enables the OIDC issuer and creates the Vault JWT credential type and credential instance. **This step is shared — run it once regardless of which Vault deployment model you use.**

### Create the Job Template

1. Go to **Resources → Templates → Add → Add job template**
2. Set:

   | Field | Value |
   |---|---|
   | **Name** | `1 - Configure AAP Vault Credential` |
   | **Job Type** | `Run` |
   | **Inventory** | Any inventory that resolves `localhost` |
   | **Project** | `AAP Vault OIDC Integration Pattern` |
   | **Playbook** | `examples/aap-config/configure_aap_vault_oidc.yml` |
   | **Execution Environment** | Default EE (no extra collections required) |

3. Under **Credentials**, add a credential of type **Red Hat Ansible Automation Platform** configured with your AAP host, admin username, and admin password.

   > **Note:** AAP admin credentials are stored in the AAP platform credential — do not put them in Extra Variables.

4. In the **Extra Variables** field, supply the non-sensitive values:

   ```yaml
   vault_addr: "https://<your-vault-host>:8200"
   vault_jwt_mount_path: "jwt"
   vault_jwt_role_name: "aap-automation"
   ```

5. Click **Save**, then **Launch**

### Verify

After the job completes, confirm in the AAP web UI:

- **Resources → Credential Types** — types named `HashiCorp Vault JWT`, `Vault Bootstrap Token`, `HCP Vault Bootstrap Token`, and `AAP Admin Credential` exist
- **Resources → Credentials** — a credential named `Vault JWT - aap-automation` exists

> ⚠️ **OIDC issuer endpoint.** The playbook calls `PATCH /api/gateway/v1/settings/` to enable the AAP OIDC issuer. This is correct for AAP 2.7 platform-gateway deployments. On standalone controller-only deployments the endpoint may differ — see the [AAP 2.7 OIDC docs](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/whats_new-oidc_authentication_for_hashicorp_vault) and update the task if your job fails here.

---

## Step 3 — Configure Vault

Choose the path that matches your Vault deployment. If you are using **both** self-managed and HCP Vault Dedicated, complete each path independently.

---

### Path A — Self-Managed Vault

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

3. Under **Credentials**, add a credential of type **Vault Bootstrap Token** containing your Vault admin token. This credential is created by Step 2 — run that first.

4. In the **Extra Variables** field, supply the non-sensitive environment values:

   ```yaml
   vault_addr: "https://<your-vault-host>:8200"
   aap_oidc_discovery_url: "https://<your-aap-host>/api/gateway/v1/jwt/"
   jwt_bound_audience: "https://<your-vault-host>:8200"
   ```

   > The remaining variables (`vault_jwt_mount_path`, `vault_jwt_role_name`, `vault_policy_name`, etc.) can be left at their defaults unless you need to change mount paths or names. The `vault_token` is injected securely from the credential — do not add it here.

5. Click **Save**, then **Launch**

### Verify

After the job completes successfully, confirm in Vault:

- `vault auth list` shows `jwt/` is enabled
- `vault read auth/jwt/config` shows your AAP OIDC discovery URL
- `vault kv get secret/aap-demo/config` returns the demo secret

---

### Path B — HCP Vault Dedicated

Run the HCP Vault Dedicated configuration playbook as an AAP job template. It enables the JWT auth method in the `admin` namespace, creates the policy and role, and writes the demo secret.

#### Create the Job Template

1. Go to **Resources → Templates → Add → Add job template**
2. Set:

   | Field | Value |
   |---|---|
   | **Name** | `2b - Configure HCP Vault OIDC` |
   | **Job Type** | `Run` |
   | **Inventory** | Any inventory that resolves `localhost` |
   | **Project** | `AAP Vault OIDC Integration Pattern` |
   | **Playbook** | `examples/hcp-vault-config/configure_hcp_vault_oidc.yml` |
   | **Execution Environment** | An EE with `community.hashi_vault >= 6.x` |

3. Under **Credentials**, add a credential of type **HCP Vault Bootstrap Token** containing your HCP service principal token. This credential type is created by Step 2 — run that first.

4. In the **Extra Variables** field, supply the non-sensitive environment values:

   ```yaml
   vault_addr: "https://<cluster-id>.vault.<region>.hashicorp.cloud:8200"
   aap_oidc_discovery_url: "https://<your-aap-host>/api/gateway/v1/jwt/"
   jwt_bound_audience: "https://<cluster-id>.vault.<region>.hashicorp.cloud:8200"
   vault_namespace: "admin"
   ```

   > The `vault_token` is injected securely from the credential — do not add it here.

5. Click **Save**, then **Launch**

#### Verify

After the job completes, confirm in HCP Vault Dedicated (in the `admin` namespace):

- `vault auth list` shows `jwt/` is enabled
- `vault read auth/jwt/config` shows your AAP OIDC discovery URL
- `vault kv get secret/aap-demo/config` returns the demo secret

---

## Step 4 — Run the End-to-End Demo

Choose the path that matches your Vault deployment.

---

### Path A — Self-Managed Vault Demo

Run the demo playbook as an AAP job template. It uses the AAP-injected JWT token to authenticate to self-managed Vault and retrieve the demo secret — no static credentials.

#### Create the Job Template

1. Go to **Resources → Templates → Add → Add job template**
2. Set:

   | Field | Value |
   |---|---|
   | **Name** | `3a - Demo: Self-Managed Vault Secret via OIDC JWT` |
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

#### Expected Output

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

### Path B — HCP Vault Dedicated Demo

Run the HCP demo playbook as an AAP job template. It uses the AAP-injected JWT token to authenticate to HCP Vault Dedicated and retrieve the demo secret — no static credentials.

#### Create the Job Template

1. Go to **Resources → Templates → Add → Add job template**
2. Set:

   | Field | Value |
   |---|---|
   | **Name** | `3b - Demo: HCP Vault Secret via OIDC JWT` |
   | **Job Type** | `Run` |
   | **Inventory** | Any inventory that resolves `localhost` |
   | **Project** | `AAP Vault OIDC Integration Pattern` |
   | **Playbook** | `examples/hcp-demo-playbook/read_hcp_vault_secret.yml` |
   | **Execution Environment** | An EE with `community.hashi_vault >= 6.x` |
   | **Credentials** | Add `Vault JWT - aap-automation` (type: *HashiCorp Vault JWT*) |

3. In the **Extra Variables** field:

   ```yaml
   vault_addr: "https://<cluster-id>.vault.<region>.hashicorp.cloud:8200"
   vault_namespace: "admin"
   ```

4. Click **Save**, then **Launch**

#### Expected Output

A successful run looks like this:

```
TASK [Capture AAP-injected JWT token from environment] *************************
ok: [localhost]

TASK [Fail if AAP_JWT_TOKEN is not set] ****************************************
skipping: [localhost]

TASK [Authenticate to HCP Vault Dedicated using the AAP JWT token] *************
ok: [localhost]

TASK [Read secret from HCP Vault Dedicated] ************************************
ok: [localhost]

TASK [Confirm secret retrieval (keys only — values are not printed)] ***********
ok: [localhost] => {
    "msg": "Secret retrieved successfully from HCP Vault Dedicated path secret/data/aap-demo/config. Keys found: ['username', 'password']"
}
```

> The `vault_login` and `vault_read` tasks use `no_log: true` — secret values are never printed to the job output.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `AAP_JWT_TOKEN environment variable is not set` | OIDC issuer not enabled, or playbook run locally | Confirm Step 2 completed and the job is running as an AAP job template |
| Vault returns `401 permission denied` on login | `aud` claim mismatch | Confirm `jwt_bound_audience` in the Vault config job's extra vars exactly matches the `vault_addr` value; inspect the raw JWT at [jwt.io](https://jwt.io) |
| Vault returns `403 permission denied` on secret read | Policy path mismatch | Confirm `vault_secret_path` in the Vault config job's extra vars matches the path the demo playbook reads |
| Step 2 AAP config playbook fails at the OIDC issuer `PATCH` task | Wrong endpoint for your AAP topology | See the note in Step 2 above |
| Vault config playbook fails saying mount already exists | JWT auth method was partially configured previously | Run `vault auth disable jwt` and re-run the Vault config job template |
| HCP Vault returns `403` on any API call | Missing or wrong namespace | Confirm `vault_namespace: admin` is set in the HCP job template extra vars |
| HCP Vault returns `403` on login despite correct namespace | Service principal token expired or insufficient role | Regenerate the HCP service principal token and confirm it has the Contributor role on the cluster |
