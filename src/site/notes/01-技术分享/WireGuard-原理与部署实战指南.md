---
{"dg-publish":true,"permalink":"/01-技术分享/WireGuard-原理与部署实战指南/","title":"WireGuard：原理、部署与常用命令实战指南","tags":["wireguard","vpn","networking","linux"],"noteIcon":"","created":"2026-05-22T11:25:37.355+08:00","updated":"2026-05-22T11:26:14.512+08:00"}
---


# WireGuard：原理、部署与常用命令实战指南

## 什么是 WireGuard

WireGuard 是一个极简、高性能的现代 VPN 协议和实现。它于 2020 年合并进 Linux 5.6 内核主线，代码量仅约 4000 行（对比 OpenVPN 约 10 万行），攻击面极小，易于审计。

核心设计哲学：**像 SSH 一样简单**——交换公钥即可建立加密隧道。

---

## 核心原理

### 加密原语

WireGuard 不提供密码套件协商（这是大量 VPN 漏洞的来源），而是固定使用一组现代密码学原语：

| 用途 | 算法 |
|------|------|
| 密钥交换 | Curve25519 (ECDH) |
| 数据加密 | ChaCha20-Poly1305 (AEAD) |
| 哈希 | BLAKE2s |
| 密钥派生 | HKDF |
| 握手构造 | Noise Protocol Framework (IK 模式) |

如果未来某个原语被攻破，WireGuard 会整体升级版本，而非引入复杂的协商机制。

### CryptoKey Routing

WireGuard 的核心概念是 **CryptoKey Routing**——将公钥与允许的 IP 地址绑定：

```
[Peer]
PublicKey = xTIBA5rboUvnH4htodjb6e697QjLERt1NAB4mZqp8Dg=
AllowedIPs = 10.0.0.2/32, 192.168.1.0/24
```

这意味着：
- **发送时**：目标 IP 匹配 `AllowedIPs` → 用该 Peer 的公钥加密 → 发送到该 Peer 的 Endpoint
- **接收时**：解密成功 → 检查源 IP 是否在该 Peer 的 `AllowedIPs` 中 → 通过则接受

`AllowedIPs` 同时充当路由表和 ACL，这是 WireGuard 设计的精髓。

### Noise IK 握手

握手过程仅需 **1-RTT**（一个往返）即可建立会话：

```
Initiator                          Responder
    |                                  |
    |--- Handshake Initiation -------->|  (含发起方临时公钥 + 静态公钥)
    |                                  |
    |<-- Handshake Response -----------|  (含响应方临时公钥)
    |                                  |
    |=== 双方派生出对称会话密钥 =========|
    |                                  |
    |<-- Transport Data -------------->|  (ChaCha20-Poly1305 加密)
```

- 发起方已知响应方公钥（配置中的 `PublicKey`）
- 握手每 2 分钟自动轮换密钥（即使无数据传输）
- 内置重放攻击防护（计数器）

### 无状态设计

WireGuard 是 **"沉默"** 的：
- 没有数据要发送时，不发任何包（对扫描不可见）
- 没有"连接"概念——收到合法加密包就自动更新 Endpoint
- 天然支持漫游（手机从 WiFi 切到 4G，隧道自动恢复）

---

## 部署实战

### 环境准备

```bash
# Ubuntu/Debian
apt install wireguard

# CentOS/RHEL 8+
dnf install wireguard-tools

# macOS
brew install wireguard-tools

# 确认内核模块
modprobe wireguard
```

### 生成密钥对

```bash
# 生成私钥
wg genkey > privatekey

# 从私钥导出公钥
cat privatekey | wg pubkey > publickey

# 一行搞定
wg genkey | tee privatekey | wg pubkey > publickey

# 可选：生成预共享密钥（额外的对称加密层，抗量子计算）
wg genpsk > presharedkey
```

### 场景：两台服务器互联

#### 服务端（Server: 公网 IP 203.0.113.1）

```ini
# /etc/wireguard/wg0.conf
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <server_private_key>

# 开启转发（如果需要做网关）
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# Client
PublicKey = <client_public_key>
AllowedIPs = 10.0.0.2/32
# PresharedKey = <optional_psk>
```

#### 客户端（Client: 任意网络环境）

```ini
# /etc/wireguard/wg0.conf
[Interface]
Address = 10.0.0.2/24
PrivateKey = <client_private_key>
# 可选：指定 DNS
DNS = 1.1.1.1, 8.8.8.8

[Peer]
# Server
PublicKey = <server_public_key>
Endpoint = 203.0.113.1:51820
AllowedIPs = 0.0.0.0/0, ::/0    # 全部流量走隧道
# AllowedIPs = 10.0.0.0/24      # 仅内网流量走隧道
PersistentKeepalive = 25         # NAT 保活（秒）
```

#### 启动

```bash
# 启动接口
wg-quick up wg0

# 设置开机自启
systemctl enable wg-quick@wg0
```

### 场景：全流量代理 vs 分流

| AllowedIPs 配置 | 效果 |
|-----------------|------|
| `0.0.0.0/0, ::/0` | 所有流量走隧道（全局 VPN） |
| `10.0.0.0/24` | 仅访问 VPN 内网时走隧道（Split Tunnel） |
| `10.0.0.0/24, 192.168.0.0/16` | 多个内网网段走隧道 |

### 防火墙放行

```bash
# UFW
ufw allow 51820/udp

# firewalld
firewall-cmd --permanent --add-port=51820/udp
firewall-cmd --reload

# iptables
iptables -A INPUT -p udp --dport 51820 -j ACCEPT
```

### 开启 IP 转发（网关模式必需）

```bash
# 临时生效
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.ipv6.conf.all.forwarding=1

# 永久生效
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.all.forwarding = 1" >> /etc/sysctl.conf
sysctl -p
```

---

## 常用命令速查

### wg-quick（高级管理）

```bash
wg-quick up wg0          # 启动接口（读取 /etc/wireguard/wg0.conf）
wg-quick down wg0        # 关闭接口
wg-quick strip wg0       # 输出去除 wg-quick 特有字段的纯 wg 配置
```

### wg（底层工具）

```bash
wg                        # 显示所有接口状态（简洁）
wg show                   # 同上
wg show wg0               # 显示 wg0 详细状态
wg show wg0 dump          # 机器可读格式（适合脚本解析）
wg showconf wg0           # 输出当前运行配置

# 动态管理 Peer（无需重启）
wg set wg0 peer <pubkey> allowed-ips 10.0.0.3/32 endpoint 1.2.3.4:51820
wg set wg0 peer <pubkey> remove

# 生成密钥
wg genkey                 # 生成私钥
wg pubkey                 # 从 stdin 读取私钥，输出公钥
wg genpsk                 # 生成预共享密钥
```

### systemctl 管理

```bash
systemctl start wg-quick@wg0
systemctl stop wg-quick@wg0
systemctl restart wg-quick@wg0
systemctl enable wg-quick@wg0    # 开机自启
systemctl status wg-quick@wg0
```

### 调试排查

```bash
# 查看接口是否存在
ip link show wg0

# 查看隧道 IP
ip addr show wg0

# 查看路由
ip route | grep wg0

# 抓包调试（WireGuard 使用 UDP）
tcpdump -i eth0 udp port 51820

# 查看内核日志（需要开启动态调试）
echo module wireguard +p > /sys/kernel/debug/dynamic_debug/control
dmesg | grep wireguard

# 测试连通性
ping 10.0.0.1    # ping 对端隧道 IP
```

### wg show 输出解读

```
interface: wg0
  public key: abc123...
  private key: (hidden)
  listening port: 51820

peer: xyz789...
  endpoint: 203.0.113.1:51820
  allowed ips: 10.0.0.0/24
  latest handshake: 32 seconds ago    ← 超过 2 分钟说明连接有问题
  transfer: 1.48 GiB received, 368.15 MiB sent
  persistent keepalive: every 25 seconds
```

关键指标：
- `latest handshake` > 2-3 分钟 → 连接可能断开
- `transfer` 有数据 → 隧道正常工作

---

## 多客户端管理技巧

### 批量生成配置脚本

```bash
#!/bin/bash
SERVER_PUB="<server_public_key>"
SERVER_ENDPOINT="203.0.113.1:51820"
BASE_IP="10.0.0"

for i in $(seq 2 10); do
  mkdir -p clients/client${i}
  wg genkey | tee clients/client${i}/privatekey | wg pubkey > clients/client${i}/publickey

  cat > clients/client${i}/wg0.conf << EOF
[Interface]
Address = ${BASE_IP}.${i}/24
PrivateKey = $(cat clients/client${i}/privatekey)
DNS = 1.1.1.1

[Peer]
PublicKey = ${SERVER_PUB}
Endpoint = ${SERVER_ENDPOINT}
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
EOF

  echo "Client ${i}: PublicKey = $(cat clients/client${i}/publickey), AllowedIPs = ${BASE_IP}.${i}/32"
done
```

### 动态添加 Peer（不重启服务）

```bash
# 添加新 Peer
wg set wg0 peer <new_client_pubkey> allowed-ips 10.0.0.11/32

# 保存到配置文件（持久化）
wg-quick save wg0
```

---

## 与其他 VPN 对比

| 特性 | WireGuard | OpenVPN | IPSec/IKEv2 |
|------|-----------|---------|-------------|
| 代码量 | ~4,000 行 | ~100,000 行 | 极其复杂 |
| 加密协商 | 无（固定原语） | 可配置 | 可配置 |
| 性能 | 内核态，接近线速 | 用户态，较慢 | 内核态，快 |
| 握手延迟 | 1-RTT | 多次往返 | 多次往返 |
| 漫游支持 | 原生 | 需重连 | 部分支持 |
| 配置复杂度 | 极低 | 中等 | 高 |
| 隐蔽性 | 低（固定 UDP 端口） | 可伪装 TCP/443 | 中等 |

---

## 常见问题

**Q: 握手成功但 ping 不通？**
- 检查服务端 `ip_forward` 是否开启
- 检查 `AllowedIPs` 是否包含目标网段
- 检查防火墙是否放行 `wg0` 接口转发

**Q: 如何让 WireGuard 更隐蔽？**
- 使用 [wstunnel](https://github.com/erebe/wstunnel) 将 UDP 封装为 WebSocket
- 或使用 [udp2raw](https://github.com/wangyu-/udp2raw-tunnel) 伪装为 TCP/ICMP

**Q: 手机客户端？**
- iOS/Android 官方 App 支持扫描二维码导入配置
- 生成二维码：`qrencode -t ansiutf8 < client.conf`

**Q: PersistentKeepalive 什么时候需要？**
- 客户端在 NAT 后面时必须设置（推荐 25 秒）
- 双方都有公网 IP 时不需要

---

## 总结

WireGuard 的优势在于：
1. **极简** — 配置就是交换公钥 + 指定 AllowedIPs
2. **高性能** — 内核态实现，吞吐量接近线速
3. **安全** — 代码量小易审计，无密码协商，固定现代密码学原语
4. **漫游友好** — 无连接状态，IP 变化自动适应

适合场景：站点互联、远程办公、科学上网中继、IoT 设备组网。不适合需要 TCP 伪装或复杂认证体系的场景。
