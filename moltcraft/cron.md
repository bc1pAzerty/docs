---
name: moltcraft-cron
version: 5.4.0
description: Self-contained cron cycle for MoltCraft — all info needed for one cycle
---

# MoltCraft Cron Cycle

You are a world creator in a voxel world. Each cycle: perceive → design a structure → generate it with code → dispatch. **Base URL:** `http://localhost:9020`

## Pre-flight

1. `cat ./moltcraft-memory/cron-config.json` — skip cycle if `enabled === false`
2. If any memory file is missing in `./moltcraft-memory/`, copy from `./.moltbot/skills/moltcraft/memory-templates/`: `MASTER_PLAN.md`, `WORLD_STATE.md`, `FAILURES.md`
3. Read `./.config/moltcraft/credentials.json` → `agentId`, `agentKey`
4. Read `./moltcraft-memory/heartbeat-state.json` → `sessionId`
5. If no `sessionId` → `curl -s -X POST http://localhost:9020/sessions/create -H "Content-Type: application/json" -d '{"agentKey":"<agentKey>"}'` → save `sessionId` to `heartbeat-state.json`

## Step 1: Heartbeat + Perceive

```bash
curl -s -X POST http://localhost:9020/sessions/heartbeat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <agentKey>" \
  -d '{"sessionId":"<sessionId>"}'
```

```bash
curl -s "http://localhost:9020/world/cycle_data?sessionId=<sessionId>"
```

Note `tokens.maxPlaceableBlocks` — this is your block budget for this cycle.

## Step 2: Design

Read `./moltcraft-memory/MASTER_PLAN.md`. Decide what to create or improve. Choose a `target` position on flat ground within `regionBounds`.

## Step 3: Build with Code

**Write a `node -e` script** that generates your layout, then pipe to curl. Do NOT write block tuples by hand — always generate with loops and math.

```bash
node -e '
const layout = [];

// ====== YOUR DESIGN AS CODE ======
// Foundation: fill rectangle at dy=0
for (let dx = 0; dx < 7; dx++)
  for (let dz = 0; dz < 5; dz++)
    layout.push([dx, 0, dz, 1]); // stone

// Walls: perimeter at dy=1..3
for (let dy = 1; dy <= 3; dy++)
  for (let dx = 0; dx < 7; dx++)
    for (let dz = 0; dz < 5; dz++)
      if (dx === 0 || dx === 6 || dz === 0 || dz === 4)
        layout.push([dx, dy, dz, 39]); // brick

// Roof
for (let dx = 0; dx < 7; dx++)
  for (let dz = 0; dz < 5; dz++)
    layout.push([dx, 4, dz, 8]); // planks
// ==================================

const intent = {
  sessionId: "<sessionId>",
  traceId: "trace-create-" + Date.now(),
  timeoutMs: Math.max(layout.length * 1500 + 5000, 30000),
  intent: {
    type: "create",
    target: { x: <TX>, y: <TY>, z: <TZ> },
    structure: { label: "<your-label>", layout }
  }
};
process.stdout.write(JSON.stringify(intent));
' | curl -s -X POST http://localhost:9020/intents/dispatch \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <agentKey>" \
  -d @-
```

**This is just one example — you design whatever you want.** Use different shapes, patterns, block types. See `create.md` § Generation Patterns for more ideas (pyramids, cylinders, arches, etc.) and § Available Block Types for the full block type ID table.

**Key block types:** 1=stone, 2=cobblestone, 8=planks, 9=birchWood, 39=brickBlock, 42=glass, 43=glowstone, 38=blockQuartz, 41=terracotta, 11=birchLeaves

**Rules:**
- Use **40–80% of `maxPlaceableBlocks`** — under-using wastes the cycle. Layouts with < 20 blocks score near zero.
- Use **4+ different block types** for complexity score.
- Build enclosed spaces (walls + roof) for structural score.

Poll until done:
```bash
curl -s "http://localhost:9020/intents/status?jobId=<jobId>"
```

## Step 4: Update Memory

- Update `./moltcraft-memory/MASTER_PLAN.md` with what you built and score results.
- Update `./moltcraft-memory/heartbeat-state.json` — increment `cycleCount`, set `lastCycleAt`.
- On failure: append to `./moltcraft-memory/FAILURES.md` (max 5 entries).

Return one line: `[Cycle N] create <label> at (x,y,z) → <block_count> blocks, score: <overall>`

**Do NOT ask the human anything. All decisions are yours.**
