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

Run the AAP configuration playbook as a job template. It creates the Vault JWT credential type and credential instance, **and automatically creates all remaining job templates** (Steps 3–5). **This step is shared — run it once regardless of which Vault deployment model you use.**

> **Before running this step**, verify that the AAP OIDC issuer is active:
> ```bash
> curl -L ${CONTROLLER_HOST}/o/.well-known/openid-configuration/
> ```
> A valid OIDC discovery document confirms the issuer is live. The playbook does not enable the issuer — it is on by default in AAP 2.7 platform-gateway deployments.

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

3. Under **Credentials**, attach the `AAP Admin Credential` instance (type: *AAP Admin Credential*). This injects `controller_host`, `controller_username`, `controller_password`, and `controller_verify_ssl` as extra vars.

4. In the **Extra Variables** field, supply the Vault connection and project values:

   ```yaml
   vault_addr: "https://<your-vault-host>:8200"
   vault_jwt_mount_path: "jwt"
   vault_jwt_role_name: "aap-automation"
   aap_project_name: "AAP Vault OIDC Integration Pattern"
   aap_inventory_name: "Demo Inventory"
   ```

   > `aap_project_name` must exactly match the project name from Step 1. `aap_inventory_name` must match an inventory in AAP that resolves `localhost`. Both default to the values above in `vars.yml`.

5. Click **Save**, then **Launch**

### Verify

After the job completes, confirm in the AAP web UI:

- **Resources → Credential Types** — types named `HashiCorp Vault JWT` and `Vault Bootstrap Token` exist
- **Resources → Credentials** — a credential named `Vault JWT - aap-automation` exists
- **Resources → Templates** — job templates for Steps 2–5 are created automatically:
  - `2 - Configure HCP Vault OIDC`
  - `2a - Configure Vault OIDC (self-managed)`
  - `3a - Demo: Self-Managed Vault Secret via OIDC JWT`
  - `3b - Demo: HCP Vault Secret via OIDC JWT`
  - `4 - Configure Vault OIDC Secret Lookup`
  - `5 - Demo: Vault OIDC Secret Lookup`

> If the `Vault Bootstrap Token - HCP` credential already exists in AAP when this playbook runs, it will be attached automatically to the Vault config templates. Otherwise, attach it manually after creating the credential instance (see Step 3).


---

## Step 3 — Configure Vault

Choose the path that matches your Vault deployment. If you are using **both** self-managed and HCP Vault Dedicated, complete each path independently.

> The job templates for this step were created automatically by Step 2. You only need to supply credentials and extra variables, then launch.

---

### Path A — Self-Managed Vault

Run the Vault configuration playbook as an AAP job template. It enables the JWT auth method, creates the policy and role, and writes the demo secret.

1. Go to **Resources → Templates** → open `2a - Configure Vault OIDC (self-managed)`
2. Under **Credentials**, attach the `Vault Bootstrap Token - <your-cluster>` credential (type: *Vault Bootstrap Token*). This injects `VAULT_TOKEN` as an environment variable (masked in log output).
3. In the **Extra Variables** field, supply the environment-specific values:

   ```yaml
   vault_addr: "https://<your-vault-host>:8200"
   aap_oidc_discovery_url: "https://<your-aap-host>/o"
   jwt_bound_audience: "https://<your-vault-host>:8200"
   ```

   > The remaining variables (`vault_jwt_mount_path`, `vault_jwt_role_name`, `vault_policy_name`, etc.) default to sensible values in `vars.yml`.

4. Click **Save**, then **Launch**

### Verify

After the job completes successfully, confirm in Vault:

- `vault auth list` shows `jwt/` is enabled
- `vault read auth/jwt/config` shows your AAP OIDC discovery URL
- `vault kv get secret/aap-demo/config` returns the demo secret

---

### Path B — HCP Vault Dedicated

Run the HCP Vault Dedicated configuration playbook as an AAP job template. It enables the JWT auth method in the `admin` namespace, creates the policy and role, and writes the demo secret.

1. Go to **Resources → Templates** → open `2 - Configure HCP Vault OIDC`
2. Under **Credentials**, attach the `Vault Bootstrap Token - HCP` credential (type: *Vault Bootstrap Token*). This injects `VAULT_TOKEN` as an environment variable (masked in log output).
3. In the **Extra Variables** field, supply the environment-specific values:

   ```yaml
   vault_addr: "https://<cluster-id>.vault.<region>.hashicorp.cloud:8200"
   aap_oidc_discovery_url: "https://<your-aap-host>/o"
   jwt_bound_audience: "https://<cluster-id>.vault.<region>.hashicorp.cloud:8200"
   vault_namespace: "admin"
   ```

4. Click **Save**, then **Launch**

#### Verify

After the job completes, confirm in HCP Vault Dedicated (in the `admin` namespace):

- `vault auth list` shows `jwt/` is enabled
- `vault read auth/jwt/config` shows your AAP OIDC discovery URL
- `vault kv get secret/aap-demo/config` returns the demo secret

---

## Step 4 — Run the End-to-End Demo

Choose the path that matches your Vault deployment.

> The job templates for this step were created automatically by Step 2 and the `Vault JWT - aap-automation` credential is attached automatically where it already existed. Verify the credential attachment before launching.

---

### Path A — Self-Managed Vault Demo

The demo playbook uses the AAP-injected JWT token to authenticate to self-managed Vault and retrieve the demo secret — no static credentials.

1. Go to **Resources → Templates** → open `3a - Demo: Self-Managed Vault Secret via OIDC JWT`
2. Confirm `Vault JWT - aap-automation` (type: *HashiCorp Vault JWT*) is listed under **Credentials**
3. No extra variables needed — `vault_addr` is injected by the credential
4. Click **Launch**

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

The HCP demo playbook uses the AAP-injected JWT token to authenticate to HCP Vault Dedicated and retrieve the demo secret — no static credentials.

1. Go to **Resources → Templates** → open `3b - Demo: HCP Vault Secret via OIDC JWT`
2. Confirm `Vault JWT - aap-automation` (type: *HashiCorp Vault JWT*) is listed under **Credentials**
3. No extra variables needed — `vault_addr` and `vault_namespace` are injected by the credential
4. Click **Launch**

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

---

## Step 5 — AAP-Native Vault Secret Lookup (Optional)

This step demonstrates the **AAP-native OIDC secret lookup** pattern. Unlike Steps 3–4 where the playbook calls Vault directly, here AAP fetches the secret *before* the job runs and injects it as an extra var. The playbook never handles tokens or Vault API calls.

| Pattern | When to use |
|---|---|
| **JWT injection** (Steps 3–4) | Playbook needs to make multiple dynamic Vault lookups at runtime |
| **OIDC secret lookup** (this step) | Playbook needs a specific secret value available at startup — simpler playbooks |

### Step 5a — Create the Vault OIDC Lookup Credential

1. Go to **Resources → Templates** → open `4 - Configure Vault OIDC Secret Lookup`
2. Under **Credentials**, confirm the `AAP Admin Credential` instance is attached
3. In the **Extra Variables** field, supply the Vault connection values:

   ```yaml
   vault_addr: "https://<cluster-id>.vault.<region>.hashicorp.cloud:8200"
   vault_jwt_mount_path: "jwt"
   vault_jwt_role_name: "aap-automation"
   vault_namespace: "admin"
   ```

4. Click **Save**, then **Launch**

After the job completes, the credential **`Vault OIDC Lookup - aap-automation`** (type: *HashiCorp Vault Secret Lookup*) will exist in AAP.

### Step 5b — Link a Credential Field to Vault

The lookup credential cannot inject a value directly — it must be linked to a field on another credential:

1. AAP UI → **Resources → Credentials** → open any credential with a field you want to populate from Vault (e.g. a Machine credential's **Password** field)
2. Click **Edit** → next to the target field click the **key icon** 🔑
3. In the dialog:
   - **Credential:** `Vault OIDC Lookup - aap-automation`
   - **Secret Backend:** `secret`
   - **Path to Secret:** `aap-demo/config`
   - **Key Name:** `username` (or `password`)
4. Click **OK**, then **Save**

### Step 5c — Run the Demo

1. Go to **Resources → Templates** → open `5 - Demo: Vault OIDC Secret Lookup`
2. Under **Credentials**, attach the credential with the Vault-linked field from Step 5b
3. Click **Launch**

#### Expected Output

```
TASK [Fail if hashicorp_vault_value was not injected] **************************
skipping: [localhost]

TASK [Confirm secret was retrieved and injected by AAP] ************************
ok: [localhost] => {
    "msg": "Secret successfully retrieved from Vault via AAP OIDC lookup and injected as extra var. Value length: 9 chars. (Value is not printed — use no_log: true on any task that consumes it.)"
}
```

See [`examples/vault-secret-lookup/README.md`](examples/vault-secret-lookup/README.md) for full setup details.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `AAP_JWT_TOKEN environment variable is not set` | OIDC issuer not active, or playbook run locally | Run `curl -L ${CONTROLLER_HOST}/o/.well-known/openid-configuration/` to confirm the issuer is live; ensure the job is running as an AAP job template |
| Vault returns `401 permission denied` on login | `aud` claim mismatch | Confirm `jwt_bound_audience` in the Vault config job's extra vars exactly matches the `vault_addr` value; inspect the raw JWT at [jwt.io](https://jwt.io) |
| Vault returns `403 permission denied` on secret read | Policy path mismatch | Confirm `vault_secret_path` in the Vault config job's extra vars matches the path the demo playbook reads |
| Vault config playbook fails saying mount already exists | JWT auth method was partially configured previously | Run `vault auth disable jwt` and re-run the Vault config job template |
| Step 2 reports `Project 'AAP Vault OIDC Integration Pattern' not found` | Project name mismatch or sync not yet complete | Confirm the project synced green in Step 1 and that `aap_project_name` in extra vars matches exactly |
| Step 2 reports `Inventory 'Demo Inventory' not found` | Inventory name mismatch | Update `aap_inventory_name` in extra vars to match an inventory that resolves `localhost` |
| `configure_aap_vault_oidc.yml` needs to update an existing credential type schema | AAP does not update in place on POST | Delete the existing `HashiCorp Vault JWT` credential type and any credentials using it in AAP, then re-run the playbook |
| HCP Vault returns `403` on any API call | Missing or wrong namespace | Confirm `vault_namespace: admin` is set in the HCP job template extra vars |
| HCP Vault returns `403` on login despite correct namespace | Service principal token expired or insufficient role | Regenerate the HCP service principal token and confirm it has the Contributor role on the cluster |
