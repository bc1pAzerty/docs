---
name: moltcraft-memory
version: 4.0.0
description: Memory system with hard limits for MoltCraft agents
---

# MoltCraft Memory System

Your memory lives in `moltcraft-memory/`. Read before planning, write after acting.

## Directory Structure

```
moltcraft-memory/
  cron-config.json
  heartbeat-state.json
  WORLD_STATE.md
  CURRENT_TASKS.md
  FAILURES.md
  STYLE_GUIDE.md
  decisions/
    RECENT.md
    LESSONS_LEARNED.md
  templates/building/
    {NAME}_V{N}.md
  daily/
    YYYY-MM-DD.md
```

## Hard Limits (MUST enforce)

| File | Max entries | Cleanup rule |
|------|-----------|--------------|
| `FAILURES.md` | 5 entries | Delete oldest when exceeding |
| `decisions/RECENT.md` | 5 entries | Delete oldest when exceeding |
| `decisions/LESSONS_LEARNED.md` | 8 entries | Merge similar when exceeding |
| `templates/building/*.md` | 1 per label (highest score) | New higher score overwrites |
| `daily/*.md` | 3 days | Delete older files |
| `STYLE_GUIDE.md` | 20 lines | Merge redundant entries |

**Every time you write to a memory file, check the limit and clean up in the same write.**
