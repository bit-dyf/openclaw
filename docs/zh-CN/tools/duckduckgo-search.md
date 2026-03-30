---
read_when:
  - 你想使用无需 API 密钥的网络搜索提供商
  - 你想将 DuckDuckGo 用于 web_search
  - 你需要零配置搜索回退
summary: DuckDuckGo 网络搜索 — 无需密钥的回退提供商（实验性，基于 HTML）
title: DuckDuckGo 搜索
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 4170a499f44ab411493619632e8f7982bcad3331176aaecacdaa817ba426c85a
  source_path: tools/duckduckgo-search.md
  workflow: 15
---

# DuckDuckGo 搜索

OpenClaw 支持将 DuckDuckGo 作为**无需密钥**的 `web_search` 提供商。无需 API 密钥或账户。

<Warning>
  DuckDuckGo 是一个**实验性的非官方**集成，它从 DuckDuckGo 的非 JavaScript 搜索页面抓取结果——而非官方 API。预计可能会因机器人拦截页面或 HTML 结构变化而偶发故障。
</Warning>

## 设置

无需 API 密钥——只需将 DuckDuckGo 设置为你的提供商：

<Steps>
  <Step title="配置">
    ```bash
    openclaw configure --section web
    # 选择 "duckduckgo" 作为提供商
    ```
  </Step>
</Steps>

## 配置

```json5
{
  tools: {
    web: {
      search: {
        provider: "duckduckgo",
      },
    },
  },
}
```

可选的插件级设置（区域和安全搜索）：

```json5
{
  plugins: {
    entries: {
      duckduckgo: {
        config: {
          webSearch: {
            region: "us-en", // DuckDuckGo 区域代码
            safeSearch: "moderate", // "strict"、"moderate" 或 "off"
          },
        },
      },
    },
  },
}
```

## 工具参数

| 参数         | 描述                                                             |
| ------------ | ---------------------------------------------------------------- |
| `query`      | 搜索查询（必填）                                                 |
| `count`      | 返回结果数量（1-10，默认：5）                                    |
| `region`     | DuckDuckGo 区域代码（如 `us-en`、`uk-en`、`de-de`）             |
| `safeSearch` | 安全搜索级别：`strict`、`moderate`（默认）或 `off`              |

区域和安全搜索也可在插件配置中设置（见上文）——工具参数会覆盖每次查询的配置值。

## 说明

- **无需 API 密钥** — 开箱即用，零配置
- **实验性** — 从 DuckDuckGo 的非 JavaScript HTML 搜索页面获取结果，非官方 API 或 SDK
- **机器人拦截风险** — DuckDuckGo 可能在大量或自动化使用时提供验证码或阻断请求
- **HTML 解析** — 结果依赖页面结构，可能随时发生变化
- **自动检测顺序** — DuckDuckGo 在自动检测中排在最后（顺序 100），因此任何有密钥的 API 支持提供商优先
- **安全搜索默认为 moderate**（未配置时）

<Tip>
  在生产环境中，建议使用 [Brave Search](/tools/brave-search)（提供免费层）或其他 API 支持的提供商。
</Tip>

## 相关内容

- [Web 搜索概览](/tools/web) — 所有提供商和自动检测
- [Brave Search](/tools/brave-search) — 带免费层的结构化结果
- [Exa 搜索](/tools/exa-search) — 带内容提取的神经搜索
