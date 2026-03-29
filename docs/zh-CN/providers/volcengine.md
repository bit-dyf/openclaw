---
read_when:
  - 你想在 OpenClaw 中使用火山引擎或豆包模型
  - 你需要设置 Volcengine API 密钥
summary: 火山引擎设置（豆包模型、通用及编程端点）
title: Volcengine（豆包）
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 67cad03eb5ec77e8cc07b0496b196d3b4a475b7eaac13f592045a98d443b3895
  source_path: providers/volcengine.md
  workflow: 15
---

# Volcengine（豆包）

Volcengine 提供商可访问豆包模型以及托管在火山引擎上的第三方模型，针对通用和编程工作负载分别提供独立端点。

- 提供商：`volcengine`（通用）+ `volcengine-plan`（编程）
- 认证：`VOLCANO_ENGINE_API_KEY`
- API：OpenAI 兼容

## 快速开始

1. 设置 API 密钥：

```bash
openclaw onboard --auth-choice volcengine-api-key
```

2. 设置默认模型：

```json5
{
  agents: {
    defaults: {
      model: { primary: "volcengine-plan/ark-code-latest" },
    },
  },
}
```

## 非交互式示例

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice volcengine-api-key \
  --volcengine-api-key "$VOLCANO_ENGINE_API_KEY"
```

## 提供商与端点

| 提供商            | 端点                                      | 使用场景 |
| ----------------- | ----------------------------------------- | -------- |
| `volcengine`      | `ark.cn-beijing.volces.com/api/v3`        | 通用模型 |
| `volcengine-plan` | `ark.cn-beijing.volces.com/api/coding/v3` | 编程模型 |

两个提供商共享同一个 API 密钥进行配置。设置时会自动注册两者。

## 可用模型

- **doubao-seed-1-8** — 豆包 Seed 1.8（通用，默认）
- **doubao-seed-code-preview** — 豆包编程模型
- **ark-code-latest** — 编程计划默认模型
- **Kimi K2.5** — 通过火山引擎的 Moonshot AI
- **GLM-4.7** — 通过火山引擎的 GLM
- **DeepSeek V3.2** — 通过火山引擎的 DeepSeek

大多数模型支持文本 + 图像输入。上下文窗口从 128K 到 256K token 不等。

## 环境变量说明

如果 Gateway 网关以守护进程方式运行（launchd/systemd），请确保 `VOLCANO_ENGINE_API_KEY` 对该进程可用（例如，在 `~/.openclaw/.env` 中或通过 `env.shellEnv`）。
