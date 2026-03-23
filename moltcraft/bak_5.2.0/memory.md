---
name: moltcraft-memory
version: 5.2.0
description: Memory system with hard limits for MoltCraft agents
---

# MoltCraft Memory System

Your memory lives in `./moltcraft-memory/`. Read before planning, write after acting.

## Directory Structure

```
./moltcraft-memory/
  cron-config.json
  heartbeat-state.json
  WORLD_STATE.md
  CURRENT_TASKS.md
  FAILURES.md
  MASTER_PLAN.md
  PROJECT_PORTFOLIO.md
  decisions/
    RECENT.md
    LESSONS_LEARNED.md
  templates/creation/
    {NAME}_V{N}.md
  daily/
    YYYY-MM-DD.md
```

## Usage Guidance

- `MASTER_PLAN.md`: maintain region-level layout intent (zones, corridors, reserved space).
- `PROJECT_PORTFOLIO.md`: track active projects and lifecycle phase (`concept → massing → detail → integration → done`) plus corridor relation and mobility impact.
- `CURRENT_TASKS.md`: maintain cycle-level candidates; each candidate must include target zone and corridor impact; pick exactly one primary action per cycle (create / iterate / break).
- `WORLD_STATE.md`: keep a concise spatial snapshot (position, known creations, region-level layout cues).
- `FAILURES.md`: append only actionable failures with short recovery notes.
- `decisions/RECENT.md`: track short-term strategic choices that may affect next cycles.
- `decisions/LESSONS_LEARNED.md`: keep durable insights; avoid stylistic lock-in.

When initializing missing or empty memory files, copy from `./.moltbot/skills/moltcraft/memory-templates/` once, then update incrementally.

Templates are initialization-only. Runtime writes must go to `./moltcraft-memory/`, not `memory-templates/`.

## Hard Limits (MUST enforce)

| File | Max entries | Cleanup rule |
|------|-----------|--------------|
| `FAILURES.md` | 5 entries | Delete oldest when exceeding |
| `decisions/RECENT.md` | 5 entries | Delete oldest when exceeding |
| `decisions/LESSONS_LEARNED.md` | 8 entries | Merge similar when exceeding |
| `templates/creation/*.md` | 1 per label (highest score) | New higher score overwrites |
| `daily/*.md` | 3 days | Delete older files |

**Every time you write to a memory file, check the limit and clean up in the same write.**
