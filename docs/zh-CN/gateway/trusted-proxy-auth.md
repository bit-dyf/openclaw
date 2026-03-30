---
read_when:
  - 在身份感知代理后运行 OpenClaw
  - 在 OpenClaw 前设置 Pomerium、Caddy 或带 OAuth 的 nginx
  - 修复反向代理设置中的 WebSocket 1008 未授权错误
  - 决定在哪里设置 HSTS 和其他 HTTP 加固标头
summary: 将 Gateway 网关认证委托给可信反向代理（Pomerium、Caddy、nginx + OAuth）
title: 可信代理认证
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 5a9fdebd20b4b8afb18f1ab108080a9dae9c7775794441f092b09e90dc95e6e5
  source_path: gateway/trusted-proxy-auth.md
  workflow: 15
---

# 可信代理认证

> ⚠️ **安全敏感功能。** 此模式将认证完全委托给你的反向代理。配置错误可能导致你的 Gateway 网关暴露给未授权访问。启用前请仔细阅读本页。

## 何时使用

在以下情况下使用 `trusted-proxy` 认证模式：

- 你在**身份感知代理**后运行 OpenClaw（Pomerium、带 OAuth 的 Caddy、带 oauth2-proxy 的 nginx、带前向认证的 Traefik）
- 你的代理处理所有认证并通过标头传递用户身份
- 你在 Kubernetes 或容器环境中，代理是访问 Gateway 网关的唯一路径
- 你遇到 WebSocket `1008 未授权`错误，因为浏览器无法在 WS 载荷中传递令牌

## 何时不使用

- 如果你的代理不认证用户（仅 TLS 终止器或负载均衡器）
- 如果有任何绕过代理访问 Gateway 网关的路径（防火墙漏洞、内部网络访问）
- 如果你不确定你的代理是否正确剥离/覆盖转发标头
- 如果你只需要个人单用户访问（考虑使用 Tailscale Serve + 回环地址更简单）

## 工作原理

1. 你的反向代理认证用户（OAuth、OIDC、SAML 等）
2. 代理添加带有认证用户身份的标头（例如 `x-forwarded-user: nick@example.com`）
3. OpenClaw 检查请求是否来自**可信代理 IP**（在 `gateway.trustedProxies` 中配置）
4. OpenClaw 从配置的标头中提取用户身份
5. 如果一切检查通过，请求即被授权

## 配置

```json5
{
  gateway: {
    // 对同主机代理设置使用回环地址；对远程代理主机使用 lan/custom
    bind: "loopback",

    // 关键：仅在此处添加代理的 IP
    trustedProxies: ["10.0.0.1", "172.17.0.1"],

    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        // 包含认证用户身份的标头（必填）
        userHeader: "x-forwarded-user",

        // 可选：必须存在的标头（代理验证）
        requiredHeaders: ["x-forwarded-proto", "x-forwarded-host"],

        // 可选：限制到特定用户（空 = 允许所有）
        allowUsers: ["nick@example.com", "admin@company.org"],
      },
    },
  },
}
```

## 代理设置示例

### Pomerium

```json5
{
  gateway: {
    bind: "lan",
    trustedProxies: ["10.0.0.1"],
    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        userHeader: "x-pomerium-claim-email",
        requiredHeaders: ["x-pomerium-jwt-assertion"],
      },
    },
  },
}
```

### 带 OAuth 的 Caddy

```json5
{
  gateway: {
    bind: "lan",
    trustedProxies: ["127.0.0.1"],
    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        userHeader: "x-forwarded-user",
      },
    },
  },
}
```

### nginx + oauth2-proxy

```json5
{
  gateway: {
    bind: "lan",
    trustedProxies: ["10.0.0.1"],
    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        userHeader: "x-auth-request-email",
      },
    },
  },
}
```

## 安全清单

启用可信代理认证前，验证：

- [ ] **代理是唯一路径**：Gateway 网关端口防火墙隔离，仅允许代理访问
- [ ] **trustedProxies 最小化**：仅是你实际的代理 IP，而非整个子网
- [ ] **代理剥离标头**：你的代理覆盖（而非追加）来自客户端的 `x-forwarded-*` 标头
- [ ] **TLS 终止**：你的代理处理 TLS；用户通过 HTTPS 连接
- [ ] **设置了 allowUsers**（推荐）：限制为已知用户，而非允许任何已认证用户

## 故障排查

### "trusted_proxy_untrusted_source"

请求不是来自 `gateway.trustedProxies` 中的 IP。检查：

- 代理 IP 是否正确？（Docker 容器 IP 可能会变化）
- 代理前面是否有负载均衡器？

### "trusted_proxy_user_missing"

用户标头为空或缺失。检查：

- 你的代理是否配置为传递身份标头？
- 标头名称是否正确？

### WebSocket 仍然失败

确保你的代理：

- 支持 WebSocket 升级（`Upgrade: websocket`、`Connection: upgrade`）
- 在 WebSocket 升级请求（不仅仅是 HTTP）上传递身份标头

## 相关内容

- [安全](/gateway/security) — 完整安全指南
- [配置](/gateway/configuration) — 配置参考
- [远程访问](/gateway/remote) — 其他远程访问模式
- [Tailscale](/gateway/tailscale) — 仅限 Tailnet 访问的更简单替代方案
