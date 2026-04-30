# afolabi-next — personal site + Ask-the-Desk frontend

## Context

Afolabi needs a public-facing site geared at recruiters: landing page, blog (Filed Reports), résumé, and an "Ask the Desk" chat surface where visitors can query an AI understudy about his background. The existing `blog/` Hugo site served the blog purpose but is static-only — the chat UI needs a runtime, and the editorial direction wants more layout control than PaperMod gives.

`afolabi-next` is the replacement. It does NOT own the agent — that's a separate service (`afolabi-desk`, upcoming). This doc covers the frontend only; the agent gets its own plan when we start scaffolding it.

### Author rules (apply while building)

- **No unnecessary comments.** Same rule as the theo siblings scaffold. No explanatory JSX comments, no docstrings that restate signatures, no "// removed X" breadcrumbs.
- **Content in `content/`, not in components.** Blog posts as Markdown, experience as YAML, tooling inventory as a TS data file that's still data-shaped (not markup). Anyone editing a bullet should not touch React.
- **Ops-log entry on every meaningful scaffolding session.** Same rule as `2026-04-20-theo-siblings-scaffold-v3.md`.

---

## Goals

1. Replace the Hugo blog — migrate all posts, keep URLs stable where possible.
2. Give recruiters a résumé surface — both a downloadable PDF and an on-page "Experience" section.
3. Give curious visitors a conversational front door — Ask the Desk, streaming, non-embarrassing.
4. Deploy on humboldt alongside the rest of the Cloudhaven stack. Public-facing via Envoy Gateway + Cloudflare DNS.
5. Content updates (posts, experience, tooling) do not require a code change by the author.

## Non-goals

- Server-rendered personalization, user accounts, analytics dashboards.
- Building the agent inside this repo — `/api/ask` is a proxy, nothing more.
- A CMS. Markdown + YAML in git is the CMS.
- Multi-language. English only.

---

## Decisions

| Area | Decision |
|---|---|
| Framework | **Next.js 16** App Router + Turbopack + React 19 + Tailwind v4. CSS-first Tailwind config in `globals.css` — no `tailwind.config.ts`. |
| Agent architecture | **Separate service** (`afolabi-desk`, next session). `afolabi-next/src/app/api/ask/route.ts` is a thin proxy, no model SDKs, no system prompts, no RAG. |
| Proxy contract | `POST $AGENT_URL/ask` with `{ messages: [{role, content}] }` → `200 OK`, `text/plain`, streamed body. Optional `Authorization: Bearer $AGENT_TOKEN`. |
| Blog storage | **Markdown in `content/posts/`**, rendered via `next-mdx-remote/rsc` + `remark-gfm` + `rehype-slug` + `rehype-autolink-headings`. Frontmatter parsed by `gray-matter`. |
| Experience storage | **YAML in `content/experience.yaml`**, rendered on `/resume`. Structure: `summary`, `roles[]`, `certifications[]`, `education[]`. |
| Résumé PDF host | **Cloudflare R2**, custom domain `resume.cloudhaven.work`. URL passed via `RESUME_URL` env var. Page shows a disabled "PDF pending" state when unset. |
| Tooling inventory | **TS data file** `src/lib/tools.ts` — 7 categories, each with tools + optional one-line glosses. Rendered as an editorial list on the landing page, NOT a generic logo grid. |
| Design direction | Editorial zine. Fraunces display (variable, SOFT + WONK axes pushed) + Newsreader body + JetBrains Mono for technical marginalia. Warm cream paper (`#EFE6D3`), ink-dark text, burnt copper accent (`#B8491F`). SVG paper grain + asymmetric radial gradients. |
| Brand vs repo name | Brand: "Cloudhaven · Field Notes" (on every masthead). Repo / deployment / ArgoCD app: **`afolabi-next`**. Separate concerns. |
| Config | `AGENT_URL`, `AGENT_TOKEN` (optional), `RESUME_URL`. Synced from AWS Parameter Store into the pod via ESO. No `.env.example`. |
| Container | **Next.js `output: "standalone"`**, three-stage `Dockerfile` on `node:22-alpine`. Non-root user, explicit `COPY content/` in the runner stage (standalone tracing doesn't include non-code assets by default). |
| Image registry | **ECR `cloudhaven-dev` / `cloudhaven-prod`** with tag namespace `<project>-<component>-<sha>` (`afolabi-next-<sha>`). Same lifecycle as Theo images. |
| CI | **`Cloudhaven-IDP/Actions`** reusable workflows — add a `nextjs-ci/` composite (lint + typecheck + build) alongside the existing `python-ci/`. Reuse `docker-build-push-ecr` + `gitops-bump-overlay` + `crane-copy-promote` unchanged. |
| CD | Argo CD on humboldt. Dev autosync, prod via `argocd app sync` from the gitops-side workflow. |
| Ingress | **Envoy Gateway (humboldt) — public zone**. `HTTPRoute` for `afolabi.cloudhaven.work`. Separate cert path from the `*.internal.cloudhaven.work` wildcard used by Theo; needs its own Cloudflare DNS-01 certificate. |
| Rate-limit | Cloudflare Turnstile + per-IP token bucket in the `/api/ask` proxy. Lands before the site goes fully public — otherwise the first scraper runs up a Bedrock bill on the agent. |

---

## Architecture

```
Visitor browser
     │
     ├── GET /, /posts, /posts/:slug, /resume  →  static HTML (prerendered by Next)
     │
     └── POST /api/ask (streaming)
                │
                ▼
      afolabi-next pod (humboldt)
                │ POST $AGENT_URL/ask
                ▼
      afolabi-desk pod (humboldt, separate repo)
                │  - bio system prompt (Langfuse)
                │  - RAG over content/posts + experience
                │  - Bedrock Claude Sonnet
                ▼
           streamed plain text
                │
                ◀ forwarded back through the proxy
```

Config flow:

```
AWS Parameter Store (/cloudhaven/afolabi-next/agent_url, …)
     │ ESO sync (pull)
     ▼
K8s Secret (ns: afolabi-next)
     │ envFrom
     ▼
afolabi-next pod env (AGENT_URL, AGENT_TOKEN, RESUME_URL)
```

Content flow:

```
  content/posts/*.md         → getAllPosts() → /posts + /posts/:slug (SSG)
  content/experience.yaml    → loadExperience() → /resume (SSG)
  src/lib/tools.ts           → TOOLING → landing page (SSG)
```

---

## Phases

### P0 — Local scaffold ✅ (this session, 2026-04-21)

- Next.js 16 + TS + Tailwind v4 + App Router + src dir + ESLint. `npm install` motion (trimmed later), gray-matter, next-mdx-remote, remark-gfm, rehype-slug, rehype-autolink-headings, lucide-react, clsx, reading-time, js-yaml.
- Design system in `globals.css` (colors, fonts, rule variants, drop-cap, prose-editorial, caret animation, rise/draw keyframes).
- Components: `Masthead`, `Footer`, `Rule` / `RuleWithMark`, `Ornament` (engraved SVG printer's mark).
- Pages: `/`, `/posts`, `/posts/[slug]`, `/ask`, `/resume`, `/api/ask` (proxy).
- Blog migration from `blog/content/posts/` (non-destructive copy, normalised filenames, dropped `hello-hugo.md`).
- `experience.yaml` stub with example roles / certifications / education.
- `Dockerfile` + `.dockerignore` + `output: "standalone"` + `outputFileTracingIncludes`.
- README reflecting config-via-ESO, not `.env`.

Outcome: `npm run build` green, all 5 posts prerender as SSG, dev server live, `/api/ask` streams placeholder until agent exists.

### P1 — Content population

Not a me task — the author:
- Populates `content/experience.yaml` with real roles, dates, bullets, stack.
- Uploads `afolabi-fajobi-resume.pdf` to R2 (Phase P2).
- Reviews / rewrites landing copy (the bio paragraph, tooling-inventory glosses) for voice.

### P2 — Resume PDF host

1. Create R2 bucket (`afolabi-resume` or similar, zero egress). Cloudflare dashboard is fine for the first one; Terraform under `Infrastructure/platform/shared/r2/` if we expect more.
2. Bind custom domain `resume.cloudhaven.work` via Cloudflare → R2 public bucket binding. DNS record is a CNAME.
3. Upload PDF. Set `RESUME_URL=https://resume.cloudhaven.work/afolabi-fajobi-resume.pdf`.
4. Once set, the Download PDF button on `/resume` becomes active.

PDF updates post-P2 require only a re-upload — no redeploy.

### P3 — Build + publish pipeline

1. Add `nextjs-ci/` composite to `Cloudhaven-IDP/Actions`: npm ci → lint → typecheck → build. Mirrors `python-ci/` shape.
2. `afolabi-next/.github/workflows/ci.yaml` — calls the composite on PR + push.
3. `afolabi-next/.github/workflows/build-and-deploy-dev.yaml` — triggers on PR. `docker-build-push-ecr` with image name `afolabi-next-<sha>`, tag prefix `afolabi-next-` in `cloudhaven-dev`. `gitops-bump-overlay` targets `overlays/dev/pr-<N>/afolabi-next/kustomization.yaml`.
4. `afolabi-next/.github/workflows/promote-to-prod.yaml` — on push to main. `crane-copy-promote` dev→prod. `gitops-bump-overlay` opens a PR for the prod overlay.
5. Multi-arch build (amd64 + arm64) via buildx — same pattern as Theo. QEMU emulation is OK on hosted runners until ARC lands.

### P4 — Deployment (gitops + k8s)

1. `gitops/apps/afolabi-next/base/` — Deployment (1 replica, resource requests: 80m cpu / 96Mi memory, limits 500m / 256Mi — Next's standalone idle is ~60 MB RSS), Service (ClusterIP:3000), ConfigMap (non-secret config), ExternalSecret (secret config from SSM).
2. `gitops/apps/afolabi-next/overlays/dev/` + `overlays/prod/` — environment-specific `kustomization.yaml` with image tag overlay (updated by CI) and per-env ConfigMap patches.
3. Envoy Gateway `HTTPRoute` on the humboldt public listener for `afolabi.cloudhaven.work`. Needs a public-zone Certificate from the existing cert-manager Cloudflare DNS-01 ClusterIssuer — scoped to `afolabi.cloudhaven.work` specifically, not the internal wildcard.
4. Cloudflare DNS: `afolabi.cloudhaven.work` → the humboldt gateway's public IP (through the existing gateway infra, whatever that shape is).
5. ArgoCD `Application` registered in the gitops app-of-apps.

### P5 — Agent integration (blocks on afolabi-desk existing)

1. `afolabi-desk` ships (separate session / separate plan).
2. `afolabi-desk.prod.svc.cluster.local:8080` reachable from `afolabi-next`.
3. SSM `/cloudhaven/afolabi-next/agent_url` populated. ESO picks it up on the next sync.
4. `/ask` stops streaming the placeholder and starts streaming real answers.

### P6 — Rate-limit + abuse control

- Cloudflare Turnstile widget on the `/ask` page. Token verified server-side in `/api/ask` before proxying.
- Per-IP token bucket in the proxy (in-memory Redis or Upstash; could also be a sidecar).
- Short circuit: if the `X-Turnstile-Response` is missing or stale, return 403 before the upstream fetch.

### P7 — Blog deprecation

- `blog/` Hugo site stays live until `afolabi.cloudhaven.work` has been in prod for ~2 weeks and all legacy inbound links are redirected.
- Permanent redirect from the old domain to `afolabi.cloudhaven.work` at the Cloudflare edge.
- `blog/` directory archived (not deleted immediately — useful reference while porting any missed content).

---

## Rules

- Frontend repo owns no agent logic. `/api/ask` is a proxy. Forever.
- Content the author might tune (posts, experience, tooling glosses) lives in `content/` as Markdown / YAML / TS data.
- No `.env.example` files. Env-var names documented in README, production config syncs from SSM via ESO.
- Public-facing downloads → R2 over S3 unless there's a specific reason.
- UI brand and deployment name are separate concerns — editorial brand, pragmatic repo/deployment name.
- `output: "standalone"` requires both `outputFileTracingIncludes` config AND explicit `COPY content/` in the Dockerfile runner stage.
- No descriptive comments. Including JSX section labels (`{/* title row */}`).
- Proxy routes stay proxies: no retries, no caching, no sanitization, no prompt injection guarding. All of that is the agent's job.

---

## Open questions

1. **Reuse `agent-core` in `afolabi-desk`?** Gets us task-keyed `model_client.invoke`, Langfuse TTL cache, and the image translation between Anthropic / OpenAI / Gemini message shapes for free. Downside: a second consumer of `agent-core` shifts it from "Theo-internal shared lib" to "generic internal SDK" — worth the API stability commitment?
2. **Qdrant collection for the desk agent.** Separate `desk_bio` on humboldt, or carve a namespace inside `theo_memory`? Leaning separate — recruiter-facing content is orthogonal to user-specific fitness memory, and a one-dim-mismatch mistake would be harder to debug in a shared collection.
3. **First public app — cert + DNS path.** The existing cert-manager ClusterIssuer issues `*.internal.cloudhaven.work` via Cloudflare DNS-01. `afolabi.cloudhaven.work` is a separate zone (public root, not internal subdomain). Same ClusterIssuer can issue for it, but we need to confirm the Cloudflare API token scope covers both zones.
4. **Rate-limit storage.** In-memory per-pod = reset on rollout, useless under horizontal scaling. Upstash Redis is $0 at our volume. Or a Cloudflare Worker in front of `/api/ask` that does the per-IP bucket before the request reaches humboldt at all. Leaning Cloudflare Worker — moves rate-limit cost and logic to the edge.
5. **`afolabi-next` namespace.** Own namespace (`afolabi-next`) or shared with the agent in a `portfolio` namespace? Leaning separate — easier to apply NetworkPolicy later, cleaner RBAC, and the coupling is only `AGENT_URL` which is a cross-namespace Service name.
