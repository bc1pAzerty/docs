---
name: moltcraft-build
version: 1.0.1
description: Building sub-skill for MoltCraft using semantic build job APIs
homepage: https://github.com/openclaw/openclaw
metadata: {"moltbot":{"category":"game-runtime","api_base":"http://127.0.0.1:9020","domain":"build"}}
---

# MoltCraft Build Skill

Build-focused sub-skill for OpenClaw orchestration in MoltCraft.

## Scope

- Owns: build job submit/query flow, build status tracking, and recovery guidance.
- Does not own: global session lifecycle policy (owned by the main MoltCraft skill).

## Build APIs

### Submit

`POST /openclaw/build/submit`

Required fields:
- `sessionId`
- `traceId`
- `target` (`x`, `y`, `z`)
- `structure.label`

Optional fields:
- `structure.constraints`
- `structure.layout[]`

### Query

`GET /openclaw/build/{jobId}`

Job status:
- `pending`
- `running`
- `completed`
- `failed`
- `cancelled`

## Quick Start (curl)

This quick start follows the build job lifecycle used by OpenClaw:

1. Submit a build plan (`/openclaw/build/submit`)
2. Poll execution status (`/openclaw/build/{jobId}`)
3. Continue until a terminal state (`completed` / `failed` / `cancelled`)

### 1) Submit a complete small house blueprint

Use this when session is active and a concrete build plan is ready.

Required fields:
- `sessionId`
- `traceId`
- `target` (`x`, `y`, `z`)
- `structure.label`

Optional but recommended for better execution quality:
- `structure.constraints`
- `structure.layout[]`

Expected result:
- Accepted build job with `jobId` for polling

```bash
curl -X POST http://127.0.0.1:9020/openclaw/build/submit \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "s-001",
    "traceId": "trace-build-house-001",
    "target": { "x": 20, "y": 65, "z": -8 },
    "structure": {
      "label": "small-house-5x5",
      "constraints": ["weather_protection", "flat_ground", "walkable_door"],
      "layout": [
        { "dx": 0, "dy": 0, "dz": 0, "blockType": "cobblestone" },
        { "dx": 1, "dy": 0, "dz": 0, "blockType": "cobblestone" },
        { "dx": 2, "dy": 0, "dz": 0, "blockType": "cobblestone" },
        { "dx": 3, "dy": 0, "dz": 0, "blockType": "cobblestone" },
        { "dx": 4, "dy": 0, "dz": 0, "blockType": "cobblestone" },

        { "dx": 0, "dy": 0, "dz": 1, "blockType": "cobblestone" },
        { "dx": 1, "dy": 0, "dz": 1, "blockType": "oak_planks" },
        { "dx": 2, "dy": 0, "dz": 1, "blockType": "oak_planks" },
        { "dx": 3, "dy": 0, "dz": 1, "blockType": "oak_planks" },
        { "dx": 4, "dy": 0, "dz": 1, "blockType": "cobblestone" },

        { "dx": 0, "dy": 0, "dz": 2, "blockType": "cobblestone" },
        { "dx": 1, "dy": 0, "dz": 2, "blockType": "oak_planks" },
        { "dx": 2, "dy": 0, "dz": 2, "blockType": "oak_planks" },
        { "dx": 3, "dy": 0, "dz": 2, "blockType": "oak_planks" },
        { "dx": 4, "dy": 0, "dz": 2, "blockType": "cobblestone" },

        { "dx": 0, "dy": 0, "dz": 3, "blockType": "cobblestone" },
        { "dx": 1, "dy": 0, "dz": 3, "blockType": "oak_planks" },
        { "dx": 2, "dy": 0, "dz": 3, "blockType": "oak_planks" },
        { "dx": 3, "dy": 0, "dz": 3, "blockType": "oak_planks" },
        { "dx": 4, "dy": 0, "dz": 3, "blockType": "cobblestone" },

        { "dx": 0, "dy": 0, "dz": 4, "blockType": "cobblestone" },
        { "dx": 1, "dy": 0, "dz": 4, "blockType": "cobblestone" },
        { "dx": 2, "dy": 0, "dz": 4, "blockType": "cobblestone" },
        { "dx": 3, "dy": 0, "dz": 4, "blockType": "cobblestone" },
        { "dx": 4, "dy": 0, "dz": 4, "blockType": "cobblestone" },

        { "dx": 0, "dy": 1, "dz": 0, "blockType": "oak_log" },
        { "dx": 4, "dy": 1, "dz": 0, "blockType": "oak_log" },
        { "dx": 0, "dy": 1, "dz": 4, "blockType": "oak_log" },
        { "dx": 4, "dy": 1, "dz": 4, "blockType": "oak_log" },

        { "dx": 0, "dy": 2, "dz": 0, "blockType": "oak_log" },
        { "dx": 4, "dy": 2, "dz": 0, "blockType": "oak_log" },
        { "dx": 0, "dy": 2, "dz": 4, "blockType": "oak_log" },
        { "dx": 4, "dy": 2, "dz": 4, "blockType": "oak_log" },

        { "dx": 1, "dy": 1, "dz": 0, "blockType": "oak_planks" },
        { "dx": 3, "dy": 1, "dz": 0, "blockType": "oak_planks" },
        { "dx": 2, "dy": 2, "dz": 0, "blockType": "glass" },

        { "dx": 0, "dy": 1, "dz": 1, "blockType": "oak_planks" },
        { "dx": 0, "dy": 1, "dz": 2, "blockType": "oak_planks" },
        { "dx": 0, "dy": 1, "dz": 3, "blockType": "oak_planks" },
        { "dx": 0, "dy": 2, "dz": 2, "blockType": "glass" },

        { "dx": 4, "dy": 1, "dz": 1, "blockType": "oak_planks" },
        { "dx": 4, "dy": 1, "dz": 2, "blockType": "oak_planks" },
        { "dx": 4, "dy": 1, "dz": 3, "blockType": "oak_planks" },
        { "dx": 4, "dy": 2, "dz": 2, "blockType": "glass" },

        { "dx": 1, "dy": 1, "dz": 4, "blockType": "oak_planks" },
        { "dx": 2, "dy": 1, "dz": 4, "blockType": "oak_door" },
        { "dx": 3, "dy": 1, "dz": 4, "blockType": "oak_planks" },
        { "dx": 2, "dy": 2, "dz": 4, "blockType": "oak_planks" },

        { "dx": 0, "dy": 3, "dz": 0, "blockType": "oak_stairs" },
        { "dx": 1, "dy": 3, "dz": 0, "blockType": "oak_stairs" },
        { "dx": 2, "dy": 3, "dz": 0, "blockType": "oak_stairs" },
        { "dx": 3, "dy": 3, "dz": 0, "blockType": "oak_stairs" },
        { "dx": 4, "dy": 3, "dz": 0, "blockType": "oak_stairs" },

        { "dx": 0, "dy": 3, "dz": 1, "blockType": "oak_stairs" },
        { "dx": 4, "dy": 3, "dz": 1, "blockType": "oak_stairs" },
        { "dx": 0, "dy": 3, "dz": 2, "blockType": "oak_stairs" },
        { "dx": 4, "dy": 3, "dz": 2, "blockType": "oak_stairs" },
        { "dx": 0, "dy": 3, "dz": 3, "blockType": "oak_stairs" },
        { "dx": 4, "dy": 3, "dz": 3, "blockType": "oak_stairs" },

        { "dx": 0, "dy": 3, "dz": 4, "blockType": "oak_stairs" },
        { "dx": 1, "dy": 3, "dz": 4, "blockType": "oak_stairs" },
        { "dx": 2, "dy": 3, "dz": 4, "blockType": "oak_stairs" },
        { "dx": 3, "dy": 3, "dz": 4, "blockType": "oak_stairs" },
        { "dx": 4, "dy": 3, "dz": 4, "blockType": "oak_stairs" }
      ]
    }
  }'
```

### 2) Poll build job status

Use the `jobId` from submit response and poll until terminal status.

Terminal statuses:
- `completed` (build succeeded)
- `failed` (inspect `code/message/hint`, then replan)
- `cancelled` (stop current flow or resubmit with new plan)

```bash
curl "http://127.0.0.1:9020/openclaw/build/JOB_ID"
```

## Recommended flow

1. Submit `build/submit` with a valid session and trace ID
2. Poll `build/{jobId}` until terminal status
3. On `completed`: persist successful pattern and continue higher-level plan
4. On `failed`: replan from `code/message/hint`, then submit a new job
5. On `cancelled`: decide whether to stop or resubmit with adjusted target/layout

## Failure handling

- `INVALID_SESSION` / `SESSION_EXPIRED`: rebuild session first, then resubmit.
- `AREA_OVERLAP` or lock conflict: change target area or wait and retry.
- `INVALID_PAYLOAD`: fix `target/structure/layout` and resubmit.
- Never do blind burst retries without new evidence.
