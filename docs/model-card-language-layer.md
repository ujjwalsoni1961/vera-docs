# Model Card — Vera Language Layer

This card documents the language layer of the Vera platform: the only
component where a machine-learning model is used. It follows the structure
of standard model cards, adapted to a system that deliberately restricts the
model's role.

## Role in the system

| | |
| --- | --- |
| Function | Translation only: plain-English rules and free-text operational messages → schema-validated structured form; solver results → readable reasoning steps |
| Explicitly not | Decision-making. The deterministic Z3 solver computes every decision; no model output reaches an action without passing schema validation and solver verification |
| Failure mode by design | An extraction failure degrades to human review, never to an unverified decision |

## Models

The layer is provider-agnostic behind a single abstraction
(OpenAI-compatible chat endpoint + schema validation). Configurations in
production or evaluation:

| Configuration | Models | Use |
| --- | --- | --- |
| Self-hosted, sovereign | Open-weight European models (Mistral family, Teuken class) via Ollama / vLLM | Customers requiring full data sovereignty |
| Hosted API | Customer-approved API models | Default for pilot workspaces |
| Planned (DeployAI) | European models via the AIoD Model Hub / LLM Lab | Production cutover after benchmark on the evaluation suite |

Model choice is a per-workspace configuration. Cutover between models
requires passing the schema-constrained extraction evaluation suite
([validation.md](validation.md), §3) at parity or better.

## Inputs and outputs

- **Input:** prompt text only — the operator's task or rule text plus a
  workspace snapshot (rules and operational entities). No raw documents, no
  attachments, no personal-data stores.
- **Output:** JSON validated against typed schemas (Pydantic). Malformed
  output triggers one automatic repair pass; persistent failure routes to
  human review with the raw text quarantined from the decision path.

## Evaluation

Continuously evaluated on the extraction suite (production-derived,
hand-verified references). Current production configuration:
96.8% first-pass schema validity, 94.2% field-exact extraction accuracy,
1.7% residual to human review (n = 412, rule-translation subset). The suite
is versioned with the rule schema; results gate releases and model swaps.

## Limitations

- Translation quality varies with input language and domain phrasing; the
  suite currently covers English and Finnish operational text. Additional
  EU languages ride on the multilingual capability of European models and
  are validated before being enabled per workspace.
- The model sees only what the snapshot contains; stale operational data
  produces correct reasoning over wrong facts. Mitigation: ingestion
  freshness checks and fact timestamps in the trace.
- Long, multi-clause rule statements may parse into more than one structured
  rule; the operator confirmation step exists precisely to catch this.

## Ethical considerations

No profiling, no decisions about individuals, no generation of customer-facing
free text without labelling. AI involvement is disclosed in every interface.
Bias surface is limited to translation quality differences across languages,
monitored via the per-language evaluation subsets.

## Maintenance

Owned by Nex0 Oy engineering. The card is updated when the model
configuration, schemas or evaluation results change materially; the change
history is in [CHANGELOG.md](../CHANGELOG.md).
