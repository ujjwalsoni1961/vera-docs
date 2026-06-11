# Vera — API Reference

Vera exposes its reasoning engine through a REST API and an MCP server. Both
surfaces share the same engine contract and return the same `Decision`
structure the operator console renders.

Base URL: customer-specific, e.g. `https://<workspace>.vera.nex0.tech/api`.
The demonstration console hosts a rate-limited subset at
`https://vera.nex0.tech/api/v1` against the demonstration workspace.

## Authentication

Production workspaces authenticate with scoped API keys:

```
Authorization: Bearer vera_sk_...
```

Keys are issued per workspace and per integration, carry role-based scopes
(`reason`, `rules:read`, `rules:write`, `audit:read`), and can be rotated and
revoked in workspace settings. All traffic is TLS 1.2+.

## REST API

### POST /v1/reason

Submit an operational task in plain language. Returns a verified decision
with its complete reasoning trace.

Request:

```json
{
  "task": "Mikko called in sick. Reassign his jobs for today.",
  "context": { "workspace_time": "2026-06-10T07:55:00+03:00" }
}
```

Response (abridged):

```json
{
  "id": "dec_01J9XK...",
  "kind": "task",
  "status": "recommended",
  "summary": "Mikko Virtanen is out sick with 3 assigned jobs today...",
  "action": "Reassign WO-4811 to Anna Mäkelä, WO-4815 to Sami Nieminen...",
  "steps": [
    {
      "text": "Pulled today's board: 3 open assignments for Mikko Virtanen...",
      "refs": ["WO-4811", "WO-4815", "WO-4819"]
    },
    {
      "text": "Solver verification (z3-solver): R3 — F-gas certification: satisfied; R4 — working day ≤ 10 h: satisfied.",
      "refs": ["R3", "R4"]
    }
  ],
  "checks": [
    { "rule": "R3", "kind": "rule_check", "result": "satisfied" },
    { "rule": "R4", "kind": "working_hours", "result": "satisfied" }
  ],
  "widget": { "kind": "assignments", "title": "Reassigned jobs", "...": "..." },
  "rulebookVersion": "2026-06-09T14:02:11Z",
  "trace_id": "tr_01J9XK..."
}
```

Semantics:

- `status` is `recommended` (requires approval), `applied` (autonomous
  execution), or `informational`.
- Every step that depends on a rule or entity carries `refs` resolvable via
  the memory API — this is the audit-grade reasoning trace.
- The solver-verification step is always present for decisions; it reports
  the result of the Z3 check per applicable rule.
- Identical `task` + identical workspace state + identical rulebook version
  ⇒ identical response (determinism guarantee; see validation.md).

### POST /v1/plans/evaluate

Check an explicit candidate plan against the active rulebook. This is the
endpoint third-party systems use to add formal verification to their own
planning.

```json
{
  "plan": {
    "assignments": [
      { "job": "WO-4811", "technician": "T-103", "window": ["13:00", "15:00"] }
    ]
  }
}
```

Response: per-rule SAT/UNSAT, and for UNSAT plans the unsat core — the
minimal set of rules and facts that conflict:

```json
{
  "feasible": false,
  "results": [
    { "rule": "R2", "result": "violated" }
  ],
  "unsat_core": {
    "rules": ["R2"],
    "facts": ["T-103.certifications", "WO-4811.equipment_class"],
    "explanation": "R2 requires a Tukes gas certification for gas-boiler work; T-103 holds none."
  }
}
```

### POST /v1/rules/parse

Translate a plain-English rule into its structured, machine-checkable form.
Used by the rulebook UI, file import, and conversational rule capture — the
operator always confirms the structured form before the rule activates.

```json
{ "text": "Parts over 500 euros require service manager approval." }
```

```json
{
  "naturalLanguage": "Parts over 500 euros require service manager approval.",
  "structured": "rule R_parts_approval: job.part.cost > 500 → require approval(job.part, role=service_manager)",
  "scope": "Parts & inventory",
  "confidence": 0.97
}
```

### Rules, memory, audit

```
GET  /v1/rules               Active rulebook (natural language + structured form + source + version)
POST /v1/rules               Add a confirmed rule        (rules:write)
GET  /v1/memory/graph        Workspace knowledge graph (entities + relationships)
GET  /v1/audit               Decision log; filter by date, decision class, status
GET  /v1/audit/{id}          Single decision with full trace (export: JSON)
```

Audit entries are immutable and exportable; each contains the decision, the
trace, the rulebook version in force, the autonomy mode, and the acting
operator (for approve mode).

## MCP server

The same operations are exposed over the
[Model Context Protocol](https://modelcontextprotocol.io) so agent frameworks
can delegate constraint checking and verified decision-making to Vera.

Server: `vera-mcp` (stdio or SSE transport; Docker image ships both).

| Tool | Maps to | Purpose |
| --- | --- | --- |
| `vera_reason` | `POST /v1/reason` | Verified decision with trace for a natural-language task |
| `vera_evaluate_plan` | `POST /v1/plans/evaluate` | SAT/UNSAT + unsat core for an explicit plan |
| `vera_parse_rule` | `POST /v1/rules/parse` | Plain language → structured rule |
| `vera_get_rules` | `GET /v1/rules` | Active rulebook |
| `vera_query_memory` | `GET /v1/memory/graph` | Entities and relationships |

Example tool call (`vera_evaluate_plan`):

```json
{
  "name": "vera_evaluate_plan",
  "arguments": {
    "plan": { "assignments": [ { "job": "WO-4811", "technician": "T-103" } ] }
  }
}
```

The MCP descriptor (tool schemas, auth) ships with the component package; the
same scoped API keys apply.

## Autonomy configuration

Autonomy is set per decision class in workspace settings (or via the API by
an admin-scoped key):

```
GET/PUT /v1/settings/autonomy
```

```json
{
  "job_reassignment":      "approve",
  "dispatch_routing":      "approve",
  "customer_notification": "autonomous",
  "maintenance_rebooking": "suggest"
}
```

Approve is the default for any class that changes operations. Autonomous
execution is logged identically to approved execution.

## Errors and limits

- Standard HTTP semantics; errors return `{ "error": { "code", "message" } }`.
- `422` — input failed schema validation after repair; the response includes
  the validation detail. Nothing reaches the solver on a `422`.
- `409` — rulebook changed between trace generation and execution; the
  decision must be re-derived (prevents acting on stale rules).
- Rate limits are per key; defaults suit operational use (single-digit
  requests/second). Solver evaluation of a typical workspace decision
  completes in well under a second; end-to-end latency is dominated by the
  language layer (1.5–30 s depending on the configured model).
