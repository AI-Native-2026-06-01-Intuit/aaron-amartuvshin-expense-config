# aaron-amartuvshin-expense-config

GitOps config repo for `expense-api` (W6 D2). Argo CD (in the k3d cluster) watches
`main` and pulls desired state in — the cluster no longer accepts inbound `kubectl apply`.

- `base/` — the W5 D3 manifests (namespace/deployment/service/configmap/secret/servicemonitor).
- `overlays/{dev,staging,prod}/` — Kustomize overlays (namespace + replicas + image tag + log level).
- `argocd/` — AppProject (`expense`), the dev Application anchor, and the ApplicationSet matrix.
- `argocd-system/` — notifications ConfigMap (Slack on sync-failed / health-degraded).
- `cfn/` — CloudFormation substrate: W6 D3 bootstrap/network/artifacts/app stacks, plus the
  W6 D4 `expense-cost-dev.yaml` cost stack (SNS + cost alarm + budget + CUR; model via the
  Anthropic API, so no Bedrock IRSA role). See
  `expense-api/INFRA.md`; LLM cost accounting is documented in the app repo's `expense-api/COST.md`.

Ongoing deploys land as **image-bump PRs** opened by the app repo's `_bump-config` workflow.
