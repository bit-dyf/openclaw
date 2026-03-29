---
title: "内存配置参考"
summary: "OpenClaw 内存搜索、嵌入提供商、QMD 后端、混合搜索和多模态内存的完整配置参考"
read_when:
  - 希望配置内存搜索提供商或嵌入模型
  - 希望设置 QMD 后端
  - 希望调优混合搜索、MMR 或时间衰减
  - 希望启用多模态内存索引
---

# 内存配置参考

本页涵盖 OpenClaw 内存搜索的完整配置界面。有关概念概览（文件布局、内存工具、何时写入内存以及自动刷新），请参阅[内存](/concepts/memory)。

## 内存搜索默认值

- 默认启用。
- 监控内存文件的变更（去抖动）。
- 在 `agents.defaults.memorySearch` 下配置内存搜索（而非顶级 `memorySearch`）。
- `memorySearch.provider` 和 `memorySearch.fallback` 接受活跃内存插件注册的**适配器 ID**。
- 默认 `memory-core` 插件注册以下内置适配器 ID：`local`、`openai`、`gemini`、`voyage`、`mistral` 和 `ollama`。
- 使用默认 `memory-core` 插件时，如果未设置 `memorySearch.provider`，OpenClaw 会自动选择：
  1. 如果配置了 `memorySearch.local.modelPath` 且文件存在，选择 `local`。
  2. 如果可以解析 OpenAI 密钥，选择 `openai`。
  3. 如果可以解析 Gemini 密钥，选择 `gemini`。
  4. 如果可以解析 Voyage 密钥，选择 `voyage`。
  5. 如果可以解析 Mistral 密钥，选择 `mistral`。
  6. 否则内存搜索保持禁用状态直到配置完成。
- 本地模式使用 node-llama-cpp，可能需要 `pnpm approve-builds`。
- 使用 sqlite-vec（如果可用）加速 SQLite 内的向量搜索。
- 使用默认 `memory-core` 插件时，也支持 `memorySearch.provider = "ollama"` 用于本地/自托管 Ollama 嵌入（`/api/embeddings`），但不会自动选择。

远程嵌入**需要**为嵌入提供商提供 API 密钥。OpenClaw 从认证配置文件、`models.providers.*.apiKey` 或环境变量解析密钥。Codex OAuth 仅涵盖聊天/完成，**不满足**内存搜索的嵌入需求。对于 Gemini，使用 `GEMINI_API_KEY` 或 `models.providers.google.apiKey`。对于 Voyage，使用 `VOYAGE_API_KEY` 或 `models.providers.voyage.apiKey`。对于 Mistral，使用 `MISTRAL_API_KEY` 或 `models.providers.mistral.apiKey`。Ollama 通常不需要真实的 API 密钥（在本地策略需要时，像 `OLLAMA_API_KEY=ollama-local` 这样的占位符就足够了）。
使用自定义 OpenAI 兼容端点时，设置 `memorySearch.remote.apiKey`（和可选的 `memorySearch.remote.headers`）。

## QMD 后端（实验性）

设置 `memory.backend = "qmd"` 将内置 SQLite 索引器替换为 [QMD](https://github.com/tobi/qmd)：一个结合 BM25 + 向量 + 重排序的本地优先搜索边车。Markdown 保持真相之源；OpenClaw 调用 QMD 进行检索。要点：

### 先决条件

- 默认禁用。按配置选择加入（`memory.backend = "qmd"`）。
- 单独安装 QMD CLI（`bun install -g https://github.com/tobi/qmd` 或获取发布版本）并确保 `qmd` 二进制文件在 Gateway 的 `PATH` 中。
- QMD 需要允许扩展的 SQLite 构建（macOS 上使用 `brew install sqlite`）。
- QMD 通过 Bun + `node-llama-cpp` 完全本地运行，首次使用时从 HuggingFace 自动下载 GGUF 模型（不需要单独的 Ollama 守护进程）。
- Gateway 通过设置 `XDG_CONFIG_HOME` 和 `XDG_CACHE_HOME` 在 `~/.openclaw/agents/<agentId>/qmd/` 下的自包含 XDG 主目录中运行 QMD。
- 操作系统支持：一旦安装了 Bun + SQLite，macOS 和 Linux 开箱即用。Windows 最好通过 WSL2 支持。

### 边车运行方式

- Gateway 在 `~/.openclaw/agents/<agentId>/qmd/`（配置 + 缓存 + sqlite 数据库）下写入自包含的 QMD 主目录。
- 通过 `qmd collection add` 从 `memory.qmd.paths`（加上默认工作区内存文件）创建集合，然后在启动时和可配置间隔（`memory.qmd.update.interval`，默认 5 分钟）运行 `qmd update` + `qmd embed`。
- Gateway 现在在启动时初始化 QMD 管理器，因此即使在第一次 `memory_search` 调用之前也会启动定期更新计时器。
- 启动刷新现在默认在后台运行，因此聊天启动不会被阻塞；设置 `memory.qmd.update.waitForBootSync = true` 以保持之前的阻塞行为。
- 搜索通过 `memory.qmd.searchMode` 运行（默认 `qmd search --json`；也支持 `vsearch` 和 `query`）。如果选定的模式在您的 QMD 构建上拒绝标志，OpenClaw 会用 `qmd query` 重试。如果 QMD 失败或二进制文件丢失，OpenClaw 会自动回退到内置 SQLite 管理器，因此内存工具继续工作。
- OpenClaw 今天不公开 QMD 嵌入批量大小调优；批量行为由 QMD 本身控制。
- **首次搜索可能很慢**：QMD 可能在第一次 `qmd query` 运行时下载本地 GGUF 模型（重排序器/查询扩展）。
  - OpenClaw 在运行 QMD 时自动设置 `XDG_CONFIG_HOME`/`XDG_CACHE_HOME`。
  - 如果您想手动预下载模型（并预热 OpenClaw 使用的相同索引），请使用智能体的 XDG 目录运行一次性查询。

OpenClaw 的 QMD 状态位于您的**状态目录**下（默认为 `~/.openclaw`）。
您可以通过导出 OpenClaw 使用的相同 XDG 变量将 `qmd` 指向完全相同的索引：

```bash
# 选择 OpenClaw 使用的相同状态目录
STATE_DIR="${OPENCLAW_STATE_DIR:-$HOME/.openclaw}"

export XDG_CONFIG_HOME="$STATE_DIR/agents/main/qmd/xdg-config"
export XDG_CACHE_HOME="$STATE_DIR/agents/main/qmd/xdg-cache"

# （可选）强制索引刷新 + 嵌入
qmd update
qmd embed

# 预热 / 触发首次模型下载
qmd query "test" -c memory-root --json >/dev/null 2>&1
```

### 配置界面（`memory.qmd.*`）

- `command`（默认 `qmd`）：覆盖可执行文件路径。
- `searchMode`（默认 `search`）：选择哪个 QMD 命令支持 `memory_search`（`search`、`vsearch`、`query`）。
- `includeDefaultMemory`（默认 `true`）：自动索引 `MEMORY.md` + `memory/**/*.md`。
- `paths[]`：添加额外的目录/文件（`path`、可选 `pattern`、可选稳定 `name`）。
- `sessions`：选择加入会话 JSONL 索引（`enabled`、`retentionDays`、`exportDir`）。
- `update`：控制刷新节奏和维护执行：（`interval`、`debounceMs`、`onBoot`、`waitForBootSync`、`embedInterval`、`commandTimeoutMs`、`updateTimeoutMs`、`embedTimeoutMs`）。
- `limits`：限制回调负载（`maxResults`、`maxSnippetChars`、`maxInjectedChars`、`timeoutMs`）。
- `scope`：与 [`session.sendPolicy`](/gateway/configuration-reference#session) 相同的模式。默认仅限私信（`deny` 全部，`allow` 直接聊天）；放宽它以在群组/渠道中显示 QMD 命中。
  - `match.keyPrefix` 匹配**标准化**的会话密钥（小写，去除任何前导 `agent:<id>:`）。例如：`discord:channel:`。
  - `match.rawKeyPrefix` 匹配**原始**会话密钥（小写），包括 `agent:<id>:`。例如：`agent:main:discord:`。
  - 遗留：`match.keyPrefix: "agent:..."` 仍被视为原始密钥前缀，但为了清晰起见，建议使用 `rawKeyPrefix`。
- 当 `scope` 拒绝搜索时，OpenClaw 记录包含派生 `channel`/`chatType` 的警告，以便更容易调试空结果。
- 来自工作区外部的片段在 `memory_search` 结果中显示为 `qmd/<collection>/<relative-path>`；`memory_get` 理解该前缀并从配置的 QMD 集合根读取。
- 当 `memory.qmd.sessions.enabled = true` 时，OpenClaw 将清理的会话记录（用户/助手轮次）导出到 `~/.openclaw/agents/<id>/qmd/sessions/` 下的专用 QMD 集合中，因此 `memory_search` 可以回忆最近的对话而无需触及内置 SQLite 索引。
- 当 `memory.citations` 为 `auto`/`on` 时，`memory_search` 片段现在包含 `Source: <path#line>` 页脚；设置 `memory.citations = "off"` 以保持路径元数据内部（智能体仍接收用于 `memory_get` 的路径，但片段文本省略页脚，系统提示词警告智能体不要引用它）。

### QMD 示例

```json5
memory: {
  backend: "qmd",
  citations: "auto",
  qmd: {
    includeDefaultMemory: true,
    update: { interval: "5m", debounceMs: 15000 },
    limits: { maxResults: 6, timeoutMs: 4000 },
    scope: {
      default: "deny",
      rules: [
        { action: "allow", match: { chatType: "direct" } },
        // 标准化会话密钥前缀（去除 `agent:<id>:`）。
        { action: "deny", match: { keyPrefix: "discord:channel:" } },
        // 原始会话密钥前缀（包含 `agent:<id>:`）。
        { action: "deny", match: { rawKeyPrefix: "agent:main:discord:" } },
      ]
    },
    paths: [
      { name: "docs", path: "~/notes", pattern: "**/*.md" }
    ]
  }
}
```

### 引用和回退

- `memory.citations` 无论后端如何都适用（`auto`/`on`/`off`）。
- 当 `qmd` 运行时，我们标记 `status().backend = "qmd"` 以便诊断显示哪个引擎提供了结果。如果 QMD 子进程退出或无法解析 JSON 输出，搜索管理器记录警告并返回内置提供商（现有 Markdown 嵌入），直到 QMD 恢复。

## 额外内存路径

如果您想在默认工作区布局外索引 Markdown 文件，添加显式路径：

```json5
agents: {
  defaults: {
    memorySearch: {
      extraPaths: ["../team-docs", "/srv/shared-notes/overview.md"]
    }
  }
}
```

注意：

- 路径可以是绝对路径或工作区相对路径。
- 目录递归扫描 `.md` 文件。
- 默认情况下，仅索引 Markdown 文件。
- 如果 `memorySearch.multimodal.enabled = true`，OpenClaw 也会仅在 `extraPaths` 下索引支持的图像/音频文件。默认内存根（`MEMORY.md`、`memory.md`、`memory/**/*.md`）保持仅 Markdown。
- 忽略符号链接（文件或目录）。

## 多模态内存文件（Gemini 图像 + 音频）

使用 Gemini embedding 2 时，OpenClaw 可以索引来自 `memorySearch.extraPaths` 的图像和音频文件：

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "gemini",
      model: "gemini-embedding-2-preview",
      extraPaths: ["assets/reference", "voice-notes"],
      multimodal: {
        enabled: true,
        modalities: ["image", "audio"], // 或 ["all"]
        maxFileBytes: 10000000
      },
      remote: {
        apiKey: "YOUR_GEMINI_API_KEY"
      }
    }
  }
}
```

注意：

- 多模态内存目前仅支持 `gemini-embedding-2-preview`。
- 多模态索引仅适用于通过 `memorySearch.extraPaths` 发现的文件。
- 此阶段支持的模态：图像和音频。
- 启用多模态内存时，`memorySearch.fallback` 必须保持 `"none"`。
- 在索引期间，匹配的图像/音频文件字节被上传到配置的 Gemini 嵌入端点。
- 支持的图像扩展名：`.jpg`、`.jpeg`、`.png`、`.webp`、`.gif`、`.heic`、`.heif`。
- 支持的音频扩展名：`.mp3`、`.wav`、`.ogg`、`.opus`、`.m4a`、`.aac`、`.flac`。
- 搜索查询保持文本形式，但 Gemini 可以将这些文本查询与索引的图像/音频嵌入进行比较。
- `memory_get` 仍仅读取 Markdown；二进制文件可搜索但不作为原始文件内容返回。

## Gemini 嵌入（原生）

将提供商设置为 `gemini` 以直接使用 Gemini 嵌入 API：

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "gemini",
      model: "gemini-embedding-001",
      remote: {
        apiKey: "YOUR_GEMINI_API_KEY"
      }
    }
  }
}
```

注意：

- `remote.baseUrl` 是可选的（默认为 Gemini API 基础 URL）。
- `remote.headers` 允许您在需要时添加额外标头。
- 默认模型：`gemini-embedding-001`。
- 也支持 `gemini-embedding-2-preview`：8192 令牌限制和可配置维度（768 / 1536 / 3072，默认 3072）。

### Gemini Embedding 2（预览版）

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "gemini",
      model: "gemini-embedding-2-preview",
      outputDimensionality: 3072,  // 可选：768、1536 或 3072（默认）
      remote: {
        apiKey: "YOUR_GEMINI_API_KEY"
      }
    }
  }
}
```

> **需要重新索引：** 从 `gemini-embedding-001`（768 维度）切换到 `gemini-embedding-2-preview`（3072 维度）会改变向量大小。如果您在 768、1536 和 3072 之间更改 `outputDimensionality` 也是如此。
> 当 OpenClaw 检测到模型或维度变更时会自动重新索引。

## 自定义 OpenAI 兼容端点

对于自托管或不同的 OpenAI 兼容嵌入服务：

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "openai",
      model: "text-embedding-3-small",
      remote: {
        baseUrl: "https://your-embedding-server.com/v1",
        apiKey: "YOUR_API_KEY",
        headers: {
          "Custom-Header": "value"
        }
      }
    }
  }
}
```

注意：

- 使用 OpenAI `/v1/embeddings` 格式。
- 所有 `remote.*` 字段都是可选的；未设置时使用 OpenAI 默认值。

## 本地嵌入（node-llama-cpp）

通过本地 GGUF 模型运行嵌入：

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "local",
      local: {
        modelPath: "~/Downloads/bge-base-en-v1.5-q8_0.gguf"
      }
    }
  }
}
```

注意：

- 需要兼容的 GGUF 嵌入模型。
- 首次使用可能需要 `pnpm approve-builds` 来编译本机依赖项。
- 使用 node-llama-cpp 作为推理引擎。

## Voyage AI 嵌入

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "voyage",
      model: "voyage-large-2",
      remote: {
        apiKey: "YOUR_VOYAGE_API_KEY"
      }
    }
  }
}
```

支持的模型：`voyage-large-2`、`voyage-code-2`、`voyage-2`、`voyage-large-2-instruct`。

## Mistral 嵌入

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "mistral",
      model: "mistral-embed",
      remote: {
        apiKey: "YOUR_MISTRAL_API_KEY"
      }
    }
  }
}
```

## Ollama 嵌入

对于本地 Ollama 实例：

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "ollama",
      model: "nomic-embed-text",
      remote: {
        baseUrl: "http://localhost:11434",
        apiKey: "ollama-local"  // 可选占位符
      }
    }
  }
}
```

注意：

- 使用 Ollama 的 `/api/embeddings` 端点。
- 确保模型已通过 `ollama pull nomic-embed-text` 下载。

## 混合搜索和重排序

启用混合搜索以结合语义（向量）和词汇（BM25）匹配：

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "openai",
      hybridSearch: {
        enabled: true,
        semanticWeight: 0.7,
        lexicalWeight: 0.3,
        rerankingModel: "cross-encoder/ms-marco-MiniLM-L-6-v2"
      }
    }
  }
}
```

## 最大边际相关性（MMR）

通过 MMR 减少结果多样性来减少冗余：

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "openai",
      mmr: {
        enabled: true,
        diversityLambda: 0.7,  // 0.0 = 最大多样性，1.0 = 最大相关性
        fetchK: 20  // 重排序前获取的候选数量
      }
    }
  }
}
```

## 时间衰减

按年龄对搜索结果进行加权，偏向较新内容：

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "openai",
      temporalDecay: {
        enabled: true,
        halfLife: "30d",  // 分数减半的时间
        decayFactor: 0.5  // 衰减强度
      }
    }
  }
}
```

## 高级配置

### 搜索限制和超时

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "openai",
      limits: {
        maxResults: 10,
        maxSnippetChars: 2000,
        maxInjectedChars: 8000,
        timeoutMs: 5000
      }
    }
  }
}
```

### 回退提供商

如果主要提供商失败，配置回退：

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "openai",
      fallback: "local",
      local: {
        modelPath: "~/models/backup-embedding.gguf"
      }
    }
  }
}
```

### 索引配置

控制索引行为：

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "openai",
      indexing: {
        chunkSize: 1000,
        chunkOverlap: 200,
        batchSize: 100,
        debounceMs: 5000
      }
    }
  }
}
```

## 诊断和监控

启用详细的内存搜索日志记录：

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "openai",
      diagnostics: {
        enabled: true,
        logLevel: "debug",
        traceQueries: true,
        dumpIndex: false
      }
    }
  }
}
```

## 环境变量

所有内存搜索配置也可以通过环境变量设置：

- `OPENCLAW_MEMORY_PROVIDER` - 设置默认提供商
- `OPENCLAW_MEMORY_MODEL` - 设置默认模型
- `GEMINI_API_KEY` - Gemini 嵌入
- `VOYAGE_API_KEY` - Voyage AI 嵌入
- `MISTRAL_API_KEY` - Mistral 嵌入
- `OLLAMA_API_KEY` - Ollama API 密钥（可选）

## 故障排除

### 常见问题

1. **内存搜索不工作**
   - 检查是否配置了提供商和 API 密钥
   - 验证内存文件是否存在且可读
   - 检查 Gateway 日志中的错误

2. **搜索结果差**
   - 尝试不同的嵌入模型
   - 启用混合搜索
   - 调整分块大小和重叠

3. **性能问题**
   - 增加批处理大小
   - 减少最大结果
   - 使用本地嵌入以降低延迟

### 调试命令

```bash
# 检查内存搜索状态
openclaw memory status

# 测试搜索查询
openclaw memory search "your query"

# 重建索引
openclaw memory reindex

# 显示内存统计
openclaw memory stats
```

相关文档：

- [内存概念](/concepts/memory)
- [Gateway 配置参考](/gateway/configuration-reference)
- [插件 SDK 概览](/plugins/sdk-overview)