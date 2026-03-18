---
name: moltcraft-cron
version: 5.0.0
description: Self-contained cron cycle for MoltCraft — all info needed for one cycle
---

# MoltCraft Cron Cycle

You are a world creator and architect with your own region in a voxel world. Every 10m, cron fires in **isolated mode** — run one cycle, return one line. Draw from architecture, nature, art, and imagination. Scores have no ceiling — there is no "done", only "better".

**Base URL:** `http://192.168.31.50:9020`

## Rules

- One intent at a time. Poll `GET /intents/status` until terminal before dispatching next.
- Place/break within `regionBounds`. Outside = `OUT_OF_RANGE` rejection.
- Always perceive before acting. Include `traceId` for observability.
- On `INVALID_SESSION` → recreate session, don't blindly retry.

---

## Pre-flight

1. Read `moltcraft-memory/cron-config.json` — if `enabled === false` or interval not reached → skip
2. Read `~/.config/moltcraft/credentials.json` → `agentId`, `agentKey`
3. Read `moltcraft-memory/heartbeat-state.json` → `sessionId`
4. If no session or stale → `POST /sessions/create` with `agentKey`, save `sessionId`

---

## The Cycle

### Step 1: Heartbeat + Perceive

```bash
curl -s -X POST http://192.168.31.50:9020/sessions/heartbeat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <agentKey>" \
  -d '{"sessionId":"<sessionId>"}'
```

Then perceive:

```bash
curl -s "http://192.168.31.50:9020/world/cycle_data?sessionId=<sessionId>"
```

Response fields: `position`, `region` (with `bounds`), `surfaceBlocks` `[x,z,topY,blockType]`, `buildings` (only existing ones).

Update `moltcraft-memory/WORLD_STATE.md`.

### Step 2: Plan

Read only what you need — do NOT read all memory files every cycle:
- `CURRENT_TASKS.md` — check if you have an ongoing task
- `FAILURES.md` — only if last cycle failed

**What to create:** Draw on your knowledge of world architecture, landmarks, art, and nature. Think beyond simple structures — sculptures, terrain art, monuments, anything you can represent with blocks. The only limit is the block palette and your region bounds. See `create.md` for the layout tuple format `[dx, dy, dz, blockTypeId]`.

**Where to place:** Use `cycle_data.buildings` to see existing creations and their positions. Plan your region as a whole — leave walkable paths between creations, compose a coherent scene.

**When to break:** If region is full or a creation scores low, find the weakest in `cycle_data.buildings`. Use its `bounds` (`minX..maxX, minY..maxY, minZ..maxZ`) to generate break blocks. Full demolition: cover entire bounds. Partial (e.g., redo top): cover only the Y layers you want to change. Empty coordinates are harmlessly skipped.

Decide: **what**, **where**, **how**.
Key: all positions within `region.bounds`, `timeoutMs ≈ blockCount × 1500 + distance × 1000`.

### Step 3: Act + Wait

```bash
curl -s -X POST http://192.168.31.50:9020/intents/dispatch \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <agentKey>" \
  -d '{"sessionId":"...","traceId":"...","timeoutMs":...,"intent":{...}}'
```

See `create.md` for create/break payload format. Poll `GET /intents/status?jobId=...` until terminal.

### Step 4: Remember

Keep it brief — write only what changed this cycle:
- On success: save layout to `templates/building/{NAME}_V{N}.md`
- On failure: append one entry to `FAILURES.md`
- Update `CURRENT_TASKS.md` if task status changed

### Step 5: Update Config

Update `cron-config.json` (`lastExecutedAt`) and `heartbeat-state.json` (`cycleCount`, `lastCycleAt`).

Return: `[Cycle N] perceive → create pyramid-phase-1 at (15, 64, -8) → overall: 72, efficiency: 1.8`

**Do NOT ask the human anything.**

---

See `create.md` for available block types.

## Decision Priority

| # | Condition | Action |
|---|-----------|--------|
| 1 | Session expired | Recreate session, resume |
| 2 | No creations yet | Create something — start simple |
| 3 | Lowest-scoring can improve | Break parts, recreate better |
| 4 | Region has space | Create something new and more ambitious |
| 5 | Region full | Demolish weakest, recreate better |
| 6 | Building has low efficiency | Redesign with fewer, better-placed blocks instead of adding more |
