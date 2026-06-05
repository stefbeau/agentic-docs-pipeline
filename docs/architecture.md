# Architecture

## Overview

The pipeline is a seven-agent orchestration system designed around one core constraint: **humans own every decision that reaches production**.

Agents handle the legwork — classification, retrieval, drafting, evaluation, logging. The human approval gate is not a feature that can be toggled off. It is load-bearing.

---

## Design philosophy

### Start with the constraint, not the capability

Most agentic systems are designed around what agents *can* do. This pipeline was designed around what they *shouldn't* do unsupervised — publish content to a live knowledge base used by AI-powered products.

That constraint shaped everything:
- Agents produce *proposals*, never publications
- Every proposal surfaces to a human interface before it can proceed
- The audit log captures what was proposed, who approved it, and when

### Deterministic where cheap, LLM where necessary

The most consequential design decision in the pipeline is **where not to use an LLM**.

The intake triage layer — which classifies incoming change events and routes them through the pipeline — uses zero LLM. Eight rule-based detectors make every routing decision. The output is deterministic, auditable, and fast.

LLM cost and latency are reserved for tasks that genuinely require judgment: impact analysis, content drafting, quality evaluation, communications writing.

This is Pattern C. See [pattern-c.md](pattern-c.md) for the full writeup.

### Bounded agent responsibilities

Each agent has exactly one job. No agent has visibility into the full pipeline state. This is not accidental — it is the property that makes individual agents testable, replaceable, and debuggable in isolation.

When something breaks, you know which agent broke it.

---

## Pipeline stages

### Stage 0 — Orchestrator

The entry point. Receives the product change event, validates the payload, and routes to the intake agent.

Key behaviour: **confidence gating**. If the orchestrator cannot determine routing with sufficient confidence, it escalates to human review immediately — before any downstream agent fires. This prevents low-quality signals from propagating through the pipeline.

### Stage 1 — Intake (Deterministic Triage)

The classification layer. Eight deterministic detectors analyse the incoming event:

| Detector | What it classifies |
|----------|-------------------|
| Change type | New feature / enhancement / deprecation / fix |
| Scope | Single product / cross-product / platform-wide |
| Audience | Internal / client-facing / developer |
| Urgency | Routine / expedited / emergency |
| Notify trigger | Whether a communications message is required |
| Content trigger | Whether documentation update is required |
| Review trigger | Whether a full quality review is required |
| Escalation trigger | Whether human review is required before proceeding |

Zero LLM. Every routing decision is a deterministic function of the input. The output is a structured routing manifest consumed by the orchestrator.

### Stage 2 — Impact Analysis

Semantic search across the full documentation corpus. The impact agent identifies which pages are affected by the change, scores severity, and produces a ranked impact list.

This is the most retrieval-intensive stage. The agent queries a vector index of corpus chunks, scores results against the change description, and filters by relevance threshold.

Output: structured impact manifest with page identifiers, severity scores, and change rationale.

### Stage 3 — Content Preparation

Drafts documentation updates for each affected page identified in the impact manifest.

Output format is a structured DIFF package — not a raw text block, but a machine-readable structure showing:
- What the current content says
- What it should say after the change
- Why the change is needed
- Confidence score for the suggestion

This structure feeds directly into the human approval interface, where writers review the diff and approve, reject, or edit each suggestion.

### Stage 4 — Communications (Notify)

Drafts the client-facing communications message that accompanies the documentation update — the announcement that reaches end users.

This agent applies voice and tone standards, terminology guidelines, and audience-appropriate framing. It is the most stylistically constrained agent in the pipeline.

Sub-agent architecture: a pre-processing script handles deterministic work (template selection, variable extraction, deduplication); two bounded sub-agents handle the judgment work (draft generation, voice normalisation).

### ✋ Human Approval Gate

Between Stage 4 and Stage 5, the pipeline stops.

The writer reviews:
- The content diff package from Stage 3
- The communications draft from Stage 4
- The impact manifest from Stage 2

They can approve, reject, or edit any component. Nothing proceeds without explicit approval.

This is not a UX decision. It is an architectural guarantee.

### Stage 5 — Quality Review

After approval, the quality review agent evaluates the approved content against the full rule corpus:
- Prose rules (sentence structure, clarity, active voice)
- Style rules (tone, register, formatting conventions)
- Terminology rules (approved terms, prohibited terms, consistency)

77 rules across three evaluation sub-agents. Output is a structured findings payload with rule IDs, severity levels, and remediation suggestions.

### Stage 6 — Governance

Every pipeline run produces a structured audit log:
- Routing decisions and confidence scores
- Impact manifest summary
- Approval event (who, when, what was changed)
- Quality evaluation findings
- Pipeline timing and token consumption

The governance agent publishes this to the pipeline health dashboard and flags any anomalies for review.

---

## Token efficiency

The original pipeline architecture consumed ~165,000 tokens per run — viable for prototyping, not for production.

Re-architecture around Skills-type agents (replacing an earlier Graph-type sub-agent design) reduced per-run token consumption to ~35,500 tokens — a 78% reduction — while preserving all pipeline behaviour.

The key lever: eliminating redundant context passing between agents. Each agent now receives only the context it needs for its specific task, not the full pipeline state.

---

## Testing

318 tests across the pipeline codebase, all green.

Test coverage is weighted toward the deterministic triage layer (Stage 1), where correctness is verifiable and regressions are detectable. The 8 detectors have comprehensive unit test coverage. LLM-dependent stages use integration tests with fixture inputs.

---

## What this is not

This pipeline does not:
- Publish anything autonomously
- Bypass the human approval gate
- Make irreversible changes without a human decision
- Operate outside the scope of the structured change events it is designed to process

The goal is not autonomous documentation. The goal is to make the human writer's job tractable at a scale where manual processes have already broken down.
