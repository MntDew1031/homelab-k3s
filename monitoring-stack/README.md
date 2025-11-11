# Monitoring K3s with kube-prometheus-stack (Prometheus + Grafana)

## Prerequisites
- A healthy K3s cluster and `kubectl` context pointing at it
- Helm v3 installed on your Mac (`brew install helm`)
- Namespace `monitoring` will be created if it doesn't exist
- Optional: Tailscale installed on `think1` if you want tailnet-only access via Serve
- `sudo` on nodes; UFW disabled or opened per rules in your cluster README
- Time sync enabled on nodes: `sudo timedatectl set-ntp true`

## 1) Setup
```bash
kubectl create ns monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
## 2) Install with a minimal, stable values file
```bash
helm upgrade --install prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring -f monitoring-stack/values-min.yaml --reset-values
kubectl -n monitoring get pods -o wide
```
Expected healthy:
- `prometheus-stack-grafana-*` → **Running**
- `prometheus-stack-kube-prom-operator-*` → **Running**
- `prometheus-prometheus-stack-prometheus-0` → **2/2 Running**
- `alertmanager-prometheus-stack-alertmanager-0` → **2/2 Running**

## 3) Access Grafana

Port-forward (reliable, anywhere):
```bash
kubectl -n monitoring port-forward svc/prometheus-stack-grafana 3000:80
# http://localhost:3000  (admin / ChangeMe123!)
```
Tailscale Serve (on think1):
```bash
# copy unit, enable forward
sudo cp monitoring-stack/systemd/grafana-pf.service /etc/systemd/system/
sudo systemctl daemon-reload && sudo systemctl enable --now grafana-pf

# serve 443 -> localhost:3000
sudo tailscale serve reset
sudo tailscale serve --bg --https=443 http://127.0.0.1:3000
# open: https://think1.<your-tailnet>.ts.net/
```
(Optional) Traefik Ingress (LAN/public):
```bash
TRAEFIK_IP=$(kubectl -n kube-system get svc traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
helm upgrade prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring \
  --set grafana.service.type=ClusterIP \
  --set grafana.ingress.enabled=true \
  --set grafana.ingress.ingressClassName=traefik \
  --set-string grafana.ingress.hosts\[0\]="grafana.${TRAEFIK_IP}.nip.io"
```
## 4) After it’s stable (optional features)
```bash
helm upgrade prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring \
  --set grafana.sidecar.dashboards.enabled=true \
  --set grafana.sidecar.datasources.enabled=true
```

```bash
# (later) re-enable operator webhooks if you want:
helm upgrade prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring \
  --set prometheusOperator.admissionWebhooks.enabled=true
```
## 5) Troubleshooting
- **Grafana `CreateContainerConfigError: secret not found`** → create admin secret and restart:
```bash
kubectl -n monitoring create secret generic prometheus-stack-grafana \
  --from-literal=admin-user='admin' \
  --from-literal=admin-password='ChangeMe123!' \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl -n monitoring rollout restart deploy/prometheus-stack-grafana
```
- **Operator stuck on tls-secret** → install with `prometheusOperator.admissionWebhooks.enabled=false`.
- **zsh `no matches found` on `hosts[0]`** → escape with `hosts\[0\]` or use `--set-string`.
---

## `monitoring-stack/values-min.yaml`
```yaml
grafana:
  adminUser: admin
  # Do NOT commit your real password; override with a Secret in prod.
  adminPassword: "ChangeMe123!"
  persistence:
    enabled: false
  service:
    type: ClusterIP
  ingress:
    enabled: false
  sidecar:
    dashboards:
      enabled: false
    datasources:
      enabled: false

prometheusOperator:
  admissionWebhooks:
    enabled: false

prometheus:
  prometheusSpec:
    storageSpec: {}
    retention: 10d

alertmanager:
  alertmanagerSpec: {}
```