---
{"dg-publish":true,"permalink":"/01-技术分享/BrowserStation 浏览器池搭建/","tags":["k3s","kube","ray","browserstation","chrome","cdp","stagehand"],"noteIcon":"","created":"2026-03-30T18:49:20.276+08:00","updated":"2026-03-31T18:52:37.758+08:00"}
---


# K3s + Ray + BrowserStation 浏览器池部署


基于 [ReinforceNow/browserstation](https://github.com/ReinforceNow/browserstation) 项目实际架构，完整的纯内网 10 节点部署方案。




## 架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                      K3s Cluster                            │
│                                                             │
│  ┌─────────────────────────────────┐                        │
│  │  Master (master01 / 10.0.0.10) │                        │
│  │  ┌───────────────────────────┐  │                        │
│  │  │ Ray Head Pod              │  │                        │
│  │  │  FastAPI :8050            │  │  ← NodePort 30050     │
│  │  │  Ray GCS :6379            │  │                        │
│  │  │  num-cpus=0 (不跑任务)    │  │                        │
│  │  └───────────────────────────┘  │                        │
│  │  KubeRay Operator               │                        │
│  └─────────────────────────────────┘                        │
│                                                             │
│  ┌──────────────────────┐  ┌──────────────────────┐        │
│  │ Agent01 (10.0.0.11)  │  │ Agent02 (10.0.0.12)  │  ...×9│
│  │ ┌──────────────────┐ │  │ ┌──────────────────┐ │        │
│  │ │ Ray Worker Pod    │ │  │ │ Ray Worker Pod    │ │        │
│  │ │ ┌──────────────┐ │ │  │ │ ┌──────────────┐ │ │        │
│  │ │ │ ray-worker   │ │ │  │ │ │ ray-worker   │ │ │        │
│  │ │ │ BrowserActor │ │ │  │ │ │ BrowserActor │ │ │        │
│  │ │ └──────┬───────┘ │ │  │ │ └──────┬───────┘ │ │        │
│  │ │        │localhost │ │  │ │        │localhost │ │        │
│  │ │ ┌──────▼───────┐ │ │  │ │ ┌──────▼───────┐ │ │        │
│  │ │ │ Chrome       │ │ │  │ │ │ Chrome       │ │ │        │
│  │ │ │ CDP :9222    │ │ │  │ │ │ CDP :9222    │ │ │        │
│  │ │ └──────────────┘ │ │  │ │ └──────────────┘ │ │        │
│  │ └──────────────────┘ │  │ └──────────────────┘ │        │
│  └──────────────────────┘  └──────────────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

客户端 → POST /browsers → 分配 Worker → 返回 WebSocket CDP URL
客户端 → WS /ws/browsers/{id}/devtools/browser → 代理到对应 Chrome


每个 Worker Pod 是一个 Sidecar 模式：ray-worker 容器 + chrome 容器共享 localhost 网络，通过 CDP 9222 端口通信。Head 节点的 FastAPI 负责调度和 WebSocket 代理。




## 一、离线资源准备（有网环境）

```bash
mkdir -p ~/browserstation-offline && cd ~/browserstation-offline

# 1) K3s
K3S_VER="v1.29.4+k3s1"
wget https://github.com/k3s-io/k3s/releases/download/${K3S_VER}/k3s
wget https://github.com/k3s-io/k3s/releases/download/${K3S_VER}/k3s-airgap-images-amd64.tar.zst
wget https://get.k3s.io -O install.sh

# 2) 构建 BrowserStation 镜像
git clone https://github.com/ReinforceNow/browserstation.git
cd browserstation
docker build -t browserstation:v1.0 -f Dockerfile.x86_64 .
cd ..

# 3) 拉取依赖镜像
docker pull zenika/alpine-chrome:100
docker pull quay.io/kuberay/operator:v1.3.0

# 4) 导出镜像
docker save browserstation:v1.0 zenika/alpine-chrome:100 | gzip > browserstation-images.tar.gz
docker save quay.io/kuberay/operator:v1.3.0 | gzip > kuberay-operator-image.tar.gz

# 5) KubeRay Helm Chart
helm repo add kuberay https://ray-project.github.io/kuberay-helm/ && helm repo update
helm pull kuberay/kuberay-operator --version 1.3.0
```


最终文件清单：

browserstation-offline/
├── k3s
├── k3s-airgap-images-amd64.tar.zst
├── install.sh
├── browserstation-images.tar.gz
├── kuberay-operator-image.tar.gz
├── kuberay-operator-1.3.0.tgz
├── setup-master.sh
└── setup-agent.sh





## 二、所有节点基础配置（10 台）

```bash
swapoff -a && sed -i '/swap/d' /etc/fstab
systemctl disable --now firewalld 2>/dev/null || true

cat >> /etc/hosts <<'EOF'
10.0.0.10 master01
10.0.0.11 agent01
10.0.0.12 agent02
10.0.0.13 agent03
10.0.0.14 agent04
10.0.0.15 agent05
10.0.0.16 agent06
10.0.0.17 agent07
10.0.0.18 agent08
10.0.0.19 agent09
EOF
```


需要放行的端口（如果不能关防火墙）：

| 端口 | 协议 | 用途 |
|------|------|------|
| 6443 | TCP | K3s API Server |
| 8472 | UDP | VXLAN (Flannel) |
| 10250 | TCP | Kubelet |
| 51820 | UDP | WireGuard (可选) |
| 6379 | TCP | Ray GCS (Pod 间) |
| 30050 | TCP | BrowserStation NodePort |




## 三、分发离线文件

```bash
# 在 master01 上执行
AGENTS="10.0.0.11 10.0.0.12 10.0.0.13 10.0.0.14 10.0.0.15 10.0.0.16 10.0.0.17 10.0.0.18 10.0.0.19"
D="/opt/k3s-offline"

for ip in $AGENTS; do
  ssh root@${ip} "mkdir -p ${D} /var/lib/rancher/k3s/agent/images/"
  scp ${D}/k3s root@${ip}:/usr/local/bin/k3s
  ssh root@${ip} "chmod +x /usr/local/bin/k3s"
  scp ${D}/k3s-airgap-images-amd64.tar.zst root@${ip}:/var/lib/rancher/k3s/agent/images/
  scp ${D}/install.sh ${D}/browserstation-images.tar.gz ${D}/setup-agent.sh root@${ip}:${D}/
done
```






## 四、Master 一键脚本 setup-master.sh

```bash
#!/usr/bin/env bash
set -euo pipefail

MASTER_IP="${MASTER_IP:-10.0.0.10}"
NODE_NAME="${NODE_NAME:-master01}"
D="${OFFLINE_DIR:-/opt/k3s-offline}"
REPLICAS="${WORKER_REPLICAS:-9}"

# --- K3s Server ---
echo ">>> [1/6] K3s 离线文件"
cp "${D}/k3s" /usr/local/bin/k3s && chmod +x /usr/local/bin/k3s
mkdir -p /var/lib/rancher/k3s/agent/images/
cp "${D}/k3s-airgap-images-amd64.tar.zst" /var/lib/rancher/k3s/agent/images/

echo ">>> [2/6] 安装 K3s Server"
swapoff -a
INSTALL_K3S_SKIP_DOWNLOAD=true bash "${D}/install.sh" \
  --node-name "${NODE_NAME}" \
  --tls-san "${MASTER_IP}" \
  --disable traefik \
  --write-kubeconfig-mode 644

export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl wait --for=condition=Ready node/"${NODE_NAME}" --timeout=180s

# --- 镜像导入 ---
echo ">>> [3/6] 导入镜像"
gunzip -c "${D}/browserstation-images.tar.gz" | k3s ctr images import -
gunzip -c "${D}/kuberay-operator-image.tar.gz" | k3s ctr images import -

# --- KubeRay Operator ---
echo ">>> [4/6] 安装 KubeRay Operator"
kubectl create namespace ray-system --dry-run=client -o yaml | kubectl apply -f -
helm install kuberay-operator "${D}/kuberay-operator-1.3.0.tgz" \
  --namespace ray-system \
  --set image.pullPolicy=Never \
  --wait --timeout 180s

# --- RayService ---
echo ">>> [5/6] 部署 BrowserStation RayService"
cat <<YAML | kubectl apply -f -
apiVersion: ray.io/v1alpha1
kind: RayService
metadata:
  name: browser-cluster
  namespace: ray-system
spec:
  rayClusterConfig:
    rayVersion: '2.47.1'
    headGroupSpec:
      rayStartParams:
        dashboard-host: '0.0.0.0'
        num-cpus: "0"
      template:
        spec:
          containers:
          - name: ray-head
            image: browserstation:v1.0
            imagePullPolicy: Never
            env:
            - name: RAY_memory_usage_threshold
              value: "0.95"
            ports:
            - containerPort: 8050
              name: http
            command:
            - /bin/bash
            - -c
            - >
              ray start --head --port=6379
              --dashboard-host=0.0.0.0
              --metrics-export-port=8080
              --num-cpus=0 --block &
              sleep 10 &&
              uvicorn app.main:app --host 0.0.0.0 --port 8050
    workerGroupSpecs:
    - groupName: browser-workers
      replicas: ${REPLICAS}
      rayStartParams: {}
      template:
        spec:
          containers:
          - name: ray-worker
            image: browserstation:v1.0
            imagePullPolicy: Never
            resources:
              requests:
                cpu: "100m"
                memory: "256Mi"
          - name: chrome
            image: zenika/alpine-chrome:100
            imagePullPolicy: Never
            securityContext:
              runAsUser: 0
              runAsNonRoot: false
            args:
            - --no-sandbox
            - --remote-debugging-address=0.0.0.0
            - --remote-debugging-port=9222
            ports:
            - containerPort: 9222
              name: devtools
            resources:
              requests:
                cpu: "900m"
                memory: "768Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: browser-cluster-public
  namespace: ray-system
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: kuberay
    ray.io/node-type: head
  ports:
  - name: serve
    port: 8050
    targetPort: 8050
    nodePort: 30050
YAML

# --- 等待就绪 ---
echo ">>> [6/6] 等待服务就绪"
kubectl wait --for=condition=Ready pod -l ray.io/node-type=head \
  -n ray-system --timeout=300s

TOKEN=$(cat /var/lib/rancher/k3s/server/node-token)
cat <<EOF

============================================
✅ Master 部署完成

  K3S_URL:        https://${MASTER_IP}:6443
  K3S_TOKEN:      ${TOKEN}
  BrowserStation: http://${MASTER_IP}:30050

  Agent 安装:
    K3S_TOKEN='${TOKEN}' MASTER_IP=${MASTER_IP} bash setup-agent.sh
============================================
EOF

```





## 五、Agent 一键脚本 setup-agent.sh

```bash
#!/usr/bin/env bash
set -euo pipefail

MASTER_IP="${MASTER_IP:-10.0.0.10}"
NODE_NAME="${NODE_NAME:-$(hostname)}"
D="${OFFLINE_DIR:-/opt/k3s-offline}"

[ -z "${K3S_TOKEN:-}" ] && { echo "❌ 需要 K3S_TOKEN"; exit 1; }

echo ">>> [1/3] K3s 离线文件"
cp "${D}/k3s" /usr/local/bin/k3s && chmod +x /usr/local/bin/k3s
mkdir -p /var/lib/rancher/k3s/agent/images/
cp "${D}/k3s-airgap-images-amd64.tar.zst" /var/lib/rancher/k3s/agent/images/

echo ">>> [2/3] 加入集群"
swapoff -a
INSTALL_K3S_SKIP_DOWNLOAD=true \
  K3S_URL="https://${MASTER_IP}:6443" \
  K3S_TOKEN="${K3S_TOKEN}" \
  bash "${D}/install.sh" --node-name "${NODE_NAME}"

echo ">>> [3/3] 导入镜像"
gunzip -c "${D}/browserstation-images.tar.gz" | k3s ctr images import -

echo "✅ Agent ${NODE_NAME} 已加入 https://${MASTER_IP}:6443"
```



## 六、部署验证

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# 集群节点
kubectl get nodes
# NAME       STATUS   ROLES                  AGE   VERSION
# master01   Ready    control-plane,master   10m   v1.29.4+k3s1
# agent01    Ready    <none>                 8m    v1.29.4+k3s1
# ...共 10 个 Ready

# Ray 集群 Pod
kubectl get pods -n ray-system
# NAME                                  READY   STATUS    RESTARTS
# kuberay-operator-xxx                  1/1     Running   0
# browser-cluster-head-xxx              1/1     Running   0
# browser-cluster-worker-browser-xxx    2/2     Running   0    ← 注意 2/2 (sidecar)
# ...共 9 个 worker

# 健康检查
curl http://10.0.0.10:30050/
# {"status": "ok"}

# 创建浏览器实例
curl -X POST http://10.0.0.10:30050/browsers
# {"id": "xxx", "ws_url": "ws://...", ...}

# 查看浏览器池
curl http://10.0.0.10:30050/browsers
# [{"id": "xxx", "status": "running", "cpu_usage": ...}, ...]

# 关闭浏览器
curl -X DELETE http://10.0.0.10:30050/browsers/{id}
```




## 七、对接 AI Agent 示例

BrowserStation 提供标准 CDP WebSocket，可直接对接 Browser-use / Langchain / Playwright：

```python
# pip install playwright
from playwright.async_api import async_playwright
import httpx, asyncio

API = "http://10.0.0.10:30050"

async def main():
    # 从浏览器池申请一个实例
    r = httpx.post(f"{API}/browsers")
    browser_info = r.json()
    ws_url = browser_info["ws_url"]

    async with async_playwright() as p:
        browser = await p.chromium.connect_over_cdp(ws_url)
        page = browser.contexts[0].pages[0]
        await page.goto("https://example.com")
        print(await page.title())
        await browser.close()

    # 释放实例
    httpx.delete(f"{API}/browsers/{browser_info['id']}")

asyncio.run(main())
```



## 关键注意事项

- 每个 Worker Pod 需要约 1 CPU + 1Gi 内存（ray-worker 100m+256Mi + chrome 900m+768Mi），确保 Agent 节点资源充足
- imagePullPolicy: Never 是离线部署的关键，确保所有节点都已导入镜像再部署 RayService
- WORKER_REPLICAS 默认 9（每个 Agent 一个），可根据节点资源调整，单节点跑多个 Worker 也可以
- Head 节点设置 num-cpus=0 确保不会被调度浏览器任务，只做 API 网关和调度
- 如果需要 API Key 认证，设置环境变量 BROWSERSTATION_API_KEY 到 Head Pod 的 env 中