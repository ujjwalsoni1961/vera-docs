# Changelog

Product release history. Engine and gateway are versioned in lockstep;
console and agent ship independently but are listed here under the platform
release they accompanied. Dates are release-to-production dates.

## 1.4 — June 2026

- Public documentation set released, with a hosted demonstration
  deployment (dedicated demonstration workspace, sign-in gated) exercising the full
  decision path: live LLM translation + Z3 verification.
- Result widgets in the agent thread: route changes render on an interactive
  map (previous vs. proposed path, distance and drive-time deltas);
  reassignments and SLA reviews render as structured widgets. Applied
  decisions retain their widget in the executed state.
- Conversational rule capture: policies stated in the agent thread are
  detected, parsed to structured form and proposed as rulebook additions
  (operator confirmation required, as with every rule path).
- Rule import: from connected systems and from CSV/text files, with
  per-rule review before activation.
- Language layer: model configuration moved fully behind the
  provider-agnostic endpoint abstraction; reasoning-latency budget enforced
  per deployment.

## 1.3 — March 2026

- Rule source tracking: every rule carries its origin (workspace, import,
  conversation) and full version history with author and timestamp.
- Knowledge-graph memory extended with certification and site-access
  entities; semantic recall tuned for mixed Finnish/English rule corpora.
- Audit export: structured JSON export of decision traces for external
  compliance tooling (first production use in a customer SLA dispute).

## 1.2 — January 2026

- MCP server shipped: `vera_reason`, `vera_evaluate_plan`,
  `vera_parse_rule`, `vera_get_rules`, `vera_query_memory` exposed as tools
  for third-party agent frameworks.
- Autonomy configuration per decision class (suggest / approve /
  autonomous) with approve as default; autonomous execution for customer
  ETA notifications enabled at the first production customer.
- Nex0 Oy incorporation: operations transferred from the founder's sole
  proprietorship; no product or customer interruption.

## 1.1 — November 2025

- Second production customer live (refrigeration & commercial-kitchen field
  service): dispatch and SLA-driven emergency response.
- Unsat-core explanations surfaced end-user-readable: every rejected option
  names the violating rule and the facts behind it.
- Extraction evaluation suite v2: production-derived corpus, per-language
  subsets, release-gating thresholds.

## 1.0 — September 2025

- First production release. Daily dispatch, routing and disruption handling
  at a Finnish building-systems field-service operator.
- Deterministic decision engine on Z3 with golden-trace regression suite.
- Plain-English rulebook with operator-confirmed structured forms.
- Immutable decision log with complete reasoning traces.

## 0.x — 2025 (pre-1.0)

- 0.4 (July 2025): supervised pilot at the first customer; suggest mode
  only; rule-capture workshops with the operations team.
- 0.3 (May 2025): schema-validated extraction with automatic repair pass;
  Pydantic schema set frozen for pilot.
- 0.2 (April 2025): Z3 integration replacing the prototype rule evaluator;
  unsat cores adopted as the explanation mechanism.
- 0.1 (February 2025): internal prototype — LLM translation chained to a
  constraint check over a hand-built workspace model.
