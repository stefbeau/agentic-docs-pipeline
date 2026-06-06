# Agent 06 — Governance

## Role

The Governance agent produces the audit record for every pipeline run. It collects outputs from all upstream stages, constructs a structured run log, publishes pipeline health metrics, and flags anomalies for review.

It is the last agent in the pipeline and the one that makes the pipeline trustworthy at scale.

---

## Responsibilities

### 1. Run log construction

At the end of every pipeline run, the Governance agent assembles a complete run log from the outputs of all upstream stages:

```json
{
  "run_id": "...",
  "timestamp": "...",
  "change_event_id": "...",
  "routing_manifest": { ... },
  "agents_dispatched": ["01", "02", "03", "04", "05"],
  "impact_summary": {
    "pages_affected": 4,
    "severity_breakdown": { "critical": 1, "high": 2, "medium": 1 }
  },
  "approval_event": {
    "approver": "...",
    "timestamp": "...",
    "components_approved": 3,
    "components_edited": 1,
    "components_rejected": 0
  },
  "quality_summary": {
    "findings_critical": 0,
    "findings_warning": 2,
    "findings_info": 1
  },
  "token_consumption": 34800,
  "pipeline_duration_seconds": 47,
  "anomalies": []
}
```

Every field is populated from structured outputs produced upstream. The Governance agent does not infer or estimate — it assembles.

### 2. Anomaly detection

The Governance agent evaluates each run against a set of anomaly thresholds:

| Signal | Anomaly threshold | Action |
|--------|------------------|--------|
| Token consumption | > 150% of baseline | Flag for review |
| Pipeline duration | > 200% of baseline | Flag for review |
| Confidence scores | Any stage < minimum threshold | Flag for review |
| Approval edits | > 50% of components edited | Flag for review |
| Quality findings | Any critical severity | Flag for review |
| Escalation events | Any escalation triggered | Flag for review |

Flagged runs are surfaced in the pipeline health dashboard. They do not block publication — the approval gate has already cleared them — but they are visible to the pipeline maintainer for investigation.

### 3. Pipeline health metrics

Across runs, the Governance agent accumulates metrics that support pipeline maintenance:

- **Per-agent confidence trends** — is any agent's confidence score degrading over time?
- **Approval edit rate** — which agent's outputs are being edited most frequently?
- **Rejection rate by component** — which output types are being rejected?
- **Rule firing frequency** — which quality rules are triggered most often?
- **Token consumption trends** — is cost per run increasing?

These metrics are the primary signal for when prompt tuning, rule corpus updates, or detector adjustments are needed.

### 4. Corpus change record

For every approved and published update, the Governance agent writes a corpus change record:

- Page identifier
- Change type (update / addition / removal)
- Change event that triggered the update
- Approver identity and timestamp
- Pre-update and post-update content hashes

The corpus change record is append-only. It provides a complete history of every change made to the knowledge base through the pipeline — traceable to the product change event that initiated it.

---

## Prompt pattern

The Governance agent has a minimal LLM footprint. Most of its work is deterministic — assembling structured outputs, evaluating thresholds, constructing records.

The LLM component is limited to one task: **anomaly interpretation**. When a run produces multiple anomaly signals, the agent produces a short plain-language summary of what the anomaly pattern suggests — not a diagnosis, but a pointer toward likely causes. This summary is surfaced alongside the anomaly flags in the dashboard.

The prompt for this task is narrow: given the anomaly signals from this run, describe in two to three sentences what this pattern typically indicates and what the maintainer should check first.

---

## Key design decisions

**Why Governance is the last agent**
The Governance agent needs complete outputs from all upstream stages to construct a meaningful run log. Running it last is the only position where all that context is available.

**Why anomaly detection uses thresholds rather than LLM judgment**
Threshold-based anomaly detection is transparent and tunable. A maintainer who sees a false positive can adjust the threshold. An LLM-based anomaly detector would produce judgments that are harder to understand and harder to calibrate. For monitoring infrastructure, determinism is more valuable than sophistication.

**Why the corpus change record is append-only**
An append-only record cannot be edited to remove evidence of a change. This is intentional. The corpus change record is not just operational logging — it is the evidentiary trail that makes the pipeline auditable. Any system that can modify the change record cannot be fully trusted as an audit trail.

**Why approval edit rate is a key metric**
High approval edit rates on a specific agent's output are the clearest signal that the agent's prompt needs tuning. If writers are consistently editing the Notify agent's communications drafts in the same way — adding context, adjusting tone, correcting terminology — that pattern should propagate back into the prompt, not remain as recurring manual work at the approval gate.

---

## Inputs and outputs

| | Type | Description |
|--|------|-------------|
| **Input** | All upstream stage outputs | Routing manifest, impact manifest, DIFF package, communications draft, quality findings |
| **Input** | Approval event record | Approver, timestamp, edit/reject decisions |
| **Output** | Run log | Complete structured record of the pipeline run |
| **Output** | Anomaly flags | Signals outside threshold with plain-language summary |
| **Output** | Corpus change record | Append-only record of published knowledge base changes |
| **Output** | Health metrics update | Incremental update to pipeline health dashboard |    
