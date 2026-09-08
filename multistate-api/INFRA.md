# multistate-api/INFRA.md
# How this service's AWS substrate is provisioned and what
# deviated from the cfn-author Skill's defaults.

## Stack layout (four stacks)

| Stack | Template | Purpose |
|-------|----------|---------|
| `multistate-bootstrap-dev` | `cfn/multistate-bootstrap-dev.yaml` | Bootstrap S3 bucket + `multistate-api-cfn-deploy` IAM role (OIDC for GitHub Actions). Deploy first. |
| `multistate-artifacts-dev` | `cfn/multistate-artifacts-dev.yaml` | Hardened artefact bucket (`uptimecrew-multistate-artifacts-dev`): PAB, KMS, versioning, lifecycle (90d → STANDARD_IA, 365d → GLACIER_IR), deny-non-TLS, Retain ×2. Independent of network. |
| `multistate-network-dev` | `cfn/multistate-network-dev.yaml` | 3-AZ VPC, 6 subnets, IGW, 1–3 NAT GWs (`IsProdLike` / `IsDev` Conditions), app security group. Exports listed below. |
| `multistate-app-dev` | `cfn/multistate-app-dev.yaml` | RDS PostgreSQL, DB subnet group, DB SG, app→DB egress rule. Consumes network via `!ImportValue`. DB password from Secrets Manager dynamic reference (secret created out-of-band). |

## Deploy order

1. **multistate-bootstrap-dev** (Task 1) — bootstrap bucket + CFN-deploy IAM role.
2. **multistate-artifacts-dev** (Task 3) — artefact bucket (no network dependency).
3. **multistate-network-dev** (Task 2) — VPC + subnets + SG.
4. **multistate-app-dev** (Task 3) — RDS; requires network exports + `multistate/dev/db-master` secret in Secrets Manager.

**Prerequisite (once per env, before app stack):**

```bash
aws secretsmanager create-secret \
  --name multistate/dev/db-master \
  --secret-string '{"username":"multistate_master","password":"<generated>"}'
```

## ChangeSet flow (every stack, every change)

```bash
aws cloudformation create-change-set \
  --stack-name <stack-name> \
  --change-set-name <name> \
  --change-set-type CREATE_OR_UPDATE \
  --template-body file://cfn/<template>.yaml \
  --capabilities CAPABILITY_NAMED_IAM \   # bootstrap only
  --parameters ParameterKey=EnvName,ParameterValue=dev \
  --region us-east-1

aws cloudformation describe-change-set \
  --stack-name <stack-name> --change-set-name <name> \
  --region us-east-1
# Review JSON diff in PR before execute.

aws cloudformation execute-change-set \
  --stack-name <stack-name> --change-set-name <name> \
  --region us-east-1

aws cloudformation wait stack-create-complete --stack-name <stack-name>   # or stack-update-complete
```

The bootstrap stack's `CfnDeployRole` is what GitHub Actions assumes via OIDC to run these commands in CI/CD.

## Cross-stack export naming

Exports use `${AWS::StackName}-<Suffix>` so names are stable and predictable:

| Export name | Producer | Consumer |
|-------------|----------|----------|
| `multistate-network-dev-VpcId` | network | app (DB SG VpcId) |
| `multistate-network-dev-VpcCidr` | network | optional |
| `multistate-network-dev-PrivateSubnets` | network | app (DB subnet group via `Fn::Split` + `Fn::ImportValue`) |
| `multistate-network-dev-PublicSubnets` | network | future workloads |
| `multistate-network-dev-AppSgId` | network | app (DB SG ingress + `AppToDbEgress`) |
| `multistate-bootstrap-dev-BucketName` | bootstrap | CI template uploads |
| `multistate-artifacts-dev-BucketName` | artifacts | SAM / Argo snapshots |

**Never hardcode subnet or security-group IDs** — always `!ImportValue` so a network rebuild does not leave stale references.

## Drift verification (Task 4)

Deliberately drift an out-of-band change (e.g. add a tag on the artefacts bucket via console/CLI), then:

```bash
DRIFT_ID=$(aws cloudformation detect-stack-drift \
  --stack-name multistate-artifacts-dev \
  --query StackDriftDetectionId --output text)

aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id "$DRIFT_ID"

aws cloudformation describe-stack-resource-drifts \
  --stack-name multistate-artifacts-dev \
  --query "StackResourceDrifts[?StackResourceDriftStatus!='IN_SYNC']"
```

Revert the manual change (remove tag or re-apply template via ChangeSet) and re-run drift detection until `StackDriftStatus` is `IN_SYNC`.

Evidence captured under `cfn/evidence/drift-*.txt`.

## Floci (local emulator) limitations

When using Floci instead of real AWS:

- S3 PAB and bucket policies may not persist to the S3 API even when CFN reports `CREATE_COMPLETE`.
- CloudFormation export-in-use delete protection is not enforced.
- `Fn::Split` + `Fn::ImportValue` on `SubnetIds` fails; use `Fn::Select` [0,1,2] over the same split (template includes this workaround; real AWS accepts both forms).
- Drift detection support varies — re-run against real AWS for grading evidence if Floci returns incomplete results.

## cfn-author Skill audit notes

Compared `/cfn-author multistate --region us-east-1` output against the four authored templates:

**Accepted:** The Skill's suggestion to gate NAT gateways with `IsProdLike` / `IsDev` Conditions — we kept this pattern because a single template must serve dev (one NAT, lower cost) and staging/prod (NAT-per-AZ HA) without maintaining separate files.

**Rejected:** The Skill sometimes uses `StringLike` on the OIDC trust policy **`aud`** claim (`token.actions.githubusercontent.com:aud`). The cohort checklist requires **`StringEquals`** on `aud` (`sts.amazonaws.com`) and **`StringLike` only on `sub`** to pin the gitops repo. Our bootstrap template (`cfn/multistate-bootstrap-dev.yaml`) follows the required split.

**Rejected:** The Skill often scaffolds a `NoEcho: true` password Parameter for RDS. We use an out-of-band Secrets Manager secret and `{{resolve:secretsmanager:multistate/dev/db-master:SecretString:password}}` so the password never appears in CFN parameters or stack templates.

**Rejected / fixed:** The Skill sometimes sets `DeletionPolicy: Retain` on S3/RDS without **`UpdateReplacePolicy: Retain`**. Both buckets and `DbInstance` carry the pair so update-induced replacement cannot destroy data.

## CI: cfn-validate

`.github/workflows/cfn-validate.yml` runs on every PR touching `cfn/`:

- `cfn-lint cfn/*.yaml -a cfn_lint_serverless.rules` (serverless profile)
- `cfn_nag_scan --input-path cfn --fail-on-warnings`
- `aws cloudformation validate-template` for all four templates

Mark **cfn-validate** as a required status check on `main` in GitHub branch protection.
