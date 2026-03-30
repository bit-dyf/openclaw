---
read_when:
  - 在 WSL2 中运行 OpenClaw Gateway 而 Chrome 在 Windows 上
  - 看到跨 WSL2 和 Windows 的浏览器/控制界面错误叠加
  - 在分机部署中决定是使用本机 Chrome MCP 还是原始远程 CDP
summary: 分层排查 WSL2 Gateway + Windows Chrome 远程 CDP
title: WSL2 + Windows + 远程 Chrome CDP 故障排查
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 76a6db5ee93dce528538f460a7e3b8def3ee29b2293aff69c0d224a266dc2e0f
  source_path: tools/browser-wsl2-windows-remote-cdp-troubleshooting.md
  workflow: 15
---

# WSL2 + Windows + 远程 Chrome CDP 故障排查

本指南涵盖常见的分机部署场景：

- OpenClaw Gateway 网关在 WSL2 内部运行
- Chrome 在 Windows 上运行
- 浏览器控制必须跨越 WSL2/Windows 边界

本指南还涵盖来自 [issue #39369](https://github.com/openclaw/openclaw/issues/39369) 的分层故障模式：多个独立问题可能同时出现，导致表面上看起来是错误的层出现问题。

## 首先选择正确的浏览器模式

你有两种有效方案：

### 方案 1：从 WSL2 到 Windows 的原始远程 CDP

使用指向 WSL2 到 Windows Chrome CDP 端点的远程浏览器配置文件。

适用场景：

- Gateway 网关保留在 WSL2 内部
- Chrome 在 Windows 上运行
- 需要浏览器控制跨越 WSL2/Windows 边界

### 方案 2：本机 Chrome MCP

仅在 Gateway 网关与 Chrome 在同一主机上运行时使用 `existing-session` / `user`。

适用场景：

- OpenClaw 和 Chrome 在同一台机器上
- 你需要本地已登录的浏览器状态
- 不需要跨主机浏览器传输

对于 WSL2 Gateway + Windows Chrome，优先使用原始远程 CDP。Chrome MCP 是本机的，不是 WSL2 到 Windows 的桥接。

## 工作架构

参考结构：

- WSL2 在 `127.0.0.1:18789` 上运行 Gateway 网关
- Windows 在普通浏览器中通过 `http://127.0.0.1:18789/` 打开控制界面
- Windows Chrome 在端口 `9222` 上暴露 CDP 端点
- WSL2 可以访问该 Windows CDP 端点
- OpenClaw 将浏览器配置文件指向 WSL2 可访问的地址

## 为何此设置令人困惑

多个故障可能叠加：

- WSL2 无法访问 Windows CDP 端点
- 控制界面从非安全来源打开
- `gateway.controlUi.allowedOrigins` 与页面来源不匹配
- 令牌或配对缺失
- 浏览器配置文件指向错误地址

因此，修复一层后可能仍会看到另一层的错误。

## 控制界面的关键规则

从 Windows 打开界面时，除非有专用 HTTPS 设置，否则使用 Windows localhost。

使用：

`http://127.0.0.1:18789/`

不要默认使用局域网 IP 作为控制界面地址。在局域网或 Tailnet 地址上的普通 HTTP 可能触发与 CDP 本身无关的不安全来源/设备认证行为。参见[控制界面](/web/control-ui)。

## 分层验证

从上到下进行，不要跳过步骤。

### 第 1 层：验证 Chrome 在 Windows 上提供 CDP 服务

在 Windows 上以远程调试模式启动 Chrome：

```powershell
chrome.exe --remote-debugging-port=9222
```

首先在 Windows 上验证 Chrome 本身：

```powershell
curl http://127.0.0.1:9222/json/version
curl http://127.0.0.1:9222/json/list
```

如果在 Windows 上失败，问题尚未出在 OpenClaw 上。

### 第 2 层：验证 WSL2 可以访问该 Windows 端点

从 WSL2 中，测试你计划在 `cdpUrl` 中使用的确切地址：

```bash
curl http://WINDOWS_HOST_OR_IP:9222/json/version
curl http://WINDOWS_HOST_OR_IP:9222/json/list
```

预期结果：

- `/json/version` 返回带有 Browser / Protocol-Version 元数据的 JSON
- `/json/list` 返回 JSON（如果没有打开的页面，空数组也正常）

如果失败：

- Windows 尚未将端口暴露给 WSL2
- 地址对 WSL2 侧来说是错误的
- 防火墙/端口转发/本地代理仍然缺失

在修改 OpenClaw 配置之前先修复该问题。

### 第 3 层：配置正确的浏览器配置文件

对于原始远程 CDP，将 OpenClaw 指向 WSL2 可访问的地址：

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "remote",
    profiles: {
      remote: {
        cdpUrl: "http://WINDOWS_HOST_OR_IP:9222",
        attachOnly: true,
        color: "#00AA00",
      },
    },
  },
}
```

说明：

- 使用 WSL2 可访问的地址，而非只在 Windows 上有效的地址
- 对于外部管理的浏览器保留 `attachOnly: true`
- 在期望 OpenClaw 成功之前，先用 `curl` 测试相同 URL

### 第 4 层：单独验证控制界面层

从 Windows 打开界面：

`http://127.0.0.1:18789/`

然后验证：

- 页面来源与 `gateway.controlUi.allowedOrigins` 期望的内容匹配
- 令牌认证或配对配置正确
- 你没有把控制界面认证问题当作浏览器问题来调试

参考页面：[控制界面](/web/control-ui)

### 第 5 层：验证端对端浏览器控制

从 WSL2：

```bash
openclaw browser open https://example.com --browser-profile remote
openclaw browser tabs --browser-profile remote
```

预期结果：

- 标签页在 Windows Chrome 中打开
- `openclaw browser tabs` 返回目标
- 后续操作（`snapshot`、`screenshot`、`navigate`）从同一配置文件正常工作

## 常见误导性错误

将每条信息视为特定层的线索：

- `control-ui-insecure-auth`——界面来源/安全上下文问题，不是 CDP 传输问题
- `token_missing`——认证配置问题
- `pairing required`——设备审批问题
- `Remote CDP for profile "remote" is not reachable`——WSL2 无法访问配置的 `cdpUrl`
- `gateway timeout after 1500ms`——通常仍然是 CDP 可达性问题或远程端点缓慢/不可达
- `No Chrome tabs found for profile="user"`——选择了本机 Chrome MCP 配置文件，但没有本地标签页可用

## 快速排查清单

1. Windows：`curl http://127.0.0.1:9222/json/version` 是否有效？
2. WSL2：`curl http://WINDOWS_HOST_OR_IP:9222/json/version` 是否有效？
3. OpenClaw 配置：`browser.profiles.<name>.cdpUrl` 是否使用了 WSL2 可访问的确切地址？
4. 控制界面：你是否在打开 `http://127.0.0.1:18789/` 而非局域网 IP？
5. 你是否在尝试跨 WSL2 和 Windows 使用 `existing-session` 而非原始远程 CDP？

## 实践要点

该设置通常是可行的。难点在于浏览器传输、控制界面来源安全性和令牌/配对各自可能独立失败，但从用户角度看起来相似。

如有疑问：

- 首先在 Windows 上本地验证 Chrome 端点
- 其次从 WSL2 验证相同端点
- 然后才调试 OpenClaw 配置或控制界面认证
