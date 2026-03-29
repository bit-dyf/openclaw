---
read_when:
  - 你想为本地 OpenClaw 状态创建完整的备份归档
  - 你想在重置或卸载前预览将包含哪些路径
summary: "`openclaw backup` 的 CLI 参考（创建本地备份归档）"
title: backup
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 4ee33408825acdb3983ed5ddf57610062730fa18d079bbad902c272938393c54
  source_path: cli/backup.md
  workflow: 15
---

# `openclaw backup`

为 OpenClaw 的状态、配置、凭证、会话以及可选的工作区创建本地备份归档。

```bash
openclaw backup create
openclaw backup create --output ~/Backups
openclaw backup create --dry-run --json
openclaw backup create --verify
openclaw backup create --no-include-workspace
openclaw backup create --only-config
openclaw backup verify ./2026-03-09T00-00-00.000Z-openclaw-backup.tar.gz
```

## 说明

- 归档中包含一个 `manifest.json` 文件，记录了已解析的源路径和归档布局。
- 默认输出为当前工作目录下带时间戳的 `.tar.gz` 归档。
- 如果当前工作目录位于已备份的源目录树中，OpenClaw 会回退到主目录作为默认归档位置。
- 已有的归档文件不会被覆盖。
- 指向源状态/工作区树内部的输出路径会被拒绝，以避免自我包含。
- `openclaw backup verify <archive>` 用于验证归档是否包含且仅包含一个根清单，拒绝带有目录穿越的归档路径，并检查清单声明的每个有效负载是否存在于 tar 包中。
- `openclaw backup create --verify` 在写入归档后立即运行该验证。
- `openclaw backup create --only-config` 仅备份当前活跃的 JSON 配置文件。

## 备份内容

`openclaw backup create` 从你本地的 OpenClaw 安装中规划备份来源：

- OpenClaw 本地状态解析器返回的状态目录，通常为 `~/.openclaw`
- 活跃配置文件路径
- OAuth / 凭证目录
- 从当前配置中发现的工作区目录，除非传入 `--no-include-workspace`

如果使用 `--only-config`，OpenClaw 会跳过状态、凭证和工作区发现，仅归档活跃配置文件路径。

OpenClaw 在构建归档前会规范化路径。如果配置、凭证或工作区已经位于状态目录内部，则不会作为单独的顶层备份来源重复添加。缺失的路径会被跳过。

归档有效负载存储来自这些源目录树的文件内容，嵌入的 `manifest.json` 记录已解析的绝对源路径以及每个资产所使用的归档布局。

## 配置无效时的行为

`openclaw backup` 有意绕过正常的配置预检，以便在恢复期间仍能提供帮助。由于工作区发现依赖于有效的配置，当配置文件存在但无效且仍启用工作区备份时，`openclaw backup create` 会快速失败。

如果你仍需要在该情况下进行部分备份，请重新运行：

```bash
openclaw backup create --no-include-workspace
```

这样会保留状态、配置和凭证的备份范围，同时完全跳过工作区发现。

如果你只需要配置文件本身的副本，`--only-config` 在配置格式错误时也有效，因为它不依赖解析配置来发现工作区。

## 大小与性能

OpenClaw 不强制执行内置的最大备份大小或单文件大小限制。

实际限制来自本地机器和目标文件系统：

- 可用空间（临时归档写入加上最终归档）
- 遍历大型工作区目录树并将其压缩为 `.tar.gz` 所需的时间
- 如果使用 `openclaw backup create --verify` 或运行 `openclaw backup verify`，重新扫描归档所需的时间
- 目标路径的文件系统行为。OpenClaw 优先使用无覆盖的硬链接发布步骤，当不支持硬链接时回退到排他性复制

大型工作区通常是归档大小的主要驱动因素。如果希望备份更小或更快，请使用 `--no-include-workspace`。

如需最小归档，使用 `--only-config`。
