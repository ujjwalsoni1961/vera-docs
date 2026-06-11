# Vera — Validation & Operational Evidence

This document summarizes the evidence behind Vera's technical readiness
claim: a complete system deployed and proven in an operational environment
with paying customers (TRL 8). It covers production deployments, the
language-layer evaluation suite, solver verification, and the operational
results observed at customers.

Customer identities are withheld here under the confidentiality terms of our
data processing agreements; they can be disclosed to programme reviewers on
request.

## 1. Production deployments

| | Deployment A | Deployment B |
| --- | --- | --- |
| Segment | Building-systems field service (heating & ventilation) | Field service (refrigeration & commercial kitchens) |
| Country | Finland | Finland |
| In production since | 2025 | 2025 |
| Scope | Daily dispatch, routing, disruption handling | Dispatch and SLA-driven emergency response |
| Users | Operations team incl. dispatchers and field technicians | Operations team |
| Autonomy in use | Approve (reassignment, routing), Autonomous (customer ETA notifications) | Suggest → Approve (progressive rollout) |
| Commercial status | Paying subscription | Paying subscription |

Field-service execution and logistics execution are the same class of
decision problem — allocating constrained resources to time-critical jobs
under changing conditions and explicit business rules — which is why this
evidence carries to the logistics target segment.

Combined commercial evidence: €10,000 in revenue from these two production
deployments; over €30,000 cumulative revenue from operations since 2025
(operations began under the founder's Finnish sole proprietorship and were
incorporated as Nex0 Oy in January 2026; product, team and customers are
continuous).

## 2. What "in production" means operationally

- Every working day, real dispatch and disruption decisions for both
  customers flow through the full pipeline: ingestion → language layer →
  memory → Z3 verification → agent action → audit record.
- Customer operators author and maintain their own rule sets in plain
  English. Rule catalogues at both deployments are operator-maintained
  without engineering involvement — rule changes ship in minutes, not
  change-request cycles.
- 100% of decisions carry a complete reasoning trace; audit exports have
  been used by one customer in an SLA dispute with their own end customer.
- Routine disruptions (technician absence, vehicle breakdown, emergency
  insertion) are resolved in minutes; the same situations previously waited
  on a senior dispatcher's availability.

## 3. Language-layer evaluation suite

The riskiest link in any LLM-adjacent system is extraction quality. Vera
confines the LLM to schema-constrained translation and measures it
continuously with a purpose-built evaluation suite, which is also the
instrument we will use to benchmark European models for the inference
migration.

**Method.** A corpus of operator-authored rules and real (anonymized)
disruption messages from production, each with a hand-verified reference
extraction. Candidate models translate every item; output must parse against
the Pydantic schemas; parsed output is compared field-by-field to reference.

**Metrics.**

| Metric | Definition |
| --- | --- |
| Schema validity | % of outputs parsing on first attempt (before repair) |
| Extraction accuracy | % of items with all fields matching reference |
| Repair recovery | % of first-pass failures fixed by the automatic repair pass |
| Residual to review | % of items routed to human review after repair |

**Indicative results, current production configuration** (rule-translation
subset, n = 412 items):

| Metric | Result |
| --- | --- |
| Schema validity (first pass) | 96.8% |
| Extraction accuracy | 94.2% |
| Repair recovery | 78% of first-pass failures |
| Residual to review | 1.7% |

Two properties matter more than the absolute numbers. First, no unvalidated
extraction can reach the solver — schema validation is structural, so an
extraction failure degrades to human review, never to a wrong decision.
Second, the suite makes model swaps safe: a candidate model must match or
beat the incumbent on this suite before cutover.

## 4. Solver verification and determinism

The reasoning core is exercised by a regression suite that runs on every
engine release:

- **Determinism.** Golden-trace tests: a fixed set of workspace states ×
  tasks must produce byte-identical decisions and traces across runs,
  platforms and releases. Any diff blocks the release.
- **Unsat-core correctness.** For every scenario with a known violation, the
  solver must name exactly the violating rule set — no over- or
  under-approximation. This is what makes rejections auditable.
- **Constraint coverage.** Every rule-scope category in production
  (certifications, working time, SLA windows, parts, site access, customer
  communication) has positive and negative scenario coverage.
- **Performance.** Decision evaluation on reference workspaces (≈ 10 rules,
  ≈ 12 active jobs, ≈ 8 technicians) completes in tens of milliseconds;
  solver time grows with rule and plan count, not with prose length, and a
  hard timeout (`VERA_SOLVER_TIMEOUT_MS`) degrades to suggest mode with a
  partial trace rather than blocking operations.

## 5. Demonstration environment

A complete, self-contained demonstration of the decision path runs at
https://vera.nex0.tech — a field-service
demonstration workspace with realistic Finnish operational geography,
running the live LLM-translation + Z3-verification pipeline. (Customer
workspaces cannot be exposed publicly under our data-processing agreements.) Reviewers can:

1. Submit a free-form operational task and watch the reasoning trace stream,
   including the solver-verification step naming the rules checked.
2. State a policy in conversation and see it captured as a structured rule
   (memory capture path).
3. Import rules from a file and confirm the parsed structured forms
   (import path).
4. Inspect the knowledge graph, the audit trail, and the developer surface.

The demonstration intentionally contains no customer data; production
behaviour beyond it (multi-store memory, full autonomy loop, connectors) is
documented in [architecture.md](architecture.md) and verifiable in a guided
session with the team.

## 6. Readiness summary

| Evidence class | Status |
| --- | --- |
| Complete system in operational environment | Two Finnish field-service operators, daily production use since 2025 |
| Commercial validation | Paying subscriptions; €10k from production deployments, €30k+ cumulative since 2025 |
| Technical operations | Docker-packaged services on EU-region cloud; REST + MCP in live production use |
| Quality instrumentation | Extraction evaluation suite; determinism and unsat-core regression suites; immutable audit trail |
| External verifiability | Live demonstration console + this documentation set |
