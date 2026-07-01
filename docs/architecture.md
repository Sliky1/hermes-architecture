# Architecture Overview

## Core Distinction: Event-Driven vs State-Driven

The three axes are grouped by their persistence strategy, not just by their content type. They run as **Gateway Hooks** (`hooks/instinct-layer/`), executing on every conversation turn.

```
┌─────────────────────────────────────────────────────────────────┐
│                  Three Axes (Runtime Layer)                       │
│                                                                   │
│  ┌──── EVENT-DRIVEN ─────┐  ┌──── STATE-DRIVEN ────────┐        │
│  │                       │  │                           │        │
│  │     Center Axis       │  │      Hermes Axis          │        │
│  │   ├─ instinct_timeline│  │   ├─ instinct_hermes_     │        │
│  │   │  .jsonl           │  │   │  snapshot.jsonl       │        │
│  │   │  (append-only)    │  │   │  (append-only)        │        │
│  │   ├─ Conv via state.db│  │   ├─ rules.db (SQLite)    │        │
│  │   │  (FTS5 searchable)│  │   ├─ instinct_context     │        │
│  │   │                   │  │   │  .json (每轮最新)      │        │
│  │   │                   │  │   └─ skill_feedback       │        │
│  │   │                   │  │      .jsonl (反馈回路)     │        │
│  │   "what happened"     │  │   "current rules/skills"  │        │
│  │   "what tools used"   │  │   "user state"            │        │
│  ├───────────────────────┤  └───────────────────────────┘        │
│  │     Object Axis       │                                       │
│  │   ├─ object_tasks     │                                       │
│  │   │  .jsonl           │                                       │
│  │   │  (task tracking)  │                                       │
│  │   │                   │                                       │
│  │   "project status"    │                                       │
│  └───────────────────────┘                                       │
└──────────────────────────┬────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│   Semantic Retrieval / Knowledge API Layer (Hermes Brain :18700)  │
│                                                                   │
│  ├─ FTS5 + LLM RAG (/qa endpoint)                                │
│  ├─ 331 concepts, 1428 vectors                                   │
│  └─ Web fallback: Tavily / Exa                                    │
│                                                                   │
└──────────────────────────┬────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│   Context Injection (instinct_context.md)                         │
│                                                                   │
│  每轮注入到 Agent 上下文:                                          │
│  ├─ Center: 最近事件、近期工具                                     │
│  ├─ Hermes: 规则分布、技能状态、用户记忆、skill反馈统计               │
│  └─ Object: 活跃任务                                               │
└──────────────────────────────────────────────────────────────────┘
```

## Data Flow (Current Runtime)

```
每轮对话 (via Gateway Hook)
  │
  ├─ agent:start → context_builder.write_context()
  │   ├─ build_center_axis():  读 timeline.jsonl → "最近事件"
  │   ├─ build_hermes_axis():  读 rules.db + events + user_status
  │   │                        → "规则+技能+反馈统计"
  │   └─ build_object_axis():  读 object_tasks.jsonl → "活跃任务"
  │   └─ 写入 instinct_context.json + instinct_context.md
  │
  ├─ agent:step  → rule_gate.query_rules() → 推荐 skills
  │                 event_recorder → 事件缓冲区
  │
  └─ agent:end
      ├─ write_timeline_entry() → timeline.jsonl (Center轴)
      ├─ _append_jsonl(snapshot) → hermes_snapshot.jsonl (Hermes轴)
      ├─ record_skill_feedback() → skill_feedback.jsonl ← 反馈回路
      ├─ 每5轮: checkpoint_manager → checkpoints.jsonl
      ├─ 任务检测 → object_tasks.jsonl (Object轴)
      ├─ flush_events() → instinct_events/today.jsonl
      └─ Knowledge search → Hermes Brain API
```

## Loop Engineering: Skill Feedback Circuit

```
     ┌──────────┐
     │  LLM     │ 读 context.md → 看到 skill_stats
     └────┬─────┘
          │ 根据采纳率调整行为
          ▼
 handler.on_agent_step()
   └─ rule_gate.query_rules() → 推荐 skills
          │
          ▼
 handler.on_agent_end()
   ├─ record_skill_feedback()
   │   (推荐了什么 vs 实际用了什么工具)
   ├─ build_hermes_axis()
   │   └─ _build_skill_stats() → 最近20条采纳率
   └─ 写入 context.md
          │
          └──→ 下一轮注入 → 回到顶部 →
```

## Comparison: Original Design vs Current Runtime

| Dimension | Original Design (repo) | Current Runtime (hooks) |
|-----------|----------------------|------------------------|
| Execution | `autofour_v2.py` script | Gateway Hook, per-turn |
| Conv storage | JSONL append | state.db (FTS5) |
| Hermes snapshot | Independent JSONL | JSONL + in context.json |
| Events | Full history kept | Today-only (rolling) |
| Skill tracking | None | Feedback loop with stats |
| File count | ~18 data files + dirs | 7 data files + 1 dir (~53KB) |
| Code | ~2,500 lines | ~1,900 lines across 7 .py |

## Update Rules

| Axis | Driving | When | Strategy |
|------|---------|------|----------|
| Center | Event | Every reply | Append timeline + state.db |
| Hermes | State | Every reply | Append snapshot + rule query |
| Object | State | On task change | Append changelog + dedup |
