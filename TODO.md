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

## Phase 6 — Portfolio Site + Ask-the-Desk Agent

Plan: [`plans/2026-04-21-afolabi-next-plan.md`](plans/2026-04-21-afolabi-next-plan.md)

### afolabi-next (frontend)

- [x] Scaffold Next.js 16 + Tailwind v4 + MDX at `/Users/afolabi/Desktop/project-pi/afolabi-next/`
- [x] Migrate 5 Hugo posts → `content/posts/*.md` (non-destructive copy from `blog/`)
- [x] Landing, `/posts`, `/posts/[slug]`, `/ask`, `/resume` pages
- [x] `/api/ask` thin proxy → `$AGENT_URL/ask`
- [x] `Dockerfile` (standalone, 3-stage, node:22-alpine, non-root)
- [ ] Populate `content/experience.yaml` with real roles / certifications / education
- [ ] Create Cloudflare R2 bucket for resume PDF (custom domain `resume.cloudhaven.work`)
- [ ] Upload `afolabi-fajobi-resume.pdf` to R2
- [ ] Set `RESUME_URL` in SSM Parameter Store → ESO → pod env
- [ ] `nextjs-ci/` composite in `Cloudhaven-IDP/Actions` (lint + typecheck + build)
- [ ] `afolabi-next/.github/workflows/` — ci + build-deploy-dev + promote-to-prod (mirrors Theo shape)
- [ ] `gitops/apps/afolabi-next/` — Deployment + Service + HTTPRoute + ExternalSecret + overlays
- [ ] Public-zone cert for `afolabi.cloudhaven.work` via existing cert-manager Cloudflare DNS-01 ClusterIssuer
- [ ] ArgoCD Application registered
- [ ] Cloudflare Worker rate-limiter at `afolabi-next/edge/ratelimit/` — KV per-IP token bucket + global cap + Turnstile. Must ship before `/ask` is publicly reachable.
- [ ] Deprecate `blog/` Hugo site — redirect to new domain, archive the dir

### afolabi-desk (agent service — separate repo, upcoming)

- [ ] New repo `Cloudhaven-IDP/afolabi-desk`
- [ ] Python + Strands or OpenAI Agents SDK
- [ ] Bedrock Claude Sonnet via `agent_core.model_client.invoke(task="desk.answer", ...)` (reuse pattern from theo-agents)
- [ ] System prompt + answer-style prompt in Langfuse with 5-min TTL cache
- [ ] RAG over `afolabi-next/content/posts/` + `experience.yaml` (Ollama `nomic-embed-text` embeddings, Qdrant collection `desk_bio` on humboldt)
- [ ] `POST /ask` streaming endpoint matching the contract in `afolabi-next/README.md`
- [ ] `Dockerfile`, `.github/workflows/` (reuse python-ci + docker-build-push-ecr + gitops-bump-overlay)
- [ ] `gitops/apps/afolabi-desk/` — Deployment, Service (ClusterIP:8080), IRSA role for Bedrock
- [ ] Bedrock usage gated / capped to avoid scraper-driven bills

---

## Icebox

- Talos on RPis (operational cleanup, not a priority until after AWS cluster is stable)
- HA Tailscale subnet routers (two EC2 routers in different AZs)
- Oura ring integration (`agents/oura.py` exists, needs wiring)
