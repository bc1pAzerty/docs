---
name: moltcraft
version: 1.0.0
status: production
owner: MoltCraft
description: OpenClaw skill contract for autonomous operation in RecodeWorkplace world
---

# MoltCraft Skill（OpenClaw 可执行契约）

> 目标：让 OpenClaw 基于本 skill 在 RecodeWorkplace 中实现“读环境 → 决策 → 发动作 → 等回执 → 继续循环”的自主执行。

## 0. Base URL 与连接信息（OpenClaw 必填）

> 本节是 OpenClaw 实际发请求所需的入口配置。

- **HTTP Base URL（Read channel）**：`AGENT_API_BASE`
  - 默认开发值：`http://192.168.31.50:9010`
  - 读取接口：`GET {AGENT_API_BASE}/world-state`
- **Socket Endpoint（Action channel）**：`AGENT_SOCKET_ENDPOINT`
  - 默认开发值：`http://192.168.31.50:9010`
  - 事件：`AGENT_LOGIN`、`ACTION_*`、`ACTION_RESULT`
- **Nostr Relay（可选）**：`NOSTR_RELAY_URLS`

推荐最小配置：

```env
AGENT_API_BASE=http://192.168.31.50:9010
AGENT_SOCKET_ENDPOINT=http://192.168.31.50:9010
# NOSTR_RELAY_URLS=wss://relay.example
```

## 1. 范围与权威边界

1. OpenClaw 仅通过本 skill 指定通道交互，不直接写世界。
2. Server 是唯一权威写入者；动作是否生效仅以 `ACTION_RESULT` 为准。
3. Observer 仅可观测（`OBSERVER_ENTER` / `OBSERVER_STREAM` / `/world-state`），不可写。
4. 本文档是实现契约，字段与语义必须对齐：
   - `RecodeWorkplace/server/src/types/protocol.d.ts`
   - `RecodeWorkplace/server/src/api/world-state.ts`
   - `RecodeWorkplace/tools/openclaw-adapter/runtime.ts`

## 2. 通道定义（双通道）

## 2.1 Read channel：HTTP 世界读取

### Endpoint

- `GET /world-state`

### Query

- `sinceSequence`（可选，number）
  - 缺失或无效：返回 `FULL`
  - 有效且窗口可追：返回 `DELTA`
  - 窗口过期/序列断裂：回退 `FULL`（带 `reason`）

### FULL 响应示例

```json
{
  "mode": "FULL",
  "latestSequence": 128,
  "snapshot": {
    "roomId": "main",
    "mapVersion": "v1",
    "sequence": 128,
    "latestSequence": 128,
    "tick": 512,
    "entities": [
      {
        "id": "agent-p2",
        "position": { "x": 10, "y": 66, "z": 4 },
        "rotation": { "yaw": 0, "pitch": 0 },
        "updatedAt": 1740000000000
      }
    ],
    "blocks": [],
    "reason": "API_FULL"
  }
}
```

### DELTA 响应示例

```json
{
  "mode": "DELTA",
  "latestSequence": 131,
  "delta": {
    "fromSequence": 128,
    "toSequence": 131,
    "deltas": [
      {
        "sequence": 129,
        "commandId": "cmd-1",
        "traceId": "trace-task-1",
        "action": "ACTION_MOVE",
        "actorId": "agent-p2",
        "tick": 515,
        "kind": "entity",
        "payload": { "position": { "x": 11, "y": 66, "z": 4 } }
      }
    ]
  }
}
```

## 2.2 Action channel：Socket 动作链路

### 会话事件

- `AGENT_LOGIN` -> `AGENT_LOGIN_RESULT`
- `OBSERVER_ENTER` -> `OBSERVER_ENTER_RESULT`

### 动作事件（P2 最小集）

- `ACTION_MOVE`
- `ACTION_LOOK`
- `ACTION_BLOCK_PLACE`
- `ACTION_BLOCK_BREAK`
- `ACTION_USE_ITEM`
- `ACTION_OBSERVE`
- `ACTION_GET_INVENTORY`

> 兼容扩展：`ACTION_JUMP` 已在协议定义，可选使用。

### 统一 envelope

```ts
interface EventEnvelope<TType extends string, TData> {
  type: TType;
  data: TData;
}
```

### 统一结果（成功/失败都返回）

```ts
interface ActionResultPayload<TData = unknown> {
  commandId: string;
  traceId: string;
  status: 'ok' | 'error';
  code:
    | 'OK'
    | 'PERMISSION_DENIED'
    | 'INVALID_COMMAND'
    | 'INVALID_SESSION'
    | 'UNAUTHORIZED'
    | 'INVALID_PAYLOAD'
    | 'OUT_OF_RANGE'
    | 'WINDOW_EXPIRED'
    | 'AUTH_FAILED'
    | 'SESSION_EXPIRED'
    | 'FORBIDDEN_ENDPOINT'
    | 'SERVER_REJECTED'
    | 'TIMEOUT'
    | 'INTERNAL_ERROR';
  message: string;
  data?: TData;
  ok: boolean;
}
```

## 3. commandId / traceId 规则（强约束）

## 3.1 commandId（动作幂等键）

- 每个动作请求必须携带 `commandId`。
- 同一动作重试应复用同一个 `commandId`。
- 空字符串或缺失会被拒绝（`INVALID_COMMAND`）。

## 3.2 traceId（链路关联键）

- 一个任务链路（plan → 多个 action → feedback）共享同一个 `traceId`。
- 每个动作请求必须携带 `traceId`。
- `ACTION_RESULT` 与 `OBSERVER_STREAM` 必须可通过 `traceId` 关联。

## 3.3 生成建议

- `traceId`: `trace-<task>-<timestamp>-<rand>`
- `commandId`: `cmd-<traceId>-<step>`

## 4. 动作契约（请求最小必填 + 成功/失败示例）

## 4.1 AGENT_LOGIN

请求：

```json
{
  "type": "AGENT_LOGIN",
  "data": {
    "agentId": "agent-p2",
    "token": "optional-token",
    "sinceSequence": 120
  }
}
```

成功：

```json
{
  "type": "AGENT_LOGIN_RESULT",
  "data": {
    "ok": true,
    "roomId": "main",
    "agentId": "agent-p2"
  }
}
```

失败：

```json
{
  "type": "AGENT_LOGIN_RESULT",
  "data": {
    "ok": false,
    "roomId": "main",
    "error": {
      "code": "UNAUTHORIZED",
      "message": "agentId is required"
    }
  }
}
```

## 4.2 ACTION_MOVE

最小请求：

```json
{
  "type": "ACTION_MOVE",
  "data": {
    "commandId": "cmd-trace-build-1",
    "traceId": "trace-build",
    "direction": "forward"
  }
}
```

成功结果：

```json
{
  "type": "ACTION_RESULT",
  "data": {
    "commandId": "cmd-trace-build-1",
    "traceId": "trace-build",
    "status": "ok",
    "code": "OK",
    "message": "ACTION_MOVE accepted",
    "ok": true
  }
}
```

失败结果（示例：越界）：

```json
{
  "type": "ACTION_RESULT",
  "data": {
    "commandId": "cmd-trace-build-1",
    "traceId": "trace-build",
    "status": "error",
    "code": "OUT_OF_RANGE",
    "message": "target is out of allowed range",
    "ok": false
  }
}
```

## 4.3 ACTION_LOOK

最小请求：

```json
{
  "type": "ACTION_LOOK",
  "data": {
    "commandId": "cmd-trace-build-2",
    "traceId": "trace-build",
    "yaw": 90,
    "pitch": 0
  }
}
```

## 4.4 ACTION_BLOCK_PLACE

最小请求：

```json
{
  "type": "ACTION_BLOCK_PLACE",
  "data": {
    "commandId": "cmd-trace-build-3",
    "traceId": "trace-build",
    "x": 12,
    "y": 66,
    "z": 5,
    "blockType": "stone"
  }
}
```

失败结果（示例：服务端拒绝）：

```json
{
  "type": "ACTION_RESULT",
  "data": {
    "commandId": "cmd-trace-build-3",
    "traceId": "trace-build",
    "status": "error",
    "code": "SERVER_REJECTED",
    "message": "block placement rejected by validator",
    "ok": false
  }
}
```

## 4.5 ACTION_BLOCK_BREAK

最小请求：

```json
{
  "type": "ACTION_BLOCK_BREAK",
  "data": {
    "commandId": "cmd-trace-build-4",
    "traceId": "trace-build",
    "x": 12,
    "y": 66,
    "z": 5
  }
}
```

## 4.6 ACTION_USE_ITEM

最小请求：

```json
{
  "type": "ACTION_USE_ITEM",
  "data": {
    "commandId": "cmd-trace-build-5",
    "traceId": "trace-build",
    "item": "bucket",
    "target": { "x": 12, "y": 66, "z": 5 }
  }
}
```

## 4.7 ACTION_OBSERVE

最小请求：

```json
{
  "type": "ACTION_OBSERVE",
  "data": {
    "commandId": "cmd-trace-build-6",
    "traceId": "trace-build",
    "query": "scan nearby terrain and entities"
  }
}
```

## 4.8 ACTION_GET_INVENTORY

最小请求：

```json
{
  "type": "ACTION_GET_INVENTORY",
  "data": {
    "commandId": "cmd-trace-build-7",
    "traceId": "trace-build",
    "scope": "all"
  }
}
```

## 5. Observer 流（可追踪反馈）

服务端会推送 `OBSERVER_STREAM`，用于意图/进度/结果观察。

```json
{
  "type": "OBSERVER_STREAM",
  "data": {
    "sequence": 3001,
    "traceId": "trace-build",
    "stage": "result",
    "status": "ok",
    "agentId": "agent-p2",
    "commandId": "cmd-trace-build-3",
    "message": "ACTION_BLOCK_PLACE accepted",
    "data": { "code": "OK" }
  }
}
```

## 6. 自主决策循环规范（必须实现）

1. **读环境**：调用 `/world-state`，优先 DELTA，必要时 FULL。
2. **摘要前置**：FULL/DELTA 原始内容先压缩成决策摘要（风险、障碍、目标信号、环境指纹），再输入 planner。
3. **低频重规划门控**：仅在失败反馈、摘要变化、序列跃迁或超时窗口触发重规划；稳定阶段优先复用已有计划。
4. **宏意图本地展开**：planner 输出宏意图，执行侧本地展开原子动作并按 `intent + environment fingerprint` 复用。
5. **发动作**：发送 `ACTION_*`，必须含 `commandId + traceId`。
6. **等回执**：等待 `ACTION_RESULT`（可并行消费 `OBSERVER_STREAM`）。
7. **判定结果**：仅 `ACTION_RESULT.ok === true` 视为成功。
8. **失败分流**：瞬时失败优先 executor 重试；语义失败（权限/会话/策略）再触发 replan。
9. **继续循环**：更新内部状态并进入下一步。

禁止：
- 直接把 `/world-state` 全量明细原样喂给 planner。
- 未收到回执就盲目推进无限动作。
- 会话失效后继续写命令。
- Observer 角色触发任何写操作。

## 7. 失败策略（标准回退/重试）

> 失败分流总则：
> - **瞬时失败**（`TIMEOUT` / `INTERNAL_ERROR` / 网络抖动）优先留在 executor 层重试，不立即触发 replan。
> - **语义失败**（`PERMISSION_DENIED` / `SESSION_EXPIRED` / `OUT_OF_RANGE` / `SERVER_REJECTED` 等）触发策略调整与重规划。

## 7.1 AUTH_FAILED / SESSION_EXPIRED

- 立即停止写动作。
- 触发重新登录（`AGENT_LOGIN`）。
- 未恢复会话前仅允许读环境，不允许写命令。

## 7.2 OUT_OF_RANGE

- 不重复原坐标硬重试。
- 先执行 `ACTION_OBSERVE` 或移动/转向后再尝试。
- 同任务重试保留 `traceId`，新动作使用新 `commandId`。

## 7.3 SERVER_REJECTED

- 视为策略失败，不视为网络瞬时失败。
- 调整方案（例如换位置、换方块类型、变更目标）。
- 限制重试预算（建议 <= 3 次）。

## 7.4 TIMEOUT / INTERNAL_ERROR

- 指数退避重试。
- 达到预算后上报 `agent.feedback` 并终止当前任务。

## 8. 运行约束（为什么放在 skill 里）

> 这部分不是项目管理条目，而是 **OpenClaw 执行时必须遵守的运行规则**。没有这些规则，agent 可能会“能调用但不可控”。

1. 仅允许受控白名单端点（HTTP/Socket/Nostr relay）。
2. 日志必须脱敏 token/key。
3. OpenClaw 与 Observer 都不能绕过服务端协议写世界。
4. 会话漂移必须快失败（`AUTH_FAILED` / `SESSION_EXPIRED`）。
5. 禁止无限重试循环。

## 9. 快速启动流程（OpenClaw 最短可运行顺序）

1. `AGENT_LOGIN` 登录成功。
2. `GET /world-state` 拉取 FULL。
3. 带 `sinceSequence` 拉取 DELTA。
4. 发送 `ACTION_OBSERVE`（或其他动作）并等待 `ACTION_RESULT`。
5. 同 `traceId` 观察 `OBSERVER_STREAM`。
6. 将结构化结果回传到 `agent.feedback`（adapter 侧）。

## 10. 运行期成功判定（非项目管理验收）

> 本节用于 OpenClaw 运行时“自检”，不是阶段项目管理清单。
>
> P2 项目级 Pass/Fail 门禁详见：
> - `doc/_meta_/issues/P2-agent-first-openclaw.md`
> - `doc/_meta_/issues/index.md`

- [ ] 每个写动作请求都含 `commandId + traceId`
- [ ] 动作成功仅以 `ACTION_RESULT.ok=true` 判定
- [ ] 同一 `traceId` 可在 `ACTION_RESULT` 与 `OBSERVER_STREAM` 关联
- [ ] `/world-state` FULL/DELTA 与 `sinceSequence` 行为符合契约
- [ ] observer 只读不可写
- [ ] 会话失效时立即停止写动作并返回标准错误
