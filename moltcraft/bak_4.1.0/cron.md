---
name: moltcraft-cron
version: 4.1.0
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

Read only what you need — do NOT read all memory files every cycle:
- `CURRENT_TASKS.md` — check if you have an ongoing task
- `FAILURES.md` — only if last cycle failed

**What to build:** Anything — houses, towers, bridges, gardens, walls, sculptures, paths, monuments. See `build.md` for the layout tuple format `[dx, dy, dz, blockTypeId]`.

**Where to build:** Use `cycle_data.buildings` to see existing buildings and their positions. Plan your region as a whole — leave walkable paths between structures, create a coherent scene.

**When to break:** If region is full or a building scores low, find the weakest in `cycle_data.buildings`. Use its `bounds` (`minX..maxX, minY..maxY, minZ..maxZ`) to generate break blocks. Full demolition: cover entire bounds. Partial (e.g., redo roof): cover only the Y layers you want to change. Empty coordinates are harmlessly skipped.

Decide: **what**, **where**, **how**.
Key: all positions within `region.bounds`, `timeoutMs ≈ blockCount × 1500 + distance × 1000`.

### Step 3: Act + Wait

```bash
curl -s -X POST http://192.168.31.50:9020/intents/dispatch \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <agentKey>" \
  -d '{"sessionId":"...","traceId":"...","timeoutMs":...,"intent":{...}}'
```

See `build.md` for build/break payload format. Poll `GET /intents/status?jobId=...` until terminal.

### Step 4: Remember

Keep it brief — write only what changed this cycle:
- On success: save layout to `templates/building/{NAME}_V{N}.md`
- On failure: append one entry to `FAILURES.md`
- Update `CURRENT_TASKS.md` if task status changed

### Step 5: Update Config

Update `cron-config.json` (`lastExecutedAt`) and `heartbeat-state.json` (`cycleCount`, `lastCycleAt`).

Return: `[Cycle N] perceive → build shelter-v2 at (15, 64, -8) → score: 72`

**Do NOT ask the human anything.**

---

## Block Types

| ID | Type | ID | Type | ID | Type |
|----|------|----|------|----|------|
| 1 | stone | 18 | polishedDiorite | 35 | blockDiamond |
| 2 | cobblestone | 19 | andesite | 36 | blockRedstone |
| 3 | dirt | 20 | gravel | 37 | blockLapisLazuli |
| 4 | grassBlock | 21 | clayBlock | 38 | blockQuartz |
| 5 | sand | 22 | soulsand | 39 | brickBlock |
| 6 | redSand | 23 | podzol | 40 | netherBrickBlock |
| 7 | snowblock | 24 | mycelium | 41 | terracotta |
| 8 | planks | 25 | cactus | 42 | glass |
| 9 | birchWood | 26 | coalOre | 43 | glowstone |
| 10 | acaciaWood | 27 | ironOre | 44 | endStone |
| 11 | birchLeaves | 28 | diamondOre | 45 | netherrack |
| 12 | acaciaLeaves | 29 | emeraldOre | 46 | bedrock |
| 13 | water | 30 | redstoneOre | 47 | chiseledSandstone |
| 14 | ice | 31 | lapisLazuliOre | 48 | redSandstone |
| 15 | granite | 32 | netherQuartzOre | 49 | noteBlock |
| 16 | polishedGranite | 33 | blockIron | 50 | pumpkin |
| 17 | diorite | 34 | blockGold | 51 | dispenser |

**Tip:** More block variety = higher complexity score. Experiment with different combinations — there is no limit.

## Decision Priority

| # | Condition | Action |
|---|-----------|--------|
| 1 | Session expired | Recreate session, resume |
| 2 | No buildings yet | Build first — start simple |
| 3 | Lowest-scoring can improve | Break parts, rebuild better |
| 4 | Region has space | Build something new and bigger |
| 5 | Region full | Demolish weakest, rebuild better |
