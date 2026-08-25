# CLAUDE.md — AutoDefend AI

> Project charter and system design. This file is the single source of truth for Claude Code working in this repository. Read it fully before making changes. If an instruction here conflicts with an inline comment, a stale docstring, or a prior message, **this file wins**.

---

## 1. Project overview

**AutoDefend AI** is a deployed, autonomous chargeback representment service. When a cardholder disputes a payment, the merchant has a short window to contest it by assembling evidence scattered across the payment gateway, logistics carrier, and CRM. That assembly is manual and slow, so merchants either forfeit winnable disputes or waste submission fees defending unwinnable ones.

The system ingests a dispute event, autonomously gathers cross-system evidence over MCP, decides whether defending is *economically* worthwhile, generates a fully grounded representment package, and records a tamper-evident audit trail — with a human approval gate before anything is filed.

### What this project is optimising for

This is a portfolio-grade engineering artifact, not a startup MVP. Two audiences must both be satisfied:

- **Backend / SWE reviewers** — durable execution, idempotent ingestion, rate-limit and failure handling, deployed and reachable, CI with real quality gates.
- **AI/ML reviewers** — a defensible baseline hierarchy, calibrated probabilities, cost-sensitive decision theory, statistical honesty about small-sample results, and a hallucination-proof generation path.

Every design decision below exists to serve one of those two. If a proposed change serves neither, do not make it.

### Scope

**In scope:** dispute ingestion, evidence harvesting via MCP, EV-based defend/concede adjudication, grounded narrative generation, PDF package rendering, human-in-the-loop approval, full evaluation harness, deployment on permanently-free infrastructure.

**Explicit non-goals — do not build these, and state them as deliberate scope decisions:**

- **No direct filing to card networks.** Real representment is submitted through the acquirer/PSP, not by emailing a bank a PDF. The system produces a *submission-ready package* and stops at a stubbed PSP client interface.
- **Visa reason codes only** (10.x unauthorised, 13.x non-receipt / not-as-described). Do not claim Mastercard or RuPay support. Scheme code mapping is a stub with a documented TODO.
- **No fraud detection at authorisation time.** This is post-hoc dispute adjudication only.
- **No multi-tenant billing, no merchant onboarding flow, no real payment integration.**

### Domain facts to get right

Reviewers in fintech will check these. Use them correctly in code comments, docs, and the README.

| Term | Meaning |
|---|---|
| Chargeback | Cardholder-initiated reversal; funds and a fee are pulled from the merchant immediately |
| Representment | The merchant's rebuttal, submitted with documentary evidence |
| Response window | Visa allows 30 days from dispute, Mastercard 45. The 7–14 day figure commonly quoted is the **acquirer's internal merchant deadline**, which is what actually binds the merchant. Model the acquirer deadline, and label it as such. |
| Friendly fraud | Legitimate cardholder disputes a transaction they actually made — generally winnable with evidence |
| True fraud | Genuine card compromise — generally unwinnable; defending it wastes a fee and degrades the merchant's dispute ratio |
| ARN | Acquirer Reference Number, the transaction identifier banks trace by |
| AVS | Address Verification Service result — billing address match signal |
| 3DS | 3-D Secure / OTP authentication result; a pass shifts liability toward the issuer |
| POD | Proof of Delivery — carrier status, timestamp, signature, geolocation |

---

## 2. Architecture

### Request flow

```
   PSP / network dispute webhook
              │  HMAC-SHA256 signed, timestamped
              ▼
   ┌──────────────────────────────────┐
   │  POST /webhooks/dispute          │   verify signature → validate schema
   │  (FastAPI, Render)               │   → publish to queue → 202 Accepted
   └──────────────┬───────────────────┘   (no business logic on this path)
                  ▼
   ┌──────────────────────────────────┐
   │  Upstash QStash                  │   at-least-once, exponential backoff,
   │  durable job queue               │   dead-letter queue on exhaustion
   └──────────────┬───────────────────┘
                  ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  POST /internal/process   (same service, token-authenticated) │
   │                                                               │
   │  ① Idempotency gate  → Postgres UNIQUE(event_hash)            │
   │  ② Resume-or-start LangGraph run, thread = dispute_id         │
   └──────────────┬────────────────────────────────────────────────┘
                  ▼
   ┌──────────────────────────────────────────────────────────────┐
   │                  LangGraph orchestrator                       │
   │        checkpointed to Postgres after every node              │
   │                                                               │
   │   triage → harvest → extract → score → decide                 │
   │              │                            │                   │
   │              │                   ┌────────┴────────┐          │
   │              │                   ▼                 ▼          │
   │              │              CONCEDE            DEFEND         │
   │              │              (log + stop)          │           │
   │              │                                     ▼          │
   │              │                              draft → verify    │
   │              │                                     │          │
   │              │                                     ▼          │
   │              │                              gate (interrupt)  │
   │              │                                     │          │
   │              │                                     ▼          │
   │              │                              compile → emit    │
   │              ▼                                                │
   │        MCP client pool                                        │
   └──────────────┬────────────────────────────────────────────────┘
                  ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  MCP servers — 4 logical servers, streamable HTTP,            │
   │  mounted on the same process, bearer-authenticated            │
   │    logistics · gateway · crm · core                           │
   └──────────────────────────────────────────────────────────────┘

   Cross-cutting services
   ├── LLM gateway ....... rate limiting, backoff, provider fallback, caching, cost ledger
   ├── Neon Postgres ..... disputes, checkpoints, audit chain, cost ledger, PDFs
   ├── Upstash Redis ..... response cache, distributed token buckets
   ├── Langfuse .......... traces, token and cost attribution per node
   └── Console (server-rendered) ... dispute queue, evidence view, approve/reject, eval dashboard
```

### Why the queue exists

Free compute scales to zero and takes tens of seconds to wake. A webhook caller times out long before that, and the dispute is silently lost. Terminating the webhook immediately and handing off to a durable queue means the worker is *permitted* to be asleep — QStash retries with backoff until it wakes. Scale-to-zero becomes a cost optimisation instead of a correctness bug. This is the single most important infrastructure decision in the project; record it as an ADR.

### Deliberate consolidation

All HTTP surfaces — webhook receiver, internal processor, four MCP servers, console — run in **one FastAPI process on one free instance**. They are separate routers with separate auth, and the MCP servers speak real HTTP MCP so an external client could consume them. Document this as a conscious constraint-driven decision with a clear split path, not as an oversight.

---

## 3. Tech stack and free-tier budget

| Layer | Choice | Free allowance | Binding constraint | Handling |
|---|---|---|---|---|
| Compute | Render free web service | 512 MB, 0.1 CPU, spins down when idle | cold start, RAM | queue absorbs cold start; keep the image slim, avoid heavy ML imports at module load |
| Queue | Upstash QStash | ~1,000 messages/day | daily cap | eval runs bypass the queue and invoke the graph directly |
| Cache + limiter | Upstash Redis | ~500K commands/month, 256 MB | command count | cache only LLM and MCP responses, not hot-path reads |
| Database | Neon Postgres | scale-to-zero, branching, ~0.5 GB | cold start, storage | pooled connections; PDFs pruned on a retention policy |
| Primary LLM | Gemini `gemini-3.6-flash` | ~10–15 RPM, ~1,500 RPD, 250K TPM | **requests per day** | caching + record/replay fixtures; this is the limit that will bite |
| Fallback LLM | Groq (open-weights, OpenAI-compatible) | ~30 RPM, high daily ceiling | model catalogue churn | startup healthcheck against every configured model |
| Tracing | Langfuse cloud free tier | generous | retention | sample non-eval traces |
| CI | GitHub Actions | unlimited on public repos | — | keep the repository public |
| Object storage | none — PDFs stored in Postgres as binary | within DB quota | 0.5 GB | migrate to S3-compatible storage only if needed |

**Rules for this layer:**

- Model identifiers, provider names, and rate limits live in configuration, never hardcoded in call sites.
- `gemini-1.5-*` and `gemini-2.x` are shut down and return 404. Never reference them.
- `temperature`, `top_p`, and `top_k` are deprecated on the Gemini 3.x line. **Do not attempt to pin determinism via sampling parameters.** Non-determinism is a property of the system and must be measured, not suppressed — see §10.
- Free-tier quotas are revised frequently and Google no longer publishes a fixed public table. Before building against any number above, verify it against the live dashboard and update this file.
- Do not adopt Fly.io, Koyeb compute, or Railway free tiers — all have been withdrawn or reduced to trial credits.

---

## 4. Repository layout

```
autodefend/
├── CLAUDE.md                    ← this file
├── README.md                    architecture diagram, demo GIF, live URL, results table
├── docs/
│   ├── adr/                     architecture decision records
│   ├── threat-model.md
│   └── eval-report.md           generated, committed
├── app/
│   ├── main.py                  FastAPI assembly, router mounting, lifespan
│   ├── config.py                typed settings, all env-driven
│   ├── routes/                  webhook, internal, console, health, metrics
│   ├── middleware/              request id, structured logging, auth
│   └── security/                hmac verification, redaction
├── graph/
│   ├── state.py                 the single state schema
│   ├── build.py                 graph assembly, edges, conditional routing
│   ├── nodes/                   one module per node
│   ├── policy.py                pure decision functions — no I/O, no LLM
│   └── checkpoint.py            Postgres checkpointer wiring
├── mcp_servers/
│   ├── logistics/ gateway/ crm/ core/
│   └── common/                  shared error envelope, auth, timeout, models
├── llm/
│   ├── gateway.py               the only module that talks to a provider
│   ├── providers/               gemini, groq adapters behind one interface
│   ├── limiter.py               distributed token buckets
│   ├── cache.py                 content-hash response cache
│   └── ledger.py                token and cost accounting
├── ml/
│   ├── features.py              typed evidence feature schema
│   ├── calibrate.py             isotonic / Platt fitting and persistence
│   └── baseline_tabular.py      sklearn classifier
├── evidence/
│   ├── claims.py                structured claim model with evidence references
│   └── verify.py                deterministic grounding validator
├── render/
│   ├── templates/               representment document layout
│   └── compile.py               PDF generation
├── eval/
│   ├── generate.py              synthetic case generator
│   ├── datasets/                committed cases with splits
│   ├── fixtures/                recorded LLM responses for CI replay
│   ├── baselines/               rules engine, no-tools LLM, majority
│   ├── runner.py                orchestrates all arms
│   └── metrics.py               CIs, calibration, groundedness, cost
├── db/
│   ├── models.py                ORM models
│   └── migrations/              alembic
├── tests/
│   ├── unit/ contract/ integration/
└── infra/
    ├── Dockerfile  render.yaml  .github/workflows/
```

---

## 5. Core invariants

These are non-negotiable. Violating any of them is a bug regardless of whether tests pass.

1. **Money is never a float.** All amounts are integer minor units with an explicit currency field.
2. **The LLM never decides.** LLM calls extract structured features and draft claims. Defend/concede routing is computed by pure functions in `graph/policy.py` that take structured inputs and return structured outputs. Those functions contain no I/O and no model calls.
3. **No claim reaches the output document unless its evidence reference resolves.** Verification is deterministic field resolution, not a second model call.
4. **Every LLM call goes through `llm/gateway.py`.** No provider SDK is imported anywhere else in the codebase.
5. **Every mutation of dispute state is checkpointed** before the next node begins.
6. **Ingestion is idempotent.** The same event delivered N times produces exactly one run, one document, one audit chain.
7. **No raw PAN, full postal address, email address, or signature image is ever placed in an outbound model prompt.** Redaction is enforced at the gateway boundary and covered by a test that fails on pattern match.
8. **The audit log is append-only and hash-chained.** No update or delete paths exist.
9. **MCP tools never raise into the agent.** Failures return a typed error envelope; the graph records the gap and continues.
10. **The test split is touched exactly once**, at the end, after all prompt and threshold work is frozen. No iteration against test numbers.
11. **Reported metrics carry uncertainty intervals.** Bare point estimates on small samples are not acceptable output.
12. **No metric is ever written into a document by hand.** Every number in `README.md` and `eval-report.md` is generated by the eval runner.

---

## 6. Data model

Conceptual tables. Field lists, not DDL — implement with migrations.

| Table | Purpose | Key fields |
|---|---|---|
| `disputes` | one row per dispute event | dispute_id (pk), merchant_id, transaction_id, arn, reason_code, amount_minor, currency, acquirer_deadline_at, received_at, status |
| `processed_events` | idempotency ledger | event_hash (pk), dispute_id, run_id, first_seen_at |
| `runs` | one row per graph execution | run_id (pk), dispute_id, started_at, finished_at, terminal_node, outcome, total_cost_usd |
| `evidence_records` | raw and redacted tool responses | id, run_id, source_server, tool_name, request_hash, payload_raw, payload_redacted, status, latency_ms |
| `decisions` | adjudication outputs | run_id, p_win_raw, p_win_calibrated, ev_threshold, expected_value_minor, decision, decided_at, policy_version |
| `claims` | structured narrative units | id, run_id, text, evidence_ref, exhibit_label, resolved, dropped_reason |
| `documents` | generated packages | id, run_id, format, bytes, sha256, created_at |
| `approvals` | HITL gate | run_id, required, decided_by, decided_at, action, note |
| `audit_log` | tamper-evident trail | seq (pk), run_id, actor, event_type, payload, prev_hash, hash |
| `llm_calls` | cost and token ledger | id, run_id, node, provider, model, prompt_tokens, completion_tokens, latency_ms, cost_usd, cache_hit |
| LangGraph checkpoint tables | managed by the checkpointer | — |

Row-level scoping by `merchant_id` on all merchant-visible tables.

---

## 7. Graph specification

### State

A single typed state object carries: dispute identity and terms (ids, reason code, amount, currency, acquirer deadline); harvested evidence keyed by tool name plus a list of recorded gaps; a typed evidence feature record; raw and calibrated win probability, computed threshold, and decision; the structured claim list with resolved and dropped subsets; the rendered narrative; and an execution trace with accumulated cost.

State is additive. Nodes append to the trace; they never rewrite prior entries.

### Nodes

| Node | Type | Responsibility | Failure behaviour |
|---|---|---|---|
| `triage` | deterministic | Map reason code to a required-evidence manifest via lookup table. **No LLM call** — this saves quota and removes a failure mode. | Unknown reason code → route to `REVIEW` |
| `harvest` | I/O | Concurrently invoke the manifest's MCP tools with per-tool timeouts. Record gaps rather than throwing. | Mandatory evidence missing → route to `REVIEW` |
| `extract` | LLM | Single schema-constrained call producing the typed evidence feature record. Booleans, enums, numerics — no prose. | Schema validation failure → one retry with repair prompt, then `REVIEW` |
| `score` | LLM or model | Produce raw win probability, then apply the fitted calibrator. Two interchangeable implementations behind a flag (see §9). | Calibrator missing → fail loudly, do not silently pass raw |
| `decide` | pure | Compute the per-case EV threshold, compare, route to DEFEND / CONCEDE / REVIEW. Fully unit-testable. | n/a — total function |
| `draft` | LLM | Emit an ordered list of structured claims, each carrying an evidence reference and exhibit label. Not free prose. | Malformed output → retry once, then `REVIEW` |
| `verify` | deterministic | Resolve every evidence reference against harvested evidence. Drop unresolvable claims and count them. Drop rate above threshold → `REVIEW`. | n/a |
| `gate` | interrupt | Pause for human approval per merchant policy. State persists; the run may resume on a different process, later. | Timeout → remains paused, surfaces in console |
| `compile` | deterministic | Render verified claims into the representment document with exhibits, timeline, and header block. | Render failure → retry, then `REVIEW` |
| `emit` | I/O | Persist document, seal the audit chain, hand to the stubbed PSP client. | n/a |

### Routing

- After `harvest`: gaps in mandatory evidence → `REVIEW`, else `extract`.
- After `decide`: `DEFEND` → `draft`; `CONCEDE` → terminal concession log with recorded rationale; `REVIEW` → human queue.
- After `verify`: unsupported-claim rate above policy → `REVIEW`, else `gate`.
- After `gate`: approved → `compile`; rejected → concession log with the reviewer's rationale.

Terminal states are `FILED`, `CONCEDED`, and `REVIEW_PENDING`. There is no silent failure path.

### Document structure

Header block (merchant identity, ARN, case id, disputed amount, reason code) → executive summary asserting why the dispute is invalid → chronological timeline table (order → authentication → dispatch → delivery → post-purchase contact) → exhibit section with explicit labelled references. Every factual assertion in the summary and timeline traces to a verified claim.

---

## 8. MCP server contracts

Four logical servers over streamable HTTP, each bearer-authenticated with its own token.

| Server | Tools | Returns |
|---|---|---|
| `logistics` | proof-of-delivery lookup; carrier telemetry lookup | delivery timestamp, delivery address, carrier status, signature presence and signatory name, geolocation, delivery-agent notes |
| `gateway` | authentication telemetry; pre-auth risk snapshot | AVS result, CVV result, 3DS/OTP outcome, originating IP, device fingerprint, checkout risk score |
| `crm` | customer communications lookup; ticket intent summary | tickets, email threads, chat logs within a window around the transaction; categorised customer intent |
| `core` | account profile; order details | registration date, lifetime spend, prior dispute count, saved addresses; itemised order, digital access and download logs |

**Requirements for every tool:**

- Typed input and output models; validation at the boundary.
- Uniform error envelope with a discriminated code — not-found, timeout, upstream-failure, unauthorised. Never raise.
- Hard timeout, structured logging with request correlation, per-token rate limiting.
- A contract test suite covering happy path, missing identifier, malformed identifier, timeout, and schema conformance for every tool.
- A published manifest so an external MCP client could consume the servers.

**Framing note:** MCP is table stakes in this ecosystem, not a differentiator — every major agent framework speaks it. Do not position it as the project's innovation in the README or in interviews. The innovation is §9 and §10.

### Mock data backing

The four servers read from a seeded relational store with realistic cross-table consistency: orders join to shipments join to carrier events join to tickets join to accounts. Inconsistencies must be *intentional*, injected by the generator (§10.2), never accidental.

---

## 9. Decision layer

### Expected-value threshold

Defending is worth it only when expected recovery exceeds expected penalty. With win probability `p`, dispute amount `A`, and failed-representment penalty `F`, the break-even is:

**p\* = F / (A + F)**

This makes the threshold **per-case**, not global. A small dispute against a fixed penalty needs high confidence; a large dispute justifies defending on thin confidence. A flat cutoff is strictly dominated by this rule and must not be used. Extend the model with the merchant's per-submission operational cost and, optionally, a penalty term for dispute-ratio degradation. The policy version is recorded on every decision row so results remain reproducible when the policy changes.

### Calibration

Raw model confidence is uncalibrated — self-reported probabilities cluster in a narrow high band, so thresholding them is meaningless. Fit a calibrator (isotonic regression, with Platt scaling as the comparison) on the training split, select on the dev split, evaluate on held-out test.

Required artifacts:

- Reliability diagram before and after calibration
- Expected Calibration Error and Brier score, before and after
- Net-recovered-revenue plotted against threshold, demonstrating that the EV threshold sits at or near the empirical optimum

This is the project's genuine machine learning content: a fitted model, a proper split discipline, and a persisted artifact with a version.

### Scoring implementations

Two interchangeable scorers behind one interface, both calibrated the same way:

- **LLM scorer** — judgment from the evidence feature record.
- **Tabular scorer** — a gradient-boosted or logistic model over the same features, yielding feature importances.

Both are evaluated. If the tabular scorer wins on classification, report that. See §10.4.

---

## 10. Evaluation protocol

### 10.1 Arms

| Arm | Description | Purpose |
|---|---|---|
| B0 | Majority class | Floor |
| B1 | Hand-written rule engine over evidence features | The baseline reviewers will demand. If the agent cannot beat it, that finding is the result. |
| B2 | Tabular classifier over the same features | Real ML baseline with interpretable importances |
| B3 | LLM on the raw dispute payload, no tools | Isolates the value of retrieval |
| B4 | Full agent | The system |

All five appear in one table in `eval-report.md` and the README.

### 10.2 Dataset

Approximately 150 cases, generated programmatically. Composition:

- Friendly fraud, true fraud, and ambiguous cases (family purchase, damaged return without authorisation, delivered without signature) in roughly a 5:5:3 ratio
- A dedicated **adversarial subset** engineered so the rule engine is confidently wrong: valid delivery on a genuinely compromised card; address-verification mismatch caused by a legitimate address change; delivery signed by a household member
- Injected degradation: a fraction of cases with tool timeouts, partial evidence, or mutually contradictory signals

**Critical constraint:** ground-truth labels come from the generator's latent state and must **not** be derivable from the evidence surface by the rules stated in this document. If a reader could label a case correctly using only §9's decision logic, the case teaches nothing.

Split: train (calibrator and tabular fitting) / dev (threshold selection, prompt iteration) / test (evaluated once, at the end). Splits are committed and stratified by class and by amount band.

### 10.3 Metrics

- Precision, recall, F1 — **each with a 95% interval** (Wilson or bootstrap). At this sample size the interval is wide; reporting the interval is the point.
- Net recovered revenue against each baseline — the headline business metric
- Calibration: ECE, Brier, reliability diagram
- **Unsupported-claim rate** — target zero — and exhibit-reference accuracy
- **Run-to-run variance**: repeat each case across several runs, report mean and standard deviation and a self-consistency rate. This is mandatory because sampling parameters are no longer available to pin determinism.
- Cost and latency per dispute, decomposed per node
- Confusion matrix plus a **manual error taxonomy** over every test-set failure, categorised as retrieval miss, extraction error, reasoning error, or fabrication

### 10.4 Ablations

Remove CRM tools; remove logistics tools; remove calibration; remove the verifier; single prompt instead of the graph. One table, one row each.

### 10.5 The honest finding

Expect B1 and B2 to be competitive with B4 on classification accuracy. If that happens, report it plainly and state the defensible thesis: **the agent's value lies in autonomous cross-system retrieval and grounded document generation, not in the classification decision, which a calibrated tabular model handles more cheaply and more predictably.** Do not tune the benchmark until the agent wins. An honest negative result, well measured, is stronger evidence of judgment than an inflated score.

---

## 11. LLM gateway requirements

The gateway is the only module that touches a provider, and much of the backend engineering signal lives here.

- **Distributed token buckets** in Redis tracking requests-per-minute, tokens-per-minute, and requests-per-day. Process-local limiters are incorrect the moment a second worker exists.
- **Exponential backoff with jitter** on rate-limit responses, distinguishing per-minute exhaustion (short wait, retry) from per-day exhaustion (fail fast, resets at the provider's daily boundary). Both surface as the same HTTP status with different bodies.
- **Provider fallback chain** behind a single interface: primary Gemini, secondary Groq. Free model catalogues are pruned without notice and a hardcoded identifier can start returning 404 between runs, so the service performs a **startup healthcheck against every configured model** and fails loudly on a missing one.
- **Content-hash response cache** in Redis. On repeated evaluation runs this is what makes the workload fit inside the daily request ceiling.
- **Cost ledger write on every call**, recording node, provider, model, token counts, latency, cost, and cache-hit status. Cost per dispute is a far better headline than a time-saving percentage.
- **Optional:** circuit breaker per provider. Add only after the above are complete.

---

## 12. Observability, security, and operations

**Observability.** Langfuse traces spanning the whole run with per-node attribution. Structured JSON logs correlated by run identifier. A liveness endpoint and a readiness endpoint that checks database, cache, and MCP reachability. A metrics endpoint exposing throughput, decision distribution, per-node latency percentiles, tool error rate, rate-limit rate, and cost per dispute.

**Security.** Constant-time HMAC verification on the webhook with a timestamp-skew rejection window for replay protection. Bearer tokens per MCP server. Row-level scoping by merchant. Redaction at the gateway boundary reducing card numbers to last-four, addresses to a locality plus match flag, emails to hashes, and excluding signature images entirely — they are referenced by exhibit label and embedded only during local document rendering. A hash-chained append-only audit log. All secrets from environment configuration, with a committed example file and no committed values.

**Free-tier operational hazards.** Idle databases pause; schedule a periodic keep-alive and warm the stack before any live demonstration. Free tiers may use submitted prompts for model improvement; note this in the threat model — the redaction layer is what makes it acceptable.

---

## 13. Conventions

- Python with full type annotations; strict type checking in CI. Pydantic models at every boundary — HTTP, MCP, LLM output, database.
- One responsibility per module. Nodes are thin; logic lives in pure functions that are unit-testable without a network.
- Errors are values at boundaries and exceptions internally. Never swallow an exception without recording it in the trace.
- All configuration through a single typed settings object. No environment lookups scattered through the codebase.
- Structured logging only. No bare print statements.
- Every new external dependency requires justification in the pull request description.
- Tests: unit for pure logic, contract for MCP tools, integration for the full graph against seeded data with replayed LLM fixtures.
- Conventional commit messages. Small, reviewable pull requests.
- Architecture decision records for: the queue-based ingestion design, durable checkpointing, expected-value thresholding over a fixed cutoff, and deterministic verification over model-based judging. Reviewers read ADRs and almost no student project has them.

---

## 14. Command surface

The following must exist and be documented in the README. Implement as Makefile targets or a task runner.

| Command | Purpose |
|---|---|
| `setup` | install dependencies, provision local environment |
| `db.migrate` / `db.seed` | apply migrations; load mock backing data |
| `dev` | run the service locally with reload |
| `test.unit` / `test.contract` / `test.integration` | the three test tiers |
| `eval.generate` | build the synthetic dataset and splits |
| `eval.run` | execute all arms, write results |
| `eval.report` | render the metrics document from results |
| `ml.calibrate` | fit and persist the calibrator |
| `lint` / `typecheck` | static checks |
| `deploy` | build and release |

---

## 15. Phases

Phases are sequential. **Do not begin a phase until the previous phase's exit criteria are met and verified.** No phase is complete because code exists; it is complete when its criteria are demonstrably satisfied.

### Phase 0 — Foundation

Repository skeleton, dependency and tooling setup, typed configuration, containerisation, database migrations, provisioning of database / cache / queue / model accounts, health and readiness endpoints, CI running lint and type checks on every push.

**Exit:** the service builds, starts, connects to all external dependencies, reports ready, and CI is green on an empty test suite.

### Phase 1 — Evidence layer

The four MCP servers with full tool contracts, typed models, uniform error envelopes, timeouts, and bearer authentication. Seeded mock backing store with cross-table consistency. Complete contract test suite. MCP client pool in the application with concurrent invocation and per-tool timeouts.

**Exit:** every tool returns valid typed responses for happy path and every failure mode; contract tests pass; the client pool harvests a full evidence set concurrently for a sample dispute.

### Phase 2 — Inference layer

The LLM gateway with provider abstraction, distributed rate limiting, backoff, fallback chain, startup model healthcheck, response cache, redaction boundary, and cost ledger.

**Exit:** the gateway survives a forced rate-limit storm without dropping work; failover to the secondary provider is demonstrated; the redaction test proves no sensitive pattern reaches an outbound prompt; cost rows are written for every call.

### Phase 3 — Orchestration

The state schema, all graph nodes, conditional routing, Postgres checkpointing, and the human-in-the-loop interrupt. Pure decision functions in the policy module with exhaustive unit tests. Idempotent ingestion path, webhook signature verification, queue publication and internal consumption, audit chain.

**Exit:** a dispute flows end to end to a terminal state; the same event delivered repeatedly produces exactly one run; the process can be killed mid-run and resumes at the last completed node rather than restarting; a paused run resumes correctly after a restart.

### Phase 4 — Decision science

Synthetic case generator with controlled confounders and the adversarial subset; committed dataset and splits; the rule-engine, majority, tabular, and no-tools baselines; the evaluation runner; the metrics module with intervals, calibration measures, and variance; calibrator fitting and persistence; the expected-value threshold wired into the decision node.

**Exit:** all five arms produce results on the dev split; calibration improves ECE measurably; the net-revenue-versus-threshold curve is produced; the threshold in use is computed, not chosen.

### Phase 5 — Generation and verification

Structured claim generation, deterministic evidence-reference resolution, drop accounting, document templating with header block, executive summary, timeline table and labelled exhibits, PDF compilation, document persistence, and the stubbed submission client.

**Exit:** unsupported-claim rate is zero on the dev split; a fabricated reference is provably dropped rather than rendered; a complete document is generated and retrievable.

### Phase 6 — Surface and evidence of quality

Server-rendered console for the dispute queue, evidence inspection, decision explanation, approval action, and results dashboard. Tracing, metrics endpoint. CI extended with the integration tier and the **evaluation regression gate** — the dev-split evaluation replayed from recorded fixtures, failing the build on an F1 regression beyond tolerance or any unsupported claim. Held-out test evaluation run once. Ablations, variance runs, error taxonomy. Generated evaluation report, ADRs, threat model, README with architecture diagram and live URL, demonstration recording, production deployment.

**Exit:** the service is publicly reachable; CI gates merges on evaluation quality; every number in the README traces to generated output; the demonstration shows crash-resume, duplicate suppression, a rejected fabricated claim, and the approval gate.

### If scope must be cut

Phases 0 through 5 are the contract. Within Phase 6, priority order is: evaluation regression gate, evaluation report and README, tracing and cost dashboard, console, ablations. Cut from the end.

---

## 16. Definition of done

- Publicly reachable deployed URL with working health and readiness endpoints
- End-to-end flow demonstrable from webhook to generated document
- Crash mid-run resumes rather than restarts
- Duplicate delivery produces exactly one run and one document
- A fabricated evidence reference is provably dropped before rendering
- Human approval gate pauses and resumes across a process restart
- Five evaluation arms reported with intervals, calibration curves, variance, ablations, and an error taxonomy
- CI gates merges on lint, types, all three test tiers, and evaluation regression
- Four ADRs, a threat model, and a generated evaluation report committed
- README carries the architecture diagram, the live URL, the results table, and a demonstration recording
- No number anywhere in the documentation was typed by hand

---

## 17. Risk register

| Risk | Impact | Mitigation |
|---|---|---|
| Daily model request quota exhausted mid-evaluation | Evaluation blocked until reset | Response cache, recorded fixtures for CI replay, secondary provider, full evaluations run deliberately rather than casually |
| Free model silently removed from provider catalogue | Silent pipeline death | Startup healthcheck across all configured models; provider abstraction makes substitution a config change |
| Compute cold start | Dropped webhooks | Queue-based ingestion with retry — already the core design |
| Database idle pause | Dead demonstration | Scheduled keep-alive; warm the stack before any live demo |
| Queue daily message cap | Bulk replay blocked | Evaluation invokes the graph directly, bypassing the queue |
| Single instance hosting all surfaces | Coupled failure domain | Documented as a deliberate constraint with a stated split path |
| Agent fails to beat the rule baseline | Perceived project failure | Reframed in advance as the honest finding — see §10.5 |
| Scope creep | Nothing finished | Phases 0–5 are the contract; Phase 6 is cut from the end |

---

## 18. Working agreement for Claude Code

- Read this file before any change. Re-read §5 before touching the graph, policy, or gateway.
- Confirm the current phase before starting work. Do not build forward into an unstarted phase.
- Never introduce a new external service or dependency without stating the justification and the free-tier implications.
- Never hardcode a model identifier, provider name, threshold, or rate limit outside configuration.
- Never write a metric into documentation by hand, and never invent placeholder numbers that could be mistaken for results.
- When a design question is genuinely ambiguous, ask rather than guessing — then record the answer as an ADR or an update to this file.
- When this file becomes inaccurate, updating it is part of the change, not a follow-up task.