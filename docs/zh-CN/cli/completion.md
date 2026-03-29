---
read_when:
  - 你想为 zsh/bash/fish/PowerShell 启用 shell 补全
  - 你需要在 OpenClaw 状态目录下缓存补全脚本
summary: "`openclaw completion` 的 CLI 参考（生成/安装 shell 补全脚本）"
title: completion
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 7bbf140a880bafdb7140149f85465d66d0d46e5a3da6a1e41fb78be2fd2bd4d0
  source_path: cli/completion.md
  workflow: 15
---

# `openclaw completion`

生成 shell 补全脚本，并可选地将其安装到你的 shell 配置文件中。

## 用法

```bash
openclaw completion
openclaw completion --shell zsh
openclaw completion --install
openclaw completion --shell fish --install
openclaw completion --write-state
openclaw completion --shell bash --write-state
```

## 选项

- `-s, --shell <shell>`：目标 shell（`zsh`、`bash`、`powershell`、`fish`；默认：`zsh`）
- `-i, --install`：通过在 shell 配置文件中添加 source 行来安装补全
- `--write-state`：将补全脚本写入 `$OPENCLAW_STATE_DIR/completions`，不打印到 stdout
- `-y, --yes`：跳过安装确认提示

## 说明

- `--install` 会在你的 shell 配置文件中写入一个小型"OpenClaw Completion"代码块，并指向缓存的脚本。
- 不带 `--install` 或 `--write-state` 时，命令会将脚本打印到 stdout。
- 补全生成会主动加载命令树，因此包含嵌套子命令。
