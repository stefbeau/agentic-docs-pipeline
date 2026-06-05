# agentic-docs-pipeline

> A seven-agent orchestration pipeline that automates the documentation lifecycle — from product change events through impact analysis, content drafting, communications, and governance reporting.

**Runner-Up, Internal Innovation Competition 2026**

---

## The problem

At scale, documentation accuracy is not just a UX concern — it's an AI data-quality problem.

When a knowledge base of 25,000+ pages drifts out of sync with the product, every LLM-powered feature that retrieves from it gets worse. Chatbots hallucinate. Search returns stale results. Support costs rise. The root cause isn't bad writing — it's a broken feedback loop between product changes and documentation updates.

This pipeline closes that loop.

---

## How it works

A product change event (a structured release document) enters the pipeline. Seven agents handle the journey from raw signal to approved, published documentation update — with a mandatory human-in-the-loop gate before anything reaches production.

```
Product Change Event
        │
        ▼
┌───────────────┐
│  00 · Orch.   │  Routes the event. Confidence-gated — low-confidence
│  Orchestrator │  signals escalate to human review immediately.
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  01 · Intake  │  Deterministic triage. 8 rule-based detectors classify
│  Agent        │  the change type. Zero LLM — fully auditable routing.
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  02 · Impact  │  Semantic search across the corpus. Identifies which
│  Agent        │  pages are affected and scores change severity.
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  03 · Content │  Drafts documentation updates. Structured DIFF output
│  Prep Agent   │  showing exactly what changes and why.
└───────┬───────┘
        │
        ▼
┌───────────────┐     ┌─────────────────────┐
│  04 · Notify  │     │  ✋ HUMAN APPROVAL   │
│  Agent        │────▶│  Writer reviews and  │
└───────┬───────┘     │  approves before     │
        │             │  anything publishes  │
        ▼             └─────────────────────┘
┌───────────────┐
│  05 · Review  │  Quality evaluation against content rules, voice/tone
│  Agent        │  standards, and terminology guidelines.
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  06 · Govern. │  Audit logging, metrics, pipeline health reporting.
│  Agent        │  Every run is traceable.
└───────────────┘
```

---

## Design principles

**Deterministic-first (Pattern C)**
Determinism where it's cheap; LLM where judgment is genuinely required. The intake triage layer uses zero LLM — 8 rule-based detectors make every routing decision provably auditable. LLM cost and latency are reserved for bounded judgment tasks.

**Human-in-the-loop by design**
The approval gate is not optional and not bypassable. Writers own every suggestion before it publishes. The pipeline accelerates the work; humans own the decisions.

**Bounded agent responsibilities**
Each agent has one job. No agent knows about the full pipeline. This makes individual agents testable, replaceable, and debuggable in isolation.

**Audit-first**
Every pipeline run produces a structured audit log. Impact scores, routing decisions, and approval events are all captured. Nothing is a black box.

---

## Technical stack

| Layer | Technology |
|-------|-----------|
| Agent platform | Skills-type agents, A2A orchestration |
| Tool protocol | MCP (Model Context Protocol), FastMCP |
| Retrieval | Vector search across corpus chunks |
| Pre/post scripts | Python, deterministic rule detectors |
| Testing | 318 tests, all green |
| Token efficiency | ~78% reduction through agent-type re-architecture |

---

## Repository contents

```
├── docs/
│   ├── architecture.md          Core design decisions and pipeline overview
│   ├── pattern-c.md             Deterministic-first pattern explained
│   ├── human-in-the-loop.md     Approval gate philosophy and implementation
│   └── corpus-quality.md        Docs as AI data quality infrastructure
│
├── agents/
│   ├── 00-orchestrator.md       Routing logic and confidence gating
│   ├── 01-intake.md             Deterministic triage, 8 detectors
│   ├── 02-impact.md             Semantic impact analysis
│   ├── 03-content-prep.md       Content drafting and DIFF output
│   ├── 04-notify.md             Communications agent
│   ├── 05-review.md             Quality evaluation agent
│   └── 06-governance.md         Audit and reporting
│
├── visuals/
│   ├── pipeline-overview.html   Interactive pipeline diagram
│   ├── architecture.html        System architecture visual
│   └── pattern-c.html           Pattern C decision flow
│
└── patterns/
    ├── deterministic-triage.md  Reusable pattern: rule-based routing
    ├── mcp-tool-design.md       FastMCP tool design principles
    └── a2a-orchestration.md     Agent-to-agent coordination patterns
```

---

## Status

This repository is an anonymised portfolio representation of production work. Agent names, corpus details, and internal tooling references have been abstracted. The architecture, design decisions, and implementation patterns are real.

Production code lives on private enterprise infrastructure.

---

*Built by a technical writer who got tired of describing systems and started building them.*
