# Agent 00 — Orchestrator

## Role

The Orchestrator is the entry point and coordinator of the pipeline. It receives the product change event, validates the payload, interprets the routing manifest produced by the Intake agent, and dispatches to downstream agents in the correct sequence.

It does not draft, evaluate, or publish anything. Its job is coordination and confidence gating.

---

## Responsibilities

### 1. Payload validation

Before any downstream agent fires, the Orchestrator validates the incoming event payload:

- Required fields present and correctly typed
- Change event identifier is unique (deduplication check)
- Payload version matches the expected schema

Validation failures are logged and surfaced immediately. The pipeline does not proceed on a malformed payload.

### 2. Routing manifest interpretation

The Intake agent (Stage 1) produces a structured routing manifest — a set of typed classification signals that determine which downstream agents are needed for this run.

The Orchestrator reads the manifest and constructs the execution plan:

```
Routing manifest
      │
      ├── content_trigger: required     → dispatch Agent 02, 03
      ├── notify_trigger: required      → dispatch Agent 04
      ├── review_trigger: required      → dispatch Agent 05
      ├── escalation_trigger: proceed   → continue
      └── urgency: routine              → standard sequencing
```

Not every pipeline run dispatches every agent. A change event that requires a communications message but no documentation update dispatches Agent 04 only. The Orchestrator constructs the minimal execution plan for each run.

### 3. Confidence gating

Every agent output carries a confidence score. The Orchestrator evaluates confidence at each stage transition:

| Confidence level | Orchestrator behaviour |
|-----------------|----------------------|
| High (≥ threshold) | Proceed to next stage |
| Medium (near threshold) | Proceed, flag for closer review at approval gate |
| Low (< threshold) | Escalate to human review before proceeding |

Low-confidence signals do not proceed automatically. The Orchestrator surfaces them to the human approval interface with an escalation flag. The writer decides whether to proceed, redirect, or discard the run.

This is the earliest point at which a human can intervene. It is intentional — catching low-quality signals before they consume downstream agent compute.

### 4. Sequencing and handoff

The Orchestrator manages the handoff between agents. Each agent receives only the context it needs — not the full pipeline state.

Handoff packages are constructed at each stage transition:

```
Agent 02 (Impact) receives:
  - Change event summary
  - Routing manifest
  - Corpus search parameters

Agent 03 (Content Prep) receives:
  - Impact manifest from Agent 02
  - Change event detail
  - Affected page identifiers

Agent 04 (Notify) receives:
  - Change event summary
  - Audience classification from routing manifest
  - Approved content summary (post-gate)
```

Agents do not communicate directly with each other. All inter-agent context flows through the Orchestrator.

### 5. Error handling and recovery

The Orchestrator handles agent failures without collapsing the full pipeline run:

- **Retriable failures** (transient tool errors, timeout) — retry with backoff, log the attempt
- **Non-retriable failures** (malformed agent output, schema violation) — escalate to human review, log the failure with full context
- **Partial failures** (one agent fails, others succeed) — surface the failure at the approval gate; the writer decides whether to proceed with partial output or restart

No silent failures. Every error produces a log entry and a human-visible signal.

---

## Prompt pattern

The Orchestrator prompt is structured around three principles:

**Explicit scope definition**
The prompt defines precisely what the Orchestrator does and does not do. It does not draft content. It does not evaluate quality. It does not make publishing decisions. Scope creep in the Orchestrator propagates to every downstream stage.

**Routing as the primary task**
The prompt treats routing manifest interpretation as the primary cognitive task. The Orchestrator is instructed to read the manifest carefully, construct the minimal execution plan, and state its routing decisions explicitly before dispatching.

**Escalation as a first-class action**
The prompt treats escalation not as a failure mode but as a normal, expected outcome for a meaningful fraction of runs. The Orchestrator is instructed to escalate readily on low confidence rather than proceeding and producing low-quality downstream output.

---

## Key design decisions

**Why the Orchestrator does not do triage**
Triage (classifying the change event) is deterministic work. It belongs in the Intake pre-script, not the Orchestrator. The Orchestrator receives the result of triage — the routing manifest — and acts on it. This keeps the Orchestrator's LLM budget focused on coordination and confidence evaluation, not classification.

**Why agents do not communicate directly**
Direct agent-to-agent communication creates implicit dependencies that are hard to test and hard to debug. Centralising all handoffs through the Orchestrator makes the execution plan explicit and traceable. Every context object that flows between agents is constructed and logged by the Orchestrator.

**Why confidence gating is at the Orchestrator level**
Confidence gating could be distributed — each agent could decide whether to escalate its own output. Centralising it at the Orchestrator creates a single, consistent escalation policy. Threshold values are configured in one place. Escalation behaviour is predictable and auditable.

---

## Inputs and outputs

| | Type | Description |
|--|------|-------------|
| **Input** | Change event payload | Structured product change document |
| **Input** | Routing manifest | Output of Intake agent deterministic triage |
| **Output** | Execution plan | Ordered list of agents to dispatch with handoff packages |
| **Output** | Escalation signal | Raised when confidence is below threshold |
| **Output** | Run audit entry | Routing decisions, confidence scores, timing |
