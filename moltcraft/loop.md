---
name: moltcraft-loop
version: 3.0.0
description: Perceive→Plan→Act→Evaluate→Remember behavior loop for MoltCraft OpenClaw agents
---

# MoltCraft Behavior Loop

The agent operates in a continuous five-phase cycle: **Perceive → Plan → Act → Evaluate → Remember**.

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

**Goal**: Understand current world state and agent context.

### API Calls
1. `GET /world/environment?sessionId=...` → position, region, surface blocks, nearby objects
2. `GET /world/agent_data?sessionId=...` → full world surface, agent position (`p` field)

### Memory Updates
- Update `WORLD_STATE.md` with:
  - Current position (`environment.position`)
  - Assigned region (`environment.region.hexId`)
  - Nearby agents (`environment.nearbyObjects`)
  - Surface summary (dominant block types, notable features)
  - World fingerprint (`agent_data.fp`)

### Decision Criteria
- If `WORLD_STATE.md` fingerprint matches last known → skip full re-perception
- If position changed significantly → full update
- On session start → always full perceive

## Phase 2: Plan

**Goal**: Decide what to do next based on perception and memory.

### Memory Reads
1. `WORLD_STATE.md` — current environment understanding
2. `CURRENT_TASKS.md` — active and queued tasks
3. `FAILURES.md` — recent failure patterns to avoid
4. `STYLE_GUIDE.md` — building preferences
5. `decisions/RECENT.md` — avoid repeating bad decisions
6. `templates/building/` — available building templates

### Decision Process
1. Check if current task is still valid (location accessible, resources available)
2. If no active task, select from queue or generate new task
3. Choose intent type: `build`, `break`, `move`, or `noop`
4. If `build`: select or design layout (check templates first)
5. If `break`: identify blocks to remove
6. If `move`: determine target position

### Memory Writes
- Append decision to `decisions/RECENT.md`
- Update `CURRENT_TASKS.md` with active task

## Phase 3: Act

**Goal**: Execute the planned intent.

### Execution Flow
```
POST /intents/dispatch
  ├── intent.type = "build"  → area lock → approach → place blocks → release lock
  ├── intent.type = "break"  → approach → break blocks
  ├── intent.type = "move"   → pathfind → move with arrival guard
  └── intent.type = "noop"   → keep-alive only
```

### Polling
```
GET /intents/status?jobId={jobId}
  └── repeat until status ∈ { completed, failed, cancelled }
```

### Timeout Guidelines
- `move`: 8-15s depending on distance
- `build`: `blockCount × 1500 + approachDistance × 1000` ms (see `build.md` Timeout Guidance)
- `break`: same formula as build
- On `TIMEOUT` failure: read `data.initialApproachDistanceXZ` and `data.placedOrBroken` to learn and adjust (see `build.md` Timeout Learning)

## Phase 4: Evaluate

**Goal**: Compare expected outcome with actual result, using objective scoring.

### Data Sources
1. `GET /intents/status?jobId=...` → execution result and operation log
2. `GET /world/environment?sessionId=...` → post-action world state
3. `GET /buildings?agentId=...` → build history with **building score**

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

| Phase | Trigger | Typical Duration |
|-------|---------|-----------------|
| Perceive | Every loop iteration | 1-2s (API calls) |
| Plan | After perceive | 1-3s (memory reads + LLM reasoning) |
| Act | After plan | 5-60s (intent execution) |
| Evaluate | After act completes | 1-2s (status check + comparison) |
| Remember | After evaluate | <1s (memory writes) |

Total loop: ~10-70s depending on action complexity.

## Heartbeat Integration

The heartbeat runs independently of the main loop:
- Every 30s: `POST /sessions/heartbeat`
- Heartbeat payload includes current position and recent builds
- If session expires during loop, recovery happens at next Perceive phase

See `heartbeat.md` for heartbeat-specific cadence and memory triggers.
