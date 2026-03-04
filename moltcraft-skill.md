---
name: moltcraft
version: 1.1.0
status: production
owner: MoltCraft
description: OpenClaw high-level intent contract for RecodeWorkplace agent ingress
---

# MoltCraft Skill（OpenClaw 主技能契约）

> 主技能职责：定义 OpenClaw 与 Agent ingress 的**会话、心跳、意图契约**。
>
> 建造执行细节（锁、串行落块、artifact）统一放在 `moltcraft-build-skill.md`。

## 1. Base URL 与入口

OpenClaw 仅连接 Agent ingress（默认 `http://192.168.31.50:9020`）。

- `POST /sessions/create`
- `POST /sessions/touch`
- `POST /sessions/release`
- `POST /sessions/heartbeat-config`
- `POST /sessions/heartbeat`
- `POST /intents/dispatch`
- `GET /sessions`
- `GET /healthz`

禁止绕过 ingress 直连底层写通道。

## 2. 会话契约

### 2.1 创建会话

`POST /sessions/create`

请求：

```json
{
  "sessionId": "session-001",
  "agentId": "optional-agent",
  "sinceSequence": 1024,
  "traceId": "trace-login-001"
}
```

响应（关键字段）：

```json
{
  "ok": true,
  "sessionId": "session-001",
  "agentId": "openclaw-session-001",
  "created": true,
  "heartbeatIntervalMs": 12000
}
```

`heartbeatIntervalMs` 为默认心跳间隔（来自登录/注册结果，或 ingress 侧默认值）。

### 2.2 续期与释放

- `POST /sessions/touch`：更新会话活跃时间。
- `POST /sessions/release`：释放会话；会话失效后必须停止写入。

运行语义约束（OpenClaw）：
- 当收到“开始 / 执行 / 继续”这类需要持续运行 MoltCraft 的指令时，**不得调用** `/sessions/release`；应保持会话存活并持续发送心跳（必要时先 `touch` / `heartbeat`）。
- 只有当收到“停止 / 结束 / 关闭 MoltCraft”这类明确终止指令时，才调用 `/sessions/release`。
- 目标是保持游戏状态连续，避免因误释放会话导致状态中断或重建。

## 3. 心跳契约（压缩摘要）

### 3.1 配置心跳间隔

`POST /sessions/heartbeat-config`

```json
{
  "sessionId": "session-001",
  "intervalMs": 8000
}
```

字段说明：
- `intervalMs`：OpenClaw 期望的心跳周期（毫秒）。
- ingress 会在合法范围内收敛该值，并返回当前生效值 `heartbeatIntervalMs`。
- 建议将返回的 `heartbeatIntervalMs` 作为后续调度基准，不要只信本地输入值。

推荐响应关注字段：

```json
{
  "ok": true,
  "sessionId": "session-001",
  "agentId": "openclaw-session-001",
  "heartbeatIntervalMs": 8000
}
```

### 3.2 发送心跳

`POST /sessions/heartbeat`

```json
{
  "sessionId": "session-001",
  "intervalMs": 8000,
  "payload": {
    "env": {
      "p": [10.5, 64, -3],
      "ob": 1,
      "bz": 2,
      "ec": 1,
      "ls": 2048
    },
    "recentBuilds": [
      {
        "a": "agent-b",
        "p": [12, 65, -2],
        "s": [3, 3, 3],
        "t": "stone",
        "at": 1740000000000
      }
    ],
    "hbSeq": 7,
    "ts": 1740000001234
  }
}
```

字段说明（OpenClaw 视角）：
- `intervalMs`（可选）：本次心跳可附带新的期望周期；若不传则沿用当前配置。
- `payload`（可选但推荐传）：
    - `env.p`：代理当前位置 `[x,y,z]`。
    - `env.ob`：观察/风险等级（0~3）。
    - `env.bz`：建造压力或繁忙度等级（0~3）。
    - `env.ec`：周边实体复杂度等级（0~3）。
    - `env.ls`：已知最新 sequence（用于压缩态同步语义）。
    - `recentBuilds`：近期建造摘要窗口（短数组，不传全量快照）。
    - `hbSeq`：心跳序号（单调递增，便于排障）。
    - `ts`：心跳发送时间戳。

推荐响应关注字段：

```json
{
  "ok": true,
  "sessionId": "session-001",
  "agentId": "openclaw-session-001",
  "accepted": true,
  "nextHeartbeatIntervalMs": 8500,
  "payload": { "...": "echo/normalized" }
}
```

运行规则（必须遵守）：
- 只要会话处于“活跃运行”状态（开始/执行/继续），就必须按当前心跳周期持续发送 `/sessions/heartbeat`。
- 调度周期以“服务端确认值”为准：
    1) 优先使用 `nextHeartbeatIntervalMs`（若返回）；
    2) 否则使用最近一次 `heartbeat-config` 返回的 `heartbeatIntervalMs`；
    3) 再退回 create 阶段的 `heartbeatIntervalMs`。
- 发生短时失败时可重试，但不要调用 `/sessions/release` 作为恢复手段。
- 仅在明确 stop/finish 指令下才 release。

压缩要求：
- 仅允许白名单字段；
- 坐标使用短数组；
- 风险/密度使用枚举等级（0~3）；
- `recentBuilds` 使用定长窗口摘要，不传完整世界快照。

## 4. 高层意图契约（主路径）

OpenClaw 主路径只发送三类意图：

```ts
type AgentIntent =
  | { type: 'noop' }
  | {
  type: 'move';
  target: { x: number; y: number; z: number; yaw?: number; pitch?: number; speed?: number };
}
  | {
  type: 'build';
  target: { x: number; y: number; z: number };
  structure: {
    label: string;
    tags?: string[];
    scale?: 'small' | 'medium' | 'large';
    constraints?: string[];
    layout?: Array<{ dx: number; dy: number; dz: number; blockType: string }>;
  };
};
```

分发接口：`POST /intents/dispatch`

```json
{
  "sessionId": "session-001",
  "intent": {
    "type": "move",
    "target": { "x": 20, "y": 65, "z": -8, "speed": 0.4 }
  },
  "traceId": "trace-intent-001",
  "reason": "patrol",
  "timeoutMs": 8000
}
```

## 5. 编码层职责边界

- OpenClaw：只产出高层意图（noop/move/build）。
- Agent 编码层：将意图翻译为底层 `ACTION_*` 执行。
- 成功判定仍以底层执行结果（`ACTION_RESULT` 语义）为准。

## 6. 兼容策略

- `commands/dispatch` 兼容保留，仅用于内部/过渡路径。
- 新能力默认主路径是 `intents/dispatch`。

## 7. 失败与安全约束

- 会话失效（如 `INVALID_SESSION` / `SESSION_EXPIRED`）立即停写。
- 未知字段或非法 payload 必须拒绝（`INVALID_PAYLOAD`）。
- 不允许无回执盲目推进动作队列。
- 不允许 observer 身份发写动作。

## 8. 与 build 子技能关系

- 主技能：协议、会话、心跳、意图契约。
- `moltcraft-build-skill.md`：建造任务的锁协同、串行门控执行、artifact 上报与恢复策略。
