# Human-in-the-Loop — The Approval Gate

## The core position

AI agents should accelerate human judgment, not replace it.

This is not a cautious hedge. It is a design constraint that shaped every architectural decision in this pipeline. The approval gate is not a feature. It is load-bearing infrastructure.

---

## Why an approval gate

### The trust problem

An agentic pipeline that publishes autonomously has a trust problem that compounds over time.

When it gets something wrong — and it will — the error propagates silently. Content that contradicts the product ships. Users read it. Support tickets arrive. By the time the error surfaces, it has already done damage. And the humans responsible for documentation quality had no opportunity to catch it.

An approval gate makes errors visible before they propagate. The writer sees the proposal, recognises the error, and rejects it. The pipeline learns nothing from this — agents do not update from rejections — but the knowledge base stays clean.

### The accountability problem

Autonomous publication also has an accountability problem.

Who is responsible for content that an agent published without human review? In practice, the answer is nobody — which means everybody. Documentation quality standards erode because no individual owns the output.

The approval gate restores ownership. A writer who approves a suggestion owns that suggestion. They can edit it, reject it, or accept it — but they cannot ignore it. Every published update has a named human who reviewed it.

### The AI data quality problem

For organisations whose AI-powered products retrieve from the same knowledge base that the pipeline maintains, the stakes are higher than UX.

Stale or inaccurate documentation does not just confuse users. It contaminates the retrieval corpus. Every LLM-powered feature that queries that corpus inherits the error. The failure mode is invisible and systemic.

Human review at the approval gate is the last checkpoint before content enters the corpus. It is worth the friction.

---

## What the gate looks like

The approval gate sits between Stage 4 (content and communications drafting) and Stage 5 (quality review).

At the gate, the writer sees:

**1. The impact manifest**
Which pages are affected. Severity scores. Why each page was flagged.

**2. The content diff**
For each affected page: current content, proposed content, change rationale, agent confidence score. Displayed as a structured diff, not a wall of text.

**3. The communications draft**
The client-facing announcement that will accompany the update. Voice, tone, and terminology applied by the communications agent.

The writer can:
- **Approve** any component as-is
- **Edit** any component before approving
- **Reject** any component — sending it back with a reason
- **Escalate** — flagging the full run for senior review before proceeding

Nothing proceeds until every component has an explicit decision.

---

## What the gate is not

### Not a rubber stamp

A gate that writers approve without reading is not a gate. It is theatre.

The interface is designed to make review tractable, not invisible. Diff views highlight exactly what changed. Confidence scores surface low-certainty suggestions for closer scrutiny. Rejection is a first-class action, not an edge case.

### Not a bottleneck by design

The pipeline does the work that used to make review slow: retrieval, cross-referencing, drafting, formatting. By the time content reaches the gate, the writer is reviewing a near-complete proposal — not starting from a blank page.

The goal is to compress review time without compressing review quality.

### Not optional

The gate cannot be bypassed. There is no administrative override, no bulk-approve flow, no scheduled autonomous publish. Every pipeline run that produces output requires a human decision before that output reaches production.

This is enforced at the architecture level, not the policy level.

---

## The broader argument

Human-in-the-loop is sometimes framed as a concession to risk aversion — a temporary constraint until agents are trustworthy enough to operate autonomously.

This pipeline takes the opposite position: the approval gate is not a limitation to be engineered away. It is what makes the pipeline trustworthy enough to operate at all.

The value of agentic automation is not in removing humans from the loop. It is in changing what humans spend their time on — from mechanical retrieval and first-draft production to review, judgment, and quality ownership.

That is a better use of a technical writer's expertise. It is also a more defensible system.

---

## Implementation notes

### Confidence score surfacing

Every agent output carries a confidence score. Scores below threshold are flagged visually at the approval interface — not blocked, but highlighted for closer review. Writers develop calibration over time: they learn which agent outputs to trust quickly and which to scrutinise.

### Rejection logging

Every rejection is logged with the reason provided by the writer. This creates a feedback corpus that can be used to evaluate agent performance over time — not for real-time agent learning, but for periodic prompt and rule updates by the pipeline maintainer.

### Partial approval

Writers can approve the content diff and reject the communications draft, or vice versa. Components are reviewed independently. A rejected component sends only that component back through the relevant agent; the rest of the run is not discarded.

### Audit trail

Every approval event is recorded in the governance log: component approved, writer identity, timestamp, whether edits were made, and the final approved content. The audit trail is append-only.  
