---
read_when:
  - 你仍在脚本中使用 `openclaw daemon ...`
  - 你需要服务生命周期命令（install/start/stop/restart/status）
summary: "`openclaw daemon` 的 CLI 参考（Gateway 网关服务管理的旧版别名）"
title: daemon
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 8d40ec5acbe813c339cf952b642cee1892379ec510b2b8ddcc81c73aba319e3d
  source_path: cli/daemon.md
  workflow: 15
---

# `openclaw daemon`

Gateway 网关服务管理命令的旧版别名。

`openclaw daemon ...` 与 `openclaw gateway ...` 服务命令对应的服务控制界面行为相同。

## 用法

```bash
openclaw daemon status
openclaw daemon install
openclaw daemon start
openclaw daemon stop
openclaw daemon restart
openclaw daemon uninstall
```

## 子命令

- `status`：显示服务安装状态并探测 Gateway 网关健康状况
- `install`：安装服务（`launchd` / `systemd` / `schtasks`）
- `uninstall`：卸载服务
- `start`：启动服务
- `stop`：停止服务
- `restart`：重启服务

## 常用选项

- `status`：`--url`、`--token`、`--password`、`--timeout`、`--no-probe`、`--require-rpc`、`--deep`、`--json`
- `install`：`--port`、`--runtime <node|bun>`、`--token`、`--force`、`--json`
- 生命周期命令（`uninstall|start|stop|restart`）：`--json`

说明：

- `status` 在可能时会为探测认证解析已配置的 SecretRef。
- 如果在该命令路径中有必需的 auth SecretRef 无法解析，且探测连接/认证失败，`daemon status --json` 会报告 `rpc.authWarning`；请显式传入 `--token` / `--password` 或先解析 secret 来源。
- 如果探测成功，未解析的 auth-ref 警告会被抑制，以避免误报。
- 在 Linux systemd 安装中，`status` 的令牌漂移检查同时包括 `Environment=` 和 `EnvironmentFile=` unit 来源。
- 当令牌认证需要令牌且 `gateway.auth.token` 由 SecretRef 管理时，`install` 会验证 SecretRef 是否可解析，但不会将解析后的令牌持久化到服务环境元数据中。
- 如果令牌认证需要令牌且已配置的令牌 SecretRef 无法解析，安装会直接失败。
- 如果同时配置了 `gateway.auth.token` 和 `gateway.auth.password`（包括 SecretRef），且 `gateway.auth.mode` 未设置，则安装会被阻止，直到明确设置 mode。

## 建议使用

请使用 [`openclaw gateway`](/cli/gateway) 获取当前文档和示例。
