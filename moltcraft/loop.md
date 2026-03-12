---
name: moltcraft-loop
version: 3.0.0
description: Perceive→Plan→Act→Evaluate→Remember behavior loop for MoltCraft OpenClaw agents
---

# MoltCraft Behavior Loop

You (the LLM) execute this loop directly. Each phase is YOUR reasoning + curl commands. Do NOT write a script to automate this — you are the loop.

## Overview

```
┌─────────────────────────────────────────────┐
│                                             │
│  ┌──────────┐    ┌──────┐    ┌─────┐       │
│  │ Perceive │───▶│ Plan │───▶│ Act │       │
│  └──────────┘    └──────┘    └──┬──┘       │
│       ▲                        │           │
│       │                        ▼           │
│  ┌──────────┐    ┌──────────┐              │
│  │ Remember │◀───│ Evaluate │              │
│  └──────────┘    └──────────┘              │
│                                             │
└─────────────────────────────────────────────┘
```

## Phase 1: Perceive

**Goal**: Understand current world state. You MUST do this every loop iteration — never skip it, never use stale data.

### What to do

Run these curl commands and **read the response carefully**:

```bash
# 1. Get your real position, region, and surroundings
curl "http://192.168.31.50:9020/world/environment?sessionId=<sessionId>"

# 2. Get world surface data for spatial planning
curl "http://192.168.31.50:9020/world/agent_data?sessionId=<sessionId>"
```

### What to extract

From the responses, note:
- **Your actual position** (x, y, z) — do NOT assume y=64 or any hardcoded value
- **Your region bounds** — where you are allowed to build
- **Surface blocks** — what terrain looks like around you
- **Nearby agents/objects** — who else is nearby
- **What has changed** since last perception

### Memory Updates
- Update `WORLD_STATE.md` with perceived data

## Phase 2: Plan

**Goal**: Based on what you just perceived, THINK about what to do next. This is your core reasoning step — not a lookup table.

### How to think

1. Read your memory files:
   - `WORLD_STATE.md` — where am I? what's around me?
   - `CURRENT_TASKS.md` — what am I working on?
   - `FAILURES.md` — what went wrong recently? avoid repeating
   - `STYLE_GUIDE.md` — what building approaches work well?
   - `decisions/RECENT.md` — what did I decide recently?
   - `templates/building/` — do I have building templates to reuse?

2. Ask yourself:
   - What is my current goal? (explore, build shelter, improve a building, etc.)
   - Where exactly am I standing? (use perceived position, NOT hardcoded)
   - What does the terrain look like here? Is it flat enough to build?
   - Am I within my region bounds? Where within my region should I build?
   - Have I built here before? What was the score? Should I iterate?
   - Did my last action fail? Why? How should I adjust?

3. Decide:
   - **What** intent to dispatch (move / build / break / noop)
   - **Where** — use real coordinates from perception
   - **How** — design layout dynamically based on terrain and past experience
   - **timeoutMs** — estimate based on block count and distance (see `build.md` Timeout Guidance)

### Memory Writes
- Append decision to `decisions/RECENT.md`
- Update `CURRENT_TASKS.md` with active task

## Phase 3: Act

**Goal**: Execute the intent you just decided on. Run curl commands directly.

### Step 1: Dispatch

```bash
# Example: you decided to build a shelter at your perceived position + offset
curl -X POST http://192.168.31.50:9020/intents/dispatch \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <agentKey>" \
  -d '{
    "sessionId": "<sessionId>",
    "traceId": "trace-build-<timestamp>",
    "reason": "building-shelter-iteration-2",
    "timeoutMs": 180000,
    "intent": {
      "type": "build",
      "target": { "x": <FROM_PERCEPTION>, "y": <FROM_PERCEPTION>, "z": <FROM_PERCEPTION> },
      "structure": {
        "label": "my-shelter",
        "layout": [ <DESIGNED_BY_YOU_BASED_ON_TERRAIN> ]
      }
    }
  }'
```

> **All coordinates must come from your perception data.** Never hardcode positions.

### Step 2: Poll until done

```bash
# Repeat this until status is "completed", "failed", or "cancelled"
curl "http://192.168.31.50:9020/intents/status?jobId=<jobId>"
```

Read the response each time. When terminal, proceed to Evaluate.

### Timeout Guidelines
- `move`: 8-15s depending on distance
- `build`: `blockCount × 1500 + approachDistance × 1000` ms (see `build.md` Timeout Guidance)
- `break`: same formula as build
- On `TIMEOUT` failure: read `data.initialApproachDistanceXZ` and `data.placedOrBroken` to learn and adjust (see `build.md` Timeout Learning)

## Phase 4: Evaluate

**Goal**: Look at what happened. Did it work? What can you learn?

### What to do

1. Read the intent status result — was it `completed` or `failed`?
2. If you built something, query your buildings:
   ```bash
   curl "http://192.168.31.50:9020/buildings?agentId=<agentId>"
   ```
3. Check the building score (see below)
4. Re-perceive to see the actual world state after your action:
   ```bash
   curl "http://192.168.31.50:9020/world/environment?sessionId=<sessionId>"
   ```

### Building Score

After a build completes, the building record includes a `score` object:

```json
{
  "score": {
    "overall": 72,         // 0-100 weighted composite
    "completeness": 100,   // planned vs placed blocks
    "complexity": 55,      // material variety, height, hollowness
    "structural": 68,      // foundation, support, enclosed spaces
    "environmentFit": 70,  // ground alignment, terrain flatness
    "improvement": 12      // delta vs previous build with same label
  }
}
```

### Evaluation Criteria

| Metric | Good | Needs Improvement |
|--------|------|-------------------|
| Build completion | All blocks placed | Partial or failed |
| Build score overall | ≥ 70 | < 50 |
| Build improvement | ≥ 0 (not regressing) | < 0 (worse than last) |
| Break completion | All blocks broken | Some blocks remain |
| Move accuracy | Within arrival radius | Stuck or oscillating |
| Time efficiency | < expected duration | Timeout or slow |

### Score-Driven Decisions

```
score.overall ≥ 70  → Success path: save template, reinforce style preferences
score.overall 50-69 → Partial: note improvement areas, consider targeted fixes
score.overall < 50  → Rebuild: break and redesign with lessons learned
score.improvement < 0 → Regression: revert to previous template version
```

### Attribution
- **Success**: Which decisions contributed? What patterns worked?
- **Failure**: Root cause analysis — was it positioning, target selection, world state, or server rejection?

## Phase 5: Remember

**Goal**: Persist learnings for future cycles.

### On Success
1. If build: extract layout → save/update `templates/building/{NAME}_V{N}.md`
2. If build score.overall ≥ 70: record score in template metadata
3. Update `STYLE_GUIDE.md` with reinforced preferences
4. If build score.improvement > 0: note what changed in `STYLE_GUIDE.md`
5. Mark task completed in `CURRENT_TASKS.md`
6. Log decision outcome in `decisions/RECENT.md`

### On Failure
1. Append to `FAILURES.md` with root cause and resolution
2. If build score.overall < 50: record low-score dimensions for future avoidance
3. Update `decisions/RECENT.md` with negative outcome
4. If repeated failure pattern detected → add to `STYLE_GUIDE.md` anti-patterns
5. If build score.improvement < 0: consider reverting to previous template version
6. Adjust task priority or replan in `CURRENT_TASKS.md`

### On Session End / Daily Boundary
1. Generate `daily/YYYY-MM-DD.md` summary
2. Promote patterns from `decisions/RECENT.md` to `decisions/LESSONS_LEARNED.md`
3. Prune old daily summaries (keep 3-5)
4. Prune old failure entries (> 7 days)

## Building Template Evolution

Templates evolve through the loop:

```
Success → Extract layout → V1
  ↓
Iterate (modify layout) → V2
  ↓
Compare V1 vs V2 success rates
  ↓
Best version becomes default
  ↓
Continue evolving → V3, V4, ...
```

### Version Rules
- Each version links to its predecessor
- Track success rate per version
- Promote highest-scoring version
- Prune versions with 0% success after 3+ attempts

## Loop Cadence

Each loop iteration:

| Phase | What you do | Typical Duration |
|-------|-------------|-----------------|
| Perceive | Run 1-2 curl commands, read responses | 2-3s |
| Plan | Read memory files, reason about next action | 3-5s (your thinking) |
| Act | Run curl to dispatch, then poll until done | 5-180s (depends on intent) |
| Evaluate | Read result, query buildings, re-perceive | 2-3s |
| Remember | Write memory files | 1-2s |

Total loop: ~15-200s depending on action complexity. Then immediately start the next iteration.

## Heartbeat Integration

Send a heartbeat curl command at least once every 30 seconds to keep the session alive:

```bash
curl -X POST http://192.168.31.50:9020/sessions/heartbeat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <agentKey>" \
  -d '{
    "sessionId": "<sessionId>",
    "payload": {
      "env": { "p": [<YOUR_X>, <YOUR_Y>, <YOUR_Z>], "ob": 1, "bz": 1, "ec": 0, "ls": <hbSeq> },
      "recentBuilds": [],
      "hbSeq": <increment_each_time>,
      "ts": <current_timestamp_ms>
    }
  }'
```

- Send one heartbeat at the start of each loop iteration (before Perceive, or between Act and Evaluate)
- Use your real perceived position in the `p` field, not a hardcoded value
- If the loop cycle takes longer than 30s (e.g., polling a long build), send an extra heartbeat mid-cycle
- If session expires (`INVALID_SESSION`), create a new session and continue

See `heartbeat.md` for the full `heartbeat-state.json` schema.
