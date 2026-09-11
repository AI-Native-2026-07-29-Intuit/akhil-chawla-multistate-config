# CloudFormation evidence — captured command output

Files here are **stdout/stderr from real commands** (Floci, local shell, or CI metadata), not mocked. Re-run with:

```bash
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test AWS_SECRET_ACCESS_KEY=test
```

## W6 D4 (cost stack)

| File | Source |
|------|--------|
| `cost-describe-stacks.json` | `aws cloudformation describe-stacks --stack-name multistate-cost-dev` |
| `cost-describe-alarms.json` | `aws cloudwatch describe-alarms --alarm-names multistate/estimated-charges-dev` |
| `cost-describe-budget.txt` | Expected real-AWS output from template + Floci gap note |
| `cost-describe-budget-raw.txt` | Actual Floci stderr from `aws budgets describe-budget` |
| `cost-explorer-report.txt` | Procedure only — no AWS console access |
| `cost-notbreaching-grep.txt` | `grep -RIn notBreaching cfn/` |
| `cost-ci-cfn-validate.txt` | GitHub Actions cfn-validate **pass** (run 34525916322) |
| `cost-tag-resources.txt` | Tag API on Floci (empty list) |

Limitations table: `multistate-api/INFRA.md` § **Floci / no-AWS limitations by criterion**.
