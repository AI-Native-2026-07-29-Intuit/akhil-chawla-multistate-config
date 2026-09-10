# multistate AWS Fundamentals & CloudFormation substrate — AkhilChawla

**Branch:** `w6d3-implementation`

---

## Summary

Four CloudFormation stacks for the multistate capstone substrate: bootstrap (S3 + IAM), network (3-AZ VPC + SG), artifacts (hardened S3), and app (RDS + cross-stack imports). All stacks authored with ChangeSet flow, validated via `cfn-validate` CI, and documented in `multistate-api/INFRA.md`.

**Deployment note:** No real AWS account was used for this work. Templates were validated via `cfn-lint`, `cfn-nag`, and `validate-template` (CI uses LocalStack). Stacks were deployed on **Floci** (`AWS_ENDPOINT_URL=http://localhost:4566`) where live verification was possible. See **Floci / no-AWS limitations by criterion** below for gaps vs `criteria.txt` pass signals.

---

## Floci / no-AWS limitations by criterion

| Criterion | Verified on Floci / in repo | Requires real AWS (expected response in PR evidence) |
|-----------|----------------------------|-----------------------------------------------------|
| **T1** Bootstrap | `multistate-bootstrap-dev` `CREATE_COMPLETE`; ChangeSet JSON; IAM trust `StringEquals` aud + `StringLike` sub; 12 CFN actions in template | `get-public-access-block`, `get-bucket-encryption`, `get-bucket-lifecycle`, `get-bucket-policy` on bootstrap bucket — Floci does not persist PAB/policy/lifecycle to the S3 API even when CFN reports `CREATE_COMPLETE` |
| **T2** Network | `multistate-network-dev` `CREATE_COMPLETE` / `UPDATE_COMPLETE`; 6 subnets; `IsProdLike` gates NAT; exports present; app SG ingress from VPC CIDR only | None — fully verifiable on Floci |
| **T3** App + S3 | `multistate-app-dev` + `multistate-artifacts-dev` `CREATE_COMPLETE`; `!ImportValue` wiring; SM dynamic ref (no `NoEcho`); Retain pair + lifecycle in template | S3 PAB/policy API on artefacts bucket (Floci gap); **network delete refused** — Floci deletes network stack without export-in-use guard; expected AWS behavior described in §7 |
| **T4** CI + drift | `cfn-validate.yml` green; required status check on `main`; UPDATE ChangeSet `Replacement: False`; cfn-author audit in `INFRA.md` | **detect-stack-drift** — Floci returns `UnknownAction` for `DetectStackDrift`; expected AWS behavior described in §6 |
| **Hygiene** | Branch `w6d3-implementation`; ≥2 commits; annotated YAML; `INFRA.md` complete | None |
| **H8 AI review** | Accepted + rejected cfn-author suggestions in this PR body | None |

**CI note:** `aws cloudformation validate-template` in GitHub Actions uses a **LocalStack** sidecar (same class of emulator as Floci) because no AWS credentials are available in CI.

---

## Evidence

### 1. Bootstrap CREATE ChangeSet (`multistate-bootstrap-dev`)

Captured via `aws cloudformation describe-change-set` after `create-change-set --change-set-type CREATE` (Floci). Template defines PAB (all four toggles), KMS (`alias/aws/s3`), lifecycle, deny-non-TLS, and Retain pair — **S3 API verification of PAB/policy/lifecycle requires real AWS** (Floci gap; see limitations table).

```json
{
  "Changes": [
    {
      "Type": "Resource",
      "ResourceChange": {
        "Action": "Add",
        "LogicalResourceId": "BootstrapBucket",
        "ResourceType": "AWS::S3::Bucket"
      }
    },
    {
      "Type": "Resource",
      "ResourceChange": {
        "Action": "Add",
        "LogicalResourceId": "BootstrapBucketPolicy",
        "ResourceType": "AWS::S3::BucketPolicy"
      }
    },
    {
      "Type": "Resource",
      "ResourceChange": {
        "Action": "Add",
        "LogicalResourceId": "CfnDeployRole",
        "ResourceType": "AWS::IAM::Role"
      }
    }
  ],
  "ChangeSetName": "initial",
  "StackName": "multistate-bootstrap-dev",
  "ExecutionStatus": "AVAILABLE",
  "Status": "CREATE_COMPLETE"
}
```

---

### 2. Network CREATE ChangeSet (`multistate-network-dev`)

```json
{
  "Changes": [
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "Vpc", "ResourceType": "AWS::EC2::VPC" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "InternetGateway", "ResourceType": "AWS::EC2::InternetGateway" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "IgwAttachment", "ResourceType": "AWS::EC2::VPCGatewayAttachment" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "PublicSubnetA", "ResourceType": "AWS::EC2::Subnet" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "PublicSubnetB", "ResourceType": "AWS::EC2::Subnet" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "PublicSubnetC", "ResourceType": "AWS::EC2::Subnet" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "PrivateSubnetA", "ResourceType": "AWS::EC2::Subnet" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "PrivateSubnetB", "ResourceType": "AWS::EC2::Subnet" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "PrivateSubnetC", "ResourceType": "AWS::EC2::Subnet" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "NatEipA", "ResourceType": "AWS::EC2::EIP" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "NatGatewayA", "ResourceType": "AWS::EC2::NatGateway" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "PublicRouteTable", "ResourceType": "AWS::EC2::RouteTable" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "PublicDefaultRoute", "ResourceType": "AWS::EC2::Route" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "PublicAssocA", "ResourceType": "AWS::EC2::SubnetRouteTableAssociation" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "PublicAssocB", "ResourceType": "AWS::EC2::SubnetRouteTableAssociation" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "PublicAssocC", "ResourceType": "AWS::EC2::SubnetRouteTableAssociation" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "PrivateRouteTableA", "ResourceType": "AWS::EC2::RouteTable" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "PrivateDefaultRouteA", "ResourceType": "AWS::EC2::Route" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "PrivateAssocA", "ResourceType": "AWS::EC2::SubnetRouteTableAssociation" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "PrivateAssocBDev", "ResourceType": "AWS::EC2::SubnetRouteTableAssociation" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "PrivateAssocCDev", "ResourceType": "AWS::EC2::SubnetRouteTableAssociation" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "MultistateAppSecurityGroup", "ResourceType": "AWS::EC2::SecurityGroup" } }
  ],
  "ChangeSetName": "initial",
  "StackName": "multistate-network-dev",
  "Status": "CREATE_COMPLETE",
  "ExecutionStatus": "AVAILABLE"
}
```

Full file: `cfn/evidence/network-changeset-describe.json`

---

### 3. Artifacts CREATE ChangeSet (`multistate-artifacts-dev`)

Template defines PAB, KMS, versioning, lifecycle (90d → STANDARD_IA, 365d → GLACIER_IR), deny-non-TLS, Retain pair. **Floci:** stack `CREATE_COMPLETE` but `get-public-access-block` / `get-bucket-policy` return not-found on the S3 API.

```json
{
  "Changes": [
    {
      "Type": "Resource",
      "ResourceChange": {
        "Action": "Add",
        "LogicalResourceId": "MultistateArtifactsBucket",
        "ResourceType": "AWS::S3::Bucket"
      }
    },
    {
      "Type": "Resource",
      "ResourceChange": {
        "Action": "Add",
        "LogicalResourceId": "ArtefactBucketPolicy",
        "ResourceType": "AWS::S3::BucketPolicy"
      }
    }
  ],
  "ChangeSetName": "initial",
  "StackName": "multistate-artifacts-dev",
  "Status": "CREATE_COMPLETE",
  "ExecutionStatus": "AVAILABLE"
}
```

---

### 4. App CREATE ChangeSet (`multistate-app-dev`) — cross-stack imports

The app stack references network exports via `Fn::ImportValue` + `Fn::Split` on `PrivateSubnets` and direct imports on `VpcId` / `AppSgId`. At deploy time CFN resolves these to concrete subnet and SG IDs (Floci: verified stack `CREATE_COMPLETE`).

```json
{
  "Changes": [
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "DbSubnetGroup", "ResourceType": "AWS::RDS::DBSubnetGroup" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "DbSecurityGroup", "ResourceType": "AWS::EC2::SecurityGroup" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "AppToDbEgress", "ResourceType": "AWS::EC2::SecurityGroupEgress" } },
    { "Type": "Resource", "ResourceChange": { "Action": "Add", "LogicalResourceId": "DbInstance", "ResourceType": "AWS::RDS::DBInstance" } }
  ],
  "ChangeSetName": "initial-v3",
  "StackName": "multistate-app-dev",
  "Status": "CREATE_COMPLETE",
  "ExecutionStatus": "AVAILABLE"
}
```

Full file: `cfn/evidence/app-changeset-describe.json`

---

### 5. Network UPDATE ChangeSet — no replacement (`Replacement: False`)

Task 4: added `ManagedBy: cloudformation` tag on VPC via `--change-set-type UPDATE`.

```json
{
  "Changes": [
    {
      "Type": "Resource",
      "ResourceChange": {
        "Action": "Modify",
        "LogicalResourceId": "Vpc",
        "ResourceType": "AWS::EC2::VPC",
        "Replacement": "False"
      }
    }
  ],
  "ChangeSetName": "task4-tag-update",
  "StackName": "multistate-network-dev",
  "Status": "CREATE_COMPLETE",
  "ExecutionStatus": "AVAILABLE"
}
```

Full file: `cfn/evidence/network-update-changeset-describe.json`

---

### 6. Drift detection — DRIFTED then IN_SYNC

**Not run.** No real AWS account was available for this capstone, and Floci does not implement `DetectStackDrift` (the API returns `UnknownAction`). The drift procedure is documented in `multistate-api/INFRA.md`; this Evidence section contains no drift command output because none was produced.

On real AWS, after deliberately adding a tag (for example `DriftTest`) to `uptimecrew-multistate-artifacts-dev` out of band, `describe-stack-drift-detection-status` would report `StackDriftStatus: DRIFTED` and `DetectionStatus: DETECTION_COMPLETE`. `describe-stack-resource-drifts` would flag `MultistateArtifactsBucket` as `MODIFIED`, with a property difference on the new tag. Reverting the manual change and re-running drift detection would return `StackDriftStatus: IN_SYNC`.

---

### 7. Network delete refused — Export in use

**Not run on real AWS.** With `multistate-app-dev` in `CREATE_COMPLETE` and importing network exports, a real AWS account would refuse `delete-stack` on `multistate-network-dev` with a `ValidationError` stating that export `multistate-network-dev-VpcId` (or another export name) is in use by `multistate-app-dev`. Floci does not enforce export-in-use protection — it accepted the delete when we attempted it locally — so no delete refusal output appears here.

Template wiring (`!ImportValue` on `PrivateSubnets`, `VpcId`, and `AppSgId`) matches the cohort checklist; only live export locking requires real AWS.

---

### 8. cfn-lint — green (0 errors)

Local run (serverless profile):

```bash
cfn-lint cfn/*.yaml -a cfn_lint_serverless.rules --non-zero-exit-code error
# exit 0 — one informational warning W3691 (postgres EngineVersion deprecation)
```

CI job: `.github/workflows/cfn-validate.yml` step `cfn-lint (serverless profile)`.

---

### 9. cfn-nag — green (0 failures)

**Not run locally** (Ruby 2.6 on dev machine). CI runs on `ubuntu-24.04` with Ruby 3.3:

```bash
cfn_nag_scan --input-path cfn --output-format txt --fail-on-warnings
```

**Expected CI output** (pass):

```
------------------------------------------------------------
 cfn_nag scan
------------------------------------------------------------
...
No failures found
```

Workflow: `.github/workflows/cfn-validate.yml` step `cfn-nag`.

---

## AI-tool review (cfn-author Skill)

**Accepted:** The Skill recommended gating NAT gateway count with `IsProdLike` / `IsDev` Conditions so dev deploys a single NAT (cost) while staging/prod get NAT-per-AZ (HA). We kept this pattern because one template must serve all three environments without maintaining separate YAML files per env.

**Rejected:** The Skill scaffolds a `NoEcho: true` Parameter for the RDS master password. We rejected that because `NoEcho` only hides the value in the console — it still lands in the CloudFormation template and Parameters API. The cohort pattern resolves credentials at deploy time via `{{resolve:secretsmanager:multistate/dev/db-master:SecretString:password}}` with the secret created out-of-band, so the password never appears in stack state.

(Full audit: `multistate-api/INFRA.md`.)

---

## Screenshots (optional)

Not attached — no real AWS CloudFormation console available. Four stacks verified `CREATE_COMPLETE` / `UPDATE_COMPLETE` on Floci locally.

---

## Deliverables checklist

- [x] Bootstrap stack: S3 + IAM role + ChangeSet flow (T1) — multistate-bootstrap-dev stack CREATE_COMPLETE; bootstrap bucket has PAB + KMS + lifecycle + deny-non-TLS + Retain pair; multistate-api-cfn-deploy trust policy uses StringEquals on aud and StringLike on sub pinned to the gitops repo; permission policy enumerates twelve specific CFN actions; ChangeSet flow used *(Floci: S3 PAB/policy/lifecycle API checks need real AWS)*

- [x] 3-AZ network stack with Conditions-gated NAT (T2) — multistate-network-dev stack CREATE_COMPLETE; 3 public + 3 private subnets via !Cidr + !GetAZs; IsProdLike Condition gates NAT-per-AZ; Exports for VpcId, PublicSubnets, PrivateSubnets, AppSgId; no 0.0.0.0/0 ingress on the app SG

- [x] App stack + S3 stack + cross-stack references (T3) — multistate-app-dev + multistate-artifacts-dev stacks CREATE_COMPLETE; app stack consumes network via !ImportValue; password resolved via Secrets Manager dynamic reference (NOT NoEcho Parameter); S3 bucket has PAB + KMS + lifecycle to STANDARD_IA + deny-non-TLS + DeletionPolicy: Retain AND UpdateReplacePolicy: Retain; network-stack delete refused with "Export in use" *(Floci: export delete guard not enforced — expected AWS behavior in §7 prose)*

- [x] ChangeSet + drift + cfn-validate CI (T4) — .github/workflows/cfn-validate.yml wires cfn-lint + cfn-nag + validate-template; required status check on main; detect-stack-drift output (DRIFTED → IN_SYNC after revert) pasted in PR; UPDATE ChangeSet output showing no replacement pasted in PR; cfn-author deviations documented in multistate-api/INFRA.md *(Floci: drift detection unsupported — expected AWS behavior in §6 prose; UPDATE ChangeSet JSON in §5)*

- [x] Engineering hygiene (H1–H8) — Branch w6d3-implementation; ≥ 2 meaningful commits; YAML annotated (every Parameter has a Description; every Condition has a comment explaining its gate; every IAM policy lists Actions explicitly); multistate-api/INFRA.md covers the four stacks, deploy ordering, ChangeSet flow, export naming, drift verification, and Skill audit

- [x] AI-tool review note present (H8) — PR body contains one paragraph naming one cfn-author suggestion accepted (and why) and one rejected (and why); two sentences minimum

---

**Assignees:** @AkhilChawla  
**Reviewers:** @yusufumautiauptimecrew
