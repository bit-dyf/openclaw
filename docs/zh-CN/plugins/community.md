---
summary: "社区维护的 OpenClaw 插件：浏览、安装并提交您自己的插件"
read_when:
  - 您想要查找第三方 OpenClaw 插件
  - 您想要发布或列出您自己的插件
title: "社区插件"
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 6be62d948af344ce2f4bcfe2c375ef83bf49f1dfd6605a0faae1b704fc521fc0
  source_path: docs/plugins/community.md
  workflow: 15
---

# 社区插件

社区插件是由社区构建和维护的第三方包，用于通过新的渠道、工具、提供商或其他功能扩展 OpenClaw。它们发布在 [ClawHub](/tools/clawhub) 或 npm 上，可以通过一个命令安装。

```bash
openclaw plugins install <package-name>
```

OpenClaw 会先检查 ClawHub，然后自动回退到 npm。

## 已列出的插件

### Codex App Server Bridge

用于 Codex App Server 对话的独立 OpenClaw 桥接。将聊天绑定到 Codex 线程，使用纯文本与其对话，并通过聊天原生命令控制它，包括恢复、规划、审查、模型选择、压缩等。

- **npm:** `openclaw-codex-app-server`
- **repo:** [github.com/pwrdrvr/openclaw-codex-app-server](https://github.com/pwrdrvr/openclaw-codex-app-server)

```bash
openclaw plugins install openclaw-codex-app-server
```

### DingTalk

使用 Stream 模式的企业机器人集成。支持通过任何 DingTalk 客户端发送文本、图像和文件消息。

- **npm:** `@largezhou/ddingtalk`
- **repo:** [github.com/largezhou/openclaw-dingtalk](https://github.com/largezhou/openclaw-dingtalk)

```bash
openclaw plugins install @largezhou/ddingtalk
```

### Lossless Claw (LCM)

OpenClaw 的无损上下文管理插件。基于 DAG 的对话总结与增量压缩——在减少令牌使用的同时保持完整的上下文保真度。

- **npm:** `@martian-engineering/lossless-claw`
- **repo:** [github.com/Martian-Engineering/lossless-claw](https://github.com/Martian-Engineering/lossless-claw)

```bash
openclaw plugins install @martian-engineering/lossless-claw
```

### Opik

官方插件，可将智能体跟踪导出到 Opik。监控智能体行为、成本、令牌、错误等。

- **npm:** `@opik/opik-openclaw`
- **repo:** [github.com/comet-ml/opik-openclaw](https://github.com/comet-ml/opik-openclaw)

```bash
openclaw plugins install @opik/opik-openclaw
```

### QQbot

通过 QQ Bot API 将 OpenClaw 连接到 QQ。支持私聊、群组提及、频道消息，以及包括语音、图像、视频和文件在内的富媒体。

- **npm:** `@sliverp/qqbot`
- **repo:** [github.com/sliverp/qqbot](https://github.com/sliverp/qqbot)

```bash
openclaw plugins install @sliverp/qqbot
```

### wecom

OpenClaw 企业微信渠道插件。
由企业微信 AI Bot WebSocket 持久连接驱动的机器人插件，支持私信和群聊、流式回复以及主动消息。

- **npm:** `@wecom/wecom-openclaw-plugin`
- **repo:** [github.com/WecomTeam/wecom-openclaw-plugin](https://github.com/WecomTeam/wecom-openclaw-plugin)

```bash
openclaw plugins install @wecom/wecom-openclaw-plugin
```

## 提交您的插件

我们欢迎有用、有文档且安全操作的社区插件。

<Steps>
  <Step title="发布到 ClawHub 或 npm">
    您的插件必须可以通过 `openclaw plugins install \<package-name\>` 安装。
    发布到 [ClawHub](/tools/clawhub)（首选）或 npm。
    请参阅[构建插件](/plugins/building-plugins)获取完整指南。

  </Step>

  <Step title="在 GitHub 上托管">
    源代码必须位于具有设置文档和问题跟踪器的公共存储库中。

  </Step>

  <Step title="打开 PR">
    将您的插件添加到此页面，包括：

    - 插件名称
    - npm 包名称
    - GitHub 存储库 URL
    - 单行描述
    - 安装命令

  </Step>
</Steps>

## 质量标准

| 要求                     | 原因                                   |
| ------------------------ | -------------------------------------- |
| 发布在 ClawHub 或 npm 上 | 用户需要 `openclaw plugins install` 正常工作 |
| 公共 GitHub 存储库       | 源代码审查、问题跟踪、透明度           |
| 设置和使用文档           | 用户需要知道如何配置它                 |
| 积极维护                 | 最近更新或响应问题处理                 |

低努力的封装、不明确的所有权或未维护的包可能会被拒绝。

## 相关

- [安装和配置插件](/tools/plugin) — 如何安装任何插件
- [构建插件](/plugins/building-plugins) — 创建您自己的插件
- [插件清单](/plugins/manifest) — 清单架构
