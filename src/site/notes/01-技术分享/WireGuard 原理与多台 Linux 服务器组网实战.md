---
{"dg-publish":true,"permalink":"/01-技术分享/WireGuard 原理与多台 Linux 服务器组网实战/","title":"WireGuard 原理与多台 Linux 服务器组网实战","tags":["网络","技术","vpn","wireguard"],"noteIcon":"","created":"2026-03-25T20:20:15.257+08:00","updated":"2026-03-25T20:25:12.221+08:00"}
---


# WireGuard 原理与多台 Linux 服务器组网实战

## 一、WireGuard 原理

### 1.1 什么是 WireGuard

WireGuard 是一个现代、高性能的 VPN 协议和实现，直接集成在 Linux 内核中（5.6+）。相比 OpenVPN 和 IPSec，它的代码量极小（
约 4000 行），攻击面更小，性能更高。

### 1.2 核心设计原理

Cryptokey Routing（密钥路由）

WireGuard 的核心概念是将公钥与允许的 IP 地址绑定。每个 peer 有：
- 一对密钥（公钥 + 私钥）
- 一组 AllowedIPs（允许通过隧道的 IP 范围）

当发送数据包时，WireGuard 根据目标 IP 查找对应的 peer 公钥进行加密；收到数据包时，根据来源公钥验证源 IP 是否在
AllowedIPs 中。

协议栈

应用层数据
    ↓
WireGuard 接口 (wg0) — 加密封装
    ↓
UDP 包 (默认端口 51820)
    ↓
物理网卡发出


加密方案（不可协商，固定选择）
- 密钥交换：Curve25519 (ECDH)
- 数据加密：ChaCha20-Poly1305
- 哈希：BLAKE2s
- 密钥派生：HKDF

无状态设计
- 没有"连接"概念，不需要维护会话状态
- 静默节点不产生任何流量（Stealth）
- 内置漫游支持：endpoint 自动更新为最新收到包的源地址




## 二、组网实战：3 台 Linux 服务器星型 + 全互联

### 2.1 场景说明

三台服务器分布在不同机房/云厂商，通过 WireGuard 组建私有网络：

| 节点 | 角色 | 公网 IP | WireGuard IP | 备注 |
|------|------|---------|-------------|------|
| Node-A | Hub | 1.1.1.1 | 10.0.0.1/24 | 有固定公网 IP |
| Node-B | Peer | 2.2.2.2 | 10.0.0.2/24 | 有固定公网 IP |
| Node-C | Peer | 动态/NAT后 | 10.0.0.3/24 | 无固定公网 IP |

组网拓扑：全互联（Full Mesh），Node-C 因为没有公网 IP，通过 Node-A 或 Node-B 作为中继。

### 2.2 安装 WireGuard

bash
# Ubuntu/Debian
apt update && apt install -y wireguard

# CentOS/RHEL 8+
dnf install -y wireguard-tools

# 验证内核模块
modprobe wireguard


### 2.3 生成密钥对（每台机器都执行）

```bash
wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/pubkey
chmod 600 /etc/wireguard/privatekey
```

假设生成的密钥如下（示例值）：

| 节点 | 私钥 | 公钥 |
|------|------|------|
| Node-A | aPrivKeyA... | aPubKeyA... |
| Node-B | bPrivKeyB... | bPubKeyB... |
| Node-C | cPrivKeyC... | cPubKeyC... |

### 2.4 配置文件

Node-A (/etc/wireguard/wg0.conf)

```ini
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = aPrivKeyA...
# 可选：开启转发，让 Node-C 通过 A 中继到 B
PostUp = sysctl -w net.ipv4.ip_forward=1
PostDown = sysctl -w net.ipv4.ip_forward=0

[Peer]
# Node-B
PublicKey = bPubKeyB...
Endpoint = 2.2.2.2:51820
AllowedIPs = 10.0.0.2/32
PersistentKeepalive = 25

[Peer]
# Node-C（无固定公网 IP，不设 Endpoint，等它主动连过来）
PublicKey = cPubKeyC...
AllowedIPs = 10.0.0.3/32
PersistentKeepalive = 25
```

Node-B (/etc/wireguard/wg0.conf)

```ini
[Interface]
Address = 10.0.0.2/24
ListenPort = 51820
PrivateKey = bPrivKeyB...
PostUp = sysctl -w net.ipv4.ip_forward=1
PostDown = sysctl -w net.ipv4.ip_forward=0

[Peer]
# Node-A
PublicKey = aPubKeyA...
Endpoint = 1.1.1.1:51820
AllowedIPs = 10.0.0.1/32
PersistentKeepalive = 25

[Peer]
# Node-C
PublicKey = cPubKeyC...
AllowedIPs = 10.0.0.3/32
PersistentKeepalive = 25
```


Node-C (/etc/wireguard/wg0.conf)

```ini
[Interface]
Address = 10.0.0.3/24
ListenPort = 51820
PrivateKey = cPrivKeyC...

[Peer]
# Node-A（必须设 Endpoint，因为 C 在 NAT 后面需要主动发起）
PublicKey = aPubKeyA...
Endpoint = 1.1.1.1:51820
AllowedIPs = 10.0.0.1/32, 10.0.0.2/32
PersistentKeepalive = 25

[Peer]
# Node-B
PublicKey = bPubKeyB...
Endpoint = 2.2.2.2:51820
AllowedIPs = 10.0.0.2/32
PersistentKeepalive = 25
```


│ **关键点：** Node-C 的 Peer Node-A 中 AllowedIPs = 10.0.0.1/32, 10.0.0.2/32 表示如果 Node-C 无法直连 Node-B，可以通过
Node-A 中继到达 10.0.0.2。如果 Node-C 能直连 Node-B，则两个 Peer 各自只写 /32 即可。

### 2.5 启动与管理

```bash
# 启动
wg-quick up wg0

# 开机自启
systemctl enable wg-quick@wg0

# 查看状态
wg show

# 停止
wg-quick down wg0

```
### 2.6 防火墙放行

```bash
# 放行 WireGuard UDP 端口
ufw allow 51820/udp
# 或 firewalld
firewall-cmd --permanent --add-port=51820/udp && firewall-cmd --reload
```

### 2.7 验证连通性

```bash
# 在 Node-A 上
ping 10.0.0.2  # → Node-B
ping 10.0.0.3  # → Node-C

# 查看握手状态
wg show wg0
```


正常输出应该能看到每个 peer 的 latest handshake 时间和 transfer 数据量：

peer: bPubKeyB...
  endpoint: 2.2.2.2:51820
  allowed ips: 10.0.0.2/32
  latest handshake: 12 seconds ago
  transfer: 1.24 MiB received, 856.32 KiB sent





## 三、进阶配置

### 3.1 全流量转发（让 Peer 通过某节点上网）

如果想让 Node-C 的所有流量都通过 Node-A 出去：

Node-C 的配置中修改 Node-A 的 AllowedIPs：
```ini
AllowedIPs = 0.0.0.0/0

```
Node-A 上添加 NAT：
```ini
PostUp = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; sysctl -w net.ipv4.ip_forward=1
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```


### 3.2 DNS 配置

```ini
[Interface]
DNS = 10.0.0.1, 1.1.1.1
```

### 3.3 动态添加 Peer（不重启）

```bash
wg set wg0 peer <新节点公钥> allowed-ips 10.0.0.4/32 endpoint 3.3.3.3:51820
```




## 四、常见问题排查

| 问题 | 排查方向 |
|------|---------|
| 无法握手 | 检查防火墙是否放行 UDP 51820；NAT 后的节点是否设了 Endpoint 和 PersistentKeepalive |
| 能 ping 不能访问服务 | 检查 AllowedIPs 是否正确，服务是否监听了 WireGuard 接口的 IP |
| 单向通信 | 检查双方是否都正确配置了对方的公钥和 AllowedIPs |
| 性能差 | 检查 MTU（默认 1420），可尝试调低：MTU = 1380 |




## 五、总结

WireGuard 的优势在于：
- **简洁**：每个节点一个配置文件，密钥对 + AllowedIPs 就是全部
- **高性能**：内核态实现，吞吐量接近线速
- **安全**：现代密码学，无历史包袱，无需协商加密套件
- **易扩展**：新增节点只需生成密钥、交换公钥、添加 Peer 配置

对于多台服务器组网，Full Mesh 拓扑适合节点数较少（<10）的场景。节点更多时可以考虑 Hub-Spoke 拓扑或使用 [Netmaker](https://github.com/gravitl/netmaker)、[Tailscale](https://tailscale.com) 等基于 WireGuard 的管理工具来自动化配置分发。
