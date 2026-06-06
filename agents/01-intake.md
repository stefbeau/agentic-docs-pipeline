# Agent 01 — Intake

## Role

The Intake agent is the classification layer of the pipeline. It receives the raw product change event and produces a structured routing manifest that tells the Orchestrator which downstream agents are needed for this run.

It uses zero LLM. Every classification decision is deterministic.

---

## Responsibilities

The Intake agent runs eight rule-based detectors against the incoming event payload. Each detector is an independent function that returns a typed classification signal.

### The eight detectors

| Detector | What it classifies | Output values |
|----------|-------------------|---------------|
| `detect_change_type` | Nature of the change | `new_feature` / `enhancement` / `deprecation` / `fix` |
| `detect_scope` | Breadth of impact | `single` / `cross_product` / `platform` |
| `detect_audience` | Who is affected | `internal` / `client` / `developer` |
| `detect_urgency` | Processing priority | `routine` / `expedited` / `emergency` |
| `detect_notify_trigger` | Communications required? | `required` / `not_required` / `uncertain` |
| `detect_content_trigger` | Documentation update required? | `required` / `not_required` / `uncertain` |
| `detect_review_trigger` | Quality review required? | `required` / `not_required` |
| `detect_escalation_trigger` | Human review before proceeding? | `escalate` / `proceed` |

All eight detectors run on every event. Their outputs are assembled into the routing manifest.

### Routing manifest structure

```json
{
  "change_type": "enhancement",
  "scope": "single",
  "audience": "client",
  "urgency": "routine",
  "notify_trigger": "required",
  "content_trigger": "required",
  "review_trigger": "required",
  "escalation_trigger": "proceed",
  "confidence": 0.94,
  "detector_version": "2.1.0"
}
```

The Orchestrator reads this manifest and constructs the execution plan.

---

## Design decisions

**Why zero LLM**
Change event classification is a deterministic problem. The incoming payload contains structured fields — change category, product identifiers, audience flags, priority signals — that can be classified by rule without reasoning.

Using an LLM for classification would introduce non-determinism into the pipeline's most consequential decision point: which agents fire. A routing error in Stage 1 propagates through every downstream stage. Deterministic detectors make routing errors rare, detectable, and debuggable.

**Why eight separate detectors**
Each detector has a single classification responsibility. This makes them independently testable — a change to the notify trigger detector does not affect the scope detector. It also makes failures localised: if one detector produces an unexpected result, the others are unaffected.

**Why `uncertain` is a valid output**
For the notify and content triggers, `uncertain` is a first-class output — not an error state. When a detector cannot confidently classify a trigger, `uncertain` routes the decision to the Orchestrator's confidence gating logic, which surfaces it for human review. Forcing a binary output on ambiguous inputs would produce confident wrong answers.

**Why the manifest includes a version field**
Detector logic evolves as new change event patterns are encountered. The manifest version field makes it possible to trace any routing decision back to the exact detector version that produced it. This is essential for auditing unexpected pipeline behaviour after a detector update.

---

## Testing

The deterministic nature of the Intake layer makes it the most thoroughly tested component of the pipeline. Each detector has:

- Unit tests for every explicit rule
- Boundary tests for threshold values
- Negative tests for inputs that should not trigger
- Regression tests for historical edge cases

A new change event pattern that a detector handles incorrectly becomes a regression test. The test suite grows with the pipeline's operational history.

---

## Inputs and outputs

| | Type | Description |
|--|------|-------------|
| **Input** | Change event payload | Raw structured product change document |
| **Output** | Routing manifest | Typed classification signals for all eight detectors |
| **Output** | Confidence score | Aggregate confidence across all detector outputs |
| **Output** | Intake audit entry | Detector outputs, processing time, payload hash |
