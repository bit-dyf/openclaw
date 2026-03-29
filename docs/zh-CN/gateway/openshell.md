---
read_when:
  - 你想使用云管理沙箱代替本地 Docker
  - 你在设置 OpenShell 插件
  - 你需要在镜像模式和远程工作区模式之间进行选择
summary: 将 OpenShell 用作 OpenClaw 智能体的托管沙箱后端
title: OpenShell
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: aaf9027d0632a70fb86455f8bc46dc908ff766db0eb0cdf2f7df39c715241ead
  source_path: gateway/openshell.md
  workflow: 15
---

# OpenShell

OpenShell 是 OpenClaw 的托管沙箱后端。OpenClaw 将沙箱生命周期委托给 `openshell` CLI，而不是在本地运行 Docker 容器，后者通过 SSH 命令执行提供远程环境。

OpenShell 插件复用了与通用 [SSH 后端](/gateway/sandboxing#ssh-backend) 相同的核心 SSH 传输和远程文件系统桥接。它增加了 OpenShell 专用的生命周期管理（`sandbox create/get/delete`、`sandbox ssh-config`）以及可选的 `mirror` 工作区模式。

## 先决条件

- `openshell` CLI 已安装并在 `PATH` 中（或通过 `plugins.entries.openshell.config.command` 设置自定义路径）
- 具有沙箱访问权限的 OpenShell 账户
- 在主机上运行的 OpenClaw Gateway 网关

## 快速开始

1. 启用插件并设置沙箱后端：

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "session",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
        },
      },
    },
  },
}
```

2. 重启 Gateway 网关。下次智能体轮次时，OpenClaw 会创建 OpenShell 沙箱并通过其路由工具执行。

3. 验证：

```bash
openclaw sandbox list
openclaw sandbox explain
```

## 工作区模式

这是使用 OpenShell 时最重要的决定。

### `mirror`（镜像）

当你希望**本地工作区保持规范状态**时，使用 `plugins.entries.openshell.config.mode: "mirror"`。

行为：

- 在 `exec` 之前，OpenClaw 将本地工作区同步到 OpenShell 沙箱。
- 在 `exec` 之后，OpenClaw 将远程工作区同步回本地工作区。
- 文件工具仍通过沙箱桥接操作，但本地工作区在各轮次之间仍是真实来源。

适用场景：

- 你在 OpenClaw 外部本地编辑文件，并希望这些更改在沙箱中自动可见。
- 你希望 OpenShell 沙箱尽可能像 Docker 后端那样运行。
- 你希望主机工作区在每次执行轮次后反映沙箱写入。

权衡：每次执行前后都有额外的同步开销。

### `remote`（远程）

当你希望 **OpenShell 工作区成为规范状态**时，使用 `plugins.entries.openshell.config.mode: "remote"`。

行为：

- 沙箱首次创建时，OpenClaw 从本地工作区单次播种远程工作区。
- 此后，`exec`、`read`、`write`、`edit` 和 `apply_patch` 直接在远程 OpenShell 工作区上操作。
- OpenClaw **不**将远程更改同步回本地工作区。
- 提示时媒体读取仍然有效，因为文件和媒体工具通过沙箱桥接读取。

适用场景：

- 沙箱应主要存在于远程端。
- 你希望降低每轮次的同步开销。
- 你不希望主机本地编辑悄悄覆盖远程沙箱状态。

重要：如果你在初始播种后在主机上的 OpenClaw 外部编辑文件，远程沙箱**不会**看到这些更改。使用 `openclaw sandbox recreate` 重新播种。

### 选择模式

|                      | `mirror`（镜像）               | `remote`（远程）               |
| -------------------- | ------------------------------ | ------------------------------ |
| **规范工作区**       | 本地主机                       | 远程 OpenShell                 |
| **同步方向**         | 双向（每次执行）               | 单次播种                       |
| **每轮次开销**       | 较高（上传 + 下载）            | 较低（直接远程操作）           |
| **本地编辑是否可见** | 是，在下次执行时               | 否，直到重新创建               |
| **适用场景**         | 开发工作流                     | 长时运行智能体、CI             |

## 配置参考

所有 OpenShell 配置位于 `plugins.entries.openshell.config` 下：

| 键                        | 类型                     | 默认值        | 描述                                                  |
| ------------------------- | ------------------------ | ------------- | ----------------------------------------------------- |
| `mode`                    | `"mirror"` 或 `"remote"` | `"mirror"`    | 工作区同步模式                                        |
| `command`                 | `string`                 | `"openshell"` | `openshell` CLI 的路径或名称                          |
| `from`                    | `string`                 | `"openclaw"`  | 首次创建的沙箱来源                                    |
| `gateway`                 | `string`                 | —             | OpenShell 网关名称（`--gateway`）                     |
| `gatewayEndpoint`         | `string`                 | —             | OpenShell 网关端点 URL（`--gateway-endpoint`）        |
| `policy`                  | `string`                 | —             | 沙箱创建的 OpenShell 策略 ID                          |
| `providers`               | `string[]`               | `[]`          | 沙箱创建时要附加的提供商名称                          |
| `gpu`                     | `boolean`                | `false`       | 请求 GPU 资源                                         |
| `autoProviders`           | `boolean`                | `true`        | 沙箱创建时传递 `--auto-providers`                     |
| `remoteWorkspaceDir`      | `string`                 | `"/sandbox"`  | 沙箱内的主要可写工作区                                |
| `remoteAgentWorkspaceDir` | `string`                 | `"/agent"`    | 智能体工作区挂载路径（用于只读访问）                  |
| `timeoutSeconds`          | `number`                 | `120`         | `openshell` CLI 操作的超时时间                        |

## 另请参阅

- [沙箱](/gateway/sandboxing) — 模式、范围和后端比较
- [沙箱 vs 工具策略 vs 提升权限](/gateway/sandbox-vs-tool-policy-vs-elevated) — 调试阻断的工具
- [多智能体沙箱和工具](/tools/multi-agent-sandbox-tools) — 每智能体覆盖
- [沙箱 CLI](/cli/sandbox) — `openclaw sandbox` 命令
