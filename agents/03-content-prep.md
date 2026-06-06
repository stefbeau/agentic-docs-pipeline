# Agent 03 — Content Preparation

## Role

The Content Prep agent drafts documentation updates for each affected page identified in the impact manifest. It produces a structured DIFF package — a machine-readable set of proposals showing current content, proposed content, and the rationale for each change.

---

## Responsibilities

### 1. Page retrieval

For each page in the impact manifest, the Content Prep agent retrieves the current page content via MCP tool call. It reads the live page, not a cached version — ensuring the draft is based on the current state of the corpus.

### 2. Change scoping

Before drafting, the agent scopes the required change for each page:

- What specifically needs to change on this page?
- Is this an update to existing content, an addition, or a removal?
- Are there dependencies — other pages that reference this one — that affect how the change should be framed?

Change scoping is derived from the impact manifest's change rationale and the agent's reading of the current page content.

### 3. DIFF package construction

For each page, the agent produces a structured DIFF package:

```json
{
  "page_id": "...",
  "severity": "critical",
  "change_type": "update",
  "current_content": "...",
  "proposed_content": "...",
  "change_rationale": "...",
  "confidence": 0.87,
  "dependencies": ["page_id_a", "page_id_b"]
}
```

The DIFF package is the primary input to the human approval interface. Writers review current vs. proposed content side by side, read the rationale, and make an approve / edit / reject decision.

### 4. Dependency flagging

When a proposed change on one page has implications for other pages, the agent flags the dependency relationship. Dependencies are surfaced at the approval gate so the writer can review related pages together rather than in isolation.

---

## Prompt pattern

The Content Prep agent prompt is structured around three constraints:

**Minimal change principle**
The prompt instructs the agent to make the smallest change that accurately reflects the product update. It should not restructure pages, update unrelated content, or apply style improvements beyond the scope of the change. Scope creep in content drafting produces proposals that are harder to review and more likely to introduce unintended changes.

**Preserve voice**
The agent drafts in the established voice of the existing page content. It does not impose a new style. If the current page uses second person, the draft uses second person. If the current page uses a specific terminology convention, the draft maintains it. Voice consistency is enforced by the Quality Review agent in Stage 5 — the Content Prep agent's job is accuracy, not style.

**Explicit rationale required**
Every proposed change must include a rationale that connects the change to the product change event. Proposals without clear rationale are rejected by the prompt. This ensures the writer at the approval gate can evaluate not just what is changing but why.

---

## Key design decisions

**Why live page retrieval via MCP**
Using cached page content risks drafting against a stale version of the page. If the page was updated recently — by another pipeline run or a manual edit — the draft will be based on outdated content. Live retrieval via MCP tool call ensures the agent always works from the current corpus state.

**Why structured DIFF rather than free-form draft**
A free-form draft ("here is the updated page") forces the writer to read the entire page to understand what changed. A structured DIFF shows exactly what changed and nothing else. Review time at the approval gate drops significantly. Writers can scan a diff in seconds; reading a full page takes minutes.

**Why confidence scores matter here**
Content Prep confidence scores are surfaced prominently at the approval gate. A high-confidence proposal on a critical-severity page warrants a quick review. A low-confidence proposal on any page warrants close scrutiny. The confidence score gives writers a triage signal without requiring them to read every proposal with equal attention.

**Why dependencies are flagged but not auto-resolved**
Automatically updating dependent pages would expand the scope of each pipeline run unpredictably. Flagging dependencies gives the writer visibility without creating cascading automatic changes. The writer decides whether to address dependencies in the current run or create a separate pipeline run for them.

---

## Inputs and outputs

| | Type | Description |
|--|------|-------------|
| **Input** | Impact manifest | Ranked affected pages with severity scores and rationales |
| **Input** | Live page content | Retrieved via MCP tool call at runtime |
| **Input** | Change event detail | Full product change description |
| **Output** | DIFF package | Structured current/proposed/rationale for each affected page |
| **Output** | Dependency flags | Pages with cross-page implications |
| **Output** | Content prep audit entry | Pages processed, confidence scores, tool call log |  
