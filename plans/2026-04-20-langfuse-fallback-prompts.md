# Langfuse fallback prompts — implementation plan

**Status:** ready for review, not yet implemented
**Tracking issues:**
- [Cloudhaven-IDP/theo-agents#1](https://github.com/Cloudhaven-IDP/theo-agents/issues/1) — nutrition + meal
- [Cloudhaven-IDP/theo#1](https://github.com/Cloudhaven-IDP/theo/issues/1) — `theo-persona` + `theo-onboarding-checklist` + adds the 5-min TTL cache theo-core lacks today

---

## Context

Every model-facing prompt in theo and theo-agents is fetched at runtime from Langfuse via `fetch_prompt(name, label)` (agent-core) or `get_prompt(name, label)` (theo-core). This gives us hot-reload: a prompt edit in the Langfuse UI propagates to running pods within 5 minutes, no redeploy. We moved *every* prompt there — vision, planner, parse, persona, onboarding checklist — because non-engineers (or future-Afolabi) need to tune them without a build.

Problem: today the apps have no bootstrap story and no safety net. Three concrete failure modes:

1. **Langfuse reachable, prompt unseeded** → SDK raises `NotFoundError` → unhandled → 500 on `/run`.
2. **Langfuse disabled** (`LANGFUSE_ENABLED=false`, default in local dev) → `fetch_prompt` returns `""`. Then `meal/planner.py` does `json.loads("")` → crash. Nutrition vision fires with empty prompts → garbage response.
3. **Langfuse transiently unreachable** (network blip, DNS, pod restart) → same as (1).

`/health` is 200 in all three. Any real request is broken. We noticed this mid-scaffold and needed to pick a path.

---

## Options considered

| | Approach | Verdict |
|---|---|---|
| A | Init-container seeder writes missing prompts to Langfuse on pod start; runtime still fetches with TTL cache | Defer. Real future option if env drift becomes painful, but more plumbing than we need for MVP. |
| B | Embedded fallback string registered per prompt name; `fetch_prompt` returns fallback on 404/timeout/disabled | **Chosen.** Simplest path; app survives any Langfuse state. |
| C | Human clicks prompts into Langfuse UI before first deploy | Rejected. Not repeatable across preview / DR / new tenant. Drift-prone. |
| D | Auto-seed on app startup, first pod wins | Rejected. Race conditions with N replicas; still needs "default" in code. |

### Why B beats A for now

- App boots and serves **regardless of Langfuse state**. Langfuse outage = degraded persona, not a page.
- No new deploy-time primitive (init-container, Job, sidecar). Uses the runtime code path we already have.
- Fallback strings are explicitly labeled `"MINIMAL FALLBACK — replace in Langfuse UI"` — greppable, obviously-v0, impossible to mistake for the live prompt.
- When we want A later, it adds on top of B without rework: seeder writes fallback values into Langfuse if prompt name is missing. Fallback-in-code + seeder-on-deploy compose cleanly.

### Accepted tradeoff

Two sources of truth: embedded fallback vs live Langfuse prompt. They will drift — that's fine, fallbacks are the "last-known-safe v0", not current production. Sibling enhancement in the Out-of-scope section proposes CI drift detection when this starts to bite.

---

## Design

### 1. Public API (both `agent_core.langfuse` and `theo_core.langfuse_client`)

Both libraries expose the same shape. Name can differ (`fetch_prompt` in agent-core, `get_prompt` in theo-core — keep existing names, don't churn callers).

```python
def register_fallback(name: str, body: str) -> None: ...

def fetch_prompt(name: str, label: str = "production") -> str:
    """
    Return the live Langfuse prompt if reachable and seeded.
    Fall back to the registered fallback on:
      - client is None (LANGFUSE_ENABLED=false or no host configured)
      - NotFoundError from the Langfuse SDK
      - httpx.HTTPError / httpx.TimeoutException
    Raise if the name has no registered fallback AND Langfuse can't serve it.
    """
```

**Contract:** unknown prompt names (nothing registered, nothing in Langfuse) raise — silent empty strings are a footgun that hides missing registrations during development.

### 2. TTL cache (theo-core parity fix)

`agent_core.langfuse.fetch_prompt` already has a 5-minute TTL cache. `theo_core.langfuse_client.get_prompt` does not — every orchestrator call hits Langfuse today. Bring theo-core into parity while we're in the file. The cache keys on `(name, label)` and invalidates on fetch-error to avoid poisoning. Fallbacks are never cached; the next request retries Langfuse.

### 3. Where fallback bodies live

Default: inline in the registering module as a Python string. Fallbacks are short by design (`"MINIMAL FALLBACK — replace in Langfuse UI. You are a nutrition expert. Be precise about portions."`), so triple-quoted strings are fine.

Escalation path: if any one fallback grows past ~10 lines (e.g. the master persona prompt), promote *that one* to `prompts/<name>.md` next to the module and load via `Path(__file__).parent / "prompts" / "foo.md"`.read_text(). Don't preemptively build a loader framework.

### 4. Registrations happen at import time

Each module calls `register_fallback` at import, so fallbacks are in place before any request arrives. Tests that import the module under test automatically pick up the registrations — no fixture needed.

---

## File-by-file changes

### theo-agents (tracking [#1](https://github.com/Cloudhaven-IDP/theo-agents/issues/1))

| File | Change |
|---|---|
| `libs/agent-core/agent_core/langfuse.py` | Add `register_fallback` + `_fallback`. Wrap `fetch_prompt` body in try/except for `NotFoundError`, `httpx.HTTPError`, `httpx.TimeoutException`. Do not cache fallback results. |
| `libs/agent-core/tests/test_langfuse.py` | New file. Cover: disabled-client → fallback; `NotFoundError` → fallback; timeout → fallback; live prompt → live prompt; unregistered name → raise; TTL cache honored for live fetches; cache not poisoned by transient error. |
| `nutrition/src/nutrition/vision.py` | `register_fallback` for `nutrition.vision.describe_food.{system,user}` and `nutrition.vision.describe_label.{system,user}` at module top. |
| `nutrition/src/nutrition/handlers.py` | `register_fallback` for `nutrition.parse_food_text` (template with `{input}` placeholder — fallback must preserve the placeholder so the `.replace("{input}", ...)` call still makes sense). |
| `meal/src/meal/planner.py` | `register_fallback` for `meal.planner.{system,user}`. User prompt template must preserve `{context}`, `{avoid}`, `{days}` placeholders. The system-level fallback should ask the model to return a JSON array so `json.loads` in `generate()` doesn't crash on fallback usage. |

### theo (tracking [#1](https://github.com/Cloudhaven-IDP/theo/issues/1))

| File | Change |
|---|---|
| `libs/theo-core/theo_core/langfuse_client.py` | Add `register_fallback` + `_fallback`. Add 5-min TTL cache (currently missing). Wrap `get_prompt` body in the same try/except shape as agent-core. |
| `libs/theo-core/tests/test_langfuse_client.py` | New file. Same six paths as agent-core's test file, plus a cache-honors-TTL test. |
| `orchestrator/src/orchestrator/coach/persona.py` | `register_fallback("theo-persona", ...)` with a minimal but functional coach system prompt. |
| `orchestrator/src/orchestrator/coach/onboarding.py` | `register_fallback("theo-onboarding-checklist", ...)` with a **valid JSON body** — `json.loads()` on the fallback must succeed. One or two questions is enough. |

---

## Testing plan

Per-library test matrix (six paths each):

1. `LANGFUSE_ENABLED=false` → returns registered fallback.
2. Live prompt exists in Langfuse → returns live prompt, not fallback.
3. Langfuse 404s (`NotFoundError`) → returns fallback.
4. Langfuse times out → returns fallback; cache not populated.
5. `get_prompt("unregistered.name")` with Langfuse 404 → raises.
6. Second call within TTL window returns cached live value without hitting Langfuse.

Integration-level sanity checks:

- `POST /run` on nutrition with `LANGFUSE_ENABLED=false` returns a sensible response (no 500).
- `POST /run` on meal with `LANGFUSE_ENABLED=false` doesn't crash in `json.loads` — fallback returns valid JSON array.
- Orchestrator `/run` with `LANGFUSE_ENABLED=false` returns a response with the fallback persona engaged.
- Onboarding node under Langfuse outage uses fallback checklist and advances through at least one question.

---

## Acceptance criteria

- [ ] Unknown prompt names raise. No silent empties anywhere in the codebase.
- [ ] All fallback strings start with the literal `"MINIMAL FALLBACK — replace in Langfuse UI"` prefix. Greppable.
- [ ] Fallback strings that carry template placeholders (`{input}`, `{context}`, `{avoid}`, `{days}`) preserve them, so callers' `.replace(...)` calls are still valid against the fallback body.
- [ ] The onboarding-checklist fallback is valid JSON and `json.loads()`-es cleanly.
- [ ] theo-core's `get_prompt` has a 5-min TTL cache matching agent-core's.
- [ ] Nutrition, meal, and theo services all return non-500 responses to normal requests with `LANGFUSE_ENABLED=false`.
- [ ] Test coverage for the six paths on each library.

---

## Out of scope (future issues, not this PR)

1. **Init-container seeder** — revisit when fallback drift across envs becomes painful, or when we want a new tenant / DR environment to bootstrap without manual Langfuse UI clicks.
2. **Fallback-drift CI check** — warns when an embedded fallback diverges from the current `production`-labeled version in Langfuse beyond a similarity threshold. Prevents fallbacks rotting silently into uselessness.
3. **Extract `prompts-core` shared package** — once theo-core and agent-core expose the identical prompt API, factor it out. Not worth the extra publish dance until both surfaces stabilize.
4. **Promote long fallbacks to `prompts/<name>.md`** — only when any one fallback exceeds ~10 lines. YAGNI for MVP.

---

## Open questions (flag before implementation)

1. **Does `register_fallback` reject duplicate registration?** Proposal: warn on duplicate name to catch copy-paste bugs, last-write-wins to let tests override if they need to. Not a hard error.
2. **Should the prompt cache distinguish `(name, label)` or just `name`?** Proposal: `(name, label)` — staging vs production labels may return different bodies during promotion windows, and caching only on name would poison the first label fetched.
3. **Fallback for `meal.planner.user`** is tricky because the real prompt returns JSON. Proposal: fallback user prompt literally asks the model to `"Return an empty JSON array: []"` so `json.loads` always succeeds in fallback mode. Ugly but correct; real planner prompt in Langfuse does the actual work.

---

## References

- Plan for the parent scaffolding work: `ops-log/plans/2026-04-20-theo-siblings-scaffold-v3.md`
- Ops-log entry (LOUD banner + Second pass section + Priority 1 bullet linking here): `ops-log/log/2026-04-20.md`
- Memory: `feedback_prompts_in_langfuse.md` (the rule that created this problem)
