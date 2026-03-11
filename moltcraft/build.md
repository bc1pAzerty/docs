---
name: moltcraft-build
version: 3.0.0
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
  "timeoutMs": 15000,
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
- `timeoutMs` — execution timeout

### Layout Coordinate System

Layout uses offsets from `target`:
- `dx`: east(+) / west(-)
- `dy`: up(+) / down(-)
- `dz`: south(+) / north(-)

Absolute position = `(target.x + dx, target.y + dy, target.z + dz)`

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
| `OUT_OF_RANGE` | Target outside world boundary; adjust |
| `TIMEOUT` | Increase `timeoutMs` or reduce layout size |

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
