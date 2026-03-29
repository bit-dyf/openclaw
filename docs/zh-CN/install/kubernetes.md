---
summary: "使用 Kustomize 将 OpenClaw Gateway 网关部署到 Kubernetes 集群"
read_when:
  - 您希望在 Kubernetes 集群上运行 OpenClaw
  - 您希望在 Kubernetes 环境中测试 OpenClaw
title: "Kubernetes"
x-i18n:
  generated_at: "2025-01-09T00:00:00Z"
  model: "claude-sonnet-4-6"
  provider: "pi"
  source_hash: "968fa9bd4cf31da9d50b96ac8dadea024a8a340f640e42274be67a54a6c80537"
  source_path: "docs/install/kubernetes.md"
  workflow: 15
---

# 在 Kubernetes 上运行 OpenClaw

在 Kubernetes 上运行 OpenClaw 的最小起点 — 不是生产就绪的部署。它涵盖核心资源,旨在适应您的环境。

## 为什么不使用 Helm?

OpenClaw 是一个带有一些配置文件的单一容器。有趣的自定义在于智能体内容(markdown 文件、Skills、配置覆盖),而不是基础架构模板。Kustomize 无需 Helm chart 的开销即可处理覆盖。如果您的部署变得更加复杂,可以在这些清单之上添加 Helm chart。

## 您需要准备的内容

- 正在运行的 Kubernetes 集群(AKS、EKS、GKE、k3s、kind、OpenShift 等)
- `kubectl` 已连接到您的集群
- 至少一个模型提供商的 API 密钥

## 快速开始

```bash
# 替换为您的提供商:ANTHROPIC、GEMINI、OPENAI 或 OPENROUTER
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh

kubectl port-forward svc/openclaw 18789:18789 -n openclaw
open http://localhost:18789
```

检索 Gateway 网关令牌并将其粘贴到控制 UI 中:

```bash
kubectl get secret openclaw-secrets -n openclaw -o jsonpath='{.data.OPENCLAW_GATEWAY_TOKEN}' | base64 -d
```

对于本地调试,`./scripts/k8s/deploy.sh --show-token` 会在部署后打印令牌。

## 使用 Kind 进行本地测试

如果您没有集群,请使用 [Kind](https://kind.sigs.k8s.io/) 在本地创建一个:

```bash
./scripts/k8s/create-kind.sh           # 自动检测 docker 或 podman
./scripts/k8s/create-kind.sh --delete  # 拆除
```

然后照常使用 `./scripts/k8s/deploy.sh` 进行部署。

## 分步指南

### 1) 部署

**选项 A** — 环境中的 API 密钥(一步):

```bash
# 替换为您的提供商:ANTHROPIC、GEMINI、OPENAI 或 OPENROUTER
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh
```

该脚本使用 API 密钥和自动生成的 Gateway 网关令牌创建 Kubernetes Secret,然后进行部署。如果 Secret 已存在,它会保留当前的 Gateway 网关令牌和任何未更改的提供商密钥。

**选项 B** — 单独创建 Secret:

```bash
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh --create-secret
./scripts/k8s/deploy.sh
```

如果您想为本地测试将令牌打印到 stdout,请对任一命令使用 `--show-token`。

### 2) 访问 Gateway 网关

```bash
kubectl port-forward svc/openclaw 18789:18789 -n openclaw
open http://localhost:18789
```

## 部署内容

```
Namespace: openclaw(可通过 OPENCLAW_NAMESPACE 配置)
├── Deployment/openclaw        # 单个 pod,初始化容器 + Gateway 网关
├── Service/openclaw           # ClusterIP,端口 18789
├── PersistentVolumeClaim      # 10Gi 用于智能体状态和配置
├── ConfigMap/openclaw-config  # openclaw.json + AGENTS.md
└── Secret/openclaw-secrets    # Gateway 网关令牌 + API 密钥
```

## 自定义

### 智能体说明

编辑 `scripts/k8s/manifests/configmap.yaml` 中的 `AGENTS.md` 并重新部署:

```bash
./scripts/k8s/deploy.sh
```

### Gateway 网关配置

编辑 `scripts/k8s/manifests/configmap.yaml` 中的 `openclaw.json`。有关完整参考,请参阅 [Gateway 网关配置](/gateway/configuration)。

### 添加提供商

使用导出的额外密钥重新运行:

```bash
export ANTHROPIC_API_KEY="..."
export OPENAI_API_KEY="..."
./scripts/k8s/deploy.sh --create-secret
./scripts/k8s/deploy.sh
```

除非您覆盖它们,否则现有提供商密钥会保留在 Secret 中。

或者直接修补 Secret:

```bash
kubectl patch secret openclaw-secrets -n openclaw \
  -p '{"stringData":{"<PROVIDER>_API_KEY":"..."}}'
kubectl rollout restart deployment/openclaw -n openclaw
```

### 自定义命名空间

```bash
OPENCLAW_NAMESPACE=my-namespace ./scripts/k8s/deploy.sh
```

### 自定义镜像

编辑 `scripts/k8s/manifests/deployment.yaml` 中的 `image` 字段:

```yaml
image: ghcr.io/openclaw/openclaw:latest # 或从 https://github.com/openclaw/openclaw/releases 固定到特定版本
```

### 超越端口转发公开

默认清单将 Gateway 网关绑定到 pod 内的回环地址。这适用于 `kubectl port-forward`,但不适用于需要到达 pod IP 的 Kubernetes `Service` 或 Ingress 路径。

如果您想通过 Ingress 或负载均衡器公开 Gateway 网关:

- 将 `scripts/k8s/manifests/configmap.yaml` 中的 Gateway 网关绑定从 `loopback` 更改为与您的部署模型匹配的非回环绑定
- 保持启用 Gateway 网关认证并使用适当的 TLS 终止入口点
- 使用支持的 Web 安全模型(例如 HTTPS/Tailscale Serve 以及在需要时显式允许的源)为远程访问配置控制 UI

## 重新部署

```bash
./scripts/k8s/deploy.sh
```

这将应用所有清单并重新启动 pod 以获取任何配置或 Secret 更改。

## 拆除

```bash
./scripts/k8s/deploy.sh --delete
```

这将删除命名空间及其中的所有资源,包括 PVC。

## 架构说明

- 默认情况下,Gateway 网关绑定到 pod 内的回环地址,因此包含的设置适用于 `kubectl port-forward`
- 没有集群范围的资源 — 一切都在单个命名空间中
- 安全性:`readOnlyRootFilesystem`、`drop: ALL` 功能、非 root 用户(UID 1000)
- 默认配置将控制 UI 保持在更安全的本地访问路径上:回环绑定加上 `kubectl port-forward` 到 `http://127.0.0.1:18789`
- 如果您超越 localhost 访问,请使用支持的远程模型:HTTPS/Tailscale 加上适当的 Gateway 网关绑定和控制 UI 源设置
- Secrets 在临时目录中生成并直接应用于集群 — 没有 Secret 材料写入 repo checkout

## 文件结构

```
scripts/k8s/
├── deploy.sh                   # 创建命名空间 + Secret,通过 kustomize 部署
├── create-kind.sh              # 本地 Kind 集群(自动检测 docker/podman)
└── manifests/
    ├── kustomization.yaml      # Kustomize 基础
    ├── configmap.yaml          # openclaw.json + AGENTS.md
    ├── deployment.yaml         # 具有安全加固的 Pod 规范
    ├── pvc.yaml                # 10Gi 持久化存储
    └── service.yaml            # ClusterIP,端口 18789
```
