---
read_when:
  - 你想在 OpenClaw 中使用 DeepSeek
  - 你需要 API 密钥环境变量或 CLI 认证选项
summary: DeepSeek 设置（认证 + 模型选择）
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: c43199a17dd473c41696f6e8228d52356990d3e90d640e35401159c92f2885e2
  source_path: providers/deepseek.md
  workflow: 15
---

# DeepSeek

[DeepSeek](https://www.deepseek.com) 提供具有 OpenAI 兼容 API 的强大 AI 模型。

- 提供商：`deepseek`
- 认证：`DEEPSEEK_API_KEY`
- API：OpenAI 兼容

## 快速开始

设置 API 密钥（推荐：为 Gateway 网关存储密钥）：

```bash
openclaw onboard --auth-choice deepseek-api-key
```

系统将提示你输入 API 密钥，并将 `deepseek/deepseek-chat` 设置为默认模型。

## 非交互式示例

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice deepseek-api-key \
  --deepseek-api-key "$DEEPSEEK_API_KEY" \
  --skip-health \
  --accept-risk
```

## 环境变量说明

如果 Gateway 网关以守护进程方式运行（launchd/systemd），请确保 `DEEPSEEK_API_KEY` 对该进程可用（例如，在 `~/.openclaw/.env` 中或通过 `env.shellEnv`）。

## 可用模型

| 模型 ID             | 名称                     | 类型     | 上下文长度 |
| ------------------- | ------------------------ | -------- | ---------- |
| `deepseek-chat`     | DeepSeek Chat (V3.2)     | 通用     | 128K       |
| `deepseek-reasoner` | DeepSeek Reasoner (V3.2) | 推理     | 128K       |

- **deepseek-chat** 对应非思维模式下的 DeepSeek-V3.2。
- **deepseek-reasoner** 对应带链式思维推理的思维模式下的 DeepSeek-V3.2。

在 [platform.deepseek.com](https://platform.deepseek.com/api_keys) 获取你的 API 密钥。
