---
read_when:
  - 你想使用 Tavily 支持的网络搜索
  - 你需要 Tavily API 密钥
  - 你想将 Tavily 作为 web_search 提供商
  - 你想从 URL 提取内容
summary: Tavily 搜索和提取工具
title: Tavily
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: db530cc101dc930611e4ca54e3d5972140f116bfe168adc939dc5752322d205e
  source_path: tools/tavily.md
  workflow: 15
---

# Tavily

OpenClaw 可以以两种方式使用 **Tavily**：

- 作为 `web_search` 提供商
- 作为显式插件工具：`tavily_search` 和 `tavily_extract`

Tavily 是专为 AI 应用设计的搜索 API，返回针对 LLM 消费优化的结构化结果。它支持可配置的搜索深度、主题过滤、域名过滤、AI 生成的答案摘要以及从 URL 提取内容（包括 JavaScript 渲染的页面）。

## 获取 API 密钥

1. 在 [tavily.com](https://tavily.com/) 创建 Tavily 账户。
2. 在控制台中生成 API 密钥。
3. 将其存储到配置中，或在 Gateway 网关环境中设置 `TAVILY_API_KEY`。

## 配置 Tavily 搜索

```json5
{
  plugins: {
    entries: {
      tavily: {
        enabled: true,
        config: {
          webSearch: {
            apiKey: "tvly-...", // 如果已设置 TAVILY_API_KEY 则可选
            baseUrl: "https://api.tavily.com",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "tavily",
      },
    },
  },
}
```

说明：

- 在新手引导或 `openclaw configure --section web` 中选择 Tavily 会自动启用内置 Tavily 插件。
- 将 Tavily 配置存储在 `plugins.entries.tavily.config.webSearch.*` 下。
- 使用 Tavily 的 `web_search` 支持 `query` 和 `count`（最多 20 条结果）。
- 对于 Tavily 专用控制（如 `search_depth`、`topic`、`include_answer` 或域名过滤器），请使用 `tavily_search`。

## Tavily 插件工具

### `tavily_search`

当你需要 Tavily 专用搜索控制（而非通用 `web_search`）时使用此工具。

| 参数              | 描述                                                                 |
| ----------------- | -------------------------------------------------------------------- |
| `query`           | 搜索查询字符串（保持在 400 字符以内）                               |
| `search_depth`    | `basic`（默认，均衡）或 `advanced`（最高相关性，较慢）              |
| `topic`           | `general`（默认）、`news`（实时更新）或 `finance`                   |
| `max_results`     | 结果数量，1-20（默认：5）                                            |
| `include_answer`  | 包含 AI 生成的答案摘要（默认：false）                               |
| `time_range`      | 按时效过滤：`day`、`week`、`month` 或 `year`                        |
| `include_domains` | 限制结果范围的域名数组                                               |
| `exclude_domains` | 从结果中排除的域名数组                                               |

**搜索深度：**

| 深度       | 速度 | 相关性 | 适用场景                         |
| ---------- | ---- | ------ | -------------------------------- |
| `basic`    | 较快 | 高     | 通用查询（默认）                 |
| `advanced` | 较慢 | 最高   | 精确查询、特定事实、研究         |

### `tavily_extract`

使用此工具从一个或多个 URL 提取干净的内容。可处理 JavaScript 渲染的页面，并支持针对目标提取的查询专注分块。

| 参数                | 描述                                                   |
| ------------------- | ------------------------------------------------------ |
| `urls`              | 要提取的 URL 数组（每次请求 1-20 个）                  |
| `query`             | 按与此查询的相关性对提取的分块重新排序                 |
| `extract_depth`     | `basic`（默认，快速）或 `advanced`（适用于重 JS 页面） |
| `chunks_per_source` | 每个 URL 的分块数，1-5（需要 `query`）                 |
| `include_images`    | 在结果中包含图像 URL（默认：false）                    |

**提取深度：**

| 深度       | 使用场景                           |
| ---------- | ---------------------------------- |
| `basic`    | 简单页面——先试这个                 |
| `advanced` | JS 渲染的 SPA、动态内容、表格      |

技巧：

- 每次请求最多 20 个 URL。将较大的列表分批为多次调用。
- 使用 `query` + `chunks_per_source` 仅获取相关内容，而非完整页面。
- 先试 `basic`；如果内容缺失或不完整，回退到 `advanced`。

## 选择合适的工具

| 需求                               | 工具             |
| ---------------------------------- | ---------------- |
| 快速网络搜索，无特殊选项           | `web_search`     |
| 带深度、主题、AI 答案的搜索        | `tavily_search`  |
| 从特定 URL 提取内容                | `tavily_extract` |

## 相关内容

- [Web 搜索概览](/tools/web) — 所有提供商和自动检测
- [Firecrawl](/tools/firecrawl) — 带内容提取的搜索 + 爬取
- [Exa 搜索](/tools/exa-search) — 带内容提取的神经搜索
