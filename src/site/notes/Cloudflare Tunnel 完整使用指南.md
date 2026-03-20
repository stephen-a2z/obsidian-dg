---
{"dg-publish":true,"permalink":"/Cloudflare Tunnel 完整使用指南/","tags":["技术","教程"],"created":"2026-03-12T11:46:22.686+08:00","updated":"2026-03-20T12:18:40.420+08:00"}
---



## 1. 安装 cloudflared

macOS:

```bash
brew install cloudflare/cloudflare/cloudflared
```

Linux:

```bash
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64
sudo mv cloudflared-linux-amd64 /usr/local/bin/cloudflared
sudo chmod +x /usr/local/bin/cloudflared
```



验证安装:

```bash
cloudflared --version
```


## 2. 登录 Cloudflare 账户

```bash
cloudflared tunnel login
```


- 会打开浏览器
- 选择要使用的域名/zone
- 授权后会在 ~/.cloudflared/ 生成 cert.pem

## 3. 创建 Tunnel

```bash
cloudflared tunnel create my-tunnel
```

会生成：
- Tunnel ID
- Credentials 文件：`~/.cloudflared/<tunnel-id>.json`

查看所有 tunnel:

```bash
cloudflared tunnel list
```


## 4. 创建配置文件

创建 ~/.cloudflared/config.yml:


```yml
	tunnel: <tunnel-id>
	credentials-file: /Users/yourname/.cloudflared/<tunnel-id>.json
	
	ingress:
	  # SSH 服务
	  - hostname: ssh.example.com
	    service: ssh://localhost:22
	
	  # Web 应用
	  - hostname: app.example.com
	    service: http://localhost:3000
	
	  # HTTPS 应用
	  - hostname: api.example.com
	    service: https://localhost:8443
	
	  # 多个域名指向同一服务
	  - hostname: www.example.com
	    service: http://localhost:80
	  - hostname: example.com
	    service: http://localhost:80
	
	  # 兜底规则（必须）
	  - service: http_status:404
```


常用 service 类型:
- ssh://localhost:22 - SSH
- http://localhost:3000 - HTTP
- https://localhost:8443 - HTTPS
- tcp://localhost:5432 - TCP（如数据库）
- unix:/path/to/socket - Unix socket

## 5. 配置 DNS 路由

为每个域名创建 DNS 记录：


```bash
cloudflared tunnel route dns my-tunnel ssh.example.com
cloudflared tunnel route dns my-tunnel app.example.com
cloudflared tunnel route dns my-tunnel api.example.com
```


这会在 Cloudflare DNS 中自动创建 CNAME 记录指向 `<tunnel-id>.cfargotunnel.com`

查看路由:
```bash
cloudflared tunnel route list
```


删除路由:
```bash
cloudflared tunnel route dns delete ssh.example.com
```


## 6. 测试配置

```
bash
cloudflared tunnel ingress validate
```


测试特定 URL:
```bash
cloudflared tunnel ingress rule https://app.example.com
```

## 7. 运行 Tunnel

前台运行（测试用）:
```bash
cloudflared tunnel run my-tunnel
```


后台运行（生产环境）:

### Linux (systemd):

bash
# 安装服务
```bash
sudo cloudflared service install
```

# 启动服务
```bash
sudo systemctl start cloudflared
sudo systemctl enable cloudflared
```
# 查看状态
```sh
sudo systemctl status cloudflared
```

# 查看日志
```sh
sudo journalctl -u cloudflared -f
```

# 重启服务
```sh
sudo systemctl restart cloudflared
```


### macOS (launchd):

# 安装服务
```zsh
brew services start cloudflare/cloudflare/cloudflared
```

# 重启服务
```sh
brew services restart cloudflare/cloudflare/cloudflared
```

# 停止服务
brew services stop cloudflare/cloudflare/cloudflared


## 8. 添加新域名

1. 更新配置文件 ~/.cloudflared/config.yml:
```yaml
ingress:
  - hostname: new.example.com
    service: http://localhost:4000
  # ... 其他配置
  - service: http_status:404
```


2. 添加 DNS 路由:
```bash
cloudflared tunnel route dns my-tunnel new.example.com
```


3. 重启服务:
```bash
# systemd
sudo systemctl restart cloudflared
```

# macOS
```
brew services restart cloudflare/cloudflare/cloudflared
```

# 手动运行
# Ctrl+C 停止后重新运行
```
cloudflared tunnel run my-tunnel
```


## 9. 常用管理命令

bash
# 列出所有 tunnel
cloudflared tunnel list

# 查看 tunnel 信息
cloudflared tunnel info my-tunnel

# 删除 tunnel
cloudflared tunnel delete my-tunnel

# 清理未使用的连接
cloudflared tunnel cleanup my-tunnel

# 查看路由
cloudflared tunnel route list


## 10. 验证和调试

检查 DNS:
bash
dig ssh.example.com
nslookup app.example.com


测试连接:
bash
# HTTP/HTTPS
curl https://app.example.com

# SSH
ssh user@ssh.example.com


查看日志:
bash
# systemd
sudo journalctl -u cloudflared -f

# 手动运行时直接看终端输出
cloudflared tunnel run my-tunnel


## 11. 多账户/多 Zone 支持

一个 tunnel 可以服务多个域名和 zone：

```yaml
ingress:
  - hostname: app.domain-a.com
    service: http://localhost:3000
  - hostname: api.domain-b.com
    service: http://localhost:4000
  - hostname: ssh.domain-c.com
    service: ssh://localhost:22
  - service: http_status:404
```


只要这些域名都在你的 Cloudflare 账户中即可。

## 12. 配置文件位置

- **Linux:** /etc/cloudflared/config.yml 或 ~/.cloudflared/config.yml
- **macOS:** ~/.cloudflared/config.yml
- **Credentials:** ~/.cloudflared/<tunnel-id>.json
- **Cert:** ~/.cloudflared/cert.pem

## 常见问题

Q: 修改配置后不生效？
A: 记得重启服务

Q: DNS 解析不到？
A: 等待 DNS 传播（通常几分钟），检查 cloudflared tunnel route list

Q: 连接失败？
A: 检查本地服务是否运行，查看 cloudflared 日志

Q: 想用不同的配置文件？
A: cloudflared tunnel --config /path/to/config.yml run my-tunnel