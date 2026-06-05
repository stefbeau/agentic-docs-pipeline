# Pattern C — Deterministic-First Design

## What it is

Pattern C is an architectural pattern for agentic systems that separates work into two categories:

- **Deterministic work** — rule-based, auditable, fast, cheap. No LLM.
- **Judgment work** — context-dependent, requires reasoning. LLM justified.

The pattern's core rule: **default to deterministic. Use LLM only where judgment is genuinely required and cannot be reduced to a rule.**

---

## Why it matters

LLMs are powerful but they are also:
- Expensive per token
- Non-deterministic (same input, different output)
- Slow relative to rule-based code
- Opaque — hard to audit when something goes wrong

In an agentic pipeline, these properties compound. A pipeline that uses LLM for every step is costly, unpredictable, and difficult to debug. When it fails, you don't know why.

Pattern C constrains LLM usage to the stages where it earns its cost.

---

## The three layers

### Layer 1 — Pre-script (deterministic)

Runs before any agent fires. Handles everything that can be determined from the raw input without reasoning:

- Input validation and normalisation
- Variable extraction (dates, identifiers, structured fields)
- Template selection based on explicit rules
- Deduplication and flag resolution
- Routing signal preparation

No LLM. Deterministic functions of the input. Output is a clean, structured context object passed to the agent layer.

**Why this matters:** pre-scripts are unit-testable. Every decision they make is verifiable. They are the cheapest and most reliable part of the pipeline.

### Layer 2 — Bounded agent (LLM, constrained)

Receives the structured context from the pre-script. Performs a single, bounded judgment task:

- Draft a communications message in a specific voice
- Evaluate a content block against a named rule
- Score the relevance of a document to a change description
- Generate a structured diff between current and proposed content

The agent's scope is narrow by design. It does not have access to the full pipeline state. It does not make routing decisions. It produces one output and stops.

**Why this matters:** narrow scope makes agent behaviour predictable and testable. A bounded agent that drafts communications messages should always produce a communications message — nothing else.

### Layer 3 — Post-script (deterministic)

Runs after the agent. Handles everything that can be determined from the agent's output without further reasoning:

- Output validation (required fields present, format correct)
- Envelope construction (wrapping output in the expected handoff structure)
- Anomaly flagging (output outside expected bounds)
- Audit log entry

No LLM. Deterministic functions of the agent output.

---

## Applied example — Intake triage

The intake stage classifies incoming change events and produces a routing manifest.

A naive implementation would pass the raw event to an LLM and ask it to classify. Pattern C rejects this.

Instead:

```
Raw change event
      │
      ▼
┌─────────────────────────────┐
│  Pre-script                 │
│  8 deterministic detectors  │  ← Zero LLM
│  Rule-based classification  │
└─────────────┬───────────────┘
              │
              ▼
     Structured routing manifest
```

Each detector is a function that takes the event payload and returns a typed classification signal. The detectors are:

| Detector | Input signals | Output |
|----------|--------------|--------|
| `detect_change_type` | Change category field, keyword patterns | `new_feature` / `enhancement` / `deprecation` / `fix` |
| `detect_scope` | Product identifiers, cross-reference fields | `single` / `cross_product` / `platform` |
| `detect_audience` | Audience flags, distribution lists | `internal` / `client` / `developer` |
| `detect_urgency` | Priority field, deadline signals | `routine` / `expedited` / `emergency` |
| `detect_notify_trigger` | Communication flags, release type | `required` / `not_required` / `uncertain` |
| `detect_content_trigger` | Documentation flags, change scope | `required` / `not_required` / `uncertain` |
| `detect_review_trigger` | Quality flags, change severity | `required` / `not_required` |
| `detect_escalation_trigger` | Confidence signals, edge case flags | `escalate` / `proceed` |

All eight detectors run. Their outputs are assembled into the routing manifest. The orchestrator reads the manifest and routes accordingly.

**Result:** every routing decision is a deterministic function of the input. Fully auditable. Unit-testable. No LLM cost. No non-determinism.

---

## Applied example — Quality review

The quality review stage evaluates content against 77 rules across three categories.

A naive implementation would pass the content and all 77 rules to a single LLM call and ask for a findings list. Pattern C rejects this for different reasons: context window pressure, rule interference, and unverifiable outputs.

Instead, three bounded sub-agents each own a rule category:

```
Approved content
      │
      ├──▶ Prose sub-agent    (39 rules) ──▶ Prose findings
      ├──▶ Style sub-agent    (19 rules) ──▶ Style findings
      └──▶ Terminology agent  (19 rules) ──▶ Terminology findings
                                                    │
                                                    ▼
                                          Consolidated findings payload
```

Each sub-agent receives only its rule set and the content to evaluate. It produces a structured findings payload with rule IDs, severity levels, and remediation suggestions.

**Result:** findings are traceable to specific rules. Individual sub-agents can be retested in isolation. Rule sets can be updated without touching the others.

---

## When to break the pattern

Pattern C is a default, not a dogma. LLM is justified when:

1. The classification genuinely requires reading and understanding natural language context — not just matching fields or patterns
2. The output space is too large or too variable to enumerate in rules
3. The cost of a wrong deterministic rule is higher than the cost of LLM non-determinism

When you find yourself writing a deterministic rule that covers 95% of cases and fails unpredictably on the other 5%, that is a signal to consider a bounded LLM agent for that specific detector.

The test: **can you write a unit test for this decision?** If yes, it's deterministic. If no, it might be judgment.

---

## Summary

| Property | Deterministic (pre/post scripts) | LLM (bounded agents) |
|----------|----------------------------------|----------------------|
| Cost | Minimal | Significant |
| Speed | Fast | Slow |
| Auditability | Complete | Partial |
| Testability | Unit-testable | Integration-testable |
| Use when | Classification, validation, formatting | Drafting, scoring, reasoning |

Pattern C is what makes a production agentic pipeline feel like software rather than a black box.  
