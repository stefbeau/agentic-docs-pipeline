# Agent 04 — Communications (Notify)

## Role

The Notify agent drafts the client-facing communications message that accompanies the documentation update — the announcement that reaches end users when a product change ships.

It is the most stylistically constrained agent in the pipeline. Accuracy is necessary but not sufficient; the output must also meet voice, tone, and terminology standards enforced by a dedicated rule corpus.

---

## Responsibilities

### 1. Pre-script processing (deterministic)

Before the LLM agent fires, a deterministic pre-script handles everything that can be determined from the structured inputs:

- **Template selection** — which communications template applies to this change type and audience
- **Variable extraction** — product names, version identifiers, release dates, feature labels
- **Flag deduplication** — resolving conflicting signals from the routing manifest
- **Audience framing** — selecting the appropriate register (client-facing / internal / developer)

The pre-script produces a clean, structured context object. The LLM agent receives this, not the raw event payload.

### 2. Draft generation

The LLM agent drafts the communications message from the structured context. The draft follows the selected template's structure but fills in content based on the change event detail and the impact manifest summary.

Output structure:

```
Subject line       — action-oriented, audience-appropriate
Opening sentence   — what is changing and when
Body               — what users need to know
Call to action     — what users should do (if applicable)
```

### 3. Voice normalisation

A second bounded sub-agent evaluates the draft against the voice and tone standard and applies normalisation:

- Active voice where passive has been used
- Approved terminology substitutions
- Register consistency (no register mixing within a single message)
- Sentence length and complexity appropriate to audience

Voice normalisation is a post-draft pass, not a rewrite. The normalisation agent makes targeted corrections, not structural changes.

### 4. Output envelope construction (deterministic)

A post-script constructs the final output envelope:

- Final approved draft
- Template identifier used
- Audience classification
- Handoff package for the approval gate

---

## Prompt pattern

The Notify agent has two distinct prompts — one for draft generation, one for voice normalisation. They are kept separate deliberately.

**Draft generation prompt**
Structured around the selected template. The agent is instructed to follow the template structure precisely, fill in content from the structured context, and produce a complete draft — not a partial draft or a set of options. The approval gate gives the writer the opportunity to edit; the agent's job is to produce a complete, usable starting point.

**Voice normalisation prompt**
Narrow and rule-referenced. The normalisation agent is given the rule corpus identifiers relevant to voice and tone, not the full rule set. It is instructed to flag and correct specific rule violations, not to rewrite or improve the draft beyond the scope of those rules. Normalisation agents that over-correct produce drafts that no longer sound like the established communications voice.

---

## Key design decisions

**Why a pre-script rather than passing raw inputs to the LLM**
Raw change event payloads contain noise — fields irrelevant to communications drafting, inconsistent formatting, variable completeness. A deterministic pre-script extracts and structures exactly what the drafting agent needs. Cleaner input produces more consistent output. It also makes the drafting agent's behaviour more predictable and testable.

**Why two sub-agents rather than one**
Drafting and voice normalisation are different cognitive tasks. A single agent asked to draft and normalise simultaneously tends to over-normalise during drafting — producing technically correct but flat output — or under-normalise during review. Separating the tasks produces better output on both dimensions.

**Why the output envelope is constructed deterministically**
The envelope structure — template identifier, audience classification, handoff format — is fully determined by the inputs. There is no judgment involved. A deterministic post-script is faster, cheaper, and more reliable than asking the LLM to construct it.

**Why SendEmail is not a pipeline tool**
The Notify agent drafts communications. It does not send them. Email dispatch is a post-approval action, handled outside the pipeline after the writer has reviewed and approved the draft. The agent has no access to email sending tooling. This is an architectural guarantee, not a policy constraint.

---

## Inputs and outputs

| | Type | Description |
|--|------|-------------|
| **Input** | Change event summary | Processed by pre-script from raw payload |
| **Input** | Routing manifest | Audience classification, notify trigger type |
| **Input** | Approved content summary | Post-gate summary of documentation changes |
| **Output** | Communications draft | Structured subject + body in approved voice |
| **Output** | Template identifier | Which template was applied |
| **Output** | Notify audit entry | Draft version, normalisation changes applied, confidence |  
