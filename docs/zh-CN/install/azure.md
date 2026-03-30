---
summary: "在 Azure Linux 虚拟机上全天候运行 OpenClaw Gateway 网关并保持持久化状态"
read_when:
  - 您希望在 Azure 上全天候运行 OpenClaw,并通过网络安全组进行加固
  - 您希望在自己的 Azure Linux 虚拟机上搭建生产级、始终在线的 OpenClaw Gateway 网关
  - 您希望使用 Azure Bastion SSH 进行安全管理
title: "Azure"
x-i18n:
  generated_at: "2025-01-09T00:00:00Z"
  model: "claude-sonnet-4-6"
  provider: "pi"
  source_hash: "dcdcf6dcf5096cd21e1b64f455656f7d77b477d03e9a088db74c6e988c3031db"
  source_path: "docs/install/azure.md"
  workflow: 15
---

# 在 Azure Linux 虚拟机上运行 OpenClaw

本指南介绍如何使用 Azure CLI 设置 Azure Linux 虚拟机,应用网络安全组 (NSG) 加固,配置 Azure Bastion 进行 SSH 访问,并安装 OpenClaw。

## 您将要做的事情

- 使用 Azure CLI 创建 Azure 网络资源(虚拟网络、子网、网络安全组)和计算资源
- 应用网络安全组规则,使虚拟机 SSH 仅允许来自 Azure Bastion 的访问
- 使用 Azure Bastion 进行 SSH 访问(虚拟机上无公共 IP)
- 使用安装脚本安装 OpenClaw
- 验证 Gateway 网关

## 您需要准备的内容

- 具有创建计算和网络资源权限的 Azure 订阅
- 已安装 Azure CLI(如需要,请参阅 [Azure CLI 安装步骤](https://learn.microsoft.com/cli/azure/install-azure-cli))
- SSH 密钥对(本指南涵盖了在需要时如何生成密钥对)
- 约 20-30 分钟

## 配置部署

<Steps>
  <Step title="登录到 Azure CLI">
    ```bash
    az login
    az extension add -n ssh
    ```

    Azure Bastion 原生 SSH 隧道需要 `ssh` 扩展。

  </Step>

  <Step title="注册所需的资源提供商(一次性操作)">
    ```bash
    az provider register --namespace Microsoft.Compute
    az provider register --namespace Microsoft.Network
    ```

    验证注册状态。等待两者都显示 `Registered`。

    ```bash
    az provider show --namespace Microsoft.Compute --query registrationState -o tsv
    az provider show --namespace Microsoft.Network --query registrationState -o tsv
    ```

  </Step>

  <Step title="设置部署变量">
    ```bash
    RG="rg-openclaw"
    LOCATION="westus2"
    VNET_NAME="vnet-openclaw"
    VNET_PREFIX="10.40.0.0/16"
    VM_SUBNET_NAME="snet-openclaw-vm"
    VM_SUBNET_PREFIX="10.40.2.0/24"
    BASTION_SUBNET_PREFIX="10.40.1.0/26"
    NSG_NAME="nsg-openclaw-vm"
    VM_NAME="vm-openclaw"
    ADMIN_USERNAME="openclaw"
    BASTION_NAME="bas-openclaw"
    BASTION_PIP_NAME="pip-openclaw-bastion"
    ```

    根据您的环境调整名称和 CIDR 范围。Bastion 子网必须至少为 `/26`。

  </Step>

  <Step title="选择 SSH 密钥">
    如果您已有公钥,请使用现有公钥:

    ```bash
    SSH_PUB_KEY="$(cat ~/.ssh/id_ed25519.pub)"
    ```

    如果您还没有 SSH 密钥,请生成一个:

    ```bash
    ssh-keygen -t ed25519 -a 100 -f ~/.ssh/id_ed25519 -C "you@example.com"
    SSH_PUB_KEY="$(cat ~/.ssh/id_ed25519.pub)"
    ```

  </Step>

  <Step title="选择虚拟机大小和操作系统磁盘大小">
    ```bash
    VM_SIZE="Standard_B2as_v2"
    OS_DISK_SIZE_GB=64
    ```

    选择您的订阅和区域中可用的虚拟机大小和操作系统磁盘大小:

    - 对于轻度使用,从较小规模开始,稍后再扩展
    - 对于更繁重的自动化、更多渠道或更大的模型/工具工作负载,使用更多 vCPU/RAM/磁盘
    - 如果某个虚拟机大小在您的区域或订阅配额中不可用,请选择最接近的可用 SKU

    列出目标区域中可用的虚拟机大小:

    ```bash
    az vm list-skus --location "${LOCATION}" --resource-type virtualMachines -o table
    ```

    检查当前 vCPU 和磁盘使用量/配额:

    ```bash
    az vm list-usage --location "${LOCATION}" -o table
    ```

  </Step>
</Steps>

## 部署 Azure 资源

<Steps>
  <Step title="创建资源组">
    ```bash
    az group create -n "${RG}" -l "${LOCATION}"
    ```
  </Step>

  <Step title="创建网络安全组">
    创建网络安全组并添加规则,使只有 Bastion 子网可以通过 SSH 访问虚拟机。

    ```bash
    az network nsg create \
      -g "${RG}" -n "${NSG_NAME}" -l "${LOCATION}"

    # 仅允许来自 Bastion 子网的 SSH
    az network nsg rule create \
      -g "${RG}" --nsg-name "${NSG_NAME}" \
      -n AllowSshFromBastionSubnet --priority 100 \
      --access Allow --direction Inbound --protocol Tcp \
      --source-address-prefixes "${BASTION_SUBNET_PREFIX}" \
      --destination-port-ranges 22

    # 拒绝来自公共互联网的 SSH
    az network nsg rule create \
      -g "${RG}" --nsg-name "${NSG_NAME}" \
      -n DenyInternetSsh --priority 110 \
      --access Deny --direction Inbound --protocol Tcp \
      --source-address-prefixes Internet \
      --destination-port-ranges 22

    # 拒绝来自其他虚拟网络源的 SSH
    az network nsg rule create \
      -g "${RG}" --nsg-name "${NSG_NAME}" \
      -n DenyVnetSsh --priority 120 \
      --access Deny --direction Inbound --protocol Tcp \
      --source-address-prefixes VirtualNetwork \
      --destination-port-ranges 22
    ```

    规则按优先级(数字最低的最先)进行评估:Bastion 流量在 100 处被允许,然后所有其他 SSH 在 110 和 120 处被阻止。

  </Step>

  <Step title="创建虚拟网络和子网">
    创建虚拟网络和虚拟机子网(附加网络安全组),然后添加 Bastion 子网。

    ```bash
    az network vnet create \
      -g "${RG}" -n "${VNET_NAME}" -l "${LOCATION}" \
      --address-prefixes "${VNET_PREFIX}" \
      --subnet-name "${VM_SUBNET_NAME}" \
      --subnet-prefixes "${VM_SUBNET_PREFIX}"

    # 将网络安全组附加到虚拟机子网
    az network vnet subnet update \
      -g "${RG}" --vnet-name "${VNET_NAME}" \
      -n "${VM_SUBNET_NAME}" --nsg "${NSG_NAME}"

    # AzureBastionSubnet — Azure 要求使用此名称
    az network vnet subnet create \
      -g "${RG}" --vnet-name "${VNET_NAME}" \
      -n AzureBastionSubnet \
      --address-prefixes "${BASTION_SUBNET_PREFIX}"
    ```

  </Step>

  <Step title="创建虚拟机">
    虚拟机没有公共 IP。SSH 访问完全通过 Azure Bastion 进行。

    ```bash
    az vm create \
      -g "${RG}" -n "${VM_NAME}" -l "${LOCATION}" \
      --image "Canonical:ubuntu-24_04-lts:server:latest" \
      --size "${VM_SIZE}" \
      --os-disk-size-gb "${OS_DISK_SIZE_GB}" \
      --storage-sku StandardSSD_LRS \
      --admin-username "${ADMIN_USERNAME}" \
      --ssh-key-values "${SSH_PUB_KEY}" \
      --vnet-name "${VNET_NAME}" \
      --subnet "${VM_SUBNET_NAME}" \
      --public-ip-address "" \
      --nsg ""
    ```

    `--public-ip-address ""` 防止分配公共 IP。`--nsg ""` 跳过创建每个网络接口的网络安全组(子网级网络安全组处理安全)。

    **可重现性:** 上述命令使用 `latest` 作为 Ubuntu 映像。要固定特定版本,请列出可用版本并替换 `latest`:

    ```bash
    az vm image list \
      --publisher Canonical --offer ubuntu-24_04-lts \
      --sku server --all -o table
    ```

  </Step>

  <Step title="创建 Azure Bastion">
    Azure Bastion 提供对虚拟机的托管 SSH 访问,无需公开公共 IP。基于 CLI 的 `az network bastion ssh` 需要具有隧道功能的标准 SKU。

    ```bash
    az network public-ip create \
      -g "${RG}" -n "${BASTION_PIP_NAME}" -l "${LOCATION}" \
      --sku Standard --allocation-method Static

    az network bastion create \
      -g "${RG}" -n "${BASTION_NAME}" -l "${LOCATION}" \
      --vnet-name "${VNET_NAME}" \
      --public-ip-address "${BASTION_PIP_NAME}" \
      --sku Standard --enable-tunneling true
    ```

    Bastion 配置通常需要 5-10 分钟,但在某些区域可能需要长达 15-30 分钟。

  </Step>
</Steps>

## 安装 OpenClaw

<Steps>
  <Step title="通过 Azure Bastion 通过 SSH 连接到虚拟机">
    ```bash
    VM_ID="$(az vm show -g "${RG}" -n "${VM_NAME}" --query id -o tsv)"

    az network bastion ssh \
      --name "${BASTION_NAME}" \
      --resource-group "${RG}" \
      --target-resource-id "${VM_ID}" \
      --auth-type ssh-key \
      --username "${ADMIN_USERNAME}" \
      --ssh-key ~/.ssh/id_ed25519
    ```

  </Step>

  <Step title="安装 OpenClaw(在虚拟机 shell 中)">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh -o /tmp/install.sh
    bash /tmp/install.sh
    rm -f /tmp/install.sh
    ```

    安装程序会在尚未安装的情况下安装 Node LTS 和依赖项,安装 OpenClaw,并启动新手引导向导。详情请参阅[安装](/install)。

  </Step>

  <Step title="验证 Gateway 网关">
    新手引导完成后:

    ```bash
    openclaw gateway status
    ```

    大多数企业 Azure 团队已经拥有 GitHub Copilot 许可证。如果您的情况是这样,我们建议在 OpenClaw 新手引导向导中选择 GitHub Copilot 提供商。请参阅 [GitHub Copilot 提供商](/providers/github-copilot)。

  </Step>
</Steps>

## 成本考虑

Azure Bastion 标准 SKU 每月约 **\$140**,虚拟机 (Standard_B2as_v2) 每月约 **\$55**。

降低成本的方法:

- **在不使用时解除分配虚拟机**(停止计算计费;磁盘费用仍然存在)。OpenClaw Gateway 网关在虚拟机解除分配时将无法访问 — 需要时再重新启动它:

  ```bash
  az vm deallocate -g "${RG}" -n "${VM_NAME}"
  az vm start -g "${RG}" -n "${VM_NAME}"   # 稍后重新启动
  ```

- **在不需要时删除 Bastion** 并在需要 SSH 访问时重新创建。Bastion 是最大的成本组成部分,仅需几分钟即可配置。
- **使用基本 Bastion SKU**(每月约 \$38),如果您只需要基于门户的 SSH 而不需要 CLI 隧道(`az network bastion ssh`)。

## 清理

要删除本指南创建的所有资源:

```bash
az group delete -n "${RG}" --yes --no-wait
```

这将删除资源组及其中的所有内容(虚拟机、虚拟网络、网络安全组、Bastion、公共 IP)。

## 下一步

- 设置消息渠道:[渠道](/channels)
- 将本地设备配对为节点:[节点](/nodes)
- 配置 Gateway 网关:[Gateway 网关配置](/gateway/configuration)
- 有关使用 GitHub Copilot 模型提供商在 Azure 上部署 OpenClaw 的更多详细信息:[在 Azure 上使用 GitHub Copilot 运行 OpenClaw](https://github.com/johnsonshi/openclaw-azure-github-copilot)
