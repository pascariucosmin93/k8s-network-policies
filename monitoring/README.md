# Monitoring Policies

Status: ingress-only hardening applied.

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

Still pending:
- egress audit for Grafana -> Prometheus / Loki
- egress audit for Alloy -> Loki
- scrape-path review for Prometheus targets
- safe `default-deny-egress` rollout
