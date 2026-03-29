---
title: "插件 SDK 迁移"
sidebarTitle: "迁移到 SDK"
summary: "从旧版向后兼容层迁移到现代插件 SDK"
read_when:
  - 您看到 OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED 警告
  - 您看到 OPENCLAW_EXTENSION_API_DEPRECATED 警告
  - 您正在将插件更新到现代插件架构
  - 您维护外部 OpenClaw 插件
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 0037300ae8d7a3b1f7ac2c3f1029f738386b1732b94eed5631f5079ddaf1833f
  source_path: docs/plugins/sdk-migration.md
  workflow: 15
---

# 插件 SDK 迁移

OpenClaw 已从广泛的向后兼容层转向具有集中、文档化导入的现代插件架构。如果您的插件是在新架构之前构建的，本指南将帮助您迁移。

## 正在改变的内容

旧的插件系统提供了两个开放式表面，让插件可以从单个入口点导入它们需要的任何内容：

- **`openclaw/plugin-sdk/compat`** — 一个重新导出数十个帮助器的单一导入。它是为了在构建新插件架构时保持旧的基于钩子的插件工作而引入的。
- **`openclaw/extension-api`** — 一个桥接，使插件能够直接访问主机端帮助器，例如嵌入式智能体运行器。

这两个表面现在都已**弃用**。它们在运行时仍然有效，但新插件不得使用它们，现有插件应在下一个主要版本删除它们之前迁移。

<Warning>
  向后兼容层将在未来的主要版本中删除。当这种情况发生时，仍从这些表面导入的插件将中断。
</Warning>

## 为什么会改变

旧方法导致了问题：

- **启动缓慢** — 导入一个帮助器会加载数十个不相关的模块
- **循环依赖** — 广泛的重新导出使创建导入循环变得容易
- **不清楚的 API 表面** — 无法区分哪些导出是稳定的还是内部的

现代插件 SDK 解决了这个问题：每个导入路径（`openclaw/plugin-sdk/\<subpath\>`）都是一个小型、自包含的模块，具有明确的目的和文档化的契约。

## 如何迁移

<Steps>
  <Step title="查找已弃用的导入">
    在您的插件中搜索来自任一已弃用表面的导入：

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="替换为集中导入">
    旧表面的每个导出都映射到特定的现代导入路径：

    ```typescript
    // 之前（已弃用的向后兼容层）
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // 之后（现代集中导入）
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    对于主机端帮助器，使用注入的插件运行时而不是直接导入：

    ```typescript
    // 之前（已弃用的 extension-api 桥接）
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // 之后（注入的运行时）
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    相同的模式适用于其他旧版桥接帮助器：

    | 旧导入 | 现代等效项 |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | 会话存储帮助器 | `api.runtime.agent.session.*` |

  </Step>

  <Step title="构建和测试">
    ```bash
    pnpm build
    pnpm test -- my-plugin/
    ```
  </Step>
</Steps>

## 导入路径参考

<Accordion title="完整导入路径表">
  | 导入路径 | 目的 | 关键导出 |
  | --- | --- | --- |
  | `plugin-sdk/plugin-entry` | 规范插件入口帮助器 | `definePluginEntry` |
  | `plugin-sdk/core` | 渠道入口定义、渠道构建器、基本类型 | `defineChannelPluginEntry`、`createChatChannelPlugin` |
  | `plugin-sdk/channel-setup` | 设置向导适配器 | `createOptionalChannelSetupSurface` |
  | `plugin-sdk/channel-pairing` | DM 配对原语 | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | 回复前缀 + 输入连接 | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | 配置适配器工厂 | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | 配置架构构建器 | 渠道配置架构类型 |
  | `plugin-sdk/channel-policy` | 组/DM 策略解析 | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | 帐户状态跟踪 | `createAccountStatusSink` |
  | `plugin-sdk/channel-runtime` | 运行时连接帮助器 | 渠道运行时实用程序 |
  | `plugin-sdk/channel-send-result` | 发送结果类型 | 回复结果类型 |
  | `plugin-sdk/runtime-store` | 持久插件存储 | `createPluginRuntimeStore` |
  | `plugin-sdk/approval-runtime` | 批准提示帮助器 | 执行/插件批准有效负载和回复帮助器 |
  | `plugin-sdk/collection-runtime` | 有界缓存帮助器 | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | 诊断门控帮助器 | `isDiagnosticFlagEnabled`、`isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | 错误格式化帮助器 | `formatUncaughtError`、错误图帮助器 |
  | `plugin-sdk/fetch-runtime` | 包装的 fetch/代理帮助器 | `resolveFetch`、代理帮助器 |
  | `plugin-sdk/host-runtime` | 主机规范化帮助器 | `normalizeHostname`、`normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | 重试帮助器 | `RetryConfig`、`retryAsync`、策略运行器 |
  | `plugin-sdk/allow-from` | 允许列表格式化 | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | 允许列表输入映射 | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | 命令门控 | `resolveControlCommandGate` |
  | `plugin-sdk/secret-input` | 密钥输入解析 | 密钥输入帮助器 |
  | `plugin-sdk/webhook-ingress` | Webhook 请求帮助器 | Webhook 目标实用程序 |
  | `plugin-sdk/webhook-request-guards` | Webhook 主体保护帮助器 | 请求主体读取/限制帮助器 |
  | `plugin-sdk/reply-payload` | 消息回复类型 | 回复有效负载类型 |
  | `plugin-sdk/provider-onboard` | 提供商新手引导补丁 | 新手引导配置帮助器 |
  | `plugin-sdk/keyed-async-queue` | 有序异步队列 | `KeyedAsyncQueue` |
  | `plugin-sdk/testing` | 测试实用程序 | 测试帮助器和模拟 |
</Accordion>

使用与工作匹配的最窄导入。如果找不到导出，请检查 `src/plugin-sdk/` 的源代码或在 Discord 中询问。

## 删除时间表

| 何时                   | 发生什么                                                        |
| ---------------------- | --------------------------------------------------------------- |
| **现在**               | 已弃用的表面发出运行时警告                                      |
| **下一个主要版本**     | 已弃用的表面将被删除；仍在使用它们的插件将失败                  |

所有核心插件都已迁移。外部插件应在下一个主要版本之前迁移。

## 暂时抑制警告

在迁移工作时设置这些环境变量：

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

这是一个临时逃生舱口，而不是永久解决方案。

## 相关

- [入门](/plugins/building-plugins) — 构建您的第一个插件
- [SDK 概览](/plugins/sdk-overview) — 完整子路径导入参考
- [渠道插件](/plugins/sdk-channel-plugins) — 构建渠道插件
- [提供商插件](/plugins/sdk-provider-plugins) — 构建提供商插件
- [插件内部](/plugins/architecture) — 架构深入探讨
- [插件清单](/plugins/manifest) — 清单架构参考
