# prd

Environment-specific values (subscription/tenant IDs, naming tokens, IP allow-lists) for the `prd`
environment. Workload parameter files (`main.prd.bicepparam`) live next to their template under the
workload's own folder (e.g. `terraform-bootstrap/`) and reference shared values from here where needed.

Production hardening applies here first — e.g. `networkAcls.defaultAction: 'Deny'` with explicit
`ipRules` (see Open Items in the bootstrap context doc).
