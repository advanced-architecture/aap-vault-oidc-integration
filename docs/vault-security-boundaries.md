# Vault Security Boundaries with AAP OIDC Workload Identity

> **Source:** AAP 2.7 claims reference + HashiCorp Vault JWT auth method documentation  
> **Date:** 2026-09-14  
> **Scope:** Design guidance — JWT claim → Vault role/policy mapping patterns

---

## 1. AAP 2.7 JWT Claim Groups

Every job launched by AAP 2.7 produces a short-lived, signed JWT. The payload
contains the following claims, organised by the security boundary they represent:

| Boundary | Claim | Description |
|---|---|---|
| **User** | `aap_controller_launched_by_name` | Username or system entity that initiated the job |
| **User** | `aap_controller_launched_by_id` | Unique identifier of the launching user/entity |
| **Application / Project** | `aap_controller_project_name` | Name of the AAP project |
| **Application / Project** | `aap_controller_project_id` | Unique identifier of the AAP project |
| **Application / Project** | `aap_controller_job_template_name` | Name of the job template |
| **Application / Project** | `aap_controller_job_template_id` | Unique identifier of the job template |
| **System / Hostname** | `aap_controller_instance_group_name` | Name of the instance group (execution node pool) |
| **System / Hostname** | `aap_controller_instance_group_id` | Unique identifier of the instance group |
| **Organization** | `aap_controller_organization_name` | Name of the AAP organization |
| **Organization** | `aap_controller_organization_id` | Unique identifier of the AAP organization |
| **Inventory** | `aap_controller_inventory_name` | Name of the inventory used in the job |
| **Inventory** | `aap_controller_inventory_id` | Unique identifier of the inventory |
| **Execution Environment** | `aap_controller_execution_environment_name` | Name of the execution environment |
| **Execution Environment** | `aap_controller_execution_environment_id` | Unique identifier of the execution environment |
| **Job** | `aap_controller_job_id` | Unique identifier of the controller job |
| **Job** | `aap_controller_job_name` | Name of the job template or project sync job |
| **Job** | `aap_controller_job_type` | Type of job: `run`, `cleanup`, etc. |
| **Job** | `aap_controller_launch_type` | How the job was initiated: `manual`, `scheduled`, `webhook`, `workflow` |
| **Job** | `aap_controller_playbook_name` | Name of the playbook being executed |

---

## 2. Vault Security Boundary Relationships

The diagram below shows how AAP JWT claims map to Vault roles, policies, and KV
secret namespaces. The JWT is a pass — Vault **roles** gate which subset of
secrets that JWT can open.

```
AAP Controller (JWT Issuer)
├── User boundary        → launched_by_name / launched_by_id
├── Project boundary     → project_name / project_id
├── System boundary      → instance_group_name / instance_group_id
└── Org boundary         → org_name / org_id
          │
          │  short-lived JWT (all claims bundled as signed assertions)
          ▼
Vault JWT Auth Method
├── Role: aap-<project>        bound_claims: project_name=<value>
│                              token_policies: [kv-project-read, ...]
├── Role: aap-<user>           bound_claims: launched_by_name=<value>
│                              token_policies: [kv-user-read]
├── Role: aap-<instancegroup>  bound_claims: instance_group_name=<value>
│                              token_policies: [kv-infra-read]
└── Role: aap-<org>            bound_claims: org_name=<value>
                               token_policies: [kv-org-read]
          │
          │  Vault token (short TTL, policy-bounded)
          ▼
Vault KV v2
├── kv/data/projects/<project_name>/...
├── kv/data/users/<launched_by_name>/...
├── kv/data/infra/<instance_group_name>/...
└── kv/data/orgs/<org_name>/shared/...
```

---

## 3. How One JWT Accesses Multiple Secrets

### Pattern A — One role, multiple policies (single login)

A Vault JWT role accepts an **array** of `token_policies`. All policies are
attached to the returned token in one `vault write auth/jwt/login` call.

```bash
vault write auth/jwt/role/aap-myapp \
  role_type="jwt" \
  user_claim="aap_controller_project_name" \
  bound_audiences="https://vault.example.com" \
  bound_claims='{"aap_controller_project_name":"my-app"}' \
  claim_mappings='{
    "aap_controller_project_name":      "project_name",
    "aap_controller_organization_name": "org_name",
    "aap_controller_launched_by_name":  "launched_by"
  }' \
  token_policies='["kv-project-read","kv-shared-infra-read","kv-org-read"]' \
  token_ttl="300" \
  token_max_ttl="300"
```

The returned token grants read on all three policy paths in one shot. No second
login is required.

### Pattern B — Templated policy (dynamic, zero-maintenance)

Use `claim_mappings` to promote claim values into token metadata, then reference
them inside policy HCL using the Vault identity template syntax. A **single
policy file** expands at evaluation time to the exact KV path for the job's
project and org — no policy update is needed when a new project is added.

```hcl
# Policy: aap-project-policy.hcl
# AUTH_JWT_ACCESSOR = output of: vault auth list -format=json | jq -r '.["jwt/"].accessor'

path "kv/data/projects/{{identity.entity.aliases.AUTH_JWT_ACCESSOR.metadata.project_name}}/*" {
  capabilities = ["read"]
}

path "kv/data/orgs/{{identity.entity.aliases.AUTH_JWT_ACCESSOR.metadata.org_name}}/shared/*" {
  capabilities = ["read"]
}
```

```json
// Role claim_mappings fragment
{
  "claim_mappings": {
    "aap_controller_project_name":      "project_name",
    "aap_controller_organization_name": "org_name"
  }
}
```

> **Important:** Vault policy path templates do not allow wildcards (`*`, `+`)
> or slashes (`/`) in the *rendered output* of a template variable. Keep secret
> paths flat under the templated prefix and access sub-paths with explicit
> capability grants.

---

## 4. Protecting Production While Reusing IaC

Use `bound_claims_type: glob` on `aap_controller_inventory_name` — the
strongest environmental signal in an AAP JWT. Your Terraform/Ansible IaC module
is identical across environments; only the input variables differ.

```hcl
# Terraform — single module, env-parameterised
resource "vault_jwt_auth_backend_role" "aap_env" {
  backend        = vault_jwt_auth_backend.aap.path
  role_name      = "aap-${var.env}"
  role_type      = "jwt"
  user_claim     = "aap_controller_project_name"
  bound_audiences = [var.vault_audience]

  bound_claims_type = "glob"
  bound_claims = {
    aap_controller_inventory_name = "*${var.env}*"
  }

  token_policies = ["kv-${var.env}-read", "kv-shared-infra-read"]
  # Production gets shortest possible TTL
  token_ttl     = var.env == "prod" ? 300 : 900
  token_max_ttl = var.env == "prod" ? 300 : 1800
}
```

**Production hardening** lives entirely in variable values:
- Shorter `token_ttl` / `token_max_ttl`
- Stricter `bound_claims` (exact inventory name instead of glob, if desired)
- Separate `kv-prod-read` policy that only grants paths under `kv/data/*/prod/`

The module code is reused unchanged across dev, staging, and prod.

---

## 5. Team Pattern: Shared Playbooks, Inventory, and Secrets

`aap_controller_organization_name` is the natural group boundary in AAP. All
members of an org share one Vault role → one policy set → one KV namespace. The
individual audit trail is preserved because `aap_controller_launched_by_name`
appears in Vault's audit log per request via `claim_mappings` metadata.

```
AAP Organization: platform-team
├── Users: alice, bob, carol
├── Job Templates: deploy-app, rotate-creds, run-tests
├── Inventory: dc01-prod (shared)
└── Project: platform-playbooks (shared)
          │
          │  Each JWT carries:
          │    aap_controller_organization_name = platform-team
          │    aap_controller_launched_by_name  = alice | bob | carol
          ▼
Vault Role: aap-platform-team
  bound_claims:  org_name = platform-team
  token_policies: [kv-platform-read, kv-shared-infra-read]
          │
          ▼
Vault KV v2
├── kv/data/orgs/platform-team/*       (team secrets)
└── kv/data/infra/dc01/*               (infra secrets)
```

**Audit query example** — find all secret reads launched by a specific user:

```bash
# Vault audit log (jsonl) — filter by launched_by metadata
grep '"launched_by":"alice"' /var/log/vault/audit.log | jq '.request.path'
```

---

## 6. JWT Claims in an AAP Workflow (Multiple Playbooks)

### How it works

Each **child job node** in a Workflow Job receives its own fresh, independently
signed JWT. The `aap_controller_job_id` and `aap_controller_playbook_name`
differ per node; `aap_controller_project_name`, `aap_controller_organization_name`,
and `aap_controller_launch_type` (`workflow`) remain consistent.

```
AAP Workflow Launch
├── Node 1 — Job Template: provision
│     JWT: job_id=101, playbook=provision.yml, launch_type=workflow, project=my-app
│     → vault write auth/jwt/login role=aap-myapp   (fresh JWT-101)
│     → vault token (TTL 5m) → read kv/data/projects/my-app/db
│     → token expires when job node completes
│
├── Node 2 — Job Template: configure
│     JWT: job_id=102, playbook=configure.yml, launch_type=workflow, project=my-app
│     → vault write auth/jwt/login role=aap-myapp   (fresh JWT-102)
│     → vault token (TTL 5m) → read kv/data/projects/my-app/api-key
│
└── Node 3 — Job Template: verify
      JWT: job_id=103, playbook=verify.yml, launch_type=workflow, project=my-app
      → vault write auth/jwt/login role=aap-myapp   (fresh JWT-103)
      → vault token (TTL 5m) → read kv/data/projects/my-app/health-endpoint
```

**Key conclusions:**
- **One Vault role covers the entire workflow.** Bind on the stable
  `project_name` (and optionally `org_name`) claim — not on `job_id` or
  `playbook_name`, which change per node.
- **No separate role per playbook.** A single role with a templated policy
  grants read to all paths the workflow needs.
- **`launch_type=workflow` as an optional guard.** Add it to `bound_claims` if
  you want to prevent the same role from being used by manual runs:
  ```json
  { "aap_controller_launch_type": ["workflow", "scheduled"] }
  ```
- **No token sharing between nodes.** Each node authenticates independently;
  if one node's token is somehow compromised, it cannot be used by another node.

### Complete role and policy example

```hcl
# ── Policy: aap-project-policy.hcl ──────────────────────────────────────────
# Covers all playbooks in any workflow for any project.
# AUTH_JWT_ACCESSOR replaced at apply time with: vault auth list | grep jwt

path "kv/data/projects/{{identity.entity.aliases.AUTH_JWT_ACCESSOR.metadata.project_name}}/*" {
  capabilities = ["read"]
}

path "kv/metadata/projects/{{identity.entity.aliases.AUTH_JWT_ACCESSOR.metadata.project_name}}/*" {
  capabilities = ["list"]
}

path "kv/data/orgs/{{identity.entity.aliases.AUTH_JWT_ACCESSOR.metadata.org_name}}/shared/*" {
  capabilities = ["read"]
}
```

```bash
# ── Vault JWT Auth Role ──────────────────────────────────────────────────────
vault write auth/jwt/role/aap-myapp \
  role_type="jwt" \
  user_claim="aap_controller_project_name" \
  bound_audiences="https://vault.example.com" \
  bound_claims_type="string" \
  bound_claims='{
    "aap_controller_project_name":      "my-app",
    "aap_controller_organization_name": "platform-team"
  }' \
  claim_mappings='{
    "aap_controller_project_name":        "project_name",
    "aap_controller_organization_name":   "org_name",
    "aap_controller_launched_by_name":    "launched_by"
  }' \
  token_policies="aap-project-policy" \
  token_ttl="300" \
  token_max_ttl="300"
```

> `claim_mappings` serves two purposes: (1) enables templated policy path
> resolution at read time, and (2) writes `project_name`, `org_name`, and
> `launched_by` into Vault's audit log for every secret access.

---

## 7. Programmatic Sync: Auto-provision Vault Roles from AAP Project Names

`aap_controller_project_name` acts as the **shared key** between AAP and Vault.
When a new project is onboarded in AAP, an idempotent playbook task creates the
matching Vault role automatically — no human coordination needed.

### Onboarding flow

```
New AAP Project registered
        │
        │ triggers (webhook / service catalog)
        ▼
Ansible playbook: vault_sync_aap_project.yml
        │
        ├── vault write auth/jwt/role/aap-<project_name>-<env>
        │     bound_claims: project_name=<project_name>
        │     token_policies: [aap-project-policy]   ← reused, never changed
        │
        ├── vault kv put kv/projects/<project_name>/placeholder value=init
        │     (seed the KV path so it exists)
        │
        └── done — project can now authenticate to Vault on first job run
```

### Sync playbook task

```yaml
# tasks/vault_sync_aap_project.yml
- name: Ensure Vault JWT role exists for AAP project
  community.hashi_vault.vault_write:
    path: "auth/jwt/role/aap-{{ project_name }}-{{ env }}"
    data:
      role_type: "jwt"
      user_claim: "aap_controller_project_name"
      bound_audiences: "{{ vault_audience }}"
      bound_claims_type: "string"
      bound_claims:
        aap_controller_project_name:      "{{ project_name }}"
        aap_controller_organization_name: "{{ org_name }}"
      claim_mappings:
        aap_controller_project_name:        "project_name"
        aap_controller_organization_name:   "org_name"
        aap_controller_launched_by_name:    "launched_by"
      token_policies:
        - "aap-project-policy"      # templated policy — same for every project
        - "kv-shared-infra-read"
      token_ttl:     "{{ '300' if env == 'prod' else '900' }}"
      token_max_ttl: "{{ '300' if env == 'prod' else '900' }}"

- name: Seed KV path for new project
  community.hashi_vault.vault_kv2_write:
    path: "projects/{{ project_name }}/placeholder"
    data:
      initialized: "true"
    engine_mount_point: "kv"
```

Because the policy uses path templating, this task is the **only change**
required in Vault when a project is added. The policy itself is written once and
never touched again.

---

## 8. Decision Reference

| Goal | Vault mechanism | Key AAP claim |
|---|---|---|
| Scope secrets to a project | `bound_claims: project_name` + templated policy path | `aap_controller_project_name` |
| Scope secrets to a user / service account | `bound_claims: launched_by_name` | `aap_controller_launched_by_name` |
| Scope secrets to an infrastructure zone | `bound_claims: instance_group_name` | `aap_controller_instance_group_name` |
| Protect prod vs non-prod with same IaC | `bound_claims_type=glob` on inventory name + env variable in Terraform/Ansible | `aap_controller_inventory_name` |
| One JWT → many secrets | Single role, `token_policies` array + templated policy HCL | `claim_mappings` → metadata |
| No-upkeep project sync | Templated policy (written once) + per-project role created by pipeline | `aap_controller_project_name` as shared key |
| Team shares playbooks + secrets | Org-scoped role; individual audit via `launched_by` in metadata | `aap_controller_organization_name` |
| Workflow with N playbooks | One stable role bound to `project_name`; each node gets its own JWT | `aap_controller_launch_type=workflow` (optional guard) |

---

## References

- [AAP 2.7 — Claims for workload identity](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/whats_new-claims_for_workload_identity#GUID-274163f4-8d73-4459-9919-56d73908cdbb__section_1)
- [HashiCorp Vault — JWT/OIDC auth method](https://developer.hashicorp.com/vault/docs/auth/jwt)
- [HashiCorp Vault — Templated policies](https://developer.hashicorp.com/vault/docs/concepts/policies#templated-policies)
- [Terraform — vault_jwt_auth_backend_role](https://registry.terraform.io/providers/hashicorp/vault/latest/docs/resources/jwt_auth_backend_role)
