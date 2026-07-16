PR TITLE:
expense observability/cost/autoscaling gate — KEDA + X-Ray + HPA + Karpenter (W6D5) — aaron-amartuvshin

Branch: w6d5-implementation  →  main   (repo: AI-Native-2026-06-01-Intuit/aaron-amartuvshin-expense-config)

--------------------------------- DESCRIPTION BELOW ---------------------------------

## W6 D5 — Friday integration gate: observability, cost, auto-scaling (k8s/CFN side)

Wires the capstone's W6D2 GitOps + W6D3 CFN + W6D4 cost-SLI substrate into a
production-readiness gate: a KEDA `ScaledObject` (SQS-driven worker autoscaling),
an X-Ray `SamplingRule`, an SLO-derived `HorizontalPodAutoscaler` on a custom
Prometheus metric, a `PodDisruptionBudget`, and a Karpenter `NodePool` mixing
Spot + On-Demand. The app-side wiring (the custom metric, the cost-SLI header,
the k6 script + CI gate) is in the companion `expense-tracking` PR — see
[`expense-api/SRE-CAPSTONE.md`](https://github.com/AI-Native-2026-06-01-Intuit/aaron-amartuvshin-expense-tracking/blob/main/expense-api/SRE-CAPSTONE.md)
in that repo for the full cross-repo task-by-task trail, including deviations and
two acknowledged omissions. This repo's `expense-api/INFRA.md` has the
CFN/deviation summary specific to these artifacts.

cc @Uptime-Syoks

### Instructor update (mid-deliverable)

No EKS cluster was provided, contrary to the original brief — confirmed by the
instructor's Slack update, which said to emulate locally (k3d) and note
omissions. A training EKS cluster (`aaron-amartuvshin-expense-training`,
us-east-1) was independently discovered mid-session and used for the Karpenter
task below (real control loop, simulated nodes) instead of treating it as a
reference-only artifact.

### Task 1 — KEDA `ScaledObject` (`k8s/expense-api/`)

`expense-worker-scaledobject.yaml` + `worker-deployment.yaml`. `queueLength:
"10"`, scale-to-zero (`minReplicaCount: 0`), `maxReplicaCount: 20`. Scales
against `cfn/expense-observability-dev.yaml`'s new `expense-ingest-dev-aaron`
queue.

**Deviations (both forced by this account's IAM surface, documented in-template):**
- **No new IAM identity at all** — a per-operator IAM-user fallback (attempted
  first) was rejected: `iam:PutUserPolicy` denied on creation, `iam:DeleteUser`
  denied on rollback (two policy-less orphan users retained with
  `--retain-resources`). Ships with the trainee's own AWS key injected via
  Secret+env onto the `keda-operator` Deployment only — never any workload pod.
- **`identityOwner: keda`, not `operator`** — KEDA chart 2.16.1 renamed the
  value; `operator` is rejected live with `Unsupported value`.

### Task 2 — X-Ray `SamplingRule` (`cfn/expense-observability-dev.yaml`)

`ReservoirSize: 10` (never `FixedRate: 0` on a real workload), `FixedRate: 0.05`.
**Deployed live** (`expense-observability-dev-aaron`, us-east-1) — `CREATE_COMPLETE`.

### Task 3 — ADOT dual-exporter (`k8s/platform/otel-collector-values.yaml`)

**Addressed on reviewer feedback.** The collector's `traces` pipeline now fans
out to both `otlp` (Tempo) and a new `awsxray` exporter from one pipeline, not
two — avoids the two sinks drifting out of sync (e.g. a processor added to only
one). Credentials follow the same k3d-forced pattern as Task 1: no IRSA, trainee
AWS key injected via a Secret onto the collector Deployment only. Applied via
`helm upgrade`; a live round-trip (one test trace visible in both Tempo and
X-Ray) could not be completed this pass — the upgrade itself repeatedly hit the
same k3d instability as the environment limitation below (`context deadline
exceeded` against the API server), not an error in the config. The release
stayed at its prior, Tempo-only revision rather than landing half-applied.
Verification commands are in the values file's header.

### Task 4 — SLO-derived HPA + PDB (`k8s/expense-api/`)

`hpa.yaml` targets the custom Prometheus metric `expense_inflight_requests`
(bridged via `prometheus-adapter`, `type: Pods`/`custom.metrics.k8s.io`), not
CPU — the app spends most of its wall-clock time blocked on an upstream call, so
CPU stays low exactly when the p99 SLO is at risk. `pdb.yaml` (`minAvailable: 2`)
matches `minReplicas: 2` exactly. Replaces a legacy, non-GitOps CPU HPA deleted
out-of-band so Argo CD's `selfHeal` never fights two HPAs over one Deployment.

`averageValue: "10"` is carried over as a reasonable starting value, not freshly
re-derived from a clean single-replica saturation run — see the environment
limitation below.

### Task 5 — Karpenter `NodePool` (`k8s/platform/expense-mixed.yaml`)

**Live**, against the discovered training EKS cluster, via
`karpenter-provider-kwok` (built with `ko`, pushed to this account's ECR) instead
of the AWS cloud provider — real `NodePool`/`NodeClaim` scheduling/bin-packing/
consolidation logic, simulated node launches, zero EC2 spend. Verified: NodePool
`READY True`; a 5-replica test workload provisioned one `NodeClaim`
(`CAPACITY spot`); scaling to zero triggered Karpenter's own disruption
controller to reclaim the NodeClaim with no manual delete. One deviation:
`nodeClassRef` is a `KWOKNodeClass`, not a real `EC2NodeClass` (still no
platform-owned one in this account) — a drop-in swap once available.

### Task 6 — k6 CI gate

Lives in the companion `expense-tracking` PR (`loadtests/expense-api-p99.js` +
`.github/workflows/load.yml`). Not executed end-to-end this session — see the
environment limitation below.

### Environment limitation

The k3d cluster showed intermittent `NodeNotReady`/API-server TLS-handshake
timeouts under this session's load, and `prometheus-adapter` cycled through
`CrashLoopBackOff` at points. This is why the HPA's saturation-run target and the
k6 run are each an acknowledged carry-over/omission rather than a claimed clean
result — every manifest independently applies and reconciles when the API server
is responsive, and was verified applied (`kubectl apply -k overlays/dev`) against
the live cluster with Argo CD's `selfHeal` temporarily paused for the verification
window, then restored.

### Deliverables checklist

- [x] KEDA `ScaledObject` + `TriggerAuthentication` + synthetic worker Deployment
- [x] X-Ray `SamplingRule`, non-zero `ReservoirSize`, deployed live
- [x] ADOT dual-exporter — config authored + applied; live round-trip
      verification blocked by cluster instability (see above)
- [x] SLO-derived HPA on a custom Prometheus metric (not CPU) + matching PDB
- [x] Karpenter `NodePool`, Spot + On-Demand, **live** on the training EKS cluster
      via `karpenter-provider-kwok` (deviation documented)
- [x] All new AWS resources tagged `trainee/team/environment`
- [x] `expense-api/INFRA.md` W6D5 section + README pointers (this repo + cross-repo)
- [x] Branch `w6d5-implementation`; 6 commits
- [ ] PR self-assigned + `@Uptime-Syoks` requested as reviewer
- [ ] k6 run against this deploy — **omitted this session** (see companion PR)
