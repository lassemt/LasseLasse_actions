# LasseLasse Actions

Shared reusable GitHub Actions workflows for hub-and-spoke infrastructure. Spoke app repos reference these workflows via `workflow_call`.

## Workflows

| Workflow | Description | Docs |
|---|---|---|
| `terraform-app` | Terraform plan-on-PR and apply-on-merge for spoke app repos | [docs/terraform-app.md](docs/terraform-app.md) |

## Versioning

This repo uses semver tags. Reference a major version to get non-breaking updates automatically:

```yaml
uses: lassemt/LasseLasse_actions/.github/workflows/<workflow>.yml@v1
```

For exact version pinning:

```yaml
uses: lassemt/LasseLasse_actions/.github/workflows/<workflow>.yml@v1.0.0
```
