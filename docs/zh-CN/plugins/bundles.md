---
summary: "安装并使用 Codex、Claude 和 Cursor 捆绑包作为 OpenClaw 插件"
read_when:
  - 您想要安装 Codex、Claude 或 Cursor 兼容捆绑包
  - 您需要了解 OpenClaw 如何将捆绑包内容映射到原生功能
  - 您正在调试捆绑包检测或缺失的功能
title: "插件捆绑包"
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: fcd692f3fcc97b1fa08d12fd6dd6889ef8ebd3b733e9aaa914758fa0e03daba7
  source_path: docs/plugins/bundles.md
  workflow: 15
---

# 插件捆绑包

OpenClaw 可以从三个外部生态系统安装插件：**Codex**、**Claude** 和 **Cursor**。这些被称为**捆绑包** — 内容和元数据包，OpenClaw 会将其映射到原生功能，如 skills、钩子和 MCP 工具。

<Info>
  捆绑包**不同于**原生 OpenClaw 插件。原生插件在进程内运行，可以注册任何功能。捆绑包是具有选择性功能映射和更窄的信任边界的内容包。
</Info>

## 为什么存在捆绑包

许多有用的插件以 Codex、Claude 或 Cursor 格式发布。OpenClaw 检测这些格式并将其支持的内容映射到原生功能集，而不是要求作者将它们重写为原生 OpenClaw 插件。这意味着您可以安装 Claude 命令包或 Codex skill 捆绑包并立即使用它。

## 安装捆绑包

<Steps>
  <Step title="从目录、存档或市场安装">
    ```bash
    # 本地目录
    openclaw plugins install ./my-bundle

    # 存档
    openclaw plugins install ./my-bundle.tgz

    # Claude 市场
    openclaw plugins marketplace list <marketplace-name>
    openclaw plugins install <plugin-name>@<marketplace-name>
    ```

  </Step>

  <Step title="验证检测">
    ```bash
    openclaw plugins list
    openclaw plugins inspect <id>
    ```

    捆绑包显示为 `Format: bundle`，子类型为 `codex`、`claude` 或 `cursor`。

  </Step>

  <Step title="重启并使用">
    ```bash
    openclaw gateway restart
    ```

    映射的功能（skills、钩子、MCP 工具）将在下次会话中可用。

  </Step>
</Steps>

## OpenClaw 从捆绑包映射的内容

并非每个捆绑包功能都能在今天的 OpenClaw 中运行。以下是有效的和已检测但尚未连接的内容。

### 当前支持

| 功能          | 如何映射                                                                                                                | 适用于         |
| ------------- | ----------------------------------------------------------------------------------------------------------------------- | -------------- |
| Skill 内容    | 捆绑包 skill 根目录作为普通 OpenClaw skills 加载                                                                         | 所有格式       |
| Commands      | `commands/` 和 `.cursor/commands/` 被视为 skill 根目录                                                                   | Claude、Cursor |
| Hook 包       | OpenClaw 风格的 `HOOK.md` + `handler.ts` 布局                                                                           | Codex          |
| MCP 工具      | 捆绑包 MCP 配置合并到嵌入式 Pi 设置中；支持的 stdio 服务器作为子进程启动                                                | 所有格式       |
| Settings      | Claude `settings.json` 作为嵌入式 Pi 默认值导入                                                                          | Claude         |

### 已检测但未执行

这些被识别并显示在诊断中，但 OpenClaw 不运行它们：

- Claude `agents`、`hooks.json` 自动化、`lspServers`、`outputStyles`
- Cursor `.cursor/agents`、`.cursor/hooks.json`、`.cursor/rules`
- Codex 内联/应用元数据（除了功能报告之外）

## 捆绑包格式

<AccordionGroup>
  <Accordion title="Codex 捆绑包">
    标记：`.codex-plugin/plugin.json`

    可选内容：`skills/`、`hooks/`、`.mcp.json`、`.app.json`

    当 Codex 捆绑包使用 skill 根目录和 OpenClaw 风格的 hook-pack 目录（`HOOK.md` + `handler.ts`）时，它们最适合 OpenClaw。

  </Accordion>

  <Accordion title="Claude 捆绑包">
    两种检测模式：

    - **基于清单：** `.claude-plugin/plugin.json`
    - **无清单：** 默认 Claude 布局（`skills/`、`commands/`、`agents/`、`hooks/`、`.mcp.json`、`settings.json`）

    Claude 特定行为：

    - `commands/` 被视为 skill 内容
    - `settings.json` 被导入到嵌入式 Pi 设置中（shell 覆盖键被清理）
    - `.mcp.json` 将支持的 stdio 工具暴露给嵌入式 Pi
    - `hooks/hooks.json` 被检测但未执行
    - 清单中的自定义组件路径是附加的（它们扩展默认值，不替换它们）

  </Accordion>

  <Accordion title="Cursor 捆绑包">
    标记：`.cursor-plugin/plugin.json`

    可选内容：`skills/`、`.cursor/commands/`、`.cursor/agents/`、`.cursor/rules/`、`.cursor/hooks.json`、`.mcp.json`

    - `.cursor/commands/` 被视为 skill 内容
    - `.cursor/rules/`、`.cursor/agents/` 和 `.cursor/hooks.json` 仅检测

  </Accordion>
</AccordionGroup>

## 检测优先级

OpenClaw 首先检查原生插件格式：

1. `openclaw.plugin.json` 或带有 `openclaw.extensions` 的有效 `package.json` — 作为**原生插件**处理
2. 捆绑包标记（`.codex-plugin/`、`.claude-plugin/` 或默认 Claude/Cursor 布局）— 作为**捆绑包**处理

如果目录同时包含两者，OpenClaw 使用原生路径。这可以防止双格式包被部分安装为捆绑包。

## 安全性

捆绑包的信任边界比原生插件更窄：

- OpenClaw **不会**在进程内加载任意捆绑包运行时模块
- Skills 和 hook-pack 路径必须保持在插件根目录内（边界检查）
- 设置文件使用相同的边界检查进行读取
- 支持的 stdio MCP 服务器可能作为子进程启动

这使得捆绑包默认更安全，但您仍应将第三方捆绑包视为它们所暴露功能的可信内容。

## 故障排除

<AccordionGroup>
  <Accordion title="捆绑包被检测但功能未运行">
    运行 `openclaw plugins inspect <id>`。如果功能被列出但标记为未连接，这是产品限制 — 而不是安装损坏。
  </Accordion>

  <Accordion title="Claude 命令文件未出现">
    确保捆绑包已启用，并且 markdown 文件位于检测到的 `commands/` 或 `skills/` 根目录内。
  </Accordion>

  <Accordion title="Claude 设置未应用">
    仅支持 `settings.json` 中的嵌入式 Pi 设置。OpenClaw 不将捆绑包设置视为原始配置补丁。
  </Accordion>

  <Accordion title="Claude 钩子未执行">
    `hooks/hooks.json` 仅检测。如果您需要可运行的钩子，请使用 OpenClaw hook-pack 布局或提供原生插件。
  </Accordion>
</AccordionGroup>

## 相关

- [安装和配置插件](/tools/plugin)
- [构建插件](/plugins/building-plugins) — 创建原生插件
- [插件清单](/plugins/manifest) — 原生清单架构
