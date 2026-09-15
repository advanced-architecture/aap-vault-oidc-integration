# Vault OIDC Secret Lookup — AAP-Native Workflow

This example demonstrates the **AAP-native OIDC secret lookup** pattern, which is distinct from the JWT-injection demo in `../hcp-demo-playbook/`.

| Pattern | How it works | Playbook complexity |
|---|---|---|
| **JWT injection** (`hcp-demo-playbook`) | AAP injects a JWT env var; playbook calls Vault API directly | Playbook handles login + read |
| **OIDC secret lookup** (this example) | AAP fetches the secret from Vault transparently before the job runs; value injected via credential linkage | Playbook just uses the injected var |

---

## How the AAP-native lookup works

```
Job launch
  │
  ├── AAP issues a short-lived JWT (OIDC workload identity)
  ├── AAP authenticates to Vault using the JWT (HashiCorp Vault Secret Lookup credential)
  ├── AAP fetches the secret value from Vault KV
  └── AAP injects the value into the job via credential field linkage
        │
        └── Playbook runs — secret already present as extra var, no Vault calls needed
```

This is the recommended pattern when a playbook needs a specific secret value injected before it runs. The JWT-injection pattern (`hcp-demo-playbook`) is better when the playbook needs to make multiple dynamic Vault lookups at runtime.

---

## Prerequisites

- The Vault JWT auth method and KV secret are already configured — run `examples/hcp-vault-config/configure_hcp_vault_oidc.yml` first.
- The `configure_vault_secret_lookup.yml` playbook has been run to create the `Vault OIDC Lookup` credential.
- A credential field has been linked to the Vault lookup credential in the AAP UI (see [Step 2](#step-2--link-the-credential-field-in-the-aap-ui) below).

---

## Setup

### Step 1 — Run the configuration playbook

Create the job template for `configure_vault_secret_lookup.yml`:

| Field | Value |
|---|---|
| **Name** | `4 - Configure Vault OIDC Secret Lookup` |
| **Playbook** | `examples/vault-secret-lookup/configure_vault_secret_lookup.yml` |
| **Credentials** | `AAP Admin - sandbox` (type: *Red Hat Ansible Automation Platform*) |

**Extra Variables:**
```yaml
vault_addr: "https://<cluster-id>.vault.<region>.hashicorp.cloud:8200"
vault_jwt_mount_path: "jwt"
vault_jwt_role_name: "aap-automation"
vault_namespace: "admin"
```

After it runs, the credential **`Vault OIDC Lookup - aap-automation`** (type: *HashiCorp Vault Secret Lookup*) will exist in AAP.

---

### Step 2 — Link the credential field in the AAP UI

The `HashiCorp Vault Secret Lookup` credential cannot be attached directly to a job template to inject a value as an extra var. Instead, link it as the source for a field in another credential:

1. AAP UI → **Resources → Credentials** → open any credential that has a field you want to populate from Vault (e.g. a Machine credential's **Password** field, or a custom credential)
2. Click **Edit**
3. Next to the target field, click the **key icon** 🔑
4. In the dialog:
   - **Credential:** select `Vault OIDC Lookup - aap-automation`
   - **Secret Backend:** the KV mount name (e.g. `secret`)
   - **Path to Secret:** the path within the mount (e.g. `aap-demo/config`)
   - **Key Name:** the key to retrieve (e.g. `username`)
5. Click **OK**, then **Save**

The linked credential is now populated from Vault at job launch time via OIDC — no static value stored in AAP.

---

### Step 3 — Create and launch the demo job template

| Field | Value |
|---|---|
| **Name** | `5 - Demo: Vault OIDC Secret Lookup` |
| **Playbook** | `examples/vault-secret-lookup/demo_vault_secret_lookup.yml` |
| **Credentials** | The credential with the Vault-linked field (from Step 2) |

---

## Expected Output

```
TASK [Fail if hashicorp_vault_value was not injected] **************************
skipping: [localhost]

TASK [Confirm secret was retrieved and injected by AAP] ************************
ok: [localhost] => {
    "msg": "Secret successfully retrieved from Vault via AAP OIDC lookup and injected as extra var. Value length: 9 chars. (Value is not printed — use no_log: true on any task that consumes it.)"
}
```

---

## Variables

| Variable | Description | Default |
|---|---|---|
| `vault_addr` | HCP Vault cluster URL | `https://<cluster-id>.vault.<region>.hashicorp.cloud:8200` |
| `vault_jwt_mount_path` | JWT auth mount path in Vault | `jwt` |
| `vault_jwt_role_name` | JWT role name in Vault | `aap-automation` |
| `vault_namespace` | Vault namespace (`admin` for HCP Vault Dedicated) | `admin` |
| `aap_organization` | AAP organization to own the credentials | `Default` |

---

## Reference

- [Just-in-time access to HashiCorp Vault using the Red Hat Ansible Automation Platform OIDC provider](https://developers.redhat.com/articles/2026/08/11/just-in-time-access-to-hashicorp-vault-with-ansible-oidc-provider)
- [AAP 2.7 — OIDC authentication for HashiCorp Vault](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/whats_new-oidc_authentication_for_hashicorp_vault)
