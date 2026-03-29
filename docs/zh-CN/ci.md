---
title: CI 流水线
summary: "CI 任务图、范围门控及本地等效命令"
read_when:
  - 需要了解 CI 任务为何运行或未运行
  - 正在调试失败的 GitHub Actions 检查
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 6df608d301136a8aa9d89103635ea3c3240ac62310e9ed520ab256de1154157e
  source_path: ci.md
  workflow: 15
---

# CI 流水线

CI 在每次推送到 `main` 分支及每个 Pull Request 时运行。它使用智能范围检测，当仅有无关区域发生变更时跳过高成本任务。

## 任务概览

| 任务               | 用途                                                                     | 运行时机                                   |
| ------------------ | ------------------------------------------------------------------------ | ------------------------------------------ |
| `preflight`        | 文档范围、变更范围、密钥扫描、工作流审计、生产依赖审计 | 始终；仅在非文档变更时进行基于 node 的审计 |
| `docs-scope`       | 检测仅文档变更                                                           | 始终                                       |
| `changed-scope`    | 检测哪些区域发生了变更（node/macos/android/windows）                     | 非文档变更                                 |
| `check`            | TypeScript 类型、lint、格式                                              | 非文档变更，node 相关变更                  |
| `check-docs`       | Markdown lint + 断链检查                                                 | 文档发生变更                               |
| `secrets`          | 检测泄露的密钥                                                           | 始终                                       |
| `build-artifacts`  | 构建 dist 一次，与 `release-check` 共享                                  | 推送到 `main`，node 相关变更               |
| `release-check`    | 验证 npm pack 内容                                                       | 构建后推送到 `main`                        |
| `checks`           | PR 的 Node 测试 + 协议检查；推送时的 Bun 兼容性                          | 非文档变更，node 相关变更                  |
| `compat-node22`    | 最低支持的 Node 运行时兼容性                                             | 推送到 `main`，node 相关变更               |
| `checks-windows`   | Windows 专项测试                                                         | 非文档变更，Windows 相关变更               |
| `macos`            | Swift lint/构建/测试 + TS 测试                                           | PR 包含 macOS 相关变更                     |
| `android`          | Gradle 构建 + 测试                                                       | 非文档变更，Android 相关变更               |

## 快速失败顺序

任务经过排序，以便低成本检查在高成本任务运行前先行失败：

1. `docs-scope` + `changed-scope` + `check` + `secrets`（并行，低成本门控优先）
2. PR：`checks`（Linux Node 测试拆分为 2 个分片）、`checks-windows`、`macos`、`android`
3. 推送到 `main`：`build-artifacts` + `release-check` + Bun 兼容性 + `compat-node22`

范围逻辑位于 `scripts/ci-changed-scope.mjs`，单元测试在 `src/scripts/ci-changed-scope.test.ts` 中。
同一共享范围模块还通过较窄的 `changed-smoke` 门控驱动独立的 `install-smoke` 工作流，因此 Docker/安装冒烟测试仅在安装、打包和容器相关变更时运行。

## 运行器

| 运行器                           | 任务                                |
| -------------------------------- | ----------------------------------- |
| `blacksmith-16vcpu-ubuntu-2404`  | 大多数 Linux 任务，包括范围检测     |
| `blacksmith-32vcpu-windows-2025` | `checks-windows`                    |
| `macos-latest`                   | `macos`、`ios`                      |

## 本地等效命令

```bash
pnpm check          # 类型 + lint + 格式
pnpm test           # vitest 测试
pnpm check:docs     # 文档格式 + lint + 断链检查
pnpm release:check  # 验证 npm pack
```
