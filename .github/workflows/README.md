# workflows

GitHub Actions pipelines for linting, validating (`what-if`), and deploying the Bicep workloads in
this repo. One workflow per workload is expected (e.g. `terraform-bootstrap.yml`), gated by
environment-specific approvals for `tst`/`prd`.
