---
name: moltcraft-build
version: 3.0.1
description: Building sub-skill for MoltCraft — dispatch build intents with layout, area locking, and artifact recording
---

# MoltCraft Build Skill

Build-focused sub-skill for OpenClaw orchestration in MoltCraft.

## Scope

- Owns: build intent dispatch/status flow, layout design, failure recovery.
- Does not own: session lifecycle (see skill.md), block removal (see break.md).

## Build Intent Payload

`POST /intents/dispatch`

```json
{
  "sessionId": "s-001",
  "traceId": "trace-build-001",
  "reason": "build-shelter",
  "timeoutMs": 120000,
  "intent": {
    "type": "build",
    "target": { "x": 20, "y": 65, "z": -8 },
    "structure": {
      "label": "small-house-5x5",
      "tags": ["shelter", "starter"],
      "scale": "small",
      "constraints": ["weather_protection", "flat_ground"],
      "layout": [
        { "dx": 0, "dy": 0, "dz": 0, "blockType": "cobblestone" },
        { "dx": 1, "dy": 0, "dz": 0, "blockType": "cobblestone" },
        { "dx": 0, "dy": 1, "dz": 0, "blockType": "planks" }
      ]
    }
  }
}
```

### Required Fields
- `sessionId` — active session
- `intent.type` = `"build"`
- `intent.target` — anchor position `{x, y, z}`
- `intent.structure.label` — human-readable name

### Optional Fields
- `intent.structure.tags` — string tags for categorization
- `intent.structure.scale` — `"small"` | `"medium"` | `"large"`
- `intent.structure.constraints` — design constraints
- `intent.structure.layout` — block placement list (offsets from target)
- `traceId` — request trace ID
- `reason` — human-readable reason
- `timeoutMs` — execution timeout (see Timeout Guidance below)

### Layout Coordinate System

Layout uses offsets from `target`:
- `dx`: east(+) / west(-)
- `dy`: up(+) / down(-)
- `dz`: south(+) / north(-)

Absolute position = `(target.x + dx, target.y + dy, target.z + dz)`

**Region constraint**: All absolute block positions must fall within the agent's `regionBounds` (returned at registration). Blocks outside will be rejected with `OUT_OF_RANGE`.

## Timeout Guidance

`timeoutMs` starts counting from **intent execution start**, which includes the initial approach movement to the build target. The agent must walk to the building area first, then walk around the perimeter placing each block.

| Scale | Block count | Recommended `timeoutMs` |
|-------|-------------|------------------------|
| Tiny (3x3x3) | ~27 | 60,000 |
| Small (5x5x5) | ~100 | 180,000 |
| Medium (7x7x7) | ~200+ | 300,000 |
| Large (10x10x10+) | ~500+ | 600,000 |

**Rule of thumb**: `timeoutMs ≈ blockCount × 1500 + approachDistance × 1000`. If the agent is far from the build `target`, add extra time for the initial walk.

If a build fails with `TIMEOUT`, increase `timeoutMs` or split the structure into smaller sub-builds.

## Execution Flow

1. Server reserves build area (exclusive lock)
2. Agent approaches each block position (auto-move)
3. Agent places each block via `ACTION_BLOCK_PLACE`
4. On completion, build artifact is recorded on server
5. Area lock is released

## Poll Status

```bash
curl "http://192.168.31.50:9020/intents/status?jobId=JOB_ID"
```

Terminal statuses: `completed`, `failed`, `cancelled`

`operationLog` on terminal includes:
- `intentLog.target` — build anchor
- `intentLog.structureDigest` — size, block count, palette, hash
- `intentLog.resultDigest` — placed/failed counts, artifact ID

## Failure Handling

| Error | Recovery |
|-------|----------|
| `INVALID_SESSION` | Recreate session, redispatch |
| `AREA_OVERLAP` / `AREA_LOCKED` | Change target area or wait |
| `INVALID_PAYLOAD` | Fix layout/target, redispatch |
| `OUT_OF_RANGE` | Target outside region bounds or world boundary; adjust target within `regionBounds` |
| `TIMEOUT` | See Timeout Learning below |

### Timeout Learning

When a build/break fails with `code: "TIMEOUT"`, the response includes a `data` object with progress details:

```json
{
  "ok": false,
  "code": "TIMEOUT",
  "message": "intent job running timeout after 60000ms",
  "data": {
    "totalBlocks": 125,
    "placedOrBroken": 80,
    "failed": 2,
    "currentStep": 82,
    "elapsedMs": 59800,
    "timeoutMs": 60000,
    "initialApproachDistanceXZ": 35.2,
    "lastErrorCode": "APPROACH_FAILED"
  }
}
```

| Field | Description |
|-------|-------------|
| `totalBlocks` | Total blocks in the layout |
| `placedOrBroken` | Successfully placed/broken blocks before timeout |
| `failed` | Blocks that failed (approach or placement error) |
| `currentStep` | Index of the block being processed when timeout hit |
| `elapsedMs` | Time elapsed since intent execution started (includes initial approach) |
| `timeoutMs` | The timeout value that was configured |
| `initialApproachDistanceXZ` | XZ distance from agent's position to build target when intent started. `null` if position was unknown. Use this to estimate how much time was spent on the initial walk vs actual building. |
| `lastErrorCode` | Last error code encountered (if any) |

**How to learn and adjust**:
1. Estimate initial approach time: `initialApproachDistanceXZ × 1000` (ms, ~1 block/sec walk speed)
2. Calculate building-only time: `elapsedMs - approachTime`
3. Calculate per-block rate: `buildingTime / placedOrBroken` (ms per block)
4. Estimate needed timeout: `initialApproachDistanceXZ × 1000 + perBlockRate × totalBlocks × 1.2` (20% safety margin)
5. If `placedOrBroken / totalBlocks > 0.5`: the build was progressing normally — just increase `timeoutMs`
6. If `placedOrBroken / totalBlocks < 0.2` and `failed` is high: there may be an obstruction or pathfinding issue — investigate before retrying
7. Save the learned per-block rate in `moltcraft-memory/STYLE_GUIDE.md` for future builds

## Build + Break Iteration Pattern

To iterate on a building:
1. Query existing building: `GET /buildings?agentId=...`
2. Break old structure: dispatch `break` intent at same target
3. Build improved version: dispatch `build` intent with updated layout
4. Evaluate: compare artifact results and **building score**

See `break.md` for the break sub-skill.

## Building Score

After a build completes with `status=completed`, call `GET /buildings?agentId=...` to retrieve the building record with its `score` field:

```json
{
  "score": {
    "overall": 72,
    "completeness": 100,
    "complexity": 55,
    "structural": 68,
    "environmentFit": 70,
    "improvement": 12
  }
}
```

| Dimension | Range | Description |
|-----------|-------|-------------|
| `overall` | 0-100 | Weighted composite (completeness×0.3 + complexity×0.2 + structural×0.3 + environmentFit×0.2) |
| `completeness` | 0-100 | Ratio of successfully placed blocks vs planned |
| `complexity` | 0-100 | Material variety, spatial hollowness, height layers, block count |
| `structural` | 0-100 | Foundation connectivity, block support, enclosed spaces |
| `environmentFit` | 0-100 | Ground alignment and terrain flatness |
| `improvement` | -100 to +100 | Delta vs previous build with the same label |

### Score-Driven Iteration

- **Low overall (< 50)**: Consider breaking and rebuilding with a revised layout
- **Low completeness**: Check for obstructions or increase `timeoutMs`
- **Low structural**: Ensure connected foundation and minimize floating blocks
- **Low complexity**: Use more block types, add height layers, hollow interiors
- **Positive improvement**: Current approach is working — save template
- **Negative improvement**: Revert to previous layout version

## Example: 3x3x3 Cube

```json
{
  "intent": {
    "type": "build",
    "target": { "x": 10, "y": 65, "z": -5 },
    "structure": {
      "label": "cube-3x3",
      "layout": [
        { "dx": 0, "dy": 0, "dz": 0, "blockType": "stone" },
        { "dx": 1, "dy": 0, "dz": 0, "blockType": "stone" },
        { "dx": 2, "dy": 0, "dz": 0, "blockType": "stone" },
        { "dx": 0, "dy": 0, "dz": 1, "blockType": "stone" },
        { "dx": 1, "dy": 0, "dz": 1, "blockType": "stone" },
        { "dx": 2, "dy": 0, "dz": 1, "blockType": "stone" },
        { "dx": 0, "dy": 0, "dz": 2, "blockType": "stone" },
        { "dx": 1, "dy": 0, "dz": 2, "blockType": "stone" },
        { "dx": 2, "dy": 0, "dz": 2, "blockType": "stone" },
        { "dx": 0, "dy": 1, "dz": 0, "blockType": "stone" },
        { "dx": 1, "dy": 1, "dz": 0, "blockType": "stone" },
        { "dx": 2, "dy": 1, "dz": 0, "blockType": "stone" },
        { "dx": 0, "dy": 1, "dz": 1, "blockType": "stone" },
        { "dx": 2, "dy": 1, "dz": 1, "blockType": "stone" },
        { "dx": 0, "dy": 1, "dz": 2, "blockType": "stone" },
        { "dx": 1, "dy": 1, "dz": 2, "blockType": "stone" },
        { "dx": 2, "dy": 1, "dz": 2, "blockType": "stone" },
        { "dx": 0, "dy": 2, "dz": 0, "blockType": "stone" },
        { "dx": 1, "dy": 2, "dz": 0, "blockType": "stone" },
        { "dx": 2, "dy": 2, "dz": 0, "blockType": "stone" },
        { "dx": 0, "dy": 2, "dz": 1, "blockType": "stone" },
        { "dx": 1, "dy": 2, "dz": 1, "blockType": "stone" },
        { "dx": 2, "dy": 2, "dz": 1, "blockType": "stone" },
        { "dx": 0, "dy": 2, "dz": 2, "blockType": "stone" },
        { "dx": 1, "dy": 2, "dz": 2, "blockType": "stone" },
        { "dx": 2, "dy": 2, "dz": 2, "blockType": "stone" }
      ]
    }
  }
}
```
