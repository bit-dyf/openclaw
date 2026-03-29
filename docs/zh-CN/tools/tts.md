---
read_when:
  - 为回复启用文字转语音功能
  - 配置 TTS 提供商或限制
  - 使用 /tts 命令
summary: 用于出站回复的文字转语音（TTS）
title: 文字转语音
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: b646dac9123d0273fac52145cc749aefc24d5456e68690bd16a30cebda52e9e8
  source_path: tools/tts.md
  workflow: 15
---

# 文字转语音（TTS）

OpenClaw 可以使用 ElevenLabs、Microsoft 或 OpenAI 将出站回复转换为音频。
它在 OpenClaw 能发送音频的任何地方均可使用。

## 支持的服务

- **ElevenLabs**（主提供商或回退提供商）
- **Microsoft**（主提供商或回退提供商；当前内置实现使用 `node-edge-tts`）
- **OpenAI**（主提供商或回退提供商；也用于摘要）

### Microsoft 语音说明

内置的 Microsoft 语音提供商当前通过 `node-edge-tts` 库使用 Microsoft Edge 的在线神经 TTS 服务。这是一个托管服务（非本地），使用 Microsoft 端点，无需 API 密钥。`node-edge-tts` 暴露了语音配置选项和输出格式，但不是所有选项都被该服务支持。使用 `edge` 的旧版配置和指令输入仍然有效，并已规范化为 `microsoft`。

由于此路径是没有已发布 SLA 或配额的公共 Web 服务，请将其视为尽力而为的服务。如果你需要有保障的限制和支持，请使用 OpenAI 或 ElevenLabs。

## 可选密钥

如果你想使用 OpenAI 或 ElevenLabs：

- `ELEVENLABS_API_KEY`（或 `XI_API_KEY`）
- `OPENAI_API_KEY`

Microsoft 语音**不需要** API 密钥。

如果配置了多个提供商，将优先使用所选提供商，其他为回退选项。
自动摘要使用已配置的 `summaryModel`（或 `agents.defaults.model.primary`），
因此如果你启用摘要，该提供商也必须已通过认证。

## 服务链接

- [OpenAI 文字转语音指南](https://platform.openai.com/docs/guides/text-to-speech)
- [OpenAI 音频 API 参考](https://platform.openai.com/docs/api-reference/audio)
- [ElevenLabs 文字转语音](https://elevenlabs.io/docs/api-reference/text-to-speech)
- [ElevenLabs 认证](https://elevenlabs.io/docs/api-reference/authentication)
- [node-edge-tts](https://github.com/SchneeHertz/node-edge-tts)
- [Microsoft 语音输出格式](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech#audio-outputs)

## 默认是否启用？

否。自动 TTS **默认关闭**。在配置中通过 `messages.tts.auto` 启用，或通过 `/tts always`（别名：`/tts on`）按会话启用。

当 `messages.tts.provider` 未设置时，OpenClaw 按注册表自动选择顺序选择第一个已配置的语音提供商。

## 配置

TTS 配置位于 `openclaw.json` 的 `messages.tts` 下。
完整 schema 参见 [Gateway 网关配置](/gateway/configuration)。

### 最小配置（启用 + 提供商）

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "elevenlabs",
    },
  },
}
```

### OpenAI 主提供商加 ElevenLabs 回退

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "openai",
      summaryModel: "openai/gpt-4.1-mini",
      modelOverrides: {
        enabled: true,
      },
      providers: {
        openai: {
          apiKey: "openai_api_key",
          baseUrl: "https://api.openai.com/v1",
          model: "gpt-4o-mini-tts",
          voice: "alloy",
        },
        elevenlabs: {
          apiKey: "elevenlabs_api_key",
          baseUrl: "https://api.elevenlabs.io",
          voiceId: "voice_id",
          modelId: "eleven_multilingual_v2",
          seed: 42,
          applyTextNormalization: "auto",
          languageCode: "en",
          voiceSettings: {
            stability: 0.5,
            similarityBoost: 0.75,
            style: 0.0,
            useSpeakerBoost: true,
            speed: 1.0,
          },
        },
      },
    },
  },
}
```

### Microsoft 主提供商（无需 API 密钥）

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "microsoft",
      providers: {
        microsoft: {
          enabled: true,
          voice: "en-US-MichelleNeural",
          lang: "en-US",
          outputFormat: "audio-24khz-48kbitrate-mono-mp3",
          rate: "+10%",
          pitch: "-5%",
        },
      },
    },
  },
}
```

### 禁用 Microsoft 语音

```json5
{
  messages: {
    tts: {
      providers: {
        microsoft: {
          enabled: false,
        },
      },
    },
  },
}
```

## 斜杠命令用法

只有一个命令：`/tts`。
启用详情参见[斜杠命令](/tools/slash-commands)。

Discord 说明：`/tts` 是 Discord 内置命令，因此 OpenClaw 在该平台注册 `/voice` 作为原生命令。文本 `/tts ...` 仍然有效。

```
/tts off
/tts always
/tts inbound
/tts tagged
/tts status
/tts provider openai
/tts limit 2000
/tts summary off
/tts audio Hello from OpenClaw
```

## 智能体工具

`tts` 工具将文本转换为语音，并返回音频附件用于回复传递。当渠道为飞书、Matrix、Telegram 或 WhatsApp 时，音频以语音消息而非文件附件的形式传递。

## Gateway 网关 RPC

Gateway 网关方法：

- `tts.status`
- `tts.enable`
- `tts.disable`
- `tts.convert`
- `tts.setProvider`
- `tts.providers`
