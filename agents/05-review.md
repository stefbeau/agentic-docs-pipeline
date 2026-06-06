# Agent 05 — Quality Review

## Role

The Quality Review agent evaluates approved documentation content against a structured rule corpus before it enters the live knowledge base. It produces a findings payload — a machine-readable list of rule violations with severity levels and remediation suggestions.

It runs after the human approval gate, not before. The writer has already approved the content; the Quality Review agent is the final automated check before publication.

---

## Responsibilities

### 1. Rule corpus application

The agent evaluates content against 77 rules organised across three categories:

| Category | Rules | Scope |
|----------|-------|-------|
| Prose | 39 | Sentence structure, clarity, active voice, concision |
| Style | 19 | Tone, register, formatting conventions, heading structure |
| Terminology | 19 | Approved terms, prohibited terms, consistency, capitalisation |

### 2. Sub-agent architecture

Three bounded sub-agents run in parallel, one per rule category:

```
Approved content
      │
      ├──▶ Prose sub-agent    (39 rules) ──▶ Prose findings
      ├──▶ Style sub-agent    (19 rules) ──▶ Style findings  
      └──▶ Terminology agent  (19 rules) ──▶ Terminology findings
                                                    │
                                              Post-script
                                                    │
                                          Consolidated findings payload
```

Each sub-agent receives only its own rule set and the content to evaluate. Sub-agents do not share context. A post-script consolidates their outputs into a single structured findings payload.

### 3. Findings payload structure

```json
{
  "page_id": "...",
  "findings": [
    {
      "rule_id": "PROSE_014",
      "category": "prose",
      "severity": "warning",
      "location": "paragraph_2_sentence_3",
      "description": "Passive voice construction",
      "current_text": "...",
      "suggested_text": "...",
      "confidence": 0.91
    }
  ],
  "summary": {
    "critical": 0,
    "warning": 3,
    "info": 2
  }
}
```

Findings are traceable to specific rule IDs, locations within the content, and confidence scores. The writer can act on findings or override them — with every decision logged.

---

## Prompt pattern

Each sub-agent prompt follows the same structure, applied to its specific rule category.

**Rule-referenced evaluation**
The prompt references rule IDs explicitly. The agent is instructed to evaluate each rule independently and produce a finding only when a violation is identified. It is not asked to rate overall quality or suggest structural improvements — only to flag specific rule violations.

**Location precision**
The prompt requires the agent to identify the specific location of each violation — paragraph, sentence, or phrase. Vague findings ("this section is unclear") are not actionable. Location-precise findings give the writer an exact target for remediation.

**Confidence threshold**
The prompt instructs the agent to include a confidence score with each finding and to suppress findings below a minimum confidence threshold. Low-confidence findings add noise without value. The threshold is tuned per rule category — some rule types (passive voice detection) support higher confidence than others (register consistency).

---

## Key design decisions

**Why three sub-agents rather than one**
A single agent evaluating 77 rules faces context window pressure and rule interference — rules from different categories can produce contradictory signals that a single agent struggles to resolve cleanly. Three bounded sub-agents, each owning one category, produce more consistent findings with fewer false positives.

**Why parallel execution**
The three sub-agents run concurrently. There is no dependency between prose, style, and terminology evaluation. Parallel execution reduces the quality review stage from three sequential LLM calls to the latency of one.

**Why findings are traceable to rule IDs**
Rule IDs make the findings payload machine-readable and auditable. A finding tagged `PROSE_014` can be traced to the exact rule definition, the rule version, and the historical frequency of that rule being triggered. This supports rule corpus maintenance — rules that fire too frequently or are frequently overridden by writers are candidates for revision.

**Why Quality Review runs after the approval gate**
Running quality review before the approval gate would mean evaluating content that the writer may edit during review — producing findings against content that no longer exists in its evaluated form. Running it after the gate evaluates the content the writer actually approved. The findings are a final quality check, not a pre-approval filter.

---

## Inputs and outputs

| | Type | Description |
|--|------|-------------|
| **Input** | Approved content | Writer-approved DIFF output from Content Prep |
| **Input** | Rule corpus | 77 rules across three categories, versioned |
| **Output** | Findings payload | Structured rule violations with IDs, severity, location, remediation |
| **Output** | Review summary | Finding counts by severity category |
| **Output** | Review audit entry | Sub-agent outputs, rule corpus version, processing time |  
