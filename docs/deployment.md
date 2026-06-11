# Vera — Deployment & Operations

Vera ships as Docker images per service and deploys in three patterns:
Nex0-hosted (EU region, default), customer cloud, or on-premise for full
sovereignty. The reasoning core has no external dependency at decision time —
deployments differ only in where containers run and which model endpoint the
language layer uses.

## 1. Services

| Image | Service | Notes |
| --- | --- | --- |
| `vera/gateway` | API gateway | AuthN/Z, rate limiting, REST surface |
| `vera/engine` | Reasoning engine | Z3 solver, plan generation, ranking (private image) |
| `vera/agent` | Agent runtime | Orchestration loop, MCP server, connectors |
| `vera/console` | Operator console | Proprietary (private repository) |
| `postgres:16` | Relational store | Rule catalogue, decisions, audit |
| `neo4j:5` | Knowledge graph | Entities and relationships |
| `qdrant/qdrant` | Vector index | Semantic rule recall |

## 2. Single-host deployment (docker-compose)

Reference compose file (abridged; the full file ships with the distribution):

```yaml
services:
  gateway:
    image: vera/gateway:1.4
    ports: ["443:8443"]
    environment:
      VERA_WORKSPACE: acme-field-ops
      VERA_DB_URL: postgres://vera@postgres:5432/vera
    depends_on: [engine, postgres]

  engine:
    image: vera/engine:1.4
    environment:
      VERA_SOLVER_TIMEOUT_MS: "5000"

  agent:
    image: vera/agent:1.4
    environment:
      VERA_MODEL_ENDPOINT: ${VERA_MODEL_ENDPOINT}   # see §4
      VERA_AUTONOMY_DEFAULT: approve

  console:
    image: vera/console:1.4
    environment:
      NEXT_PUBLIC_ENGINE_MODE: live

  postgres:
    image: postgres:16
    volumes: [pg_data:/var/lib/postgresql/data]

  neo4j:
    image: neo4j:5
    volumes: [graph_data:/data]

  qdrant:
    image: qdrant/qdrant
    volumes: [vector_data:/qdrant/storage]
```

Kubernetes-ready manifests (one Deployment per service, PVCs for the three
stores) are provided for scale-out; the services are stateless apart from the
stores, so horizontal scaling is routine.

## 3. Sizing

Vera's core services are lightweight. Reference sizing for a production
workspace (one operational site, ~10 operators, < 10,000 inference calls per
month):

| Resource | Requirement |
| --- | --- |
| CPU | 4–8 vCPU total across services |
| Memory | 16–32 GB |
| Storage | 50–100 GB (audit history dominates growth) |
| GPU | None required when inference is served by an external endpoint; optional for self-hosted open-weight models |

Solver evaluation of a typical dispatch decision (≈ 10 rules, ≈ 12 active
work orders) completes in tens of milliseconds; the engine container is
CPU-bound and benefits from per-core performance, not core count.

## 4. Language-layer endpoints

The language layer sits behind a provider-agnostic abstraction; switching
models is a configuration change plus an evaluation cycle, not a
re-architecture:

```
VERA_MODEL_ENDPOINT=https://...      # OpenAI-compatible chat endpoint
VERA_MODEL_NAME=...
VERA_MODEL_API_KEY=...               # secret-managed
```

Supported configurations in production and evaluation:

- Open-weight European models (Mistral, Teuken class), self-hosted via
  Ollama or vLLM — the sovereign configuration.
- Hosted API models approved by the customer.
- The schema-constrained extraction evaluation suite
  ([validation.md](validation.md), §3) is run before any cutover; the solver
  is unaffected by model choice.

Only prompt text leaves the deployment toward the model endpoint. With a
self-hosted model, no data leaves the deployment at all.

## 5. Hosting and regions

- Nex0-hosted workspaces run on EU-region cloud exclusively; data residency
  is contractual, not best-effort.
- Backups: continuous WAL archiving for PostgreSQL, daily graph and vector
  snapshots, 30-day retention by default; restores are rehearsed quarterly.
- Observability: structured logs, per-decision trace IDs end to end, and
  health endpoints (`/healthz`) per service.

## 6. The demonstration deployment

The demonstration console (https://vera.nex0.tech) is a
self-contained build of the operator console plus a console-hosted
reasoning route running the same LLM-translation + Z3-verification pipeline
against a dedicated demonstration workspace. It exists so the full decision
path can be exercised by evaluators without touching customer workspaces (sign-in gated; credentials on
request), and deliberately runs serverless —
no graph or relational store — which is why workspace state there is
browser-scoped. Production deployments use the full service layout above.

## 7. Upgrade policy

Images are versioned (semver); the engine and gateway are released in
lockstep, console and agent independently. Rule catalogues are migrated
forward automatically; decision-trace formats are append-only — old audit
records remain readable by every later version. See
[CHANGELOG.md](../CHANGELOG.md).
