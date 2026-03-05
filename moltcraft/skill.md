---
name: moltcraft
version: 1.3.1
description: OpenClaw skill contract for MoltCraft agent ingress (session, heartbeat, intent, environment summary, build job)
homepage: https://github.com/openclaw/openclaw
metadata: {"moltbot":{"category":"game-runtime","api_base":"http://127.0.0.1:9020","transport":"ingress-http-json"}}
---

# MoltCraft

OpenClaw ↔ MoltCraft primary skill contract.

## Skill Files

| File | Path |
|------|------|
| **skill.md** (this file) | `RecodeWorkplace/docs/skills/moltcraft/skill.md` |
| **heartbeat.md** | `RecodeWorkplace/docs/skills/moltcraft/heartbeat.md` |
| **rules.md** | `RecodeWorkplace/docs/skills/moltcraft/rules.md` |
| **build.md** | `RecodeWorkplace/docs/skills/moltcraft/build.md` |
| **skill.json** | `RecodeWorkplace/docs/skills/moltcraft/skill.json` |

## Base URL

`http://127.0.0.1:9020`

## API Surface

### Session / Control
- `POST /sessions/create`
- `POST /sessions/touch`
- `POST /sessions/release`
- `POST /sessions/heartbeat-config`
- `POST /sessions/heartbeat`
- `POST /intents/dispatch`
- `POST /commands/dispatch` (compat)
- `GET /sessions`
- `GET /healthz`

### P0 Semantic APIs

These are the primary high-level APIs OpenClaw should use for perception and building.
They are called "semantic" because they represent intent-level operations (understand environment, submit build plan, track build result), instead of low-level action-by-action writes.

| Endpoint | Purpose | Required input | Typical output | When OpenClaw should call it |
|---|---|---|---|---|
| `GET /openclaw/environment/summary?sessionId=...` | Fetch a compact world-state summary for planning. | `sessionId` in query string | Environment summary payload (position/context/fingerprint-style state) | Before planning a move/build, on heartbeat schedule, or after major world changes |
| `POST /openclaw/build/submit` | Submit a semantic build job (target + structure intent/layout). | `sessionId`, `traceId`, `target`, `structure.label` (plus optional constraints/layout) | Job acceptance response including `jobId` and initial status | When a concrete build plan is ready and session is valid |
| `GET /openclaw/build/{jobId}` | Poll build-job execution state until terminal result. | `jobId` in path | Job status (`pending/running/completed/failed/cancelled`) and execution details | After submit, repeatedly query until terminal status |

#### Why this section exists

This section gives OpenClaw a stable minimum contract for P0 behavior:

1. **Perceive** current world (`environment/summary`)
2. **Act** with a single high-level build request (`build/submit`)
3. **Observe result** and recover/replan if needed (`build/{jobId}`)

Using this contract keeps orchestration simple, traceable, and compatible with the session + heartbeat model.


## Build Sub-skill Integration

Build execution is delegated to `build.md`:

- Sub-skill file: `RecodeWorkplace/docs/skills/moltcraft/build.md`
- Entry API: `POST /openclaw/build/submit`
- Tracking API: `GET /openclaw/build/{jobId}`

Main skill orchestration path for build:
1. Ensure session (`/sessions/create` if needed)
2. Keep heartbeat alive (`/sessions/heartbeat`)
3. Submit build job (`/openclaw/build/submit`)
4. Poll build status (`/openclaw/build/{jobId}`)
5. On terminal status, write build memory and continue/replan

## Set Up Your Heartbeat

Add MoltCraft to your periodic runtime heartbeat so sessions, environment context, and build progress stay fresh.

### Step 1: Add MoltCraft to your heartbeat routine

Add this to your `HEARTBEAT.md` (or equivalent periodic task list):

```markdown
## MoltCraft (every 8-12s, server interval is source of truth)
1. Read `RecodeWorkplace/docs/skills/moltcraft/heartbeat.md`
2. Ensure session exists (`/sessions/create` when needed)
3. Send `/sessions/heartbeat`
4. If `skill.json` version changed, re-read all MoltCraft skill files
```

### Step 2: Track the last applied MoltCraft skill version

Create or update `memory/heartbeat-state.json`:

```json
{
  "lastMoltcraftSkillVersion": null
}
```

Only track the version. If `RecodeWorkplace/docs/skills/moltcraft/skill.json` `version` changes, reload all MoltCraft skill files.

### Step 3: Use `heartbeat.md` as the operational source

For cadence, payload shape, session invalidation recovery, and environment/build memory update triggers, follow:

- `RecodeWorkplace/docs/skills/moltcraft/heartbeat.md`

## Quick Start (curl)

This quick start follows the actual OpenClaw runtime order:

1. Create or ensure a session
2. Keep heartbeat alive
3. Refresh environment summary for planning
4. Dispatch intent when needed
5. Submit build job
6. Poll build job to terminal status
7. Release session only when explicitly stopping

### 1) Create session

Use this when there is no active session, or when recovering from `INVALID_SESSION` / `SESSION_EXPIRED`.

Required fields:
- `sessionId`: unique session key used by all subsequent requests
- `agentId`: runtime agent identity for this session

```bash
curl -X POST http://127.0.0.1:9020/sessions/create \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "s-001",
    "agentId": "openclaw-s-001"
  }'
```

### 2) Heartbeat

Call this periodically using the server-provided interval (`nextHeartbeatIntervalMs` preferred).

Purpose:
- Keep session alive
- Report compact runtime state (`payload`)
- Drive memory update triggers in heartbeat flow

```bash
curl -X POST http://127.0.0.1:9020/sessions/heartbeat \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "s-001",
    "payload": {
      "env": { "p": [10.5, 64, -3], "ob": 1, "bz": 2, "ec": 1, "ls": 2048 },
      "recentBuilds": [],
      "hbSeq": 7,
      "ts": 1740000001234
    }
  }'
```

### 3) Get environment summary (P0-1)

Use this before planning movement/build actions, or when heartbeat flow decides environment context is stale.

Input:
- `sessionId` query parameter

```bash
curl "http://127.0.0.1:9020/openclaw/environment/summary?sessionId=s-001"
```

### 4) Dispatch intent

Use this for non-build semantic actions (for example movement or patrol intent).

Important fields:
- `intent.type`: intent category
- `intent.target`: target coordinates when applicable
- `traceId`: request trace for debugging/observability
- `reason`: optional rationale

```bash
curl -X POST http://127.0.0.1:9020/intents/dispatch \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "s-001",
    "intent": {
      "type": "move",
      "target": { "x": 1, "y": 65, "z": 2 }
    },
    "traceId": "trace-002",
    "reason": "patrol"
  }'
```

### 5) Submit build job (P0-2, delegated to build sub-skill)

Use this when a concrete build plan is ready.

Minimum recommended payload:
- `sessionId`
- `traceId`
- `target` (`x`, `y`, `z`)
- `structure.label`

Expected result:
- accepted build job with `jobId`

```bash
curl -X POST http://127.0.0.1:9020/openclaw/build/submit \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "s-001",
    "traceId": "trace-build-001",
    "target": { "x": 10, "y": 65, "z": -3 },
    "structure": {
      "label": "shelter",
      "constraints": ["weather_protection"],
      "layout": [
        { "dx": 0, "dy": 0, "dz": 0, "blockType": "planks" }
      ]
    }
  }'
```

For a complete house layout and build-specific recovery guidance, see:
- `RecodeWorkplace/docs/skills/moltcraft/build.md`

### 6) Query build job

Poll this endpoint using `jobId` from submit response until terminal status:
- `completed`
- `failed`
- `cancelled`

```bash
curl "http://127.0.0.1:9020/openclaw/build/JOB_ID"
```

### 7) Release session (only on explicit stop/finish)

Do not release during active runtime loops.
Release only when the current run is explicitly finished or stopped.

```bash
curl -X POST http://127.0.0.1:9020/sessions/release \
  -H "Content-Type: application/json" \
  -d '{ "sessionId": "s-001" }'
```

## Envelope Contract

Ingress keeps compatibility envelope:

```json
{ "ok": true, "...": "..." }
```

Adapter normalizes to:

```json
{ "success": true, "data": { } }
```

or

```json
{ "success": false, "error": { "code": "...", "message": "...", "requestId": "...", "traceId": "...", "hint": "..." } }
```

Error extraction priority:
1. `error.code/error.message`
2. `code/message`
3. fallback `INTERNAL_ERROR + unknown adapter error`

## Runtime Rules

- During active runtime (start/execute/continue), do not release the session; heartbeat must stay active.
- On `INVALID_SESSION` / `SESSION_EXPIRED`, stop using the old session, recreate it, then continue writes.
- For build operations, follow `submit -> query`; do not blindly issue low-level write actions.

## OpenClaw Local Memory Contract

OpenClaw workspace local files (not adapter-owned):

- `memory/heartbeat-state.json`
- `memory/game-understanding.json`
- `memory/build-memory.json`
- `memory/runtime-policy.json`

### `memory/game-understanding.json` (canonical)

```json
{
  "lastEnvironmentFingerprint": "",
  "lastPlayerState": { "x": 0, "y": 0, "z": 0, "yaw": 0, "pitch": 0 },
  "recentHazards": [],
  "reachableZones": [],
  "resourceHints": [],
  "anchors": []
}
```

Write triggers:
- heartbeat success -> heartbeat-state
- environment summary refresh -> game-understanding
- build job terminal status -> build-memory
- policy changed -> runtime-policy
