# Pattern: A2A Orchestration

## What it is

A coordination pattern for multi-agent pipelines where a central orchestrator manages all agent-to-agent communication, context passing, and execution sequencing. Agents do not communicate directly with each other — all inter-agent context flows through the orchestrator.

A2A (agent-to-agent) orchestration is the structural pattern that makes a set of individual agents behave as a coherent pipeline.

---

## The core rule

**Agents do not talk to each other.**

Every piece of context that flows between agents is constructed, packaged, and dispatched by the orchestrator. An agent that needs output from a previous agent receives it as part of its input package — not by calling the previous agent directly.

This rule is what makes the pipeline debuggable. When something breaks, you examine the orchestrator's handoff packages to find where the bad context entered the pipeline. In a mesh of agents communicating directly, that trace is much harder to follow.

---

## Orchestrator responsibilities

### Execution planning

The orchestrator reads the routing manifest (produced by the triage layer) and constructs the minimal execution plan for this run. Not every agent fires on every run:

```
Routing manifest signals:
  content_trigger: required    → include agents 02, 03
  notify_trigger: required     → include agent 04
  review_trigger: not_required → skip agent 05
  escalation_trigger: proceed  → continue

Execution plan: [02, 03, 04, 06]
```

The execution plan is explicit and logged. Every run produces a record of which agents fired and in what order.

### Handoff package construction

Before dispatching each agent, the orchestrator constructs a handoff package containing exactly the context that agent needs — nothing more.

```
Agent 02 (Impact Analysis) receives:
  - change_event_summary       ← from orchestrator input
  - scope_classification       ← from routing manifest
  - corpus_search_parameters   ← derived from scope + change type

Agent 03 (Content Prep) receives:
  - impact_manifest            ← from Agent 02 output
  - change_event_detail        ← from orchestrator input
  - affected_page_ids          ← extracted from impact_manifest

Agent 04 (Communications) receives:
  - change_event_summary       ← from orchestrator input
  - audience_classification    ← from routing manifest
  - approved_content_summary   ← post-gate, from approval event
```

Agents receive purpose-built packages, not the full pipeline state. This keeps agent prompts focused and reduces token consumption — an agent that receives only what it needs is cheaper and more predictable than one that receives everything.

### Confidence gating

The orchestrator evaluates the confidence score of each agent output before dispatching the next agent. Low-confidence outputs escalate to human review rather than proceeding automatically.

```
Agent 02 output received
      │
      ├── confidence ≥ threshold  → construct Agent 03 handoff, dispatch
      ├── confidence near threshold → dispatch, flag for closer review at gate
      └── confidence < threshold  → escalate to human review, pause pipeline
```

Centralising confidence gating at the orchestrator creates a single, consistent escalation policy. Thresholds are configured in one place and applied uniformly across all stage transitions.

### Error handling

The orchestrator handles agent failures without collapsing the full pipeline run:

| Failure type | Orchestrator behaviour |
|-------------|----------------------|
| Transient (timeout, network) | Retry with exponential backoff; log attempt |
| Non-retriable (malformed output, schema violation) | Escalate to human review; log with full context |
| Partial (one agent fails, others succeed) | Surface at approval gate; writer decides whether to proceed |

No failure is silent. Every error produces a log entry and a human-visible signal at the approval gate or escalation interface.

---

## Skills-type vs. Graph-type agents

The agent type choice has significant consequences for orchestration behaviour.

### Graph-type agents (avoid in production pipelines)

Graph-type agents maintain memory across turns. In a multi-turn orchestration flow, they accumulate context from previous turns — including context from earlier in the pipeline that is no longer relevant. This leads to:

- Hallucination on long runs as irrelevant context influences outputs
- Unpredictable behaviour that varies based on conversation history
- High token consumption as accumulated context grows

### Skills-type agents (preferred)

Skills-type agents execute purely from the system prompt and the current input. No memory accumulation. Each agent call is stateless — the output depends only on the system prompt and the handoff package received.

Benefits:
- Predictable, reproducible behaviour
- No context accumulation hallucination
- Dramatically lower token consumption
- Easier to test — same input always produces comparable output

**Empirical result:** re-architecting from Graph-type to Skills-type sub-agents produced a 78% token reduction (165k → 35.5k per pipeline run) with no degradation in output quality.

---

## Pin management

After deploying a new version of any sub-agent prompt, manually re-select that agent's version on the orchestrator configuration. Agent platform "latest" pointers can silently reference old pinned versions after a prompt deployment — the orchestrator may continue dispatching the previous version without any visible error.

Workflow after any sub-agent prompt update:
1. Deploy new prompt version
2. Verify deployment in the platform
3. Open orchestrator configuration
4. Re-select the updated sub-agent version explicitly
5. Verify the orchestrator references the new version before running tests

This is a platform quirk, not a design flaw — but it causes silent regressions if not managed deliberately.

---

## Context minimisation

A common failure mode in multi-agent pipelines is passing the full accumulated context of the run to every agent. This feels safe — the agent has everything it might need — but it has significant costs:

- Token consumption grows with every stage
- Agents are distracted by irrelevant context
- Prompt focus degrades as context grows
- Debugging becomes harder as handoff packages become larger

The correct approach is **purpose-built handoff packages**. The orchestrator extracts and structures exactly what each agent needs from the accumulated run context. Agents receive small, focused packages. Token consumption stays flat across pipeline stages.

---

## Minimal execution plans

Not every pipeline run should dispatch every agent. The orchestrator should construct the minimal execution plan that satisfies the routing manifest — and nothing more.

A change event that requires only a communications message should dispatch:
- Agent 04 (Communications) — draft the message
- Agent 06 (Governance) — log the run

It should not dispatch Impact Analysis or Content Prep — those agents have nothing to do and would consume tokens producing empty or irrelevant output.

Minimal execution plans reduce cost per run, reduce latency, and make the pipeline's behaviour easier to reason about. The routing manifest is the input to execution planning — use it fully.
