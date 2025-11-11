# Homelab K3s

Docs and configs for a 3‑node K3s cluster (Lenovo ThinkCentre Tiny) with Rancher, MetalLB, and a Prometheus + Grafana stack.

## Guides
- **Cluster setup (K3s + Rancher):** [`cluster-setup/README.md`](cluster-setup/README.md)
- **Monitoring (Prometheus + Grafana):** [`monitoring-stack/README.md`](monitoring-stack/README.md)

## Files
- MetalLB pool: [`cluster-setup/metallb-ip-pool.yaml`](cluster-setup/metallb-ip-pool.yaml)
- Systemd port‑forwards:
  - Rancher: [`cluster-setup/systemd/rancher-pf.service`](cluster-setup/systemd/rancher-pf.service)
  - Grafana: [`monitoring-stack/systemd/grafana-pf.service`](monitoring-stack/systemd/grafana-pf.service)
