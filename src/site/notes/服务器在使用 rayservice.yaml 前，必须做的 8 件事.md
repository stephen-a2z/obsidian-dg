---
{"dg-publish":true,"permalink":"/服务器在使用 rayservice.yaml 前，必须做的 8 件事/","created":"2026-03-19T17:35:11.011+08:00","updated":"2026-03-19T17:50:35.457+08:00"}
---


---
# 0. 服务器配置要求（必须满足）

- 系统：Ubuntu 20.04 / 22.04（最稳定）
- CPU：**至少 4 核**
- 内存：**至少 8G**
- 干净新系统最好（避免冲突）

# 1. 安装 Docker（必须）

```bash
apt update && apt install -y docker.io 
systemctl enable --now docker
```

验证：
```bash
docker --version
```

# **2. 安装 Kubernetes（必须，单节点也可以）**

### 安装 **k3s**（最轻量、最简单、10 秒安装）

```bash
curl -sfL https://get.k3s.io | sh -
```
验证：

```bash
kubectl get node
```

出现 `Ready` 就成功。

# 3. 安装 KubeRay Operator
 使用helm 安装(推荐)
 ```bash
 
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm repo add kuberay https://ray-project.github.io/kuberay-helm/
helm repo update
# Install both CRDs and KubeRay operator v1.5.1.
helm install kuberay-operator kuberay/kuberay-operator --version 1.5.1
 ```
验证:
```bash
 helm list
```

##  **Helm 最常用的 5 条命令（背会就够用）**

```bash
helm install      安装应用
helm uninstall    卸载
helm list         看安装了什么
helm upgrade      升级
helm repo add     添加商店
```
# 最后就可以部署`rayservice.yaml`了

```bash
kubectl apply -f rayservice.yaml
```
察看结果:
```bash

kubectl get pods -n ray-system -w

```