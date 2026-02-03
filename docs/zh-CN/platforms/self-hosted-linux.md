---
summary: "在自托管的 Linux 虚拟机上安装 OpenClaw"
read_when:
  - 您想在自己的 Linux 服务器或虚拟机上运行 OpenClaw
  - 在自托管基础设施上设置 OpenClaw
  - 寻找通用的 Linux 安装指南
title: "自托管 Linux 虚拟机"
---

# 自托管 Linux 虚拟机

本指南介绍如何在自托管的 Linux 虚拟机或服务器上安装 OpenClaw，无论是在您自己的硬件、家庭实验室还是通用 VPS 提供商上运行。

## 目标

在您自己的 Linux 虚拟机上运行持久的 OpenClaw 网关，并从您的设备远程访问。

## 先决条件

- Linux 虚拟机或服务器（Ubuntu 22.04+、Debian 11+ 或类似系统）
- 对虚拟机的 SSH 访问
- Root 或 sudo 权限
- 约 30 分钟

## 支持的 Linux 发行版

OpenClaw 适用于大多数支持 Node.js 22+ 的现代 Linux 发行版：

- Ubuntu 22.04 LTS、24.04 LTS
- Debian 11 (Bullseye)、12 (Bookworm)
- Fedora 38+
- Rocky Linux 9+
- Arch Linux（滚动发布）

## 快速开始

对于希望快速启动的经验丰富的用户：

```bash
# 1. 安装 Node.js 22+（如果尚未安装）
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs

# 2. 安装 OpenClaw
curl -fsSL https://openclaw.ai/install.sh | bash

# 3. 运行入门向导
openclaw onboard --install-daemon

# 4. 从您的笔记本电脑访问（SSH 隧道）
ssh -N -L 18789:127.0.0.1:18789 user@your-vm-host

# 5. 打开 http://localhost:18789 并粘贴您的令牌
```

## 分步安装

### 1) 连接到您的虚拟机

```bash
ssh user@your-vm-host
```

将 `user` 和 `your-vm-host` 替换为您的实际用户名和主机名或 IP 地址。

### 2) 更新系统软件包

```bash
sudo apt update && sudo apt upgrade -y
```

对于基于 RedHat 的系统（Fedora、Rocky Linux）：

```bash
sudo dnf update -y
```

### 3) 安装 Node.js 22+

OpenClaw 需要 Node.js 22 或更高版本。

#### 选项 A：使用 NodeSource（Ubuntu/Debian）

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs
```

#### 选项 B：使用 NodeSource（Fedora/Rocky）

```bash
curl -fsSL https://rpm.nodesource.com/setup_22.x | sudo bash -
sudo dnf install -y nodejs
```

#### 选项 C：使用 nvm（任何发行版）

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 22
nvm use 22
```

验证安装：

```bash
node -v  # 应显示 v22.x.x 或更高版本
npm -v
```

### 4) 安装构建工具（原生模块所需）

```bash
# Ubuntu/Debian
sudo apt-get install -y build-essential python3

# Fedora/Rocky
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y python3
```

### 5) 安装 OpenClaw

运行 OpenClaw 安装程序：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

安装程序将：
- 通过 npm 全局安装 `openclaw` CLI
- 如需要，设置 PATH
- 运行入门向导（可选）

如果您想在安装期间跳过入门：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --no-onboard
```

### 6) 运行入门向导

如果您跳过了入门或想要重新配置：

```bash
openclaw onboard --install-daemon
```

入门向导将引导您完成：
- 设置身份验证（API 密钥/令牌）
- 配置消息通道
- 将网关安装为 systemd 服务
- 测试您的设置

当询问"如何孵化您的机器人？"时，您可以：
- 立即配置通道，或
- 选择 **"稍后执行"** 并通过控制 UI 配置

### 7) 配置网关以进行远程访问

默认情况下，网关绑定到环回（仅本地主机）。对于远程访问，您有几个选项：

#### 选项 A：SSH 隧道（推荐用于安全性）

将网关保留在环回上并使用 SSH 隧道：

```bash
# 从您的本地计算机
ssh -N -L 18789:127.0.0.1:18789 user@your-vm-host
```

然后在本地计算机上访问 `http://localhost:18789`。

#### 选项 B：Tailscale（推荐用于轻松访问）

安装 Tailscale 以实现安全的远程访问：

```bash
# 在虚拟机上
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh --hostname=openclaw-gateway

# 为 Tailscale 配置 OpenClaw
openclaw config set gateway.tailscale.mode serve
openclaw config set gateway.trustedProxies '["127.0.0.1"]'
systemctl --user restart openclaw-gateway
```

从 Tailscale 网络上的任何设备访问：
```
https://openclaw-gateway.<tailnet-name>.ts.net/
```

#### 选项 C：直接网络绑定（安全性较低）

如果您必须绑定到网络接口，请使用令牌身份验证：

```bash
openclaw config set gateway.bind lan
openclaw config set gateway.auth.mode token
openclaw doctor --generate-gateway-token
systemctl --user restart openclaw-gateway
```

**重要提示：** 如果绑定到 `lan`，请确保您的虚拟机防火墙仅允许来自受信任 IP 的连接。

### 8) 配置 Systemd 服务

安装程序应该已经创建了一个 systemd 用户服务。验证它正在运行：

```bash
systemctl --user status openclaw-gateway
```

如果您需要启用 lingering（在注销后保持服务运行）：

```bash
sudo loginctl enable-linger $USER
```

常用 systemd 命令：

```bash
# 检查状态
systemctl --user status openclaw-gateway

# 启动/停止/重启
systemctl --user start openclaw-gateway
systemctl --user stop openclaw-gateway
systemctl --user restart openclaw-gateway

# 查看日志
journalctl --user -u openclaw-gateway -f
```

### 9) 验证安装

```bash
# 检查版本
openclaw --version

# 检查网关健康状况
openclaw health

# 检查网关状态
openclaw status

# 运行诊断
openclaw doctor
```

测试本地访问：

```bash
curl http://localhost:18789
```

有关完整的配置选项、安全最佳实践、故障排除和高级用例，请参阅[英文完整指南](/platforms/self-hosted-linux)。

## 另请参阅

- [入门指南](/zh-CN/start/getting-started) — 首次设置指南
- [网关配置](/zh-CN/gateway/configuration) — 所有配置选项
- [平台特定指南](/zh-CN/platforms) — Oracle、Hetzner、GCP 等
- [更新](/zh-CN/install/updating) — 如何更新 OpenClaw
- [VPS 托管](/vps) — VPS 提供商比较
