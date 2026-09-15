# LLM Session Transcript

**Date:** 2026-09-14  
**Engine / Tool:** Bob (IBM Bob, agent mode)  
**Workspace:** `aap-vault-oidc-integration`

---

## Prompt Summary

The user asked two related research questions across two messages:

**Message 1:**
> Using the AAP 2.7 OIDC claims reference and product knowledge about the
> HashiCorp Vault JWT auth method, diagram and document relationships between
> AAP and Vault to define Vault security boundaries around user
> (`aap_controller_launched_by_name`), application (Project), and
> system/hostname. Answer:
> - How can one JWT access multiple secrets?
> - How can access be programmatically synced, based on app name, to reduce
>   ongoing upkeep for synchronisation of claims?

**Message 2 (addendum):**
> Also answer:
> - How can production environments be protected while using the same IaC in
>   all environments?
> - What is a good pattern for a group or multiple individuals to share
>   playbooks, inventory, and a set of secrets?
> - How are JWT claims handled in an AAP workflow of multiple playbooks? Are
>   multiple JWTs needed? What does the Vault JWT auth role and policy look like?

---

## Research Performed

1. **Extracted AAP 2.7 claims table** from
   `https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/whats_new-claims_for_workload_identity`
   — retrieved the full list of claims grouped by boundary (user, project,
   system, org, inventory, job).

2. **Searched Vault JWT auth documentation** at
   `https://developer.hashicorp.com/vault/docs/auth/jwt` — extracted
   `bound_claims`, `bound_claims_type`, `claim_mappings`, and identity-template
   policy path syntax.

3. **Searched Vault policy templating documentation** at
   `https://developer.hashicorp.com/vault/docs/concepts/policies` — confirmed
   `{{identity.entity.aliases.ACCESSOR.metadata.key}}` syntax and the
   restriction that rendered template values may not contain wildcards or
   unescaped slashes.

4. **Cross-referenced GitLab × Vault JWT pattern** (public docs) to confirm
   `bound_claims_type=glob`, per-environment role pattern, and
   `claim_mappings` + templated policy approach in production use.

---

## Key Findings

- **Each AAP Workflow node issues its own fresh JWT** — `job_id` and
  `playbook_name` differ per node; `project_name` and `org_name` are stable
  across the whole workflow. One Vault role bound on `project_name` covers
  the entire workflow.

- **One JWT → multiple secrets** via two mechanisms:
  1. `token_policies` array on the Vault role (multi-policy, single login)
  2. Identity-templated policy path (`claim_mappings` → metadata →
     `{{identity.entity.aliases.ACCESSOR.metadata.project_name}}`)

- **Env isolation with shared IaC**: `bound_claims_type=glob` on
  `aap_controller_inventory_name` (`*prod*`, `*staging*`, etc.) combined with
  a Terraform/Ansible module parameterised by `var.env`.

- **Team/group pattern**: org-scoped role (`aap_controller_organization_name`);
  per-user audit trail preserved via `claim_mappings` writing `launched_by`
  into token metadata.

- **Programmatic sync**: `aap_controller_project_name` as the shared key
  between AAP and Vault. One idempotent `community.hashi_vault.vault_write`
  task creates the matching Vault role on project onboarding. The templated
  policy is written once and never updated.

---

## Changes Made

| File | Action |
|---|---|
| `docs/vault-security-boundaries.md` | **Created** — 295-line design reference document covering all eight topic areas with ASCII diagrams, HCL examples, and a decision table |

---

## Upcoming Commit

```
docs(vault): add security boundaries and JWT auth pattern reference

Document Vault security boundary design for AAP OIDC workload identity:
- Full AAP 2.7 claim table grouped by boundary (user/project/system/org)
- One-JWT-multiple-secrets patterns (multi-policy + templated policy)
- Prod/non-prod env isolation with shared IaC (glob bound_claims on inventory)
- Team/group shared playbook and secrets pattern
- Workflow multi-playbook JWT behaviour (one role covers all nodes)
- Programmatic project sync pattern (project_name as shared key)

Transcript: .llm/2026-09-14-vault-security-boundaries.md
```
