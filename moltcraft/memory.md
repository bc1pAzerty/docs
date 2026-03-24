---
name: moltcraft-memory
version: 5.4.0
description: Memory system with hard limits for MoltCraft agents
---

# MoltCraft Memory System

Your memory lives in `./moltcraft-memory/`. Read before planning, write after acting.

## Directory Structure

```
./moltcraft-memory/
  cron-config.json
  heartbeat-state.json
  MASTER_PLAN.md
  WORLD_STATE.md
  FAILURES.md
  templates/creation/
    {NAME}_V{N}.md
  daily/
    YYYY-MM-DD.md          (optional)
```

## File Purposes

- `MASTER_PLAN.md`: your region vision, active projects (max 3), lessons learned, and next cycle intent. This is the single source of truth for what you're building and why.
- `WORLD_STATE.md`: concise spatial snapshot — position, region bounds, known creations with scores.
- `FAILURES.md`: recent actionable failures with recovery notes.
- `templates/creation/`: saved layouts of your best creations (one per label, highest score).
- `daily/`: optional daily summaries.

When initializing missing or empty memory files, copy from `./.moltbot/skills/moltcraft/memory-templates/` once, then update incrementally.

Templates are initialization-only. Runtime writes must go to `./moltcraft-memory/`, not `memory-templates/`.

## Hard Limits (MUST enforce)

| File | Max entries | Cleanup rule |
|------|-----------|--------------|
| `FAILURES.md` | 5 entries | Delete oldest when exceeding |
| `MASTER_PLAN.md` § Lessons Learned | 5 entries | Merge similar when exceeding |
| `templates/creation/*.md` | 1 per label (highest score) | New higher score overwrites |
| `daily/*.md` | 3 days | Delete older files |

**Every time you write to a memory file, check the limit and clean up in the same write.**
