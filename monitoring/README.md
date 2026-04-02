# Monitoring Policies

Status: not applied yet.

Reason:
- this namespace is operationally sensitive
- Grafana, Prometheus, Alloy, and Loki flows need a dedicated egress audit before enforcing `default-deny-egress`

Planned next step:
- document required scrape paths
- document Grafana -> Prometheus/Loki dependencies
- document Alloy -> Loki dependency
- apply policies only after validating those flows
