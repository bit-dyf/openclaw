---
read_when:
  - 为提供商凭证和 auth-profiles.json 引用配置 SecretRef
  - 在生产环境中安全地操作密钥重载、审计、配置和应用
  - 了解启动快速失败、非活跃表面过滤和最后已知良好状态行为
summary: 密钥管理：SecretRef 合约、运行时快照行为和安全单向清理
title: 密钥管理
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 077ee142db1966581caf1cee688452ac9cd16299329df28ae6a7b513e2d9a5b5
  source_path: gateway/secrets.md
  workflow: 15
---

# 密钥管理

OpenClaw 支持可添加的 SecretRef，使受支持的凭证无需以明文存储在配置中。

明文仍然有效。SecretRef 是每个凭证可选启用的。

## 目标和运行时模型

密钥被解析为内存中的运行时快照。

- 激活时急切解析，而非在请求路径上懒加载。
- 当有效活跃的 SecretRef 无法解析时，启动快速失败。
- 重载使用原子交换：完全成功，或保留最后已知良好快照。
- 运行时请求仅从活跃的内存快照中读取。
- 出站传递路径也从该活跃快照中读取（例如 Discord 回复/线程传递和 Telegram 操作发送）；它们不会在每次发送时重新解析 SecretRef。

这使密钥提供商中断与热请求路径隔离。

## 活跃表面过滤

SecretRef 仅在有效活跃的表面上验证。

- 已启用的表面：未解析的引用会阻断启动/重载。
- 非活跃表面：未解析的引用不会阻断启动/重载。
- 非活跃引用发出带代码 `SECRETS_REF_IGNORED_INACTIVE_SURFACE` 的非致命诊断信息。

非活跃表面示例：

- 已禁用的渠道/账户条目。
- 没有已启用账户继承的顶级渠道凭证。
- 已禁用的工具/功能表面。
- 未被 `tools.web.search.provider` 选中的网络搜索提供商专用密钥。

## SecretRef 合约

在所有地方使用同一个对象结构：

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

### `source: "env"`

```json5
{ source: "env", provider: "default", id: "OPENAI_API_KEY" }
```

### `source: "file"`

```json5
{ source: "file", provider: "filemain", id: "/providers/openai/apiKey" }
```

### `source: "exec"`

```json5
{ source: "exec", provider: "vault", id: "providers/openai/apiKey" }
```

## 提供商配置

在 `secrets.providers` 下定义提供商：

```json5
{
  secrets: {
    providers: {
      default: { source: "env" },
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json",
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        args: ["--profile", "prod"],
        passEnv: ["PATH", "VAULT_ADDR"],
        jsonOnly: true,
      },
    },
  },
}
```

## Exec 集成示例

### 1Password CLI

```json5
{
  secrets: {
    providers: {
      onepassword_openai: {
        source: "exec",
        command: "/opt/homebrew/bin/op",
        allowSymlinkCommand: true,
        trustedDirs: ["/opt/homebrew"],
        args: ["read", "op://Personal/OpenClaw QA API Key/password"],
        passEnv: ["HOME"],
        jsonOnly: false,
      },
    },
  },
}
```

### HashiCorp Vault CLI

```json5
{
  secrets: {
    providers: {
      vault_openai: {
        source: "exec",
        command: "/opt/homebrew/bin/vault",
        allowSymlinkCommand: true,
        trustedDirs: ["/opt/homebrew"],
        args: ["kv", "get", "-field=OPENAI_API_KEY", "secret/openclaw"],
        passEnv: ["VAULT_ADDR", "VAULT_TOKEN"],
        jsonOnly: false,
      },
    },
  },
}
```

## 审计和配置工作流

默认操作员流程：

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets audit --check
```

### `secrets audit`

审计发现包括：

- 静态存储的明文值（`openclaw.json`、`auth-profiles.json`、`.env` 和生成的 `agents/*/agent/models.json`）
- 未解析的引用
- 优先级遮蔽（`auth-profiles.json` 优先于 `openclaw.json` 引用）
- 旧版残留（`auth.json`、OAuth 提醒）

### `secrets configure`

交互式辅助工具：

- 首先配置 `secrets.providers`（`env`/`file`/`exec`，添加/编辑/删除）
- 让你为 `openclaw.json` 及 `auth-profiles.json` 中受支持的密钥承载字段选择 SecretRef
- 运行预检解析
- 可立即应用

## 相关文档

- CLI 命令：[secrets](/cli/secrets)
- 计划合约详情：[密钥应用计划合约](/gateway/secrets-plan-contract)
- 凭证表面：[SecretRef 凭证表面](/reference/secretref-credential-surface)
- 认证设置：[认证](/gateway/authentication)
- 环境优先级：[环境变量](/help/environment)
