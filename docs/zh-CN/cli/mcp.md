---
read_when:
  - 将 Codex、Claude Code 或其他 MCP 客户端连接到 OpenClaw 支持的渠道
  - 运行 `openclaw mcp serve`
  - 管理 OpenClaw 保存的 MCP 服务器定义
summary: 通过 MCP 暴露 OpenClaw 渠道对话，并管理已保存的 MCP 服务器定义
title: mcp
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 77e501fff9df1bf9d4efa007ca4d89949eea6fc6385df97bba2911cf4bff4f00
  source_path: cli/mcp.md
  workflow: 15
---

# mcp

`openclaw mcp` 有两项职责：

- 通过 `openclaw mcp serve` 将 OpenClaw 作为 MCP 服务器运行
- 通过 `list`、`show`、`set` 和 `unset` 管理 OpenClaw 拥有的出站 MCP 服务器定义

换句话说：

- `serve` 是 OpenClaw 作为 MCP 服务器对外提供服务
- `list` / `show` / `set` / `unset` 是 OpenClaw 作为其运行时后续可能使用的其他 MCP 服务器的客户端注册表

当 OpenClaw 应该托管编码运行时会话本身并通过 ACP 路由该运行时时，请使用 [`openclaw acp`](/cli/acp)。

## OpenClaw 作为 MCP 服务器

这是 `openclaw mcp serve` 路径。

## 何时使用 `serve`

在以下情况使用 `openclaw mcp serve`：

- Codex、Claude Code 或其他 MCP 客户端应直接与 OpenClaw 支持的渠道对话
- 你已有一个带路由会话的本地或远程 OpenClaw Gateway 网关
- 你希望通过一个 MCP 服务器跨 OpenClaw 所有渠道后端工作，而不是为每个渠道运行独立的桥接

当 OpenClaw 应托管编码运行时本身并将智能体会话保留在 OpenClaw 内部时，请改用 [`openclaw acp`](/cli/acp)。

## 工作原理

`openclaw mcp serve` 启动一个 stdio MCP 服务器。MCP 客户端拥有该进程。当客户端保持 stdio 会话打开时，桥接通过 WebSocket 连接到本地或远程 OpenClaw Gateway 网关，并通过 MCP 暴露已路由的渠道对话。

生命周期：

1. MCP 客户端启动 `openclaw mcp serve`
2. 桥接连接到 Gateway 网关
3. 已路由的会话成为 MCP 对话和转录/历史记录工具
4. 桥接连接期间，实时事件在内存中排队
5. 如果启用了 Claude 渠道模式，同一会话还可以接收 Claude 专用推送通知

重要行为：

- 实时队列状态在桥接连接时开始
- 较旧的转录历史通过 `messages_read` 读取
- Claude 推送通知仅在 MCP 会话存活期间存在
- 客户端断开连接时，桥接退出且实时队列消失

## 选择客户端模式

以两种不同方式使用同一桥接：

- 通用 MCP 客户端：仅标准 MCP 工具。使用 `conversations_list`、`messages_read`、`events_poll`、`events_wait`、`messages_send` 和审批工具。
- Claude Code：标准 MCP 工具加上 Claude 专用渠道适配器。启用 `--claude-channel-mode on` 或保留默认的 `auto`。

目前，`auto` 与 `on` 行为相同。尚无客户端能力检测。

## `serve` 暴露的内容

桥接使用现有的 Gateway 网关会话路由元数据来暴露渠道支持的对话。当 OpenClaw 已有包含已知路由的会话状态时，对话会显示出来，例如：

- `channel`
- 收件人或目的地元数据
- 可选的 `accountId`
- 可选的 `threadId`

这为 MCP 客户端提供了一个统一的地方，用于：

- 列出最近的已路由对话
- 读取最近的转录历史
- 等待新的入站事件
- 通过同一路由发送回复
- 查看桥接连接期间到达的审批请求

## 用法

```bash
# 本地 Gateway 网关
openclaw mcp serve

# 远程 Gateway 网关
openclaw mcp serve --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token

# 使用密码认证的远程 Gateway 网关
openclaw mcp serve --url wss://gateway-host:18789 --password-file ~/.openclaw/gateway.password

# 启用详细桥接日志
openclaw mcp serve --verbose

# 禁用 Claude 专用推送通知
openclaw mcp serve --claude-channel-mode off
```

## 桥接工具

当前桥接暴露以下 MCP 工具：

- `conversations_list`
- `conversation_get`
- `messages_read`
- `attachments_fetch`
- `events_poll`
- `events_wait`
- `messages_send`
- `permissions_list_open`
- `permissions_respond`

### `conversations_list`

列出在 Gateway 网关会话状态中已有路由元数据的最近会话支持对话。

常用过滤器：

- `limit`
- `search`
- `channel`
- `includeDerivedTitles`
- `includeLastMessage`

### `conversation_get`

按 `session_key` 返回一个对话。

### `messages_read`

读取一个会话支持对话的最近转录消息。

### `attachments_fetch`

从一条转录消息中提取非文本内容块。这是转录内容的元数据视图，而非独立的持久附件 Blob 存储。

### `events_poll`

从数字游标位置读取排队的实时事件。

### `events_wait`

长轮询，直到下一个匹配的排队事件到达或超时。

当通用 MCP 客户端需要近实时传递且无 Claude 专用推送协议时使用此方法。

### `messages_send`

通过会话上已记录的同一路由发送文本。

当前行为：

- 需要现有的对话路由
- 使用会话的渠道、收件人、账户 ID 和线程 ID
- 仅发送文本

### `permissions_list_open`

列出桥接自连接到 Gateway 网关以来观察到的待处理 exec / 插件审批请求。

### `permissions_respond`

使用以下结果解决一个待处理的 exec / 插件审批请求：

- `allow-once`
- `allow-always`
- `deny`

## 事件模型

桥接在连接期间维护一个内存事件队列。

当前事件类型：

- `message`
- `exec_approval_requested`
- `exec_approval_resolved`
- `plugin_approval_requested`
- `plugin_approval_resolved`
- `claude_permission_request`

重要限制：

- 队列仅为实时队列；从 MCP 桥接启动时开始
- `events_poll` 和 `events_wait` 不会自行重放较旧的 Gateway 网关历史
- 持久积压应通过 `messages_read` 读取

## Claude 渠道通知

桥接还可以暴露 Claude 专用渠道通知。这是 OpenClaw 的 Claude Code 渠道适配器等价物：标准 MCP 工具仍然可用，但实时入站消息也可以作为 Claude 专用 MCP 通知到达。

标志：

- `--claude-channel-mode off`：仅标准 MCP 工具
- `--claude-channel-mode on`：启用 Claude 渠道通知
- `--claude-channel-mode auto`：当前默认值；与 `on` 桥接行为相同

启用 Claude 渠道模式后，服务器会宣告 Claude 实验性能力，并可发出：

- `notifications/claude/channel`
- `notifications/claude/channel/permission`

当前桥接行为：

- 入站 `user` 转录消息作为 `notifications/claude/channel` 转发
- 通过 MCP 收到的 Claude 权限请求在内存中跟踪
- 如果链接的对话后来发送 `yes abcde` 或 `no abcde`，桥接将其转换为 `notifications/claude/channel/permission`
- 这些通知仅为实时会话通知；如果 MCP 客户端断开连接，则无推送目标

这是有意针对特定客户端的设计。通用 MCP 客户端应依赖标准轮询工具。

## MCP 客户端配置

示例 stdio 客户端配置：

```json
{
  "mcpServers": {
    "openclaw": {
      "command": "openclaw",
      "args": [
        "mcp",
        "serve",
        "--url",
        "wss://gateway-host:18789",
        "--token-file",
        "/path/to/gateway.token"
      ]
    }
  }
}
```

对于大多数通用 MCP 客户端，从标准工具界面开始，忽略 Claude 模式。仅对真正理解 Claude 专用通知方法的客户端启用 Claude 模式。

## 选项

`openclaw mcp serve` 支持：

- `--url <url>`：Gateway 网关 WebSocket URL
- `--token <token>`：Gateway 网关令牌
- `--token-file <path>`：从文件读取令牌
- `--password <password>`：Gateway 网关密码
- `--password-file <path>`：从文件读取密码
- `--claude-channel-mode <auto|on|off>`：Claude 通知模式
- `-v`、`--verbose`：向 stderr 输出详细日志

可能时，优先使用 `--token-file` 或 `--password-file`，而非内联 secret。

## 安全与信任边界

桥接不创造路由。它仅暴露 Gateway 网关已知如何路由的对话。

这意味着：

- 发送者允许列表、配对和渠道级信任仍属于底层 OpenClaw 渠道配置
- `messages_send` 只能通过现有的已存储路由进行回复
- 审批状态仅为当前桥接会话的实时/内存状态
- 桥接认证应使用与其他任何远程 Gateway 网关客户端相同的 Gateway 网关令牌或密码控制

如果 `conversations_list` 中缺少某个对话，通常原因不是 MCP 配置问题，而是底层 Gateway 网关会话中缺少或不完整的路由元数据。

## 测试

OpenClaw 为该桥接提供了确定性的 Docker 冒烟测试：

```bash
pnpm test:docker:mcp-channels
```

该冒烟测试：

- 启动一个预植入数据的 Gateway 网关容器
- 启动第二个容器，在其中启动 `openclaw mcp serve`
- 验证对话发现、转录读取、附件元数据读取、实时事件队列行为和出站发送路由
- 通过真实 stdio MCP 桥接验证 Claude 风格的渠道和权限通知

这是在不接入真实 Telegram、Discord 或 iMessage 账户的情况下验证桥接工作的最快方式。

更多测试背景，请参阅[测试](/help/testing)。

## 故障排除

### 未返回对话

通常意味着 Gateway 网关会话尚不可路由。确认底层会话已存储渠道/提供商、收件人以及可选的账户/线程路由元数据。

### `events_poll` 或 `events_wait` 遗漏较旧消息

符合预期。实时队列在桥接连接时开始。使用 `messages_read` 读取较旧的转录历史。

### Claude 通知未显示

检查以下所有内容：

- 客户端保持 stdio MCP 会话打开
- `--claude-channel-mode` 为 `on` 或 `auto`
- 客户端确实理解 Claude 专用通知方法
- 入站消息发生在桥接连接之后

### 审批缺失

`permissions_list_open` 仅显示桥接连接期间观察到的审批请求。它不是持久的审批历史 API。

## OpenClaw 作为 MCP 客户端注册表

这是 `openclaw mcp list`、`show`、`set` 和 `unset` 路径。

这些命令不会通过 MCP 暴露 OpenClaw。它们管理 OpenClaw 在 OpenClaw 配置中 `mcp.servers` 下拥有的 MCP 服务器定义。

这些已保存的定义供 OpenClaw 后续启动或配置的运行时使用，例如嵌入式 Pi 和其他运行时适配器。OpenClaw 集中存储这些定义，这样这些运行时就不需要维护自己的重复 MCP 服务器列表。

重要行为：

- 这些命令仅读取或写入 OpenClaw 配置
- 它们不连接到目标 MCP 服务器
- 它们不验证命令、URL 或远程传输是否当前可达
- 运行时适配器在执行时决定它们实际支持哪些传输形式

## 已保存的 MCP 服务器定义

OpenClaw 还在配置中为需要 OpenClaw 管理 MCP 定义的界面存储了一个轻量级 MCP 服务器注册表。

命令：

- `openclaw mcp list`
- `openclaw mcp show [name]`
- `openclaw mcp set <name> <json>`
- `openclaw mcp unset <name>`

示例：

```bash
openclaw mcp list
openclaw mcp show context7 --json
openclaw mcp set context7 '{"command":"uvx","args":["context7-mcp"]}'
openclaw mcp set docs '{"url":"https://mcp.example.com"}'
openclaw mcp unset context7
```

示例配置结构：

```json
{
  "mcp": {
    "servers": {
      "context7": {
        "command": "uvx",
        "args": ["context7-mcp"]
      },
      "docs": {
        "url": "https://mcp.example.com"
      }
    }
  }
}
```

典型字段：

- `command`
- `args`
- `env`
- `cwd` 或 `workingDirectory`
- `url`

这些命令仅管理已保存的配置。它们不启动渠道桥接，不打开实时 MCP 客户端会话，也不验证目标服务器是否可达。

## 当前限制

本页记录当前已发布的桥接状态。

当前限制：

- 对话发现依赖现有的 Gateway 网关会话路由元数据
- 除 Claude 专用适配器外，没有通用推送协议
- 尚无消息编辑或反应工具
- 尚无专用 HTTP MCP 传输
- `permissions_list_open` 仅包括桥接连接期间观察到的审批
