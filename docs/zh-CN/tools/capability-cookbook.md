---
read_when:
  - 向 OpenClaw 插件系统添加新的共享能力
  - 决定代码属于核心、供应商插件还是功能插件
  - 为渠道或工具连接新的运行时辅助工具
sidebarTitle: 添加能力
summary: 向 OpenClaw 插件系统添加新共享能力的贡献者指南
title: 添加能力（贡献者指南）
x-i18n:
  generated_at: "2026-03-29T11:37:15Z"
  model: claude-sonnet-4-6
  provider: pi
  source_hash: 995cf9ec293ff485ff6e26fea106b3c168eedc8a1de38e29c169d20403644bfc
  source_path: tools/capability-cookbook.md
  workflow: 15
---

# 添加能力

<Info>
  这是 OpenClaw 核心开发者的**贡献者指南**。如果你正在构建外部插件，请参阅[构建插件](/plugins/building-plugins)。
</Info>

当 OpenClaw 需要一个新的领域（如图像生成、视频生成或某个未来的供应商支持功能区域）时使用本指南。

规则：

- 插件 = 所有权边界
- 能力 = 共享核心合约

这意味着你不应该一开始就将供应商直接连接到渠道或工具中。先定义能力。

## 何时创建能力

当以下所有条件都成立时，创建新能力：

1. 不止一个供应商可以合理地实现它
2. 渠道、工具或功能插件应该在不关心供应商的情况下使用它
3. 核心需要拥有回退、策略、配置或传递行为

如果工作仅针对某个供应商且尚无共享合约，请停下来先定义合约。

## 标准步骤

1. 定义类型化的核心合约。
2. 为该合约添加插件注册。
3. 添加共享运行时辅助工具。
4. 连接一个真实的供应商插件作为验证。
5. 将功能/渠道消费者迁移到运行时辅助工具。
6. 添加合约测试。
7. 为操作员面向的配置和所有权模型编写文档。

## 代码归属

核心：

- 请求/响应类型
- 提供商注册表 + 解析
- 回退行为
- 配置 schema 及标签/帮助
- 运行时辅助工具界面

供应商插件：

- 供应商 API 调用
- 供应商认证处理
- 供应商专用请求规范化
- 能力实现的注册

功能/渠道插件：

- 调用 `api.runtime.*` 或对应的 `plugin-sdk/*-runtime` 辅助工具
- 永远不直接调用供应商实现

## 文件清单

对于新能力，预计需要修改以下区域：

- `src/<capability>/types.ts`
- `src/<capability>/...registry/runtime.ts`
- `src/plugins/types.ts`
- `src/plugins/registry.ts`
- `src/plugins/captured-registration.ts`
- `src/plugins/contracts/registry.ts`
- `src/plugins/runtime/types-core.ts`
- `src/plugins/runtime/index.ts`
- `src/plugin-sdk/<capability>.ts`
- `src/plugin-sdk/<capability>-runtime.ts`
- 一个或多个内置插件包
- 配置/文档/测试

## 示例：图像生成

图像生成遵循标准结构：

1. 核心定义 `ImageGenerationProvider`
2. 核心暴露 `registerImageGenerationProvider(...)`
3. 核心暴露 `runtime.imageGeneration.generate(...)`
4. `openai` 和 `google` 插件注册供应商支持的实现
5. 未来的供应商可以注册相同合约，无需更改渠道/工具

配置键与视觉分析路由分开：

- `agents.defaults.imageModel` = 分析图像
- `agents.defaults.imageGenerationModel` = 生成图像

保持两者分离，使回退和策略保持明确。

## 审查清单

在发布新能力之前，验证：

- 没有渠道/工具直接导入供应商代码
- 运行时辅助工具是共享路径
- 至少有一个合约测试断言内置所有权
- 配置文档命名了新的模型/配置键
- 插件文档解释了所有权边界

如果 PR 跳过能力层并将供应商行为硬编码到渠道/工具中，请退回并先定义合约。
