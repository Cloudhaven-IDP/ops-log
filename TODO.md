# TODO

Living backlog for the Cloudhaven fitness agent project. Updated as work progresses.
See `log/` for daily entries on what was actually done.

---

## Phase 0 — RPi Cluster Cleanup ✅

- [x] Wiped all 3 RPis with k3s-uninstall.sh / k3s-agent-uninstall.sh + reboot
  - pi-1 (10.0.0.179) — control plane, needed manual reboot after script killed SSH session
  - pi-2 (10.0.0.226) — worker, clean
  - pi-3 (10.0.0.129) — worker, clean
- [ ] Fix K8s-Bootstrap repo — remove `afo-pi-cluster/` dir, replace with `nebulosa/` structure

---

## Phase 1 — Tailscale Mesh

- [x] Tailscale on all 3 nebulosa RPis
  - pi-1 → 100.86.215.90
  - pi-2 → 100.111.58.89
  - pi-3 → 100.94.214.115
- [ ] Tailscale ACL policy + tags via Terraform (Tailscale provider) — `tag:nebulosa`, `tag:humboldt`
- [ ] Terraform: Tailscale EC2 subnet router module (advertises VPC private CIDRs into tailnet)
  - t3.micro, tagged auth key from Secrets Manager, SSM only (no SSH)
  - `autoApprovers` policy so routes don't need manual approval
- [ ] Validate: laptop → nebulosa nodes via Tailscale IPs ✅ (already working)
- [ ] Validate: AWS resources → nebulosa via Tailscale
- [ ] Add Tailscale module to `Infrastructure/modules/`

---

## Phase 2 — AWS EC2 + Talos Cluster

- [ ] Provision EC2 instances for Talos control plane + workers
- [ ] Talos config generation + bootstrap
- [ ] Connect AWS Talos cluster to Tailscale mesh
- [ ] ArgoCD deployed in AWS cluster, both clusters registered
  - Home cluster reachable via Tailscale
- [ ] Move observability heavy stack to AWS Talos
  - VictoriaMetrics
  - Loki
  - Grafana (internal only — Tailscale access, no public exposure)
- [ ] ArgoCD managing K8s-Bootstrap manifests for home cluster as downstream target

---

## Phase 3 — Fitness Agent Infra

- [ ] `terraform/envs/prod/main.tf` — fitness agent prod root module
- [ ] DynamoDB tables (6 tables from schema)
- [ ] S3 bucket (`fitness-agent-data-{account-id}`)
- [ ] IAM roles for agent workloads (IRSA / pod identity)
- [ ] ECR or GHCR image repos for agent services
- [ ] Secrets Manager entries (Nutritionix, LangSmith)

---

## Phase 4 — Fitness Agent Deployment

- [ ] Orchestrator agent — home RPi cluster (k3s)
- [ ] Nutrition agent — AWS Talos cluster
- [ ] Fitness agent — AWS Talos cluster
- [ ] Coach agent — AWS Talos cluster
- [ ] Meal agent — AWS Talos cluster
- [ ] Arize Phoenix (LLM observability) — AWS Talos cluster
- [ ] APScheduler proactive jobs (morning/lunch/evening check-ins)
- [ ] Mac Mini: Ollama pull `nomic-embed-text` + light model
- [ ] End-to-end test: message → intent → Qdrant → specialist agent → response

---

## Phase 5 — Mobile App

- [ ] React Native Expo app (`svc-frontend/`)
- [ ] Apple Health iOS Shortcut → sync endpoint
- [ ] TestFlight distribution

---

## Icebox

- Talos on RPis (operational cleanup, not a priority until after AWS cluster is stable)
- HA Tailscale subnet routers (two EC2 routers in different AZs)
- Oura ring integration (`agents/oura.py` exists, needs wiring)
