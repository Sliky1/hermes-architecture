# Design Document: Four-Axis Memory System

A personal knowledge management system that decomposes "memory" into four dimensions: time, state, project, and dialogue. Covers the full knowledge lifecycle from raw data to vector retrieval to readable knowledge entries.

Not a replacement for vector databases—but an organizational framework on top of them.

---

## Why "Four Axes"

Problems with existing memory systems:

| Approach      | Problem                                      |
|---------------|----------------------------------------------|
| Pure vector   | Loses temporal order, can't separate "state" and "event" |
| MemGPT        | Limited context, no upper-level organization for long-term memory |
| Graph DB      | Heavy extraction overhead, overkill for personal use |
| Flat logs     | Low retrieval efficiency, hard to find "that thing from before" |

Four axes separate memory by **purpose** and **persistence strategy**, not by format:

| Axis      | Driving Property | Persistence Strategy | Purpose | Human Analogy |
|-----------|-----------------|----------------------|---------|---------------|
| Timeline  | **Event-Driven** | Append-only, immutable | What happened, in order | "What happened that day" |
| God Agent | **State-Driven** | Last-write-wins | Current snapshot of who I am | "How I'm feeling right now" |
| Object    | **State-Driven** | Append changelog | Task/project tracking | "How's that project going" |
| Session   | **Event-Driven** | Append-only, immutable | Raw dialogue archive | "What exactly did they say" |

## Event-Driven vs State-Driven: The Core Distinction

This is the most important design insight in this system.

Most memory systems treat all data the same way: either everything is append-only (infinite growth, hard to query current state), or everything overwrites (lose valuable history). Four axes make this distinction explicit.

### Event-Driven Axes (Timeline + Session)

**Persist what happened, exactly as it happened. Never overwrite, never merge.**

- Every event is an independent, immutable record
- Ordered by time
- Used for: recalling the past, answering "what happened when"
- Examples: "2026-06-13: system first deployed", "2026-06-21: all three layers automated"
- Concurrency strategy: **append-only**. History is sacred.

### State-Driven Axes (God Agent + Object)

**Persist what is true right now. Old states are replaced, but transitions are recorded.**

- Only the latest snapshot matters for direct query
- State transitions still leave a trace in changelogs
- Used for: knowing current mood/location/activity, checking project status
- Examples: "mood: sleepy → hungry → content" (transition trace), "project: in_progress → completed"
- Concurrency strategy: **last-write-wins** for God Agent (fast overwrite of current state), **append changelog** for Object (traceable transitions)

### Why This Matters

A pure vector search can't tell "I bought a coat three years ago" (event) from "I just had lunch" (another event) — they're both just vectors. A flat log can't answer "what's the current mood" without scanning all entries. By separating persistence strategies, four axes let "remembering the past" and "querying the present" use different paths, without interfering with each other.

---

## Three-Layer Architecture

```
┌──────────────────────────────────────────────────────────────┐
│              Layer 1: Four Axes (Raw Data)                    │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────┐│
│  │   Timeline   │  │  God Agent   │  │   Object     │  │Sess││
│  │  append-only │  │ last-write   │  │ changelog    │  │app-││
│  │  event-driven│  │ state-driven │  │ state-driven │  │only││
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──┬─┘│
│         │                  │                  │             │   │
└─────────┼──────────────────┼──────────────────┼─────────────┼───┘
          │                  │                  │             │
          ▼                  ▼                  ▼             ▼
┌──────────────────────────────────────────────────────────────┐
│         Layer 2: Semantic Retrieval (LanceDB + Embedding)     │
│                                                              │
│  extract-fact → LLM extraction → embedding → LanceDB        │
│  Cross-axis semantic search via memory_recall.py             │
│                                                              │
│  Solves: "I remember something about X, but not which axis"  │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│              Layer 3: Wiki Vault (Knowledge Base)             │
│                                                              │
│  concepts/   → conceptual definitions                        │
│  syntheses/  → distilled knowledge from dialogue              │
│  entities/   → entity profiles (tools, people, config items)  │
│  sources/    → fact provenance tracking                       │
│                                                              │
│  Auto sediment: facts with importance ≥ 4 auto-create pages  │
└──────────────────────────────────────────────────────────────┘
```

---

## Design Principles

1. **Append-only, never mutate** — history is immutable
2. **Separate by responsibility, retrieve on demand** — don't overload one dimension
3. **Event vs state, explicit strategy** — each axis knows why it exists
4. **Local-first, API-enhanced** — agent persists memory directly to JSONL files, no separate storage service required
5. **Auto-sediment, grow organically** — knowledge grows from conversation naturally

---

## Axis Specifications

### Timeline (`timeline.jsonl`) — Event-Driven

Temporal event index. One entry per event. Append-only.

```json
{"ts": "2026-06-21T22:54:45+08:00", "event": "System Upgrade", "level": "milestone", "summary": "All three layers automated"}
```

Fields:
- `ts`: ISO 8601 timestamp with timezone
- `event`: event name
- `level`: daily | important | milestone
- `summary`: event description

### God Agent (`god/{agent}.jsonl`) — State-Driven

Live state snapshot chain. Last-write-wins. Only the latest entry is the current state.

```json
{"ts": "2026-06-22T09:21:19+08:00", "type": "state_update", "state": {"mood": "sleepy", "location": "home", "activity": "coding"}}
```

Concurrency: **last-write-wins**. Single-user, no collisions.

### Session (`session/YYYY-MM-DD-channel/reply-01.jsonl`) — Event-Driven

Raw dialogue archive, organized by date and channel. Append-only. Mandatory write.

```json
{"ts": "2026-06-22T09:21:00+08:00", "who": "user", "msg": "Good morning", "channel": "telegram"}
```

Highest precision layer. Used for exact recall.

### Object (`object/{name}.jsonl`) — State-Driven

Independent task/role state tracking. Append changelog — each entry traces a state transition.

```json
{"ts": "2026-06-20T01:19:04+08:00", "event": "task_created", "name": "doc_design_v2", "state": "in_progress", "detail": "Adding three-layer architecture"}
```

---

## Update Model

| Axis      | When                  | How                     | Driving       | Concurrency      |
|-----------|-----------------------|-------------------------|---------------|------------------|
| Timeline  | Important event       | `--event "name"`        | Event-driven  | Append-only      |
| God Agent | State change          | `--state '{}'`          | State-driven  | Last-write-wins  |
| Session   | Every reply (forced)  | `--msg "text"`          | Event-driven  | Append-only      |
| Object    | Task progress         | Manual jsonl write      | State-driven  | Append changelog |
| Semantic  | Every reply           | `extract-fact` auto     | —             | Upsert           |
| Static    | Auto scan             | `check-static`          | —             | Append-only      |
| Wiki      | Important facts       | `extract-fact` auto sync| —             | Append or create |

### One Command

```bash
python3 autofour_v2.py --mode auto-update \
  --channel "channel" \
  --who "speaker" \
  --msg "dialog text" \
  --auto-state
```

The `auto-update` command evaluates each axis independently:
- Session: always writes
- God Agent: writes only if state changed (auto-inferred from dialogue)
- Timeline: writes only if a significant event occurred
- Object: writes only if task progress was made

---

## Query Model

### Most queries don't need semantic search

| Query                              | Path                           | Axis Type   | Needs Vector? |
|------------------------------------|--------------------------------|-------------|---------------|
| "What happened recently?"          | Last N entries in timeline     | Event       | ❌ Direct file read |
| "How is she feeling?"              | Last 1 entry in god agent      | State       | ❌ Direct file read |
| "What'd they say that day?"        | Session file for that date     | Event       | ❌ Direct file read |
| "How's that project going?"        | Object file for that task      | State       | ❌ Direct file read |
| "I remember something about..."    | Semantic search (cross-axis)   | Hybrid      | ✅ Only this one |
| "What's the system architecture?"  | Wiki concepts                  | Knowledge   | ❌ File check |
| "We discussed this knowledge"      | Wiki syntheses                 | Knowledge   | ❌ File check |

Four axes are designed so that **the axis itself is the index** — no vector search needed for the most common queries. Semantic search (Layer 2) exists only for the fuzzy cross-axis case: "I remember something but I don't know which axis it's in."

---

## Knowledge Lifecycle

### Three-Layer Sedimentation

```
Raw facts (four axes)
   ↓ LLM extract-fact
Vectorized knowledge (LanceDB)
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
| **Roleplay / character consistency** | ⭐⭐⭐⭐ | God Agent keeps character state consistent; Object tracks storylines |
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

| | **Four Axes** | MemGPT | Pure Vector | Graph DB | Flat Log |
|---|---|---|---|---|---|
| **Time awareness** | ✅ Timeline (event-driven) | ✅ Recursive summary | ❌ Loses order | ❌ | ❌ Scan all |
| **State awareness** | ✅ God Agent (state-driven) | ❌ | ❌ Extra design needed | ❌ | ❌ |
| **Project isolation** | ✅ Object (state-driven) | ❌ | ❌ No grouping | ❌ Too heavy | ❌ |
| **Raw dialogue** | ✅ Session (event-driven) | ❌ Compressed loses info | ✅ Unstructured | ❌ | ✅ Unstructured |
| **Semantic search** | ⚠️ Optional enhancement | ❌ | ✅ Core ability | ❌ | ❌ |
| **Readable knowledge** | ✅ Wiki auto-sediment | ❌ | ❌ | ❌ | ❌ |
| **Deploy complexity** | 🟢 **Low — single script** | 🟡 Medium | 🟡 Medium | 🔴 High | 🟢 Low |
| **Memory strategy** | 🎯 **Dual: event + state** | ❌ Single | ❌ Single | ❌ Single | ❌ Single |
| **Single user** | ✅ Naturally fits | ✅ Works | ⚠️ Overkill | ❌ Way too heavy | ✅ But low recall |
| **Cross-axis query** | ✅ Automatic | ❌ | ❌ | ✅ Manual relations | ❌ |

### Key Advantages

1. **Dual persistence strategy (event-driven + state-driven).** Most systems are either all-append (infinite growth) or all-overwrite (loss of history). Four axes let each data type use its natural strategy.

2. **Three-layer knowledge sedimentation pipeline.** Raw data → vector search → readable wiki, automatically. No manual curating required.

3. **Agent-native, no extra infrastructure.** Four axes is designed to run inside an AI agent runtime — no separate database server, no container orchestration, no microservices. The agent reads and writes JSONL directly as part of its reply loop. A single python script handles all axis operations. Optional LLM/embedding for enhanced features.

4. **Designed for "memory" in the human sense.** Not a generic knowledge management tool — it's built for the relationship between a user and their AI. Timeline feel, state consistency, dialogue recall are core requirements, not afterthoughts.

5. **Queries don't need vector search.** 90% of memory lookups resolve at the axis level with direct file reads. Semantic search exists only for fuzzy cross-axis queries.

6. **One command writes all axes.** `auto-update` evaluates each axis independently and writes only what changed. No manual decisions needed.

7. **Immutability where it matters.** Event-driven data is never modified — history is preserved exactly as it happened.

---

## Comparison with Other "Memory for AI" Systems

| | **Four Axes** | MemGPT | Mem0 | Letta | Zep |
|---|---|---|---|---|---|
| Core mechanism | File + optional vector | Recursive summarization | Vector + graph | Agentic loop | Vector + LLM reasoning |
| Self-hostable | ✅ Full | ❌ Cloud-reliant | ✅ Self + Cloud | ❌ Cloud | ✅ Self + Cloud |
| Data format | JSONL | Proprietary | Proprietary | Proprietary | Proprietary |
| Axis separation | ✅ Explicit (4) | ❌ Single timeline | ⚠️ Core+user | ❌ | ❌ |
| Pair-of-AI design | ✅ Built for | ❌ | ❌ | ❌ | ❌ |
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
├── timeline.jsonl           # Event index
├── god/
│   ├── agent.jsonl          # State snapshots
│   └── user.jsonl
├── session/
│   └── YYYY-MM-DD-channel/  # Dialogue archives
├── object/
│   └── {name}.jsonl         # Task/role state
├── inventory.md             # Auto-maintained item log
└── important_dates.md       # Auto-maintained dates

wiki/
├── concepts/                # Conceptual definitions
├── syntheses/               # Distilled knowledge entries
├── entities/                # Entity profiles
├── sources/                 # Source tracking
└── reports/                 # Lint reports
```

---

**License:** MIT  
**Author:** Anonymous  
**Last updated:** 2026-06-30
