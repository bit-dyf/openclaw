---
read_when:
  - 你想在 OpenClaw 中使用 Grok 模型
  - 你正在配置 xAI 认证或模型 ID
summary: 在 OpenClaw 中使用 xAI Grok 模型
title: xAI
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 946ed2b766e2195e87e322a6f58ca6ea376ee1a44c9e016347c95c14203a22da
  source_path: providers/xai.md
  workflow: 15
---

# xAI

OpenClaw 内置了用于 Grok 模型的 `xai` 提供商插件。

## 设置

1. 在 xAI 控制台中创建 API 密钥。
2. 设置 `XAI_API_KEY`，或运行：

```bash
openclaw onboard --auth-choice xai-api-key
```

3. 选择一个模型，例如：

```json5
{
  agents: { defaults: { model: { primary: "xai/grok-4" } } },
}
```

OpenClaw 现在使用 xAI Responses API 作为内置的 xAI 传输。同一个 `XAI_API_KEY` 还可以驱动 Grok 支持的 `web_search`、一级 `x_search` 和远程 `code_execution`。
如果你在 `plugins.entries.xai.config.webSearch.apiKey` 下存储了 xAI 密钥，内置的 xAI 模型提供商现在也会将该密钥作为回退使用。
`code_execution` 调优配置位于 `plugins.entries.xai.config.codeExecution`。

## 当前内置模型目录

OpenClaw 现在内置了以下 xAI 模型系列：

- `grok-4`、`grok-4-0709`
- `grok-4-fast-reasoning`、`grok-4-fast-non-reasoning`
- `grok-4-1-fast-reasoning`、`grok-4-1-fast-non-reasoning`
- `grok-4.20-reasoning`、`grok-4.20-non-reasoning`
- `grok-code-fast-1`

当新的 `grok-4*` 和 `grok-code-fast*` ID 遵循相同 API 格式时，插件也会向前解析这些 ID。

## 网络搜索

内置的 `grok` 网络搜索提供商也使用 `XAI_API_KEY`：

```bash
openclaw config set tools.web.search.provider grok
```

## 已知限制

- 目前仅支持 API 密钥认证。OpenClaw 中尚无 xAI OAuth / 设备码流程。
- `grok-4.20-multi-agent-experimental-beta-0304` 不支持普通 xAI 提供商路径，因为它需要与标准 OpenClaw xAI 传输不同的上游 API 界面。

## 说明

- OpenClaw 在共享运行器路径上自动应用 xAI 专用的工具 schema 和工具调用兼容性修复。
- `web_search`、`x_search` 和 `code_execution` 作为 OpenClaw 工具暴露。OpenClaw 在每个工具请求中启用所需的特定 xAI 内置功能，而不是在每次聊天轮次中附加所有原生工具。
- `x_search` 和 `code_execution` 属于内置的 xAI 插件，而非硬编码到核心模型运行时中。
- `code_execution` 是远程 xAI 沙箱执行，不是本地 [`exec`](/tools/exec)。
- 有关提供商的整体概述，请参阅[模型提供商](/providers/index)。
