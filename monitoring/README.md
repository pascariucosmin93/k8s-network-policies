# Monitoring Policies

Status: ingress-only hardening planned.

Reason:
- this namespace is operationally sensitive
- Grafana, Prometheus, Alloy, and Loki flows need a dedicated egress audit before enforcing `default-deny-egress`
- safe first step is ingress-only hardening

Planned next step:
- apply `default-deny-ingress`
- keep same-namespace traffic open
- allow public ingress only to Grafana and Prometheus
- document required scrape paths
- document Grafana -> Prometheus/Loki dependencies
- document Alloy -> Loki dependency
- apply policies only after validating those flows
