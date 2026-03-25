---
{"dg-publish":true,"permalink":"/TCP 指纹识别：原理、差异与规避技术/","title":"TCP 指纹识别：原理、差异与规避技术","tags":["tcp","指纹","技术","操作系统"],"noteIcon":"","created":"2026-03-25T12:14:00.935+08:00","updated":"2026-03-25T12:17:46.666+08:00"}
---


# TCP 指纹识别：原理、差异与规避技术

## 什么是 TCP 指纹？

TCP 指纹（TCP Fingerprinting）是通过分析 TCP/IP 协议栈实现细节来识别远程主机操作系统或设备类型的技术。由于不同操作系统（甚至同一 OS 的不同版本）在实现 TCP
/IP 协议栈时会做出不同的选择，这些细微差异就像"指纹"一样，可以用来推断对方的身份。

## 核心原理

TCP 指纹识别主要依赖 TCP/IP 协议中那些**未被 RFC 严格规定**或**允许实现自由选择**的参数。常见的指纹特征包括：

### 1. SYN 包中的关键字段

| 特征字段 | 说明 |
|---|---|
| Initial TTL | 初始 TTL 值，不同 OS 默认值不同 |
| Window Size | TCP 初始窗口大小 |
| MSS (Maximum Segment Size) | 最大报文段长度 |
| Window Scaling | 窗口缩放因子 |
| SACK Permitted | 是否支持选择性确认 |
| NOP | No-Operation 填充 |
| TCP Options 顺序 | 选项的排列顺序 |
| DF (Don't Fragment) | IP 头中的分片标志 |
| TOS (Type of Service) | 服务类型字段 |

### 2. 被动 vs 主动指纹识别

- 被动指纹识别（Passive）：仅通过监听网络流量分析 SYN/SYN-ACK 包，不发送任何探测包。代表工具：p0f。
- 主动指纹识别（Active）：主动发送精心构造的探测包，分析响应差异。代表工具：nmap -O。

## 不同操作系统的 TCP 指纹差异

以下是几个主流操作系统的典型 SYN 包指纹特征：

### Linux（内核 4.x/5.x/6.x）

TTL:          64
Window Size:  65535 或 29200
MSS:          1460
Options 顺序: MSS, SACKperm, Timestamp, NOP, Window Scale
Window Scale:  7
DF:           1 (设置)


### Windows 10/11

TTL:          128
Window Size:  65535 或 8192 的倍数
MSS:          1460
Options 顺序: MSS, NOP, Window Scale, NOP, NOP, SACKperm
Window Scale:  8
DF:           1 (设置)
Timestamp:    不发送（默认关闭）


### macOS (Darwin)

TTL:          64
Window Size:  65535
MSS:          1460
Options 顺序: MSS, NOP, Window Scale, NOP, NOP, Timestamp, SACKperm, EOL
Window Scale:  6
DF:           1 (设置)


### FreeBSD

TTL:          64
Window Size:  65535
MSS:          1460
Options 顺序: MSS, NOP, Window Scale, SACKperm, Timestamp
Window Scale:  6
DF:           1 (设置)


### 关键差异总结

| 特征 | Linux | Windows | macOS | FreeBSD |
|---|---|---|---|---|
| TTL | 64 | 128 | 64 | 64 |
| Timestamp | ✅ 默认开启 | ❌ 默认关闭 | ✅ 默认开启 | ✅ 默认开启 |
| Window Scale | 7 | 8 | 6 | 6 |
| Options 顺序 | M,S,T,N,W | M,N,W,N,N,S | M,N,W,N,N,T,S,E | M,N,W,S,T |
| 典型 Window Size | 29200 | 8192×n | 65535 | 65535 |

这些差异使得 p0f 和 nmap 能够以很高的准确率识别远程操作系统。

## 指纹识别工具实战

### p0f（被动识别）

```bash
# 监听 eth0 接口，被动分析经过的流量
p0f -i eth0

# 输出示例：

# .-[ 192.168.1.100/45678 -> 10.0.0.1/443 (syn) ]-
# | client   = 192.168.1.100
# | os       = Linux 3.11 and newer
# | dist     = 0
# | params   = none
# | raw_sig  = 4:64+0:0:1460:mss*20,7:mss,sok,ts,nop,ws:df,id+:0


```
### nmap（主动识别）

```bash
nmap -O 192.168.1.100

# 输出示例：
# OS details: Linux 5.4 - 5.15
# TCP Sequence Prediction: Difficulty=261 (Good luck!)
# IP ID Sequence Generation: All zeros
```


## 如何绕过 TCP 指纹识别

绕过 TCP 指纹识别的核心思路是：**修改本机 TCP/IP 协议栈的行为，使其特征匹配另一个操作系统，或变得无法识别。**

### 方法一：通过内核参数调整（Linux）

在 Linux 上，可以通过 sysctl 修改部分 TCP 参数：

bash
# 修改默认 TTL（模拟 Windows 的 128）
sysctl -w net.ipv4.ip_default_ttl=128

# 关闭 TCP Timestamps（模拟 Windows 行为）
sysctl -w net.ipv4.tcp_timestamps=0

# 修改初始窗口大小相关参数
sysctl -w net.ipv4.tcp_window_scaling=1
sysctl -w net.core.rmem_default=8192

# 关闭 SACK
sysctl -w net.ipv4.tcp_sack=0


这种方法简单但不够彻底，因为 TCP Options 的排列顺序无法通过 sysctl 修改。

### 方法二：使用 iptables/nftables 修改出站包

利用 iptables 的 NFQUEUE 或 mangle 表在包离开前修改字段：

bash
# 修改出站 SYN 包的 TTL 为 128
iptables -t mangle -A POSTROUTING -p tcp --tcp-flags SYN,ACK SYN \
  -j TTL --ttl-set 128

# 使用 nftables 等效写法
nft add rule ip mangle postrouting tcp flags syn / syn,ack \
  ip ttl set 128


### 方法三：使用专用工具 — OSfooler-ng

[OSfooler-ng](https://github.com/segofensiva/OSfooler-ng) 是一个专门用于欺骗 OS 指纹识别的工具，可以同时对抗主动和被动探测：

bash
# 安装
git clone https://github.com/segofensiva/OSfooler-ng.git
cd OSfooler-ng
python setup.py install

# 伪装成 Windows
osfooler-ng -m "Microsoft Windows 10"

# 查看可伪装的 OS 列表
osfooler-ng -g


### 方法四：使用 TCP 代理/中间件重写

通过在网络路径上部署代理，重写 TCP 握手包的特征字段。这是最彻底的方式：

```python
from scapy.all import *

def rewrite_syn(pkt):
    """将出站 SYN 包伪装成 Windows 指纹"""
    if pkt.haslayer(TCP) and pkt[TCP].flags == 'S':
        # 设置 Windows 风格的 TTL
        pkt[IP].ttl = 128
        # 设置 Windows 风格的窗口大小
        pkt[TCP].window = 65535
        # 重写 TCP Options 为 Windows 顺序: MSS, NOP, WScale, NOP, NOP, SACKperm
        pkt[TCP].options = [
            ('MSS', 1460),
            ('NOP', None),
            ('WScale', 8),
            ('NOP', None),
            ('NOP', None),
            ('SAckOK', b''),
        ]
        del pkt[IP].chksum
        del pkt[TCP].chksum
    return pkt
```

# 使用 NetfilterQueue 拦截并修改
```python
from netfilterqueue import NetfilterQueue

def process_packet(packet):
    pkt = IP(packet.get_payload())
    pkt = rewrite_syn(pkt)
    packet.set_payload(bytes(pkt))
    packet.accept()

nfqueue = NetfilterQueue()
nfqueue.bind(0, process_packet)
nfqueue.run()
```


配合 iptables 规则将 SYN 包导入队列：

```bash
iptables -A OUTPUT -p tcp --tcp-flags SYN,ACK SYN -j NFQUEUE --queue-num 0
```


### 方法五：使用 eBPF 在内核层修改

这是最现代、性能最好的方式，直接在内核中修改出站包：

```c
// 简化示例：通过 tc-bpf 修改出站 SYN 包的 TTL
SEC("tc")
int modify_syn_ttl(struct __sk_buff *skb) {
    void *data = (void *)(long)skb->data;
    void *data_end = (void *)(long)skb->data_end;

    struct iphdr *ip = data + sizeof(struct ethhdr);
    if ((void *)(ip + 1) > data_end) return TC_ACT_OK;

    if (ip->protocol != IPPROTO_TCP) return TC_ACT_OK;

    struct tcphdr *tcp = (void *)ip + (ip->ihl * 4);
    if ((void *)(tcp + 1) > data_end) return TC_ACT_OK;

    // 只修改 SYN 包
    if (tcp->syn && !tcp->ack) {
        ip->ttl = 128;  // 伪装成 Windows
        // 重新计算校验和...
    }
    return TC_ACT_OK;
}
```


## 防御视角：如何应对指纹伪装

如果你是防御方，需要注意：

1. 不要仅依赖单一指纹特征 — 组合多层特征（TCP + HTTP User-Agent + TLS JA3/JA4 + 行为分析）进行交叉验证
2. TLS 指纹（JA3/JA4）比 TCP 指纹更难伪造，因为它涉及 TLS 握手中的密码套件、扩展等大量参数
3. HTTP/2 指纹（如 Akamai 的 HTTP/2 fingerprinting）提供了又一个识别维度
4. 行为分析（请求频率、时序模式、鼠标轨迹等）是最难伪造的

## 总结

TCP 指纹识别利用的是协议实现的多样性。绕过它的关键在于理解目标指纹库（如 p0f 的 p0f.fp、nmap 的 nmap-os-db）期望看到什么样的特征组合，然后精确地模拟这些特
征。但在现代安全体系中，TCP 指纹只是多层识别中的一环，真正有效的规避需要同时处理 TCP、TLS、HTTP 等多个层面的指纹。