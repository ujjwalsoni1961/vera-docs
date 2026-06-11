# Vera — Security, Compliance & Data Governance

## 1. Data protection (GDPR)

Vera processes operational logistics and field-service data that is largely
non-personal: jobs, routes, schedules, equipment, parts. Personal data
appears incidentally — names and contact details of operations staff or
technicians inside dispatch messages — and is handled under:

- **Roles.** The customer is the data controller; Nex0 Oy is the processor
  under a data processing agreement (DPA) signed with every customer.
- **Data minimisation.** Only the fields required for the decision context
  are extracted and stored; free-text source messages are retained only as
  long as operationally necessary.
- **Residency.** EU-region storage exclusively (PostgreSQL, Neo4j, vector
  index). Residency is contractual.
- **No transfers outside the EU.** With a self-hosted or EU-hosted model
  endpoint, no data leaves the EU at any stage.

## 2. EU AI Act position

Vera is a decision-support system for logistics operations with human
oversight. It makes no decisions about individuals' rights, health,
employment, credit or access to services and performs no biometric
processing — it does not fall under the Annex III high-risk categories. We
classify it as limited/minimal risk and nonetheless meet transparency
obligations:

- Users are informed they interact with an AI system; Vera introduces itself
  as an AI agent in every interface.
- AI-generated reasoning summaries are labelled as such; the decision itself
  is computed by the deterministic solver and is fully traceable.
- General-purpose model obligations rest with the model providers; using EU
  models further simplifies the compliance chain.

The product also strengthens regulatory compliance for its users: sectoral
transport and safety rules (driving-time limits, ADR/hazmat, certifications)
are encoded as explicit, checkable rules that cannot be silently bypassed,
and every decision documents which rules it satisfied.

## 3. Trustworthy AI by construction

- **Explainability is not post-hoc.** Every decision is the output of a
  deterministic solver and carries the rules applied, the facts used, the
  options rejected and the exact rule each rejection violated.
- **No profiling.** Decisions are made over explicit operational
  constraints; Vera performs no statistical decision-making about people.
- **Human oversight.** Three autonomy modes (suggest / approve /
  autonomous), configured per decision class, approval by default; an
  immutable decision log supports monitoring, spot-checks and regression
  evaluation.
- **Low-confidence handling.** Schema-validated extraction with an automatic
  repair pass; persistently low-confidence extractions route to human review
  and never reach the solver.
- **Versioned rules.** Every rule change is recorded with author and
  timestamp; every decision records the rulebook version in force.

## 4. Security controls

| Control | Implementation |
| --- | --- |
| Encryption in transit | TLS 1.2+ on all interfaces, service-to-service included |
| Encryption at rest | Encrypted volumes for all three stores; per-workspace keys |
| Access control | Role-based access in the console; scoped API keys (`reason`, `rules:write`, `audit:read`, admin) per integration; key rotation and revocation |
| Tenant isolation | Customer-scoped databases and graph namespaces; rule catalogues and decision history never cross tenants |
| Audit integrity | Append-only decision log with per-decision trace IDs; exports are customer-initiated |
| Secrets | Environment-injected, secret-manager backed; never in images or repositories |
| Backups | Continuous WAL archiving (PostgreSQL), daily snapshots (graph, vector), 30-day retention, quarterly restore rehearsal |

## 5. Data ownership and model training

- **Customers own their data** — rules, facts and decision history in full.
  Structured exports are available at any time, in open formats.
- **No training on customer data.** Vera does not train or fine-tune models.
  It uses pre-trained models for translation and a symbolic store for
  knowledge. Customer data is never used to train anything, by us or by
  model providers (pinned no-training API terms, or self-hosted models where
  the question does not arise).
- **Inference exposure is minimal.** Only prompt text reaches inference
  endpoints; operational stores never do.

## 6. Sovereignty

- Technology and IP are 100% owned by Nex0 Oy, a Finnish company with EU
  ownership and control.
- EU-region cloud today; self-hosted open-weight deployment (Ollama) is
  available for customers requiring full on-premise sovereignty.
- The model-agnostic language layer means sovereignty is a configuration
  choice, not a product variant.

## 7. Responsible disclosure

Security reports: security@nex0.tech. We acknowledge within 2 business days
and coordinate disclosure timelines with the reporter.
