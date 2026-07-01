# Update Logic (Current Runtime)

The three-axis system runs as a **Gateway Hook** (`hooks/instinct-layer/`), executing every conversation turn. No external script needed.

## Core Flow (per turn)

1. **Center Axis** (mandatory)
   - Writes timeline entry: `instinct_timeline.jsonl` (`type: time_event`)
   - Conversation archive: stored in `state.db` (Hermes kernel), not duplicated
   - Source: conversation info (`session_key`, `timestamp`, `platform`)

2. **Hermes Axis** (every turn)
   - Rule query: `rules.db` (SQLite-backed, 39 rules, skill_map)
   - Snapshot: appends `instinct_hermes_snapshot.jsonl` (rules + skills + errors + user_status)
   - Skill feedback: appends `instinct_skill_feedback.jsonl` (recommended vs used)
   - Context: writes `instinct_context.json` + `instinct_context.md`

3. **Object Axis** (on demand)
   - Reads `object_tasks.jsonl` (single source, dedup)
   - Empty file = skip (added 2026-07-01)
   - Task detection writes to `instinct_tasks.jsonl`

## Axis File Structure

```
Instinct Layer (~53KB total)
├── instinct_context.json          — Three-axis summary (every turn)
├── instinct_context.md            — Derived injection text (every turn)
├── instinct_timeline.jsonl        — Center axis: time index
├── instinct_hermes_snapshot.jsonl — Hermes axis: history (restored 2026-07-01)
├── instinct_skill_feedback.jsonl  — Feedback loop: skill adoption stats
├── instinct_checkpoints.jsonl     — Every 5 turns: checkpoint summary
├── instinct_user_status.jsonl     — Last platform / turn count (external)
├── instinct_user_memories.md      — Manual user memories (edited by user)
└── instinct_events/
    └── events_YYYYMMDD.jsonl      — Today's debug events only
```

## Runtime Files (7 Python files)

| File | Function | Lines |
|------|----------|-------|
| `handler.py` | Event lifecycle hook | ~437 |
| `context_builder.py` | Three-axis summary builder | ~635 |
| `event_recorder.py` | Event + skill feedback recording | ~300 |
| `checkpoint_manager.py` | Checkpoint creation/persistence | ~232 |
| `behavior_validator.py` | Turn behavior validation | ~150 |
| `rule_gate.py` | Rules DB query + skill_map | ~200 |
| `knowledge_api_client.py` | Hermes Brain API client | ~80 |

## Ponytail Optimization History

### 2026-07-01 — Cleanup Round 1
- Deleted: `instinct_conversations/` (24KB, state.db duplicate)
- Deleted: `instinct_backup/` (60KB, old code backup)
- Deleted: 8 baseline/old-format files
- Deleted: Old events (June 28-30, 440KB → today only)
- Removed: `archive_conversation_turn()` dead function
- Removed: Dual source reading in object_axis (same file twice)
- Added: Empty-file early exit in object_axis
- Fixed: context.json + context.md now sync every turn
- Fixed: Events reading = binary seek last line (was 100-line scan)
- **Result: 18+ instinct files → 7 files + 1 dir, ~780KB → ~53KB**

### 2026-07-01 — Cleanup Round 2 (Feedback Loop)
- Added: `record_skill_feedback()` — records recommended vs used tools
- Added: `_build_skill_stats()` — reads last 20 feedback, computes adoption
- Added: `skill_stats` in context.json → visible in my context injection
- Restored: `hermes_snapshot.jsonl` append (Ponytail over-reached, history needed)
- **Result: Full loop engineering — collect → analyze → inject → adjust**

## Key Decisions

| Decision | Rationale |
|----------|-----------|
| Conv in state.db, not JSONL | state.db has FTS5, ACID, session_search tool |
| hermes_snapshot restored | History backtracking for rules/skills state |
| skill_feedback separate file | Clean append-only data for LLM-driven analysis |
| Events: today only | Debug events not useful after current day |
| context.md per turn | Was checkpoint-only, led to stale injection |
