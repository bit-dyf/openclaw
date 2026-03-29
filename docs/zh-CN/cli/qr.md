---
read_when:
  - 你想快速将 iOS 应用与 Gateway 网关配对
  - 你需要远程/手动共享的设置码输出
summary: "`openclaw qr` 的 CLI 参考（生成 iOS 配对二维码和设置码）"
title: qr
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 5ea2888291219804dee48e9e88e572d26207c33989782dd8eb1eae0dda974a4b
  source_path: cli/qr.md
  workflow: 15
---

# `openclaw qr`

根据你当前的 Gateway 网关配置生成 iOS 配对二维码和设置码。

## 用法

```bash
openclaw qr
openclaw qr --setup-code-only
openclaw qr --json
openclaw qr --remote
openclaw qr --url wss://gateway.example/ws
```

## 选项

- `--remote`：使用配置中的 `gateway.remote.url` 以及远程令牌/密码
- `--url <url>`：覆盖有效载荷中使用的 Gateway 网关 URL
- `--public-url <url>`：覆盖有效载荷中使用的公共 URL
- `--token <token>`：覆盖引导流程用于认证的 Gateway 网关令牌
- `--password <password>`：覆盖引导流程用于认证的 Gateway 网关密码
- `--setup-code-only`：仅打印设置码
- `--no-ascii`：跳过 ASCII 二维码渲染
- `--json`：以 JSON 格式输出（`setupCode`、`gatewayUrl`、`auth`、`urlSource`）

## 说明

- `--token` 和 `--password` 互斥。
- 设置码本身携带一个不透明的短效 `bootstrapToken`，而非共享的 Gateway 网关令牌/密码。
- 使用 `--remote` 时，如果有效活跃的远程凭证被配置为 SecretRef，且你未传入 `--token` 或 `--password`，命令会从活跃的 Gateway 网关快照中解析它们。若 Gateway 网关不可用，命令会快速失败。
- 不使用 `--remote` 时，若未传入 CLI 认证覆盖项，本地 Gateway 网关认证 SecretRef 会按以下规则解析：
  - `gateway.auth.token` 在令牌认证可以胜出时解析（显式设置 `gateway.auth.mode="token"` 或推断模式下无密码来源胜出）。
  - `gateway.auth.password` 在密码认证可以胜出时解析（显式设置 `gateway.auth.mode="password"` 或推断模式下认证/环境变量中无胜出令牌）。
- 如果同时配置了 `gateway.auth.token` 和 `gateway.auth.password`（包括 SecretRef），且 `gateway.auth.mode` 未设置，则设置码解析会失败，直到明确设置 mode。
- Gateway 版本差异说明：该命令路径需要支持 `secrets.resolve` 的 Gateway 网关；旧版 Gateway 网关会返回未知方法错误。
- 扫码后，使用以下命令批准设备配对：
  - `openclaw devices list`
  - `openclaw devices approve <requestId>`
