---
{"dg-publish":true,"permalink":"/K3s vs K8s 完整对比教程/","created":"2026-03-19T18:12:41.806+08:00","updated":"2026-03-20T11:19:49.397+08:00"}
---

# K3s vs K8s 完整对比教程

## 一、什么是 K8s 和 K3s

K8s（Kubernetes） — CNCF 维护的容器编排标准，功能完整，适合企业级大规模集群。

K3s — Rancher Labs（现 SUSE）开发的轻量级 Kubernetes 发行版，单个二进制文件 < 100MB，完全兼容 K8s API，专为边缘计算、IoT、开发环境和资源受限场景设计。

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


## 二、核心架构对比
```
┌─────────────────────────────────────────────────────────────┐
│                    K8s 架构（标准）                           │
│                                                             │
│  Control Plane:                                             │
│  ┌──────────┐ ┌───────────┐ ┌──────────┐ ┌──────────────┐  │
│  │kube-api  │ │controller │ │scheduler │ │  etcd (集群)  │  │
│  │server    │ │manager    │ │          │ │              │  │
│  └──────────┘ └───────────┘ └──────────┘ └──────────────┘  │
│                                                             │
│  Worker Node:                                               │
│  ┌──────────┐ ┌───────────┐ ┌──────────────────┐           │
│  │ kubelet  │ │kube-proxy │ │ Container Runtime│           │
│  │          │ │           │ │ (containerd/CRI-O)│          │
│  └──────────┘ └───────────┘ └──────────────────┘           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    K3s 架构（精简）                           │
│                                                             │
│  Server (单进程):                                            │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  k3s server (内含 api-server + scheduler +          │    │
│  │  controller-manager + SQLite/etcd + containerd)     │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  Agent:                                                     │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  k3s agent (内含 kubelet + flannel + containerd)    │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


## 三、详细对比表

| 维度           | K8s                    | K3s                                   |
| ------------ | ---------------------- | ------------------------------------- |
| 二进制大小        | 数百 MB（多组件）             | < 100MB（单二进制）                         |
| 最低内存         | Control Plane 2GB+     | Server 512MB 可运行                      |
| 默认存储后端       | etcd（集群模式）             | SQLite（单节点）/ 可选 etcd、MySQL、PostgreSQL |
| 容器运行时        | containerd / CRI-O     | 内置 containerd                         |
| 网络 CNI       | 需手动安装（Calico/Cilium 等） | 内置 Flannel（可替换）                       |
| Ingress      | 需手动部署                  | 内置 Traefik                            |
| 负载均衡         | 需 MetalLB 或云 LB        | 内置 ServiceLB（Klipper）                 |
| 存储           | 需手动配置 CSI              | 内置 Local Path Provisioner             |
| 安装时间         | 30 分钟 - 数小时            | < 1 分钟                                |
| HA 方案        | 3+ etcd 节点             | 嵌入式 etcd 或外部 DB                       |
| 认证           | CNCF 认证                | 同样通过 CNCF 一致性认证                       |
| Helm/kubectl | 兼容                     | 完全兼容                                  |
| 适用规模         | 数百到数千节点                | 建议 < 500 节点                           |
| GPU 支持       | 完整                     | 支持（需额外配置）                             |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


## 四、安装对比

### K8s 安装（kubeadm 方式）

bash
# --- 每台机器都要执行 ---
# 1. 安装容器运行时
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd

# 2. 安装 kubeadm/kubelet/kubectl

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```

# 3. 关闭 swap
```bash
sudo swapoff -a
```

# --- Master 节点 ---
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
```

# 4. 安装 CNI（以 Calico 为例）
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```
# --- Worker 节点 ---
```bash
sudo kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```


### K3s 安装

```bash
# --- Server 节点（一条命令）---
curl -sfL https://get.k3s.io | sh -

# 获取 token
sudo cat /var/lib/rancher/k3s/server/node-token

# --- Agent 节点（一条命令）---
curl -sfL https://get.k3s.io | K3S_URL=https://<server-ip>:6443 K3S_TOKEN=<token> sh -

# kubectl 直接可用
sudo k3s kubectl get nodes
# 或复制 kubeconfig
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config

```
```
安装时间对比：K8s 约 30-60 分钟 vs K3s 约 30 秒。

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


## 五、高可用（HA）对比

### K8s HA

                    ┌──────────────┐
                    │  Load Balancer│
                    └──────┬───────┘
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Master 1 │ │ Master 2 │ │ Master 3 │
        │ etcd     │ │ etcd     │ │ etcd     │
        │ api-srv  │ │ api-srv  │ │ api-srv  │
        └──────────┘ └──────────┘ └──────────┘


需要：3 台 Master + 外部 LB + etcd 集群管理，运维复杂度高。

### K3s HA（嵌入式 etcd）

bash
# 第一个 Server
curl -sfL https://get.k3s.io | sh -s - server --cluster-init

# 加入更多 Server
curl -sfL https://get.k3s.io | K3S_TOKEN=<token> sh -s - server --server https://<first-server>:6443


### K3s HA（外部数据库）

bash
# 使用已有的 PostgreSQL/MySQL
curl -sfL https://get.k3s.io | sh -s - server \
  --datastore-endpoint="postgres://user:pass@db-host:5432/k3s"


K3s 的 HA 配置明显更简单，且支持复用已有的关系型数据库。

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


## 六、资源消耗实测对比

场景：空集群，1 Server + 2 Worker

```

┌──────────────┬────────────────┬────────────────┐
│              │   K8s (kubeadm)│   K3s          │
├──────────────┼────────────────┼────────────────┤
│ Master 内存  │   ~1.2 GB      │   ~400 MB      │
│ Worker 内存  │   ~300 MB      │   ~150 MB      │
│ 磁盘占用     │   ~3 GB        │   ~200 MB      │
│ 系统组件 Pod │   8-10 个      │   3-5 个       │
│ 启动时间     │   2-5 分钟     │   15-30 秒     │
└──────────────┴────────────────┴────────────────┘
```


K3s 在资源消耗上大约是 K8s 的 1/3 到 1/5。

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


## 七、日常运维对比

### 升级

```bash
# K8s 升级（需逐节点操作）
sudo apt-get update
sudo apt-get install -y kubeadm=1.31.x-*
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.31.x
sudo apt-get install -y kubelet=1.31.x-* kubectl=1.31.x-*
sudo systemctl daemon-reload && sudo systemctl restart kubelet
# 每个 Worker 重复以上步骤...

# K3s 升级（重新运行安装脚本即可）
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=latest sh -
```

### 备份

```bash
# K8s — 需要单独备份 etcd
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# K3s — SQLite 模式直接复制文件
sudo cp /var/lib/rancher/k3s/server/db/state.db /backup/k3s-state.db

# K3s — etcd 模式
k3s etcd-snapshot save --name my-backup
```


### 卸载

bash
# K8s
kubeadm reset
sudo rm -rf /etc/kubernetes /var/lib/etcd ~/.kube

# K3s
/usr/local/bin/k3s-uninstall.sh        # Server
/usr/local/bin/k3s-agent-uninstall.sh   # Agent


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


## 八、选型决策树

你的场景是什么？
```
│
├─ 边缘/IoT/ARM 设备 ──────────────────→ K3s
├─ 开发/测试/CI 环境 ──────────────────→ K3s
├─ 个人项目 / 小团队（< 10 服务）──────→ K3s
├─ VPS 上跑几个服务（内存 < 4GB）──────→ K3s
│
├─ 企业生产环境（> 50 节点）───────────→ K8s（或托管 K8s）
├─ 需要高级网络策略（Cilium/Istio）───→ K8s
├─ 多租户隔离要求高 ──────────────────→ K8s
├─ 已有 K8s 运维团队 ─────────────────→ K8s
│
└─ 不想自己运维 ──────────────────────→ 托管服务
   ├─ AWS EKS
   ├─ GCP GKE
   ├─ Azure AKS
   └─ DigitalOcean DOKS
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


## 九、从 K3s 迁移到 K8s

因为 K3s 完全兼容 K8s API，迁移很平滑：

```bash
# 1. 导出所有资源
k3s kubectl get all --all-namespaces -o yaml > all-resources.yaml

# 2. 导出关键资源（排除系统命名空间）
for ns in $(k3s kubectl get ns -o jsonpath='{.items[*].metadata.name}' | tr ' ' '\n' | grep -v kube-); do
  k3s kubectl -n $ns get deploy,svc,configmap,secret,ingress -o yaml > "${ns}-resources.yaml"
done

# 3. 在 K8s 集群上应用
kubectl apply -f all-resources.yaml

```
Helm charts、kubectl 命令、CI/CD 流水线都不需要改动。

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


## 十、总结


|     | K3s | K8s |
|---|---|---|
| 一句话 | 轻量快速，开箱即用 | 功能完整，生态丰富 |
| 学习曲线 | 低 | 高 |
| 运维成本 | 低 | 高 |
| 灵活性 | 够用 | 极高 |
| 适合 | 个人/小团队/边缘/开发 | 企业/大规模/复杂场景 |

实际建议： 如果你不确定选哪个，先用 K3s。它和 K8s API 完全兼容，学到的知识可以直接迁移，等真正遇到 K3s 的瓶颈时再切换到 K8s 也不迟。对于在 VPS 上跑几个服务的场景，K3s 是明显更合适的选择。