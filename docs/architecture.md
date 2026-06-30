# Architecture Overview

## Core Distinction: Event-Driven vs State-Driven

The three axes are grouped by their persistence strategy, not just by their content type.

```
┌─────────────────────────────────────────────────────────────┐
│                 Three Axes (Raw Data Layer)                  │
│                                                             │
│  ┌──── EVENT-DRIVEN ─────┐  ┌──── STATE-DRIVEN ──────┐    │
│  │                       │  │                         │    │
│  │     Center Axis       │  │      Hermes Axis        │    │
│  │   ├─ Time Index       │  │   last-write-wins       │    │
│  │   │  (append-only)    │  │   current state         │    │
│  │   ├─ Conv Archive     │  │   + queryable rules     │    │
│  │      (append-only)    │  │   + skill_map           │    │
│  │                       │  │   + system snapshots    │    │
│  │   "what happened"     │  │   "current status"      │    │
│  │   + "exact words"     │  │   + "how to behave"     │    │
│  ├───────────────────────┤  ├─────────────────────────┤    │
│  │     Object Axis       │  │                         │    │
│  │   append changelog    │  │                         │    │
│  │   task transitions    │  │                         │    │
│  │   "project status"    │  │                         │    │
│  └───────────────────────┘  └─────────────────────────┘    │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│   Semantic Retrieval (LanceDB / FTS5) / Knowledge API Layer  │
│                                                              │
│  Cross-axis semantic search: "I remember something about..." │
│  extract-fact → embed → search OR jieba FTS5 direct         │
│                                                              │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│              Wiki Vault (Knowledge Base Layer)                │
│                                                              │
│  Auto-sedimented knowledge from conversation                 │
│  importance ≥ 4 → auto wiki page                             │
│                                                              │
│  concepts/ · syntheses/ · entities/ · sources/               │
└──────────────────────────────────────────────────────────────┘
```

## Why Merge Timeline and Session?

Both are event-driven, append-only. They always co-occur: every conversation creates both a time-line entry ("user said something") and a dialogue archive entry ("exact words"). Keeping them separate meant writing two append-only files per turn with overlapping metadata. Merging into Center Axis reduces write operations by 50% and simplifies the query path: "what happened" and "exact words" live under one axis, queryable together or separately by type filter.

## Why Timeline + Session Merge Works

- Both are event-driven → no conflict in persistence strategy
- Both use append-only → no overwrite risk
- Together they form a complete record: "when" (time-index) + "what was said" (conversation)
- Event-driven principle preserved: history is still immutable

## Data Flow

```
User Message
   ↓ Always write
Center Axis (time index + conversation archive)
   ↓ Auto-evaluate
├── State/rule change? → Hermes Axis (user state + rules query)
├── Task progress? → Object Axis (task changelog)
├── Knowledge to extract? → extract-fact
        ↓
  Semantic search (FTS5 / Vector / External Web fallback)
        ↓
  importance ≥ 4? → Wiki Vault
```

## Update Rules

| Axis   | Driving | When | How | Strategy |
|--------|---------|------|-----|----------|
| Center | Event   | Every reply | auto-write | Append-only (time-index + conv archive) |
| Hermes | State   | On state/rule change | auto-update + rule query | Last-write-wins |
| Object | State   | On task progress | Manual or auto | Append changelog |

## One Command

```bash
python3 autofour_v2.py --mode auto-update \
  --channel "channel" \
  --who "agent" \
  --msg "dialog text" \
  --auto-state
```

The command auto-evaluates all three axes independently.
