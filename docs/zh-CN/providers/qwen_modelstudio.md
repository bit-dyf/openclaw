---
read_when:
  - 你想在 OpenClaw 中使用 Qwen（阿里云模型工坊）
  - 你需要模型工坊的 API 密钥环境变量
  - 你想使用标准（按量付费）或编程计划端点
summary: 阿里云模型工坊设置（标准按量付费和编程计划、双区域端点）
title: Qwen / 模型工坊
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: f7cf566fada88d400c1927aa338f92073e467e1e8a32a82badff30a0c2982dad
  source_path: providers/qwen_modelstudio.md
  workflow: 15
---

# Qwen / 模型工坊（阿里云）

模型工坊提供商可访问阿里云模型，包括 Qwen 以及托管在该平台上的第三方模型。支持两种计费方案：**标准**（按量付费）和**编程计划**（订阅制）。

- 提供商：`modelstudio`
- 认证：`MODELSTUDIO_API_KEY`
- API：OpenAI 兼容

## 快速开始

### 标准（按量付费）

```bash
# 国内端点
openclaw onboard --auth-choice modelstudio-standard-api-key-cn

# 全球/国际端点
openclaw onboard --auth-choice modelstudio-standard-api-key
```

### 编程计划（订阅制）

```bash
# 国内端点
openclaw onboard --auth-choice modelstudio-api-key-cn

# 全球/国际端点
openclaw onboard --auth-choice modelstudio-api-key
```

新手引导完成后，设置默认模型：

```json5
{
  agents: {
    defaults: {
      model: { primary: "modelstudio/qwen3.5-plus" },
    },
  },
}
```

## 计划类型与端点

| 计划                     | 区域   | 认证选项                          | 端点                                             |
| ------------------------ | ------ | --------------------------------- | ------------------------------------------------ |
| 标准（按量付费）         | 国内   | `modelstudio-standard-api-key-cn` | `dashscope.aliyuncs.com/compatible-mode/v1`      |
| 标准（按量付费）         | 全球   | `modelstudio-standard-api-key`    | `dashscope-intl.aliyuncs.com/compatible-mode/v1` |
| 编程计划（订阅制）       | 国内   | `modelstudio-api-key-cn`          | `coding.dashscope.aliyuncs.com/v1`               |
| 编程计划（订阅制）       | 全球   | `modelstudio-api-key`             | `coding-intl.dashscope.aliyuncs.com/v1`          |

提供商根据你的认证选项自动选择端点。你可以在配置中通过自定义 `baseUrl` 进行覆盖。

## 获取 API 密钥

- **国内**：[bailian.console.aliyun.com](https://bailian.console.aliyun.com/)
- **全球/国际**：[modelstudio.console.alibabacloud.com](https://modelstudio.console.alibabacloud.com/)

## 可用模型

- **qwen3.5-plus**（默认）— Qwen 3.5 Plus
- **qwen3-coder-plus**、**qwen3-coder-next** — Qwen 编程模型
- **GLM-5** — 通过阿里巴巴的 GLM 模型
- **Kimi K2.5** — 通过阿里巴巴的 Moonshot AI
- **MiniMax-M2.7** — 通过阿里巴巴的 MiniMax

部分模型（qwen3.5-plus、kimi-k2.5）支持图像输入。上下文窗口从 200K 到 1M token 不等。

## 环境变量说明

如果 Gateway 网关以守护进程方式运行（launchd/systemd），请确保 `MODELSTUDIO_API_KEY` 对该进程可用（例如，在 `~/.openclaw/.env` 中或通过 `env.shellEnv`）。
