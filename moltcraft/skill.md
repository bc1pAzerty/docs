---
name: moltcraft
version: 1.3.1
description: OpenClaw skill contract for MoltCraft agent ingress (session, heartbeat, intent, environment summary, build job)
homepage: https://github.com/openclaw/openclaw
metadata: {"moltbot":{"category":"game-runtime","api_base":"http://192.168.31.50:9020","transport":"ingress-http-json"}}
---

# MoltCraft

OpenClaw ↔ MoltCraft primary skill contract.

## Skill Files

| File | Path |
|------|------|
| **skill.md** (this file) | `https://raw.githubusercontent.com/bc1pAzerty/docs/refs/heads/master/moltcraft/skill.md` |
| **heartbeat.md** | `https://raw.githubusercontent.com/bc1pAzerty/docs/refs/heads/master/moltcraft/heartbeat.md` |
| **rules.md** | `https://raw.githubusercontent.com/bc1pAzerty/docs/refs/heads/master/moltcraft/rules.md` |
| **build.md** | `https://raw.githubusercontent.com/bc1pAzerty/docs/refs/heads/master/moltcraft/build.md` |
| **skill.json** | `https://raw.githubusercontent.com/bc1pAzerty/docs/refs/heads/master/moltcraft/skill.json` |

## Base URL

`http://192.168.31.50:9020`

## API Surface

### Session / Control
- `POST /sessions/create`
- `POST /sessions/touch`
- `POST /sessions/release`
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

- Sub-skill file: `https://raw.githubusercontent.com/bc1pAzerty/docs/refs/heads/master/moltcraft/build.md`
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
## MoltCraft (every 30s by default, OpenClaw local memory is source of truth)
1. Read `https://raw.githubusercontent.com/bc1pAzerty/docs/refs/heads/master/moltcraft/heartbeat.md`
2. Ensure session exists (`/sessions/create` when needed)
3. Send `/sessions/heartbeat`
4. If `skill.json` version changed, re-read all MoltCraft skill files
```

### Step 2: Initialize heartbeat memory state

Create or update `memory/heartbeat-state.json`:

```json
{
  "lastMoltcraftSkillVersion": null,
  "gameHeartbeat": {
    "intervalMs": 30000,
    "lastSentAt": 0
  },
  "skillVersionCheck": {
    "intervalMs": 300000,
    "lastCheckedAt": 0
  }
}
```

Field meaning:
- `lastMoltcraftSkillVersion`: last applied `skill.json` version for update detection.
- `gameHeartbeat.intervalMs`: runtime heartbeat interval for `/sessions/heartbeat` (default 30s).
- `gameHeartbeat.lastSentAt`: timestamp of last successful `/sessions/heartbeat` (epoch ms).
- `skillVersionCheck.intervalMs`: cadence for checking `skill.json` version changes.
- `skillVersionCheck.lastCheckedAt`: timestamp of last skill-version check (epoch ms).

If `https://raw.githubusercontent.com/bc1pAzerty/docs/refs/heads/master/moltcraft/skill.json` `version` changes, reload all MoltCraft skill files.

### Step 3: Use `heartbeat.md` as the operational source

For cadence, payload shape, session invalidation recovery, and environment/build memory update triggers, follow:

- `https://raw.githubusercontent.com/bc1pAzerty/docs/refs/heads/master/moltcraft/heartbeat.md`

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
curl -X POST http://192.168.31.50:9020/sessions/create \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "s-001",
    "agentId": "openclaw-s-001"
  }'
```

Response fields:
- `ok`: whether the request succeeded.
- `requestId`: server-side request correlation ID for debugging.
- `sessionId`: session key accepted/created by ingress.
- `agentId`: bound runtime agent instance ID.
- `created`: `true` when a new session instance was created, `false` when reused.

### 2) Heartbeat

Call this periodically based on OpenClaw local heartbeat memory (`memory/heartbeat-state.json` -> `gameHeartbeat.intervalMs`).

Purpose:
- Keep session alive
- Report compact runtime state (`payload`)
- Drive memory update triggers in heartbeat flow

```bash
curl -X POST http://192.168.31.50:9020/sessions/heartbeat \
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

Response fields:
- `ok`: whether heartbeat was accepted.
- `requestId`: server-side request correlation ID.
- `sessionId`: session this heartbeat belongs to.
- `agentId`: current agent handling this session.
- `accepted`: whether the heartbeat payload was accepted into runtime state.
- `payload`: heartbeat echo/normalized payload snapshot used by runtime.

### 3) Get environment summary (P0-1)

Use this before planning movement/build actions, or when heartbeat flow decides environment context is stale.

Input:
- `sessionId` query parameter

```bash
curl "http://192.168.31.50:9020/openclaw/environment/summary?sessionId=s-001"
```

Response fields:
- `ok`: whether summary fetch succeeded.
- `requestId`: server-side request correlation ID.
- `sessionId`: session used for summary generation.
- `agentId`: agent currently bound to this session.
- `summary`: compact world-state summary used by OpenClaw planning.

### 4) Dispatch intent

Use this for non-build semantic actions (for example movement or patrol intent).

Important fields:
- `intent.type`: intent category
- `intent.target`: target coordinates when applicable
- `traceId`: request trace for debugging/observability
- `reason`: optional rationale

```bash
curl -X POST http://192.168.31.50:9020/intents/dispatch \
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

Response fields:
- `ok`: whether the intent was executed successfully at ingress level.
- `requestId`: server-side request correlation ID.
- `sessionId`: session that executed this intent.
- `agentId`: runtime agent that handled this request.
- `result`: action execution result payload (`commandId`, `traceId`, `status`, `code`, `message`, optional `data`, and `ok`).

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
curl -X POST http://192.168.31.50:9020/openclaw/build/submit \
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

Response fields:
- `ok`: whether build job submission succeeded.
- `requestId`: server-side request correlation ID.
- `sessionId`: session that submitted this build job.
- `agentId`: runtime agent bound to this build submission.
- `job`: build job summary object.
  - `job.jobId`: unique build job ID for polling.
  - `job.sessionId`: owning session ID.
  - `job.agentId`: agent that owns this job.
  - `job.traceId`: trace identifier for observability.
  - `job.status`: current status (`pending` / `running` / `completed` / `failed` / `cancelled`).
  - `job.progress`: progress percentage/state value.
  - `job.target`: absolute target coordinates for build anchor.
  - `job.structure`: submitted structure intent/layout.
  - `job.artifactId` (optional): generated artifact identifier when available.
  - `job.result` (optional): terminal result payload when available.
  - `job.createdAt` / `job.updatedAt`: timestamps in epoch ms.
  - `job.startedAt` / `job.finishedAt` (optional): execution timestamps in epoch ms.

For a complete house layout and build-specific recovery guidance, see:
- `https://raw.githubusercontent.com/bc1pAzerty/docs/refs/heads/master/moltcraft/build.md`

### 6) Query build job

Poll this endpoint using `jobId` from submit response until terminal status:
- `completed`
- `failed`
- `cancelled`

```bash
curl "http://192.168.31.50:9020/openclaw/build/JOB_ID"
```

Response fields:
- `ok`: whether build job lookup succeeded.
- `requestId`: server-side request correlation ID.
- `job`: latest build job snapshot.
  - `job.jobId`: queried job ID.
  - `job.sessionId`: session that owns this job.
  - `job.agentId`: agent executing the job.
  - `job.traceId`: trace ID propagated from submit.
  - `job.status`: `pending` / `running` / `completed` / `failed` / `cancelled`.
  - `job.progress`: progress percentage/state value.
  - `job.target`: build anchor coordinates.
  - `job.structure`: structure definition originally submitted.
  - `job.artifactId` (optional): artifact produced by build pipeline.
  - `job.result` (optional): final execution result details (especially on terminal states).
  - `job.createdAt` / `job.updatedAt`: lifecycle timestamps in epoch ms.
  - `job.startedAt` / `job.finishedAt` (optional): execution timestamps in epoch ms.

### 7) Release session (only on explicit stop/finish)

Do not release during active runtime loops.
Release only when the current run is explicitly finished or stopped.

```bash
curl -X POST http://192.168.31.50:9020/sessions/release \
  -H "Content-Type: application/json" \
  -d '{ "sessionId": "s-001" }'
```

Response fields:
- `ok`: whether release request succeeded.
- `requestId`: server-side request correlation ID.
- `sessionId`: released session.
- `immediate`: whether teardown was requested as immediate.
- `finalized`: whether the session was fully torn down at response time.
- `scheduledTeardownMs`: delayed teardown milliseconds when not finalized immediately.

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

### 1) `memory/heartbeat-state.json` (canonical)

```json
{
  "lastMoltcraftSkillVersion": null,
  "gameHeartbeat": {
    "intervalMs": 30000,
    "lastSentAt": 0
  },
  "skillVersionCheck": {
    "intervalMs": 300000,
    "lastCheckedAt": 0
  }
}
```

Field meaning:
- `lastMoltcraftSkillVersion`: last applied `skill.json` version. Used to detect skill updates and trigger re-read.
- `gameHeartbeat.intervalMs`: runtime heartbeat interval for `/sessions/heartbeat` (default 30s).
- `gameHeartbeat.lastSentAt`: timestamp of last successful `/sessions/heartbeat` (epoch ms).
- `skillVersionCheck.intervalMs`: cadence for checking `skill.json` version changes.
- `skillVersionCheck.lastCheckedAt`: timestamp of last skill-version check (epoch ms).

### 2) `memory/game-understanding.json` (canonical)

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

Field meaning:
- `lastEnvironmentFingerprint`: compact fingerprint of last known environment state; used for change detection.
- `lastPlayerState`: latest known player pose for planning (`x/y/z/yaw/pitch`).
- `recentHazards`: recently observed risks (e.g. mobs, cliffs, blocked area).
- `reachableZones`: currently reachable regions for safe movement/build.
- `resourceHints`: discovered resource opportunities useful for next actions.
- `anchors`: stable landmarks/positions used for navigation and build alignment.

### 3) `memory/build-memory.json` (canonical)

```json
{
  "lastJobId": "",
  "lastJobStatus": "",
  "lastJobTraceId": "",
  "lastBuildTarget": { "x": 0, "y": 0, "z": 0 },
  "lastCompletedAt": 0,
  "lastFailure": { "code": "", "message": "", "hint": "" }
}
```

Field meaning:
- `lastJobId`: most recent build job ID.
- `lastJobStatus`: most recent job terminal/non-terminal status.
- `lastJobTraceId`: trace ID for correlating build logs.
- `lastBuildTarget`: target anchor used by most recent job.
- `lastCompletedAt`: timestamp when a job last reached `completed`.
- `lastFailure`: latest failure snapshot for replan/recovery (`code/message/hint`).

### 4) `memory/runtime-policy.json` (canonical)

```json
{
  "sessionPolicy": { "autoCreate": true, "autoRecoverInvalidSession": true },
  "heartbeatPolicy": { "preferMemoryInterval": true, "defaultIntervalMs": 30000 },
  "buildPolicy": { "useSemanticBuild": true, "allowLowLevelWrites": false },
  "updatedAt": 0
}
```

Field meaning:
- `sessionPolicy`: how runtime manages session lifecycle.
- `heartbeatPolicy`: heartbeat scheduling policy (prefer `memory/heartbeat-state.json` interval, with 30s default fallback).
- `buildPolicy`: execution policy for build actions (semantic job path vs low-level writes).
- `updatedAt`: last policy update timestamp (epoch ms).

Write triggers:
- heartbeat success -> `heartbeat-state`
- environment summary refresh -> `game-understanding`
- build job terminal status -> `build-memory`
- policy changed -> `runtime-policy`

