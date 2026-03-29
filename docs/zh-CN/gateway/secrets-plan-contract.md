---
read_when:
  - 生成或审查 openclaw secrets apply 计划
  - 调试"Invalid plan target path"错误
  - 了解目标类型和路径验证行为
summary: secrets apply 计划的合约：目标验证、路径匹配和 auth-profiles.json 目标范围
title: 密钥应用计划合约
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: cb89a426ca937cf4d745f641b43b330c7fbb1aa9e4359b106ecd28d7a65ca327
  source_path: gateway/secrets-plan-contract.md
  workflow: 15
---

# 密钥应用计划合约

本页定义了 `openclaw secrets apply` 强制执行的严格合约。

如果目标不符合这些规则，应用在修改配置之前会失败。

## 计划文件结构

`openclaw secrets apply --from <plan.json>` 需要一个 `targets` 数组：

```json5
{
  version: 1,
  protocolVersion: 1,
  targets: [
    {
      type: "models.providers.apiKey",
      path: "models.providers.openai.apiKey",
      pathSegments: ["models", "providers", "openai", "apiKey"],
      providerId: "openai",
      ref: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
    },
    {
      type: "auth-profiles.api_key.key",
      path: "profiles.openai:default.key",
      pathSegments: ["profiles", "openai:default", "key"],
      agentId: "main",
      ref: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
    },
  ],
}
```

## 支持的目标范围

计划目标在以下位置被接受：

- [SecretRef 凭证表面](/reference/secretref-credential-surface)

## 路径验证规则

每个目标使用以下所有规则进行验证：

- `type` 必须是已识别的目标类型。
- `path` 必须是非空的点路径。
- `pathSegments` 可以省略。如果提供，它必须规范化为与 `path` 完全相同的路径。
- 禁止的段会被拒绝：`__proto__`、`prototype`、`constructor`。
- 规范化路径必须与目标类型的注册路径结构匹配。
- `auth-profiles.json` 目标需要 `agentId`。

## 失败行为

如果目标验证失败，应用会以类似以下的错误退出：

```text
Invalid plan target path for models.providers.apiKey: models.providers.openai.baseUrl
```

无效计划不会提交任何写入。

## Exec 提供商同意行为

- `--dry-run` 默认跳过 exec SecretRef 检查。
- 包含 exec SecretRef/提供商的计划在写入模式下会被拒绝，除非设置了 `--allow-exec`。

## 操作员检查

```bash
# 在不写入的情况下验证计划
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run

# 然后实际应用
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json

# 对于包含 exec 的计划，在两种模式下都明确选择加入
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
```

## 相关文档

- [密钥管理](/gateway/secrets)
- [CLI `secrets`](/cli/secrets)
- [SecretRef 凭证表面](/reference/secretref-credential-surface)
- [配置参考](/gateway/configuration-reference)
