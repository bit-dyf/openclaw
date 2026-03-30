---
title: "提示词缓存"
summary: "提示词缓存的配置参数、合并顺序、提供商行为及调优模式"
read_when:
  - 希望通过缓存保留来降低提示词令牌成本
  - 在多智能体设置中需要按智能体配置缓存行为
  - 正在同时调优心跳保活和 cache-ttl 剪枝
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 7952e90d0d6eb23fee4e0046220dddc7c89dc19aae0129d0619290e081a92778
  source_path: reference/prompt-caching.md
  workflow: 15
---

# 提示词缓存

提示词缓存是指模型提供商可以在多个轮次中复用未变更的提示词前缀（通常是系统/开发者指令以及其他稳定的上下文），而无需每次都重新处理。首次匹配请求会写入缓存令牌（`cacheWrite`），后续匹配请求可以读回这些令牌（`cacheRead`）。

这一机制的意义在于：降低令牌成本、加快响应速度，并使长时运行会话的性能更加可预测。如果不使用缓存，每次轮次都会支付完整的提示词成本，即便大部分输入没有发生变化。

本页涵盖所有影响提示词复用和令牌成本的缓存相关配置项。

有关 Anthropic 的定价详情，请参阅：
[https://docs.anthropic.com/docs/build-with-claude/prompt-caching](https://docs.anthropic.com/docs/build-with-claude/prompt-caching)

## 主要配置项

### `cacheRetention`（模型级别和按智能体）

在模型参数中设置缓存保留策略：

```yaml
agents:
  defaults:
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "short" # none | short | long
```

按智能体覆盖：

```yaml
agents:
  list:
    - id: "alerts"
      params:
        cacheRetention: "none"
```

配置合并顺序：

1. `agents.defaults.models["provider/model"].params`
2. `agents.list[].params`（匹配智能体 id；按键覆盖）

### 旧版 `cacheControlTtl`

旧版值仍被接受并映射如下：

- `5m` -> `short`
- `1h` -> `long`

新配置请优先使用 `cacheRetention`。

### `contextPruning.mode: "cache-ttl"`

在缓存 TTL 窗口后剪除旧的工具结果上下文，使空闲后的请求不会重新缓存过大的历史记录。

```yaml
agents:
  defaults:
    contextPruning:
      mode: "cache-ttl"
      ttl: "1h"
```

完整行为请参阅[会话剪枝](/concepts/session-pruning)。

### 心跳保活

心跳机制可以维持缓存窗口的活跃状态，减少空闲间隔后的重复缓存写入。

```yaml
agents:
  defaults:
    heartbeat:
      every: "55m"
```

按智能体的心跳配置支持在 `agents.list[].heartbeat` 中设置。

## 提供商行为

### Anthropic（直连 API）

- 支持 `cacheRetention`。
- 使用 Anthropic API 密钥认证配置文件时，OpenClaw 会在未设置的情况下为 Anthropic 模型引用自动填充 `cacheRetention: "short"`。

### Amazon Bedrock

- Anthropic Claude 模型引用（`amazon-bedrock/*anthropic.claude*`）支持显式传递 `cacheRetention`。
- 非 Anthropic 的 Bedrock 模型在运行时会强制设为 `cacheRetention: "none"`。

### OpenRouter Anthropic 模型

对于 `openrouter/anthropic/*` 模型引用，OpenClaw 会在系统/开发者提示词块中注入 Anthropic `cache_control`，以提高提示词缓存复用率。

### 其他提供商

如果提供商不支持此缓存模式，`cacheRetention` 将不起作用。

## 调优模式

### 混合流量（推荐默认配置）

在主智能体上保持长期基线缓存，对突发型通知智能体禁用缓存：

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"
  list:
    - id: "research"
      default: true
      heartbeat:
        every: "55m"
    - id: "alerts"
      params:
        cacheRetention: "none"
```

### 成本优先基线

- 将基线设为 `cacheRetention: "short"`。
- 启用 `contextPruning.mode: "cache-ttl"`。
- 仅对受益于热缓存的智能体将心跳频率保持在 TTL 以内。

## 缓存诊断

OpenClaw 为嵌入式智能体运行提供专门的缓存追踪诊断功能。

### `diagnostics.cacheTrace` 配置

```yaml
diagnostics:
  cacheTrace:
    enabled: true
    filePath: "~/.openclaw/logs/cache-trace.jsonl" # 可选
    includeMessages: false # 默认 true
    includePrompt: false # 默认 true
    includeSystem: false # 默认 true
```

默认值：

- `filePath`：`$OPENCLAW_STATE_DIR/logs/cache-trace.jsonl`
- `includeMessages`：`true`
- `includePrompt`：`true`
- `includeSystem`：`true`

### 环境变量开关（临时调试）

- `OPENCLAW_CACHE_TRACE=1` 启用缓存追踪。
- `OPENCLAW_CACHE_TRACE_FILE=/path/to/cache-trace.jsonl` 覆盖输出路径。
- `OPENCLAW_CACHE_TRACE_MESSAGES=0|1` 切换完整消息载荷捕获。
- `OPENCLAW_CACHE_TRACE_PROMPT=0|1` 切换提示词文本捕获。
- `OPENCLAW_CACHE_TRACE_SYSTEM=0|1` 切换系统提示词捕获。

### 检查内容

- 缓存追踪事件为 JSONL 格式，包含分阶段快照，如 `session:loaded`、`prompt:before`、`stream:context` 和 `session:after`。
- 每轮缓存令牌的影响可通过 `cacheRead` 和 `cacheWrite` 在常规使用界面中查看（例如 `/usage full` 和会话使用摘要）。

## 快速故障排查

- 大多数轮次 `cacheWrite` 较高：检查系统提示词是否包含易变输入，并确认模型/提供商支持您的缓存设置。
- `cacheRetention` 不起作用：确认模型键与 `agents.defaults.models["provider/model"]` 匹配。
- 带有缓存设置的 Bedrock Nova/Mistral 请求：预期运行时会强制设为 `none`。

相关文档：

- [Anthropic](/providers/anthropic)
- [令牌使用与成本](/reference/token-use)
- [会话剪枝](/concepts/session-pruning)
- [Gateway 配置参考](/gateway/configuration-reference)
