# Passwordless Postgres platform on Kubernetes — implementation plan

**Status:** ready for review, not yet implemented
**Tracking issues:** TBD (open under `Cloudhaven-IDP/gitops` for the chart, `Cloudhaven-IDP/Infrastructure` for the TF module, `Cloudhaven-IDP/K8s-Bootstrap` for the operator install)
**Source context:** `~/Downloads/postgres-platform-context.md` (architecture brain-dump being turned into a blog series; this file is the actionable cut)

---

## Context

Theo, theo-agents, Temporal, and Langfuse all need Postgres. Today:

- **Nebulosa** (RPi, fresh k3s — old cluster wiped) has nothing.
- **humboldt** (Talos on EC2, single AZ today) has nothing.
- No production target named — could end up on Talos forever, on a managed K8s offering, or somewhere else. The chart should not assume.

Rather than land a one-off CNPG install on humboldt and refactor it later, design **one platform Helm chart** that runs on any cluster, keyed on storage class — not cluster name. Same chart, two values files today (`values.local-path.yaml`, `values.ebs-gp3.yaml`), more added as new storage classes appear.

The other shift — **passwordless for apps, password-in-SSM for the master**. CNPG runs its own CA; cert-manager mints per-app client certs whose CN equals the Postgres role; `pg_hba` is `hostssl ... cert clientcert=verify-full`. Apps connect with `sslmode=verify-full` — zero passwords in any app code or app-side Secret. The CNPG-generated superuser password is pushed once via ESO `PushSecret` to AWS SSM Parameter Store as a SecureString; Terraform reads it via OIDC for role/grant management. Single shared admin credential, never in committed code, never in app-side secrets — but no cert gymnastics for the one connection that needs admin rights. Certs earn their keep when there are many consumers; the master is one credential, password is fine.

This plan covers the platform. Per-tenant Cluster CRs (one for soma, one for Temporal, one for Langfuse) are downstream consumers of the same chart — covered in §7.

---

## Options considered

| | Approach | Verdict |
|---|---|---|
| A | Managed Postgres (RDS or equivalent) | Rejected. Defeats the platform-engineering goal, no portability with Nebulosa, ongoing $/mo we don't need. |
| B | Plain CNPG with password-in-secret per app | Rejected. Secret sprawl, manual rotation, no clean human-access story. Same operator we'd use for the passwordless path — pay the small upfront cost to do it right. |
| C | CNPG with cert-auth for apps + SSM-backed superuser password for Terraform (this plan) | **Chosen.** Cert-auth where it pays off (many app consumers, auto-rotation), SSM password for the one admin connection. Pragmatic split. |
| C-strict | Same as C but Terraform also uses cert-auth (terraform_admin cert + pg_ident map) | Considered, rejected. Cert plumbing for one TF connection — bootstrap Job, ESO cert push, runner-side cert materialization — buys very little vs SSM-backed password. Revisit if multiple admin clients appear. |
| D | Vault as the secret backend instead of AWS SSM | Defer. SSM SecureString is good enough while we're single-cloud and free. Vault becomes interesting only with multi-cloud or dynamic creds. The chart's `secretBackend.type` toggle (`ssm` \| `secretsmanager` \| `vault`) leaves the door open. |
| E | AWS Secrets Manager as the default backend | Supported but not default. SSM is free at our volumes, SM costs per secret + per API call. Chart supports both via the `secretBackend.type` toggle. |

---

## Design

### 1. Cluster prerequisites (per cluster, install once)

Operator/CRD-level installs, owned by `K8s-Bootstrap/<cluster>/`, ArgoCD-managed. None tenant-specific.

| Prereq | humboldt | Nebulosa | Notes |
|---|---|---|---|
| **CNPG operator** | new — `K8s-Bootstrap/humboldt/cnpg-operator/` | new — `K8s-Bootstrap/nebulosa/cnpg-operator/` | Helm chart `cnpg/cloudnative-pg`, pinned version |
| **cert-manager** | new — `K8s-Bootstrap/humboldt/cert-manager/` | new — `K8s-Bootstrap/nebulosa/cert-manager/` | Cluster-scoped install. DNS-01 for the public `*.internal.cloudhaven.work` cert is a separate concern (covered in v3 plan). |
| **External-Secrets Operator** | already present (`K8s-Bootstrap/humboldt/external-secrets/`) | new — `K8s-Bootstrap/nebulosa/external-secrets/` | Needs IRSA on humboldt for `ssm:PutParameter` on `/soma/postgres/*`. Nebulosa uses a static IAM user secret (no IRSA on RPi) — only if Nebulosa actually pushes to SSM, which it shouldn't (see §7). |
| **EBS CSI driver** | already present (`K8s-Bootstrap/humboldt/ebs-csi-driver/`) | n/a (uses `local-path`) | |
| **StorageClass `ebs-gp3`** | new manifest under `ebs-csi-driver/` with `WaitForFirstConsumer` | n/a — k3s ships `local-path` as default | Mandatory `WaitForFirstConsumer` on multi-AZ for replica/PVC AZ co-location. Single-AZ today, but no reason to set it wrong. |

**Rule:** operator installs and StorageClasses are cluster-wide infra — they live in `K8s-Bootstrap/<cluster>/`, never in `gitops/apps/`. Matches the v3 split.

### 2. The `postgres-platform` Helm chart

Single chart, lives in **`gitops/charts/postgres-platform/`** (gitops repo, alongside the ArgoCD apps that consume it). One `helm install` brings up everything below for one tenant. Each tenant (soma, temporal, langfuse) gets its own ArgoCD Application that points at this chart with a different `values.yaml`.

```
gitops/charts/postgres-platform/
├── Chart.yaml
├── values.yaml                       # documented defaults; tenants override
├── values.local-path.yaml            # k3s/RPi: 1 instance, no IRSA, secretBackend.type=none
├── values.ebs-gp3.yaml               # any cluster with ebs-gp3: 3 instances, IRSA, SSM
└── templates/
    ├── cluster.yaml                  # CNPG Cluster CR + pg_hba
    ├── issuer.yaml                   # cert-manager ClusterIssuer (CNPG self-signed CA)
    ├── certificate-app.yaml          # range over .Values.appRoles → one Certificate each
    ├── coredns-rewrite-job.yaml      # post-install Job that patches kube-system CoreDNS ConfigMap
    ├── secretstore.yaml              # ESO ClusterSecretStore → SSM (or SM, or Vault)
    ├── pushsecret-superuser.yaml     # ESO PushSecret: CNPG superuser → SSM SecureString
    ├── serviceaccount.yaml           # SA with IRSA annotation for ESO push
    ├── podmonitor.yaml               # CNPG metrics → VictoriaMetrics
    └── _helpers.tpl
```

**Rule (from memory — `feedback_helm_chart_design.md`):** the chart is generic. No `soma`, `theo`, or `nutrition` strings hardcoded — every tenant identifier comes from `values.yaml`. The `appRoles` list, `clusterName`, `dnsAlias`, `secretBackend.*` are all values.

**Why values files keyed on storage class, not cluster name:** any new cluster that uses `ebs-gp3` reuses the same overrides — no `values.<cluster>.yaml` proliferation. Adding a new storage backend (e.g. `efs-csi`, `longhorn`) = one new values file, every cluster running that backend benefits.

### 3. CNPG Cluster CR

```yaml
spec:
  instances: {{ .Values.postgres.instances }}      # 1 on local-path, 3 on ebs-gp3
  storage:
    storageClass: {{ .Values.postgres.storage.storageClass }}
    size: {{ .Values.postgres.storage.size }}
  postgresql:
    pg_hba:
      - hostssl all postgres all md5                              # superuser, password from CNPG-generated Secret
      {{- range .Values.appRoles }}
      - hostssl {{ .database }} {{ .pgRole }} all cert clientcert=verify-full
      {{- end }}
  bootstrap:
    initdb:
      database: {{ .Values.postgres.bootstrapDatabase }}
      owner: {{ .Values.postgres.bootstrapOwner }}
```

Two `pg_hba` modes: `md5` for the `postgres` superuser (Terraform reads the password from SSM); `cert clientcert=verify-full` for every app role. Apps cannot use a password — `pg_hba` doesn't allow it. The superuser cannot use a cert — also intentional, one path each.

Backup (`backup.barmanObjectStore` to S3) is a stub field, off by default — implementation deferred to v1.1 (see Out of scope).

### 4. CoreDNS rewrite

Internal-only DNS, no ExternalDNS. Each tenant exposes its own alias — `soma.internal.cloudhaven.work`, `temporal.internal.cloudhaven.work`, `langfuse.internal.cloudhaven.work` — each pointing at that tenant's `<cluster>-rw` ClusterIP service.

Implementation: post-install / post-upgrade Helm Job that `kubectl patch`es the kube-system CoreDNS ConfigMap to add the rewrite block, then bounces CoreDNS. Idempotent (looks for the marker comment before patching). `pre-delete` Job strips the same block.

**Open question (§9):** Talos reconciliation behavior on the CoreDNS ConfigMap. Verify before merging Phase 2.

### 5. Auth — apps via cert, master via SSM password

CNPG generates a self-signed CA on first boot (`<cluster>-ca` Secret). cert-manager `ClusterIssuer` of kind `CA` references that secret.

**App certs** — for each entry in `appRoles`:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: {{ .name }}-pg-cert
  namespace: {{ .namespace }}
spec:
  secretName: {{ .name }}-pg-cert
  issuerRef:
    name: {{ $.Values.postgres.clusterName }}-ca-issuer
    kind: ClusterIssuer
  commonName: {{ .pgRole }}             # MUST equal the PG login role exactly
  duration: 2160h                       # 90 days
  renewBefore: 720h                     # 30 days — cert-manager auto-rotates
  usages:
    - client auth
```

App pod mounts `/certs/{tls.crt,tls.key,ca.crt}` from the Secret. Connects:

```python
psycopg.connect(
    host="soma.internal.cloudhaven.work",
    sslmode="verify-full",
    sslcert="/certs/tls.crt",
    sslkey="/certs/tls.key",
    sslrootcert="/certs/ca.crt",
)
```

**Master password** — CNPG auto-generates `<cluster>-superuser` (`username=postgres`, generated `password`). ESO `PushSecret` writes the username + password to AWS SSM as a single SecureString JSON blob:

```
/soma/postgres/<cluster>/master       # SecureString, value = {"username": "postgres", "password": "..."}
```

GHA runner (Tailscale-joined) uses OIDC → `gha-deployer` role → `ssm:GetParameter --with-decryption` → parses the JSON → runs `terraform apply` with the password in env (`PGPASSWORD`) or wired into the provider block.

```hcl
data "aws_ssm_parameter" "soma_master" {
  name            = "/soma/postgres/soma-pg/master"
  with_decryption = true
}

locals {
  master = jsondecode(data.aws_ssm_parameter.soma_master.value)
}

provider "postgresql" {
  host            = "soma.internal.cloudhaven.work"
  port            = 5432
  database        = "postgres"
  username        = local.master.username
  password        = local.master.password
  sslmode         = "verify-full"
  sslrootcert     = "/tmp/pg-ca/ca.crt"      # CA cert pulled from a separate SSM param (public CA cert is not sensitive)
}
```

The CA cert is pushed to a separate non-sensitive SSM `String` parameter (`.../master/ca.crt`) — `verify-full` requires the runner to trust the CNPG self-signed CA. Cert is just the public portion; not a secret.

**Apps never see the master password.** It exists in three places only: the CNPG-generated Secret in-cluster, the SSM SecureString, and ephemerally in the Terraform runner's process env during a TF run.

### 6. Bootstrap — Terraform owns role creation from day one

No bootstrap Job needed. Terraform handles all role/grant creation directly using the master password from SSM. The flow on first install:

1. Helm install creates the CNPG Cluster + cert-manager Issuer + per-app Certificates + ESO PushSecret.
2. ESO pushes superuser → SSM within `refreshInterval` (≤ 1h on first sync; sub-second in practice on initial reconcile).
3. CI runs `terraform apply` for the tenant's root config. TF reads SSM, connects as `postgres`, creates group roles + LOGIN roles + grants per `app_roles` input.
4. Apps deploy with mounted certs and connect successfully.

Apps will fail to connect between (1) and (3) because their LOGIN roles don't exist yet — fine; they'll CrashLoopBackOff briefly and self-heal once TF lands. Real ordering safety belongs in the ArgoCD sync wave (`sync-wave` annotation: chart wave 0, TF-driven roles wave 1, app deployment wave 2). Documented in the chart README.

### 7. App role model — group roles + cert login roles

Standard Postgres pattern, expressed cleanly in values.yaml:

```yaml
appRoles:
  - name: theo-orchestrator              # cert SecretName + cert-manager Certificate name
    pgRole: theo_orchestrator            # PG LOGIN role; equals cert CN
    namespace: theo
    database: soma
    schema: theo
    mode: rw                             # rw | ro | admin

  - name: theo-api
    pgRole: theo_api
    namespace: theo
    database: soma
    schema: theo
    mode: ro

  - name: nutrition-agent
    pgRole: nutrition_agent
    namespace: theo-agents
    database: soma
    schema: nutrition
    mode: rw
```

Bootstrap Job (and Terraform module) derive the unique `(database, schema, mode)` triples and create a NOLOGIN **group role** per triple:

| Group role | Privileges |
|---|---|
| `<db>_<schema>_ro` | `USAGE` on schema, `SELECT` on all tables (current + future via `ALTER DEFAULT PRIVILEGES`) |
| `<db>_<schema>_rw` | RO grants + `INSERT`, `UPDATE`, `DELETE` on all tables (current + future) |
| `<db>_<schema>_admin` | `ALL` on schema + `CREATE` |

LOGIN roles (one per `appRoles` entry) inherit membership in the matching group:

```sql
CREATE ROLE theo_orchestrator LOGIN;
GRANT soma_theo_rw TO theo_orchestrator;

CREATE ROLE theo_api LOGIN;
GRANT soma_theo_ro TO theo_api;
```

**Why this shape:**
- Adding a new app = create cert + `GRANT <group> TO <login_role>`. Two lines, no per-app grant blocks.
- Adding a new table to a schema = no per-app grant churn. `ALTER DEFAULT PRIVILEGES` ensures group roles get the right privileges automatically when the table is created.
- `mode` change for an app = `REVOKE <old_group>; GRANT <new_group>`. No table-by-table fix-up.
- Audit is one query per group role — who can write to `soma.theo`? `\du+ soma_theo_rw`.

### 8. Terraform module — `Infrastructure/modules/postgres-platform/`

(Author: Afolabi.) Reusable module consuming the same `appRoles` shape from the chart's values.yaml. Sketch of the module's responsibilities:

- Inputs: `cluster_endpoint`, `cluster_name`, `app_roles` (list of objects matching the chart's schema), `secret_backend = ssm | secretsmanager`, `master_secret_path`.
- Reads the master credential JSON (`{username, password}`) and CA cert from the configured backend.
- Configures `cyrilgdn/postgresql` provider with username/password + `sslmode=verify-full` + `sslrootcert`.
- Derives unique `(database, schema, mode)` triples from `app_roles` and creates the matching group roles (`<db>_<schema>_<mode>`).
- Creates LOGIN roles for each `app_roles` entry; grants the correct group membership.
- `ALTER DEFAULT PRIVILEGES IN SCHEMA <schema> GRANT ... TO <group>` so future tables inherit the right grants.
- Outputs: per-role grant resource IDs (for downstream `depends_on` if needed).

**Rule (from memory — `feedback_module_placement.md`):** module lives at `Infrastructure/modules/postgres-platform/`, not `Infrastructure/modules/aws/<name>/` — it touches Postgres + AWS SSM, cross-provider. Per-tenant root configs live at `Infrastructure/applications/postgres-{soma,temporal,langfuse}/` and just wire the module with tenant-specific inputs.

**Rule (from memory — `feedback_no_overexplanation_in_code.md`):** no narrative comments in the module. Variable descriptions in `variables.tf` carry the documentation.

### 9. Secret rotation strategy

| Secret | Rotation | Mechanism | App-side handling |
|---|---|---|---|
| **App client certs** (every `appRoles` entry) | Auto, every 60 days (90d duration, 30d `renewBefore`) | cert-manager | Cert Secret updates in place; mounted volume reflects new cert. Postgres connections re-authenticate per-connection — pool refresh on next session, no app code change required. |
| **CNPG superuser password** | Manual / on-demand (recommended quarterly; required on suspected compromise) | Delete `<cluster>-superuser` Secret → CNPG regenerates → ESO PushSecret re-pushes within `refreshInterval` (1h) → next TF run picks up the fresh password automatically. | Zero app impact (apps don't use the password). One-line break-glass: `kubectl delete secret <cluster>-superuser -n <ns>`. Schedule a quarterly rotation as a recurring Linear/calendar item; not worth automating until we see real volume of admin usage. |
| **CNPG CA cert** | 10 years (CNPG default) | n/a | Rotation = full cluster recreation. Document the procedure in `gitops/charts/postgres-platform/README.md` but don't automate — too rare. |
| **SSM SecureString KMS key** | AWS-managed (auto-rotated annually) | KMS | No action — `aws/ssm` default key is fine. |

**Rule:** rotation must not require a redeploy. Cert-manager + mounted Secrets give us this for free as long as apps don't pin connections forever (psycopg + SQLAlchemy default pools are fine; long-lived connection strings need explicit recycling).

### 10. Per-tenant Cluster instances

Three ArgoCD Applications, each pointing at the same chart with different values:

| Tenant | `clusterName` | `dnsAlias` | `appRoles` | Values file |
|---|---|---|---|---|
| soma | `soma-pg` | `soma.internal.cloudhaven.work` | `[theo_orchestrator, theo_api, nutrition_agent, meal_agent, ...]` | `gitops/apps/postgres-soma/values.yaml` |
| temporal | `temporal-pg` | `temporal.internal.cloudhaven.work` | `[temporal]` | `gitops/apps/postgres-temporal/values.yaml` |
| langfuse | `langfuse-pg` | `langfuse.internal.cloudhaven.work` | `[langfuse]` | `gitops/apps/postgres-langfuse/values.yaml` |

Three Cluster CRs, three sets of EBS PVCs, three SSM path prefixes, no shared blast radius — Langfuse failing to apply a schema migration cannot take down Temporal.

### 11. Observability hookup

- **Logs:** CNPG emits structured JSON to stdout. Alloy DaemonSet (Phase 2 obs work) tails container logs → Loki. No CNPG-specific Alloy config — generic container path.
- **Metrics:** chart ships a `PodMonitor` referencing the CNPG metrics endpoint. VictoriaMetrics scrapes it. Dormant until VM exists.
- **Dashboards:** import the upstream CNPG Grafana dashboards (IDs `20417`, `20418`) as ConfigMaps in the Grafana namespace. Not in this chart — belongs to the Grafana install.

### 12. Human access — out of scope

No SSO, no Authentik, no CloudBeaver in v1. Future plan: a JIT cert-issuance system that mints a 1h-TTL cert tied to a human identity, on demand, for ad-hoc admin access. Until that exists, humans don't connect to Postgres directly — all DB work goes through Terraform (schema/role changes) or the apps themselves (data reads/writes).

This is a deliberate choice — not building human access machinery we'd throw away when the JIT system lands. Tracked as a separate future plan.

---

## Repo placement summary

| Concern | Path | Repo |
|---|---|---|
| CNPG operator install | `K8s-Bootstrap/{humboldt,nebulosa}/cnpg-operator/` | K8s-Bootstrap |
| cert-manager install | `K8s-Bootstrap/{humboldt,nebulosa}/cert-manager/` | K8s-Bootstrap |
| ESO install (Nebulosa) | `K8s-Bootstrap/nebulosa/external-secrets/` | K8s-Bootstrap |
| `ebs-gp3` StorageClass | `K8s-Bootstrap/humboldt/ebs-csi-driver/storageclass-gp3.yaml` | K8s-Bootstrap |
| **`postgres-platform` Helm chart** | **`gitops/charts/postgres-platform/`** | **gitops** |
| Per-tenant ArgoCD Apps + values | `gitops/apps/postgres-{soma,temporal,langfuse}/` | gitops |
| **`postgres-platform` TF module** | **`Infrastructure/modules/postgres-platform/`** | **Infrastructure** |
| Per-tenant TF root configs | `Infrastructure/applications/postgres-{soma,temporal,langfuse}/` | Infrastructure |
| ESO IRSA role (humboldt) | `Infrastructure/platform/humboldt/eso-irsa.tf` | Infrastructure |
| GHA `gha-deployer` SSM read policy extension | `Infrastructure/modules/aws/iam/gha-deployer/` | Infrastructure |

---

## Phasing

Each phase is a mergeable, independently demoable slice. Don't start phase N+1 until N is green on humboldt.

### Phase 0 — Cluster prereqs on humboldt (~2h)
- ArgoCD app: CNPG operator
- ArgoCD app: cert-manager
- Apply `ebs-gp3` StorageClass with `WaitForFirstConsumer`
- Smoke: `kubectl get pods -n cnpg-system` healthy, test PVC binds and releases

### Phase 1 — Single-tenant chart, password auth, no certs (~3h)
- Skeleton chart with Cluster CR + values.yaml + values.ebs-gp3.yaml
- Install for `soma` tenant, 3 instances, EBS gp3
- Verify: 3 pods Running, primary elected, `psql -h <svc>` works using the auto-generated superuser password from inside the cluster
- **No certs, no DNS rewrite, no ESO yet.** Just the cluster shape.

### Phase 2 — CoreDNS rewrite + per-tenant alias (~1h)
- Add post-install Job to chart
- Verify: from a debug pod, `nslookup soma.internal.cloudhaven.work` returns the `-rw` ClusterIP; `NXDOMAIN` from outside

### Phase 3 — App cert auth, one test role (~3h)
- Add `ClusterIssuer` + per-`appRoles` `Certificate` templates
- Patch `pg_hba` for one test role (`smoke_role`)
- Manually `CREATE ROLE smoke_role LOGIN` from a `psql` shell (Terraform takes over in Phase 5)
- Verify: pod with mounted cert connects passwordless via `sslmode=verify-full`
- Verify: pod **without** the cert is rejected at TLS handshake

### Phase 4 — ESO PushSecret: superuser + CA → SSM (~2h)
- Add ESO `ClusterSecretStore` (SSM backend) + `PushSecret` for the superuser Secret (JSON-encoded `{username, password}` → SecureString)
- Add a second `PushSecret` for the CNPG CA cert (`<cluster>-ca.crt` → plain SSM `String` parameter, not sensitive)
- Provision IRSA role for ESO via Terraform (`Infrastructure/platform/humboldt/eso-irsa.tf`) with `ssm:PutParameter` on `/soma/postgres/*`
- Verify: `aws ssm get-parameter --name /soma/postgres/soma-pg/master --with-decryption` returns the live `{username, password}` JSON
- Verify: rotating the CNPG superuser (delete-the-secret trick) propagates to SSM within `refreshInterval`

### Phase 5 — Terraform module (SSM password path) — Afolabi authors (~3h coordination)
- `Infrastructure/modules/postgres-platform/` consumes the `app_roles` shape + `master_secret_path`
- GHA workflow: OIDC → `gha-deployer` → `ssm:GetParameter` for master + CA → `terraform apply` over Tailscale
- Module derives unique `(database, schema, mode)` triples → creates group roles → creates LOGIN roles → grants membership → `ALTER DEFAULT PRIVILEGES`
- Verify: TF creates `smoke_role` (or replaces the manual one from Phase 3) idempotently; the cert from Phase 3 still authenticates after TF runs

### Phase 6 — Multi-tenant: temporal + langfuse Cluster CRs (~2h)
- Two more ArgoCD Apps + values files, two more TF root configs
- Verify: three independent CNPG clusters, three SSM path prefixes (`/soma/postgres/{soma-pg,temporal-pg,langfuse-pg}/master`), no cross-talk
- Unblocks Temporal install and Langfuse install in v3 plan's runtime infra list

### Phase 7 — Nebulosa overlay via `values.local-path.yaml` (~1h)
- 1 instance, `local-path` StorageClass, `secretBackend.type: none` (Pi doesn't push to SSM — dev-only cluster, master password stays in the in-cluster Secret only)
- Terraform skipped on Nebulosa (no remote Postgres provider; Pi schema is hand-managed for dev)
- Validates the cross-cluster portability claim — same chart, different storage class, different secret backend toggle

### Phase 8 — Observability wiring (deferred, depends on Phase 2 obs work)
- `PodMonitor` ships from Phase 1 onward but dormant until VictoriaMetrics exists
- Import CNPG Grafana dashboards as ConfigMaps once Grafana is up

---

## Acceptance criteria (Phase 0–6 for v1)

- [ ] `helm install postgres-platform -f values.ebs-gp3.yaml` brings up a 3-replica CNPG cluster on humboldt with no errors.
- [ ] CoreDNS resolves `soma.internal.cloudhaven.work` from any pod, `NXDOMAIN` from outside.
- [ ] An app pod with a cert-manager-issued cert (CN=`smoke_role`) authenticates with `sslmode=verify-full` and zero passwords in env or volumes.
- [ ] A pod **without** the cert is rejected at the TLS layer (proves `clientcert=verify-full` is enforcing).
- [ ] `aws ssm get-parameter --name /soma/postgres/soma-pg/terraform-admin/tls.crt --with-decryption` returns live cert material.
- [ ] `terraform apply` against `Infrastructure/applications/postgres-soma/` runs without ever reading a password — uses cert from SSM, connects as `postgres` via `pg_ident` map, creates group roles + login roles, grants membership.
- [ ] Three independent Cluster CRs exist (`soma-pg`, `temporal-pg`, `langfuse-pg`) with three separate PVC sets, three SSM path prefixes, no shared role namespace.
- [ ] Same chart deploys cleanly on Nebulosa with `values.local-path.yaml`.
- [ ] No `password=` strings in any committed YAML, Terraform, or app code.
- [ ] Cert auto-renewal verified: temporarily set `renewBefore` to a value that triggers immediate rotation, confirm new cert lands in mounted volume and SSM without a redeploy.

---

## Out of scope (v1)

- **Backups (pgbackrest to S3).** `backup` field is a stub. Real backups in v1.1 — needs S3 bucket, IAM role, retention, restore drill.
- **Read replicas exposed at `<tenant>-ro.internal.cloudhaven.work`.** Easy add later (CNPG ships a `-ro` service).
- **Connection pooling (PgBouncer / pgcat).** Add when connection counts approach Postgres' default `max_connections`. Direct CNPG until then.
- **Multi-AZ HA.** Single AZ on humboldt today. Multi-AZ is a values change (`affinity` block + 3 AZs in the EBS CSI config) once we have the AZ footprint.
- **pgvector extension.** Already on the ops-log Backlog for the semantic nutrition cache. Drop-in `shared_preload_libraries` change when the time comes.
- **Vault as backend.** Toggle exists in values; only SSM + SM implementations land in v1.
- **Human access (SSO / Authentik / CloudBeaver / web IDE).** Replaced by future JIT cert-issuance plan. Until then, humans don't directly connect.
- **TLS cert revocation list (CRL).** CNPG CA doesn't ship CRL distribution OOTB. Cert compromise → revoke = rotate the CA + reissue everything. Documented as a known limitation.

---

## Open questions

1. **Talos + CoreDNS rewrite durability.** Does Talos reconcile-roll-back our CoreDNS ConfigMap patch on next machine config apply? Test before Phase 2 lands. Fallback: deploy our own CoreDNS instance dedicated to internal rewrites and chain it from kube-system.
2. **`ClusterIssuer` vs namespaced `Issuer`.** Per-app `Certificate` lives in the app's namespace; CA Secret lives in CNPG namespace. `ClusterIssuer` is the simpler answer; confirm cert-manager can read the CA Secret across namespaces via standard RBAC.
3. **Bootstrap Job vs Terraform module overlap.** Both can create group roles. Cleanest: bootstrap Job creates only what's needed for Terraform to connect (the `pg_ident` map is the only real prerequisite — no roles needed before TF takes over since `terraform_admin` maps to `postgres`). Then Terraform module owns all other roles + grants from day one. Confirm before writing the bootstrap Job — might shrink it to a one-shot `pg_reload_conf()`.
4. **Nebulosa SSM push.** `values.local-path.yaml` defaults to `secretBackend.type: none` — Pi never pushes to SSM. Confirm Pi is genuinely dev-only and never needs cross-machine cert distribution. If wrong, add a static IAM user secret on Pi for ESO.
5. **GHA runner Tailscale → Postgres ACL.** Phase 5 needs a tailnet ACL allowing `tag:ci-runner → tag:humboldt-pg:5432`. Add to the Tailscale ACL Terraform module when Phase 5 lands.
6. **Cert-auth in `cyrilgdn/postgresql`.** Confirm the provider's `clientcert` block accepts file paths (vs inline PEM). If file paths only, the GHA step that writes cert material to `/tmp` is mandatory; if inline, can pass via env vars and skip the temp files. Worth a 5-min provider docs check before Phase 5.

---

## Dependencies on other plans

- **v3 scaffold plan (`2026-04-20-theo-siblings-scaffold-v3.md`)** — Theo's `theo-core/postgres.py` SQLAlchemy session factory expects cert files at `/certs/`. Cert mount path must match. Coordinate when Theo deploys.
- **Langfuse fallback prompts plan (`2026-04-20-langfuse-fallback-prompts.md`)** — independent.
- **Phase 2 observability (ops-log Next-Session Priority 2)** — `PodMonitor` ships dormant until VictoriaMetrics exists. No blocking dependency in either direction.
- **Temporal install** — blocked by Phase 6 (needs the `temporal-pg` Cluster).
- **Langfuse install** — blocked by Phase 6 (needs the `langfuse-pg` Cluster).
- **Future JIT human-access plan** — depends on the cert/CA infra in this plan being live, but otherwise independent.
