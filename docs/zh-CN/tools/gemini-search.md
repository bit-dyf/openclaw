---
read_when:
  - 你想将 Gemini 用于 web_search
  - 你需要 GEMINI_API_KEY
  - 你想使用 Google Search Grounding
summary: 使用 Google Search Grounding 的 Gemini 网络搜索
title: Gemini 搜索
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 9c685ae976b7e6c376d779b8b3910e3a990b2638cd34b7d805435b14c838d994
  source_path: tools/gemini-search.md
  workflow: 15
---

# Gemini 搜索

OpenClaw 支持带有内置 [Google Search Grounding](https://ai.google.dev/gemini-api/docs/grounding) 的 Gemini 模型，可返回由实时 Google 搜索结果支持的带引用 AI 合成答案。

## 获取 API 密钥

<Steps>
  <Step title="创建密钥">
    前往 [Google AI Studio](https://aistudio.google.com/apikey) 创建 API 密钥。
  </Step>
  <Step title="存储密钥">
    在 Gateway 网关环境中设置 `GEMINI_API_KEY`，或通过以下方式配置：

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
      google: {
        config: {
          webSearch: {
            apiKey: "AIza...", // 如果已设置 GEMINI_API_KEY 则可选
            model: "gemini-2.5-flash", // 默认
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "gemini",
      },
    },
  },
}
```

**环境变量替代方案：** 在 Gateway 网关环境中设置 `GEMINI_API_KEY`。对于 Gateway 网关安装，将其放入 `~/.openclaw/.env`。

## 工作原理

与返回链接和摘要列表的传统搜索提供商不同，Gemini 使用 Google Search Grounding 生成带内联引用的 AI 合成答案。结果同时包含合成答案和来源 URL。

- 来自 Gemini Grounding 的引用 URL 会自动从 Google 重定向 URL 解析为直接 URL。
- 重定向解析使用 SSRF 防护路径（HEAD + 重定向检查 + http/https 验证），然后返回最终引用 URL。
- 重定向解析使用严格 SSRF 默认值，因此重定向到私有/内部目标的请求会被阻断。

## 支持的参数

Gemini 搜索支持标准的 `query` 和 `count` 参数。
不支持提供商专用的过滤器，如 `country`、`language`、`freshness` 和 `domain_filter`。

## 模型选择

默认模型为 `gemini-2.5-flash`（快速且经济）。任何支持 Grounding 的 Gemini 模型都可以通过 `plugins.entries.google.config.webSearch.model` 使用。

## 相关内容

- [Web 搜索概览](/tools/web) — 所有提供商和自动检测
- [Brave Search](/tools/brave-search) — 带摘要的结构化结果
- [Perplexity 搜索](/tools/perplexity-search) — 结构化结果 + 内容提取
