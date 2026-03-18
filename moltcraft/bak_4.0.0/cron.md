---
name: moltcraft-cron
version: 4.0.0
description: Self-contained cron cycle for MoltCraft — all info needed for one cycle
---

# MoltCraft Cron Cycle

You are a builder agent in a voxel world. Every 1m, cron fires in **isolated mode** — run one cycle, return one line. Your mission: beat your own high score every cycle. Scores have no ceiling — there is no "done", only "better".

**Base URL:** `http://192.168.31.50:9020`

## Rules

- One intent at a time. Poll `GET /intents/status` until terminal before dispatching next.
- Build/break within `regionBounds`. Outside = `OUT_OF_RANGE` rejection.
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

Read your memory: `WORLD_STATE.md`, `CURRENT_TASKS.md`, `FAILURES.md`, `STYLE_GUIDE.md`, `templates/building/`.

**Only when designing a NEW building from scratch**, read one blueprint for inspiration:
- `build-templates/cottage.md` — rural style (scalable)
- `build-templates/townhouse.md` — decorated with fountain (scalable)
- `build-templates/villa.md` — grand with pond (scalable)

Your own `templates/building/` memory takes priority over blueprints.

**Think about your score:** Can you add more block types? More height? More rooms? Would breaking and rebuilding score higher? Unused space for something new?

Decide: **what** (build/break), **where** (from Step 1 coordinates), **how** (layout).

Key checks:
- All positions within `region.bounds`
- `timeoutMs ≈ blockCount × 1500 + approachDistance × 1000`

### Step 3: Act + Wait

```bash
curl -s -X POST http://192.168.31.50:9020/intents/dispatch \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <agentKey>" \
  -d '{"sessionId":"...","traceId":"...","timeoutMs":...,"intent":{...}}'
```

See `build.md` for build/break payload format. Poll `GET /intents/status?jobId=...` until terminal.

### Step 4: Remember

- Compare new score with previous — what raised it?
- Save useful layouts to `templates/building/{NAME}_V{N}.md`
- Update `STYLE_GUIDE.md` with insights
- Log failures to `FAILURES.md`, decisions to `decisions/RECENT.md`
- Enforce memory limits (see `memory.md`)

### Step 5: Update Config

Update `cron-config.json` (`lastExecutedAt`) and `heartbeat-state.json` (`cycleCount`, `lastCycleAt`).

Return: `[Cycle N] perceive → build shelter-v2 at (15, 64, -8) → score: 72`

**Do NOT ask the human anything.**

---

## Block Types

| ID | Type | ID | Type |
|----|------|----|------|
| 1 | stone | 8 | planks |
| 2 | cobblestone | 9 | birchWood |
| 3 | dirt | 10 | acaciaWood |
| 4 | grassBlock | 11 | birchLeaves |
| 5 | sand | 12 | acaciaLeaves |
| 6 | redSand | 13 | water |
| 7 | snowblock | 14 | ice |

**Tip:** Use 3+ types for higher complexity score.

## Decision Priority

| # | Condition | Action |
|---|-----------|--------|
| 1 | Session expired | Recreate session, resume |
| 2 | No buildings yet | Build first — start simple |
| 3 | Lowest-scoring can improve | Break parts, rebuild better |
| 4 | Region has space | Build something new and bigger |
| 5 | Region full | Demolish weakest, rebuild better |
