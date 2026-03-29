---
read_when:
  - 你想将 Brave Search 用于 web_search
  - 你需要 BRAVE_API_KEY 或计划详情
summary: 为 web_search 设置 Brave Search API
title: Brave Search
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: eac36c13baf5c514b699d56f321f0b4ab6f58adb093ce00ba234005ceb01538c
  source_path: tools/brave-search.md
  workflow: 15
---

# Brave Search API

OpenClaw 支持将 Brave Search API 作为 `web_search` 提供商。

## 获取 API 密钥

1. 在 [https://brave.com/search/api/](https://brave.com/search/api/) 创建 Brave Search API 账户
2. 在控制台中选择 **Search** 计划并生成 API 密钥。
3. 将密钥存储到配置中，或在 Gateway 网关环境中设置 `BRAVE_API_KEY`。

## 配置示例

```json5
{
  plugins: {
    entries: {
      brave: {
        config: {
          webSearch: {
            apiKey: "BRAVE_API_KEY_HERE",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "brave",
        maxResults: 5,
        timeoutSeconds: 30,
      },
    },
  },
}
```

提供商专用的 Brave 搜索设置现在位于 `plugins.entries.brave.config.webSearch.*`。
旧版 `tools.web.search.apiKey` 仍通过兼容性 shim 加载，但不再是规范的配置路径。

## 工具参数

| 参数          | 描述                                                              |
| ------------- | ----------------------------------------------------------------- |
| `query`       | 搜索查询（必填）                                                  |
| `count`       | 返回的结果数量（1-10，默认：5）                                   |
| `country`     | 2 位 ISO 国家代码（例如 "US"、"DE"）                              |
| `language`    | 搜索结果的 ISO 639-1 语言代码（例如 "en"、"de"、"fr"）           |
| `ui_lang`     | UI 元素的 ISO 语言代码                                            |
| `freshness`   | 时间过滤器：`day`（24 小时）、`week`、`month` 或 `year`           |
| `date_after`  | 仅返回此日期后发布的结果（YYYY-MM-DD）                            |
| `date_before` | 仅返回此日期前发布的结果（YYYY-MM-DD）                            |

**示例：**

```javascript
// 国家和语言专项搜索
await web_search({
  query: "renewable energy",
  country: "DE",
  language: "de",
});

// 最近结果（过去一周）
await web_search({
  query: "AI news",
  freshness: "week",
});

// 日期范围搜索
await web_search({
  query: "AI developments",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});
```

## 说明

- OpenClaw 使用 Brave **Search** 计划。如果你有旧版订阅（例如每月 2,000 次查询的原始免费计划），它仍然有效，但不包含 LLM Context 等新功能或更高速率限制。
- 每个 Brave 计划每月包含 **\$5 的免费积分**（循环刷新）。Search 计划每 1,000 次请求收费 \$5，因此积分可覆盖每月 1,000 次查询。在 Brave 控制台中设置使用限额以避免意外收费。查看 [Brave API 门户](https://brave.com/search/api/) 了解当前计划。
- Search 计划包含 LLM Context 端点和 AI 推理权限。将结果存储用于训练或微调模型需要具有明确存储权限的计划。参阅 Brave [服务条款](https://api-dashboard.search.brave.com/terms-of-service)。
- 默认情况下，结果缓存 15 分钟（可通过 `cacheTtlMinutes` 配置）。

## 相关内容

- [Web 搜索概览](/tools/web) — 所有提供商和自动检测
- [Perplexity 搜索](/tools/perplexity-search) — 带域名过滤器的结构化结果
- [Exa 搜索](/tools/exa-search) — 带内容提取的神经搜索
