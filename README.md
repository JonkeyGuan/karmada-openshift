# Karmada on OpenShift 安装指南

在 OpenShift 4.20 集群上通过 Helm Chart 安装 Karmada v1.18.0，纳管多个集群，并部署 Karmada Dashboard。

## 环境要求

- OpenShift 4.x 集群（已验证 4.20.24 / K8s v1.33.12）
- `oc` CLI 已登录且具有 cluster-admin 权限
- Helm v3+
- 集群具备动态存储（如 AWS gp3-csi）

## 架构

```
┌─────────────────────────────────────────────────┐
│  Host Cluster (cluster-9rlg7)                   │
│  ┌────────────────────────────────────────────┐  │
│  │  karmada-system namespace                  │  │
│  │  - etcd (StatefulSet, PVC)                 │  │
│  │  - karmada-apiserver (LoadBalancer:5443)    │  │
│  │  - karmada-controller-manager              │  │
│  │  - karmada-scheduler                       │  │
│  │  - karmada-webhook                         │  │
│  │  - karmada-aggregated-apiserver            │  │
│  │  - karmada-kube-controller-manager         │  │
│  │  - karmada-dashboard-api                   │  │
│  │  - karmada-dashboard-web (Route)           │  │
│  └────────────────────────────────────────────┘  │
└──────────────────┬──────────────────┬────────────┘
                   │ Push             │ Push
          ┌────────▼────────┐  ┌──────▼──────────┐
          │  cluster-9rlg7  │  │  cluster-5dll8  │
          │  (member)       │  │  (member)       │
          └─────────────────┘  └─────────────────┘
```

## 安装步骤

### 1. 添加 Helm Repo

```bash
helm repo add karmada-charts https://raw.githubusercontent.com/karmada-io/karmada/master/charts
helm repo update karmada-charts
```

### 2. 创建 Namespace 并授权 SCC

OpenShift 默认的 restricted SCC 不允许容器以 root 运行，Karmada 的 cfssl、etcd 等组件需要 anyuid 权限。**必须在 helm install 之前完成**，否则 pre-install hook 会因权限不足失败。

```bash
oc create namespace karmada-system
oc adm policy add-scc-to-user anyuid -z default -n karmada-system
oc adm policy add-scc-to-user anyuid -z karmada-hook-job -n karmada-system
```

### 3. Helm 安装 Karmada

- etcd 使用 PVC 存储（OpenShift 限制 hostPath）
- apiServer 使用 LoadBalancer 类型对外暴露

```bash
helm install karmada karmada-charts/karmada \
  --namespace karmada-system \
  --version v1.18.0 \
  --set etcd.internal.storageType=pvc \
  --set etcd.internal.pvc.storageClass=gp3-csi \
  --set etcd.internal.pvc.size=10Gi \
  --set apiServer.serviceType=LoadBalancer
```

### 4. 等待 Pod 就绪

```bash
oc get pods -n karmada-system -w
```

所有 Pod 应在约 60 秒内变为 Running。

### 5. 生成 Karmada Kubeconfig

获取 LoadBalancer 地址，生成可从外部访问的 kubeconfig：

```bash
# 等待 LB 分配 EXTERNAL-IP（AWS ELB DNS 传播可能需要 2-3 分钟）
oc get svc karmada-apiserver -n karmada-system

# 获取 LB 地址
LB_HOST=$(oc get svc karmada-apiserver -n karmada-system \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# 生成 kubeconfig，替换集群内部地址为 LB 地址
oc get secret karmada-kubeconfig -n karmada-system \
  -o jsonpath='{.data.kubeconfig}' | base64 -d \
  | sed "s|https://karmada-apiserver.karmada-system.svc.cluster.local:5443|https://${LB_HOST}:5443|g" \
  > ~/.kube/karmada.config

# 跳过 TLS 验证（LB 地址不在证书 SAN 中）
kubectl config set-cluster karmada-apiserver \
  --kubeconfig=$HOME/.kube/karmada.config \
  --insecure-skip-tls-verify=true

# 验证
kubectl --kubeconfig ~/.kube/karmada.config get ns
```

> **注意**: 如果 DNS 尚未传播，可先用 `nslookup <LB_HOST> 8.8.8.8` 获取 IP，用 IP 替代 hostname。

### 6. 安装 karmadactl CLI

```bash
# 自动安装（需要 sudo）
curl -s https://raw.githubusercontent.com/karmada-io/karmada/master/hack/install-cli.sh | sudo bash

# 或手动下载（macOS ARM64 示例）
VERSION="1.18.0"
curl -sL "https://github.com/karmada-io/karmada/releases/download/v${VERSION}/karmadactl-darwin-arm64.tgz" \
  -o /tmp/karmadactl.tgz
tar -xzf /tmp/karmadactl.tgz -C /tmp/
mkdir -p ~/bin && cp /tmp/karmadactl ~/bin/
export PATH=$HOME/bin:$PATH
```

### 7. 纳管成员集群（Push 模式）

```bash
# 加入 Host 集群自身
karmadactl --kubeconfig ~/.kube/karmada.config join <cluster-name> \
  --cluster-kubeconfig=$HOME/.kube/config \
  --cluster-context=$(oc config current-context)

# 加入其他集群（需先准备目标集群的 kubeconfig）
# 方式一：oc login 后使用 context
oc login https://api.<cluster-domain>:6443 -u admin -p <password>

# 方式二：使用 ServiceAccount Token 创建 kubeconfig
kubectl config set-cluster <cluster-name> \
  --server=https://api.<cluster-domain>:6443 \
  --insecure-skip-tls-verify=true \
  --kubeconfig=$HOME/.kube/<cluster-name>.config
kubectl config set-credentials <cluster-name>-admin \
  --token='<sa-token>' \
  --kubeconfig=$HOME/.kube/<cluster-name>.config
kubectl config set-context <cluster-name> \
  --cluster=<cluster-name> \
  --user=<cluster-name>-admin \
  --kubeconfig=$HOME/.kube/<cluster-name>.config
kubectl config use-context <cluster-name> \
  --kubeconfig=$HOME/.kube/<cluster-name>.config

# 加入
karmadactl --kubeconfig ~/.kube/karmada.config join <cluster-name> \
  --cluster-kubeconfig=$HOME/.kube/<cluster-name>.config \
  --cluster-context=<cluster-name>

# 验证
kubectl --kubeconfig ~/.kube/karmada.config get clusters
```

### 8. 安装 Karmada Dashboard

```bash
# 克隆 Dashboard 仓库
git clone https://github.com/karmada-io/dashboard.git /tmp/karmada-dashboard

# 创建 Dashboard 连接 Karmada API 所需的 kubeconfig Secret（使用集群内部地址）
oc get secret karmada-kubeconfig -n karmada-system \
  -o jsonpath='{.data.kubeconfig}' | base64 -d > /tmp/karmada-dashboard-kubeconfig

oc create secret generic karmada-dashboard-config \
  --from-file=karmada.config=/tmp/karmada-dashboard-kubeconfig \
  -n karmada-system

# 部署 Dashboard
oc apply -k /tmp/karmada-dashboard/artifacts/overlays/nodeport-mode

# 在 Karmada API 上创建 Dashboard RBAC
kubectl --kubeconfig ~/.kube/karmada.config apply \
  -f /tmp/karmada-dashboard/artifacts/dashboard/karmada-dashboard-sa.yaml
kubectl --kubeconfig ~/.kube/karmada.config apply \
  -f /tmp/karmada-dashboard/artifacts/dashboard/karmada-dashboard-clusterrolebinding.yaml
kubectl --kubeconfig ~/.kube/karmada.config apply \
  -f /tmp/karmada-dashboard/artifacts/dashboard/member-cluster-clusterrolebinding.yaml

# 创建 OpenShift Route 暴露 Dashboard Web UI
oc create route edge karmada-dashboard \
  --service=karmada-dashboard-web --port=8000 -n karmada-system

# 获取访问地址
oc get route karmada-dashboard -n karmada-system -o jsonpath='{.spec.host}'
```

### 9. 获取 Dashboard 登录 Token

```bash
kubectl --kubeconfig ~/.kube/karmada.config \
  -n karmada-system get secret karmada-dashboard-secret \
  -o jsonpath='{.data.token}' | base64 -d | tr -d '\n'
```

将输出的 token 粘贴到 Dashboard 登录页面即可。

## 验证与使用

### 多集群分发验证

创建一个 Deployment + Service，通过 PropagationPolicy 分发到所有成员集群：

```bash
# 1. 创建 Namespace 和应用（在 Karmada API 上）
cat <<'EOF' | kubectl --kubeconfig ~/.kube/karmada.config apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: karmada-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  namespace: karmada-demo
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo
  namespace: karmada-demo
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF

# 2. 在成员集群上为 demo namespace 授权 SCC（OpenShift 必须）
oc adm policy add-scc-to-user anyuid -z default -n karmada-demo
oc --kubeconfig ~/.kube/cluster-5dll8.config adm policy add-scc-to-user anyuid -z default -n karmada-demo

# 3. 创建 PropagationPolicy（按权重分发副本到两个集群）
cat <<'EOF' | kubectl --kubeconfig ~/.kube/karmada.config apply -f -
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-propagation
  namespace: karmada-demo
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx-demo
    - apiVersion: v1
      kind: Service
      name: nginx-demo
  placement:
    clusterAffinity:
      clusterNames:
        - cluster-9rlg7
        - cluster-5dll8
    replicaScheduling:
      replicaDivisionPreference: Weighted
      replicaSchedulingType: Divided
      weightPreference:
        staticWeightList:
          - targetCluster:
              clusterNames:
                - cluster-9rlg7
            weight: 1
          - targetCluster:
              clusterNames:
                - cluster-5dll8
            weight: 1
EOF

# 4. 验证分发结果
kubectl --kubeconfig ~/.kube/karmada.config get rb -n karmada-demo -o wide
oc get pods -n karmada-demo
kubectl --kubeconfig ~/.kube/cluster-5dll8.config get pods -n karmada-demo
```

预期结果：2 个副本按 1:1 权重分发，每个集群各运行 1 个 Pod。

### 可视化验证（浏览器直观对比）

部署一个显示所在集群信息的 Web 应用，通过两个集群各自的 Route 访问，直观看到同一份定义运行在不同集群上：

```bash
# 1. 创建应用（Duplicated 模式：每个集群完整运行一份副本）
cat <<'EOF' | kubectl --kubeconfig ~/.kube/karmada.config apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: karmada-verify
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-info
  namespace: karmada-verify
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-info
  template:
    metadata:
      labels:
        app: cluster-info
    spec:
      containers:
      - name: cluster-info
        image: python:3-alpine
        ports:
        - containerPort: 8080
        command: ["/bin/sh", "-c"]
        args:
        - |
          mkdir -p /tmp/www
          cat > /tmp/www/index.html <<HTMLEOF
          <html>
          <head><title>Karmada Verify</title></head>
          <body style="font-family:Arial; text-align:center; padding:40px; background:#f5f5f5;">
          <h1 style="color:#1565C0;">Karmada Multi-Cluster Verification</h1>
          <div style="background:white; border-radius:10px; padding:30px; margin:20px auto; max-width:500px; box-shadow:0 2px 8px rgba(0,0,0,0.1);">
          <h2 style="color:#2E7D32;">Pod Details</h2>
          <table style="margin:auto; text-align:left; font-size:16px; line-height:2;">
          <tr><td><b>Hostname:</b></td><td>${HOSTNAME}</td></tr>
          <tr><td><b>Node:</b></td><td>${NODE_NAME}</td></tr>
          <tr><td><b>Pod IP:</b></td><td>${POD_IP}</td></tr>
          <tr><td><b>Namespace:</b></td><td>${POD_NAMESPACE}</td></tr>
          </table>
          </div>
          <p style="color:#888; font-size:13px;">Deployed by Karmada PropagationPolicy (Duplicated Mode)</p>
          </body></html>
          HTMLEOF
          cd /tmp/www && python -m http.server 8080
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
---
apiVersion: v1
kind: Service
metadata:
  name: cluster-info
  namespace: karmada-verify
spec:
  selector:
    app: cluster-info
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: cluster-info-pp
  namespace: karmada-verify
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: cluster-info
    - apiVersion: v1
      kind: Service
      name: cluster-info
  placement:
    clusterAffinity:
      clusterNames:
        - cluster-9rlg7
        - cluster-5dll8
    replicaScheduling:
      replicaSchedulingType: Duplicated
EOF

# 2. 在两个集群上授权 SCC
oc adm policy add-scc-to-user anyuid -z default -n karmada-verify
oc --kubeconfig ~/.kube/<member-cluster>.config adm policy add-scc-to-user anyuid -z default -n karmada-verify

# 3. 在两个集群上分别创建 Route（注意 --port=8080）
oc create route edge cluster-info --service=cluster-info --port=8080 -n karmada-verify
oc --kubeconfig ~/.kube/<member-cluster>.config create route edge cluster-info --service=cluster-info --port=8080 -n karmada-verify

# 4. 获取两个 Route URL
oc get route cluster-info -n karmada-verify -o jsonpath='{.spec.host}'
oc --kubeconfig ~/.kube/<member-cluster>.config get route cluster-info -n karmada-verify -o jsonpath='{.spec.host}'
```

在浏览器中打开两个 URL，可以直观看到：
- **相同的应用** — 同一份 Deployment 定义
- **不同的集群** — Hostname、Node、Pod IP 各不相同

> **注意**: OpenShift 上使用 `python:3-alpine` 镜像替代 `nginx`，因为 nginx 官方镜像在 anyuid SCC 下存在文件系统权限问题。

清理：

```bash
kubectl --kubeconfig ~/.kube/karmada.config delete ns karmada-verify
```

### 常用操作

```bash
# 查看纳管集群状态
kubectl --kubeconfig ~/.kube/karmada.config get clusters

# 查看跨集群资源分发状态
kubectl --kubeconfig ~/.kube/karmada.config get rb -A -o wide

# 查看 PropagationPolicy
kubectl --kubeconfig ~/.kube/karmada.config get pp -A

# 查看 OverridePolicy
kubectl --kubeconfig ~/.kube/karmada.config get op -A

# 在 Karmada API 上创建资源（与普通 kubectl 用法一致）
kubectl --kubeconfig ~/.kube/karmada.config apply -f <resource.yaml>

# Dashboard 访问
# URL: https://$(oc get route karmada-dashboard -n karmada-system -o jsonpath='{.spec.host}')
# Token:
kubectl --kubeconfig ~/.kube/karmada.config \
  -n karmada-system get secret karmada-dashboard-secret \
  -o jsonpath='{.data.token}' | base64 -d | tr -d '\n'
```

### 清理验证资源

```bash
kubectl --kubeconfig ~/.kube/karmada.config delete ns karmada-demo
kubectl --kubeconfig ~/.kube/karmada.config delete ns karmada-verify
```

## 卸载

```bash
# 删除 Dashboard
oc delete route karmada-dashboard -n karmada-system
oc delete -k /tmp/karmada-dashboard/artifacts/overlays/nodeport-mode
oc delete secret karmada-dashboard-config -n karmada-system

# 移除成员集群
karmadactl --kubeconfig ~/.kube/karmada.config unjoin <cluster-name>

# 卸载 Karmada
helm uninstall karmada -n karmada-system

# 清理残余资源
oc delete sa karmada-hook-job -n karmada-system
oc delete clusterrole karmada-hook-job
oc delete clusterrolebinding karmada-hook-job
oc delete pvc --all -n karmada-system
oc delete ns karmada-system
```

## 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| pre-install hook 报 `Permission denied` | 未授权 SCC | `oc adm policy add-scc-to-user anyuid -z default -n karmada-system` |
| LB DNS 无法解析 | AWS ELB DNS 传播延迟 | 等待 2-3 分钟，或用 `nslookup <host> 8.8.8.8` 获取 IP |
| kubeconfig 报 TLS 证书错误 | LB 地址不在证书 SAN 中 | kubeconfig 中设置 `insecure-skip-tls-verify: true` |
| Dashboard token 报 "Invalid format" | token 复制不完整（含换行） | 用 `tr -d '\n'` 去除换行，或写入文件后 `pbcopy` 复制 |
| etcd PVC Pending | StorageClass 不存在 | 确认 `oc get sc` 中有对应的 StorageClass |
| nginx Pod CrashLoopBackOff | nginx 镜像在 anyuid SCC 下有文件权限问题 | 使用 `python:3-alpine` 或 OpenShift 兼容镜像替代 |
