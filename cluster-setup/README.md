# 3-Node K3s on ThinkCentre Tiny + Rancher

## Overview
- Nodes: `think1` (control plane), `think2` + `think3` (workers)
- Ubuntu Server, LAN; Tailscale for remote; MetalLB for LoadBalancer; Traefik (bundled); Rancher for cluster UI.

## Prerequisites
- Ubuntu Server 22.04 or 24.04 LTS on `think1`–`think3`, SSH enabled.
- Reserved/static LAN IPs for each node (e.g., `192.168.1.x`).
- A free MetalLB range chosen **outside** your DHCP scope (e.g., `192.168.1.20–192.168.1.29`).
- Your Mac with `kubectl` and `helm` installed (Homebrew: `brew install kubectl helm`).
- Tailscale installed and logged in on all nodes (for private remote access).
- `sudo` on all nodes; UFW either disabled or opened per the rules below.
- Time sync enabled: `sudo timedatectl set-ntp true`.

## 1) Install K3s
### Control plane (run on `think1`)
```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="server --disable servicelb --write-kubeconfig-mode=644 --tls-san <THINK1_LAN_IP>" \
  sh -

# grab join token for workers
sudo cat /var/lib/rancher/k3s/server/node-token
```
### Workers (run on `think2` and `think3`)
```bash
export K3S_URL="https://<THINK1_LAN_IP>:6443"
export K3S_TOKEN="<PASTE_TOKEN_FROM_THINK1>"
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="agent" sh -
```
### Kubectl from your laptop (Mac)
```bash
# copy kubeconfig
scp tyler@<THINK1_LAN_IP>:/etc/rancher/k3s/k3s.yaml ~/.kube/config
# point API server at think1’s LAN IP (not 127.0.0.1)
sed -i.bak 's/127\.0\.0\.1/<THINK1_LAN_IP>/' ~/.kube/config

kubectl get nodes -o wide
```
## 2) MetalLB (native mode)
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```
### Create an IP pool outside your DHCP scope
```bash
kubectl apply -f cluster-setup/metallb-ip-pool.yaml
kubectl -n metallb-system get ipaddresspools,l2advertisements
```
## 3) (Optional) Pin Traefik to a known LB IP
```bash
kubectl -n kube-system patch svc traefik -p \
'{"spec":{"type":"LoadBalancer","loadBalancerIP":"192.168.1.20"}}'
kubectl -n kube-system get svc traefik -o wide
```
## 4) Rancher (Helm)
```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
kubectl create ns cattle-system
helm install rancher rancher-latest/rancher -n cattle-system \
  --set bootstrapPassword="ChangeMe!" --set replicas=1 --set ingress.enabled=false
```
### Access Rancher (from your laptop)
```bash
kubectl -n cattle-system port-forward svc/rancher 4443:443
# open https://localhost:4443
```
## 5) Tailscale Serve (tailnet-only access via think1)
Make a persistent port-forward on think1, then expose with Tailscale Serve:
```bash
# copy unit file from repo, enable it
sudo cp cluster-setup/systemd/rancher-pf.service /etc/systemd/system/
sudo systemctl daemon-reload && sudo systemctl enable --now rancher-pf

# serve over tailnet (v1.90+)
sudo tailscale serve reset
sudo tailscale serve --bg --https=444 https+insecure://127.0.0.1:4443
# open: https://think1.<your-tailnet>.ts.net:444/
```
## 6) UFW allowlist (each node)
```bash
sudo ufw reset
sudo ufw default deny incoming && sudo ufw default allow outgoing
sudo ufw allow 22/tcp 6443/tcp 10250/tcp 8472/udp
sudo ufw allow in on tailscale0
sudo ufw allow in on cni0
sudo ufw allow in on flannel.1
sudo ufw allow from 10.42.0.0/16
sudo ufw allow from 10.43.0.0/16
sudo ufw enable
```
## Troubleshooting
- 404 via nip.io → you didn’t enable Traefik ingress; use port-forward or Tailscale.
- CreateContainerConfigError → missing Secret/ConfigMap or flaky kubelet; run `sudo systemctl restart k3s-agent` on the node hosting the pod.
- zsh “no matches found” on `hosts[0]` → escape brackets `hosts\[0\]` or use `--set-string`.