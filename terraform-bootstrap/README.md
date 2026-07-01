# terraform-bootstrap

Bicep templates that provision the one-time Azure resources Terraform needs before it can manage
anything else: a storage account for remote state and a Key Vault for CI/CD secrets.

- `main.bicep`: resource-group-scoped deployment composing the AVM storage-account and key-vault modules
- `main.<env>.bicepparam`: per-environment parameter files (`dev`, `tst`, `prd`)

See [ai/context/bicep_iac_lz_context.md](../ai/context/bicep_iac_lz_context.md) for the design
decisions behind this workload (auth model, retention, role assignments) before making changes.
