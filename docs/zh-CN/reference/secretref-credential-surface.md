---
summary: "SecretRef 凭据覆盖范围的规范定义（支持与不支持）"
read_when:
  - 验证 SecretRef 凭据覆盖范围
  - 审计某个凭据是否适用于 `secrets configure` 或 `secrets apply`
  - 了解某个凭据不在支持范围内的原因
title: "SecretRef 凭据覆盖范围"
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: ad0a73f03fc6cab5c7435403c993c0b79b514eb0790ac27cf1c94df693b27da5
  source_path: reference/secretref-credential-surface.md
  workflow: 15
---

# SecretRef 凭据覆盖范围

本页定义 SecretRef 凭据覆盖范围的规范边界。

范围说明：

- 在范围内：严格由用户提供的凭据，OpenClaw 不负责生成或轮换。
- 范围外：运行时生成或轮换的凭据、OAuth 刷新材料以及类会话的工件。

## 支持的凭据

### `openclaw.json` 目标（`secrets configure` + `secrets apply` + `secrets audit`）

[//]: # "secretref-supported-list-start"

- `models.providers.*.apiKey`
- `models.providers.*.headers.*`
- `skills.entries.*.apiKey`
- `agents.defaults.memorySearch.remote.apiKey`
- `agents.list[].memorySearch.remote.apiKey`
- `talk.apiKey`
- `talk.providers.*.apiKey`
- `messages.tts.providers.*.apiKey`
- `tools.web.fetch.firecrawl.apiKey`
- `plugins.entries.brave.config.webSearch.apiKey`
- `plugins.entries.google.config.webSearch.apiKey`
- `plugins.entries.xai.config.webSearch.apiKey`
- `plugins.entries.moonshot.config.webSearch.apiKey`
- `plugins.entries.perplexity.config.webSearch.apiKey`
- `plugins.entries.firecrawl.config.webSearch.apiKey`
- `plugins.entries.tavily.config.webSearch.apiKey`
- `tools.web.search.apiKey`
- `tools.web.search.gemini.apiKey`
- `tools.web.search.grok.apiKey`
- `tools.web.search.kimi.apiKey`
- `tools.web.search.perplexity.apiKey`
- `tools.web.x_search.apiKey`
- `gateway.auth.password`
- `gateway.auth.token`
- `gateway.remote.token`
- `gateway.remote.password`
- `cron.webhookToken`
- `channels.telegram.botToken`
- `channels.telegram.webhookSecret`
- `channels.telegram.accounts.*.botToken`
- `channels.telegram.accounts.*.webhookSecret`
- `channels.slack.botToken`
- `channels.slack.appToken`
- `channels.slack.userToken`
- `channels.slack.signingSecret`
- `channels.slack.accounts.*.botToken`
- `channels.slack.accounts.*.appToken`
- `channels.slack.accounts.*.userToken`
- `channels.slack.accounts.*.signingSecret`
- `channels.discord.token`
- `channels.discord.pluralkit.token`
- `channels.discord.voice.tts.providers.*.apiKey`
- `channels.discord.accounts.*.token`
- `channels.discord.accounts.*.pluralkit.token`
- `channels.discord.accounts.*.voice.tts.providers.*.apiKey`
- `channels.irc.password`
- `channels.irc.nickserv.password`
- `channels.irc.accounts.*.password`
- `channels.irc.accounts.*.nickserv.password`
- `channels.bluebubbles.password`
- `channels.bluebubbles.accounts.*.password`
- `channels.feishu.appSecret`
- `channels.feishu.encryptKey`
- `channels.feishu.verificationToken`
- `channels.feishu.accounts.*.appSecret`
- `channels.feishu.accounts.*.encryptKey`
- `channels.feishu.accounts.*.verificationToken`
- `channels.msteams.appPassword`
- `channels.mattermost.botToken`
- `channels.mattermost.accounts.*.botToken`
- `channels.matrix.accessToken`
- `channels.matrix.password`
- `channels.matrix.accounts.*.accessToken`
- `channels.matrix.accounts.*.password`
- `channels.nextcloud-talk.botSecret`
- `channels.nextcloud-talk.apiPassword`
- `channels.nextcloud-talk.accounts.*.botSecret`
- `channels.nextcloud-talk.accounts.*.apiPassword`
- `channels.zalo.botToken`
- `channels.zalo.webhookSecret`
- `channels.zalo.accounts.*.botToken`
- `channels.zalo.accounts.*.webhookSecret`
- `channels.googlechat.serviceAccount`（通过兄弟字段 `serviceAccountRef`，兼容性例外）
- `channels.googlechat.accounts.*.serviceAccount`（通过兄弟字段 `serviceAccountRef`，兼容性例外）

### `auth-profiles.json` 目标（`secrets configure` + `secrets apply` + `secrets audit`）

- `profiles.*.keyRef`（`type: "api_key"`）
- `profiles.*.tokenRef`（`type: "token"`）

[//]: # "secretref-supported-list-end"

说明：

- 认证配置文件计划目标需要 `agentId`。
- 计划条目目标为 `profiles.*.key` / `profiles.*.token`，并写入兄弟引用字段（`keyRef` / `tokenRef`）。
- 认证配置文件引用包含在运行时解析和审计覆盖范围内。
- 对于由 SecretRef 管理的模型提供商，生成的 `agents/*/agent/models.json` 条目会为 `apiKey`/headers 字段保留非密钥标记（非已解析的密钥值）。
- 标记持久化以源配置为权威来源：OpenClaw 从活跃的源配置快照（解析前）写入标记，而非从已解析的运行时密钥值写入。
- 关于 web 搜索：
  - 显式提供商模式下（已设置 `tools.web.search.provider`），仅所选提供商的密钥有效。
  - 自动模式下（未设置 `tools.web.search.provider`），仅按优先级解析的第一个提供商密钥有效。
  - 自动模式下，未被选中的提供商引用在被选中前视为非活跃状态。
  - 旧版 `tools.web.search.*` 提供商路径在兼容窗口期间仍可解析，但 SecretRef 的规范覆盖范围是 `plugins.entries.<plugin>.config.webSearch.*`。

## 不支持的凭据

范围外的凭据包括：

[//]: # "secretref-unsupported-list-start"

- `commands.ownerDisplaySecret`
- `hooks.token`
- `hooks.gmail.pushToken`
- `hooks.mappings[].sessionKey`
- `auth-profiles.oauth.*`
- `discord.threadBindings.*.webhookToken`
- `whatsapp.creds.json`

[//]: # "secretref-unsupported-list-end"

原因说明：

- 这些凭据属于生成型、轮换型、持有会话状态或 OAuth 持久类凭据，不适合只读的外部 SecretRef 解析模式。
