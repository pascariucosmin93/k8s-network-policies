# Argo CD Policies

Status: ingress-only hardening applied.

Applied in cluster:
- `00-default-deny-ingress.yaml`
- `01-allow-same-namespace-ingress.yaml`
- `02-allow-argocd-server-public-ingress.yaml`
- `03-allow-monitoring-scrape-ingress.yaml`

What is enforced:
- all pods in `argocd` are ingress-denied by default
- same-namespace traffic is still allowed
- public ingress remains open only to `argocd-server` on pod port `8080`
- `monitoring` namespace may still scrape Argo CD metrics ports

Validation performed:
- Argo CD VIP `10.40.10.52` returned `307` from inside the cluster

Still pending:
- repo-server egress audit
- controller egress audit
- Git / Helm / OCI outbound destination inventory
- safe `default-deny-egress` rollout
