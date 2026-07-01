# CLAUDE.md

Guidance for Claude Code when working in this repository.

## Project

Subscription-level Azure landing zone IaC, implemented in Bicep. The first workload
(`terraform-bootstrap/`) provisions the storage account and Key Vault a Terraform CI/CD pipeline
needs before it can manage anything else. Additional subscription-scoped workloads will be added
as sibling top-level folders following the same pattern.

## Conventions

- Compose [Azure Verified Modules](https://azure.github.io/Azure-Verified-Modules/indexes/bicep/bicep-resource-modules/)
  (`br/public:avm/...`) rather than writing raw resource blocks; only add a module under
  `modules/` when no suitable AVM module exists.
- Pin AVM module versions explicitly and verify the latest stable version at the AVM index before
  bumping.
- RBAC over shared keys/access policies: `allowSharedKeyAccess = false` on storage,
  `enableRbacAuthorization = true` on Key Vault. Least-privilege roles for CI/CD identities,
  elevated roles for human admins.
- One top-level folder per workload (e.g. `terraform-bootstrap/`), containing `main.bicep` plus
  `main.<env>.bicepparam` files for `dev` / `tst` / `prd`.
- Before implementing or changing a workload, read its design record under `ai/context/` if one
  exists — it captures decisions (and their rationale) that aren't otherwise visible in the code.

## Structure

- `ai/context/` — per-workload design context for AI assistants (read before implementing)
- `terraform-bootstrap/`, (future workload folders) — Bicep templates + parameter files
- `modules/` — custom Bicep modules (AVM gaps only)
- `environments/` — shared per-environment values (subscription/tenant IDs, IP allow-lists)
- `scripts/` — deployment/validation script wrappers
- `tests/` — Bicep validation and deployment tests
- `docs/` — human-facing architecture notes
- `.github/workflows/` — CI/CD pipelines
