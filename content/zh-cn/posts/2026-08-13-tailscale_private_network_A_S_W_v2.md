+++
title= "Tailscale 私有网络方案：A 内网服务器 + S 公网 Gateway + W 异地工作站"
description= "Tailscale  private network"
date = "2026-08-12"
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

# 目标：A 不开放公网入站端口；W 可以通过 Tailscale 访问 A；同时允许非 Tailscale 用户通过公网访问 A 的 8080 Web 服务。

## 1. 网络环境

有三台服务器/工作站：

| 设备 | 系统 | 网络 | 作用 |
|---|---|---|---|
| A | Ubuntu | 内网，防火墙禁止公网入站，但可访问 Internet | 内部业务服务器 |
| S | Linux | 固定公网 IP，可访问 Internet | 公网 Web Gateway / 反向代理 |
| W | macOS | 异地网络，可访问 Internet | 运维工作站 |

需求：

1. W 可以 SSH 到 A。
2. W 可以访问 A 的 `8080` Web 服务。
3. A 不开放公网 `22`。
4. A 不开放公网 `8080`。
5. A 不需要端口映射。
6. 其他非 Tailscale 用户也可以访问 A 的 Web `8080` 服务。
7. W 启用 Tailscale 后，仍然正常访问普通 Internet。
8. S 有固定公网 IP，可以作为公网 Web 入口。

---

# 2. 最终推荐架构

这里建议把 S 定位为：

> **公网 Web Gateway / Reverse Proxy**

而不是传统的 VPN Server。

```text
                              Internet
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
              macOS W                     普通 Internet 用户
            Tailscale 成员                    非成员
                    │                           │
                    │ Tailscale                 │ HTTPS :443
                    │                           │
                    ▼                           ▼
             ┌─────────────┐              ┌─────────────┐
             │ Ubuntu A    │◄─────────────│ Linux S     │
             │ 内网服务器  │  Tailscale   │ 公网Gateway │
             │             │              │ 固定公网 IP │
             └─────────────┘              └─────────────┘
                    │                           │
                    └────── Web :8080 ─────────┘
```

实际访问关系：

```text
W
 │
 │ Tailscale
 ├──────────────────────► A:22
 │
 └──────────────────────► A:8080


普通 Internet 用户
 │
 │ HTTPS
 ▼
S:443
 │
 │ Tailscale
 ▼
A:8080
```

因此：

```text
A 公网 TCP/22       = 不开放
A 公网 TCP/8080     = 不开放
S 公网 TCP/443      = 开放
S → A               = Tailscale 私有网络
W → A               = Tailscale 私有网络
```

这是本场景最推荐的生产架构。

---

# 3. 为什么 S 不应该作为传统 VPN Server

传统方案：

```text
W
 │
 │ WireGuard
 ▼
S
 │
 │ NAT / Routing
 ▼
A
```

需要维护：

- WireGuard
- UDP 监听端口
- IP forwarding
- iptables/nftables
- NAT
- 路由
- 客户端密钥
- 返回路径

而使用 Tailscale：

```text
W ──────────┐
            │
S ──────────┼── Tailnet
            │
A ──────────┘
```

A、S、W 都成为 Tailnet 节点。

S 不需要作为 VPN Server。

S 只负责：

```text
公网 HTTPS
    │
    ▼
Reverse Proxy
    │
    ▼
Tailscale
    │
    ▼
A:8080
```

---

# 4. Tailscale 网络

建议三台机器都加入同一个 Tailnet：

```text
A = 100.x.x.10
S = 100.x.x.20
W = 100.x.x.30
```

这些只是示例地址，实际地址由 Tailscale 分配。

拓扑：

```text
                 Tailnet
                    │
          ┌─────────┼─────────┐
          │         │         │
          ▼         ▼         ▼
          A         S         W
       100.x.10  100.x.20  100.x.30
```

Tailscale 节点之间优先尝试建立直接连接；如果网络条件不允许，也可以使用 Tailscale 的中继机制。

---

# 5. A 安装 Tailscale

Ubuntu A：

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

启动：

```bash
sudo systemctl enable --now tailscaled
```

登录：

```bash
sudo tailscale up
```

查看状态：

```bash
tailscale status
```

查看 Tailscale IP：

```bash
tailscale ip
```

例如：

```text
100.64.10.10
```

---

# 6. S 安装 Tailscale

S：

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

启动：

```bash
sudo systemctl enable --now tailscaled
```

登录：

```bash
sudo tailscale up
```

检查：

```bash
tailscale status
```

例如：

```text
A    100.64.10.10
S    100.64.10.20
```

---

# 7. W 安装 Tailscale

macOS W 安装 Tailscale 客户端。

登录同一个 Tailnet。

确认：

```bash
tailscale status
```

应该可以看到 A：

```text
100.64.10.10   server-a
```

---

# 8. W SSH 到 A

最简单：

```bash
ssh ubuntu@100.64.10.10
```

如果启用了 MagicDNS：

```bash
ssh ubuntu@server-a
```

流量：

```text
W
 │
 │ Tailscale
 ▼
A:22
```

A 不需要开放公网 TCP/22。

Tailscale 官方文档也支持直接通过 Tailscale IP 或 MagicDNS 主机名使用普通 SSH。citeturn0search1

---

# 9. SSH 推荐：先使用普通 OpenSSH

建议第一阶段：

```text
普通 OpenSSH
        +
Tailscale 网络
```

例如：

```bash
ssh ubuntu@server-a
```

这样 A 上原来的：

```text
/etc/ssh/sshd_config
~/.ssh/authorized_keys
```

全部可以继续使用。

这是改造风险最低的方式。

---

# 10. 可选：Tailscale SSH

如果希望由 Tailscale 管理 SSH 身份认证和授权，可以使用：

```bash
sudo tailscale set --ssh
```

然后使用：

```bash
ssh ubuntu@server-a
```

Tailscale SSH 可以利用 Tailnet 的访问控制策略管理 SSH 权限。官方文档说明，启用后 Tailscale 会接管来自 Tailnet 的目标端口 22，而不会修改原来的 `sshd_config` 和 `authorized_keys`。citeturn0search0

对于已有生产环境，建议：

```text
第一阶段：
普通 SSH + Tailscale

第二阶段：
根据权限管理需求考虑 Tailscale SSH
```

---

# 11. A 的 8080 Web 服务

假设 A：

```text
Web Application
       │
       ▼
127.0.0.1:8080
```

例如：

```bash
curl http://127.0.0.1:8080
```

正常。

现在有两个需求：

### W

```text
W → A:8080
```

### 普通用户

```text
Internet → S:443 → A:8080
```

因此推荐不要让 A 直接暴露公网 8080。

---

# 12. W 访问 A:8080

如果 Web 服务可以监听：

```text
0.0.0.0:8080
```

那么 W 可以：

```text
http://server-a:8080
```

或者：

```text
http://100.64.10.10:8080
```

但是生产环境更推荐让 A 的 Web 服务继续只监听：

```text
127.0.0.1:8080
```

然后使用反向代理。

---

# 13. A → 本机反向代理

A 可以运行 Caddy/Nginx：

```text
Tailscale
    │
    ▼
A reverse proxy
    │
    ▼
127.0.0.1:8080
```

例如 A 上 Caddy：

```caddyfile
:8081 {
    reverse_proxy 127.0.0.1:8080
}
```

但是，如果 W 需要直接访问 A 的 Web，可以让 Caddy 监听 Tailscale IP 或使用 Tailscale Serve。

---

# 14. 更推荐：Tailscale Serve

如果只希望：

```text
Tailnet 用户
      │
      ▼
A Web
```

可以使用：

```bash
tailscale serve 8080
```

这样：

```text
W
 │
 │ Tailscale
 ▼
Tailscale Serve
 │
 ▼
127.0.0.1:8080
```

优点：

- Web 应用可以继续监听 localhost
- 不需要开放公网 8080
- 不需要额外公网端口
- Tailnet 内部访问

但本方案还有一个额外需求：

> **非 Tailscale 用户也必须能访问 Web。**

所以公网用户不应该直接使用 Serve，而应该经过 S。

---

# 15. S 作为公网 Web Gateway

S：

```text
公网 IP
    │
    │ TCP/443
    ▼
Caddy / Nginx
    │
    │ Tailscale
    ▼
A:8080
```

例如：

```text
https://web.example.com
```

DNS：

```text
web.example.com
       │
       ▼
S 公网 IP
```

用户访问：

```text
https://web.example.com
```

进入：

```text
S:443
```

然后：

```text
S
 │
 │ Tailscale
 ▼
A:8080
```

---

# 16. S 使用 Caddy

如果 S 已经使用 Caddy，配置非常简单。

例如：

```caddyfile
web.example.com {
    reverse_proxy 100.64.10.10:8080
}
```

其中：

```text
100.64.10.10
```

是 A 的 Tailscale IP。

完整流量：

```text
Internet
    │
    │ HTTPS :443
    ▼
S
    │
    │ Caddy reverse_proxy
    │
    │ Tailscale
    ▼
A:8080
```

---

# 17. 如果 A:8080 只监听 127.0.0.1

此时 S 不能直接：

```text
S → A:8080
```

因为 A 的：

```text
127.0.0.1:8080
```

只接受 A 本机连接。

这时有两种选择。

## 方案 A：A 运行反向代理

```text
S
 │
 │ Tailscale
 ▼
A:Caddy
 │
 ▼
127.0.0.1:8080
```

例如 A：

```caddyfile
:8081 {
    bind 100.64.10.10
    reverse_proxy 127.0.0.1:8080
}
```

S：

```caddyfile
web.example.com {
    reverse_proxy 100.64.10.10:8081
}
```

这是很清晰的生产架构。

---

# 18. 推荐的 Web 双入口架构

最终：

```text
                         ┌──────────────────────┐
                         │       Ubuntu A       │
                         │                      │
                         │  Web Application     │
                         │       :8080          │
                         │          ▲           │
                         │          │           │
                         │      Reverse Proxy   │
                         │       :8081          │
                         └──────────▲───────────┘
                                    │
                              Tailscale
                                    │
                   ┌────────────────┴────────────────┐
                   │                                 │
                   │                                 │
                   │                                 ▼
              macOS W                         Linux S
            Tailscale                         公网 Gateway
                   │                                 │
                   │                                 │
                   ▼                                 │
             A Web :8081                             │
                                                     │
                                              HTTPS :443
                                                     ▲
                                                     │
                                              Internet 用户
```

更简单地理解：

```text
W ──────────► Tailscale ──────────► A

Internet ───► S:443
                 │
                 ▼
             Tailscale
                 │
                 ▼
                 A
```

---

# 19. A 的防火墙

A：

```text
公网：
22       CLOSED
8080     CLOSED
8081     CLOSED
443      CLOSED
```

但 Tailnet：

```text
W ─────────────► A
S ─────────────► A
```

可以访问指定服务。

重点：

> S 到 A 的连接也是通过 Tailscale，而不是通过 A 的公网 IP。

所以 A 仍然没有公网入站端口。

---

# 20. 非 Tailscale 用户访问 Web

普通用户：

```text
浏览器
   │
   │ HTTPS
   ▼
https://web.example.com
   │
   ▼
S 公网 IP
   │
   ▼
Caddy :443
   │
   │ Tailscale
   ▼
A
   │
   ▼
Web :8080
```

用户完全不需要安装 Tailscale。

这是本方案和“纯 Tailscale Serve”最大的区别。

---

# 21. W 访问普通 Internet 是否受到影响？

**默认不会。**

W 启动 Tailscale 后：

```text
W
├──────────────► Internet
│
└── Tailscale ─► A
```

访问：

```text
Google
GitHub
普通网站
公司 SaaS
```

仍然走 W 当前网络的普通 Internet 出口。

只有访问：

```text
A
S
其他 Tailnet 设备
```

等 Tailnet 目标时，才使用 Tailscale 路径。

---

# 22. 不要把 S 设置成 Exit Node

如果 S 被设置为 Exit Node：

```text
W
 │
 ├──► A
 │
 └──► S ──► Internet
```

W 的普通 Internet 流量也会经过 S。

例如：

```text
W → Google
      │
      ▼
      S
      │
      ▼
    Google
```

这不是本方案默认需要的功能。

推荐：

```text
W：
Tailscale = ON
Exit Node = OFF
```

这样：

```text
W → Internet
```

仍然走本地 Internet。

而：

```text
W → A
```

走 Tailscale。

---

# 23. Tailnet ACL / Grants

建议限制：

```text
W → A
S → A
```

例如：

```text
W
 │
 ├── TCP 22 ─────► A
 │
 └── TCP 8081 ───► A

S
 │
 └── TCP 8081 ───► A
```

不建议：

```text
W ─────────► A:*
S ─────────► A:*
```

应该遵循最小权限原则。

Tailscale 的访问控制策略可以精确限制用户/设备对目标设备和服务的访问。citeturn0search6

---

# 24. 推荐 ACL 思路

逻辑上：

```text
              A
              │
       ┌──────┴──────┐
       │             │
      SSH           Web
       │             │
       ▼             ▼
       W             W
                     S
```

即：

```text
W → A:22
W → A:8081

S → A:8081
```

而：

```text
S → A:22
```

如果没有必要就不要允许。

---

# 25. Server S 的公网防火墙

S 是真正需要公网开放 Web 端口的机器。

建议：

```text
S:
TCP/443   ALLOW
TCP/80    ALLOW（如果需要 ACME HTTP challenge）
TCP/22    根据实际运维需求限制
```

而：

```text
A:
TCP/22    公网禁止
TCP/8080  公网禁止
TCP/8081  公网禁止
TCP/443   公网禁止
```

---

# 26. 如果使用 Caddy

推荐：

```text
S
│
├── Caddy
│     │
│     └── :443
│
└── Tailscale
      │
      ▼
      A
```

Caddy：

```caddyfile
web.example.com {
    reverse_proxy 100.64.10.10:8081
}
```

A：

```text
Web
 │
 ▼
127.0.0.1:8080

Caddy
 │
 ▼
100.64.10.10:8081
```

这样 Web 应用本身不需要暴露到外部网络。

---

# 27. HTTPS

公网访问：

```text
https://web.example.com
```

HTTPS 终止在 S：

```text
Internet
   │
 HTTPS
   ▼
S / Caddy
   │
   │ HTTP over Tailscale
   ▼
A
```

Tailscale 本身已经提供加密的节点间通信。

如果对端应用也要求 HTTPS，可以继续：

```text
S
 │
 │ HTTPS
 ▼
A
```

即：

```caddyfile
web.example.com {
    reverse_proxy https://100.64.10.10:8443
}
```

具体是否需要端到端 HTTPS，取决于应用和安全要求。

---

# 28. Subnet Router 扩展

如果以后需要：

```text
W
 │
 ▼
A
 │
 ├── 192.168.10.30 NAS
 ├── 192.168.10.40 Switch
 └── 192.168.10.50 Printer
```

可以让 A 成为 Subnet Router。

开启：

```bash
echo 'net.ipv4.ip_forward = 1' | \
sudo tee /etc/sysctl.d/99-tailscale.conf

sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

然后：

```bash
sudo tailscale set --advertise-routes=192.168.10.0/24
```

在 Tailscale Admin Console 批准该路由。

---

# 29. S 是否可以做 Subnet Router？

可以。

但在本场景：

```text
A → 内网
```

更自然的是让 A 或 A 所在内网的一台 Linux 机器作为 Subnet Router。

S 位于公网，通常不适合承担公司内网的 Subnet Router，除非网络拓扑确实需要这样设计。

---

# 30. 故障排查

## A 是否在线

```bash
tailscale status
```

## W 是否能到 A

W：

```bash
tailscale ping server-a
```

## 查看 A 的地址

A：

```bash
tailscale ip
```

## SSH

W：

```bash
ssh -v ubuntu@server-a
```

## A 本地 Web

A：

```bash
curl http://127.0.0.1:8080
```

## A 的 Tailscale Web

W：

```bash
curl http://100.64.10.10:8081
```

## S → A

S：

```bash
curl http://100.64.10.10:8081
```

## 公网 Web

任意普通 Internet 机器：

```bash
curl -vk https://web.example.com
```

建议按照：

```text
A 本地
 ↓
W → A
 ↓
S → A
 ↓
Internet → S → A
```

逐层测试。

---

# 31. 安全边界

最终安全边界应该是：

```text
                Internet
                   │
                   │
              ┌────▼────┐
              │    S    │
              │  :443   │
              └────┬────┘
                   │
                Tailscale
                   │
              ┌────▼────┐
              │    A    │
              │         │
              │  :8080  │
              │  :8081  │
              │  :22    │
              └─────────┘
```

公网只看到：

```text
S:443
```

公网看不到：

```text
A:22
A:8080
A:8081
```

---

# 32. 推荐最终配置

## A

```text
Tailscale：启用

公网：
22     CLOSED
8080   CLOSED
8081   CLOSED

业务：
127.0.0.1:8080

内部 Gateway：
Tailscale IP:8081
```

## S

```text
Tailscale：启用

公网：
443 OPEN
80  可选

Caddy：
web.example.com
        ↓
A-Tailscale-IP:8081
```

## W

```text
Tailscale：启用
Exit Node：不使用

SSH：
ssh ubuntu@server-a
```

---

# 33. 最终访问矩阵

| 来源 | 目标 | 是否需要 Tailscale | 推荐路径 |
|---|---|---:|---|
| W | A:22 | 是 | W → Tailscale → A |
| W | A Web | 是 | W → Tailscale → A |
| S | A Web | 是 | S → Tailscale → A |
| 普通 Internet 用户 | A Web | 否 | Internet → S:443 → Tailscale → A |
| 普通 Internet 用户 | A:22 | 否 | 不允许 |
| 普通 Internet 用户 | A:8080 | 否 | 不允许 |
| W | Google/GitHub 等 Internet | 否 | W 本地 Internet |
| W | S | 是 | W → Tailscale → S |

---

# 34. 最终架构总结

推荐最终架构：

```text
                         Internet
                            │
             ┌──────────────┴──────────────┐
             │                             │
             │                             │
             ▼                             ▼
       macOS W                        普通用户
       Tailscale                      非 Tailscale
             │                             │
             │                             │ HTTPS
             │                             ▼
             │                       ┌───────────┐
             │                       │ Server S  │
             │                       │ Caddy     │
             │                       │ :443      │
             │                       └─────┬─────┘
             │                             │
             │                         Tailscale
             │                             │
             │                             ▼
             │                       ┌───────────┐
             └────── Tailscale ─────►│ Server A  │
                                     │ 内网      │
                                     │           │
                                     │ Web :8080 │
                                     │           │
                                     │ SSH :22   │
                                     └───────────┘
```

核心原则：

1. **A 永远不需要开放公网入站端口。**
2. **W 使用 Tailscale 访问 A。**
3. **S 使用 Tailscale 访问 A。**
4. **非 Tailscale 用户通过 S 的公网 HTTPS 访问 Web。**
5. **S 是公网 Web Gateway，不是传统 VPN Server。**
6. **W 不设置 Exit Node，因此正常 Internet 访问不受影响。**
7. **通过 Tailscale ACL/Grants 限制 W、S 对 A 的访问范围。**
8. **A 的 Web 应用最好只监听 localhost，再由 A 上的 Gateway 暴露给 Tailscale。**

---

# 35. 官方参考

- Tailscale 文档：https://tailscale.com/docs
- Tailscale SSH：https://tailscale.com/docs/features/tailscale-ssh
- SSH over Tailscale：https://tailscale.com/docs/reference/ssh-over-tailscale
- Tailscale CLI：https://tailscale.com/docs/reference/tailscale-cli
- Tailscale 安全与访问控制：https://tailscale.com/kb/1429/secure


