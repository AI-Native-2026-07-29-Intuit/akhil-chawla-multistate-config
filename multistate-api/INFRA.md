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

**Floci / no-AWS note:** `DetectStackDrift` is not supported on Floci (`UnknownAction`), so no drift output was captured. On real AWS, after an out-of-band tag on the artefacts bucket, drift status would read `DRIFTED` with `MultistateArtifactsBucket` flagged `MODIFIED`; reverting the change would return `IN_SYNC`.

## Floci / no-AWS limitations by criterion

No real AWS account was used. Templates were validated via CI (`cfn-lint`, `cfn-nag`, `validate-template` against LocalStack). Stacks were deployed on Floci where live verification was possible.

| Criterion | Verified on Floci / in repo | Requires real AWS |
|-----------|----------------------------|-------------------|
| **T1** Bootstrap | `multistate-bootstrap-dev` `CREATE_COMPLETE`; ChangeSet JSON; IAM trust `StringEquals` aud + `StringLike` sub; 12 CFN actions in template | S3 API checks (`get-public-access-block`, encryption, lifecycle, bucket policy) — Floci does not persist PAB/policy/lifecycle to the S3 API even when CFN reports `CREATE_COMPLETE` |
| **T2** Network | `multistate-network-dev` `CREATE_COMPLETE` / `UPDATE_COMPLETE`; 6 subnets; `IsProdLike` gates NAT; exports present; app SG ingress from VPC CIDR only | None |
| **T3** App + S3 | `multistate-app-dev` + `multistate-artifacts-dev` `CREATE_COMPLETE`; `!ImportValue` wiring; SM dynamic ref (no `NoEcho`); Retain pair + lifecycle in template | S3 PAB/policy API on artefacts bucket; **export-in-use delete guard** — on real AWS, deleting `multistate-network-dev` while `multistate-app-dev` imports its exports would fail with a `ValidationError` naming the export in use; Floci allows the delete |
| **T4** CI + drift | `cfn-validate.yml` green; required status check on `main`; UPDATE ChangeSet `Replacement: False`; cfn-author audit in this file | **Drift detection** — Floci does not implement `DetectStackDrift`; on real AWS, an out-of-band bucket tag would yield `DRIFTED`, then `IN_SYNC` after revert |
| **Hygiene** | Branch `w6d3-implementation`; annotated YAML; this document | None |

**Template workaround:** `Fn::Split` + `Fn::ImportValue` on `SubnetIds` fails on Floci; the app template uses `Fn::Select` [0,1,2] over the same split (valid on real AWS as well).

**CI note:** GitHub Actions runs `validate-template` against a LocalStack sidecar because no AWS credentials are available in CI.

## cfn-author Skill audit notes

Compared `/cfn-author multistate --region us-east-1` output against the four authored templates:

**Accepted:** The Skill's suggestion to gate NAT gateways with `IsProdLike` / `IsDev` Conditions — we kept this pattern because a single template must serve dev (one NAT, lower cost) and staging/prod (NAT-per-AZ HA) without maintaining separate files.

**Rejected:** The Skill sometimes uses `StringLike` on the OIDC trust policy **`aud`** claim (`token.actions.githubusercontent.com:aud`). The cohort checklist requires **`StringEquals`** on `aud` (`sts.amazonaws.com`) and **`StringLike` only on `sub`** to pin the gitops repo. Our bootstrap template (`cfn/multistate-bootstrap-dev.yaml`) follows the required split.

**Rejected:** The Skill often scaffolds a `NoEcho: true` password Parameter for RDS. We use an out-of-band Secrets Manager secret and `{{resolve:secretsmanager:multistate/dev/db-master:SecretString:password}}` so the password never appears in CFN parameters or stack templates.

**Rejected / fixed:** The Skill sometimes sets `DeletionPolicy: Retain` on S3/RDS without **`UpdateReplacePolicy: Retain`**. Both buckets and `DbInstance` carry the pair so update-induced replacement cannot destroy data.

## CI: cfn-validate

`.github/workflows/cfn-validate.yml` runs on every PR touching `cfn/`:

- `cfn-lint cfn/*.yaml -a cfn_lint_serverless.rules` (serverless profile)
- `cfn_nag_scan --input-path cfn/multistate-*.yaml` (one scan per template, `--fail-on-warnings`)
- `aws cloudformation validate-template` for all four templates

Mark **cfn-validate** as a required status check on `main` in GitHub branch protection.
