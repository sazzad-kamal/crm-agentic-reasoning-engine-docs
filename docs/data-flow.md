# Complete Data Flow Diagram

## Request Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           USER QUESTION                                      │
│  "What deals closed in Q1 and how does that compare to Q2?"                 │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STAGE 1: API Route [api/chat.py]                                           │
│  ─────────────────────────────────                                          │
│  POST /api/chat/stream                                                       │
│  • Validate request (question, session_id)                                  │
│  • Create StreamingResponse                                                  │
│  • Pass to stream_agent()                                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STAGE 2: Supervisor Node [agent/supervisor/node.py]                        │
│  ───────────────────────────────────────────────────                        │
│  Intent Classification (90% heuristic, 10% LLM):                            │
│                                                                             │
│  Heuristics (no LLM call):                                                  │
│  • "vs", "compare" → COMPARE                                                │
│  • "trend", "over time" → TREND                                             │
│  • "how do I", "how to" → DOCS                                              │
│  • "connected to", "relationship" → GRAPH                                   │
│  • Health keywords → HEALTH                                                 │
│  • Matches >= 2 source buckets → COMPLEX (multi-source)                     │
│  • Short/vague → CLARIFY                                                    │
│                                                                             │
│  LLM Fallback (ambiguous queries):                                          │
│  • Classifier prompt → One word response                                    │
│                                                                             │
│  This question: "compare" + "deals closed" → multi-part fan-out needed      │
│  Result: COMPLEX (multi-part query)                                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

(See [README § Intent Classification](../README.md) for measurement methodology and the heuristic pattern table.)

```
                                    │
                                    ▼ Intent: COMPLEX
┌─────────────────────────────────────────────────────────────────────────────┐
│  STAGE 3: Planner Node [agent/planner/node.py]                              │
│  ─────────────────────────────────────────────                              │
│  Decompose into sub-queries:                                                │
│                                                                             │
│  Original: "What deals closed in Q1 and how does that compare to Q2?"      │
│                                                                             │
│  Decomposed (sub-query types: fetch, compare, trend, rag, graph):           │
│  1. fetch   : "What deals closed in Q1?"                                    │
│  2. fetch   : "What deals closed in Q2?"                                    │
│  3. compare : "Compare Q1 deals vs Q2 deals"                                │
│                                                                             │
│  Fan-out: Execute 1 & 2 in parallel, then 3 with aggregated results.        │
│  Each sub-query routes to the right data agent (fetch/compare/trend →       │
│  sql_node with intent; rag → rag_node; graph → graph_node).                 │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
┌───────────────────────────────────┐ ┌───────────────────────────────────┐
│  STAGE 4a: SQL Node (Q1, fetch)   │ │  STAGE 4b: SQL Node (Q2, fetch)   │
│  [agent/sql/node.py → intents/    │ │  [agent/sql/node.py → intents/    │
│   fetch.py]                       │ │   fetch.py]                       │
│  1. Generate SQL via LLM:         │ │  1. Generate SQL via LLM:         │
│     SELECT * FROM opportunities   │ │     SELECT * FROM opportunities   │
│     WHERE close_date >= '2024-01' │ │     WHERE close_date >= '2024-04' │
│     AND close_date < '2024-04'    │ │     AND close_date < '2024-07'    │
│     AND status = 'Closed Won'     │ │     AND status = 'Closed Won'     │
│                                   │ │                                   │
│  2. SQL Guard validates:          │ │  2. SQL Guard validates:          │
│     ✓ SELECT only                 │ │     ✓ SELECT only                 │
│     ✓ No forbidden functions      │ │     ✓ No forbidden functions      │
│     ✓ Auto-add LIMIT 1000         │ │     ✓ Auto-add LIMIT 1000         │
│                                   │ │                                   │
│  3. Execute on DuckDB             │ │  3. Execute on DuckDB             │
│                                   │ │                                   │
│  Result: 12 deals, $450K total    │ │  Result: 8 deals, $320K total     │
└───────────────────────────────────┘ └───────────────────────────────────┘
                    │                               │
                    └───────────────┬───────────────┘
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STAGE 5: SQL Node (compare) [agent/sql/intents/compare.py]                 │
│  ─────────────────────────────────────────────                              │
│  Compare aggregated results:                                                │
│                                                                             │
│  Input:                                                                     │
│  • Q1: 12 deals, $450K                                                      │
│  • Q2: 8 deals, $320K                                                       │
│                                                                             │
│  Analysis:                                                                  │
│  • Deal count: Q1 > Q2 by 4 (33% more)                                     │
│  • Revenue: Q1 > Q2 by $130K (29% more)                                    │
│  • Avg deal size: Q1 $37.5K vs Q2 $40K (Q2 higher)                         │
│                                                                             │
│  Output: Structured comparison data                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STAGE 6: Answer Node [agent/answer/node.py]                                │
│  ───────────────────────────────────────────                                │
│  Generate grounded response:                                                │
│                                                                             │
│  Input:                                                                     │
│  • Original question                                                        │
│  • Fetched data (Q1 deals, Q2 deals)                                       │
│  • Comparison analysis                                                      │
│                                                                             │
│  Output (with evidence tags):                                               │
│  "Q1 had 12 closed deals [E1] totaling $450K [E2], while Q2 had            │
│   8 deals [E3] for $320K [E4]. Q1 outperformed Q2 by 33% in deal           │
│   count [E5] and 29% in revenue [E6]."                                      │
│                                                                             │
│  Evidence:                                                                  │
│  • E1: COUNT(*) FROM opportunities WHERE Q1 = 12                           │
│  • E2: SUM(value) FROM opportunities WHERE Q1 = 450000                     │
│  • E3-E6: (similarly grounded)                                              │
│                                                                             │
│  Refinement Loop (if needs_more_data):                                      │
│  • Answer judges evidence sufficiency across SQL/RAG/Graph                  │
│  • Sets refetch_target + refetch_hint (max 2 iterations, global cap)        │
│  • Example: "Also show top deals" → refetch_target=sql, re-runs SQL         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STAGE 7: Contract Validation [agent/validate/]                             │
│  ──────────────────────────────────────────────                             │
│  Every output validated:                                                    │
│                                                                             │
│  validate(answer)                                                           │
│      │                                                                      │
│      ├── Valid? ───────────────────────► Return                            │
│      │                                                                      │
│      ▼                                                                      │
│  repair(answer, errors)                                                     │
│      │                                                                      │
│      ├── Valid? ───────────────────────► Return (repaired)                 │
│      │                                                                      │
│      ▼                                                                      │
│  fallback()                                                                 │
│      │                                                                      │
│      └─────────────────────────────────► Return (safe default)             │
│                                                                             │
│  Checks:                                                                    │
│  • Grounding: All claims reference fetched data                            │
│  • Evidence: E1, E2... tags present and valid                              │
│  • Length: Answer not too short/long                                       │
│  • Format: Proper structure                                                │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
┌───────────────────────────────────┐ ┌───────────────────────────────────┐
│  STAGE 8a: Action Node            │ │  STAGE 8b: Followup Node          │
│  [agent/action/node.py]           │ │  [agent/followup/node.py]         │
│  ─────────────────────            │ │  ─────────────────────────        │
│  Suggest CRM actions:             │ │  Suggest follow-up questions:     │
│                                   │ │                                   │
│  Based on analysis:               │ │  Based on context:                │
│  • "Review Q2 pipeline"           │ │  • "What caused the Q2 decline?"  │
│  • "Schedule Q3 forecast meeting" │ │  • "Show deals by sales rep"      │
│  • "Update opportunity stages"    │ │  • "What's the Q3 pipeline?"      │
│                                   │ │                                   │
│  Each action validated:           │ │  Each question validated:         │
│  • action_type: valid enum        │ │  • Relevant to conversation       │
│  • entity_id: exists in data      │ │  • Not already answered           │
│  • description: clear/actionable  │ │  • Diverse (not repetitive)       │
└───────────────────────────────────┘ └───────────────────────────────────┘
                    │                               │
                    └───────────────┬───────────────┘
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STAGE 9: SSE Streaming [agent/streaming.py]                                │
│  ───────────────────────────────────────────                                │
│  Events sent to frontend:                                                   │
│                                                                             │
│  1. fetch_start     →  "Querying Q1 deals..."                              │
│  2. fetch_complete  →  {rows: 12, query: "..."}                            │
│  3. fetch_start     →  "Querying Q2 deals..."                              │
│  4. fetch_complete  →  {rows: 8, query: "..."}                             │
│  5. answer_chunk    →  "Q1 had 12 closed deals..."  (streaming)            │
│  6. answer_chunk    →  "[E1] totaling $450K..."     (streaming)            │
│  7. evidence        →  [{tag: "E1", source: "..."}]                        │
│  8. action          →  [{type: "review", ...}]                             │
│  9. followup        →  ["What caused Q2 decline?", ...]                    │
│  10. done           →  {session_id: "..."}                                 │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  FRONTEND: Display response with evidence, actions, follow-ups              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Agent Routing Summary

| Intent | Agent Path | Description |
|--------|------------|-------------|
| DATA_QUERY | Supervisor → SQL (fetch) → Answer → [Action, Followup] | Simple data lookup |
| COMPARE | Supervisor → SQL (compare) → Answer → [Action, Followup] | A vs B analysis |
| TREND | Supervisor → SQL (trend) → Answer → [Action, Followup] | Time-series |
| HEALTH | Supervisor → SQL (health) → Answer → [Action, Followup] | Account scoring |
| DOCS | Supervisor → RAG → Answer → [Action, Followup] | Documentation |
| GRAPH | Supervisor → Graph → Answer → [Action, Followup] | Multi-hop relationships |
| COMPLEX | Supervisor → Planner → fan-out (SQL + RAG + Graph) → Answer → [Action, Followup] | Cross-source multi-part |
| CLARIFY | Supervisor → Answer (asks for clarification) | Vague query |
| HELP | Supervisor → Answer (no data fetch needed) | General help |

See [ARCHITECTURE.md § Supervisor Routing](./ARCHITECTURE.md#supervisor-routing) for the canonical intent-to-agent mapping.

## Universal Re-Fetch Loop

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  UNIVERSAL RE-FETCH (global cap: 2 iterations)                              │
│  ───────────────────────────────────────────                                │
│                                                                             │
│  Answer node evaluates evidence sufficiency across ALL sources              │
│  (SQL, RAG, Graph). Sets:                                                   │
│    needs_more_data: bool                                                    │
│    refetch_target : "sql" | "rag" | "graph" | None                         │
│    refetch_hint   : freeform string ("broaden doc scope", "expand hops")   │
│                                                                             │
│  _route_after_answer dispatches back to the targeted source:                │
│    • SQL  : re-invoke sql_node with preserved intent                        │
│    • RAG  : bump top_k, append hint as "Focus: ..." to the query           │
│    • Graph: expand hop depth (2 → 3), widen relationship types             │
│                                                                             │
│  Example:                                                                   │
│    User: "Show Q1 deals and their contacts"                                 │
│    Iter 1: SQL(fetch) → 12 deals. Answer judges contact detail missing.     │
│            refetch_target=sql, refetch_hint="include primary contact per    │
│            opportunity"                                                     │
│    Iter 2: SQL(fetch) re-runs with hint → contacts joined in.               │
│            Answer complete.                                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Cross-Source Request Lifecycle (Flagship Scenario)

```
User: "For Acme's renewal, is Acme using Act! Marketing Automation,
        and who on their buying committee is connected to our competitors?"
  │
  ▼
Supervisor  → matches SQL + docs + graph patterns → intent = COMPLEX
  │
  ▼
Planner     → decomposes into 3 typed sub-queries:
                1. fetch : "Acme AMA usage + renewal status"
                2. rag   : "Act! Marketing Automation"
                3. graph : "Acme buying committee ↔ competitor connections"
  │
  ├───────────────────┬───────────────────┐
  ▼                   ▼                   ▼
sql_node(fetch)     rag_node            graph_node
  │                   │                   │
  └────────┬──────────┴─────────┬─────────┘
           ▼                    ▼
  Aggregator fills:  data, rag_results, graph_results
                     (+ sql_errors, rag_errors, graph_errors for partial
                     failures)
  │
  ▼
Answer node  → cross-source synthesis prompt:
                 weave SQL rows with RAG snippets and graph relationships.
                 Tags: [E#]=sql, [D#]=docs, [G#]=graph.
               Sufficiency judge: if any source underdelivered, set
                 refetch_target + refetch_hint and loop.
  │
  ▼
Action + Followup → SSE stream → client
```

## SSE Event Types

| Event | When | Payload |
|-------|------|---------|
| `fetch_start` | Query begins | `{message: "Querying..."}` |
| `fetch_complete` | Query done | `{rows: N, query: "..."}` |
| `answer_chunk` | Token streamed | `{content: "..."}` |
| `evidence` | Citations ready | `[{tag, source, value}]` |
| `action` | Actions suggested | `[{type, entity_id, description}]` |
| `followup` | Questions ready | `["question1", "question2"]` |
| `error` | Error occurred | `{message: "..."}` |
| `done` | Stream complete | `{session_id: "..."}` |

## Key Performance Characteristics

| Metric | Target | How |
|--------|--------|-----|
| Time to first token | < 500ms | Streaming SSE |
| Classification latency | < 100ms | 90% heuristic |
| Query execution | < 2s | DuckDB in-memory |
| Total response | < 5s | Parallel fan-out |
