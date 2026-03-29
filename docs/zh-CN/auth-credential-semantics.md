---
title: "认证凭据语义"
summary: "认证配置文件的凭据资格与解析语义的规范定义"
read_when:
  - 处理认证配置文件解析或凭据路由时
  - 调试模型认证失败或配置文件顺序问题时
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: d86d38fafa18bbdc6892fc54a0f32b19ab33df8898206796aaecb9a9967aeaa5
  source_path: auth-credential-semantics.md
  workflow: 15
---

# 认证凭据语义

本文档定义了以下场景中使用的凭据资格与解析语义的规范：

- `resolveAuthProfileOrder`
- `resolveApiKeyForProfile`
- `models status --probe`
- `doctor-auth`

目标是保持选择时与运行时行为的一致性。

## 稳定的原因码

- `ok`
- `missing_credential`
- `invalid_expires`
- `expired`
- `unresolved_ref`

## Token 凭据

Token 凭据（`type: "token"`）支持内联 `token` 和/或 `tokenRef`。

### 资格规则

1. 当 `token` 和 `tokenRef` 均缺失时，该 token 配置文件不具备资格。
2. `expires` 为可选字段。
3. 若存在 `expires`，其值必须是大于 `0` 的有限数字。
4. 若 `expires` 无效（`NaN`、`0`、负数、非有限数或类型错误），则该配置文件不具备资格，原因码为 `invalid_expires`。
5. 若 `expires` 已过期，则该配置文件不具备资格，原因码为 `expired`。
6. `tokenRef` 不会绕过 `expires` 验证。

### 解析规则

1. 解析器语义与 `expires` 的资格语义保持一致。
2. 对于具备资格的配置文件，token 材料可从内联值或 `tokenRef` 中解析。
3. 无法解析的引用在 `models status --probe` 输出中产生 `unresolved_ref`。

## 向后兼容的错误消息

为保证脚本兼容性，探测错误的第一行保持不变：

`Auth profile credentials are missing or expired.`

后续行可添加人性化详情和稳定的原因码。
