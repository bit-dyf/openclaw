---
read_when:
  - 你想将 Perplexity 搜索用于网络搜索
  - 你需要设置 PERPLEXITY_API_KEY 或 OPENROUTER_API_KEY
summary: 用于 web_search 的 Perplexity Search API 以及 Sonar/OpenRouter 兼容性
title: Perplexity 搜索
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: c63efe9e35c93f16171e1139d6eb11a04203bed5680e3e2c39c181e9cd2924fa
  source_path: tools/perplexity-search.md
  workflow: 15
---

# Perplexity Search API

OpenClaw 支持将 Perplexity Search API 作为 `web_search` 提供商。
它返回包含 `title`、`url` 和 `snippet` 字段的结构化结果。

为了兼容性，OpenClaw 还支持旧版 Perplexity Sonar/OpenRouter 设置。
如果你使用 `OPENROUTER_API_KEY`、在 `plugins.entries.perplexity.config.webSearch.apiKey` 中使用 `sk-or-...` 密钥，或设置 `plugins.entries.perplexity.config.webSearch.baseUrl` / `model`，提供商会切换到聊天补全路径，返回带引用的 AI 合成答案，而非结构化 Search API 结果。

## 获取 Perplexity API 密钥

1. 在 [perplexity.ai/settings/api](https://www.perplexity.ai/settings/api) 创建 Perplexity 账户
2. 在控制台中生成 API 密钥
3. 将密钥存储到配置中，或在 Gateway 网关环境中设置 `PERPLEXITY_API_KEY`。

## OpenRouter 兼容性

如果你已经在通过 OpenRouter 使用 Perplexity Sonar，保留 `provider: "perplexity"` 并在 Gateway 网关环境中设置 `OPENROUTER_API_KEY`，或在 `plugins.entries.perplexity.config.webSearch.apiKey` 中存储 `sk-or-...` 密钥。

可选兼容性控制：

- `plugins.entries.perplexity.config.webSearch.baseUrl`
- `plugins.entries.perplexity.config.webSearch.model`

## 配置示例

### 原生 Perplexity Search API

```json5
{
  plugins: {
    entries: {
      perplexity: {
        config: {
          webSearch: {
            apiKey: "pplx-...",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "perplexity",
      },
    },
  },
}
```

### OpenRouter / Sonar 兼容

```json5
{
  plugins: {
    entries: {
      perplexity: {
        config: {
          webSearch: {
            apiKey: "<openrouter-api-key>",
            baseUrl: "https://openrouter.ai/api/v1",
            model: "perplexity/sonar-pro",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "perplexity",
      },
    },
  },
}
```

## 密钥存储位置

**通过配置：** 运行 `openclaw configure --section web`。它将密钥存储在 `~/.openclaw/openclaw.json` 的 `plugins.entries.perplexity.config.webSearch.apiKey` 下。该字段也接受 SecretRef 对象。

**通过环境变量：** 在 Gateway 网关进程环境中设置 `PERPLEXITY_API_KEY` 或 `OPENROUTER_API_KEY`。对于 Gateway 网关安装，将其放入 `~/.openclaw/.env`（或你的服务环境）。参阅[环境变量](/help/faq#env-vars-and-env-loading)。

如果配置了 `provider: "perplexity"` 且 Perplexity 密钥 SecretRef 无法解析且无环境变量回退，启动/重载会快速失败。

## 工具参数

这些参数适用于原生 Perplexity Search API 路径。

| 参数                  | 描述                                                  |
| --------------------- | ----------------------------------------------------- |
| `query`               | 搜索查询（必填）                                      |
| `count`               | 返回结果数量（1-10，默认：5）                         |
| `country`             | 2 位 ISO 国家代码（如 "US"、"DE"）                    |
| `language`            | ISO 639-1 语言代码（如 "en"、"de"、"fr"）             |
| `freshness`           | 时间过滤器：`day`（24 小时）、`week`、`month` 或 `year` |
| `date_after`          | 仅返回此日期后发布的结果（YYYY-MM-DD）                |
| `date_before`         | 仅返回此日期前发布的结果（YYYY-MM-DD）                |
| `domain_filter`       | 域名允许列表/拒绝列表数组（最多 20 个）               |
| `max_tokens`          | 总内容预算（默认：25000，最大：1000000）               |
| `max_tokens_per_page` | 每页 token 限制（默认：2048）                         |

对于旧版 Sonar/OpenRouter 兼容路径，仅支持 `query` 和 `freshness`。
仅 Search API 支持的过滤器（如 `country`、`language`、`date_after`、`date_before`、`domain_filter`、`max_tokens` 和 `max_tokens_per_page`）会返回明确的错误。

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

// 域名过滤（允许列表）
await web_search({
  query: "climate research",
  domain_filter: ["nature.com", "science.org", ".edu"],
});

// 域名过滤（拒绝列表——用 - 前缀）
await web_search({
  query: "product reviews",
  domain_filter: ["-reddit.com", "-pinterest.com"],
});

// 更多内容提取
await web_search({
  query: "detailed AI research",
  max_tokens: 50000,
  max_tokens_per_page: 4096,
});
```

### 域名过滤规则

- 每个过滤器最多 20 个域名
- 同一请求中不能混用允许列表和拒绝列表
- 拒绝列表条目使用 `-` 前缀（如 `["-reddit.com"]`）

## 说明

- Perplexity Search API 返回结构化网络搜索结果（`title`、`url`、`snippet`）
- OpenRouter 或显式设置 `plugins.entries.perplexity.config.webSearch.baseUrl` / `model` 会将 Perplexity 切换回 Sonar 聊天补全以实现兼容性
- 结果默认缓存 15 分钟（可通过 `cacheTtlMinutes` 配置）

## 相关内容

- [Web 搜索概览](/tools/web) — 所有提供商和自动检测
- [Perplexity Search API 文档](https://docs.perplexity.ai/docs/search/quickstart) — Perplexity 官方文档
- [Brave Search](/tools/brave-search) — 带国家/语言过滤器的结构化结果
- [Exa 搜索](/tools/exa-search) — 带内容提取的神经搜索
