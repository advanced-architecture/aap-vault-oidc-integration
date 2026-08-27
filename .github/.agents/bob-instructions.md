# Bob instructions for this repository

This repository is intended to demonstrate **Ansible Automation Platform (AAP) and HashiCorp Vault integration using OIDC workload identity as the trust mechanism**. The repository is still early, so keep these instructions aligned to that goal and update them as real project files are added.

## Build, test, and lint

- No build, test, or lint commands are defined yet.
- No single-test workflow exists yet.

## High-level architecture

- The expected core flow is: **AAP 2.7** issues a short-lived OIDC/JWT workload identity for a job, **Vault** trusts that issuer through its **JWT auth method**, Vault returns a short-lived token or serves a policy-scoped secret, and the resolved value is consumed during **Ansible** job execution.
- Treat **AAP as the OIDC identity provider for this pattern**, not as a passive client of some other generic issuer. Future content should preserve the trust chain of **AAP job identity -> Vault JWT auth -> KV v2 secret retrieval -> Ansible runtime consumption**.
- Keep the repository organized around that trust chain. When implementation files are added, distinguish clearly between:
  - Vault-side configuration such as JWT auth mounts, roles, policies, and KV v2 setup
  - AAP-side configuration such as credential fields, job-template configuration, and workload-identity prerequisites
  - Runtime Ansible content such as playbooks or roles that demonstrate consuming the resolved secret
  - Any bootstrap or demo-environment assets used to stand up the example
- Preserve the separation between **bootstrap setup**, **platform connection-point configuration**, and **runtime demonstration**. Future sessions should avoid mixing one-time Vault or AAP setup with the playbooks or tasks that demonstrate the live OIDC-to-Vault flow.

## Key conventions

- Optimize the example around the **AAP-issued workload identity flow**, not around static secret distribution. Future changes should prefer short-lived authentication and runtime secret retrieval over checked-in credentials, manually copied tokens, or long-lived shared secrets.
- Treat AAP workload identity, Vault authorization/policy, and Ansible secret consumption as separate layers. Avoid burying Vault auth or policy configuration inside unrelated Ansible content when a clearer boundary is possible.
- Keep environment-specific values configurable. AAP URLs, OIDC discovery endpoints, Vault addresses, auth mount paths, role names, KV mounts, and secret paths should be easy to override through variables or documented configuration points rather than hardcoded throughout the repo.
- Prefer examples that show the full path from AAP-issued identity to Vault-authenticated secret consumption, not just a static Vault lookup disconnected from the OIDC trust flow.
- Default the example pattern to **Vault JWT auth with KV secrets engine v2** unless committed repository content intentionally expands the scope.
- Incorporate future guidance from committed files such as `README.md`, CI workflows, inventories, playbooks, roles, test configuration, or additional assistant instruction files instead of replacing these instructions with generic advice.
