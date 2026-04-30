# Self-hosted GHA runners on humboldt — ARC + Karpenter + Spot

**Status:** ready for review, not yet implemented
**Tracking issues:** TBD — opening one on `Cloudhaven-IDP/Infrastructure` alongside this plan
**Blocks:** theo-agents arm64 builds, CodeArtifact-authenticated image pulls, argocd-sync-prod workflow (needs Tailscale access)

---

## Context

GitHub-hosted runners can't join the tailnet, can't assume our `gha-deployer` IRSA role without workarounds, and cost credits at higher CI volume. For the three new repos (theo, theo-agents, theo-wearables) we also want native arm64 Docker builds to match humboldt / Graviton — QEMU cross-builds under buildx are 3–5× slower and burn billable CI minutes.

Fargate would have been the obvious scale-to-zero answer, but **Fargate is EKS-only** and humboldt is Talos on EC2 — the compatibility story doesn't exist. Options reduce to:

| Option | Fit | Verdict |
|---|---|---|
| GitHub-hosted runners | No tailnet, no IRSA, QEMU arm64 | Current state — unblocked for public repos, but limits every auth-requiring CI step |
| Always-on runner Deployment on humboldt | Fine at low volume, wastes RSS at idle | Acceptable stopgap, not end-state |
| Separate EKS cluster with Fargate profile for runners | Pay-per-use, scale-to-zero | Rejected — $73/mo for the EKS control plane alone + two clusters to babysit for one workload |
| **ARC + Karpenter + Spot on humboldt** | Scale-to-zero EC2, arm64 native, single cluster | **Chosen.** Matches Fargate ergonomics without the second cluster; fits the single-cluster posture we already have |

---

## Design

```
GitHub job starts
  → ARC webhook receives `workflow_job.queued`
  → ARC scales RunnerDeployment replica 0 → 1
  → Runner pod pending (taint `ci-only:NoSchedule` unmatched by existing nodes)
  → Karpenter provisions Spot arm64 EC2 (c6g.large or c6g.xlarge)
  → Node joins, runner pod schedules, registers with GitHub
  → Job runs; on completion, runner exits, Karpenter drains + terminates node after empty-timeout (30s default)
```

### Components

**ARC (Actions Runner Controller):**
- Namespace `arc-system`
- Install via the `actions/actions-runner-controller` Helm chart
- Authenticated as a **GitHub App** installed on `Cloudhaven-IDP` org (PAT not acceptable for org-level)
- App credentials in AWS Secrets Manager → ESO `ExternalSecret` → `actions-runner-controller-auth`

**Karpenter:**
- Namespace `karpenter`
- Helm chart, IRSA for EC2 provisioning + SQS interruption handling
- One `NodePool` for CI (`ci-spot-arm64`):
  - `requirements`: `kubernetes.io/arch: arm64`, `karpenter.sh/capacity-type: spot`, `karpenter.k8s.aws/instance-category: c`, generation 6+
  - `taints`: `ci-only:NoSchedule` (runners tolerate; platform workloads don't)
  - `limits`: `cpu: 32` (hard cap on CI-driven Spot spend)
  - `disruption.consolidationPolicy: WhenEmpty`, `consolidateAfter: 30s`
- `EC2NodeClass` for the CI pool:
  - `amiFamily: Bottlerocket` (Talos would be nicer but Karpenter has no Talos support today — Bottlerocket is the closest declarative AMI family)
  - Default subnet + security group selectors via tags

**Runner set:**
- One `RunnerDeployment` per repo initially; consolidate to an org-level set later if GitHub App perms allow
- Runner image: **custom, baked into ECR** — base on `ghcr.io/actions/actions-runner:latest`, add `docker buildx`, `crane`, `kubectl`, `aws-cli v2`, `tailscale` client, GitHub CLI. Rebuilt on a CronJob from the upstream release.
- Pod spec: `tolerations: [{key: ci-only, effect: NoSchedule}]`, `nodeSelector: {karpenter.sh/nodepool: ci-spot-arm64}`
- ServiceAccount with IRSA → `gha-runner-{repo}` role. Each repo's role has just the permissions that repo's CI actually uses — no shared super-role.

### Tailscale reachability

Runners need the tailnet to reach:
- `argocd-server.internal.cloudhaven.work` for `argocd-sync-prod`
- Eventually: internal Helm chart repos, private registries

Path: **Tailscale sidecar container** in the runner pod (`ghcr.io/tailscale/tailscale`), auth via OAuth client credentials minted for `tag:ci-runner`. Tailscale ACL rule: `tag:ci-runner → tag:humboldt-argocd:443` only. No broader tailnet reach.

### IRSA / ECR / CodeArtifact

Each runner pod's SA gets:
- `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:PutImage`, `ecr:InitiateLayerUpload`, etc. on `cloudhaven-dev` + `cloudhaven-prod`
- `codeartifact:GetAuthorizationToken`, `codeartifact:ReadFromRepository` on `cloudhaven/python-private`
- `sts:AssumeRoleWithWebIdentity` (from the OIDC trust policy on `gha-runner-{repo}`)

The existing `gha-deployer` trust policy pattern is the template — same shape, narrower per-repo scope.

---

## Repo placement

| Concern | Path | Repo |
|---|---|---|
| Karpenter operator install + NodePool + EC2NodeClass | `K8s-Bootstrap/humboldt/karpenter/` | K8s-Bootstrap |
| ARC controller install + GitHub App ExternalSecret | `K8s-Bootstrap/humboldt/arc/` | K8s-Bootstrap |
| Runner image Dockerfile + build workflow | `Cloudhaven-IDP/Actions/runner-image/` (new subdir) | Actions |
| Per-repo `RunnerDeployment` | `gitops/apps/arc-runners/{repo}/` | gitops |
| Karpenter IRSA + per-repo `gha-runner-{repo}` IAM roles | `Infrastructure/platform/humboldt/karpenter-iam.tf` + `Infrastructure/modules/aws/iam/gha-runner/` | Infrastructure |
| Tailscale ACL `tag:ci-runner` | `Infrastructure/platform/humboldt/tailscale/acl.tf` | Infrastructure |

---

## Phasing

Each phase is independently demoable. Don't start N+1 until N is green.

### Phase 0 — Karpenter install (~3h)
- `K8s-Bootstrap/humboldt/karpenter/` Helm install
- TF: Karpenter controller IRSA role + SQS queue for Spot interruption handling
- Smoke: apply a test `NodePool` + `EC2NodeClass` with `limits.cpu: 2`, schedule a `Deployment` with the matching toleration, verify Karpenter provisions a Spot node within 60s

### Phase 1 — ARC controller + GitHub App (~2h)
- Register GitHub App on `Cloudhaven-IDP` org, install on the three repos (theo, theo-agents, theo-wearables). Permissions: Actions read/write, Checks read/write, Metadata read
- App credentials → AWS Secrets Manager → ESO `ExternalSecret` in `arc-system`
- ARC Helm install, pointing at the secret
- Smoke: `kubectl get horizontalrunnerautoscalers -n arc-system` healthy, no auth errors in controller logs

### Phase 2 — Custom runner image (~2h)
- `Cloudhaven-IDP/Actions/runner-image/Dockerfile` based on `ghcr.io/actions/actions-runner:latest`
- `Cloudhaven-IDP/Actions/.github/workflows/build-runner-image.yaml` — CronJob weekly, OIDC → ECR push
- Push arm64 + amd64 variants to `cloudhaven-dev`

### Phase 3 — First `RunnerDeployment` for theo-agents (~2h)
- `gitops/apps/arc-runners/theo-agents/` — `RunnerDeployment` spec, toleration, IRSA SA
- TF: `gha-runner-theo-agents` role with just the permissions theo-agents CI needs
- Smoke: open a PR on theo-agents, watch Karpenter provision a Spot node, runner pod schedule, job complete, node drain + terminate

### Phase 4 — Expand to theo + theo-wearables (~1h)
- Same pattern, two more `RunnerDeployment`s, two more IRSA roles
- Update Tailscale ACL to include `tag:ci-runner` if not already

### Phase 5 — Observability (~1h, deferred until VM lands)
- Karpenter `ServiceMonitor` → VictoriaMetrics
- Dashboard: nodes provisioned/deprovisioned, Spot interruption rate, time-to-schedule for runner pods
- Alert: Spot interruption rate > 10%/day (rebalance to on-demand or different AZ)

---

## Cost

Expected steady state at personal-project CI volume (~5–15 jobs/day, ~3–8 min each):

| Line | Estimate |
|---|---|
| Karpenter controller itself (always-on, 256 MiB on humboldt) | $0 marginal (rides existing worker) |
| Spot c6g.xlarge during CI (~30–60 runner-minutes/day × ~$0.05/hr Spot) | **~$2–5/mo** |
| ECR storage for runner images (2 arches × ~500 MB) | <$0.50/mo |
| EC2 data transfer (image pulls) | Negligible (same region) |

**Expected all-in: <$10/mo.** Compared to the current plan (GitHub-hosted runners on public repos, manually approving every workflow run from a PR, QEMU cross-builds eating CI minutes), the cost case is weaker than the *capability* case — we actually need IRSA + Tailscale + native arm64, which GitHub-hosted can't provide.

---

## Tradeoffs

- **Spot interruptions.** Karpenter's Spot provisioning has a ~2min eviction notice; ARC runners don't handle graceful eviction mid-job. Accept: CI jobs are retriable. Monitor Spot interruption rate; if >10%/day rebalance to different instance families or fall back to on-demand for that NodePool.
- **Bottlerocket, not Talos, on the runner nodes.** Karpenter doesn't have Talos AMI support today. Accept: CI nodes are ephemeral, compliance/consistency matters less than for platform workloads. If Karpenter adds Talos AMI support later, one line change in `EC2NodeClass.amiFamily`.
- **Custom runner image maintenance.** Weekly CronJob rebuild is manageable; the base image moves slowly. If it becomes a burden, switch to the upstream `actions/actions-runner` directly and mount buildx/crane/etc. via init container.

---

## Open questions

1. **GitHub App org permissions scope.** Can one App serve all three repos' runner registration, or does `RunnerDeployment` require per-repo fine-grained tokens? Leaning: one App, org-level. Verify in ARC docs before Phase 1.
2. **Runner image: `ghcr.io/actions/actions-runner` or DIY from scratch?** Base image is maintained by GitHub; saves us building glibc + toolchain. Go with the base.
3. **ARC v0 vs v2 (scale-from-zero).** ARC v2 is scale-from-zero and is where GitHub is investing; v0 is on maintenance. Go with v2 from day one.
4. **Karpenter vs Cluster Autoscaler.** Cluster Autoscaler works with any AMI family including Talos, but is per-ASG and slower. Karpenter is faster and more flexible but AMI-family constrained. Karpenter wins for this workload.
5. **Talos workers on the rest of humboldt stay Talos.** Karpenter-provisioned CI nodes are Bottlerocket, not Talos. Two AMIs in one cluster is fine; just flag in observability that CI node OS differs.

---

## Acceptance criteria

- [ ] Open a PR on theo-agents → Karpenter provisions a Spot arm64 EC2 node within 90s → ARC runner pod schedules → job runs to completion → node drains + terminates within 60s of runner exit.
- [ ] Total CI jobs for a week show <10% Spot interruption rate.
- [ ] `argocd-sync-prod` from a runner pod over Tailscale reaches the internal ArgoCD API successfully.
- [ ] No credentials (GitHub App PEM, runner registration tokens, IRSA session tokens) appear in any committed YAML, Terraform, or Docker image layer.
- [ ] `kubectl get nodes` when CI is idle shows zero Karpenter-provisioned nodes.

---

## Dependencies on other plans

- **postgres-platform plan** — independent. Karpenter can land in parallel.
- **Theo v3 scaffold** — theo, theo-agents, theo-wearables CI workflows are ready; they'll start pointing at the new runner set when Phase 3–4 lands. Until then their GitHub-hosted runs stay as-is.
- **Temporal + observability rollout** — independent. Karpenter metrics land when VM is up.
- **Tailscale ACL Terraform** — needs the `tag:ci-runner` rule before Phase 3 runners try to reach internal ArgoCD.
