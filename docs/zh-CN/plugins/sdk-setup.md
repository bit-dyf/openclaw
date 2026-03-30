---
title: "插件设置和配置"
sidebarTitle: "设置和配置"
summary: "设置向导、setup-entry.ts、配置架构和 package.json 元数据"
read_when:
  - 您正在向插件添加设置向导
  - 您需要了解 setup-entry.ts 与 index.ts
  - 您正在定义插件配置架构或 package.json openclaw 元数据
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 0fb3285471bb4e81c3fd76517ea587c0a832e2ef6db808c744fda64b2f6eb2a1
  source_path: docs/plugins/sdk-setup.md
  workflow: 15
---

# 插件设置和配置

插件打包（`package.json` 元数据）、清单（`openclaw.plugin.json`）、设置入口和配置架构的参考。

<Tip>
  **寻找演练？** 操作指南在上下文中涵盖打包：[渠道插件](/plugins/sdk-channel-plugins#step-1-package-and-manifest)和[提供商插件](/plugins/sdk-provider-plugins#step-1-package-and-manifest)。
</Tip>

## 包元数据

您的 `package.json` 需要一个 `openclaw` 字段，告诉插件系统您的插件提供什么：

**渠道插件：**

```json
{
  "name": "@myorg/openclaw-my-channel",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "channel": {
      "id": "my-channel",
      "label": "My Channel",
      "blurb": "Short description of the channel."
    }
  }
}
```

**提供商插件：**

```json
{
  "name": "@myorg/openclaw-my-provider",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "providers": ["my-provider"]
  }
}
```

### `openclaw` 字段

| 字段         | 类型       | 描述                                                                                       |
| ------------ | ---------- | ------------------------------------------------------------------------------------------ |
| `extensions` | `string[]` | 入口点文件（相对于包根目录）                                                               |
| `setupEntry` | `string`   | 轻量级仅设置入口（可选）                                                                   |
| `channel`    | `object`   | 渠道元数据：`id`、`label`、`blurb`、`selectionLabel`、`docsPath`、`order`、`aliases`       |
| `providers`  | `string[]` | 此插件注册的提供商 id                                                                      |
| `install`    | `object`   | 安装提示：`npmSpec`、`localPath`、`defaultChoice`                                          |
| `startup`    | `object`   | 启动行为标志                                                                               |

### 延迟完整加载

渠道插件可以选择延迟加载：

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

启用后，OpenClaw 在监听前启动阶段仅加载 `setupEntry`，即使对于已配置的渠道也是如此。完整入口在 Gateway 网关开始监听后加载。

<Warning>
  仅当您的 `setupEntry` 注册了 Gateway 网关在开始监听之前所需的所有内容（渠道注册、HTTP 路由、网关方法）时，才启用延迟加载。如果完整入口拥有所需的启动功能，请保持默认行为。
</Warning>

## 插件清单

每个原生插件必须在包根目录中提供 `openclaw.plugin.json`。OpenClaw 使用它来验证配置，而无需执行插件代码。

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds My Plugin capabilities to OpenClaw",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "webhookSecret": {
        "type": "string",
        "description": "Webhook verification secret"
      }
    }
  }
}
```

对于渠道插件，添加 `kind` 和 `channels`：

```json
{
  "id": "my-channel",
  "kind": "channel",
  "channels": ["my-channel"],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

即使没有配置的插件也必须提供架构。空架构是有效的：

```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

请参阅[插件清单](/plugins/manifest)获取完整的架构参考。

## 设置入口

`setup-entry.ts` 文件是 `index.ts` 的轻量级替代方案，OpenClaw 在仅需要设置表面（新手引导、配置修复、禁用的渠道检查）时加载。

```typescript
// setup-entry.ts
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/core";
import { myChannelPlugin } from "./src/channel.js";

export default defineSetupPluginEntry(myChannelPlugin);
```

这避免了在设置流程期间加载繁重的运行时代码（加密库、CLI 注册、后台服务）。

**OpenClaw 何时使用 `setupEntry` 而不是完整入口：**

- 渠道被禁用但需要设置/新手引导表面
- 渠道已启用但未配置
- 延迟加载已启用（`deferConfiguredChannelFullLoadUntilAfterListen`）

**`setupEntry` 必须注册什么：**

- 渠道插件对象（通过 `defineSetupPluginEntry`）
- Gateway 网关监听前所需的任何 HTTP 路由
- 启动期间所需的任何网关方法

**`setupEntry` 不应包含什么：**

- CLI 注册
- 后台服务
- 繁重的运行时导入（加密、SDK）
- 启动后才需要的网关方法

## 配置架构

插件配置根据清单中的 JSON Schema 进行验证。用户通过以下方式配置插件：

```json5
{
  plugins: {
    entries: {
      "my-plugin": {
        config: {
          webhookSecret: "abc123",
        },
      },
    },
  },
}
```

您的插件在注册期间接收此配置作为 `api.pluginConfig`。

对于渠道特定配置，请改用渠道配置部分：

```json5
{
  channels: {
    "my-channel": {
      token: "bot-token",
      allowFrom: ["user1", "user2"],
    },
  },
}
```

### 构建渠道配置架构

使用 `openclaw/plugin-sdk/core` 中的 `buildChannelConfigSchema` 将 Zod 架构转换为 OpenClaw 验证的 `ChannelConfigSchema` 包装器：

```typescript
import { z } from "zod";
import { buildChannelConfigSchema } from "openclaw/plugin-sdk/core";

const accountSchema = z.object({
  token: z.string().optional(),
  allowFrom: z.array(z.string()).optional(),
  accounts: z.object({}).catchall(z.any()).optional(),
  defaultAccount: z.string().optional(),
});

const configSchema = buildChannelConfigSchema(accountSchema);
```

## 设置向导

渠道插件可以为 `openclaw onboard` 提供交互式设置向导。向导是 `ChannelPlugin` 上的 `ChannelSetupWizard` 对象：

```typescript
import type { ChannelSetupWizard } from "openclaw/plugin-sdk/channel-setup";

const setupWizard: ChannelSetupWizard = {
  channel: "my-channel",
  status: {
    configuredLabel: "Connected",
    unconfiguredLabel: "Not configured",
    resolveConfigured: ({ cfg }) => Boolean((cfg.channels as any)?.["my-channel"]?.token),
  },
  credentials: [
    {
      inputKey: "token",
      providerHint: "my-channel",
      credentialLabel: "Bot token",
      preferredEnvVar: "MY_CHANNEL_BOT_TOKEN",
      envPrompt: "Use MY_CHANNEL_BOT_TOKEN from environment?",
      keepPrompt: "Keep current token?",
      inputPrompt: "Enter your bot token:",
      inspect: ({ cfg, accountId }) => {
        const token = (cfg.channels as any)?.["my-channel"]?.token;
        return {
          accountConfigured: Boolean(token),
          hasConfiguredValue: Boolean(token),
        };
      },
    },
  ],
};
```

`ChannelSetupWizard` 类型支持 `credentials`、`textInputs`、`dmPolicy`、`allowFrom`、`groupAccess`、`prepare`、`finalize` 等。请参阅捆绑插件包（例如 Discord 插件 `src/channel.setup.ts`）获取完整示例。

对于仅需要标准 `note -> prompt -> parse -> merge -> patch` 流程的 DM 允许列表提示，请优先使用 `openclaw/plugin-sdk/setup` 中的共享设置帮助器：`createPromptParsedAllowFromForAccount(...)`、`createTopLevelChannelParsedAllowFromPrompt(...)` 和 `createNestedChannelParsedAllowFromPrompt(...)`。

对于仅根据标签、分数和可选额外行变化的渠道设置状态块，请优先使用 `openclaw/plugin-sdk/setup` 中的 `createStandardChannelSetupStatus(...)`，而不是在每个插件中手动滚动相同的 `status` 对象。

对于仅应在某些上下文中出现的可选设置表面，请使用 `openclaw/plugin-sdk/channel-setup` 中的 `createOptionalChannelSetupSurface`：

```typescript
import { createOptionalChannelSetupSurface } from "openclaw/plugin-sdk/channel-setup";

const setupSurface = createOptionalChannelSetupSurface({
  channel: "my-channel",
  label: "My Channel",
  npmSpec: "@myorg/openclaw-my-channel",
  docsPath: "/channels/my-channel",
});
// 返回 { setupAdapter, setupWizard }
```

## 发布和安装

**外部插件：** 发布到 [ClawHub](/tools/clawhub) 或 npm，然后安装：

```bash
openclaw plugins install @myorg/openclaw-my-plugin
```

OpenClaw 首先尝试 ClawHub，然后自动回退到 npm。您也可以强制特定来源：

```bash
openclaw plugins install clawhub:@myorg/openclaw-my-plugin   # 仅 ClawHub
openclaw plugins install npm:@myorg/openclaw-my-plugin       # 仅 npm
```

**仓库内插件：** 放置在捆绑插件工作区树下，它们会在构建期间自动发现。

**用户可以浏览和安装：**

```bash
openclaw plugins search <query>
openclaw plugins install <package-name>
```

<Info>
  对于 npm 来源的安装，`openclaw plugins install` 运行 `npm install --ignore-scripts`（无生命周期脚本）。保持插件依赖树为纯 JS/TS，并避免需要 `postinstall` 构建的包。
</Info>

## 相关

- [SDK 入口点](/plugins/sdk-entrypoints) -- `definePluginEntry` 和 `defineChannelPluginEntry`
- [插件清单](/plugins/manifest) -- 完整清单架构参考
- [构建插件](/plugins/building-plugins) -- 分步入门指南
