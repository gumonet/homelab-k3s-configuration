# Install k3s

This script installs k3s, a lightweight Kubernetes distribution.

```bash
curl -sfL https://get.k3s.io | sh -
```

## Get kubeconfig

```bash
sudo cat /etc/rancher/k3s/k3s.yaml
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

# Install control plane with telemetry enabled and metrics enabled
helm install istiod istio/istiod \
  --namespace istio-system \
  --version 1.26.2 \
  --set global.meshID="gumonet" \
  --set global.multiCluster.clusterName="gumonet" \
  --set global.network="gumonet" \
  --set meshConfig.enablePrometheusMerge=true \
  --set meshConfig.defaultConfig.tracing.sampling=100.0 \
  --set meshConfig.defaultConfig.tracing.zipkin.address="192.16.0.20:9411" \
  --set meshConfig.defaultConfig.tracing.customTags.request_id.header.name="x-request-id" \
  --set telemetry.enabled=true \
  --set telemetry.v2.enabled=true \
  --set telemetry.v2.prometheus.enabled=true \
  --set serviceMonitor.enabled=true \
  --set serviceMonitor.selectorNilUsesHelmValues=true \
  --set serviceMonitor.additionalLabels.release=prometheus \
  --set values.cni.enabled=true \
  --set values.global.istioNamespace=istio-system
```

### Solo actualizar istiod

```bash
helm upgrade istiod istio/istiod \
  --namespace istio-system \
  --version 1.26.2 \
  --set global.meshID="gumonet" \
  --set global.multiCluster.clusterName="gumonet" \
  --set global.network="gumonet" \
  --set meshConfig.enablePrometheusMerge=true \
  --set meshConfig.defaultConfig.tracing.sampling=100.0 \
  --set meshConfig.defaultConfig.tracing.zipkin.address="192.16.0.20:9411" \
  --set meshConfig.defaultConfig.tracing.customTags.request_id.header.name="x-request-id" \
  --set telemetry.enabled=true \
  --set telemetry.v2.enabled=true \
  --set telemetry.v2.prometheus.enabled=true \
  --set serviceMonitor.enabled=true \
  --set serviceMonitor.selectorNilUsesHelmValues=true \
  --set serviceMonitor.additionalLabels.release=prometheus \
  --set values.cni.enabled=false \
  --set values.global.istioNamespace=istio-system
```

## Install istio cni

```bash
helm upgrade cni istio/cni \
  --namespace istio-system \
  --set cni.enabled=true \
  --set cni.logLevel=debug \
  --set cni.excludeNamespaces="{kube-system,istio-system}"

```

# Install argocd in cluster using charts

````bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd

#Create argocd password

#Install argocd
helm install argocd argo/argo-cd \
  --namespace argocd \
  --set controller.metrics.enabled=true \
  --set controller.metrics.serviceMonitor.enabled=false \
  --set server.metrics.enabled=true \
  --set server.metrics.serviceMonitor.enabled=false \
  --set repoServer.metrics.enabled=true \
  --set repoServer.metrics.serviceMonitor.enabled=false \
  --set configs.cm.application.dependencies.enabled=true \
  --set configs.secret.argocdServerAdminPassword='temppwd'

## Only update argocd
  helm upgrade argocd argo/argo-cd \
  --namespace argocd \
  --set controller.metrics.enabled=true \
  --set controller.metrics.serviceMonitor.enabled=false \
  --set server.metrics.enabled=true \
  --set server.metrics.serviceMonitor.enabled=false \
  --set repoServer.metrics.enabled=true \
  --set configs.cm.application.dependencies.enabled=true \
  --set repoServer.metrics.serviceMonitor.enabled=false

# Genera la contraseña en bcrypt
export ARGOCD_ADMIN_PASSWORD=$(htpasswd -nbBC 10 "" "MyNewSecurePassword" | tr -d ':\n' | sed 's/^\\$2y/\\$2a/')

## Set token for slack notifications
 1. Create a new Slack application in your Slack workspace, follow this [guide](https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/services/slack/).
 2. Upload the slack token to new secret in ArgoCD namespace:
```bash
  kubectl create secret generic argocd-notifications-secret \
  --from-literal=slack-token=REAL_TOKEN \
  -n argocd
````

Replace `REAL_TOKEN` with the token you obtained from the Slack application.

# Parchea el Secret

kubectl -n argocd patch secret argocd-secret \
 -p "{\"stringData\": {\"admin.password\": \"$ARGOCD_ADMIN_PASSWORD\"}}"

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

````

# Configure CDRS for prometheus operator

```bash
kubectl apply --server-side -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/example/prometheus-operator-crd/monitoring.coreos.com_prometheuses.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/example/prometheus-operator-crd/monitoring.coreos.com_servicemonitors.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/example/prometheus-operator-crd/monitoring.coreos.com_podmonitors.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/example/prometheus-operator-crd/monitoring.coreos.com_prometheusrules.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/example/prometheus-operator-crd/monitoring.coreos.com_alertmanagers.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/example/prometheus-operator-crd/monitoring.coreos.com_thanosrulers.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/example/prometheus-operator-crd/monitoring.coreos.com_alertmanagerconfigs.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/example/prometheus-operator-crd/monitoring.coreos.com_probes.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/example/prometheus-operator-crd/monitoring.coreos.com_prometheusagents.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/example/prometheus-operator-crd/monitoring.coreos.com_scrapeconfigs.yaml
````

# Prometheus Blackbox

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

## Set your custom registry in k3s

```bash
sudo nano /etc/rancher/k3s/registries.yaml
```

Add the following content:

```yaml
mirrors:
  "192.16.0.20:5000":
    endpoint:
      - "http://192.16.0.20:5000"
```

Restart k3s service to apply changes:

```bash
sudo systemctl restart k3s
```

Test logs pods:

```bash
kubectl run testlogger --image=busybox -n default -it --restart=Never -- sh
# dentro del pod
while true; do echo "hola $(date)"; sleep 3; done
while true; do curl api.mi-servicio.svc.cluster.local/health && echo "" ; sleep 3; done
```

kubectl run testlogger --image=nicolaka/netshoot:latest -n default -it --restart=Never -- sh

```

```
