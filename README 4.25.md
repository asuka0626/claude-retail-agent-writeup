# Retail Competitive Intelligence Platform

An LLM-powered analytics agent built for an edible oil brand owner. Turns 12 months of supermarket sales data into autonomous market-share analysis — covering competitor visibility that the client's internal distribution system structurally lacks.

> **Note on scope:** This is a private repository. The README documents the architecture, engineering decisions, and lessons learned. Source code and the underlying dataset are not public due to client confidentiality. A walkthrough demo is available on request.

---

## Table of contents

- [Problem and product positioning](#problem-and-product-positioning)
- [System overview](#system-overview)
- [Agent architecture](#agent-architecture)
- [Engineering decisions](#engineering-decisions)
- [What went wrong (and what I learned)](#what-went-wrong-and-what-i-learned)
- [Stack](#stack)
- [Status](#status)

---

## Problem and product positioning

Brand owners in fast-moving consumer goods typically run their own distribution-tracking systems (ERP, internal sell-through dashboards). These systems answer the question *"how is **our** product moving?"* — but they are structurally blind to the same supermarket shelf's competitors.

This platform fills that gap. The data source is a single retailer's full-category sales feed, refreshed monthly. The deliverable is a chat-based AI analyst that lets non-technical business users ask open-ended questions in natural language and receive structured, evidence-backed answers — without writing SQL or opening a BI tool.

Example questions the agent can answer end-to-end:

- *"Why is our share growing while the overall market is shrinking?"*
- *"Which competitor is gaining the most ground in peanut oil this quarter?"*
- *"Are our gains driven by promotions or by pricing power?"*

For each question the agent autonomously decides which tools to call, in what order, and synthesizes the results into a **finding → evidence → reasoning → confidence** report.

---

## System overview

```
┌──────────────┐    SSE stream    ┌──────────────┐
│   React UI   │ ◄──────────────► │   FastAPI    │
│  (chat +     │                  │   backend    │
│   reasoning  │                  │              │
│   trace)     │                  └──────┬───────┘
└──────────────┘                         │
                                         ▼
                                ┌──────────────────┐
                                │   Agent loop     │
                                │  (Claude Sonnet) │
                                └──────┬───────────┘
                                       │  tool_use
                              ┌────────┴────────┐
                              ▼                 ▼
                        ┌──────────┐      ┌──────────┐
                        │ Analytics│      │  SQLite  │
                        │  tools   │ ───► │ (history)│
                        │ (4 fns)  │      └──────────┘
                        └────┬─────┘
                             ▼
                       ┌──────────┐
                       │ Parquet  │
                       │ master   │
                       │ (~73K    │
                       │  rows)   │
                       └──────────┘
```

**At a glance:** ~5,000–6,000 lines of code across backend and frontend. 8 REST/SSE endpoints. 4 analytics tools. Single-user single-agent (no orchestration layer).

---

## Agent architecture

### Why a custom ReAct loop and not a framework

I considered LangChain, LlamaIndex, and a few agent frameworks. I wrote a custom loop instead. The reasoning:

1. **Single-vendor, single-domain.** I only call Anthropic, and only for retail share analysis. I don't need provider abstraction.
2. **Streaming and concurrency control matter.** Tool concurrency, prompt caching placement, and SSE backpressure all need precise control. Framework abstractions get in the way.
3. **Debuggability.** When the agent does the wrong thing, I want to read 200 lines of agent code, not chase through a framework's call stack.

The loop itself is small — a Python `async generator` that yields events (tool_use, tool_result, text, end) to the FastAPI layer, which forwards them to the React frontend over SSE.

```
┌─────────────────────────────────────────────────────┐
│  while turn < MAX_TURNS:                            │
│    response = claude.messages.stream(...)           │
│    for block in response:                           │
│      yield block                                    │
│      if block.type == "tool_use":                   │
│        results = await execute_tools_concurrent(...)│
│        yield results                                │
│    if response.stop_reason == "end_turn": break     │
└─────────────────────────────────────────────────────┘
```

### Tools

Four read-only analytics tools, all backed by the same Parquet master table:

| Tool | Purpose |
|---|---|
| `monthly_overview` | Total market size + client share + YoY, with annual rollup |
| `brand_share` | Brand-level monthly share, filterable by subcategory |
| `promo_analysis` | Client vs competitor promotional reliance breakdown |
| `channel_share` | Online vs offline channel share movement |

Tool input schemas use JSON Schema `enum` for category codes — this is a hard-won design decision, see ["What went wrong"](#what-went-wrong-and-what-i-learned).

### Tool concurrency

When Claude requests multiple tools in the same turn, all of them are executed concurrently with `asyncio.gather`:

- Measured: 3 tools complete in ~0.17s (vs ~0.5s sequential).
- Per-tool result is row-capped at 200 rows; if exceeded, the tool returns a truncation notice instructing Claude to add stricter filters.
- Errors are isolated per tool — one failing tool doesn't kill the others.

### Prompt caching

`cache_control` blocks are placed on:

1. The system prompt (stable across all turns).
2. The last block of the tool schema (so the entire tool array is cached as a unit).

Verified cache hit: ~9.1K tokens reused per turn → ~60% reduction in per-turn input cost. Anthropic's docs are correct that placement matters — putting the marker on a different tool block gave 0% cache hit.

### Stop conditions

- `stop_reason == "end_turn"` (Claude finished naturally)
- `max_turns = 8` (hard ceiling against runaway loops)
- Tool execution exceptions are caught and returned to Claude as `tool_result` blocks with `is_error: true`, so it can recover or apologize gracefully.

---

## Engineering decisions

### Why not LangChain (the long version)

| Concern | Custom loop | LangChain |
|---|---|---|
| Streaming control | Full control over event ordering | Abstracted; debugging streaming bugs is hard |
| Tool concurrency | One line: `asyncio.gather(...)` | Need to hunt through the framework |
| Multi-turn message stitching | I own the message list, can fix any field bug | Framework owns it; opaque |
| Model swap | Change one string | Change ChatXxx class, prompt format may break |
| Upgrade churn | Anthropic SDK is stable | LangChain version changes are frequent |

### Why no ORM

Two tables, fixed query patterns. SQLAlchemy adds an abstraction layer, an N+1 risk, and a migration tool. Raw `sqlite3` + 40 lines of SQL is the simplest correct solution.

### Why no state management library on the frontend

Exactly one cross-page state: the most-recently-opened conversation ID (in `localStorage`). Everything else is component-local. Redux/Zustand/Jotai's boilerplate would cost more than the benefit.

### Why no Tailwind / UI kit

The visual design system is built around restraint:
- The brand color is a deep green (#047857), used sparingly
- No emojis, no big rounded corners, no soft shadows — Stripe/Notion aesthetic
- Tailwind's utility-first style encourages "stack whatever class you want," which makes restraint harder
- Antd / MUI defaults are "demo-style," not "professional restraint" — every component would need rewriting

Hand-written CSS variables + BEM-ish class names ended up being the cleanest path.

---

## What went wrong (and what I learned)

These are the bugs that mattered. I'm including them because **how someone debugs is more revealing than how someone designs**.

### 1. The dedup bug that inverted the YoY narrative

**The bug:** The original data pipeline used a composite primary key — `[year_month + channel + sku + customer_segment]` — for deduplication. This silently dropped 4,866 valid transactions where the same dimension legitimately had multiple distinct sales records.

**The impact:** The client's true YoY growth was a low-single-digit gain. The buggy pipeline reported a mid-single-digit *decline*. Some category-level numbers were inverted by 25+ percentage points. **The narrative was completely wrong.**

**How I caught it:** I didn't — the agent did. I asked Claude to write a sanity-check report comparing against an industry benchmark, and Claude flagged that the numbers didn't square with public market data. That triggered a manual audit, which led to the dedup logic.

**The fix:** Use full-row dedup (which actually drops 0 rows — the data is clean). When aggregation is needed, use `groupby + sum`, never `drop_duplicates`.

**The takeaway:** "LLM as data reviewer" is a real engineering pattern. The agent doesn't replace human judgment, but it surfaces inconsistencies humans miss because we're anchored to our own assumptions.

### 2. Subcategory codes are counter-intuitive

**The bug:** In the source data, `23602` is sunflower oil and `23600` is soybean oil. Anyone — human or LLM — guessing from intuition will get this wrong.

**The fix:** All subcategory parameters in the tool JSON Schema are locked with `enum`. Claude literally cannot pass an invalid code. This is a small change but it eliminated a whole class of "Claude misunderstood the parameter" bugs.

### 3. Extended Thinking pollutes the message history

**The bug:** When storing the assistant's response in `self.messages` for the next turn, using `block.model_dump()` pulls in SDK-internal fields like `parsed_output`. The next API call rejects the message: `Extra inputs are not permitted`.

**The fix:** Manually construct the message blocks, including only the fields the API accepts (`type / text / id / name / input / thinking / signature`). Don't blindly serialize SDK objects.

### 4. The wrong "main growth driver" story

**Initial human-written analysis:** Soybean oil is driving the share gain.

**Agent-driven analysis after running real data:** Soybean oil is only ~4% of the dataset. Even doubling its growth percentage barely moves the annual number. The real drivers were:

1. Improved pricing competitiveness in peanut oil (the largest category)
2. Structural offline channel share gains (~+10pp)
3. Reduced promotional reliance across the board

**The takeaway:** I had written the original business hypothesis. The agent ran the actual numbers and contradicted me. I updated the project summary with the agent's version. **LLMs as a check on human business assumptions is a real workflow** — not a gimmick.

---

## Stack

**Backend**
- Python 3.11
- FastAPI + uvicorn
- Anthropic SDK (Claude Sonnet 4)
- pandas + pyarrow (Parquet master)
- SQLite (conversation history)

**Frontend**
- React 18 (no framework, no state library)
- ECharts (custom theme registered globally)
- react-markdown + remark-gfm (agent message rendering)
- Native `fetch` + `ReadableStream` (SSE consumption)

**Tooling**
- Hand-rolled smoke tests (no pytest — the surface is small enough to script)
- Manual prompt-caching verification via the SDK's usage object

---

## Status

Production-deployed for client internal use. Currently iterating on:

- Conversation persistence (SQLite-backed, with conversation list sidebar)
- Inline chart rendering (agent emits a `<chart>` tag, frontend resolves it to a mini ECharts widget)
- Per-conversation cumulative token cost tracking

---

## Demo

Walkthrough video and live screen-share demo available on request. Reach out via the contact information on my resume.
