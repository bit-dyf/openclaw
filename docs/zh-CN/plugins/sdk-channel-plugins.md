---
title: "构建渠道插件"
sidebarTitle: "Channel Plugins"
summary: "为 OpenClaw 构建消息渠道插件的分步指南"
read_when:
  - 正在构建新的消息渠道插件
  - 希望将 OpenClaw 连接到某个消息平台
  - 需要了解 ChannelPlugin 适配器接口
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 38a04ead1ada416feb16d0ae0b153e2ab42cead938c2323bacc2b17ea129ce7b
  source_path: plugins/sdk-channel-plugins.md
  workflow: 15
---

# 构建渠道插件

本指南介绍如何构建将 OpenClaw 连接到消息平台的渠道插件。完成后，您将拥有一个支持私信安全、配对、回复线程和出站消息的完整渠道。

<Info>
  如果您之前从未构建过 OpenClaw 插件，请先阅读
  [入门指南](/plugins/building-plugins)，了解基本的包结构和清单配置。
</Info>

## 渠道插件的工作原理

渠道插件无需自己注册发送/编辑/反应工具。OpenClaw 在核心中保留了一个共享的 `message` 工具。您的插件负责：

- **配置** — 账户解析和设置向导
- **安全** — 私信策略和许可列表
- **配对** — 私信审批流程
- **出站** — 向平台发送文本、媒体和投票
- **线程** — 回复如何组织成线程

核心负责共享的消息工具、提示词连接、会话记账和调度。

## 操作步骤

<Steps>
  <Step title="包和清单">
    创建标准插件文件。`package.json` 中的 `channel` 字段使其成为渠道插件：

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-chat",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "acme-chat",
          "label": "Acme Chat",
          "blurb": "Connect OpenClaw to Acme Chat."
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-chat",
      "kind": "channel",
      "channels": ["acme-chat"],
      "name": "Acme Chat",
      "description": "Acme Chat channel plugin",
      "configSchema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "acme-chat": {
            "type": "object",
            "properties": {
              "token": { "type": "string" },
              "allowFrom": {
                "type": "array",
                "items": { "type": "string" }
              }
            }
          }
        }
      }
    }
    ```
    </CodeGroup>

  </Step>

  <Step title="构建渠道插件对象">
    `ChannelPlugin` 接口有许多可选的适配器接口。从最小配置开始——`id` 和 `setup`——然后按需添加适配器。

    创建 `src/channel.ts`：

    ```typescript src/channel.ts
    import {
      createChatChannelPlugin,
      createChannelPluginBase,
    } from "openclaw/plugin-sdk/core";
    import type { OpenClawConfig } from "openclaw/plugin-sdk/core";
    import { acmeChatApi } from "./client.js"; // 您的平台 API 客户端

    type ResolvedAccount = {
      accountId: string | null;
      token: string;
      allowFrom: string[];
      dmPolicy: string | undefined;
    };

    function resolveAccount(
      cfg: OpenClawConfig,
      accountId?: string | null,
    ): ResolvedAccount {
      const section = (cfg.channels as Record<string, any>)?.["acme-chat"];
      const token = section?.token;
      if (!token) throw new Error("acme-chat: token is required");
      return {
        accountId: accountId ?? null,
        token,
        allowFrom: section?.allowFrom ?? [],
        dmPolicy: section?.dmSecurity,
      };
    }

    export const acmeChatPlugin = createChatChannelPlugin<ResolvedAccount>({
      base: createChannelPluginBase({
        id: "acme-chat",
        setup: {
          resolveAccount,
          inspectAccount(cfg, accountId) {
            const section =
              (cfg.channels as Record<string, any>)?.["acme-chat"];
            return {
              enabled: Boolean(section?.token),
              configured: Boolean(section?.token),
              tokenStatus: section?.token ? "available" : "missing",
            };
          },
        },
      }),

      // 私信安全：谁可以给机器人发消息
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // 配对：新私信联系人的审批流程
      pairing: {
        text: {
          idLabel: "Acme Chat username",
          message: "Send this code to verify your identity:",
          notify: async ({ target, code }) => {
            await acmeChatApi.sendDm(target, `Pairing code: ${code}`);
          },
        },
      },

      // 线程：回复的发送方式
      threading: { topLevelReplyToMode: "reply" },

      // 出站：向平台发送消息
      outbound: {
        attachedResults: {
          sendText: async (params) => {
            const result = await acmeChatApi.sendMessage(
              params.to,
              params.text,
            );
            return { messageId: result.id };
          },
        },
        base: {
          sendMedia: async (params) => {
            await acmeChatApi.sendFile(params.to, params.filePath);
          },
        },
      },
    });
    ```

    <Accordion title="createChatChannelPlugin 为您做了什么">
      您无需手动实现低层适配器接口，只需传入声明式选项，构建器会自动组合：

      | 选项 | 连接内容 |
      | --- | --- |
      | `security.dm` | 从配置字段派生的作用域私信安全解析器 |
      | `pairing.text` | 基于文本的私信配对流程，含验证码交换 |
      | `threading` | 回复模式解析器（固定、账户范围或自定义） |
      | `outbound.attachedResults` | 返回结果元数据（消息 ID）的发送函数 |

      如果需要完全控制，也可以传入原始适配器对象而非声明式选项。
    </Accordion>

  </Step>

  <Step title="连接入口点">
    创建 `index.ts`：

    ```typescript index.ts
    import { defineChannelPluginEntry } from "openclaw/plugin-sdk/core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineChannelPluginEntry({
      id: "acme-chat",
      name: "Acme Chat",
      description: "Acme Chat channel plugin",
      plugin: acmeChatPlugin,
      registerFull(api) {
        api.registerCli(
          ({ program }) => {
            program
              .command("acme-chat")
              .description("Acme Chat management");
          },
          { commands: ["acme-chat"] },
        );
      },
    });
    ```

    `defineChannelPluginEntry` 会自动处理设置/完整注册的拆分。
    所有选项请参阅[入口点](/plugins/sdk-entrypoints#definechannelpluginentry)。

  </Step>

  <Step title="添加设置入口">
    创建 `setup-entry.ts`，用于入门引导期间的轻量加载：

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    当渠道被禁用或未配置时，OpenClaw 会加载此文件而非完整入口，
    避免在设置流程中引入沉重的运行时代码。
    详情请参阅[设置与配置](/plugins/sdk-setup#setup-entry)。

  </Step>

  <Step title="处理入站消息">
    您的插件需要接收来自平台的消息并转发给 OpenClaw。典型模式是使用 Webhook
    验证请求并通过渠道的入站处理器分发消息：

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // 插件管理的认证（自行验证签名）
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // 您的入站处理器将消息分发到 OpenClaw。
          // 具体连接方式取决于您的平台 SDK——
          // 请参阅捆绑的 Microsoft Teams 或 Google Chat 插件包中的真实示例。
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      入站消息处理是渠道特定的。每个渠道插件拥有自己的入站流水线。
      请查看捆绑的渠道插件（例如 Microsoft Teams 或 Google Chat 插件包）了解真实模式。
    </Note>

  </Step>

  <Step title="测试">
    在 `src/channel.test.ts` 中编写并置测试：

    ```typescript src/channel.test.ts
    import { describe, it, expect } from "vitest";
    import { acmeChatPlugin } from "./channel.js";

    describe("acme-chat plugin", () => {
      it("resolves account from config", () => {
        const cfg = {
          channels: {
            "acme-chat": { token: "test-token", allowFrom: ["user1"] },
          },
        } as any;
        const account = acmeChatPlugin.setup!.resolveAccount(cfg, undefined);
        expect(account.token).toBe("test-token");
      });

      it("inspects account without materializing secrets", () => {
        const cfg = {
          channels: { "acme-chat": { token: "test-token" } },
        } as any;
        const result = acmeChatPlugin.setup!.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(true);
        expect(result.tokenStatus).toBe("available");
      });

      it("reports missing config", () => {
        const cfg = { channels: {} } as any;
        const result = acmeChatPlugin.setup!.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(false);
      });
    });
    ```

    ```bash
    pnpm test -- <bundled-plugin-root>/acme-chat/
    ```

    共享测试辅助工具请参阅[测试](/plugins/sdk-testing)。

  </Step>
</Steps>

## 文件结构

```
<bundled-plugin-root>/acme-chat/
├── package.json              # openclaw.channel 元数据
├── openclaw.plugin.json      # 含配置 schema 的清单
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # 公开导出（可选）
├── runtime-api.ts            # 内部运行时导出（可选）
└── src/
    ├── channel.ts            # 通过 createChatChannelPlugin 实现的 ChannelPlugin
    ├── channel.test.ts       # 测试
    ├── client.ts             # 平台 API 客户端
    └── runtime.ts            # 运行时存储（如需要）
```

## 进阶主题

<CardGroup cols={2}>
  <Card title="线程选项" icon="git-branch" href="/plugins/sdk-entrypoints#registration-mode">
    固定、账户范围或自定义回复模式
  </Card>
  <Card title="消息工具集成" icon="puzzle" href="/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool 和操作发现
  </Card>
  <Card title="目标解析" icon="crosshair" href="/plugins/architecture#channel-target-resolution">
    inferTargetChatType、looksLikeId、resolveTarget
  </Card>
  <Card title="运行时辅助工具" icon="settings" href="/plugins/sdk-runtime">
    通过 api.runtime 使用 TTS、STT、媒体、子智能体
  </Card>
</CardGroup>

## 下一步

- [提供商插件](/plugins/sdk-provider-plugins) — 如果您的插件还提供模型
- [SDK 概览](/plugins/sdk-overview) — 完整的子路径导入参考
- [SDK 测试](/plugins/sdk-testing) — 测试工具和合约测试
- [插件清单](/plugins/manifest) — 完整的清单 schema
