---
read_when:
  - 你想让智能体以差异对比的形式展示代码或 Markdown 编辑
  - 你想要画布就绪的查看器 URL 或渲染的差异文件
  - 你需要具有安全默认值的受控临时差异构件
summary: 智能体的只读差异查看器和文件渲染器（可选插件工具）
title: 差异对比
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: a9f0c2da0fe2729a7267a83037101901879a9687da58ad05cba6026145d54ae0
  source_path: tools/diffs.md
  workflow: 15
---

# 差异对比

`diffs` 是一个带有简短内置系统指引的可选插件工具，以及一个配套技能，可将变更内容转换为智能体的只读差异构件。

它接受以下任一输入：

- `before`（前）和 `after`（后）文本
- 统一格式的 `patch`（补丁）

它可以返回：

- 用于画布展示的 Gateway 网关查看器 URL
- 用于消息传递的渲染文件路径（PNG 或 PDF）
- 单次调用同时返回两种输出

启用后，插件会将简洁的使用指引前置到系统提示空间中，并为智能体需要更完整说明的情况暴露详细技能。

## 快速开始

1. 启用插件。
2. 对于画布优先流程，以 `mode: "view"` 调用 `diffs`。
3. 对于聊天文件传递流程，以 `mode: "file"` 调用 `diffs`。
4. 同时需要两种构件时，以 `mode: "both"` 调用 `diffs`。

## 启用插件

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
      },
    },
  },
}
```

## 禁用内置系统指引

如果你想保持 `diffs` 工具启用但禁用其内置系统提示指引，请将 `plugins.entries.diffs.hooks.allowPromptInjection` 设置为 `false`：

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
      },
    },
  },
}
```

这会阻断 diffs 插件的 `before_prompt_build` 钩子，同时保留插件、工具和配套技能可用。

如果你想同时禁用指引和工具，请禁用插件本身。

## 典型智能体工作流

1. 智能体调用 `diffs`。
2. 智能体读取 `details` 字段。
3. 智能体执行以下任一操作：
   - 使用 `canvas present` 打开 `details.viewerUrl`
   - 使用 `message` 的 `path` 或 `filePath` 发送 `details.filePath`
   - 同时执行两者

## 输入示例

前/后模式：

```json
{
  "before": "# Hello\n\nOne",
  "after": "# Hello\n\nTwo",
  "path": "docs/example.md",
  "mode": "view"
}
```

补丁模式：

```json
{
  "patch": "diff --git a/src/example.ts b/src/example.ts\n--- a/src/example.ts\n+++ b/src/example.ts\n@@ -1 +1 @@\n-const x = 1;\n+const x = 2;\n",
  "mode": "both"
}
```

## 工具输入参考

除非另有说明，所有字段均为可选：

- `before`（`string`）：原始文本。省略 `patch` 时与 `after` 一起必填。
- `after`（`string`）：更新后文本。省略 `patch` 时与 `before` 一起必填。
- `patch`（`string`）：统一差异文本。与 `before` 和 `after` 互斥。
- `path`（`string`）：前/后模式的显示文件名。
- `lang`（`string`）：前/后模式的语言覆盖提示。
- `title`（`string`）：查看器标题覆盖。
- `mode`（`"view" | "file" | "both"`）：输出模式。默认为插件默认值 `defaults.mode`。已废弃别名：`"image"` 行为与 `"file"` 相同，仍向后兼容。
- `theme`（`"light" | "dark"`）：查看器主题。
- `layout`（`"unified" | "split"`）：差异布局。
- `expandUnchanged`（`boolean`）：有完整上下文时展开未修改部分。仅限每次调用选项。
- `fileFormat`（`"png" | "pdf"`）：渲染文件格式。
- `fileQuality`（`"standard" | "hq" | "print"`）：PNG 或 PDF 渲染的质量预设。
- `fileScale`（`number`）：设备缩放覆盖（`1`-`4`）。
- `fileMaxWidth`（`number`）：最大渲染宽度（CSS 像素，`640`-`2400`）。
- `ttlSeconds`（`number`）：查看器构件 TTL（秒）。默认 1800，最大 21600。
- `baseUrl`（`string`）：查看器 URL 来源覆盖。必须为 `http` 或 `https`，不含查询/hash。

验证和限制：

- `before` 和 `after` 各最大 512 KiB。
- `patch` 最大 2 MiB。
- `path` 最大 2048 字节。
- 补丁复杂度上限：最多 128 个文件和 120000 行总计。
- `patch` 与 `before` 或 `after` 一起使用会被拒绝。

## 安全配置

- `security.allowRemoteViewer`（`boolean`，默认 `false`）
  - `false`：非回环地址对查看器路由的请求被拒绝。
  - `true`：当分词路径有效时允许远程查看器。

## 构件生命周期与存储

- 构件存储在临时子文件夹：`$TMPDIR/openclaw-diffs`。
- 默认查看器 TTL 为 30 分钟。
- 最大接受查看器 TTL 为 6 小时。
- 清理在构件创建后机会性运行。

## 故障排除

输入验证错误：

- `Provide patch or both before and after text.`——包含 `before` 和 `after`，或提供 `patch`。
- `Provide either patch or before/after input, not both.`——不要混合输入模式。
- `Invalid baseUrl: ...`——使用带可选路径的 `http(s)` 来源，不含查询/hash。

文件模式的浏览器要求：

`mode: "file"` 和 `mode: "both"` 需要 Chromium 兼容的浏览器。

解析顺序：

1. OpenClaw 配置中的 `browser.executablePath`。
2. 环境变量：`OPENCLAW_BROWSER_EXECUTABLE_PATH`、`BROWSER_EXECUTABLE_PATH`、`PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH`。
3. 平台命令/路径发现回退。

## 相关文档

- [工具概览](/tools)
- [插件](/tools/plugin)
- [浏览器](/tools/browser)
