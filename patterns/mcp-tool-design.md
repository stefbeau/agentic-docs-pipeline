# Pattern: MCP Tool Design

## What it is

A set of design principles for building MCP (Model Context Protocol) tools that agents can call reliably in a production pipeline. MCP tools are the interface between agents and live data — page content, metadata, search indexes, external systems.

Getting tool design right is what separates a pipeline that works in demos from one that works in production.

---

## Core principles

### One tool, one responsibility

Each MCP tool does one thing. A tool that fetches page content and also updates metadata is two tools collapsed into one. Split them.

Single-responsibility tools are easier to test, easier to version independently, and easier to replace when the underlying data source changes. They also make agent behaviour more predictable — an agent that calls `get_page_content` will always get page content, nothing else.

### Deterministic inputs, structured outputs

Tool inputs should be typed and validated. Tool outputs should be structured JSON — not free text that agents have to parse. An agent that receives a structured output can extract fields reliably. An agent that receives a text blob has to interpret it, which introduces non-determinism.

```json
// Good — structured output
{
  "page_id": "...",
  "title": "...",
  "content": "...",
  "last_updated": "2025-03-14T10:22:00Z",
  "status": "published"
}

// Avoid — text blob
"Page title: Getting Started\nLast updated: March 14 2025\nContent: ..."
```

### Fail loudly, fail specifically

Tools should return typed error responses, not generic failures. An agent that receives `{"error": "not found"}` knows the page doesn't exist. An agent that receives `{"error": "something went wrong"}` cannot distinguish a missing page from a network timeout from an authentication failure — and will behave unpredictably in each case.

Define your error taxonomy and return specific error types:

```json
{
  "error": "PAGE_NOT_FOUND",
  "page_id": "...",
  "message": "No page exists with this identifier"
}
```

### Tools do not make decisions

MCP tools retrieve, fetch, search, and write. They do not classify, evaluate, or recommend. A tool that returns "this page needs updating" has crossed into agent territory. Keep tools as pure data access — let agents do the reasoning.

---

## FastMCP implementation notes

### SSE response handling

FastMCP returns SSE (Server-Sent Events) format even for single-result calls. Parse the response correctly:

```javascript
// Correct — parse SSE lines
const lines = responseText.split('\n');
const dataLine = lines.find(l => l.startsWith('data: '));
const payload = JSON.parse(dataLine.replace('data: ', ''));

// Incorrect — will fail
const payload = await response.json();
```

Use `Accept: application/json, text/event-stream` in request headers.

### Direct calls vs. gateway

For expensive pre-computation steps that run before agents fire — corpus searches, page retrieval, metadata lookups — call MCP tools directly rather than routing through the agent platform gateway. Direct calls bypass platform timeouts and agent overhead when sub-agent intelligence isn't needed.

Direct call pattern:
```
Dashboard / pre-script
      │
      └──▶ MCP server (direct) ──▶ structured result
```

Gateway pattern (for agent-driven calls):
```
Agent
  │
  └──▶ Agent Platform ──▶ MCP server ──▶ structured result
```

Use direct calls for pre-computation. Use the gateway for agent-driven tool calls where the agent needs to decide whether to call the tool at all.

### Tool registration and discovery

Tools are discovered by the agent platform via the tool catalog, triggered by **Upload Config** in the deployment pipeline — not by release to staging. Release to staging is a separate step that does not affect tool catalog visibility.

After any tool deployment, verify catalog discovery before testing agent behaviour. A tool that isn't in the catalog is invisible to agents regardless of whether it deployed successfully.

---

## Tool categories in a documentation pipeline

| Category | Examples | Notes |
|----------|---------|-------|
| **Retrieval** | `get_page_content`, `get_page_metadata`, `search_corpus` | Most frequently called; cache where appropriate |
| **Write** | `update_page_draft`, `publish_approved_content` | Require human approval gate before calling |
| **Search** | `vector_search`, `keyword_search`, `find_related_pages` | Parameterise by scope to avoid full-corpus searches |
| **Metadata** | `get_change_event`, `get_approval_status`, `get_run_log` | Read-only; fast |
| **Notification** | `draft_email`, `queue_notification` | Draft only — no send capability in pipeline tools |

### Write tool guardrails

Write tools — tools that modify the corpus or send communications — should have explicit guardrails:

1. **Require approval status check** before executing. A write tool called before the approval gate has cleared should return an error, not proceed.
2. **Log every call** with the caller identity, timestamp, and the payload written.
3. **No send capability in pipeline tools**. Draft tools produce content. Dispatch is a separate, post-approval action outside the pipeline.

---

## Testing MCP tools

### Test at the tool layer, not the agent layer

Test tool behaviour independently of agent behaviour. An agent test that fails because the underlying tool returned unexpected data is harder to debug than a tool test that catches the issue directly.

### Fixture inputs for retrieval tools

Use fixture inputs — known page identifiers with known content — for retrieval tool tests. This makes tool tests deterministic and independent of the live corpus state.

### Error path coverage

Test error paths explicitly. Every defined error type should have a test that triggers it. Tools with untested error paths produce unexpected agent behaviour when those errors occur in production.  
