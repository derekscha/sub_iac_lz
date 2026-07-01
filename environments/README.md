# environments

Shared per-environment values (subscription/tenant IDs, naming tokens, IP allow-lists) for `dev`,
`tst`, and `prd`. Workload parameter files (`main.<env>.bicepparam`) live next to their template
under the workload's own folder (e.g. `terraform-bootstrap/`) and reference these shared values
where needed.
