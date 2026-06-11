# Vera — System Architecture

This document describes the architecture of the Vera platform as deployed in
production. The operator console is one of three delivery
surfaces (UI, REST, MCP); all three sit on the same engine contract.

## 1. Design principle

One rule governs the whole system: **the language model never makes the final
decision.** The LLM is confined to translation — natural language in,
structured facts and constraints out; solver results in, readable reasoning
steps out. Every decision is computed by a deterministic constraint solver
against the customer's rulebook. This is what makes Vera's output verifiable
by construction rather than explained after the fact.

## 2. Component overview

```
                ┌─────────────────────────────────────────────────┐
                │                  Delivery                       │
                │   Operator console · REST · MCP             │
                └────────────────────┬────────────────────────────┘
                                     │
┌──────────────┐    ┌────────────────▼─────────────────┐    ┌──────────────────┐
│  Ingestion   │    │       Agent / orchestrator       │    │  Language layer  │
│  REST pulls  ├───►│  observe → reason → decide → act ├───►│  LLM + Pydantic  │
│  from FSM /  │    │  autonomy: suggest / approve /   │◄───┤  schema-validated│
│  TMS / ERP   │    │  autonomous (per decision class) │    │  translation     │
└──────────────┘    └───────┬──────────────────┬───────┘    └──────────────────┘
                            │                  │
                  ┌─────────▼────────┐  ┌──────▼───────────────┐
                  │  Reasoning core  │  │       Memory         │
                  │  Z3 constraint   │  │  Neo4j graph ·       │
                  │  solver, unsat   │  │  PostgreSQL ·        │
                  │  cores, ranking  │  │  vector index        │
                  └──────────────────┘  └──────────────────────┘
```

### 2.1 Data ingestion

Vera connects to the customer's operational system of record (field-service
management, TMS, ERP) over REST and pulls jobs, resources, routes, status and
timing. No schema migration is required on the customer side; Vera maintains
its own normalized view. EDI and email connectors are on the integration
roadmap.

### 2.2 Language layer

Provider-agnostic LLM abstraction with strict schema validation:

- **Inbound translation.** Plain-English rules and free-text disruption
  messages are converted to typed, structured form. Output is validated
  against Pydantic schemas; malformed output triggers an automatic repair
  pass; persistently low-confidence extractions are routed to human review
  instead of entering the decision path.
- **Outbound translation.** Solver results (satisfied constraints, unsat
  cores, rankings) are rendered as readable reasoning steps with references
  to the exact rules and facts used.

The layer is model-agnostic: open-weight European models (Mistral, Teuken
class) self-hosted via Ollama, or any API model the customer approves. See
[model-card-language-layer.md](model-card-language-layer.md).

### 2.3 Memory

Three coordinated stores:

| Store | Holds |
| --- | --- |
| Neo4j knowledge graph | Entities (technicians, vehicles, sites, jobs, rules) and their relationships; what the agent "knows" and how facts connect |
| PostgreSQL | Versioned rule catalogue (author + timestamp per change), decision history, audit records |
| Vector index | Semantic recall: retrieving the rules and precedents relevant to the task at hand |

Rules enter memory through three paths, all converging on the same versioned
catalogue: authored in the rulebook UI, imported from connected systems or
files, or captured from operator statements in conversation (proposed by the
agent, confirmed by the operator before activation).

### 2.4 Reasoning core (Z3)

The decision engine encodes the rulebook and the candidate plans as
constraints and uses the Z3 SMT solver to:

1. **Check feasibility** of every candidate plan against every active rule.
2. **Name violations exactly.** For infeasible plans, the unsat core
   identifies the minimal set of conflicting rules and facts — the rejection
   is a proof, not a narrative.
3. **Rank valid options** by the customer's configured objectives (SLA risk,
   travel time, working-time headroom).

Determinism is a hard guarantee: identical input (facts + rulebook version)
produces an identical decision and trace. This is exercised continuously by
the regression suite (see [validation.md](validation.md)).

### 2.5 Agent / orchestrator

The agent runs an observe → reason → decide → act loop and executes outcomes
through MCP tool calls: route optimization, customer notifications, writes
back to the operational system. Autonomy is configured per decision class:

| Mode | Behaviour |
| --- | --- |
| Suggest | Decision and trace presented; operator executes manually |
| Approve | Decision prepared and executed on one-click operator approval (default) |
| Autonomous | Routine, low-risk decision classes execute directly; logged identically |

Every executed action — regardless of mode — is written to the audit log
with its full reasoning trace.

### 2.6 Delivery

All services ship as Docker images (API gateway, reasoning engine, knowledge
graph store, agent runtime) with docker-compose for single-host deployments
and Kubernetes-ready manifests for scale-out. The product is consumed three
ways:

1. **Operator console** — the daily UI for operations teams.
2. **REST API** — `POST /v1/reason`, plan evaluation, rule parsing.
3. **MCP server** — the same operations exposed as tools, so third-party
   agents can delegate constraint checking to Vera.

See [deployment.md](deployment.md).

## 3. Decision path (end to end)

1. Trigger: operator task in plain language, or an ingested disruption event.
2. Language layer extracts structured intent and facts (schema-validated).
3. Memory assembles the decision context: relevant rules (vector recall +
   scope filters), entities and relationships from the graph.
4. Candidate plans are generated from the operational state.
5. Z3 checks every candidate against every rule; infeasible plans get unsat
   cores; feasible plans are ranked.
6. Language layer renders the trace: steps, references, checks.
7. Agent acts per the autonomy configuration.
8. Decision, trace, rulebook version and outcome are stored permanently.

Failure containment at each stage: schema validation failures trigger repair
or human review (stage 2); solver timeouts fall back to suggest-mode with the
partial trace (stage 5); action failures roll back and re-queue (stage 7). A
language-layer outage degrades Vera to structured-input mode — the solver,
and therefore decision integrity, is unaffected by model availability.

## 4. The operator console

Next.js/TypeScript application. Its main surfaces:

- **Engine contract** — shared typed definitions for Decision, Rule, Entity
  and result widgets; the UI cannot drift from the engine schema.
- **Engine facade** — a single typed binding with two modes: `live` (REST to
  the reasoning pipeline) and `mock` (seeded in-process engine with identical
  types, used for UI development and automated tests).
- **Console-hosted API** — reason, rules, memory and audit endpoints.
- **Operator UI** — agent thread, reasoning panel, rulebook, memory graph,
  audit log, developer surface, workspace settings.

The console binds to the engine through a single typed facade. `live` mode
calls the reasoning pipeline over REST; `mock` mode runs a seeded in-process
engine with identical types — used for UI development, automated tests and
offline demonstrations. The demonstration deployment runs a
dedicated workspace — a Finnish building-systems field-service operation —
so that the full decision path can be exercised without exposing customer
workspaces.

## 5. Multi-tenancy and isolation

Customer workspaces are isolated end to end: scoped databases and graph
namespaces, per-workspace encryption keys, and rule catalogues that never
cross tenant boundaries. Only prompt text reaches inference endpoints; see
[security-and-data-governance.md](security-and-data-governance.md).
