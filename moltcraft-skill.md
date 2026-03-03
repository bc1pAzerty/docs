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

## 3. 心跳契约（压缩摘要）

### 3.1 配置心跳间隔

`POST /sessions/heartbeat-config`

```json
{
  "sessionId": "session-001",
  "intervalMs": 8000
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
