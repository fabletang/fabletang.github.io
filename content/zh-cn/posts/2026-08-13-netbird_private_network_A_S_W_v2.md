+++
title= "Netbird 私有网络搭建完整指南"
description= "Netbirt  private network"
date = "2026-08-13"
tags = [
    "vpn",
]
categories = [
  "技术"
]
series = [
  "工具"
]
+++

# Netbird 私有网络搭建完整指南

## 1. 架构概述

本方案通过在三台设备上部署 Netbird，组建一个安全的私有网状网络：

| 设备 | 角色 | 网络状况 |
| :--- | :--- | :--- |
| **服务器 S** (Linux, 有公网IP) | **控制平面 + 中继节点** | 部署 Netbird 管理服务、信令服务和中继服务 |
| **服务器 A** (Ubuntu, 内网) | **目标服务端** | 安装 Netbird 客户端，加入私有网络 |
| **工作站 W** (macOS, 异地) | **客户端** | 安装 Netbird 客户端，加入私有网络 |

**最终效果：**

- 工作站 W 可通过 `netbird ssh` 或原生 SSH 客户端访问服务器 A
- 服务器 A 上 8080 端口的 Web 服务可通过 `netbird expose` 命令生成公网可访问的 URL

> Netbird 是基于 WireGuard 的开源点对点 VPN 平台，能在分布式基础设施间建立加密的网状网络，无需手动配置防火墙或端口转发。

---

## 2. 前提条件

### 2.1 硬件要求

- **控制平面服务器 S**：至少 1 CPU、2GB 内存
- **中继服务器（可与控制平面共用）**：至少 1 CPU、1GB 内存
- **服务器 A 和工作站 W**：无特殊硬件要求

### 2.2 软件要求

- 服务器 S 和 A 均运行 **Ubuntu**（推荐 20.04 LTS 或更高版本）
- 服务器 S 需安装 **Docker** 和 **Docker Compose**
- 服务器 S 需有一个**域名**（用于 TLS 证书），DNS A 记录指向其公网 IP

### 2.3 防火墙端口要求

**在服务器 S（控制平面）上开放以下端口：**

| 端口 | 协议 | 用途 |
| :--- | :--- | :--- |
| 80 | TCP | HTTP（重定向到 HTTPS） |
| 443 | TCP | HTTPS（控制台、管理 API、信令、中继） |
| 33073 | TCP | 管理服务 gRPC |
| 10000 | TCP | 管理服务 HTTP |
| 33080 | TCP | 中继（WebSocket/QUIC） |
| 3478 | UDP | STUN 服务 |
| 49152-65535 | UDP | TURN 端口范围 |

**在服务器 A 和工作站 W 上：** 无需开放任何入站端口。

---

## 3. 部署 Netbird 控制平面（在服务器 S 上）

### 3.1 安装 Docker 和 Docker Compose

```bash
# 更新软件包索引
sudo apt update

# 安装依赖
sudo apt install ca-certificates curl gnupg lsb-release -y

# 添加 Docker 官方 GPG 密钥
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 添加 Docker 仓库
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装 Docker Engine 和 Compose 插件
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

### 3.2 部署 Netbird 控制平面

```bash
# 创建 Netbird 目录
mkdir -p ~/netbird && cd ~/netbird

# 下载官方快速启动脚本
curl -fsSL https://pkgs.netbird.io/selfhosted/install.sh | bash
```

> 脚本运行过程中会提示你输入域名（如 `netbird.your-domain.com`），请确保该域名的 DNS A 记录已指向服务器 S 的公网 IP。

### 3.3 验证控制平面运行状态

```bash
docker compose ps
```

应看到以下服务均处于 Running 状态：

- management
- signal
- relay
- coturn
- dashboard
- zitadel（身份提供商）

---

## 4. 配置中继服务（在服务器 S 上）

如果控制平面已包含中继服务，可跳过此步。但如果需要将中继服务独立部署或重新配置，请按以下步骤操作。

### 4.1 生成认证密钥

```bash
openssl rand -base64 32
```

保存输出的密钥，后续步骤需要用到。

### 4.2 创建中继配置

```bash
mkdir -p ~/netbird-relay && cd ~/netbird-relay
```

创建 `relay.env` 文件：

```bash
NB_LOG_LEVEL=info
NB_LISTEN_ADDRESS=:443
NB_EXPOSED_ADDRESS=rels://relay.your-domain.com:443
NB_AUTH_SECRET=你生成的密钥
NB_LETSENCRYPT_DOMAINS=relay.your-domain.com
NB_LETSENCRYPT_EMAIL=your-email@example.com
NB_LETSENCRYPT_DATA_DIR=/data/letsencrypt
NB_ENABLE_STUN=true
NB_STUN_PORTS=3478
```

> 注意：将 `relay.your-domain.com` 替换为你的中继域名，`your-email@example.com` 替换为你的邮箱。

### 4.3 创建 docker-compose.yml

```yaml
services:
  relay:
    image: netbirdio/relay:latest
    container_name: netbird-relay
    restart: unless-stopped
    ports:
      - '443:443'
      - '3478:3478/udp'
    env_file:
      - relay.env
    volumes:
      - relay_data:/data
    logging:
      driver: "json-file"
      options:
        max-size: "500m"
        max-file: "2"
volumes:
  relay_data:
```

### 4.4 启动中继服务

```bash
docker compose up -d
```

### 4.5 更新主服务器配置

在控制平面的 `management.json` 中添加中继服务器配置，然后重启管理服务：

```bash
cd ~/netbird
docker compose restart management
```

---

## 5. 安装 Netbird 客户端并加入网络

### 5.1 在服务器 A（Ubuntu）上安装

**方式一：官方安装脚本**

```bash
curl -fsSL https://pkgs.netbird.io/install.sh | sh
```

**方式二：APT 方式安装**

```bash
# 添加仓库
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg -y
curl -sSL https://pkgs.netbird.io/debian/public.key | sudo gpg --dearmor --output /usr/share/keyrings/netbird-archive-keyring.gpg
echo 'deb [signed-by=/usr/share/keyrings/netbird-archive-keyring.gpg] https://pkgs.netbird.io/debian stable main' | sudo tee /etc/apt/sources.list.d/netbird.list

# 安装
sudo apt-get update
sudo apt-get install netbird -y
```

### 5.2 在工作站 W（macOS）上安装

**方式一：下载安装包**

1. 访问 Netbird 官方下载页
2. 下载 macOS 版本的 `.pkg` 安装包
3. 双击安装并按照提示完成

**方式二：使用 Homebrew**

```bash
brew install netbirdio/tap/netbird
```

### 5.3 加入 Netbird 网络

在服务器 A 上：

```bash
sudo netbird up
```

在工作站 W 上：

```bash
netbird up
```

> 首次运行会输出一个登录 URL，在浏览器中打开并完成身份验证（使用你在控制平面配置的身份提供商，如 Zitadel）。

### 5.4 验证连接

```bash
netbird status
```

应能看到所有已连接的设备及其分配的 IP 地址（格式为 `100.x.x.x`）。

---

## 6. 配置 SSH 访问（工作站 W → 服务器 A）

Netbird 提供了内置的 SSH 服务器，无需暴露端口 22 到公网。

### 6.1 在服务器 A 上启用 SSH 服务器

```bash
sudo netbird up --allow-server-ssh
```

### 6.2 在 Netbird 控制台中启用 SSH

1. 登录 Netbird 控制台（https://netbird.your-domain.com）
2. 进入 **Peers** → 选择服务器 A
3. 将 **SSH Access** 开关设置为 **ON**

### 6.3 从工作站 W 发起 SSH 连接

**方式一：使用 netbird ssh 命令**

```bash
netbird ssh username@server-a
```

> 将 `username` 替换为服务器 A 上的用户名，`server-a` 替换为服务器 A 的设备名称或 IP 地址。

**方式二：使用原生 OpenSSH 客户端**

```bash
ssh username@100.x.x.x
```

> 使用 `netbird status` 查看服务器 A 的 IP 地址。

---

## 7. 暴露 Web 服务（公网访问服务器 A 的 8080 端口）

Netbird 的 expose 命令可以将本地服务通过反向代理暴露到公网。

### 7.1 在 Netbird 控制台中启用 Peer Expose

1. 登录 Netbird 控制台
2. 进入 **Settings** → **Clients**
3. 找到 **Peer Expose** 部分
4. 将 **Enable Peer Expose** 开关打开
5. 选择允许暴露服务的 Peer Group（可选择所有组）
6. 点击 **Save Changes**

### 7.2 在服务器 A 上暴露 Web 服务

假设 Web 服务运行在 8080 端口：

```bash
netbird expose 8080
```

成功执行后，会输出类似以下内容：

```text
Service exposed successfully!
Name: myapp-a1b2c3
URL: https://myapp-a1b2c3.proxy.your-domain.com
Protocol: http
Internal: 8080
Press Ctrl+C to stop exposing.
```

### 7.3 访问 Web 服务

在任意可以访问互联网的设备上，打开浏览器访问输出的 URL（如 `https://myapp-a1b2c3.proxy.your-domain.com`）即可。

### 7.4 其他暴露选项

暴露 TCP 服务（如数据库）：

```bash
netbird expose --protocol tcp 5432
```

指定外部端口：

```bash
netbird expose --protocol tcp --with-external-port 15432 5432
```

停止暴露：按 `Ctrl+C` 终止命令，服务将自动移除。

---

## 8. 故障排查

### 8.1 检查 Netbird 状态

```bash
netbird status
```

### 8.2 检查设备是否在线

登录 Netbird 控制台，进入 **Peers** 页面，确认所有设备显示为 **Connected** 状态。

### 8.3 SSH 连接失败

- 确认服务器 A 已执行 `sudo netbird up --allow-server-ssh`
- 确认在控制台中已将服务器 A 的 SSH Access 设为 ON
- 确认 Netbird 版本为 v0.61.0 或更高

### 8.4 expose 命令失败

- 确认已在控制台启用 Peer Expose
- 确认当前设备所在的 Peer Group 已被授权
- 确认 Web 服务确实在本地指定端口上运行

### 8.5 查看日志

```bash
# 客户端日志
sudo journalctl -u netbird -f

# 控制平面日志（在服务器 S 上）
cd ~/netbird
docker compose logs -f
```

---
