---
read_when:
  - 你想将 Grok 用于 web_search
  - 你需要用于网络搜索的 XAI_API_KEY
summary: 通过 xAI 网络接地响应实现 Grok 网络搜索
title: Grok 搜索
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 4653accf20ac0f3cf4486d184560b01783bbd58a201501870cca9947b358b7cd
  source_path: tools/grok-search.md
  workflow: 15
---

# Grok 搜索

OpenClaw 支持将 Grok 作为 `web_search` 提供商，使用 xAI 网络接地响应生成由实时搜索结果支持的带引用 AI 合成答案。

同一个 `XAI_API_KEY` 还可以驱动内置的 `x_search` 工具，用于 X（前 Twitter）帖子搜索。如果你在 `plugins.entries.xai.config.webSearch.apiKey` 下存储密钥，OpenClaw 现在也会将其作为内置 xAI 模型提供商的回退使用。

对于帖子级别的 X 指标（如转发、回复、书签或浏览量），建议使用精确帖子 URL 或状态 ID 的 `x_search`，而非宽泛的搜索查询。

## 新手引导与配置

如果你在以下操作中选择 **Grok**：

- `openclaw onboard`
- `openclaw configure --section web`

OpenClaw 可以显示一个单独的后续步骤，使用同一个 `XAI_API_KEY` 启用 `x_search`。该后续步骤：

- 仅在你为 `web_search` 选择 Grok 后显示
- 不是独立的顶级网络搜索提供商选项
- 可在同一流程中选择性地设置 `x_search` 模型

如果你跳过它，可以在之后的配置中启用或更改 `x_search`。

## 获取 API 密钥

<Steps>
  <Step title="创建密钥">
    从 [xAI](https://console.x.ai/) 获取 API 密钥。
  </Step>
  <Step title="存储密钥">
    在 Gateway 网关环境中设置 `XAI_API_KEY`，或通过以下方式配置：

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

## 配置

```json5
{
  plugins: {
    entries: {
      xai: {
        config: {
          webSearch: {
            apiKey: "xai-...", // 如果已设置 XAI_API_KEY 则可选
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "grok",
      },
    },
  },
}
```

**环境变量替代方案：** 在 Gateway 网关环境中设置 `XAI_API_KEY`。对于 Gateway 网关安装，将其放入 `~/.openclaw/.env`。

## 工作原理

Grok 使用 xAI 网络接地响应合成带内联引用的答案，类似于 Gemini 的 Google Search Grounding 方式。

## 支持的参数

Grok 搜索支持标准的 `query` 和 `count` 参数。
目前不支持提供商专用的过滤器。

## 相关内容

- [Web 搜索概览](/tools/web) — 所有提供商和自动检测
- [Web 搜索中的 x_search](/tools/web#x_search) — 通过 xAI 实现一级 X 搜索
- [Gemini 搜索](/tools/gemini-search) — 通过 Google Grounding 实现的 AI 合成答案
