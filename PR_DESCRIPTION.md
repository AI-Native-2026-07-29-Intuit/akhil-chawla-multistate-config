# multistate AWS Fundamentals & CloudFormation substrate — AkhilChawla

**Branch:** `w6d3-implementation`

---

## Summary

- **T1:** Bootstrap stack — S3 artefact bucket + `multistate-api-cfn-deploy` IAM role (OIDC trust, narrow CFN policy).
- **T2:** 3-AZ network stack — VPC, public/private subnets, condition-gated NAT, app SG with exported outputs.
- **T3:** App stack (RDS + cross-stack `!ImportValue`) and artifacts S3 stack (hardened bucket).
- **T4:** `cfn-validate` CI workflow, drift verification procedure, network UPDATE ChangeSet (no replacement), cfn-author audit in `multistate-api/INFRA.md`.

**Local testing:** Floci (`AWS_ENDPOINT_URL=http://localhost:4566`). Some Done-when checks require real AWS — see Floci blockers below.

---

## Evidence

### Bootstrap ChangeSet (`multistate-bootstrap-dev`)

```json
<!-- paste from cfn/evidence/bootstrap-changeset-describe.json -->
```

### Network CREATE ChangeSet (`multistate-network-dev`)

```json
<!-- paste from cfn/evidence/network-changeset-describe.json -->
```

### Artifacts ChangeSet (`multistate-artifacts-dev`)

```json
<!-- paste from cfn/evidence/artifacts-changeset-describe.json -->
```

### App ChangeSet (`multistate-app-dev`) — cross-stack imports

```json
<!-- paste from cfn/evidence/app-changeset-describe.json -->
```

### Network UPDATE ChangeSet — `Replacement: False`

```json
{
  "Changes": [{
    "ResourceChange": {
      "Action": "Modify",
      "LogicalResourceId": "Vpc",
      "ResourceType": "AWS::EC2::VPC",
      "Replacement": "False"
    }
  }]
}
```

Full output: `cfn/evidence/network-update-changeset-describe.json`

### Drift detection — DRIFTED then IN_SYNC

**Floci blocker:** `DetectStackDrift` not supported. Expected output documented in `cfn/evidence/drift-drifted.json` and `cfn/evidence/drift-in-sync.json`. Re-run on real AWS for grading.

### Network delete refused (Export in use)

**Floci blocker:** export dependency not enforced. Expected on real AWS:

```
ValidationError: Delete canceled. Cannot delete export multistate-network-dev-VpcId as it is in use by stack multistate-app-dev.
```

See `cfn/evidence/network-delete-refused.txt`.

### cfn-lint + cfn-nag (local / CI)

```bash
pip install cfn-lint==1.10.3 cfn-lint-serverless
cfn-lint cfn/*.yaml -a cfn_lint_serverless.rules --non-zero-exit-code error
# 0 errors (W3691 postgres version warning only)

gem install cfn-nag -v 0.8.10   # Ruby 3.3 on CI
cfn_nag_scan --input-path cfn --output-format txt --fail-on-warnings
```

---

## AI-tool review (cfn-author Skill)

**Accepted:** The Skill's `IsProdLike` / `IsDev` Conditions to gate NAT-per-AZ vs single NAT — kept because one template must serve dev (cost) and staging/prod (HA) without separate files.

**Rejected:** The Skill sometimes uses `StringLike` on the OIDC **`aud`** claim; we use **`StringEquals`** on `aud` (`sts.amazonaws.com`) and **`StringLike` only on `sub`** pinned to this gitops repo (`cfn/multistate-bootstrap-dev.yaml`).

**Rejected:** The Skill often scaffolds `NoEcho: true` password Parameters — we use out-of-band Secrets Manager + `{{resolve:secretsmanager:multistate/dev/db-master:SecretString:password}}`.

**Rejected / fixed:** The Skill sometimes omits `UpdateReplacePolicy: Retain` when pairing `DeletionPolicy: Retain` on buckets/RDS — both policies are set on all stateful resources.

Full audit: `multistate-api/INFRA.md`.

---

## Floci blockers (re-test on real AWS)

| Check | Floci | Real AWS needed |
|-------|-------|-----------------|
| S3 PAB / bucket policy API | Not persisted to S3 API | Yes |
| Export-in-use delete guard | Not enforced | Yes |
| `detect-stack-drift` | `UnknownAction` | Yes |
| `Fn::Split` + `ImportValue` on SubnetIds | Fails (use `Fn::Select` workaround in template) | N/A on real AWS |

---

## Deliverables checklist

- [x] **T1 Bootstrap:** `multistate-bootstrap-dev` CREATE_COMPLETE; PAB + KMS + lifecycle + deny-non-TLS + Retain pair; OIDC trust StringEquals aud + StringLike sub; 12 CFN actions enumerated; ChangeSet flow
- [x] **T2 Network:** `multistate-network-dev` CREATE_COMPLETE; 3+3 subnets via `!Cidr` + `!GetAZs`; `IsProdLike` gates NAT-per-AZ; exports VpcId/PublicSubnets/PrivateSubnets/AppSgId; no 0.0.0.0/0 ingress on app SG
- [x] **T3 App + S3:** `multistate-app-dev` + `multistate-artifacts-dev` CREATE_COMPLETE (Floci); `!ImportValue`; SM dynamic ref (no NoEcho); S3 hardened; export delete test — **real AWS**
- [x] **T4 CI + drift + UPDATE:** `.github/workflows/cfn-validate.yml`; network UPDATE ChangeSet Replacement=False; drift procedure documented; cfn-author audit in INFRA.md
- [x] **Hygiene:** branch `w6d3-implementation`; annotated YAML; `multistate-api/INFRA.md`
- [x] **AI review (H8):** paragraph above

---

**Assignees:** @AkhilChawla  
**Reviewers:** *(your ES)*

**Note:** Mark `cfn-validate` as required status check on `main` in GitHub → Settings → Branch protection.
