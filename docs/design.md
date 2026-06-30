# Design Document: Three-Axis Memory System

A personal knowledge management system that decomposes "memory" into three dimensions: event archive (time + dialogue), agent state (rules + snapshots), and project tracking. Covers the full knowledge lifecycle from raw data to vector retrieval to readable knowledge entries.

Not a replacement for vector databases—but an organizational framework on top of them.

---

## Why "Three Axes"

Problems with existing memory systems:

| Approach      | Problem                                      |
|---------------|----------------------------------------------|
| Pure vector   | Loses temporal order, can't separate "state" and "event" |
| MemGPT        | Limited context, no upper-level organization for long-term memory |
| Graph DB      | Heavy extraction overhead, overkill for personal use |
| Flat logs     | Low retrieval efficiency, hard to find "that thing from before" |

Three axes separate memory by **purpose** and **persistence strategy**, not by format:

| Axis    | Driving Property | Persistence Strategy | Purpose | Human Analogy |
|---------|-----------------|----------------------|---------|---------------|
| Center  | **Event-Driven** | Append-only, immutable | Time-index + dialogue archive | "What happened that day" + "exact words" |
| Hermes  | **State-Driven** | Last-write-wins + queryable rules | Current snapshot + rules engine + skills | "How I'm feeling right now" + "how to act" |
| Object  | **State-Driven** | Append changelog | Task/project tracking | "How's that project going" |

---

## Why Merge Timeline and Session

Both are event-driven. They always co-occur: every conversation creates a timeline entry AND a dialogue record. The original four-axis design required two concurrent writes of highly correlated data — every reply triggered a separate session write and a separate timeline write, even though they capture two facets of the same event.

Merging them into a single Center Axis:

- **Reduces write operations** — one axis handles both "what happened" and "exact words said"
- **Simplifies query** — a single `center/timeline.jsonl` entry links to its conversation archive date; no cross-axis join needed
- **Eliminates the "did I write both" concern** — one write path for event-driven data, no dual-write edge cases
- **Preserves separation of concerns** — the timeline is a thin index, the conversation archive holds bulk dialogue data; they share a namespace but serve different query patterns
- **Simplifies the mental model** — three axes instead of four, and the most common pair of concurrent writes becomes one

The Center Axis has two sub-components under one namespace:

```
center/
├── timeline.jsonl           # Thin time-index (one entry per event)
└── conversations/           # Bulk dialogue archive per date/channel
    └── YYYY-MM-DD-channel/
        └── reply-01.jsonl
```

A single `auto-update` command writes both: it always appends the timeline entry (if the event warrants one) and always appends the dialogue record. No caller needs to coordinate two separate axis operations.

---

## Event-Driven vs State-Driven: The Core Distinction

This is the most important design insight in this system.

Most memory systems treat all data the same way: either everything is append-only (infinite growth, hard to query current state), or everything overwrites (lose valuable history). Three axes make this distinction explicit.

### Event-Driven Axis (Center)

**Persist what happened, exactly as it happened. Never overwrite, never merge.**

- Every event is an independent, immutable record
- Ordered by time
- Used for: recalling the past, answering "what happened when" and "what exactly did they say"
- Examples: "2026-06-13: system first deployed", "2026-06-21: all three layers automated", conversation transcripts
- Concurrency strategy: **append-only**. History is sacred.

### State-Driven Axes (Hermes + Object)

**Persist what is true right now. Old states are replaced, but transitions are recorded.**

- Only the latest snapshot matters for direct query
- State transitions still leave a trace in changelogs
- Used for: knowing current mood/location/activity, checking project status, querying active rules and skills
- Examples: "mood: sleepy → hungry → content" (transition trace), "project: in_progress → completed"
- Concurrency strategy: **last-write-wins** for Hermes (fast overwrite of current state), **append changelog** for Object (traceable transitions)

### Why This Matters

A pure vector search can't tell "I bought a coat three years ago" (event) from "I just had lunch" (another event) — they're both just vectors. A flat log can't answer "what's the current mood" without scanning all entries. By separating persistence strategies, three axes let "remembering the past" and "querying the present" use different paths, without interfering with each other.

---

## Three-Layer Architecture

```
Layer 1: Three Axes (Raw Data Layer)

  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │   Center     │  │   Hermes     │  │   Object     │
  │  append-only │  │ last-write   │  │ changelog    │
  │ event-driven │  │ state-driven │  │ state-driven │
  │              │  │ + rules.db   │  │              │
  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
         │                  │                  │
         ▼                  ▼                  ▼

Layer 2: Semantic Retrieval (LanceDB + FTS5 + External Web)

  extract-fact → LLM extraction → embedding → LanceDB
  FTS5 full-text search on conversation archives
  External web search for live fact-checking

  Solves: "I remember something about X, but not which axis"
         + "Find the exact phrase someone said"
         + "What does the web say about this?"

         │
         ▼

Layer 3: Wiki Vault (Knowledge Base)

  concepts/   → conceptual definitions
  syntheses/  → distilled knowledge from dialogue
  entities/   → entity profiles (tools, people, config items)
  sources/    → fact provenance tracking

  Auto sediment: facts with importance ≥ 4 auto-create pages
```

---

## Design Principles

1. **Append-only for event-driven data** — history is immutable; never mutate events or dialogue
2. **Last-write-wins for state-driven data** — current state is always one read away; old state is still in the changelog
3. **Separate by responsibility, retrieve on demand** — don't overload one dimension
4. **Event vs state, explicit strategy** — each axis knows why it exists and uses the right persistence
5. **Agent-native (JSONL files, no external DB required)** — the agent persists memory directly to JSONL files as part of its reply loop. No separate storage service, no container, no microservices
6. **Auto-sediment, grow organically** — knowledge grows from conversation naturally; raw facts → vectorized → wiki entries

---

## Axis Specifications

### Center Axis — Event-Driven

Two sub-components under one `center/` namespace. Both are append-only and immutable.

#### center/timeline.jsonl — Time Index

One entry per important event. Thin index — enough to answer "what happened in what order" without reading bulk dialogue.

```json
{"ts": "2026-06-13T10:00:00+08:00", "type": "time_event", "event": "System Initialization", "level": "milestone", "summary": "First deployment"}
```

Fields:
- `ts`: ISO 8601 timestamp with timezone
- `type`: always `"time_event"` for timeline entries
- `event`: event name
- `level`: `daily` | `important` | `milestone`
- `summary`: event description

#### center/conversations/YYYY-MM-DD-channel/reply-01.jsonl — Dialogue Archive

Raw dialogue archive, organized by date and channel. Append-only. Mandatory write — every agent reply creates a conversation record.

```json
{"ts": "2026-06-22T09:21:00+08:00", "type": "conversation", "who": "user", "msg": "Good morning", "channel": "telegram"}
```

Fields:
- `ts`: ISO 8601 timestamp with timezone
- `type`: always `"conversation"` for dialogue entries
- `who`: speaker identifier (`"user"` or `"agent"`)
- `msg`: message content
- `channel`: source channel (e.g., `"telegram"`, `"discord"`, `"webui"`)

Used for exact recall — highest precision layer. "What exactly did they say that day."

---

### Hermes Axis — State-Driven

Evolved from the original "God Agent" concept. Now includes rules engine + skills tracking + system snapshots alongside user state.

#### hermes/user_status.jsonl — User State Snapshot

Live user state. Last-write-wins — only the latest entry is the current state.

```json
{"ts": "2026-06-22T09:21:19+08:00", "type": "user_state", "state": {"mood": "happy", "location": "home", "activity": "coding"}}
```

Concurrency: **last-write-wins**. Single-user, no collisions.

#### hermes/system_snapshot.jsonl — System State Snapshot

System status and capability summary. Last-write-wins.

```json
{"ts": "2026-06-22T09:21:19+08:00", "type": "system_snapshot", "rules_count": 39, "active_skills": ["search_files", "read_terminal"], "errors": 0}
```

#### hermes/rules.db — Queryable Rules Engine

SQLite-backed database of structured rules and skill mappings. Queryable at runtime — the agent can ask "what rules apply" or "what skills are available" without scanning JSONL.

- 39 rules with structured conditions and actions
- 74 skill-to-rule mappings
- Supports queries like: "Which rule handles file search requests?" or "What skills are registered for code review?"

---

### Object Axis — State-Driven

Independent task/role state tracking. Append changelog — each entry traces a state transition.

```json
{"ts": "2026-06-20T01:19:04+08:00", "type": "task_created", "name": "doc_design_v3", "state": "in_progress", "detail": "Converting to three-axis architecture"}
```

Fields:
- `ts`: ISO 8601 timestamp with timezone
- `type`: event type (e.g., `"task_created"`, `"state_change"`, `"task_completed"`)
- `name`: task or project identifier
- `state`: current status
- `detail`: human-readable description of the transition

Concurrency: **append changelog** — every state transition is recorded. The latest entry for a given `name` is the current state, but the full history is preserved.

---

## Update Model

| Axis    | When                  | How                                | Driving       |
|---------|-----------------------|------------------------------------|---------------|
| Center  | Every reply           | auto-write (timeline + conv archive) | Event-driven  |
| Hermes  | State/rule change     | auto-update + rule query            | State-driven  |
| Object  | Task progress         | Manual or auto                      | State-driven  |
| Semantic | Every reply          | extract-fact auto                   | Hybrid        |
| Wiki    | Important facts       | auto-sync from semantic layer       | Hybrid        |

### One Command

```bash
python3 autofour_v2.py --mode auto-update \
  --channel "channel" \
  --who "speaker" \
  --msg "dialog text" \
  --auto-state
```

The `auto-update` command evaluates each axis independently:
- Center: always writes (conversation archive mandatory + timeline if event-worthy)
- Hermes: writes only if state changed (auto-inferred from dialogue) or rules/skills updated
- Object: writes only if task progress was made

---

## Query Model

### Most queries don't need semantic search

| Query                              | Path                                              | Needs Vector? |
|------------------------------------|---------------------------------------------------|---------------|
| "What happened recently?"          | Center: last N timeline entries                   | ❌ Direct file read |
| "What'd they say that day?"        | Center: conversation archive for that date        | ❌ Direct file read |
| "How is she feeling?"              | Hermes: last user_status entry                    | ❌ Direct file read |
| "What rules apply?"                | Hermes: rules.db query                            | ❌ SQLite query |
| "What skills are used?"            | Hermes: system_snapshot skills_summary            | ❌ Direct file read |
| "How's that project going?"        | Object: task changelog for that name              | ❌ Direct file read |
| "I remember something about X"     | Semantic search (LanceDB + FTS5)                  | ✅ Only this one |

Three axes are designed so that **the axis itself is the index** — no vector search needed for the most common queries. Semantic search (Layer 2) exists only for the fuzzy cross-axis case: "I remember something but I don't know which axis it's in."

---

## Knowledge Lifecycle

### Three-Layer Sedimentation

```
Raw facts (three axes)
   ↓ LLM extract-fact
Vectorized knowledge (LanceDB + FTS5)
   ↓ importance ≥ 4 auto-sync
Readable entries (Wiki Vault)
```

### extract-fact Types

| Type            | Example                     | Storage                |
|-----------------|-----------------------------|------------------------|
| Tech config     | API key change, tool usage  | LanceDB + Wiki concepts|
| User preference | Likes, habits               | LanceDB + Wiki syntheses|
| Rules           | Behavior rules, safety words| LanceDB + Wiki syntheses|
| Skills          | What was learned            | LanceDB + Wiki syntheses|
| Relationship    | Emotional changes           | LanceDB + Wiki syntheses|
| Persona         | Background, traits          | LanceDB + Wiki entities|

---

## Application Scenarios

### Best Fit

| Scenario | Fit | Why |
|----------|-----|-----|
| **Personal AI assistant / companion** | ⭐⭐⭐⭐⭐ | Remembers preferences, habits, past conversations. "Last time you mentioned..." feels natural |
| **Long-running chatbot / virtual friend** | ⭐⭐⭐⭐⭐ | Foundation for long-term memory: shared experiences, consistent personality, no repeated questions |
| **Roleplay / character consistency** | ⭐⭐⭐⭐ | Hermes axis keeps character state consistent; Object tracks storylines |
| **Personal knowledge / second brain** | ⭐⭐⭐⭐ | Structured axes + Wiki sedimentation = auto-growing knowledge base |
| **Lightweight task management** | ⭐⭐⭐⭐ | Object axis naturally adapts to project/progress tracking |
| **Cross-channel memory** | ⭐⭐⭐ | Multi-channel merge supported, but single-user |
| **Enterprise / team use** | ⭐⭐ | Single-user architecture, no distributed support |

### Poor Fit
- High-concurrency read/write
- Multi-tenant SaaS
- Strict ACID transaction requirements
- Teams already invested in vector DB + graph DB + traditional DB (they don't need another abstraction layer)

---

## Comparison

| | **Three Axes** | MemGPT | Pure Vector | Graph DB | Single Log |
|---|---|---|---|---|---|
| **Time+Archive** | ✅ Center axis unified | ✅ Recursive summary | ❌ Loses order | ❌ | ❌ Scan all |
| **State+Rules** | ✅ Hermes axis | ❌ | ❌ Extra design needed | ❌ | ❌ |
| **Project isolation** | ✅ Object axis | ❌ | ❌ No grouping | ❌ Too heavy | ❌ |
| **Event+State dual strategy** | ✅ Explicit | ❌ Single | ❌ Single | ❌ Single | ❌ Single |
| **Queryable rules** | ✅ SQLite-backed | ❌ | ❌ | ❌ | ❌ |
| **Cross-axis semantic** | ✅ | ❌ | ✅ Core ability | ✅ Manual relations | ❌ |
| **Wiki sedimentation** | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Deploy complexity** | 🟢 **Low — single script** | 🟡 Medium | 🟡 Medium | 🔴 High | 🟢 Low |
| **Single user** | ✅ Naturally fits | ✅ Works | ⚠️ Overkill | ❌ Way too heavy | ✅ But low recall |

### Key Advantages

1. **Dual persistence strategy (event-driven + state-driven).** Most systems are either all-append (infinite growth) or all-overwrite (loss of history). Three axes let each data type use its natural strategy.

2. **Timeline and dialogue merged.** The two most tightly coupled data types (what happened + exact words) share one axis. No dual-write coordination, no cross-axis join for time-dialogue queries.

3. **Queryable rules engine (SQLite).** Rules and skill mappings live in a structured queryable database, not buried in free-form JSONL. The agent can ask "what rules apply right now" with a simple SQL query.

4. **Three-layer knowledge sedimentation pipeline.** Raw data → vector search → readable wiki, automatically. No manual curating required.

5. **Agent-native, no extra infrastructure.** Three axes are designed to run inside an AI agent runtime — no separate database server, no container orchestration, no microservices. The agent reads and writes JSONL directly as part of its reply loop.

6. **Queries don't need vector search.** 90% of memory lookups resolve at the axis level with direct file reads. Semantic search exists only for fuzzy cross-axis queries.

7. **One command writes all axes.** `auto-update` evaluates each axis independently and writes only what changed. No manual decisions needed.

8. **Immutability where it matters.** Event-driven data (Center axis) is never modified — history is preserved exactly as it happened.

---

## Comparison with Other "Memory for AI" Systems

| | **Three Axes** | MemGPT | Mem0 | Letta | Zep |
|---|---|---|---|---|---|
| Core mechanism | File + optional vector | Recursive summarization | Vector + graph | Agentic loop | Vector + LLM reasoning |
| Self-hostable | ✅ Full | ❌ Cloud-reliant | ✅ Self + Cloud | ❌ Cloud | ✅ Self + Cloud |
| Data format | JSONL + SQLite | Proprietary | Proprietary | Proprietary | Proprietary |
| Axis separation | ✅ Explicit (3) | ❌ Single timeline | ⚠️ Core+user | ❌ | ❌ |
| Persistence strategy | Event vs state dual | Single (all summary) | Single (vector) | Single | Single |
| Deploy | Just a script | Docker | Python SDK | Docker | Docker + services |

---

## Limitations

- No formal field definitions (engineering language, not academic)
- No large-scale validation (single-user scenario)
- Semantic retrieval and state inference depend on external LLM/embedding models
- No distributed/multi-tenant support
- Wiki writes are trigger-based (extract-fact), not real-time streaming
- No API for consumer applications (it's an agent-level system, not a service)

---

## Directory Structure

```
memory/
├── center/
│   ├── timeline.jsonl           # Time index (event-driven)
│   └── conversations/           # Dialogue archives (event-driven)
│       └── YYYY-MM-DD-channel/
│           └── reply-01.jsonl
├── hermes/
│   ├── user_status.jsonl        # User state (last-write-wins)
│   ├── system_snapshot.jsonl    # System state (last-write-wins)
│   └── rules.db                 # Queryable rules (SQLite)
├── object/
│   └── {name}.jsonl             # Task/role state (append changelog)
├── inventory.md                 # Auto-maintained item log
└── important_dates.md           # Auto-maintained dates

wiki/
├── concepts/                    # Conceptual definitions
├── syntheses/                   # Distilled knowledge entries
├── entities/                    # Entity profiles
├── sources/                     # Source tracking
└── reports/                     # Lint reports
```

---

**License:** MIT  
**Author:** Anonymous  
**Last updated:** 2026-06-30
