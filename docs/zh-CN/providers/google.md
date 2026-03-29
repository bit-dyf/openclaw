---
read_when:
  - 你想在 OpenClaw 中使用 Google Gemini 模型
  - 你需要 API 密钥或 OAuth 认证流程
summary: Google Gemini 设置（API 密钥 + OAuth、图像生成、媒体理解、网络搜索）
title: Google（Gemini）
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 89a9edc5dbcd59b18bab029016e1c578983c85e673b99191ed56e589bedc23e1
  source_path: providers/google.md
  workflow: 15
---

# Google（Gemini）

Google 插件通过 Google AI Studio 提供对 Gemini 模型的访问，以及图像生成、媒体理解（图像/音频/视频）和通过 Gemini Grounding 实现的网络搜索。

- 提供商：`google`
- 认证：`GEMINI_API_KEY` 或 `GOOGLE_API_KEY`
- API：Google Gemini API
- 替代提供商：`google-gemini-cli`（OAuth）

## 快速开始

1. 设置 API 密钥：

```bash
openclaw onboard --auth-choice google-api-key
```

2. 设置默认模型：

```json5
{
  agents: {
    defaults: {
      model: { primary: "google/gemini-3.1-pro-preview" },
    },
  },
}
```

## 非交互式示例

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice google-api-key \
  --gemini-api-key "$GEMINI_API_KEY"
```

## OAuth（Gemini CLI）

替代提供商 `google-gemini-cli` 使用 PKCE OAuth 而非 API 密钥。这是非官方集成；部分用户报告存在账户限制。请自行承担风险。

环境变量：

- `OPENCLAW_GEMINI_OAUTH_CLIENT_ID`
- `OPENCLAW_GEMINI_OAUTH_CLIENT_SECRET`

（或 `GEMINI_CLI_*` 变体。）

## 功能

| 功能                   | 是否支持          |
| ---------------------- | ----------------- |
| 聊天补全               | 是                |
| 图像生成               | 是                |
| 图像理解               | 是                |
| 音频转录               | 是                |
| 视频理解               | 是                |
| 网络搜索（Grounding）  | 是                |
| 思维/推理              | 是（Gemini 3.1+） |

## 环境变量说明

如果 Gateway 网关以守护进程方式运行（launchd/systemd），请确保 `GEMINI_API_KEY` 对该进程可用（例如，在 `~/.openclaw/.env` 中或通过 `env.shellEnv`）。
