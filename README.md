# Install k3s

This script installs k3s, a lightweight Kubernetes distribution.

```bash
curl -sfL https://get.k3s.io | sh -
```

# Install istio in cluster using charts

```bash
#CInstall chart base
kubectl create namespace istio-system
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

helm install istio-base istio/base \
  --namespace istio-system \
  --version 1.26.2 \
  --set global.istiod.enableAnalysis=false \
  --set global.configValidation=true \
  --set global.externalIstiod=false \
  --set global.base.enableIstioConfigCRDs=true

# Install control plane
helm install istiod istio/istiod \
  --namespace istio-system \
  --version 1.26.2 \
  --set global.meshID="gumonet" \
  --set global.multiCluster.clusterName="gumonet" \
  --set global.network="gumonet" \
  --set meshConfig.defaultConfig.tracing.sampling=100.0 \
  --set meshConfig.defaultConfig.tracing.zipkin.address="tempo.monitoring.svc.cluster.local:9411" \
  --set meshConfig.defaultConfig.tracing.customTags.request_id.header.name="x-request-id"
```

# Install argocd in cluster using charts

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd

#Create argocd password
# Requiere apache2-utils en Linux/macOS
argopwd=$(htpasswd -nbBC 10 "" 'TuPasswordSegura' | tr -d ':\n')

#Install argocd
helm install argocd argo/argo-cd \
  --namespace argocd \
  --set configs.secret.argocdServerAdminPassword='$argopwd'

#Update password if is necessary
helm upgrade argocd argo/argo-cd \
  --namespace argocd \
  --reuse-values \
  --set configs.secret.argocdServerAdminPassword='<bcrypt-password>'

# Update argocd password using kubectl
kubectl -n argocd patch secret argocd-secret \
  --type merge \
  -p '{"stringData": {
    "admin.password": "$2y$10$gKQfWrNbPkJvw8BQqy6et.ml83vcrzi380l1pGQEz7U96pH0w2tHS",
    "admin.passwordMtime": "'$(date +%FT%T%Z)'"
  }}'
```
