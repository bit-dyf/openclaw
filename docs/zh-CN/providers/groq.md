---
read_when:
  - 你想在 OpenClaw 中使用 Groq
  - 你需要 API 密钥环境变量或 CLI 认证选项
summary: Groq 设置（认证 + 模型选择）
title: Groq
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 00e494781af4c38c8b754ee8fe3cfe1c6fc485d552910db991d99d408b27ca55
  source_path: providers/groq.md
  workflow: 15
---

# Groq

[Groq](https://groq.com) 使用自研 LPU 硬件，为开源模型（Llama、Gemma、Mistral 等）提供超快速推理。OpenClaw 通过其 OpenAI 兼容 API 连接到 Groq。

- 提供商：`groq`
- 认证：`GROQ_API_KEY`
- API：OpenAI 兼容

## 快速开始

1. 从 [console.groq.com/keys](https://console.groq.com/keys) 获取 API 密钥。

2. 设置 API 密钥：

```bash
export GROQ_API_KEY="gsk_..."
```

3. 设置默认模型：

```json5
{
  agents: {
    defaults: {
      model: { primary: "groq/llama-3.3-70b-versatile" },
    },
  },
}
```

## 配置文件示例

```json5
{
  env: { GROQ_API_KEY: "gsk_..." },
  agents: {
    defaults: {
      model: { primary: "groq/llama-3.3-70b-versatile" },
    },
  },
}
```

## 音频转录

Groq 还提供基于 Whisper 的快速音频转录。当配置为媒体理解提供商时，OpenClaw 使用 Groq 的 `whisper-large-v3-turbo` 模型转录语音消息。

```json5
{
  media: {
    understanding: {
      audio: {
        models: [{ provider: "groq" }],
      },
    },
  },
}
```

## 环境变量说明

如果 Gateway 网关以守护进程方式运行（launchd/systemd），请确保 `GROQ_API_KEY` 对该进程可用（例如，在 `~/.openclaw/.env` 中或通过 `env.shellEnv`）。

## 可用模型

Groq 的模型目录变化频繁。运行 `openclaw models list | grep groq` 查看当前可用模型，或访问 [console.groq.com/docs/models](https://console.groq.com/docs/models)。

常用选择包括：

- **Llama 3.3 70B Versatile** — 通用，长上下文
- **Llama 3.1 8B Instant** — 快速，轻量
- **Gemma 2 9B** — 紧凑，高效
- **Mixtral 8x7B** — MoE 架构，推理能力强

## 链接

- [Groq 控制台](https://console.groq.com)
- [API 文档](https://console.groq.com/docs)
- [模型列表](https://console.groq.com/docs/models)
- [定价](https://groq.com/pricing)
