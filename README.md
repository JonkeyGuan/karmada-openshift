# Karmada on OpenShift Installation Guide

Deploy Karmada v1.18.0 on OpenShift 4.20 via Helm Chart, join multiple member clusters, and set up the Karmada Dashboard.

## Prerequisites

- OpenShift 4.x cluster (tested on 4.20.24 / K8s v1.33.12)
- `oc` CLI logged in with cluster-admin privileges
- Helm v3+
- Dynamic storage provisioner available (e.g., AWS gp3-csi)

## Architecture

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

## Installation

### 1. Add the Helm Repository

```bash
helm repo add karmada-charts https://raw.githubusercontent.com/karmada-io/karmada/master/charts
helm repo update karmada-charts
```

### 2. Create the Namespace and Grant SCC Permissions

OpenShift's default restricted SCC prevents containers from running as root. Karmada components (cfssl, etcd, etc.) require the `anyuid` SCC. **This must be done before `helm install`**, otherwise the pre-install hook will fail with `Permission denied`.

```bash
oc create namespace karmada-system
oc adm policy add-scc-to-user anyuid -z default -n karmada-system
oc adm policy add-scc-to-user anyuid -z karmada-hook-job -n karmada-system
```

### 3. Install Karmada via Helm

- etcd uses PVC storage (OpenShift restricts hostPath)
- apiServer uses LoadBalancer service type for external access

```bash
helm install karmada karmada-charts/karmada \
  --namespace karmada-system \
  --version v1.18.0 \
  --set etcd.internal.storageType=pvc \
  --set etcd.internal.pvc.storageClass=gp3-csi \
  --set etcd.internal.pvc.size=10Gi \
  --set apiServer.serviceType=LoadBalancer
```

### 4. Wait for Pods to Be Ready

```bash
oc get pods -n karmada-system -w
```

All pods should reach Running status within approximately 60 seconds.

### 5. Generate the Karmada Kubeconfig

Retrieve the LoadBalancer address and generate a kubeconfig for external access:

```bash
# Wait for the LB to be assigned an EXTERNAL-IP (AWS ELB DNS propagation may take 2-3 minutes)
oc get svc karmada-apiserver -n karmada-system

# Get the LB address
LB_HOST=$(oc get svc karmada-apiserver -n karmada-system \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Generate kubeconfig, replacing the in-cluster address with the LB address
oc get secret karmada-kubeconfig -n karmada-system \
  -o jsonpath='{.data.kubeconfig}' | base64 -d \
  | sed "s|https://karmada-apiserver.karmada-system.svc.cluster.local:5443|https://${LB_HOST}:5443|g" \
  > ~/.kube/karmada.config

# Skip TLS verification (LB address is not in the certificate SANs)
kubectl config set-cluster karmada-apiserver \
  --kubeconfig=$HOME/.kube/karmada.config \
  --insecure-skip-tls-verify=true

# Verify
kubectl --kubeconfig ~/.kube/karmada.config get ns
```

> **Note**: If DNS has not propagated yet, use `nslookup <LB_HOST> 8.8.8.8` to get the IP and substitute it for the hostname.

### 6. Install the karmadactl CLI

```bash
# Automated install (requires sudo)
curl -s https://raw.githubusercontent.com/karmada-io/karmada/master/hack/install-cli.sh | sudo bash

# Or manual download (macOS ARM64 example)
VERSION="1.18.0"
curl -sL "https://github.com/karmada-io/karmada/releases/download/v${VERSION}/karmadactl-darwin-arm64.tgz" \
  -o /tmp/karmadactl.tgz
tar -xzf /tmp/karmadactl.tgz -C /tmp/
mkdir -p ~/bin && cp /tmp/karmadactl ~/bin/
export PATH=$HOME/bin:$PATH
```

### 7. Join Member Clusters (Push Mode)

```bash
# Join the host cluster itself
karmadactl --kubeconfig ~/.kube/karmada.config join <cluster-name> \
  --cluster-kubeconfig=$HOME/.kube/config \
  --cluster-context=$(oc config current-context)

# Join another cluster (prepare its kubeconfig first)
# Option A: Log in via oc and use the context
oc login https://api.<cluster-domain>:6443 -u admin -p <password>

# Option B: Create a kubeconfig using a ServiceAccount token
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

# Join
karmadactl --kubeconfig ~/.kube/karmada.config join <cluster-name> \
  --cluster-kubeconfig=$HOME/.kube/<cluster-name>.config \
  --cluster-context=<cluster-name>

# Verify
kubectl --kubeconfig ~/.kube/karmada.config get clusters
```

### 8. Install the Karmada Dashboard

```bash
# Clone the dashboard repository
git clone https://github.com/karmada-io/dashboard.git /tmp/karmada-dashboard

# Create the kubeconfig secret for the dashboard to connect to the Karmada API (using the in-cluster address)
oc get secret karmada-kubeconfig -n karmada-system \
  -o jsonpath='{.data.kubeconfig}' | base64 -d > /tmp/karmada-dashboard-kubeconfig

oc create secret generic karmada-dashboard-config \
  --from-file=karmada.config=/tmp/karmada-dashboard-kubeconfig \
  -n karmada-system

# Deploy the dashboard
oc apply -k /tmp/karmada-dashboard/artifacts/overlays/nodeport-mode

# Create dashboard RBAC on the Karmada API
kubectl --kubeconfig ~/.kube/karmada.config apply \
  -f /tmp/karmada-dashboard/artifacts/dashboard/karmada-dashboard-sa.yaml
kubectl --kubeconfig ~/.kube/karmada.config apply \
  -f /tmp/karmada-dashboard/artifacts/dashboard/karmada-dashboard-clusterrolebinding.yaml
kubectl --kubeconfig ~/.kube/karmada.config apply \
  -f /tmp/karmada-dashboard/artifacts/dashboard/member-cluster-clusterrolebinding.yaml

# Create an OpenShift Route to expose the dashboard web UI
oc create route edge karmada-dashboard \
  --service=karmada-dashboard-web --port=8000 -n karmada-system

# Get the dashboard URL
oc get route karmada-dashboard -n karmada-system -o jsonpath='{.spec.host}'
```

### 9. Get the Dashboard Login Token

```bash
kubectl --kubeconfig ~/.kube/karmada.config \
  -n karmada-system get secret karmada-dashboard-secret \
  -o jsonpath='{.data.token}' | base64 -d | tr -d '\n'
```

Paste the token into the dashboard login page to sign in.

## Verification and Usage

### Multi-Cluster Distribution Test

Create a Deployment + Service and distribute them to all member clusters via a PropagationPolicy:

```bash
# 1. Create the namespace and application on the Karmada API
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

# 2. Grant SCC on each member cluster for the demo namespace (required on OpenShift)
oc adm policy add-scc-to-user anyuid -z default -n karmada-demo
oc --kubeconfig ~/.kube/<member-cluster>.config adm policy add-scc-to-user anyuid -z default -n karmada-demo

# 3. Create a PropagationPolicy (weighted replica distribution across clusters)
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

# 4. Verify the distribution
kubectl --kubeconfig ~/.kube/karmada.config get rb -n karmada-demo -o wide
oc get pods -n karmada-demo
kubectl --kubeconfig ~/.kube/<member-cluster>.config get pods -n karmada-demo
```

Expected result: 2 replicas distributed 1:1 by weight, one pod running on each cluster.

### Visual Verification (Browser-Based Comparison)

Deploy a web application that displays its cluster information. Access it via Routes on each cluster to visually confirm that the same definition runs on different clusters:

```bash
# 1. Create the application (Duplicated mode: full replica on every cluster)
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

# 2. Grant SCC on both clusters
oc adm policy add-scc-to-user anyuid -z default -n karmada-verify
oc --kubeconfig ~/.kube/<member-cluster>.config adm policy add-scc-to-user anyuid -z default -n karmada-verify

# 3. Create a Route on each cluster (note: --port=8080)
oc create route edge cluster-info --service=cluster-info --port=8080 -n karmada-verify
oc --kubeconfig ~/.kube/<member-cluster>.config create route edge cluster-info --service=cluster-info --port=8080 -n karmada-verify

# 4. Get the Route URLs
oc get route cluster-info -n karmada-verify -o jsonpath='{.spec.host}'
oc --kubeconfig ~/.kube/<member-cluster>.config get route cluster-info -n karmada-verify -o jsonpath='{.spec.host}'
```

Open both URLs in a browser to see:
- **Same application** -- identical Deployment definition
- **Different clusters** -- distinct Hostname, Node, and Pod IP values

> **Note**: Use `python:3-alpine` instead of the official `nginx` image on OpenShift, as the nginx image has filesystem permission issues under the anyuid SCC.

Cleanup:

```bash
kubectl --kubeconfig ~/.kube/karmada.config delete ns karmada-verify
```

### Common Operations

```bash
# List managed clusters
kubectl --kubeconfig ~/.kube/karmada.config get clusters

# View cross-cluster resource binding status
kubectl --kubeconfig ~/.kube/karmada.config get rb -A -o wide

# List PropagationPolicies
kubectl --kubeconfig ~/.kube/karmada.config get pp -A

# List OverridePolicies
kubectl --kubeconfig ~/.kube/karmada.config get op -A

# Create resources on the Karmada API (same as standard kubectl usage)
kubectl --kubeconfig ~/.kube/karmada.config apply -f <resource.yaml>

# Access the dashboard
# URL: https://$(oc get route karmada-dashboard -n karmada-system -o jsonpath='{.spec.host}')
# Token:
kubectl --kubeconfig ~/.kube/karmada.config \
  -n karmada-system get secret karmada-dashboard-secret \
  -o jsonpath='{.data.token}' | base64 -d | tr -d '\n'
```

### Clean Up Verification Resources

```bash
kubectl --kubeconfig ~/.kube/karmada.config delete ns karmada-demo
kubectl --kubeconfig ~/.kube/karmada.config delete ns karmada-verify
```

## Uninstallation

```bash
# Remove the dashboard
oc delete route karmada-dashboard -n karmada-system
oc delete -k /tmp/karmada-dashboard/artifacts/overlays/nodeport-mode
oc delete secret karmada-dashboard-config -n karmada-system

# Unjoin member clusters
karmadactl --kubeconfig ~/.kube/karmada.config unjoin <cluster-name>

# Uninstall Karmada
helm uninstall karmada -n karmada-system

# Clean up remaining resources
oc delete sa karmada-hook-job -n karmada-system
oc delete clusterrole karmada-hook-job
oc delete clusterrolebinding karmada-hook-job
oc delete pvc --all -n karmada-system
oc delete ns karmada-system
```

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| Pre-install hook reports `Permission denied` | SCC not granted | `oc adm policy add-scc-to-user anyuid -z default -n karmada-system` |
| LB DNS does not resolve | AWS ELB DNS propagation delay | Wait 2-3 minutes, or use `nslookup <host> 8.8.8.8` to get the IP |
| Kubeconfig reports TLS certificate error | LB address not in certificate SANs | Set `insecure-skip-tls-verify: true` in the kubeconfig |
| Dashboard token shows "Invalid format" | Token copied with newlines | Use `tr -d '\n'` to strip newlines, or save to a file and copy with `pbcopy` |
| etcd PVC stuck in Pending | StorageClass does not exist | Verify with `oc get sc` that the specified StorageClass is available |
| nginx pod CrashLoopBackOff | nginx image has filesystem permission issues under anyuid SCC | Use `python:3-alpine` or an OpenShift-compatible image instead |
