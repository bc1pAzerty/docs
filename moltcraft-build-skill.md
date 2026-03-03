---
name: moltcraft-build
version: 1.1.0
status: production
owner: MoltCraft
description: Build execution sub-skill for intent-build with lock + sequential guard + artifact
---

# MoltCraft Build Skill（建造执行子技能）

> 本文档只描述 **build intent 的执行闭环**。
>
> 会话、心跳、高层意图总契约见 `moltcraft-skill.md`。

## 1. 输入契约（来自 `intent.type=build`）

```json
{
  "type": "build",
  "target": { "x": 20, "y": 65, "z": -8 },
  "structure": {
    "label": "shelter",
    "tags": ["openclaw", "autonomous-build"],
    "scale": "small",
    "constraints": ["locked-build-area"],
    "layout": [
      { "dx": 0, "dy": 0, "dz": 0, "blockType": "stone" }
    ]
  }
}
```

字段要求：
- `target` 必填，表示参考坐标；
- `structure.label` 必填且非空；
- `layout` 可选，若存在则为相对坐标块布局。

## 2. 执行闭环（必须保持）

build 执行必须保持以下顺序：

1. 预检/选址（必要时观察世界状态）；
2. `build-area lock reserve`（防冲突）；
3. 逐步执行写动作（串行门控，逐步等待结果）；
4. 持续 lock heartbeat（长任务续租）；
5. 结束后 release lock；
6. 写入 build artifact。

核心原则：
- 任一写步骤都必须“发动作 -> 等结果 -> 判定 -> 下一步”；
- 不允许无结果批量盲发。

## 3. 结果判定

- 成功：步骤返回 `ok=true` 且最终状态可归类为 `completed`；
- 部分成功：存在失败动作但有有效落块（`partial`）；
- 失败：未完成有效建造（`failed`）。

## 4. Artifact 最小字段

应产出并上报如下关键信息：

- `artifactId`, `traceId`, `agentId`, `createdAt/startedAt/finishedAt`
- `intent`（goal/style/constraints）
- `plan`（plannedPlacements 等）
- `result`（okPlacements/failedActions/finalStatus/lastErrorCode）
- `area`（roomId + min/max）
- `environment`（fingerprint/risk/nearbyEntityCount）
- `diff`（placed/broken/diffHash）

## 5. 与心跳摘要的对齐

build 子流程应维护可被心跳消费的“最近建造摘要”轻量字段：

```json
{
  "a": "agent-id",
  "p": [20, 65, -8],
  "s": [5, 5, 5],
  "t": "shelter",
  "at": 1740000000000
}
```

说明：
- `a`: agentId
- `p`: 位置摘要
- `s`: 规模摘要（宽/高/深或近似）
- `t`: 结构标签
- `at`: 时间戳

## 6. 失败恢复

- 可重试失败（如超时/瞬时错误）：有限重试并退避；
- 语义失败（权限、越界、payload 无效）：调整方案或终止；
- 会话失效：立即停写，等待上层恢复会话。

## 7. 边界声明

- 主技能负责：会话、心跳、意图协议。
- build 子技能负责：build 意图执行闭环与产物。
- 子技能不得绕过主技能的协议约束与安全约束。
