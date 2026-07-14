# INFRA.md — expense AWS substrate (W6 D3)

How this service's AWS substrate is provisioned with CloudFormation, in what order,
how it's deployed and verified, and where it deviates from the reference scaffold /
the cfn-author Skill's defaults.

All templates live in [`cfn/`](../cfn) in this gitops repo. All resources are tagged
`trainee=aaron-amartuvshin`, `team=team67`, `environment=training` (applied as
stack-level tags at ChangeSet creation, which propagate to every taggable resource)
and deploy to **us-east-1** only.

## Shared-account naming (the `-aaron` suffix)

Account `726695008378` is shared across **team67**. The canonical unsuffixed names
(`expense-bootstrap-dev`, role `expense-api-cfn-deploy`, bucket
`uptimecrew-expense-bootstrap-dev-<acct>`, secret `expense/dev/db-master`) were
**already deployed by a teammate**, and `<acct>` doesn't disambiguate within one
account. So every *named* resource carries a `-aaron` trainee suffix — the same
convention as the W6 D1 role `expense-api-build-push-aaron`. Stack names, the deploy
role, both bucket names, the DB instance identifier, and the out-of-band secret are
all `-aaron`. Because the `Export.Name`s are built from `${AWS::StackName}`, they
inherit the suffix automatically, and the app stack imports them by the suffixed
stack name (`NetworkStackName` parameter defaults to `expense-network-dev-aaron`).

## Stack layout & deploy ordering

| # | Stack | What it holds | Depends on |
|---|-------|---------------|------------|
| 1 | `expense-bootstrap-dev-aaron` | Artefact S3 bucket (KMS/PAB/lifecycle/deny-non-TLS/Retain×2) + the OIDC `expense-api-cfn-deploy-aaron` role | — |
| 2 | `expense-artifacts-dev-aaron` | Hardened artefact bucket (KMS, PAB, 90d→STANDARD_IA / 365d→GLACIER_IR, deny-non-TLS, deny-unencrypted-PUT, Retain×2) | — |
| 3 | `expense-network-dev-aaron` | 3-AZ VPC, IGW, 3 public + 3 private subnets, dev single-NAT (staging/prod NAT-per-AZ), app SG | — |
| 4 | `expense-app-dev-aaron` | RDS PostgreSQL + DB subnet group + DB SG, consuming (3)'s exports | (3) |

Order: **bootstrap → artifacts → network → app**. 1–3 are independent; 4 must come
after 3 because it `!ImportValue`s the network exports. (The artefact bucket is
deployed before the app stack so it exists before anything writes to it.)

## The ChangeSet deploy flow (every stack)

Never a bare `create-stack`/`update-stack` — always the reviewable three-step:

```bash
aws cloudformation create-change-set  --stack-name <stack> --change-set-name <name> \
    --change-set-type CREATE|UPDATE --template-body file://cfn/<stack>.yaml \
    --parameters ... --tags trainee/team/environment ... \
    [--capabilities CAPABILITY_NAMED_IAM]   # bootstrap only (it creates a named role)
aws cloudformation describe-change-set --stack-name <stack> --change-set-name <name>   # review the resource diff → PR body
aws cloudformation execute-change-set  --stack-name <stack> --change-set-name <name>
aws cloudformation wait stack-{create,update}-complete --stack-name <stack>
```

`describe-change-set` is the gate: on an UPDATE it shows `Replacement: True|False` per
resource, so you catch a would-be data-destroying replacement *before* executing. The
four CREATE diffs and the UPDATE diff are pasted in the PR body.

## Cross-stack references

The network stack `Export`s five values; the app stack imports them. The export names
are `${AWS::StackName}-<Key>`, so:

| Export | Consumed by |
|--------|-------------|
| `expense-network-dev-aaron-PrivateSubnets` | `DbSubnetGroup.SubnetIds` (`!Split "," !ImportValue`) |
| `expense-network-dev-aaron-VpcId` | `DbSecurityGroup.VpcId` |
| `expense-network-dev-aaron-AppSgId` | `DbSecurityGroup` ingress `SourceSecurityGroupId` (5432 from app SG only) |
| `expense-network-dev-aaron-VpcCidr` | `DbSecurityGroup` egress CIDR |

Never hardcode subnet/VPC/SG IDs — imports track a network rebuild automatically. The
safety this buys: an active import **blocks deletion of the exporting stack** — deleting
`expense-network-dev-aaron` while the app stack imports it is refused with
`Export ... cannot be deleted as it is in use by expense-app-dev-aaron`.

## Secrets: CFN-managed secret + dynamic reference, never a NoEcho parameter

The RDS master password is **never** a template `Parameter` (a `NoEcho` parameter still
lands in the parameter store/history and in tooling paths). Instead the app stack:

1. **Creates the secret in-stack** — `AWS::SecretsManager::Secret` `DbMasterSecret` with
   `GenerateSecretString` (CFN generates the 32-char password at create time; no plaintext
   input ever exists), `DeletionPolicy: Retain` + `UpdateReplacePolicy: Retain`.
2. **Resolves it into the RDS** via the dynamic reference
   `{{resolve:secretsmanager:expense/dev/db-master-aaron:SecretString:password}}` — CFN
   substitutes the value straight into the RDS API call; it never enters template state.
   The `DbInstance` carries `DependsOn: DbMasterSecret` because a dynamic reference does
   **not** create a dependency, so creation order must be pinned explicitly.
3. **Binds them** with `AWS::SecretsManager::SecretTargetAttachment` so Secrets Manager
   rotation can target the live DB ARN and the secret records host/port/dbname.

## Drift verification

```bash
# deliberate console edit: add a tag to the artefact bucket, then —
aws cloudformation detect-stack-drift --stack-name expense-artifacts-dev-aaron
aws cloudformation describe-stack-resource-drifts --stack-name expense-artifacts-dev-aaron
# → StackResourceDriftStatus: MODIFIED for the bucket; revert the console edit → IN_SYNC
```

The DRIFTED→IN_SYNC output is in the PR body. Drift detection is the rhythm that catches
an out-of-band console change before it becomes an un-rollbackable 3am surprise.

## Deviations from the reference / cfn-author Skill defaults

- **`-aaron` suffix everywhere** — shared-account collision avoidance (above).
- **`cfn-validate.yml` runs on `[self-hosted, mac-local]`, not `ubuntu-24.04`** — this org
  is self-hosted-only (no GitHub-hosted Linux runner). cfn-lint + cfn-nag run via
  `docker run` on Rancher, exactly how the app repo runs hadolint + Trivy; cfn-nag also
  can't `gem install` on a modern local ruby (its old `kwalify` dep breaks on ruby ≥ 4),
  so the container is the portable path. `validate-template` uses OIDC against
  `expense-api-cfn-deploy-aaron`.
- **App stack manages the secret in-stack** — `DbMasterSecret` (GenerateSecretString) +
  `SecretTargetAttachment`, resolved via dynamic reference (see the Secrets section). An
  earlier revision hand-created the secret out-of-band and resolved that; switched to the
  in-stack pattern on ES review (W6D3) so CFN owns the credential lifecycle end-to-end.
- **`DeletionProtection: true` on the RDS** — the instance can't be dropped by accident;
  a genuine teardown disables it first (`modify-db-instance --no-deletion-protection`).
  `DeletionPolicy: Retain` + `UpdateReplacePolicy: Retain` add stack-level guards.
- **PostgreSQL `16.9`** — the reference's `16.3`/`16.4` are now deprecated for new
  instances (cfn-lint W3691); `16.9` is a current non-deprecated 16.x.
- **VPC flow logs omitted in dev** (cfn-nag W60 suppressed) — they add CloudWatch/S3
  delivery cost; staging/prod would attach one. Documented, dated suppression in-template.
- **cfn-nag warning suppressions** — every suppression (W28 explicit names, W33 public-subnet
  auto-IP, W35 bucket access-logging, W40/W5 SG egress, F38 scoped PassRole, W60 flow log)
  carries an inline `reason:` so `--fail-on-warnings` runs clean without blanket-disabling
  rules. cfn-lint: 0 findings. cfn-nag: 0 failures, 0 unsuppressed warnings.

## AI-tool review (cfn-author Skill audit)

The **`cfn-author` Skill is not installed** in this environment (same as `argocd-author`
in W6 D2), so the four templates were hand-authored and self-audited against the cohort
checklist of known cfn-author failure modes. All three documented quirks were checked and
avoided:

1. **`StringLike` on the IRSA/OIDC `aud` claim** → the bootstrap trust policy uses
   **`StringEquals`** on `token.actions.githubusercontent.com:aud` (`sts.amazonaws.com`),
   reserving `StringLike` for the `sub` claim only (branch + PR refs vary per run). **Accepted**
   the checklist's insistence here — an exact-match `aud` is the correct, non-negotiable shape.
2. **`NoEcho: true` on a password `Parameter`** → avoided entirely; the password is out-of-band
   in Secrets Manager and resolved via `{{resolve:secretsmanager:...}}` (see above).
3. **`DeletionPolicy: Retain` without its `UpdateReplacePolicy: Retain` pair** → every stateful
   resource (three S3 buckets across bootstrap/artifacts, the RDS instance) carries **both**.

**Rejected** the scaffold's cost-carrying dev extras — a **dedicated KMS CMK** for the DB
secret (cfn-nag W77) and a **VPC flow log** (W60). In dev these use the AWS-managed
`aws/secretsmanager` key and no flow log respectively, each a dated in-template suppression
with a staging/prod-flips-this note. The ~$0 training-account policy makes a per-env CMK and
always-on flow-log delivery not worth their cost here; they belong in the hardened envs.

**Incorporated on ES review (W6D3):** an earlier revision hand-created the DB secret
out-of-band and set `DeletionProtection: false` for a clean teardown. On feedback, the app
stack now **owns** the secret in-stack (`DbMasterSecret` + `SecretTargetAttachment`) and runs
with `DeletionProtection: true` — accident-proofing wins over teardown convenience, and CFN
owning the credential lifecycle is the stronger pattern.
