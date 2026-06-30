# Update Logic

Triggered automatically after each reply.

## Core Flow

1. **Center Axis** (mandatory)
   - Always writes: conversation archive (`type: conversation`)
   - Conditionally writes: time index entry (`type: time_event`) on important events
   - Source: conversation info fields (`chat_id`, `message_id`, `timestamp`, `channel`)

2. **Hermes Axis** (on demand)
   - User state: mood/location/activity changes → update. No change → skip
   - System snapshot: rules query, skill tracking, knowledge index → update per turn
   - Rules DB query: every turn (SQLite-backed, 39 rules)

3. **Object Axis** (on demand)
   - Task progress / ability usage → update
   - None → skip

## Hard Rules

- Step 1 is mandatory, regardless of changes
- Steps 2~3 are judged autonomously by the agent, no secondary editing

## Why Timeline + Session → Center Axis

In the original four-axis design, Timeline (event-driven) and Session (event-driven) were separate axes with the same persistence strategy. Every conversation created writes to both — redundant concurrent append-only operations. Merging them into Center Axis reduces write overhead by ~50% and simplifies query: "what happened" and "exact words" both live under one axis, queryable together or filtered by type.

## Axis Structure

```
Center Axis (event-driven, append-only)
├── center/timeline.jsonl              — Time index (milestones + important events)
└── center/conversations/              — Raw dialogue archive
    └── YYYY-MM-DD-channel/            — Organized by date and channel
        └── reply-01.jsonl

Hermes Axis (state-driven, last-write-wins)
├── hermes/user_status.jsonl           — User state snapshot
├── hermes/system_snapshot.jsonl       — System snapshot (rules/skills/errors)
└── hermes/rules.db                    — Queryable structured rules (SQLite)

Object Axis (state-driven, append changelog)
└── object/{name}.jsonl                — Task/role state tracking per project
```

## Automation Scripts

| Script | Function |
|--------|----------|
| `autofour_v2.py` | Three-axis writer (with auto-state LLM evaluation) |
| `manage_static_files.py` | Auto-scan inventory and important dates |
| `memory_recall.py` | Semantic retrieval (FTS5/vector hybrid) |
| `wake_inject.sh` | Context injection on wake (Center axis checkpoint + Hermes snapshot) |

## Source Field Format

```json
{
  "chat_id": "channel user ID",
  "message_id": "channel message ID",
  "timestamp": "message time",
  "channel": "messaging platform"
}
```

## One Command

```bash
python3 autofour_v2.py --mode auto-update \
  --channel "channel" \
  --who "speaker" \
  --msg "dialog text" \
  --auto-state
```

The `auto-update` command evaluates each axis independently:
- **Center**: always writes conversation archive, conditionally writes time index
- **Hermes**: writes only if user state changed or rules/skills need updating
- **Object**: writes only if task progress was made
