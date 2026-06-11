# Vera — Demonstration Walkthrough

The demonstration console (https://vera.nex0.tech) runs a
dedicated demonstration workspace: a Finnish building-systems field-service
operation (heat pumps, gas boilers, ventilation, refrigeration) with two
regions, three depots, eight technicians, six vans and a day's board of work
orders across the Helsinki capital region and Tampere. The workspace mirrors
our production deployments; the decision path is the real one — LLM translation, memory recall, Z3
verification, audit.

The console is gated behind a workspace sign-in. Evaluation credentials are
available on request from security@nex0.tech, normally within one business
day.

## Suggested 10-minute walkthrough

### 1. Disruption handling (suggestion chip)

> "Mikko Virtanen called in sick — reassign his jobs for today."

Watch the reasoning panel: today's board is pulled, candidate reassignments
are checked against certifications (R3), working-time limits (R4) and SLA
windows (R1), and the recommendation arrives with an assignment widget.
Every step carries reference chips (`R4`, `WO-4811`) that resolve to the
exact rule or fact used.

### 2. Route optimization with a verifiable result

> "Optimize this afternoon's routes for the Espoo crew."

The recommended action renders an interactive map in the thread: previous
route dashed, proposed route solid, numbered stops, distance and drive-time
deltas (41 → 28 km). Click **Approve**: the card flips to APPLIED, the map
stays, and the decision lands in the audit log with its full trace.

### 3. A task that is not scripted

Type something free-form, e.g.:

> "A refrigerant leak was reported at K-Market Munkkivuori — who can respond
> and what happens to their other jobs?"

This goes through the live pipeline: the LLM reads the workspace snapshot,
proposes a response plan, and the solver-verification step reports the rules
checked (F-gas certification, working-day limit) before the recommendation
is shown. Response time depends on the configured model (15–30 s here).

### 4. Conversational rule capture

State a policy in the thread:

> "From now on, daycare and school emergency callouts must always be staffed
> before 15:00."

Vera detects a policy statement, parses it to structured form and proposes a
rulebook addition — note that it cross-checks the existing rulebook and says
so if a current rule already covers the case. Confirm, and the rule appears
in the Rulebook with source *chat* and in the Memory graph.

### 5. Rule import

Rulebook → **Import**. Either pull policies from the connected field-service
system, or upload a CSV/text file with one rule per line; each is parsed to
its structured form and reviewed before activation, with source *imported*.

### 6. Inspect the system

- **Memory** — the workspace knowledge graph: depots, technicians,
  certifications, vans, sites, SLAs and their relationships.
- **Audit** — every decision with its permanent reasoning trace, including
  the ones you just made.
- **Developers** — the same reasoning as a callable component: REST examples
  and the MCP tool list.
- **Settings** — connected systems, per-decision-class autonomy
  (suggest / approve / autonomous), EU data residency.

## What this demonstrates — and what it does not

The demonstration proves the decision path end to end:
translation, recall, formal verification, explanation, execution, audit. It
deliberately runs serverless without the production stores, so workspace
state is browser-scoped. Production behaviour beyond it — multi-store
memory, connectors, the full autonomy loop — is documented in
[architecture.md](architecture.md) and shown in guided sessions with
production deployments (under customer confidentiality).
