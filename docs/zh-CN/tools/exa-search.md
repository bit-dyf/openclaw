---
read_when:
  - 你想将 Exa 用于 web_search
  - 你需要 EXA_API_KEY
  - 你想使用神经搜索或内容提取
summary: Exa AI 搜索 — 带内容提取的神经搜索和关键词搜索
title: Exa 搜索
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 307b727b4fb88756cac51c17ffd73468ca695c4481692e03d0b4a9969982a2a8
  source_path: tools/exa-search.md
  workflow: 15
---

# Exa 搜索

OpenClaw 支持将 [Exa AI](https://exa.ai/) 作为 `web_search` 提供商。Exa 提供神经搜索、关键词搜索和混合搜索模式，并内置内容提取（摘要句、文本、摘要）。

## 获取 API 密钥

<Steps>
  <Step title="创建账户">
    在 [exa.ai](https://exa.ai/) 注册账户，并从控制台生成 API 密钥。
  </Step>
  <Step title="存储密钥">
    在 Gateway 网关环境中设置 `EXA_API_KEY`，或通过以下方式配置：

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
      exa: {
        config: {
          webSearch: {
            apiKey: "exa-...", // 如果已设置 EXA_API_KEY 则可选
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "exa",
      },
    },
  },
}
```

**环境变量替代方案：** 在 Gateway 网关环境中设置 `EXA_API_KEY`。对于 Gateway 网关安装，将其放入 `~/.openclaw/.env`。

## 工具参数

| 参数          | 描述                                                                              |
| ------------- | --------------------------------------------------------------------------------- |
| `query`       | 搜索查询（必填）                                                                  |
| `count`       | 返回结果数量（1-100）                                                             |
| `type`        | 搜索模式：`auto`、`neural`、`fast`、`deep`、`deep-reasoning` 或 `instant`         |
| `freshness`   | 时间过滤器：`day`、`week`、`month` 或 `year`                                      |
| `date_after`  | 此日期之后的结果（YYYY-MM-DD）                                                    |
| `date_before` | 此日期之前的结果（YYYY-MM-DD）                                                    |
| `contents`    | 内容提取选项（见下文）                                                            |

### 内容提取

Exa 可以随搜索结果一起返回提取的内容。传入 `contents` 对象来启用：

```javascript
await web_search({
  query: "transformer architecture explained",
  type: "neural",
  contents: {
    text: true, // 完整页面文本
    highlights: { numSentences: 3 }, // 关键句子
    summary: true, // AI 摘要
  },
});
```

| 内容选项     | 类型                                                                  | 描述           |
| ------------ | --------------------------------------------------------------------- | -------------- |
| `text`       | `boolean \| { maxCharacters }`                                        | 提取完整页面文本 |
| `highlights` | `boolean \| { maxCharacters, query, numSentences, highlightsPerUrl }` | 提取关键句子   |
| `summary`    | `boolean \| { query }`                                                | AI 生成摘要    |

### 搜索模式

| 模式             | 描述                           |
| ---------------- | ------------------------------ |
| `auto`           | Exa 选择最佳模式（默认）       |
| `neural`         | 基于语义/含义的搜索            |
| `fast`           | 快速关键词搜索                 |
| `deep`           | 深度彻底搜索                   |
| `deep-reasoning` | 带推理的深度搜索               |
| `instant`        | 最快速结果                     |

## 说明

- 如果未提供 `contents` 选项，Exa 默认使用 `{ highlights: true }`，使结果包含关键句子摘录
- 结果在可用时保留来自 Exa API 响应的 `highlightScores` 和 `summary` 字段
- 结果描述依次从摘要句、摘要、完整文本中解析——以可用的为准
- `freshness` 和 `date_after` / `date_before` 不能同时使用——选择一种时间过滤模式
- 每次查询最多可返回 100 条结果（取决于 Exa 搜索类型限制）
- 结果默认缓存 15 分钟（可通过 `cacheTtlMinutes` 配置）
- Exa 是带结构化 JSON 响应的官方 API 集成

## 相关内容

- [Web 搜索概览](/tools/web) — 所有提供商和自动检测
- [Brave Search](/tools/brave-search) — 带国家/语言过滤器的结构化结果
- [Perplexity 搜索](/tools/perplexity-search) — 带域名过滤器的结构化结果
