# Scaffold Theo + siblings — revised plan (v3)

## Context

v2 assumed one Theo image with a CMD switch for API vs worker, v2 left wearables for "later," and v2 defaulted shared libs to git submodule. v3 reshapes based on session feedback:

- **Max separation of concerns** for Theo: 3 Dockerfiles, 3 Deployments (api, orchestrator, worker) in one path-filtered monorepo — same shape as `theo-agents/`. A coach-prompt tweak rebuilds only the orchestrator image.
- **2 ECR repos total** (`cloudhaven-dev`, `cloudhaven-prod`) with `<project>-<component>-<sha>` tagging — not 8 repos. Simpler lifecycle, same isolation via tag namespace.
- **Platform infra (Temporal, Langfuse, KEDA, cnpg, cert-manager, observability) lives in `K8s-Bootstrap/humboldt/`** — not `gitops/apps/`. `gitops/apps/` is for user workloads (Theo, agents, wearables).
- **Envoy Gateway is the gateway** — already deployed on both clusters (`K8s-Bootstrap/humboldt/envoy-gateway`, `K8s-Bootstrap/nebulosa/envoy-gateway`). Clerk JWT verification moves to the gateway layer via `SecurityPolicy`.
- **Wearables get a home** — `theo-wearables/` monorepo: per-vendor Go CronJobs (Whoop, Oura) + Apple Health push receiver.
- **Shared libs via AWS CodeArtifact** — private PyPI, IRSA auth from in-cluster runners.
- **mem0 stays skipped** — periodic Temporal activity uses Sonnet to summarize and update the S3 profile; extraction does NOT run per interaction.
- **Weekly Sunday report** default; on-demand daily brief.
- **Daily LLM-as-judge** reads thumbs-down archive → creates DRAFT Langfuse prompt versions for human promotion.
- **Shared CI lives in `Cloudhaven-IDP/Actions`** (already exists on GitHub) — clone alongside these repos; reusable workflows + composites.
- **Prod ArgoCD apps do NOT autosync.** CI triggers sync via `argocd app sync` with SSO. Dev apps autosync.
- **Flannel caveat**: both clusters run Flannel, which does not enforce NetworkPolicies. Until we swap CNI, isolation is enforced at app layer (HMAC + Envoy `AuthorizationPolicy`) — policy manifests still ship, flagged as non-enforcing.

### Author rules (apply while scaffolding)

- **No unnecessary comments.** Code is self-documenting. Only comment a non-obvious *why* (hidden constraint, subtle invariant, workaround for a specific bug). No explanatory what-it-does comments, no multi-line docstrings beyond one short summary line, no "# removed X" breadcrumbs, no reference-the-task comments.
- **Ops-log entry required.** Every meaningful scaffolding/infra session gets an entry in `ops-log/log/YYYY-MM-DD.md` (create if missing). Follow the existing structure: *Carry-Forward* → topic sections with *Rule:* callouts for gotchas → *Next Session — Priorities* → *Backlog*. Scaffolding this plan = one ops-log entry for the day it happens.

---

## Decisions (v3)

| Area | Decision |
|---|---|
| Theo split | **3 Dockerfiles, 3 Deployments, path-filtered monorepo** (same pattern as `theo-agents/`): `theo/api/`, `theo/orchestrator/`, `theo/worker/` at repo root. CI detects changed subtree and only rebuilds that image. Orchestrator stays in `theo/` (not moved to theo-agents — it is Theo's brain, not a specialist). |
| Gateway | **Envoy Gateway** — both clusters already run it. `SecurityPolicy` does Clerk JWT verify at the edge; Theo trusts the signed `X-Soma-User-Id` header Envoy injects. |
| Sub-agents | Only **nutrition** and **meal** as siblings, in `theo-agents/` monorepo with path-filtered CI. |
| Wearables | **`theo-wearables/` monorepo** — `whoop/` (Go CronJob), `oura/` (Go CronJob), `apple/` (Go push receiver behind Envoy). Shared Go libs at `libs/vendor-core/`. |
| Shared Python lib | **AWS CodeArtifact private PyPI** (`theo-agents/libs/agent-core` publishes `agent-core` wheel). Theo, nutrition, meal all `pip install agent-core`. Free at our scale. |
| Registry (images) | **2 ECR repos total**: `cloudhaven-dev`, `cloudhaven-prod`. Images tagged `<project>-<component>-<sha>` (e.g. `theo-api-abc123`, `theo-agents-nutrition-abc123`, `theo-wearables-whoop-abc123`). Single lifecycle policy per env, tag namespace handles isolation. `crane copy` still promotes between the two. |
| Preview envs | **Per-PR namespace**, 3-day TTL, auto-swept. URL: `theo-pr-<N>.internal.cloudhaven.work` via Envoy `HTTPRoute`. |
| Promotion | `crane copy dev→prod` on merge, then bump gitops prod overlay. |
| Autosync | Dev apps **autosync**; prod apps `syncPolicy: {}`, synced by CI via `argocd app sync` (OIDC SSO). |
| Theo ↔ agent data | Theo never touches nutrition/meal DynamoDB directly — always HTTP. |
| Theo owns | Workout logs, sleep logs, chat history, feedback events, S3 user profile writes. |
| Wearables own | Raw wearable data: `soma_whoop_raw`, `soma_oura_raw`, `soma_apple_raw` → also push time-series to VictoriaMetrics. |
| Qdrant | Fresh `theo_memory`, 1024-dim. Deployed on **humboldt** with **EBS gp3** (Longhorn cluster was wiped). bge-m3 embeddings served by Ollama on nebulosa. |
| User structured state | **Postgres (cnpg)** on humboldt. Tables: `user_secrets` (Apple HMAC keys, per-user OAuth tokens for Whoop/Oura), `user_preferences` (report cadence, quiet hours). DynamoDB stays for event streams only. |
| LLM gateway | Direct Bedrock inference profiles via `agent_core.model_client`. LiteLLM swap is one-file later. |
| Memory extraction | Temporal activity with Sonnet summarizes 24h chat → updates `s3://soma-user-data/<user>/{user_profile,patterns}.md`. **Gated**: only runs for users who received a thumbs-down in the last 24h, not every user nightly. Cost-conscious while we tune. |
| Feedback loop | Thumbs-down → dump conversation to `s3://soma-feedback/...` + `langfuse.score()`. **Daily** Temporal workflow runs LLM-as-judge, creates DRAFT Langfuse prompt versions. Prompts can also be edited directly in the **Langfuse UI** at any time (always available — LLM-as-judge is just one proposal source). |
| Reports | **Weekly Sunday 18:00 local** default; on-demand daily brief via `/chat "daily brief"` or preference toggle. |
| Auth | Clerk JWT at Envoy Gateway. HMAC internal between services. Agents never see Clerk tokens. |
| Logging | `structlog` JSON → stdout → **Grafana Alloy DaemonSet** → Loki on humboldt. Alloy replaces Fluentd/Promtail — current Grafana recommendation. |
| Metrics | Prometheus-format from each service → Alloy → VictoriaMetrics on humboldt. |
| NetworkPolicies | Written but **non-enforcing** on Flannel today. App-layer isolation via HMAC + Envoy `AuthorizationPolicy`. Future CNI = **Canal** (Calico's policy engine layered on Flannel's VXLAN overlay) — not Cilium. Canal runs on RPi without blowing resource budget. |
| Parlant | Called only from proactive workflows. Thin client. |
| CI | Repos consume reusable workflows from **`Cloudhaven-IDP/Actions`** (already exists). Workflows + composite actions go there. |
| GHA → AWS | OIDC `gha-deployer` role already exists in Terraform — extend trust policy for theo/theo-agents/theo-wearables repos + a `ci-bot` GitHub App for gitops commits (repo ruleset blocks direct push to main). |
| Frontend | Out of scope this pass. Note: React Native + Expo supports web via react-native-web — same codebase → iOS/Android/Web. |

---

## Pre-deploy infra checklist (ordered)

Nothing below is scaffolded *in this pass* — but it must exist before agents run. Where marked "ship with scaffolds," the Helm values / ArgoCD app definition is authored in `gitops/` alongside the code scaffolds; where marked "separate pass," it goes on the backlog.

| # | Infra | Current | Action | Where |
|---|---|---|---|---|
Platform infra (Temporal, Langfuse, KEDA, cnpg, observability) belongs in **`K8s-Bootstrap/humboldt/`** next to the existing `envoy-gateway/`, `argocd/`, `ebs-csi/`, `tailscale-operator/` entries — not `gitops/apps/`. `gitops/apps/` is for user workloads (Theo, agents, wearables); `K8s-Bootstrap/` is for platform services installed once and shared by all tenants.

| # | Infra | Current | Action | Where |
|---|---|---|---|---|
| 1 | **Envoy Gateway** (humboldt + nebulosa) | ✓ Deployed | Add `SecurityPolicy` (Clerk JWKS), `HTTPRoute` per service, wildcard cert for `*.internal.cloudhaven.work` | `K8s-Bootstrap/humboldt/envoy-gateway/` + `.../nebulosa/...` (ship with scaffolds) |
| 2 | **cert-manager** on humboldt-internal gateway | Missing | ClusterIssuer with Cloudflare DNS-01 solver (DNS already Cloudflare-managed); issues `*.internal.cloudhaven.work` cert consumed by Envoy Gateway | `K8s-Bootstrap/humboldt/cert-manager/` (ship with scaffolds) |
| 3 | **ECR** (2 repos: `cloudhaven-dev`, `cloudhaven-prod`) | Only `cloudhaven-agent` exists | Terraform 2 repos + lifecycle policies; images tagged `<project>-<component>-<sha>` | `Infrastructure/platform/shared/ecr/` (ship with scaffolds) |
| 4 | **ECR pull auth — humboldt** | Missing | Node IAM gets `AmazonEC2ContainerRegistryReadOnly`; kubelet credential provider handles pulls. No `imagePullSecrets` needed. | `Infrastructure/platform/humboldt/aws/iam/` (ship with scaffolds) |
| 5 | **ECR pull auth — nebulosa** | Missing | RPi has no EC2 metadata → `ecr-credential-refresher` CronJob (runs every 8h) mints a 12h ECR token via a dedicated IAM user or cross-account role and refreshes a `regcred` docker-registry Secret in each target namespace. | `K8s-Bootstrap/nebulosa/ecr-refresher/` (ship with scaffolds) |
| 6 | **CodeArtifact** | Missing | Terraform: domain `cloudhaven`, repo `python-private`, IAM policy for GHA OIDC role + cluster IRSA | `Infrastructure/platform/shared/codeartifact/` (ship with scaffolds) |
| 7 | **cnpg operator on humboldt** | ✓ On nebulosa only | Mirror `Infrastructure/platform/pi-cluster/operators/cnpg.tf` for humboldt | `K8s-Bootstrap/humboldt/cnpg/` + `Infrastructure/platform/humboldt/operators/cnpg.tf` (separate pass) |
| 8 | **Langfuse** | Missing | Helm chart on humboldt, Postgres via cnpg Cluster, S3 for blobs | `K8s-Bootstrap/humboldt/langfuse/` (separate pass, blocks prompt registry) |
| 9 | **Temporal** | Missing | Temporal Helm chart on humboldt, cnpg Postgres, namespace `soma` | `K8s-Bootstrap/humboldt/temporal/` (separate pass, blocks workflows) |
| 10 | **KEDA** | Missing | KEDA Helm on humboldt for workflow worker autoscaling on task-queue depth | `K8s-Bootstrap/humboldt/keda/` (separate pass) |
| 11 | **Loki + VictoriaMetrics + Grafana** | Missing | Helm charts on humboldt. Grafana Tailscale-only (no public ingress) | `K8s-Bootstrap/humboldt/observability/` (separate pass, blocks logging) |
| 12 | **Grafana Alloy** DaemonSet | Missing | Both clusters, tails pod stdout → Loki, scrapes metrics → VictoriaMetrics | `K8s-Bootstrap/{humboldt,nebulosa}/alloy/` (ship with scaffolds — agents can log via stdout even before Alloy lands) |
| 13 | **GHA OIDC extensions** | Partial (`gha-deployer` exists) | Extend trust for 3 new repos + create `ci-bot` GitHub App for gitops PR/commits | `Infrastructure/platform/shared/iam/github-oidc/` (ship with scaffolds) |
| 14 | **ARC (Actions Runner Controller)** | Missing | Self-hosted runners on humboldt, registered to Cloudhaven-IDP org | `K8s-Bootstrap/humboldt/arc/` (**backlog**) |
| 15 | **Qdrant on humboldt** | Longhorn cluster wiped | Helm chart on humboldt with **EBS gp3** PVC (5Gi), WaitForFirstConsumer | `gitops/apps/qdrant/` (separate pass) |
| 16 | **Ollama bge-m3** on nebulosa | Missing | Deployment + `local-path` PVC on Pi B (1TB NVMe) for model weights. Nodeselector pins to Pi B. | `gitops/apps/ollama/` (separate pass) |
| 17 | **Parlant** | Missing | Deployment on humboldt | `gitops/apps/parlant/` (separate pass) |
| 18 | **Postgres Cluster for user structured state** | Missing | cnpg `Cluster` CR on humboldt (post #7). Schema: `user_secrets`, `user_preferences`. | `gitops/apps/soma-postgres/` (separate pass) |
| 19 | **Clerk** | External SaaS, keys in Secrets Manager | Add JWKS URL to Envoy `SecurityPolicy` | `K8s-Bootstrap/humboldt/envoy-gateway/security-policy.yaml` (ship with scaffolds) |
| 20 | **NetworkPolicy / CNI swap to Canal** | Flannel (non-enforcing) | Document policies now; swap to **Canal** (Calico + Flannel) later — Cilium is too heavy for RPi | **backlog** |

**Unblocking order**: 1–6, 13 → scaffolds can build, push, and pull images. 7–12 → platform services for runtime. 14 is optional. 15–18 gate runtime functionality.

---

## Target architecture

```
User (browser / Expo) ─▶ Cloudflare (humboldt-external) ─▶ Envoy Gateway (humboldt)
                                                         │  SecurityPolicy: Clerk JWKS verify
                                                         │  Injects X-Soma-User-Id (HMAC-signed)
                                                         ▼
                                    HTTPRoute: /chat, /feedback, /proactive, /wearables/apple/*
                                                         │
                        ┌────────────────────────────────┼────────────────────────────────┐
                        ▼                                ▼                                ▼
                  theo-api                        theo-orchestrator                 apple-ingest
                  (FastAPI)                       (LangGraph state machine)         (Go push receiver)
                  • /chat, /feedback              • classify_intent                 • validates HMAC
                  • thin: auth verify,            • inject_memory (Qdrant)          • writes soma_apple_raw
                    enqueue to orchestrator       • load_user_context (S3)          • pushes VictoriaMetrics
                    via internal RPC,             • dispatch node ──▶ HTTP:
                    stream response                  ├─ nutrition-agent
                                                     ├─ meal-agent
                                                     └─ local coach (for fitness/
                                                        workout/summary) — in-process

                                 theo-worker
                                 (Temporal worker)
                                 • session_loader_wf
                                 • nightly_summarize_wf (Sonnet → S3 profile update)
                                 • weekly_report_wf (Sun 18:00)
                                 • proactive_nudge_wf (condition → Parlant → dispatch)
                                 • prompt_judge_daily_wf (S3 bad chats → Sonnet → Langfuse DRAFT)

Siblings (theo-agents/ monorepo on humboldt, path-filtered CI):
   ├─ nutrition/   FastAPI, soma_meal_logs (DynamoDB), Nutritionix + Bedrock Vision
   └─ meal/        FastAPI, soma_meal_plans (DynamoDB), variety + grocery

Wearables (theo-wearables/ monorepo on nebulosa, Go):
   ├─ whoop/       CronJob (hourly) → soma_whoop_raw + VictoriaMetrics
   ├─ oura/        CronJob (hourly) → soma_oura_raw + VictoriaMetrics
   └─ apple/       Deployment (push receiver) → soma_apple_raw + VictoriaMetrics

Data plane:
   DynamoDB (humboldt IRSA per agent)
     ├─ soma_chat_history       (theo-api writes, theo-worker reads)
     ├─ soma_workout_logs       (theo)
     ├─ soma_sleep_logs         (theo)
     ├─ soma_feedback_events    (theo, CDC source)
     ├─ soma_meal_logs          (nutrition)
     ├─ soma_meal_plans         (meal)
     ├─ soma_whoop_raw          (whoop scraper)
     ├─ soma_oura_raw           (oura scraper)
     └─ soma_apple_raw          (apple ingest)
   Qdrant (humboldt, EBS gp3, theo_memory, 1024-dim)  — semantic search
   Postgres / cnpg (humboldt)                    — user_secrets, user_preferences
   S3 soma-user-data/<user>/*.md                 — canonical profile (written by gated nightly wf)
   S3 soma-feedback/<user>/<trace_id>.json       — bad-chat archive
   VictoriaMetrics (humboldt)                    — wearable time-series
   Loki (humboldt)                               — logs via Alloy

Control plane:
   Langfuse (humboldt)          — traces + scores + prompt registry
   Temporal (humboldt)          — workflows + task queue
   Parlant (humboldt)           — proactive utterance policy
   ArgoCD (humboldt)            — gitops for both clusters
```

---

## Repo 1 — `theo/` (path-filtered monorepo: api + orchestrator + worker)

Same shape as `theo-agents/` — each component is its own top-level directory with own pyproject, Dockerfile, and tests. CI uses `dorny/paths-filter@v3` to detect which component changed and only rebuild/deploy that image. Shared internal code lives in `libs/theo-core/` (local editable install; not published — Theo-only).

```
theo/
├── README.md
├── .gitignore
├── .env.example
├── .github/
│   ├── CODEOWNERS
│   └── workflows/
│       ├── ci.yml                     # paths-filter → matrix python-lint-test per changed component
│       ├── build-and-deploy-dev.yml   # matrix per changed component → docker-build-push-ecr (tag theo-<component>-<sha>) → gitops dev bump
│       └── promote-to-prod.yml        # matrix crane-copy-promote → gitops prod PR → argocd-sync-prod
├── libs/
│   └── theo-core/                 # internal shared (Theo-only; not published)
│       ├── pyproject.toml
│       └── theo_core/
│           ├── config.py          # pydantic-settings; dev=.env, prod=AWS Secrets Manager
│           ├── dynamo.py          # typed accessors: chat_history, workout_logs, sleep_logs, feedback_events
│           ├── postgres.py        # SQLAlchemy session helper for user_secrets, user_preferences (cnpg)
│           ├── s3_context.py      # read/write user_profile.md, goals.md, split.md, patterns.md
│           ├── memory.py          # Qdrant 1024-dim helpers
│           ├── langfuse_client.py # get_prompt, score, create_prompt_version (DRAFT)
│           ├── parlant.py         # thin client
│           ├── internal_http.py   # HMAC-signed agent calls + orchestrator call
│           └── logging.py         # structlog config
├── api/                           # IMAGE 1 — thin chat gateway
│   ├── pyproject.toml             # depends on theo-core (editable) + agent-core (CodeArtifact)
│   ├── Dockerfile                 # CMD: uvicorn api.main:app
│   ├── src/api/
│   │   ├── main.py                # FastAPI factory, middleware
│   │   ├── auth.py                # verify Envoy-injected X-Soma-User-Id HMAC
│   │   └── routes/
│   │       ├── chat.py            # POST /chat → HTTP to orchestrator → stream response
│   │       ├── feedback.py        # POST /feedback → DynamoDB + langfuse.score + S3 bad-chat dump
│   │       ├── proactive.py       # POST /proactive/dispatch (worker callback)
│   │       └── health.py
│   └── tests/
├── orchestrator/                  # IMAGE 2 — LangGraph state machine (internal ClusterIP)
│   ├── pyproject.toml
│   ├── Dockerfile                 # CMD: python -m orchestrator.server
│   ├── src/orchestrator/
│   │   ├── server.py              # tiny HTTP server; POST /run (ConversationState) → response
│   │   ├── graph/
│   │   │   ├── state.py           # ConversationState (from agent_core)
│   │   │   ├── nodes.py           # classify_intent, inject_memory, load_user_context, dispatch, apply_persona
│   │   │   ├── router.py          # route_edge (intent → next node)
│   │   │   └── build.py           # build_graph() → compiled StateGraph
│   │   └── coach/                 # In-process for fitness/coaching/workout/summary
│   │       ├── persona.py         # Langfuse master prompt fetch + TTL cache
│   │       ├── onboarding.py      # hidden-checklist state (PRD §7)
│   │       ├── workout.py
│   │       └── summary.py
│   └── tests/
├── worker/                        # IMAGE 3 — Temporal worker
│   ├── pyproject.toml
│   ├── Dockerfile                 # CMD: python -m worker.main
│   ├── src/worker/
│   │   ├── main.py                # worker entrypoint, task queue: theo-main
│   │   └── workflows/
│   │       ├── session_loader_wf.py       # S3 → ConversationState cache
│   │       ├── nightly_summarize_wf.py    # gated: only runs for users with 24h thumbs-down
│   │       ├── weekly_report_wf.py        # Sun 18:00 local
│   │       ├── daily_brief_wf.py          # on-demand trigger from /chat "daily brief"
│   │       ├── proactive_nudge_wf.py      # condition → Parlant → HTTP
│   │       └── prompt_judge_daily_wf.py   # S3 bad chats → judge → Langfuse DRAFT
│   └── tests/
└── tests/
    └── integration/
        └── test_chat_flow.py      # api → orchestrator (in-process) → mocked Bedrock + nutrition stub
```

**Design points:**
- **Three images, three Deployments** (all in one repo). A prompt change to `coach/persona.py` rebuilds only `orchestrator`; a workflow change rebuilds only `worker`. CI matrix publishes images in parallel.
- **`api` is a thin shell**: Envoy already validated Clerk. `api/auth.py` only verifies the HMAC on Envoy's `X-Soma-User-Id` header. `/chat` forwards to `orchestrator` via HMAC-signed internal HTTP.
- **`orchestrator` hosts LangGraph** and owns all intent routing + the coach (fitness/workouts/summary stay in-process). Exposes `POST /run` on ClusterIP.
- **`worker` is Temporal** — separate image, separate deployment, separate scaling profile. No HTTP listener.
- **Langfuse is wired day one.** `@trace("<node>")` on every model call. `langfuse.score(trace_id, value=±1)` in `/feedback`. Master prompt via `tools/langfuse_client.get_prompt("theo-persona", label="production")` with TTL cache.

---

## Repo 2 — `theo-agents/` (nutrition + meal monorepo + shared lib)

```
theo-agents/
├── README.md
├── .gitignore
├── .github/
│   ├── CODEOWNERS
│   └── workflows/
│       ├── ci.yml                     # paths-filter detects changed agent → uses reusable python-lint-test
│       ├── publish-agent-core.yml     # on lib change: build wheel → CodeArtifact
│       ├── build-and-deploy-dev.yml   # matrix per changed agent → cloudhaven-dev:theo-agents-<agent>-<sha> → gitops dev bump
│       └── promote-to-prod.yml        # crane copy per agent on merge
├── libs/
│   └── agent-core/                # published to CodeArtifact as `agent-core`
│       ├── pyproject.toml         # semver, build backend
│       └── agent_core/
│           ├── config.py          # pydantic-settings base
│           ├── models.py          # ConversationState, Intent, Macros, MealLog, MealPlan, FeedbackEvent, ChatMessage
│           ├── model_client.py    # Bedrock wrapper with inference-profile ARNs + Langfuse @trace
│           ├── memory.py          # Qdrant helpers
│           ├── langfuse.py        # tracer init + @trace decorator
│           ├── logging.py         # structlog config (stdout JSON)
│           ├── auth.py            # verify HMAC from theo-orchestrator
│           ├── runtime.py         # create_app(handlers) — FastAPI factory with /health /run
│           └── testing.py         # pytest fixtures
├── nutrition/
│   ├── pyproject.toml             # depends on agent-core from CodeArtifact
│   ├── Dockerfile                 # linux/arm64
│   ├── src/nutrition/
│   │   ├── app.py                 # /run + /nutrition/daily + /nutrition/recent
│   │   ├── handlers.py            # intent dispatch within /run
│   │   ├── nutritionix.py         # typed API client
│   │   ├── vision.py              # Bedrock Sonnet vision + nutrition-label OCR
│   │   └── persistence.py         # soma_meal_logs
│   └── tests/
└── meal/
    ├── pyproject.toml
    ├── Dockerfile
    ├── src/meal/
    │   ├── app.py                 # /run + /meal/plan + /meal/grocery
    │   ├── handlers.py
    │   ├── planner.py             # Bedrock Sonnet meal_planning task
    │   ├── variety.py             # anti-repetition
    │   ├── grocery.py             # dedup ingredient list
    │   └── persistence.py         # soma_meal_plans
    └── tests/
```

**Design points:**
- **`libs/agent-core/` publishes to CodeArtifact** — theo and theo-agents both `pip install agent-core==<semver>`. Version bumps via conventional-commits → auto bump → auto publish.
- **Nutrition owns photo + label analysis.** Flow: user uploads food photo → Vision extracts portions → Nutritionix → `soma_meal_logs`. Nutrition label photo → Vision OCR → Nutritionix normalized entry → `soma_meal_logs` with `source="label_photo"`.
- **Meal agent pulls recent meals from nutrition** (HMAC-signed cross-agent call allowed via Envoy AuthorizationPolicy).
- **IRSA per agent** — nutrition's pod role can only touch `soma_meal_logs`.

---

## Repo 3 — `theo-wearables/` (Go monorepo)

```
theo-wearables/
├── README.md
├── .github/workflows/
│   ├── ci.yml                     # paths-filter per vendor
│   ├── build-and-deploy-dev.yml   # matrix per changed vendor
│   └── promote-to-prod.yml
├── libs/
│   └── vendor-core/               # Go module
│       ├── go.mod
│       └── pkg/
│           ├── secrets/           # AWS Secrets Manager fetch
│           ├── dynamo/            # typed writers per table
│           ├── vm/                # VictoriaMetrics remote_write
│           └── tracing/           # OTLP → Alloy
├── whoop/
│   ├── go.mod                     # replace directive → ../libs/vendor-core
│   ├── Dockerfile                 # multi-stage, distroless, linux/arm64
│   ├── cmd/scraper/main.go        # OAuth2 refresh → /v1/cycle, /v1/recovery, /v1/sleep
│   ├── internal/client/
│   └── deploy/cronjob.yaml        # referenced by gitops
├── oura/
│   ├── go.mod
│   ├── Dockerfile
│   ├── cmd/scraper/main.go        # Oura API v2: sleep, readiness, activity, workouts
│   ├── internal/client/
│   └── deploy/cronjob.yaml
└── apple/
    ├── go.mod
    ├── Dockerfile
    ├── cmd/receiver/main.go       # HTTP server: POST /wearables/apple
    │                              #   — HMAC from Expo app (shared secret via Clerk session)
    │                              #   — validates HealthKit payload schema
    │                              #   — writes soma_apple_raw + VictoriaMetrics
    └── deploy/deployment.yaml     # behind Envoy HTTPRoute
```

**Design points:**
- **Whoop + Oura = CronJobs on nebulosa** (cheap edge scheduling, hourly). Each writes raw payload to DynamoDB and numeric series to VictoriaMetrics.
- **Apple Health is push-based** because HealthKit cannot be queried from a server. The Expo app reads HealthKit, signs with a per-user HMAC key (issued at onboarding via Clerk), and posts to Envoy. Apple receiver validates, persists, pushes metrics.
- **Shared Go module `vendor-core`** — local replace-directive in dev, versioned via git tag once we stabilize.
- **Arm64 Docker images**, distroless base.

---

## Shared — `Cloudhaven-IDP/Actions` (existing repo, clone locally)

Already exists on GitHub. Clone next to the other repos at project root. Populate with:

```
Actions/
├── README.md
├── .github/workflows/              # reusable workflows (called via `uses: Cloudhaven-IDP/Actions/.github/workflows/<name>.yaml@main`)
│   ├── python-lint-test.yaml       # inputs: python-version, working-directory → ruff + mypy --strict + pytest + coverage
│   ├── go-lint-test.yaml           # inputs: go-version, working-directory → golangci-lint + go test
│   ├── docker-build-push-ecr.yaml  # inputs: dockerfile, image-name, environment → buildx arm64 → OIDC assume → ECR push
│   ├── docker-run-smoke.yaml       # inputs: image → docker run → curl /health → assert 200
│   ├── gitops-bump-overlay.yaml    # inputs: overlay-path, image-ref → yq bump → commit via ci-bot App
│   ├── crane-copy-promote.yaml     # inputs: src-image, dst-image → crane copy
│   └── argocd-sync-prod.yaml       # inputs: app-name → argocd CLI with SSO → sync + wait-for-healthy
└── actions/                        # composite actions
    ├── aws-oidc-assume/action.yaml # OIDC → assume gha-deployer role
    ├── codeartifact-login/action.yaml  # fetch CodeArtifact auth token, pip configure
    ├── ecr-login/action.yaml
    └── preview-url/action.yaml     # computes `theo-pr-<N>.internal.cloudhaven.work`, comments on PR
```

All three new repos (theo, theo-agents, theo-wearables) reference these via `uses:`. No copy-paste CI.

---

## Cross-cutting

### Envoy Gateway + Clerk

- `K8s-Bootstrap/humboldt/envoy-gateway/security-policy.yaml` — `SecurityPolicy` with Clerk JWKS URL (fetched from AWS Secrets Manager via External Secrets). Matches HTTPRoutes for theo-api + apple-ingest. Drops requests without valid JWT (except `/health`).
- After verification, Envoy injects `X-Soma-User-Id: <sub>` with an HMAC (short-lived secret, rotated via External Secrets).
- theo-api's `auth.py` verifies the HMAC, not the Clerk JWT. Clerk logic lives at the edge.
- Nutrition + meal + orchestrator never see either header — they trust the signed ConversationState from theo-orchestrator (HMAC).
- Dev mode: Envoy security policy bypassed for preview namespaces; `X-User-Id` header accepted directly.

### Langfuse — traces, scores, prompt registry

- `agent_core.langfuse` initializes tracer at service startup.
- `@trace("<task_name>")` on every `model_client.invoke` call — one trace tree per user turn.
- `POST /feedback` on theo-api:
  1. Write `FeedbackEvent` to `soma_feedback_events` (CDC source for future ML).
  2. `langfuse.score(trace_id, name="user_feedback", value=+1|-1, comment=note)`.
  3. On thumbs-down only: dump full conversation + trace_id to `s3://soma-feedback/<user_id>/<ISO>-<trace_id>.json`.
- Master prompt: `tools/langfuse_client.get_prompt("theo-persona", label="production")` at orchestrator startup; TTL cache 5min so promotions propagate without redeploy.
- Unit tests run with `LANGFUSE_ENABLED=false`.

### Feedback → prompt mutation loop (daily)

- `prompt_judge_daily_wf` (Temporal cron, 02:00 UTC):
  1. List objects in `s3://soma-feedback/*/` with `LastModified` in last 24h.
  2. For each bad-chat: load, fetch Langfuse trace, capture master prompt version used.
  3. Batch through Sonnet as judge: "Given this bad interaction + the master prompt, propose minimal prompt edits. Return as diff."
  4. Aggregate edits into a single DRAFT prompt version: `langfuse.create_prompt(name="theo-persona", prompt=<new_body>, labels=["draft", f"judge-{date}"], commit_message="Auto-proposed from bad chats YYYY-MM-DD")`.
  5. Post summary to a Slack/Discord webhook for human review.
- **Never** auto-promotes to `production`. Human clicks promote in Langfuse UI. Prompts can also be edited freely in the Langfuse UI at any time independent of this workflow — LLM-as-judge is one proposal source, not the only path to prompt change.
- Scaffold ships the workflow **signature + docstring**, real Sonnet call added later.

### Memory extraction (not mem0)

- `nightly_summarize_wf` (Temporal cron, 03:00 UTC):
  1. Query `soma_feedback_events` for thumbs-down rows in the last 24h → produces the set of user_ids to process. **If no user had a thumbs-down, the workflow does nothing.**
  2. For each qualifying user: pull last 24h of `soma_chat_history`, pull existing `s3://soma-user-data/<user>/{user_profile,patterns}.md`.
  3. Call Sonnet (big model) with a summarization prompt: "Given the existing profile and these new messages, propose diffs to add facts / update preferences / note patterns."
  4. Write updated markdown back to S3 + append change log to `patterns.md`.
- **Cost-gated**: we only spend Sonnet tokens on users we have reason to believe need correction. Ungated every-user-every-night is deferred until we know the cost curve.
- Scaffold ships signature + docstring; real Sonnet call in next pass.
- **Zero per-interaction LLM extraction cost** (which is the mem0 cost model we reject).

### Temporal

- Task queue: `theo-main`.
- Worker image (`theo-worker`) subscribes to the queue.
- Workflows ship as signatures + docstrings this pass:
  - `session_loader_wf(user_id, session_id)` — S3 → cache, query handler
  - `nightly_summarize_wf(user_id)` — cron, Sonnet → S3 profile
  - `weekly_report_wf(user_id)` — Sun 18:00 local
  - `proactive_nudge_wf(user_id, utterance_type, payload)` — condition → Parlant → HTTP POST to theo-api `/proactive/dispatch`
  - `prompt_judge_daily_wf()` — above
- Temporal server itself = separate pass (gitops Helm chart).

### Parlant

- Thin client `tools/parlant.py`: `evaluate(user_id, utterance_type, payload) → {allow, reason, cooldown_s}`.
- Called exclusively from `proactive_nudge_wf`.
- Parlant server deploy = separate pass.

### NetworkPolicies (Flannel caveat)

- Policy manifests still ship in `gitops/theo/base/networkpolicy.yaml`:
  - default-deny in each namespace
  - allow theo-api → theo-orchestrator
  - allow theo-orchestrator → {nutrition, meal}
  - allow meal → nutrition (for variety lookup)
  - allow theo-worker → {temporal, langfuse, s3 VPC endpoint}
  - allow Alloy DaemonSet → everywhere on metrics port
- **Flannel does not enforce these.** Until CNI swap (backlog):
  - App-layer: HMAC on every internal call (reject unsigned/invalid)
  - Envoy `AuthorizationPolicy` on east-west traffic through the gateway
- Policy YAML is audit-trail-ready for when we switch to **Canal** (Calico policy + Flannel overlay — RPi-friendly; Cilium is ruled out for edge resource reasons).

### Logging — Grafana Alloy (replaces Fluentd/Promtail)

- **Why Alloy**: Grafana unified their log + metric + trace agents into Alloy in 2024. Replaces Promtail (deprecated), consolidates with OTEL Collector. Current Grafana recommendation for new deployments.
- Alloy DaemonSet on both clusters: scrapes pod stdout, attaches k8s labels, ships to Loki (humboldt) via Tailscale.
- App side: `structlog` JSON renderer, bound `request_id`/`user_id`/`trace_id`/`service`/`version`. No explicit shipper code in agents — just stdout.
- Sensitive redaction at structlog processor: drop raw Clerk tokens, HMAC signatures, PII email/phone.

### CI/CD

**Image flow:**
```
PR opens on theo / theo-agents / theo-wearables:
  Uses Cloudhaven-IDP/Actions reusable workflows:
    ├─ python-lint-test (or go-lint-test)
    ├─ docker-build-push-ecr → cloudhaven-dev:<project>-<component>-<sha>
    │   (e.g. cloudhaven-dev:theo-api-abc123, cloudhaven-dev:theo-agents-nutrition-abc123)
    ├─ docker-run-smoke → curl /health on the freshly built image
    ├─ gitops-bump-overlay → bumps overlays/dev/pr-<N> via ci-bot App (dev autosync picks it up)
    └─ preview-url → PR comment: theo-pr-<N>.internal.cloudhaven.work + expires-at

PR merges:
    ├─ crane-copy-promote → cloudhaven-prod:<project>-<component>-<sha>
    ├─ gitops-bump-overlay → prod (via ci-bot PR to gitops, NOT direct commit)
    └─ argocd-sync-prod → argocd CLI with SSO, waits for HEALTHY
```

**No prod autosync**: prod ArgoCD apps have `syncPolicy: {}`. CI is the only sync trigger.

**Preview TTL**: overlay carries `annotations.ttl.theo.io/expires-at`. `preview-sweeper` CronJob on humboldt deletes namespaces past expiration + opens gitops PR to prune.

**ECR lifecycle** (Terraform, 2 repos):
- `cloudhaven-dev`: retain 50 most-recent tags (shared across all projects/components), expire untagged after 7d
- `cloudhaven-prod`: retain 500, no auto-expiration

**ECR pull**:
- **humboldt**: node IAM has `AmazonEC2ContainerRegistryReadOnly`. Kubelet credential provider handles pulls. No `imagePullSecrets` on pods.
- **nebulosa**: `ecr-credential-refresher` CronJob (runs every 8h) assumes an IAM role scoped to ECR read, mints a 12h token, and updates a `regcred` docker-registry Secret in each namespace that pulls ECR images. Namespaces reference it via `imagePullSecrets`.

**CI bot**: `ci-bot` GitHub App installed on Cloudhaven-IDP org. Private key in Secrets Manager. CI uses App token (not PAT) to commit to gitops. Bypasses branch ruleset only for the automated prod PR + dev overlay commits.

---

## Salvage list from `backend/`

| From | To | Notes |
|---|---|---|
| `backend/core/models.py` | `theo-agents/libs/agent-core/agent_core/models.py` | Published via CodeArtifact, imported by both theo and theo-agents. Drop `AgentName.FITNESS`. Add `ChatMessage` for chat_history. |
| `backend/core/model_router.py` | `agent_core/model_client.py` | Env-driven inference-profile ARNs, Langfuse `@trace` decorator. |
| `backend/agents/nutrition.py` | `theo-agents/nutrition/src/nutrition/` (split: `handlers.py`, `nutritionix.py`, `vision.py`, `persistence.py`) | Add Pydantic models for Nutritionix responses. Add label-OCR path in `vision.py`. |
| `backend/tools/memory.py` | `agent_core/memory.py` + `theo/libs/theo-core/theo_core/memory.py` | 768→1024 dim, `soma_memory`→`theo_memory`, embeddings via Ollama bge-m3. |
| `backend/api/auth.py` | replaced by Envoy `SecurityPolicy` + `theo/api/src/api/auth.py` (HMAC verify only) | Clerk verification moves to Envoy. |
| `backend/routes.yaml` | **Discarded** | LangGraph edges replace YAML. |
| `backend/agents/orchestrator.py` | **Discarded** | Redefined in `theo/orchestrator/src/orchestrator/graph/`. |

`backend/` gets deleted once scaffolds are green (last step, not this pass).

---

## What we're NOT building this pass

- **Langfuse, Temporal, KEDA, Loki, VictoriaMetrics, Grafana, Qdrant (humboldt/EBS), Ollama, Parlant deploys** — listed in pre-deploy infra checklist; go into `K8s-Bootstrap/humboldt/` (platform infra) or `gitops/apps/` (user workloads) per the checklist "Where" column.
- **cnpg on humboldt**
- **Soma Postgres `Cluster` CR** (tables: `user_secrets`, `user_preferences`) — depends on cnpg
- **ecr-credential-refresher CronJob** on nebulosa
- **cert-manager + Cloudflare DNS-01 ClusterIssuer** on humboldt
- **ARC (self-hosted runners)** — backlog
- **CNI swap to Canal** (Flannel → Calico policy + Flannel overlay) — backlog
- **Real workflow bodies** — signatures + docstrings only
- **Real coach inference** — persona loads master prompt but coach nodes return stubs
- **Real Sonnet summarize / judge logic** — Temporal activities are scaffolds
- **CDC pipeline infra** (Kinesis/Kafka) — model ships, sink is TODO
- **Preview-sweeper CronJob** — gitops
- **React Native `svc-frontend/`**
- **Landing page**
- **Deleting `backend/`** — last step

---

## Verification

### Per-repo local
```bash
# theo/ — each component is its own buildable subdir
aws codeartifact login --tool pip --domain cloudhaven --repository python-private
cd theo/api
python -m venv .venv && source .venv/bin/activate
pip install -e ../libs/theo-core -r requirements.txt -r requirements-dev.txt
ruff check . && mypy src --strict && pytest -q
docker build -t theo-api:dev .
docker run --rm -p 8000:8000 --env-file ../.env.example theo-api:dev &
sleep 2 && curl -f localhost:8000/health
# repeat for theo/orchestrator and theo/worker

# theo-agents/
cd theo-agents/nutrition
pip install -r requirements.txt  # agent-core from CodeArtifact
pytest -q
docker build -t theo-nutrition:dev . && docker run --rm -p 8001:8000 theo-nutrition:dev &
sleep 2 && curl -f localhost:8001/health

# theo-wearables/whoop
cd theo-wearables/whoop
go test ./...
docker build -t theo-whoop:dev . && docker run --rm theo-whoop:dev --dry-run
```

### Graph smoke (mocked Bedrock)
```bash
cd theo
pytest tests/integration/test_chat_flow.py -v
# Expect: /chat "had 2 eggs and toast" →
#   classify_intent=log_food → inject_memory (empty) →
#   orchestrator dispatches → nutrition stub 200 → macro summary in response
```

### CI green (per repo)
- ruff + mypy + pytest + docker build **+ docker run smoke** (curl /health) + ECR push — all via reusable workflows from `Cloudhaven-IDP/Actions`.
- Path-filtered: theo-agents PR touching only `nutrition/` runs nutrition jobs only.
- Merge: crane copy + gitops prod PR opened by ci-bot + argocd sync runs to HEALTHY.

### End-to-end (once dev preview deploys)
```bash
# Via Tailscale to preview namespace
curl https://theo-pr-42.internal.cloudhaven.work/health
curl -X POST https://theo-pr-42.internal.cloudhaven.work/chat \
  -H "Authorization: Bearer <clerk-jwt>" \
  -H "Content-Type: application/json" \
  -d '{"message":"had 2 eggs and toast"}'
# Expect: {"response": "Logged: ...", "intent": "log_food", ...}
```

---

## Backlog (captured for later passes)

1. **Deploy ARC** (self-hosted GHA runners on humboldt) — enables in-cluster CI, CodeArtifact via IRSA, faster arm64 builds.
2. **CNI swap to Canal** (Flannel → Calico policy + Flannel overlay) — enables NetworkPolicy enforcement while staying RPi-friendly.
3. **Ungate the nightly summarize workflow** once we have cost data — today it only runs for thumbs-downed users; eventually every active user should get periodic summarization.
4. **Semantic nutrition cache** (pgvector or Qdrant on normalized food descriptions) — revisit when Nutritionix latency/cost hurts.
5. **LiteLLM swap** — drop-in replacement for `model_client.invoke` when we want cross-provider.
6. **Landing page** — separate design/frontend session.
7. **`svc-frontend/`** (Expo React Native + Expo Web via react-native-web) — one codebase, three targets (iOS, Android, browser).
8. **Deploy Parlant server**.
9. **CDC pipeline** (Kinesis → Lambda → analytics bucket) for FeedbackEvents and chat_history.
10. **Delete `backend/`** once scaffolds green and real logic migrated.

---

## Outstanding decisions (flag before coding)

1. **Apple Health HMAC secret distribution** — per-user key issued at onboarding (Clerk session), stored in **Postgres `user_secrets`** table (cnpg, not DynamoDB — matches the "structured user state goes to Postgres" rule).
2. **Temporal namespace on humboldt** — `soma` (current) vs `theo` (rename)? Plan defaults to **`soma`** (matches existing config).
3. **CodeArtifact versioning** — semver via conventional-commits + GH Actions (auto) vs manual tag? Plan defaults to **auto semver**.
4. **Wildcard cert** for `*.internal.cloudhaven.work` via cert-manager + Cloudflare DNS-01 solver. Cloudflare zone + API token already available (DNS is Cloudflare-managed). Confirm token scope when wiring the `ClusterIssuer`.
5. **Local internal monorepo lib (`theo/libs/theo-core`)** — stays internal (editable install, not published). If we find we want to share Theo internals with another future consumer, revisit; today CodeArtifact is for `agent-core` only.
