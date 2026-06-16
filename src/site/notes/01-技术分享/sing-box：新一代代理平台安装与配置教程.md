---
{"dg-publish":true,"permalink":"/01-技术分享/sing-box：新一代代理平台安装与配置教程/","tags":["sing-box","proxy","shadowsocks","Linux","入门教程","网络工具"],"noteIcon":"","created":"2026-04-23T19:14:08.868+08:00","updated":"2026-04-23T19:37:32.705+08:00"}
---


# sing-box：新一代代理平台安装与配置教程

> [sing-box](https://github.com/SagerNet/sing-box) 是一个通用代理平台，支持 Shadowsocks、VMess、Trojan、Hysteria2 等多种协议，跨平台运行。本文面向新手，从安装到全局代理一步步走通。

## 一、sing-box 是什么？

sing-box 是 SagerNet 团队开发的下一代代理工具，定位类似 Clash / V2Ray，但设计更现代：

- 支持协议多：Shadowsocks、VMess、VLESS、Trojan、Hysteria2、TUIC、WireGuard 等
- 跨平台：Linux、macOS、Windows、Android、iOS 全覆盖
- 配置统一：一个 JSON 配置文件搞定所有
- 性能好：Go 语言编写，资源占用低

## 二、安装

### Linux（Debian/Ubuntu）

```bash
# 官方一键安装脚本
curl -fsSL https://sing-box.app/install.sh | sh
```

或者手动安装：

```bash
# 下载最新版（以 amd64 为例，arm64 换对应包）
curl -Lo sing-box.tar.gz https://github.com/SagerNet/sing-box/releases/latest/download/sing-box-1.11.0-linux-amd64.tar.gz
tar xzf sing-box.tar.gz
sudo cp sing-box-*/sing-box /usr/local/bin/
sudo chmod +x /usr/local/bin/sing-box

# 验证
sing-box version
```

### macOS

```bash
brew install sing-box
```

### Windows

从 [Releases](https://github.com/SagerNet/sing-box/releases) 下载 `sing-box-*-windows-amd64.zip`，解压后将 `sing-box.exe` 放到 PATH 中。

## 三、核心概念

sing-box 的配置由四个核心部分组成，理解了就不会迷路：

```
inbound（入站）→ route（路由）→ outbound（出站）
                    ↕
               dns（DNS 解析）
```

- **inbound**：流量怎么进来（tun 全局 / socks5 本地代理）
- **outbound**：流量怎么出去（走代理 / 直连）
- **route**：哪些流量走代理，哪些直连
- **dns**：域名怎么解析

## 四、配置 Shadowsocks 全局代理

创建配置文件 `/etc/sing-box/config.json`：

```bash
sudo mkdir -p /etc/sing-box
sudo nano /etc/sing-box/config.json
```

粘贴以下内容，修改标注了 `⬅️` 的地方：

```json
{
  "log": {
    "level": "info"
  },
  "dns": {
    "servers": [
      {
        "tag": "remote-dns",
        "address": "tls://8.8.8.8"
      },
      {
        "tag": "local-dns",
        "address": "223.5.5.5",
        "detour": "direct"
      }
    ],
    "rules": [
      {
        "outbound": "any",
        "server": "local-dns"
      }
    ]
  },
  "inbounds": [
    {
      "type": "tun",
      "tag": "tun-in",
      "interface_name": "tun0",
      "inet4_address": "172.19.0.1/30",
      "auto_route": true,
      "strict_route": true,
      "stack": "system",
      "sniff": true
    }
  ],
  "outbounds": [
    {
      "type": "shadowsocks",
      "tag": "proxy",
      "server": "你的服务器IP",
      "server_port": 8388,
      "method": "2022-blake3-aes-128-gcm",
      "password": "你的密码"
    },
    {
      "type": "direct",
      "tag": "direct"
    },
    {
      "type": "dns",
      "tag": "dns-out"
    }
  ],
  "route": {
    "rules": [
      {
        "protocol": "dns",
        "outbound": "dns-out"
      },
      {
        "geoip": ["private"],
        "outbound": "direct"
      }
    ],
    "default_mark": 255,
    "auto_detect_interface": true
  }
}
```

需要修改的字段：

| 字段 | 说明 |
|------|------|
| `server` | 你的 SS 服务器 IP 或域名 |
| `server_port` | SS 服务器端口 |
| `method` | 加密方式，和服务端保持一致 |
| `password` | 密码，和服务端保持一致 |

常见加密方式：`2022-blake3-aes-128-gcm`、`aes-256-gcm`、`aes-128-gcm`、`chacha20-ietf-poly1305`

## 五、运行与验证

### 手动运行（测试用）

```bash
# 需要 root 权限（tun 设备需要）
sudo sing-box run -c /etc/sing-box/config.json
```

看到日志输出没有报错就说明启动成功了。

### 验证代理是否生效

新开一个终端：

```bash
curl -s https://ipinfo.io/ip
```

返回的是你 SS 服务器的 IP 就说明全局代理生效了。

### 设为系统服务（推荐）

```bash
# 官方安装脚本会自动创建 systemd 服务
sudo systemctl enable sing-box
sudo systemctl start sing-box

# 查看状态
sudo systemctl status sing-box

# 实时查看日志
sudo journalctl -u sing-box -f
```

如果是手动安装的，创建 service 文件：

```bash
sudo tee /etc/systemd/system/sing-box.service << 'EOF'
[Unit]
Description=sing-box service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/sing-box run -c /etc/sing-box/config.json
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now sing-box
```

## 六、不想全局代理？用本地代理模式

如果只想让特定应用走代理，把 `tun` inbound 换成 `mixed`：

```json
{
  "inbounds": [
    {
      "type": "mixed",
      "tag": "mixed-in",
      "listen": "127.0.0.1",
      "listen_port": 1080,
      "sniff": true
    }
  ]
}
```

这样会在本地 `127.0.0.1:1080` 开一个 socks5 + http 混合代理端口，然后：

```bash
# 终端临时使用
export https_proxy=http://127.0.0.1:1080
export http_proxy=http://127.0.0.1:1080

# 或者单条命令
curl -x socks5://127.0.0.1:1080 https://ipinfo.io/ip
```

## 七、其他协议配置示例

sing-box 不只支持 Shadowsocks，换协议只需要改 outbound。

### VMess

```json
{
  "type": "vmess",
  "tag": "proxy",
  "server": "服务器IP",
  "server_port": 443,
  "uuid": "你的UUID",
  "security": "auto",
  "transport": {
    "type": "ws",
    "path": "/path"
  },
  "tls": {
    "enabled": true,
    "server_name": "你的域名"
  }
}
```

### Hysteria2

```json
{
  "type": "hysteria2",
  "tag": "proxy",
  "server": "服务器IP",
  "server_port": 443,
  "password": "你的密码",
  "tls": {
    "enabled": true,
    "server_name": "你的域名"
  }
}
```

其他部分（dns、inbounds、route）保持不变，只替换 outbound 中的代理节点即可。

## 八、常见问题

### Q: 启动报 `tun: permission denied`

需要 root 权限运行，加 `sudo`。

### Q: 启动后本地网络不通

检查 route 规则里是否有 `geoip: ["private"]` 走 direct，这条保证局域网流量不走代理。

### Q: 怎么查看实时日志？

```bash
# systemd 方式
sudo journalctl -u sing-box -f

# 手动运行时直接看终端输出，或在配置里设置 log level
```

### Q: 配置文件语法检查

```bash
sing-box check -c /etc/sing-box/config.json
```

没有输出就是没问题。

## 九、总结

| 步骤 | 命令 |
|------|------|
| 安装 | `bash <(curl -fsSL https://sing-box.app/deb-install.sh)` |
| 编辑配置 | `sudo nano /etc/sing-box/config.json` |
| 检查配置 | `sing-box check -c /etc/sing-box/config.json` |
| 启动服务 | `sudo systemctl enable --now sing-box` |
| 验证 | `curl -s https://ipinfo.io/ip` |
| 查看日志 | `sudo journalctl -u sing-box -f` |

sing-box 的配置虽然是 JSON 手写，但结构清晰，理解了 inbound → route → outbound 的流向后，换协议、加规则都很直观。官方文档在 [sing-box.sagernet.org](https://sing-box.sagernet.org)，遇到问题可以查阅。
