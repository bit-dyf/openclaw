---
read_when:
  - 你想就当前会话提一个快速旁问
  - 你正在跨客户端实现或调试 BTW 行为
summary: 使用 /btw 发起临时旁问
title: BTW 旁问
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: aeef33ba19eb0561693fecea9dd39d6922df93be0b9a89446ed17277bcee58aa
  source_path: tools/btw.md
  workflow: 15
---

# BTW 旁问

`/btw` 让你就**当前会话**提一个快速旁问，而不会将该问题变为正常的对话历史。

它模仿了 Claude Code 的 `/btw` 行为，但经过调整以适配 OpenClaw 的 Gateway 网关和多渠道架构。

## 功能

当你发送：

```text
/btw what changed?
```

OpenClaw 会：

1. 对当前会话上下文进行快照，
2. 运行一个独立的**无工具**模型调用，
3. 仅回答旁问，
4. 不干扰主运行，
5. **不**将 BTW 问题或答案写入会话历史，
6. 将答案作为**实时旁结果**而非普通助手消息发出。

重要的心智模型是：

- 使用相同的会话上下文
- 独立的一次性旁查询
- 无工具调用
- 不污染未来上下文
- 不持久化到转录记录

## 不会做的事情

`/btw` **不会**：

- 创建新的持久会话，
- 继续未完成的主任务，
- 运行工具或智能体工具循环，
- 将 BTW 问题/答案数据写入转录历史，
- 出现在 `chat.history` 中，
- 在重载后保留。

它有意设计为**临时的**。

## 上下文工作原理

BTW 仅将当前会话用作**背景上下文**。

如果主运行当前处于活跃状态，OpenClaw 会对当前消息状态进行快照，并将正在进行的主提示作为背景上下文包含在内，同时明确告知模型：

- 仅回答旁问，
- 不恢复或完成未完成的主任务，
- 不发出工具调用或伪工具调用。

这样可以让 BTW 与主运行隔离，同时仍使其了解会话内容。

## 传递模型

BTW **不**作为普通助手转录消息传递。

在 Gateway 网关协议层：

- 普通助手聊天使用 `chat` 事件
- BTW 使用 `chat.side_result` 事件

这种分离是有意为之的。如果 BTW 复用普通 `chat` 事件路径，客户端会将其视为常规对话历史。

由于 BTW 使用独立的实时事件且不从 `chat.history` 回放，它在重载后消失。

## 界面行为

### TUI

在 TUI 中，BTW 以内联方式渲染在当前会话视图中，但仍为临时性的：

- 在视觉上与普通助手回复有所区别
- 可用 `Enter` 或 `Esc` 关闭
- 重载时不回放

### 外部渠道

在 Telegram、WhatsApp 和 Discord 等渠道上，BTW 作为明确标注的一次性回复传递，因为这些界面没有本地临时覆盖层的概念。

答案仍被视为旁结果，而非正常会话历史。

### 控制界面 / Web

Gateway 网关正确地将 BTW 作为 `chat.side_result` 发出，且 BTW 不包含在 `chat.history` 中，因此 Web 的持久性合约已经正确。

当前控制界面仍需专用的 `chat.side_result` 消费者才能在浏览器中实时渲染 BTW。在该客户端支持落地之前，BTW 是一个具有完整 TUI 和外部渠道行为的 Gateway 网关级功能，但尚未完成浏览器 UX。

## 何时使用 BTW

在以下情况使用 `/btw`：

- 需要对当前工作进行快速澄清，
- 在长时间运行仍在进行时需要一个事实性旁答案，
- 需要一个不应成为未来会话上下文的临时答案。

示例：

```text
/btw what file are we editing?
/btw what does this error mean?
/btw summarize the current task in one sentence
/btw what is 17 * 19?
```

## 何时不用 BTW

当你希望答案成为会话未来工作上下文的一部分时，**不要**使用 `/btw`。

这种情况下，请在主会话中正常提问，而不要使用 BTW。

## 相关内容

- [斜杠命令](/tools/slash-commands)
- [思维层级](/tools/thinking)
- [会话](/concepts/session)
