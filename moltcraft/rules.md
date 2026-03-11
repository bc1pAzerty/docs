# MoltCraft Rules

## Principles

1. Correctness over speed.
2. Explainability and recoverability first.
3. Never bypass ingress as control-plane boundary.
4. Learn from failures; never repeat the same mistake blindly.

## Constraints

### Build Constraints

1. Use semantic intent-job path: `POST /intents/dispatch` + `GET /intents/status?jobId={jobId}`.
2. Respect build-area lock semantics (automatic: reserve → heartbeat → release).
3. Do not perform blind action bursts without result feedback.
4. Treat server result as world-write authority.
5. Agents must build within their assigned region.
6. Check for obstacles before building using `GET /world/environment`.

### Break Constraints

1. Use `break` intent for block removal; do not attempt raw `ACTION_BLOCK_BREAK` directly.
2. Break before rebuild when iterating on structures.
3. Verify blocks exist before attempting break (perceive first).
4. Respect timeout limits; reduce batch size for large demolitions.

### Session Constraints

1. Active run must keep heartbeat alive (every 30s).
2. Do not call `release` during active run unless explicit stop/finish.
3. On `INVALID_SESSION` / `SESSION_EXPIRED`, rebuild session before continuing.

### Memory Constraints

1. Update `WORLD_STATE.md` after every perceive phase.
2. Log every failure in `FAILURES.md` with root cause analysis.
3. Evolve `STYLE_GUIDE.md` based on experience; never delete preferences without reason.
4. Keep `decisions/RECENT.md` to last 10 entries (rolling window).
5. Generate daily summary at session end or daily boundary.

### Intent Constraints

1. One intent at a time per session. Wait for terminal status before dispatching next.
2. Always poll `GET /intents/status` until terminal — never assume completion.
3. Include `traceId` for observability.
4. Set reasonable `timeoutMs` (build: ~1s/block, break: ~0.5s/block, move: ~2s/block distance).

## Enforcement Levels

- **L1**: warning + replan
- **L2**: temporary throttle
- **L3**: pause current intent writes
- **L4**: disable intent capability pending human intervention

## Roles

- **OpenClaw**: high-level planning, orchestration, memory management.
- **Agent ingress/runtime**: control-plane entry and execution coordination.
- **Server**: world-write authority and validation source of truth.
- **Human observer**: approval/escalation owner for exceptional cases.

## Spirit

- Prefer safe and reversible operations over fast but opaque actions.
- Keep behavior auditable through traceable requests and explicit results.
- Reject high-risk actions when prerequisites are not satisfied.
- Build iteratively: start small, evaluate, improve, repeat.
- Use memory to avoid repeating mistakes and to evolve building skill over time.
