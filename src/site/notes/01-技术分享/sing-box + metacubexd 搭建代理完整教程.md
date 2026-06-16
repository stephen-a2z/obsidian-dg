---
{"dg-publish":true,"permalink":"/01-技术分享/sing-box + metacubexd 搭建代理完整教程/","tags":["sing-box","metacubexd","proxy","代理","Web面板","Linux"],"noteIcon":"","created":"2026-06-16T14:23:59.814+08:00","updated":"2026-06-16T14:23:59.815+08:00"}
---


# sing-box + metacubexd 搭建代理完整教程（服务端 + 客户端）

> 本教程介绍如何在云主机上部署 sing-box 代理服务端，配合 metacubexd Web 面板进行管理，并配置客户端连接。协议选用 VLESS + Reality（推荐）和 Hysteria2 两种方案。

## 架构概览

```
客户端 (sing-box) ──→ 服务端 (sing-box inbound) ──→ 互联网
        ↑                        ↑
   本地 metacubexd          服务端 metacubexd
   (管理客户端分流)         (监控服务端状态)
```

---

## 一、服务端部署

### 1.1 安装 sing-box

```bash
# Debian/Ubuntu 官方安装
bash <(curl -fsSL https://sing-box.app/deb-install.sh)

# 验证安装
sing-box version
```

### 1.2 生成密钥（VLESS Reality 方案）

```bash
# 生成 Reality 密钥对
sing-box generate reality-keypair
# 输出示例：
# PrivateKey: uMOQ...xxxxx
# PublicKey: aBcD...xxxxx

# 生成 UUID
sing-box generate uuid
# 输出示例：a1b2c3d4-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# 生成短 ID
openssl rand -hex 8
```

记下这三个值，后面要用。

### 1.3 服务端配置文件

```bash
sudo nano /etc/sing-box/config.json
```

```json
{
  "log": {
    "level": "info",
    "timestamp": true
  },
  "experimental": {
    "clash_api": {
      "external_controller": "127.0.0.1:9090",
      "external_ui": "/var/lib/sing-box/ui",
      "secret": "你的管理密码"
    }
  },
  "inbounds": [
    {
      "type": "vless",
      "tag": "vless-in",
      "listen": "::",
      "listen_port": 443,
      "users": [
        {
          "uuid": "你生成的UUID",
          "flow": "xtls-rprx-vision"
        }
      ],
      "tls": {
        "enabled": true,
        "server_name": "www.microsoft.com",
        "reality": {
          "enabled": true,
          "handshake": {
            "server": "www.microsoft.com",
            "server_port": 443
          },
          "private_key": "你生成的PrivateKey",
          "short_id": ["你生成的shortID"]
        }
      }
    },
    {
      "type": "hysteria2",
      "tag": "hy2-in",
      "listen": "::",
      "listen_port": 8443,
      "users": [
        {
          "password": "你的hysteria2密码"
        }
      ],
      "tls": {
        "enabled": true,
        "alpn": ["h3"],
        "certificate_path": "/etc/sing-box/cert.pem",
        "key_path": "/etc/sing-box/key.pem"
      }
    }
  ],
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct"
    }
  ]
}
```

> **注意**：Hysteria2 需要 TLS 证书。如果没有域名，可以用自签证书：
> ```bash
> openssl req -x509 -nodes -newkey ec:<(openssl ecparam -name prime256v1) \
>   -keyout /etc/sing-box/key.pem -out /etc/sing-box/cert.pem \
>   -subj "/CN=bing.com" -days 36500
> ```

### 1.4 部署 metacubexd 面板

```bash
# 创建 UI 目录
sudo mkdir -p /var/lib/sing-box/ui

# 下载最新 metacubexd
sudo curl -Lo /tmp/metacubexd.tar.gz https://github.com/MetaCubeXD/metacubexd/releases/latest/download/compressed-dist.tgz

# 解压到 UI 目录
sudo tar -xzf /tmp/metacubexd.tar.gz -C /var/lib/sing-box/ui --strip-components=0
```

### 1.5 启动服务

```bash
# 检查配置
sing-box check -c /etc/sing-box/config.json

# 启动并设为开机自启
sudo systemctl enable --now sing-box

# 查看状态
sudo systemctl status sing-box
```

### 1.6 防火墙放行

```bash
# 放行代理端口
sudo ufw allow 443/tcp    # VLESS Reality
sudo ufw allow 8443/udp   # Hysteria2
# 注意：9090 端口不要对外开放！
```

### 1.7 安全访问 metacubexd 面板

方法一：SSH 隧道（推荐）

```bash
# 本地执行
ssh -L 9090:127.0.0.1:9090 user@你的服务器IP

# 然后浏览器访问 http://127.0.0.1:9090/ui
# 输入 secret 密码即可管理
```

方法二：使用在线版面板

直接访问 `https://d.metacubex.one`，填入：
- Host: `http://127.0.0.1:9090`（需要 SSH 隧道）
- Secret: 你设置的管理密码

---

## 二、客户端配置

### 2.1 Linux / macOS 客户端

安装 sing-box 后，创建客户端配置文件 `config.json`：

```json
{
  "log": {
    "level": "info"
  },
  "experimental": {
    "clash_api": {
      "external_controller": "127.0.0.1:9090",
      "external_ui": "ui",
      "secret": "local-secret"
    }
  },
  "dns": {
    "servers": [
      {
        "tag": "remote-dns",
        "address": "https://8.8.8.8/dns-query",
        "detour": "proxy"
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
      },
      {
        "rule_set": "geosite-cn",
        "server": "local-dns"
      }
    ],
    "strategy": "prefer_ipv4"
  },
  "inbounds": [
    {
      "type": "tun",
      "tag": "tun-in",
      "interface_name": "tun0",
      "address": ["172.19.0.1/30"],
      "auto_route": true,
      "strict_route": true,
      "stack": "system",
      "sniff": true
    }
  ],
  "outbounds": [
    {
      "type": "selector",
      "tag": "proxy",
      "outbounds": ["vless-reality", "hysteria2", "direct"],
      "default": "vless-reality"
    },
    {
      "type": "vless",
      "tag": "vless-reality",
      "server": "你的服务器IP",
      "server_port": 443,
      "uuid": "你的UUID",
      "flow": "xtls-rprx-vision",
      "tls": {
        "enabled": true,
        "server_name": "www.microsoft.com",
        "utls": {
          "enabled": true,
          "fingerprint": "chrome"
        },
        "reality": {
          "enabled": true,
          "public_key": "你的PublicKey",
          "short_id": "你的shortID"
        }
      }
    },
    {
      "type": "hysteria2",
      "tag": "hysteria2",
      "server": "你的服务器IP",
      "server_port": 8443,
      "password": "你的hysteria2密码",
      "tls": {
        "enabled": true,
        "insecure": true,
        "alpn": ["h3"]
      }
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
        "ip_is_private": true,
        "outbound": "direct"
      },
      {
        "rule_set": "geosite-cn",
        "outbound": "direct"
      },
      {
        "rule_set": "geoip-cn",
        "outbound": "direct"
      }
    ],
    "rule_set": [
      {
        "type": "remote",
        "tag": "geosite-cn",
        "format": "binary",
        "url": "https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-cn.srs"
      },
      {
        "type": "remote",
        "tag": "geoip-cn",
        "format": "binary",
        "url": "https://raw.githubusercontent.com/SagerNet/sing-geoip/rule-set/geoip-cn.srs"
      }
    ],
    "auto_detect_interface": true
  }
}
```

### 2.2 运行客户端

```bash
# 测试运行
sudo sing-box run -c config.json

# 验证
curl -s https://ipinfo.io/ip
```

### 2.3 客户端 metacubexd 面板

启动后直接浏览器访问 `http://127.0.0.1:9090/ui`

功能：
- **代理切换**：在 selector 节点间切换（VLESS / Hysteria2 / 直连）
- **延迟测试**：一键测速所有节点
- **实时流量**：监控上下行带宽
- **连接管理**：查看/关闭当前所有连接
- **规则查看**：确认分流规则是否生效

或者使用在线版：访问 `https://d.metacubex.one`，Backend 填 `http://127.0.0.1:9090`，Secret 填 `local-secret`。

### 2.4 Windows 客户端

下载 [sing-box Windows 版](https://github.com/SagerNet/sing-box/releases)，配置相同，将 tun 的 `stack` 改为 `gvisor`：

```json
{
  "type": "tun",
  "tag": "tun-in",
  "address": ["172.19.0.1/30"],
  "auto_route": true,
  "strict_route": true,
  "stack": "gvisor",
  "sniff": true
}
```

以管理员权限运行：
```cmd
sing-box.exe run -c config.json
```

### 2.5 iOS / Android 客户端

- **iOS**: App Store 搜索 `sing-box`（官方客户端）
- **Android**: GitHub Releases 下载 APK 或 Google Play

导入配置方式：
1. 将配置文件上传到可访问的 URL
2. 在 App 中通过 URL 导入
3. 或手动粘贴 JSON 配置

移动端去掉 `clash_api` 和 `tun` 部分，App 自身会处理。

---

## 三、运维管理

### 3.1 常用命令

```bash
# 重启服务（修改配置后）
sudo systemctl restart sing-box

# 实时日志
sudo journalctl -u sing-box -f

# 检查配置语法
sing-box check -c /etc/sing-box/config.json

# 更新 sing-box
bash <(curl -fsSL https://sing-box.app/deb-install.sh)

# 更新 metacubexd 面板
sudo curl -Lo /tmp/metacubexd.tar.gz https://github.com/MetaCubeXD/metacubexd/releases/latest/download/compressed-dist.tgz
sudo rm -rf /var/lib/sing-box/ui/*
sudo tar -xzf /tmp/metacubexd.tar.gz -C /var/lib/sing-box/ui
```

### 3.2 多用户管理

在 inbound 的 `users` 数组中添加多个用户：

```json
"users": [
  { "uuid": "用户1-uuid", "flow": "xtls-rprx-vision" },
  { "uuid": "用户2-uuid", "flow": "xtls-rprx-vision" }
]
```

### 3.3 安全加固

1. SSH 密钥登录，禁用密码
2. Clash API 绑定 `127.0.0.1`，不暴露公网
3. 定期更新 sing-box 版本
4. 使用 fail2ban 防止端口扫描

---

## 四、常见问题

| 问题 | 解决方案 |
|------|---------|
| Reality 连不上 | 检查 server_name 和 handshake server 是否一致，公钥私钥是否对应 |
| Hysteria2 连不上 | 确认 UDP 端口已放行，自签证书客户端需设 `insecure: true` |
| metacubexd 无法连接后端 | 确认 SSH 隧道已建立，或 secret 是否正确 |
| DNS 泄露 | 确认 DNS 规则中国内域名走 local-dns，其他走 remote-dns |
| 客户端启动报权限错误 | tun 模式需要 root/管理员权限 |

---

## 五、总结

| 组件 | 作用 |
|------|------|
| sing-box 服务端 | 接收客户端连接，转发流量 |
| sing-box 客户端 | 本地代理/全局代理 + 智能分流 |
| metacubexd | Web 面板，可视化管理节点切换、流量监控 |
| Reality / Hysteria2 | 抗封锁协议，无需域名即可部署 |

推荐协议选择：
- **稳定优先**：VLESS + Reality（伪装正常 TLS 流量，抗检测强）
- **速度优先**：Hysteria2（基于 QUIC，高丢包环境表现好）
- **两个都配**：用 selector 在面板一键切换
