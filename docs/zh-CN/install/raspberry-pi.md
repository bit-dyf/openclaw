---
summary: "在 Raspberry Pi 上托管 OpenClaw,实现始终在线的自托管"
read_when:
  - 在 Raspberry Pi 上设置 OpenClaw
  - 在 ARM 设备上运行 OpenClaw
  - 构建廉价的始终在线个人 AI
title: "Raspberry Pi"
x-i18n:
  generated_at: "2025-01-09T00:00:00Z"
  model: "claude-sonnet-4-6"
  provider: "pi"
  source_hash: "6afe3fcd2ed3e868990e81d409d73a84d72b628be557fc83b2530483fff898bf"
  source_path: "docs/install/raspberry-pi.md"
  workflow: 15
---

# Raspberry Pi

在 Raspberry Pi 上运行持久化的、始终在线的 OpenClaw Gateway 网关。由于 Pi 只是 Gateway 网关(模型通过 API 在云中运行),即使是适度的 Pi 也能很好地处理工作负载。

## 前提条件

- 具有 2 GB+ RAM 的 Raspberry Pi 4 或 5(推荐 4 GB)
- MicroSD 卡(16 GB+)或 USB SSD(性能更好)
- 官方 Pi 电源适配器
- 网络连接(以太网或 WiFi)
- 64 位 Raspberry Pi OS(必需 -- 不要使用 32 位)
- 约 30 分钟

## 设置

<Steps>
  <Step title="刷写操作系统">
    使用 **Raspberry Pi OS Lite(64 位)** -- 无头服务器不需要桌面。

    1. 下载 [Raspberry Pi Imager](https://www.raspberrypi.com/software/)。
    2. 选择操作系统:**Raspberry Pi OS Lite(64 位)**。
    3. 在设置对话框中,预配置:
       - 主机名:`gateway-host`
       - 启用 SSH
       - 设置用户名和密码
       - 配置 WiFi(如果不使用以太网)
    4. 刷写到您的 SD 卡或 USB 驱动器,插入并启动 Pi。

  </Step>

  <Step title="通过 SSH 连接">
    ```bash
    ssh user@gateway-host
    ```
  </Step>

  <Step title="更新系统">
    ```bash
    sudo apt update && sudo apt upgrade -y
    sudo apt install -y git curl build-essential

    # 设置时区(对于 cron 和提醒很重要)
    sudo timedatectl set-timezone America/Chicago
    ```

  </Step>

  <Step title="安装 Node.js 24">
    ```bash
    curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
    sudo apt install -y nodejs
    node --version
    ```
  </Step>

  <Step title="添加交换空间(对于 2 GB 或更少很重要)">
    ```bash
    sudo fallocate -l 2G /swapfile
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile
    echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

    # 降低低 RAM 设备的交换倾向性
    echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
    sudo sysctl -p
    ```

  </Step>

  <Step title="安装 OpenClaw">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    ```
  </Step>

  <Step title="运行新手引导">
    ```bash
    openclaw onboard --install-daemon
    ```

    遵循向导。对于无头设备,推荐使用 API 密钥而不是 OAuth。Telegram 是最容易开始使用的渠道。

  </Step>

  <Step title="验证">
    ```bash
    openclaw status
    sudo systemctl status openclaw
    journalctl -u openclaw -f
    ```
  </Step>

  <Step title="访问控制 UI">
    在您的计算机上,从 Pi 获取仪表板 URL:

    ```bash
    ssh user@gateway-host 'openclaw dashboard --no-open'
    ```

    然后在另一个终端中创建 SSH 隧道:

    ```bash
    ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
    ```

    在本地浏览器中打开打印的 URL。对于始终在线的远程访问,请参阅 [Tailscale 集成](/gateway/tailscale)。

  </Step>
</Steps>

## 性能提示

**使用 USB SSD** -- SD 卡速度慢且会磨损。USB SSD 显著提高性能。请参阅 [Pi USB 启动指南](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#usb-mass-storage-boot)。

**启用模块编译缓存** -- 加速低功率 Pi 主机上重复的 CLI 调用:

```bash
grep -q 'NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache' ~/.bashrc || cat >> ~/.bashrc <<'EOF' # pragma: allowlist secret
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.bashrc
```

**减少内存使用** -- 对于无头设置,释放 GPU 内存并禁用未使用的服务:

```bash
echo 'gpu_mem=16' | sudo tee -a /boot/config.txt
sudo systemctl disable bluetooth
```

## 故障排除

**内存不足** -- 使用 `free -h` 验证交换空间是否激活。禁用未使用的服务(`sudo systemctl disable cups bluetooth avahi-daemon`)。仅使用基于 API 的模型。

**性能慢** -- 使用 USB SSD 而不是 SD 卡。使用 `vcgencmd get_throttled` 检查 CPU 限流(应返回 `0x0`)。

**服务无法启动** -- 使用 `journalctl -u openclaw --no-pager -n 100` 检查日志并运行 `openclaw doctor --non-interactive`。

**ARM 二进制文件问题** -- 如果 Skill 失败并显示"exec format error",请检查二进制文件是否有 ARM64 构建。使用 `uname -m` 验证架构(应显示 `aarch64`)。

**WiFi 掉线** -- 禁用 WiFi 电源管理:`sudo iwconfig wlan0 power off`。

## 下一步

- [渠道](/channels) -- 连接 Telegram、WhatsApp、Discord 等
- [Gateway 网关配置](/gateway/configuration) -- 所有配置选项
- [更新](/install/updating) -- 保持 OpenClaw 最新
