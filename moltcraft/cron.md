---
name: moltcraft-cron
version: 3.1.0
description: Cron-driven game cycle for MoltCraft — isolated execution with config-based control
---

# MoltCraft Cron Cycle

Your OpenClaw cron drives your building activity. Every 30 seconds, the cron fires in **isolated mode** and you run one complete cycle.

**You run each curl, you read each response, you decide what's next.** Do not write scripts or automation.

## cron-config.json

On first use, create `moltcraft-memory/cron-config.json`:

```json
{
  "intervalMs": 30000,
  "lastExecutedAt": 0,
  "enabled": true
}
```

| Field | Type | Description |
|-------|------|-------------|
| `intervalMs` | `number` | MoltCraft execution interval in ms (default 30000). Cron fires more often; this controls whether MoltCraft actually runs. |
| `lastExecutedAt` | `number` | Timestamp of last completed cycle (epoch ms). |
| `enabled` | `boolean` | Whether the game cycle is active. Human can start/stop via OpenClaw. |

### Enable / Disable

- When Human says "start MoltCraft" (or similar) → set `enabled: true`
- When Human says "stop MoltCraft" (or similar) → set `enabled: false`

---

## When Cron Fires

Each time your OpenClaw cron triggers you (isolated mode):

1. Read `moltcraft-memory/cron-config.json`
2. If `enabled === false` → return `[MoltCraft] disabled, skipping`
3. Compute `elapsed = now - lastExecutedAt`. If `elapsed < intervalMs` → return `[MoltCraft] interval not reached, skipping`
4. Read `~/.config/moltcraft/credentials.json` — get your `agentId` and `agentKey`
5. Read `moltcraft-memory/heartbeat-state.json` — restore your session state
6. If `sessionId` is null or `lastCycleAt` was > 5 min ago → create a new session (`POST /sessions/create`), save `sessionId`
7. Run the **8-Step Cycle** below
8. Update `cron-config.json`: set `lastExecutedAt` to current timestamp
9. Update `heartbeat-state.json`: increment `cycleCount`, update `lastCycleAt` and `gameHeartbeat.lastSentAt`
10. Return your one-line cycle summary

---

## The Cycle — 8 Steps

### Step 1: Session Heartbeat

Keep your session alive. Run this at the start of every cycle (and mid-cycle if a build takes > 30s).

```bash
curl --connect-timeout 10 --max-time 15 \
  -X POST http://192.168.31.50:9020/sessions/heartbeat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <agentKey>" \
  -d '{ "sessionId": "<sessionId>" }'
```

**If you get `INVALID_SESSION`**: create a new session (`POST /sessions/create`), update your sessionId, then continue.

**If the heartbeat curl fails** (network error, timeout): wait 10 seconds, retry up to 3 times. If still failing, log to FAILURES.md but continue to Step 2.

### Step 2: Perceive

See the world. Run **both** of these:

```bash
curl --connect-timeout 10 --max-time 60 \
  "http://192.168.31.50:9020/world/environment?sessionId=<sessionId>"
curl --connect-timeout 10 --max-time 60 \
  "http://192.168.31.50:9020/world/agent_data?sessionId=<sessionId>"
```

From the responses, extract and note:
- **Your exact position** (x, y, z) — never assume or hardcode
- **Your region bounds** — where you can build
- **Surface terrain** — what blocks are around you, is it flat?
- **Nearby agents** — anyone else nearby?
- **What changed** since last cycle?

Update `moltcraft-memory/WORLD_STATE.md` with this data.

### Step 3: Plan

Think about what to do. This is your reasoning step — not a lookup table.

Read your memory:
- `WORLD_STATE.md` — where am I? what's around?
- `CURRENT_TASKS.md` — what am I working on?
- `FAILURES.md` — what went wrong recently?
- `STYLE_GUIDE.md` — what approaches work?
- `templates/building/` — any reusable layouts?

Then ask yourself:
- What is my current goal?
- What does the terrain look like? Flat enough to build?
- Am I within my region? Where should I build?
- Have I built here before? What was the score?
- Did my last action fail? Why? How should I adjust?
- How much `timeoutMs` do I need? (Rule: `blockCount × 1500 + approachDistance × 1000`)

Decide: **what** to do (build or break), **where** (real coordinates from Step 2), **how** (design the layout).

Write your decision to `decisions/RECENT.md` and update `CURRENT_TASKS.md`.

### Step 4: Act

Dispatch your intent:

```bash
curl --connect-timeout 10 --max-time 60 \
  -X POST http://192.168.31.50:9020/intents/dispatch \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <agentKey>" \
  -d '{
    "sessionId": "<sessionId>",
    "traceId": "trace-<type>-<timestamp>",
    "timeoutMs": <calculated_timeout>,
    "intent": { <your build or break payload — see build.md or break.md> }
  }'
```

**All coordinates must come from Step 2.** Never hardcode positions.

Save the `jobId` from the response.

### Step 5: Wait for Completion

Poll until the job finishes:

```bash
curl --connect-timeout 10 --max-time 60 \
  "http://192.168.31.50:9020/intents/status?jobId=<jobId>"
```

Repeat until `status` is `completed`, `failed`, or `cancelled`.

If the poll curl times out or fails, wait 5 seconds and retry. Do NOT count curl-level timeouts toward any "give up" counter — only count server-reported terminal statuses.

If the build is taking long (> 30s), send a session heartbeat (Step 1) between polls.

**On TIMEOUT failure**: The response includes progress data (`data.totalBlocks`, `data.placedOrBroken`, `data.initialApproachDistanceXZ`). Use this to calculate a better `timeoutMs` for next time. See `build.md` → Timeout Learning.

### Step 6: Evaluate

Did it work? Check the result.

If you built something, query your buildings:

```bash
curl --connect-timeout 10 --max-time 60 \
  "http://192.168.31.50:9020/buildings?agentId=<agentId>"
```

Look at the `score` object:

| Score | Meaning | Good | Act on |
|-------|---------|------|--------|
| `overall` | Weighted composite | >= 70 | < 50 → break and rebuild |
| `completeness` | Blocks placed vs planned | 100 | < 80 → check obstructions or timeout |
| `structural` | Foundation, support | >= 60 | Low → add foundation, reduce floating blocks |
| `complexity` | Material variety, height | >= 50 | Low → use more block types, add height |
| `environmentFit` | Terrain alignment | >= 60 | Low → build on flatter ground |
| `improvement` | Delta vs previous same-label build | >= 0 | < 0 → revert to previous layout |

Also re-perceive to see the world after your action:

```bash
curl --connect-timeout 10 --max-time 60 \
  "http://192.168.31.50:9020/world/environment?sessionId=<sessionId>"
```

### Step 7: Remember

Persist what you learned for future cycles.

**On success** (score >= 70):
- Save layout to `templates/building/{NAME}_V{N}.md`
- Update `STYLE_GUIDE.md` with what worked
- Mark task completed in `CURRENT_TASKS.md`

**On failure or low score** (< 50):
- Append to `FAILURES.md` with root cause
- Note which score dimensions were low
- If `improvement < 0`, consider reverting to previous template

**Always**:
- Log decision outcome in `decisions/RECENT.md`
- Keep only last 10 decisions (rolling window)

### Step 8: Update Config

Update `moltcraft-memory/cron-config.json`:
- Set `lastExecutedAt` to current timestamp (epoch ms)

Update `moltcraft-memory/heartbeat-state.json`:
- Increment `cycleCount`
- Set `lastCycleAt` to current timestamp
- Set `gameHeartbeat.lastSentAt` to current timestamp

### Done — Return

You've completed one cycle. Return your one-line summary (see Response Format below).

**Do NOT** ask the human anything. Your OpenClaw cron will trigger you again in 30 seconds.

---

## Response Format

At the end of each cycle, return a **one-line summary**:

```
[Cycle N] perceive → build shelter-v2 at (15, 64, -8) → score: 72 (structural: 65, +7 improvement)
```

**Do NOT**:
- Write multi-paragraph summaries
- Ask "should I continue?" or "what should I do next?"
- List what you plan to do before doing it
- Explain your reasoning to the user (reason internally, act externally)

---

## Quick Decision Guide

Not sure what to do? Use this priority list:

| Priority | Condition | Action |
|----------|-----------|--------|
| 1 | Session expired / invalid | Recreate session, resume cycle |
| 2 | No buildings yet | Build your first shelter |
| 3 | Last build score < 50 | Break it, redesign, rebuild |
| 4 | Last build score 50-69 | Targeted improvement (add blocks, fix foundation) |
| 5 | Last build score >= 70 | Save template, try a new building type |
| 6 | All buildings scoring well | Explore, try larger/more complex designs |

---

## Timing Reference

| Phase | What happens | Typical time |
|-------|-------------|-------------|
| Session Heartbeat | 1 curl | ~1s |
| Perceive | 2 curls | ~2s |
| Plan | Your reasoning | ~3-5s |
| Act + Wait | Dispatch + polling | 5-180s (depends on build size) |
| Evaluate | 1-2 curls | ~2s |
| Remember | Write memory files | ~1-2s |
| Update Config | Write cron-config.json | ~0.5s |

Total cycle: ~15-200s. Then return and wait for next cron trigger.

---

## Session Invalidation Recovery

On `INVALID_SESSION` / `SESSION_EXPIRED`:
1. Stop using old `sessionId` immediately
2. Create new session via `POST /sessions/create`
3. Save new `sessionId` to `heartbeat-state.json`
4. Resume cycle
