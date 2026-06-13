# terraform-app

Reusable Terraform plan-on-PR and apply-on-merge workflow for spoke app repos. Authenticates via OIDC to a hub AWS account, then runs Terraform in the specified working directory.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `client-name` | Yes | — | Client name matching the infra client directory |
| `aws-management-account-id` | Yes | — | The 12-digit AWS account ID where hub IAM roles are defined |
| `working-directory` | No | `infra/prod` | Path to the Terraform root module |
| `terraform-version` | No | `1.15.6` | Terraform version to use |
| `aws-region` | No | `eu-north-1` | AWS region for authentication |

## Setup

1. In your spoke repo, add a GitHub Actions **variable** (Settings > Secrets and variables > Actions > Variables):
   - Name: `AWS_MANAGEMENT_ACCOUNT_ID`
   - Value: your 12-digit AWS hub account ID (e.g. `123456789012`)

2. Create `.github/workflows/terraform.yml` in the spoke repo:

```yaml
name: Terraform

on:
  pull_request:
    branches: [main]
    paths: ["infra/**"]
  push:
    branches: [main]
    paths: ["infra/**"]

permissions:
  contents: read
  id-token: write
  pull-requests: write

jobs:
  terraform:
    uses: lassemt/LasseLasse_actions/.github/workflows/terraform-app.yml@v1
    with:
      client-name: client-1
      aws-management-account-id: ${{ vars.AWS_MANAGEMENT_ACCOUNT_ID }}
    permissions:
      contents: read
      id-token: write
      pull-requests: write
```

## Spoke repo setup commands

Set the AWS management account ID as a repo variable:

```bash
gh variable set AWS_MANAGEMENT_ACCOUNT_ID --body "123456789012" --repo <owner>/<spoke-repo>
```

Create the `production` environment (used by the apply job):

```bash
gh api --method PUT repos/<owner>/<spoke-repo>/environments/production
```

Require the `plan` job to pass before merging (branch protection):

```bash
gh api --method PUT repos/<owner>/<spoke-repo>/branches/main/protection \
  --input - <<'EOF'
{
  "required_status_checks": {
    "strict": true,
    "contexts": ["terraform / plan"]
  },
  "enforce_admins": true,
  "required_pull_request_reviews": null,
  "restrictions": null
}
EOF
```

> **Note:** Without branch protection, a PR with a failed plan can still be merged — which triggers `terraform apply` on push. The workflow does not gate apply on a prior successful plan; that safety comes from branch protection rules.

## Requirements

- The spoke repo must be registered in the hub infra (so the OIDC trust policy allows it to assume the deploy role)
- The spoke repo must have a `production` GitHub environment configured (used by the apply job)
- The spoke repo should have branch protection requiring the `plan` job to pass before merge

## Known limitations

### Plan/apply drift

The plan shown on a pull request is generated at PR time against the current Terraform state. The apply runs after the PR is merged, against the state at that point. Between the two, state can change due to another PR merging first, manual console changes, or external automation.

As a result, the applied plan may differ from the reviewed plan. This is inherent to the pattern where plan and apply run in separate workflow triggers (`pull_request` vs `push`). Terraform plan files cannot reliably be passed between workflow runs because they contain sensitive state tied to the exact snapshot at generation time.

**Mitigations:**
- The apply job runs `terraform plan -out=tfplan` followed by `terraform apply tfplan`, so it always applies exactly what it just planned.
- The `concurrency` group prevents parallel runs for the same client and working directory.
- The `production` environment supports required reviewers as a manual approval gate before apply.
