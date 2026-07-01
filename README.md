# Azure Subscription IaC Deployment Prep

Bicep-based infrastructure-as-code for standing up a new Azure subscription as a landing zone,
starting with the resources a Terraform CI/CD pipeline needs before it can manage anything else.

## Status

Early scaffolding. The `terraform-bootstrap` workload is designed (see
[ai/context/bicep_iac_lz_context.md](ai/context/bicep_iac_lz_context.md)) but not yet implemented.

## Repository structure

| Path | Purpose |
| --- | --- |
| `terraform-bootstrap/` | Bicep templates provisioning Terraform's remote-state storage account and secrets Key Vault |
| `modules/` | Custom Bicep modules, used only when no suitable AVM module exists |
| `environments/` | Shared per-environment values (subscription/tenant IDs, IP allow-lists) for `dev`, `tst`, `prd` |
| `scripts/` | Deployment and validation script wrappers (`az deployment group create`, linting) |
| `tests/` | Bicep validation and deployment tests |
| `docs/` | Human-facing architecture notes and diagrams |
| `ai/context/` | Design records for AI assistants — read before implementing or changing a workload |
| `.github/workflows/` | CI/CD pipelines (lint, what-if, deploy) |

Each workload lives in its own top-level folder containing a `main.bicep` and one
`main.<env>.bicepparam` file per environment.

## Design principles

- Compose [Azure Verified Modules](https://azure.github.io/Azure-Verified-Modules/indexes/bicep/bicep-resource-modules/)
  (`br/public:avm/...`) rather than hand-rolled resource definitions.
- RBAC-only access: no shared storage keys, no Key Vault access policies.
- Least-privilege roles for CI/CD identities; elevated roles reserved for human admins.
- `CanNotDelete` locks and 30-day soft-delete/versioning retention on stateful resources.
- Module versions pinned explicitly and verified against the AVM index before upgrading.

## Prerequisites

- [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) with the Bicep tooling
  (`az bicep install`)
- Contributor access (or equivalent) on the target subscription
- The Azure AD object IDs of the CI/CD principal and the admin identity/group for the workload
  being deployed

## Deploying a workload

```bash
az deployment group create \
  --resource-group <bootstrap-rg-name> \
  --template-file terraform-bootstrap/main.bicep \
  --parameters terraform-bootstrap/main.dev.bicepparam
```

## Contributing

Run `az bicep lint` before committing. See [CLAUDE.md](CLAUDE.md) for the conventions this repo
follows when adding or changing workloads.
