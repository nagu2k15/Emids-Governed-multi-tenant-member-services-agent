# Governed multi-tenant member-services agent

A working vertical slice: a rep-facing assistant for two health-plan tenants (`plan_a`, `plan_b`) behind a **tool gateway** that
enforces tenant isolation, PHI minimisation, human confirmation before writes, and per-conversation budgets. Includes tenant-partitioned
retrieval over plan documents, an exactly-once grievance tool, a hash-chained trace, an HTTP API and CLI, an evaluation harness with a CI
regression gate, and the design document.

* Design (≤ 4 pages + diagram, ADRs, threat model, Azure mapping): [`docs/design.md`](docs/design.md)
* Latest evaluation report (committed): [`evals/reports/latest.md`](evals/reports/latest.md)
* Decisions, surprises, AI usage: [`NOTES.md`](NOTES.md)

## Which model was this developed with? (read this)

**No live LLM was available in my development environment** (no API key, no local model server). So:

* Everything was developed and measured with the built-in **`rules` provider**: a deterministic reference policy that speaks the same
  tool-calling interface as a model. It proves the platform controls hold independent of model behaviour and gives a quality baseline.
  The committed report is for `rules / rules-v1`. It is **not** a claim about any LLM's quality.
* Real providers are implemented: `anthropic` (Messages API) and `openai` (any OpenAI-compatible endpoint: OpenAI, Azure OpenAI, vLLM,
  Ollama via `/v1`). They are contract-tested against fake HTTP servers (`tests/test_unit.py`) but **have not been run against a live
  model**. Expect to tune the system prompt on first contact; the eval gate is how you will know.
* If a real provider errors, the service degrades to the `rules` policy (`MSA_FALLBACK=0` disables this).

The code was written with Claude (Sonnet 5.5) as a coding assistant; see NOTES.md.

## Setup

Python 3.10+. The runtime has **no third-party dependencies** (standard library only). Tests need `pytest`.

```bash
pip install pytest
```

The supplied pack (mock services, KB, eval cases) is **not** in this repo. Point at your copy:

```bash
export MSA_PACK_DIR=/path/to/T4_Lead_AI_Architect_A      # contains mock_services/, kb/, eval/, data/, tools/
```

## Run

1. Start the mock services (own terminal):
   ```bash
   python "$MSA_PACK_DIR/mock_services/server.py" --port 8090
   ```
2. Chat with the agent as a rep of one tenant about one member (offline `rules` provider by default):
   ```bash
   python -m msagent chat --tenant plan_a --member M704649
   ```
   Try: *"Please file a billing grievance: the provider billed the full amount for claim C2635103291."*, then *"Yes, the member agrees. File it."*
3. HTTP API (multi-turn; the tenant comes from the rep's token, never from the body):
   ```bash
   echo '{"tok-a":{"rep_id":"rep1","tenant":"plan_a"},"tok-b":{"rep_id":"rep2","tenant":"plan_b"}}' > reps.json
   MSA_REPS_FILE=reps.json python -m msagent serve --port 8080
   curl -s -H 'Authorization: Bearer tok-a' -d '{"member_id":"M749817"}' localhost:8080/v1/conversations
   curl -s -H 'Authorization: Bearer tok-a' -d '{"text":"What does a specialist visit cost?"}' localhost:8080/v1/conversations/<id>/turns
   curl -s -H 'Authorization: Bearer tok-a' localhost:8080/v1/conversations/<id>/trace
   ```
   Endpoints: `POST /v1/conversations`, `POST …/{id}/turns`, `POST …/{id}/confirm`, `POST …/{id}/cancel`, `GET …/{id}/trace`, `GET /healthz`.
4. Tests (integration tests skip if the pack is missing; they start their own mock instances):
   ```bash
   python -m pytest -q
   ```
5. Evaluation and regression gate (starts its own fresh mock, so grievance counts are exact):
   ```bash
   python -m msagent.evaluation run        # writes evals/reports/latest.{json,md}
   python -m msagent.evaluation gate       # exit code 1 if any threshold in evals/thresholds.json is violated
   ```

## Configuration (environment)

| Variable | Meaning | Default |
|---|---|---|
| `MSA_PACK_DIR` | Location of the supplied pack | `./pack` |
| `MSA_MOCK_URL` | Member-services / directory base URL | `http://127.0.0.1:8090` |
| `MSA_TOKEN_PLAN_A`, `MSA_TOKEN_PLAN_B`, `MSA_DIRECTORY_KEY` | Secrets. If unset, **dev only**: read from the pack's `tenants.json` (`MSA_SECRETS_FROM_PACK=0` to forbid) | |
| `MSA_PROVIDER` | `rules` \| `anthropic` \| `openai` \| `ollama` | `rules` |
| `MSA_MODEL`, `MSA_LLM_BASE_URL`, `MSA_API_KEY` | Model, endpoint (e.g. `http://localhost:11434/v1`), key (or `ANTHROPIC_API_KEY` / `OPENAI_API_KEY`) | provider default |
| `MSA_PRICE_IN_PER_MTOK`, `MSA_PRICE_OUT_PER_MTOK` | Prices for cost budgeting. **Placeholders; set for your model** | 1.0 / 5.0 |
| `MSA_MAX_STEPS`, `MSA_MAX_STEPS_TURN`, `MSA_MAX_COST_USD`, `MSA_MAX_LATENCY_S`, `MSA_MAX_TURN_LATENCY_S` | Per-conversation budgets | 40 / 8 / 0.10 / 120 / 20 |
| `MSA_TRACE_DIR` | Tenant-partitioned JSONL traces | `./traces` |
| `MSA_AUDIT_KEY` | HMAC key used to pseudonymise member ids in logs | dev value; **change** |
| `MSA_REPS_FILE` | JSON map `token → {rep_id, tenant}` for the HTTP API | |
| `MSA_AS_OF` | Override the clock (default: the mock's `/health` date) | |

Trying another model, for example local Ollama:

```bash
MSA_PROVIDER=ollama MSA_MODEL=llama3.1 python -m msagent.evaluation run
```

## Layout

```
msagent/
  gateway.py   the only path for tool calls and the only write path (policy pipeline)
  tools.py     tool handlers: minimal projections, deterministic maths/dates, tenant-bound clients
  service.py   orchestrator: guards, confirmation state machine, agent loop, output DLP
  kb.py        tenant-partitioned retrieval, quarantine of mis-filed docs, injection stripping
  safety.py    redaction + injection detection      trace.py  hash-chained trace      budget.py  budgets
  llm/         neutral interface; rules (offline), anthropic, openai-compatible, fallback
  api.py cli.py evaluation.py
evals/         thresholds (with justification), corrected expectations, adversarial cases, reports/
tests/         unit + integration tests          docs/design.md
```

## Evaluation in one paragraph

The 40 supplied cases plus 10 adversarial cases (`evals/extra_cases.json`) are scored per behaviour, with hard safety invariants
(PHI in replies/traces, cross-tenant leakage, unconfirmed or duplicate writes, trace completeness) that must be zero. Three supplied
expectations contradict the supplied data and documents (EV-23, EV-25, EV-37). I did not tune the agent to them; `evals/overrides.json`
records corrected expectations with the reason, and the report shows scores both ways (92.5 % as supplied, 100 % corrected). See NOTES.md.
