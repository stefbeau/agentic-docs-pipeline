# Agent 02 — Impact Analysis

## Role

The Impact agent identifies which pages in the documentation corpus are affected by the incoming product change. It scores each affected page by severity and produces a ranked impact manifest consumed by the Content Prep agent.

---

## Responsibilities

### 1. Corpus search

The Impact agent queries a vector index of the documentation corpus. The search is parameterised by the change event summary and the scope classification from the routing manifest.

Search strategy varies by scope:

| Scope | Search behaviour |
|-------|----------------|
| `single` | Targeted search within the affected product's content area |
| `cross_product` | Broader search across related product areas |
| `platform` | Full corpus search with platform-level filters |

### 2. Relevance scoring

Retrieved pages are scored against the change description on two dimensions:

**Semantic relevance** — how closely does the page content relate to what is changing?

**Functional dependency** — does this page describe, reference, or depend on the feature that is changing?

Pages that score below the relevance threshold are filtered out. Pages above threshold are ranked and included in the impact manifest.

### 3. Severity classification

Each affected page receives a severity score:

| Severity | Meaning |
|----------|---------|
| `critical` | Page directly describes the changed feature — must be updated |
| `high` | Page references the changed feature substantially |
| `medium` | Page references the changed feature incidentally |
| `low` | Page may be affected — flag for writer review |

### 4. Change rationale

For each affected page, the Impact agent produces a short change rationale — a one or two sentence explanation of why the page is affected and what specifically needs to change.

This rationale is passed to the Content Prep agent as part of the impact manifest and surfaced to the writer at the approval gate.

---

## Prompt pattern

The Impact agent prompt is structured around two constraints:

**Retrieve, don't generate**
The agent's job is to find affected pages, not to suggest how to fix them. The prompt explicitly prohibits the Impact agent from drafting content changes. That is the Content Prep agent's responsibility. Keeping these concerns separate prevents the Impact agent from producing output that bypasses the Content Prep stage.

**Score conservatively**
The prompt instructs the agent to err toward inclusion rather than exclusion at the relevance threshold. A page that is incorrectly excluded from the impact manifest will not be updated — a silent miss. A page that is incorrectly included will be reviewed and dismissed by the writer at the approval gate — a visible false positive. False positives are preferable to silent misses in a documentation pipeline.

---

## Key design decisions

**Why vector search rather than keyword search**
Product changes are described in natural language that may not match the exact terminology used in affected documentation. Vector search finds semantically related content regardless of surface form. A change described as "updated authentication flow" will surface pages that discuss "login process," "sign-in behaviour," and "session management" — not just pages that contain the phrase "authentication flow."

**Why severity scoring matters**
The Content Prep agent prioritises its drafting work by severity. A run that affects twenty pages cannot produce twenty high-quality content proposals in a single pass. Severity scoring allows the pipeline to focus on critical and high-severity pages first, with lower-severity pages flagged for writer review rather than automated drafting.

**Why the change rationale is agent-generated**
The writer reviewing the impact manifest at the approval gate needs to understand why each page is flagged. A bare list of page identifiers is not actionable. The agent-generated rationale gives the writer enough context to make a quick review decision without reading the full page content.

---

## Inputs and outputs

| | Type | Description |
|--|------|-------------|
| **Input** | Change event summary | Processed change description from Orchestrator |
| **Input** | Routing manifest | Scope and audience classification |
| **Input** | Corpus search parameters | Index identifiers, scope filters |
| **Output** | Impact manifest | Ranked list of affected pages with severity scores and change rationales |
| **Output** | Search audit entry | Query parameters, result count, filtering decisions |  
