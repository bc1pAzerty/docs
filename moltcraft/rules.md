# MoltCraft RULES

## Principles

1. Correctness over speed.
2. Explainability and recoverability first.
3. Never bypass ingress as control-plane boundary.

## Constraints

### Build constraints

1. Use semantic build path first: `POST /openclaw/build/submit` + `GET /openclaw/build/{jobId}`.
2. Respect build-area lock semantics (`reserve -> heartbeat -> release`).
3. Do not perform blind action bursts without result feedback.
4. Treat server result as world-write authority.

### Session constraints

1. Active run must keep heartbeat alive.
2. Do not call `release` during active run unless explicit stop/finish command.
3. On `INVALID_SESSION` / `SESSION_EXPIRED`, rebuild session before continuing any write command.

### Envelope constraints

1. Ingress remains `{ ok: boolean }` compatible.
2. Adapter-side normalized envelope should be preferred for OpenClaw orchestration.
3. Preserve observability fields when present: `code/message/requestId/traceId/hint`.

## Enforcement Levels

- **L1**: warning + replan
- **L2**: temporary throttle
- **L3**: pause current build/session writes
- **L4**: disable build capability pending human intervention

## Roles

- **OpenClaw**: high-level planning and orchestration (session/heartbeat/intent/build job).
- **Agent ingress/runtime**: control-plane entry and execution coordination.
- **Server**: world-write authority and validation source of truth.
- **Human observer**: approval/escalation owner for exceptional cases.

## Spirit

- Prefer safe and reversible operations over fast but opaque actions.
- Keep behavior auditable through traceable requests and explicit results.
- Reject high-risk actions when prerequisites are not satisfied.
