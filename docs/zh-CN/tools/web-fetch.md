---
read_when:
  - 你想获取一个 URL 并提取可读内容
  - 你需要配置 web_fetch 或其 Firecrawl 回退
  - 你想了解 web_fetch 的限制和缓存
sidebarTitle: Web Fetch
summary: web_fetch 工具 — 带可读内容提取的 HTTP 获取
title: Web Fetch
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: cfedb24604d6a8b24683bdbd6f5f0ab0f8aa2225c67fe974d2dc5c66aa914e88
  source_path: tools/web-fetch.md
  workflow: 15
---

# Web Fetch

`web_fetch` 工具执行简单的 HTTP GET 请求并提取可读内容（HTML 转 markdown 或文本）。它**不**执行 JavaScript。

对于 JS 密集型网站或需要登录的页面，请改用 [Web 浏览器](/tools/browser)。

## 快速开始

`web_fetch` **默认启用**——无需任何配置。智能体可以立即调用它：

```javascript
await web_fetch({ url: "https://example.com/article" });
```

## 工具参数

| 参数          | 类型     | 描述                                         |
| ------------- | -------- | -------------------------------------------- |
| `url`         | `string` | 要获取的 URL（必填，仅支持 http/https）       |
| `extractMode` | `string` | `"markdown"`（默认）或 `"text"`              |
| `maxChars`    | `number` | 将输出截断到此字符数                         |

## 工作原理

<Steps>
  <Step title="获取">
    发送带有类似 Chrome 的 User-Agent 和 `Accept-Language` 请求头的 HTTP GET 请求。阻断私有/内部主机名并重新检查重定向。
  </Step>
  <Step title="提取">
    对 HTML 响应运行 Readability（主内容提取）。
  </Step>
  <Step title="回退（可选）">
    如果 Readability 失败且已配置 Firecrawl，则通过 Firecrawl API 以绕过机器人检测模式重试。
  </Step>
  <Step title="缓存">
    结果缓存 15 分钟（可配置），以减少对同一 URL 的重复获取。
  </Step>
</Steps>

## 配置

```json5
{
  tools: {
    web: {
      fetch: {
        enabled: true, // 默认：true
        maxChars: 50000, // 最大输出字符数
        maxCharsCap: 50000, // maxChars 参数的硬性上限
        maxResponseBytes: 2000000, // 截断前的最大下载大小
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        readability: true, // 使用 Readability 提取
        userAgent: "Mozilla/5.0 ...", // 覆盖 User-Agent
      },
    },
  },
}
```

## Firecrawl 回退

如果 Readability 提取失败，`web_fetch` 可以回退到 [Firecrawl](/tools/firecrawl) 进行机器人绕过和更好的提取：

```json5
{
  tools: {
    web: {
      fetch: {
        firecrawl: {
          enabled: true,
          apiKey: "fc-...", // 如果已设置 FIRECRAWL_API_KEY 则可选
          baseUrl: "https://api.firecrawl.dev",
          onlyMainContent: true,
          maxAgeMs: 86400000, // 缓存时长（1 天）
          timeoutSeconds: 60,
        },
      },
    },
  },
}
```

`tools.web.fetch.firecrawl.apiKey` 支持 SecretRef 对象。

<Note>
  如果启用了 Firecrawl 且其 SecretRef 无法解析且无 `FIRECRAWL_API_KEY` 环境变量回退，Gateway 网关启动会快速失败。
</Note>

## 限制与安全

- `maxChars` 限制在 `tools.web.fetch.maxCharsCap` 以内
- 响应体在解析前限制在 `maxResponseBytes` 以内；超大响应会被截断并附带警告
- 私有/内部主机名被阻断
- 重定向经过检查并受 `maxRedirects` 限制
- `web_fetch` 是尽力而为的——某些网站需要使用 [Web 浏览器](/tools/browser)

## 工具配置文件

如果你使用工具配置文件或允许列表，请添加 `web_fetch` 或 `group:web`：

```json5
{
  tools: {
    allow: ["web_fetch"],
    // 或者：allow: ["group:web"]（同时包含 web_fetch 和 web_search）
  },
}
```

## 相关内容

- [Web 搜索](/tools/web) — 使用多个提供商搜索网络
- [Web 浏览器](/tools/browser) — 针对 JS 密集型网站的完整浏览器自动化
- [Firecrawl](/tools/firecrawl) — Firecrawl 搜索和爬取工具
