---
read_when:
  - 你想了解 OpenClaw 如何组装模型上下文
  - 你在切换旧版引擎和插件引擎
  - 你在构建上下文引擎插件
summary: 上下文引擎：可插拔的上下文组装、压缩和子智能体生命周期
title: 上下文引擎
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 0a6c121157092e0452ddcff48c7b48dbb64be7bf638914ad2fb01764a7e44fcb
  source_path: concepts/context-engine.md
  workflow: 15
---

# 上下文引擎

**上下文引擎**控制 OpenClaw 如何为每次运行构建模型上下文。
它决定包含哪些消息、如何总结较旧的历史记录，以及如何跨子智能体边界管理上下文。

OpenClaw 附带内置的 `legacy` 引擎。插件可以注册替代引擎，以替换活跃的上下文引擎生命周期。

## 快速开始

检查当前活跃的引擎：

```bash
openclaw doctor
# 或直接查看配置：
cat ~/.openclaw/openclaw.json | jq '.plugins.slots.contextEngine'
```

### 安装上下文引擎插件

上下文引擎插件的安装方式与其他 OpenClaw 插件相同。先安装，然后在插槽中选择引擎：

```bash
# 从 npm 安装
openclaw plugins install @martian-engineering/lossless-claw

# 或从本地路径安装（用于开发）
openclaw plugins install -l ./my-context-engine
```

然后在配置中启用插件并将其选为活跃引擎：

```json5
// openclaw.json
{
  plugins: {
    slots: {
      contextEngine: "lossless-claw", // 必须与插件注册的引擎 id 匹配
    },
    entries: {
      "lossless-claw": {
        enabled: true,
        // 插件专用配置（参见插件文档）
      },
    },
  },
}
```

安装并配置后重启 Gateway 网关。

要切换回内置引擎，请将 `contextEngine` 设置为 `"legacy"`（或完全删除该键——`"legacy"` 是默认值）。

## 工作原理

每次 OpenClaw 运行模型提示时，上下文引擎在四个生命周期节点参与：

1. **摄入（Ingest）**——新消息添加到会话时调用。引擎可以在自己的数据存储中存储或索引该消息。
2. **组装（Assemble）**——每次模型运行前调用。引擎返回符合 token 预算的有序消息集（以及可选的 `systemPromptAddition`）。
3. **压缩（Compact）**——上下文窗口已满或用户运行 `/compact` 时调用。引擎总结较旧的历史记录以释放空间。
4. **轮次后（After turn）**——运行完成后调用。引擎可以持久化状态、触发后台压缩或更新索引。

### 子智能体生命周期（可选）

OpenClaw 目前调用一个子智能体生命周期钩子：

- **onSubagentEnded**——子智能体会话完成或被清理时清除状态。

`prepareSubagentSpawn` 钩子是接口的一部分，供未来使用，但运行时尚未调用它。

### 系统提示追加

`assemble` 方法可以返回 `systemPromptAddition` 字符串。OpenClaw 将其前置到本次运行的系统提示中。这使引擎能够注入动态召回指引、检索指令或上下文感知提示，而无需静态工作区文件。

## 旧版引擎

内置的 `legacy` 引擎保留了 OpenClaw 的原始行为：

- **摄入**：无操作（会话管理器直接处理消息持久化）。
- **组装**：透传（运行时中现有的清理 → 验证 → 限制流水线处理上下文组装）。
- **压缩**：委托给内置的摘要压缩，该压缩创建较旧消息的单个摘要并保留最近消息不变。
- **轮次后**：无操作。

旧版引擎不注册工具，也不提供 `systemPromptAddition`。

当未设置 `plugins.slots.contextEngine`（或设置为 `"legacy"`）时，自动使用此引擎。

## 配置参考

```json5
{
  plugins: {
    slots: {
      // 选择活跃的上下文引擎。默认："legacy"。
      // 设置为插件 id 以使用插件引擎。
      contextEngine: "legacy",
    },
  },
}
```

## 与压缩和记忆的关系

- **压缩**是上下文引擎的一个职责。旧版引擎委托给 OpenClaw 的内置摘要。插件引擎可以实现任何压缩策略（DAG 摘要、向量检索等）。
- **记忆插件**（`plugins.slots.memory`）与上下文引擎分离。记忆插件提供搜索/检索；上下文引擎控制模型看到的内容。它们可以协同工作——上下文引擎可能在组装期间使用记忆插件数据。
- **会话修剪**（在内存中修剪旧工具结果）无论哪个上下文引擎处于活跃状态都会运行。

## 提示

- 使用 `openclaw doctor` 验证引擎是否正确加载。
- 如果切换引擎，现有会话将继续使用其当前历史记录。新引擎将接管未来的运行。
- 引擎错误会被记录并显示在诊断中。如果插件引擎无法注册或所选引擎 id 无法解析，OpenClaw 不会自动回退；运行将失败，直到你修复插件或将 `plugins.slots.contextEngine` 切换回 `"legacy"`。
- 对于开发，使用 `openclaw plugins install -l ./my-engine` 链接本地插件目录，无需复制。

另请参阅：[压缩](/concepts/compaction)、[上下文](/concepts/context)、[插件](/tools/plugin)、[插件清单](/plugins/manifest)。
