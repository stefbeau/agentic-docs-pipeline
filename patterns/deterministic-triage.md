# Pattern: Deterministic Triage

## What it is

A routing pattern where all classification decisions are made by rule-based code before any LLM agent fires. The triage layer reads a structured input, runs a set of typed detectors, and produces a routing manifest that downstream agents consume.

Zero LLM. Every decision is a deterministic function of the input.

---

## When to use it

Use deterministic triage when:

- The routing decision is the most consequential step in the pipeline — a wrong route propagates through every downstream stage
- The input is structured (typed fields, enumerations, identifiable patterns) rather than free-form natural language
- You need every routing decision to be auditable, reproducible, and unit-testable
- The cost of a non-deterministic routing error is higher than the cost of a rule that doesn't cover every edge case

Do not use deterministic triage when the classification genuinely requires reading and understanding unstructured natural language that cannot be reduced to field matching or pattern rules. In that case, a bounded LLM classifier with a confidence threshold is appropriate.

---

## Structure

```
Structured input
      │
      ▼
┌─────────────────────────────────────┐
│  Detector set                       │
│                                     │
│  detect_type()      → typed signal  │
│  detect_scope()     → typed signal  │
│  detect_audience()  → typed signal  │
│  detect_trigger_a() → typed signal  │
│  detect_trigger_b() → typed signal  │
│  detect_escalation()→ typed signal  │
└─────────────────┬───────────────────┘
                  │
                  ▼
         Routing manifest
         (structured JSON)
                  │
                  ▼
         Orchestrator / router
```

---

## Detector design principles

### One responsibility per detector

Each detector classifies one dimension of the input. A detector that classifies both change type and audience is two detectors collapsed into one — split them. Single-responsibility detectors are independently testable and independently updatable.

### Typed output values

Detector outputs are enumerations, not free text. `new_feature | enhancement | deprecation | fix` — not "this appears to be a new feature addition." Typed outputs make the routing manifest machine-readable and prevent downstream agents from having to interpret natural language routing signals.

### Uncertain is a valid output

For binary trigger decisions (does this require a communications message?), `uncertain` should be a first-class output value alongside `required` and `not_required`. Forcing a binary answer on genuinely ambiguous inputs produces confident wrong answers. `uncertain` routes the decision to confidence gating — a human-visible escalation path — rather than silently making the wrong call.

### Conservative on escalation

The escalation detector should err toward escalating. The cost of an unnecessary escalation is a human spending thirty seconds reviewing a run that didn't need it. The cost of a missed escalation is a low-quality signal propagating through the full pipeline and reaching the approval gate as a bad proposal. Asymmetric costs justify a conservative escalation threshold.

---

## Implementation notes

### Input normalisation first

Before running detectors, normalise the input: trim whitespace, standardise casing for field comparisons, handle null and empty field values explicitly. Detectors should receive clean, predictable inputs — not raw payloads with variable formatting.

### Detector versioning

Include a version identifier in the routing manifest output. When detector logic changes, the version field makes it possible to trace any historical routing decision back to the exact detector version that produced it. This is essential for debugging unexpected pipeline behaviour after a detector update.

### Test suite structure

Each detector should have:
- Unit tests for every explicit rule path
- Boundary tests for threshold values and edge patterns
- Negative tests — inputs that should NOT trigger a given classification
- Regression tests for historical edge cases that caused routing errors

When a real input causes a detector to misclassify, add it as a regression test before fixing the detector. The test suite should grow with the pipeline's operational history.

### Manifest schema validation

Validate the routing manifest against a schema before passing it to the orchestrator. A manifest with missing or malformed fields should fail loudly at the triage layer — not silently produce unexpected behaviour downstream.

---

## Reusability

The deterministic triage pattern is not specific to documentation pipelines. It applies anywhere that:

- A structured event needs to be classified and routed before expensive processing begins
- Routing correctness is more important than routing sophistication
- Audit trails and reproducibility are requirements

Examples: customer support ticket routing, CI/CD pipeline trigger classification, content moderation pre-filtering, incident severity triage.  
