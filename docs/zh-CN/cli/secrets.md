---
read_when:
  - 在运行时重新解析 secret 引用
  - 审计明文残留和未解析的引用
  - 配置 SecretRef 并执行单向清除操作
summary: "`openclaw secrets` 的 CLI 参考（重载、审计、配置、应用）"
title: secrets
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 9047c981289826186134757cb9b3bc966141df29496f1ef507f7fa50b64dd2ef
  source_path: cli/secrets.md
  workflow: 15
---

# `openclaw secrets`

使用 `openclaw secrets` 管理 SecretRef，并保持活跃运行时快照的健康状态。

命令职责：

- `reload`：执行 Gateway 网关 RPC（`secrets.reload`），重新解析引用，仅在完全成功时交换运行时快照（不写入配置）。
- `audit`：只读扫描配置/认证/生成模型存储及旧版残留，检查明文、未解析引用和优先级漂移（默认跳过 exec 引用，除非设置 `--allow-exec`）。
- `configure`：交互式规划器，用于提供商设置、目标映射和预检（需要 TTY）。
- `apply`：执行已保存的计划（`--dry-run` 仅用于验证；dry-run 默认跳过 exec 检查，且写入模式会拒绝包含 exec 的计划，除非设置 `--allow-exec`），然后清除目标明文残留。

推荐的操作流程：

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets audit --check
openclaw secrets reload
```

如果你的计划包含 `exec` SecretRef / 提供商，请在 dry-run 和写入 apply 命令中都传入 `--allow-exec`。

CI/门控的退出码说明：

- `audit --check` 在发现问题时返回 `1`。
- 未解析引用返回 `2`。

相关内容：

- Secrets 指南：[Secrets 管理](/gateway/secrets)
- 凭证界面：[SecretRef 凭证界面](/reference/secretref-credential-surface)
- 安全指南：[安全](/gateway/security)

## 重载运行时快照

重新解析 secret 引用并原子性地交换运行时快照。

```bash
openclaw secrets reload
openclaw secrets reload --json
```

说明：

- 使用 Gateway 网关 RPC 方法 `secrets.reload`。
- 如果解析失败，Gateway 网关保留上一个已知良好的快照并返回错误（不进行部分激活）。
- JSON 响应包含 `warningCount`。

## 审计

扫描 OpenClaw 状态，检查：

- 明文 secret 存储
- 未解析引用
- 优先级漂移（`auth-profiles.json` 中的凭证覆盖了 `openclaw.json` 中的引用）
- 生成的 `agents/*/agent/models.json` 残留（提供商 `apiKey` 值和敏感提供商请求头）
- 旧版残留（旧版认证存储条目、OAuth 提醒）

请求头残留说明：

- 敏感提供商请求头检测基于名称启发式（常见认证/凭证请求头名称和片段，如 `authorization`、`x-api-key`、`token`、`secret`、`password` 和 `credential`）。

```bash
openclaw secrets audit
openclaw secrets audit --check
openclaw secrets audit --json
openclaw secrets audit --allow-exec
```

退出行为：

- `--check` 在发现问题时以非零值退出。
- 未解析引用以更高优先级的非零代码退出。

报告结构要点：

- `status`：`clean | findings | unresolved`
- `resolution`：`refsChecked`、`skippedExecRefs`、`resolvabilityComplete`
- `summary`：`plaintextCount`、`unresolvedRefCount`、`shadowedRefCount`、`legacyResidueCount`
- 问题代码：
  - `PLAINTEXT_FOUND`
  - `REF_UNRESOLVED`
  - `REF_SHADOWED`
  - `LEGACY_RESIDUE`

## 配置（交互式辅助工具）

交互式构建提供商和 SecretRef 变更，运行预检，并可选地应用：

```bash
openclaw secrets configure
openclaw secrets configure --plan-out /tmp/openclaw-secrets-plan.json
openclaw secrets configure --apply --yes
openclaw secrets configure --providers-only
openclaw secrets configure --skip-provider-setup
openclaw secrets configure --agent ops
openclaw secrets configure --json
```

流程：

- 首先是提供商设置（为 `secrets.providers` 别名添加/编辑/删除）。
- 其次是凭证映射（选择字段并分配 `{source, provider, id}` 引用）。
- 最后是预检和可选应用。

标志：

- `--providers-only`：仅配置 `secrets.providers`，跳过凭证映射。
- `--skip-provider-setup`：跳过提供商设置，将凭证映射到现有提供商。
- `--agent <id>`：将 `auth-profiles.json` 目标发现和写入范围限定到一个智能体存储。
- `--allow-exec`：允许在预检/应用期间进行 exec SecretRef 检查（可能会执行提供商命令）。

说明：

- 需要交互式 TTY。
- 不能同时使用 `--providers-only` 和 `--skip-provider-setup`。
- `configure` 的目标是 `openclaw.json` 中的 secret 承载字段，以及所选智能体范围内的 `auth-profiles.json`。
- `configure` 支持在选择器流程中直接创建新的 `auth-profiles.json` 映射。
- 规范支持的界面：[SecretRef 凭证界面](/reference/secretref-credential-surface)。
- 它在应用前会执行预检解析。
- 如果预检/应用包含 exec 引用，两个步骤都需要保持 `--allow-exec`。
- 生成的计划默认启用清除选项（`scrubEnv`、`scrubAuthProfilesForProviderTargets`、`scrubLegacyAuthJson` 均启用）。
- 应用路径对已清除的明文值是单向的。
- 不带 `--apply` 时，CLI 在预检后仍会提示"立即应用此计划？"。
- 带 `--apply`（不带 `--yes`）时，CLI 会提示额外的不可逆确认。

exec 提供商安全说明：

- Homebrew 安装通常会在 `/opt/homebrew/bin/*` 下暴露符号链接的二进制文件。
- 仅在受信任的包管理器路径需要时才设置 `allowSymlinkCommand: true`，并配合 `trustedDirs`（例如 `["/opt/homebrew"]`）使用。
- 在 Windows 上，如果提供商路径的 ACL 验证不可用，OpenClaw 会直接失败。仅对受信任的路径，才在该提供商上设置 `allowInsecurePath: true` 来绕过路径安全检查。

## 应用已保存的计划

应用或预检之前生成的计划：

```bash
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --json
```

exec 行为：

- `--dry-run` 验证预检，不写入文件。
- dry-run 中默认跳过 exec SecretRef 检查。
- 写入模式会拒绝包含 exec SecretRef / 提供商的计划，除非设置 `--allow-exec`。
- 使用 `--allow-exec` 可在任一模式下选择加入 exec 提供商检查/执行。

计划合同详细信息（允许的目标路径、验证规则和失败语义）：

- [Secrets 应用计划合同](/gateway/secrets-plan-contract)

`apply` 可能更新的内容：

- `openclaw.json`（SecretRef 目标 + 提供商 upsert/删除）
- `auth-profiles.json`（提供商目标清除）
- 旧版 `auth.json` 残留
- `~/.openclaw/.env` 中值已迁移的已知 secret 键

## 为何没有回滚备份

`secrets apply` 有意不写入包含旧明文值的回滚备份。

安全性来自严格的预检 + 准原子性应用（失败时尽力进行内存恢复）。

## 示例

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets audit --check
```

如果 `audit --check` 仍报告明文问题，请更新报告中剩余的目标路径并重新运行 audit。
