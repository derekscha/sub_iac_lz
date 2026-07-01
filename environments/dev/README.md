# dev

Environment-specific values (subscription/tenant IDs, naming tokens, IP allow-lists) for the `dev`
environment. Workload parameter files (`main.dev.bicepparam`) live next to their template under the
workload's own folder (e.g. `terraform-bootstrap/`) and reference shared values from here where needed.
