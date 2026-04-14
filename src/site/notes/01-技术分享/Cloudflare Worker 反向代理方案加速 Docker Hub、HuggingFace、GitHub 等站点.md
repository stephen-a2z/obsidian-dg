---
{"dg-publish":true,"permalink":"/01-技术分享/Cloudflare Worker 反向代理方案加速 Docker Hub、HuggingFace、GitHub 等站点/","tags":["docker","cloudflare","github","中转","加速"],"noteIcon":"","created":"2026-04-10T15:32:23.177+08:00","updated":"2026-04-10T15:38:16.541+08:00"}
---


## 1. 创建 Worker

在 Cloudflare Dashboard → Workers & Pages → Create Worker，贴入以下代码：

```javascript
const PROXY_RULES = {
  'docker.yourdomain.com': 'registry-1.docker.io',
  'ghcr.yourdomain.com': 'ghcr.io',
  'github.yourdomain.com': 'github.com',
  'raw.yourdomain.com': 'raw.githubusercontent.com',
  'hf.yourdomain.com': 'huggingface.co',
  'ghrel.yourdomain.com': 'objects.githubusercontent.com',
};

export default {
  async fetch(request) {
    const url = new URL(request.url);
    const upstream = PROXY_RULES[url.hostname];

    if (!upstream) {
      return new Response('Not configured', { status: 404 });
    }

    url.hostname = upstream;

    const newRequest = new Request(url, {
      method: request.method,
      headers: request.headers,
      body: request.body,
      redirect: 'follow',
    });

    newRequest.headers.set('Host', upstream);

    const response = await fetch(newRequest);

    const newHeaders = new Headers(response.headers);
    // 处理 Docker Registry 的认证重定向
    const wwwAuth = newHeaders.get('www-authenticate');
    if (wwwAuth) {
      newHeaders.set(
        'www-authenticate',
        wwwAuth.replace(
          'https://auth.docker.io',
          `https://${request.headers.get('host')}`
        )
      );
    }

    return new Response(response.body, {
      status: response.status,
      headers: newHeaders,
    });
  },
};

```



## 2. 绑定自定义域名

在 Worker 设置 → Triggers → Custom Domains，添加：

| 子域名 | 用途 |
|---|---|
| docker.yourdomain.com | Docker Hub |
| ghcr.yourdomain.com | GitHub Container Registry |
| github.yourdomain.com | GitHub |
| raw.yourdomain.com | GitHub Raw 文件 |
| hf.yourdomain.com | HuggingFace |
| ghrel.yourdomain.com | GitHub Releases 下载 |

前提：域名已托管在 Cloudflare DNS 上。

## 3. 使用方式

### Docker

```bash
# /etc/docker/daemon.json
{
  "registry-mirrors": ["https://docker.yourdomain.com"]
}

# 重启 docker
sudo systemctl restart docker

# 拉取镜像
docker pull docker.yourdomain.com/library/nginx:latest
```

### HuggingFace

```bash
export HF_ENDPOINT=https://hf.yourdomain.com
pip install -U huggingface_hub
huggingface-cli download meta-llama/Llama-2-7b
```


### GitHub Release / Raw 文件

```bash
# 下载 release
wget https://ghrel.yourdomain.com/user/repo/releases/download/v1.0/file.tar.gz

# raw 文件
curl https://raw.yourdomain.com/user/repo/main/README.md


### Git Clone（通过代理）

bash
git clone https://github.yourdomain.com/user/repo.git
```

## 4. 注意事项

- Cloudflare Workers 免费版每天 10 万次请求，一般个人使用足够
- Worker 单次请求体大小限制 100MB（免费版），大文件下载可能受限
- Docker 认证流程较复杂，上面的代码做了基本的 www-authenticate 头重写，如果遇到认证问题可能需要单独处理 auth.docker.io 的代理
- 建议开启 Cloudflare 的缓存规则，对静态资源设置缓存可进一步加速
- 不要公开分享你的代理域名，避免被滥用导致 Cloudflare 封号