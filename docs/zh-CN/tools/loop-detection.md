---
read_when:
  - 用户报告智能体卡在重复的工具调用中
  - 你需要调整重复调用保护
  - 你正在编辑智能体工具/运行时策略
summary: 如何启用和调整检测重复工具调用循环的防护措施
title: 工具循环检测
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: dc3c92579b24cfbedd02a286b735d99a259b720f6d9719a9b93902c9fc66137d
  source_path: tools/loop-detection.md
  workflow: 15
---

# 工具循环检测

OpenClaw 可以防止智能体陷入重复的工具调用模式。该防护措施**默认禁用**。

仅在需要时启用它，因为严格设置可能会阻断合法的重复调用。

## 存在的原因

- 检测未取得进展的重复序列。
- 检测高频无结果循环（同一工具、相同输入、重复错误）。
- 检测已知轮询工具的特定重复调用模式。

## 配置块

全局默认值：

```json5
{
  tools: {
    loopDetection: {
      enabled: false,
      historySize: 30,
      warningThreshold: 10,
      criticalThreshold: 20,
      globalCircuitBreakerThreshold: 30,
      detectors: {
        genericRepeat: true,
        knownPollNoProgress: true,
        pingPong: true,
      },
    },
  },
}
```

每个智能体的覆盖（可选）：

```json5
{
  agents: {
    list: [
      {
        id: "safe-runner",
        tools: {
          loopDetection: {
            enabled: true,
            warningThreshold: 8,
            criticalThreshold: 16,
          },
        },
      },
    ],
  },
}
```

### 字段行为

- `enabled`：主开关。`false` 表示不执行任何循环检测。
- `historySize`：保留用于分析的最近工具调用数量。
- `warningThreshold`：将模式分类为仅警告前的阈值。
- `criticalThreshold`：阻断重复循环模式的阈值。
- `globalCircuitBreakerThreshold`：全局无进展断路器阈值。
- `detectors.genericRepeat`：检测相同工具 + 相同参数的重复模式。
- `detectors.knownPollNoProgress`：检测无状态变化的已知轮询类模式。
- `detectors.pingPong`：检测交替乒乓模式。

## 推荐设置

- 从 `enabled: true`、默认阈值开始。
- 保持阈值排序为 `warningThreshold < criticalThreshold < globalCircuitBreakerThreshold`。
- 如果出现误报：
  - 提高 `warningThreshold` 和/或 `criticalThreshold`
  - （可选）提高 `globalCircuitBreakerThreshold`
  - 仅禁用导致问题的检测器
  - 减小 `historySize` 以降低历史上下文的严格程度

## 日志和预期行为

检测到循环时，OpenClaw 会报告循环事件，并根据严重程度阻断或抑制下一个工具周期。
这可以保护用户免受失控的 token 消耗和锁死，同时保留正常的工具访问。

- 优先发出警告和临时抑制。
- 仅在重复证据积累时才升级。

## 说明

- `tools.loopDetection` 与智能体级别的覆盖合并。
- 每个智能体的配置完全覆盖或扩展全局值。
- 如果不存在任何配置，防护措施保持关闭。
