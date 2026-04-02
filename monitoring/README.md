# Monitoring Policies

Status: ingress hardening applied, targeted egress hardening applied for `grafana`, `alloy`, and `prometheus-server`.

Applied in cluster:
- `00-default-deny-ingress.yaml`
- `01-allow-same-namespace-ingress.yaml`
- `02-allow-grafana-public-ingress.yaml`
- `03-allow-prometheus-public-ingress.yaml`

What is enforced:
- all pods in `monitoring` are ingress-denied by default
- same-namespace traffic is still allowed
- Grafana keeps public ingress on pod port `3000`
- Prometheus keeps public ingress on pod port `9090`

Validation performed:
- Grafana VIP `10.40.10.60:3000` returned `302` from inside the cluster
- Prometheus VIP `10.30.10.5` returned `302` from inside the cluster

Audited outbound dependencies:
- `grafana` -> `prometheus-server.monitoring.svc.cluster.local:80`
- `grafana` -> `10.13.13.30:3100` for Loki
- `alloy` -> `10.96.0.1:443` for Kubernetes API discovery
- `alloy` -> `10.13.13.30:3100` for Loki writes
- `prometheus-server` -> `10.96.0.1:443` for Kubernetes API
- `prometheus-server` -> pod CIDR `10.244.0.0/16` for pod and endpoint scrapes
- `prometheus-server` -> node IPs `192.168.70.0/24` for node-exporter
- `prometheus-server` -> `10.13.13.206:9090` for remote_write
- `prometheus-server` -> `monitoring` Alertmanager on `9093`

Applied in cluster:
- `04-grafana-default-deny-egress.yaml`
- `05-allow-grafana-dns-egress.yaml`
- `06-allow-grafana-prometheus-egress.yaml`
- `07-allow-grafana-loki-egress.yaml`
- `08-alloy-default-deny-egress.yaml`
- `09-allow-alloy-dns-egress.yaml`
- `10-allow-alloy-kubernetes-api-egress.yaml`
- `11-allow-alloy-loki-egress.yaml`
- `12-prometheus-default-deny-egress.yaml`
- `13-allow-prometheus-dns-egress.yaml`
- `14-allow-prometheus-kubernetes-api-egress.yaml`
- `15-allow-prometheus-monitoring-components-egress.yaml`
- `16-allow-prometheus-node-exporter-egress.yaml`
- `17-allow-prometheus-pod-scrape-egress.yaml`
- `18-allow-prometheus-remote-write-egress.yaml`

Still pending:
- scrape-path review for Prometheus targets
- future tightening of pod-scrape CIDR if target inventory becomes static

Post-apply validation:
- `grafana` pod still reached Prometheus and Loki successfully
- `alloy` pods remained `2/2 Running` on all nodes after policy apply
- `prometheus-server` had `0` active targets in `down` state before apply
- `prometheus-server` had `0` active targets in `down` state after apply
- `prometheus-server` remote write failure metric stayed empty before apply
- `prometheus-server` remote write failure metric stayed empty after apply
- `prometheus-server` produced no new `error` / `failed` / `timeout` log lines after apply
