---
title: "插件入口点"
sidebarTitle: "入口点"
summary: "definePluginEntry、defineChannelPluginEntry 和 defineSetupPluginEntry 的参考"
read_when:
  - 您需要 definePluginEntry 或 defineChannelPluginEntry 的确切类型签名
  - 您想要了解注册模式（完整 vs 设置）
  - 您正在查找入口点选项
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 25a6a480f03fb23b8fe1ae0eb628f90e4cbfd8546c6275ad556561b7a4797b79
  source_path: docs/plugins/sdk-entrypoints.md
  workflow: 15
---

# 插件入口点

每个插件导出一个默认入口对象。SDK 提供三个帮助器来创建它们。

<Tip>
  **寻找演练？** 请参阅[渠道插件](/plugins/sdk-channel-plugins)或[提供商插件](/plugins/sdk-provider-plugins)获取分步指南。
</Tip>

## `definePluginEntry`

**导入：** `openclaw/plugin-sdk/plugin-entry`

用于提供商插件、工具插件、钩子插件以及**不是**消息渠道的任何内容。

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  description: "Short summary",
  register(api) {
    api.registerProvider({
      /* ... */
    });
    api.registerTool({
      /* ... */
    });
  },
});
```

| 字段           | 类型                                                             | 必需 | 默认值              |
| -------------- | ---------------------------------------------------------------- | ---- | ------------------- |
| `id`           | `string`                                                         | 是   | —                   |
| `name`         | `string`                                                         | 是   | —                   |
| `description`  | `string`                                                         | 是   | —                   |
| `kind`         | `string`                                                         | 否   | —                   |
| `configSchema` | `OpenClawPluginConfigSchema \| () => OpenClawPluginConfigSchema` | 否   | 空对象架构          |
| `register`     | `(api: OpenClawPluginApi) => void`                               | 是   | —                   |

- `id` 必须与您的 `openclaw.plugin.json` 清单匹配。
- `kind` 用于独占插槽：`"memory"` 或 `"context-engine"`。
- `configSchema` 可以是一个函数，用于延迟评估。

## `defineChannelPluginEntry`

**导入：** `openclaw/plugin-sdk/core`

使用渠道特定的连接包装 `definePluginEntry`。自动调用 `api.registerChannel({ plugin })` 并根据注册模式限制 `registerFull`。

```typescript
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/core";

export default defineChannelPluginEntry({
  id: "my-channel",
  name: "My Channel",
  description: "Short summary",
  plugin: myChannelPlugin,
  setRuntime: setMyRuntime,
  registerFull(api) {
    api.registerCli(/* ... */);
    api.registerGatewayMethod(/* ... */);
  },
});
```

| 字段           | 类型                                                             | 必需 | 默认值              |
| -------------- | ---------------------------------------------------------------- | -------- | ------------------- |
| `id`           | `string`                                                         | 是       | —                   |
| `name`         | `string`                                                         | 是       | —                   |
| `description`  | `string`                                                         | 是       | —                   |
| `plugin`       | `ChannelPlugin`                                                  | 是       | —                   |
| `configSchema` | `OpenClawPluginConfigSchema \| () => OpenClawPluginConfigSchema` | 否       | 空对象架构          |
| `setRuntime`   | `(runtime: PluginRuntime) => void`                               | 否       | —                   |
| `registerFull` | `(api: OpenClawPluginApi) => void`                               | 否       | —                   |

- `setRuntime` 在注册期间调用，以便您可以存储运行时引用（通常通过 `createPluginRuntimeStore`）。
- `registerFull` 仅在 `api.registrationMode === "full"` 时运行。在仅设置加载期间会跳过它。

## `defineSetupPluginEntry`

**导入：** `openclaw/plugin-sdk/core`

用于轻量级 `setup-entry.ts` 文件。返回仅包含 `{ plugin }` 且没有运行时或 CLI 连接的对象。

```typescript
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/core";

export default defineSetupPluginEntry(myChannelPlugin);
```

当渠道被禁用、未配置或启用延迟加载时，OpenClaw 加载此文件而不是完整入口。请参阅[设置和配置](/plugins/sdk-setup#setup-entry)了解这何时重要。

## 注册模式

`api.registrationMode` 告诉您的插件它是如何加载的：

| 模式              | 何时                          | 注册什么                  |
| ----------------- | ----------------------------- | ------------------------- |
| `"full"`          | 正常 Gateway 网关启动         | 所有内容                  |
| `"setup-only"`    | 禁用/未配置的渠道             | 仅渠道注册                |
| `"setup-runtime"` | 具有可用运行时的设置流程      | 渠道 + 轻量级运行时       |

`defineChannelPluginEntry` 会自动处理此拆分。如果您直接使用 `definePluginEntry` 用于渠道，请自己检查模式：

```typescript
register(api) {
  api.registerChannel({ plugin: myPlugin });
  if (api.registrationMode !== "full") return;

  // 仅限重型运行时注册
  api.registerCli(/* ... */);
  api.registerService(/* ... */);
}
```

## 插件形状

OpenClaw 根据其注册行为对加载的插件进行分类：

| 形状                  | 描述                                           |
| --------------------- | ---------------------------------------------- |
| **plain-capability**  | 一种功能类型（例如仅提供商）                   |
| **hybrid-capability** | 多种功能类型（例如提供商 + 语音）              |
| **hook-only**         | 仅钩子，无功能                                 |
| **non-capability**    | 工具/命令/服务但无功能                         |

使用 `openclaw plugins inspect <id>` 查看插件的形状。

## 相关

- [SDK 概览](/plugins/sdk-overview) — 注册 API 和子路径参考
- [运行时帮助器](/plugins/sdk-runtime) — `api.runtime` 和 `createPluginRuntimeStore`
- [设置和配置](/plugins/sdk-setup) — 清单、设置入口、延迟加载
- [渠道插件](/plugins/sdk-channel-plugins) — 构建 `ChannelPlugin` 对象
- [提供商插件](/plugins/sdk-provider-plugins) — 提供商注册和钩子
