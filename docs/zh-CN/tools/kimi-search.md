---
read_when:
  - 你想将 Kimi 用于 web_search
  - 你需要 KIMI_API_KEY 或 MOONSHOT_API_KEY
summary: 通过 Moonshot 网络搜索实现 Kimi 网络搜索
title: Kimi 搜索
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: dd0f9128f3b9ab6ce22521d4b30ef58a7fe8e9ff26afacf7f7cf23f0f1f47e84
  source_path: tools/kimi-search.md
  workflow: 15
---

# Kimi 搜索

OpenClaw 支持将 Kimi 作为 `web_search` 提供商，使用 Moonshot 网络搜索生成带引用的 AI 合成答案。

## 获取 API 密钥

<Steps>
  <Step title="创建密钥">
    从 [Moonshot AI](https://platform.moonshot.cn/) 获取 API 密钥。
  </Step>
  <Step title="存储密钥">
    在 Gateway 网关环境中设置 `KIMI_API_KEY` 或 `MOONSHOT_API_KEY`，或通过以下方式配置：

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
      moonshot: {
        config: {
          webSearch: {
            apiKey: "sk-...", // 如果已设置 KIMI_API_KEY 或 MOONSHOT_API_KEY 则可选
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "kimi",
      },
    },
  },
}
```

**环境变量替代方案：** 在 Gateway 网关环境中设置 `KIMI_API_KEY` 或 `MOONSHOT_API_KEY`。对于 Gateway 网关安装，将其放入 `~/.openclaw/.env`。

## 工作原理

Kimi 使用 Moonshot 网络搜索合成带内联引用的答案，类似于 Gemini 和 Grok 的接地响应方式。

## 支持的参数

Kimi 搜索支持标准的 `query` 和 `count` 参数。
目前不支持提供商专用的过滤器。

## 相关内容

- [Web 搜索概览](/tools/web) — 所有提供商和自动检测
- [Gemini 搜索](/tools/gemini-search) — 通过 Google Grounding 实现的 AI 合成答案
- [Grok 搜索](/tools/grok-search) — 通过 xAI Grounding 实现的 AI 合成答案
