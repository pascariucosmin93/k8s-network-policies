# Argo CD Policies

Status: ingress hardening applied, targeted egress hardening applied for `argocd-server`.

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

Audited outbound dependencies:
- `argocd-server` -> `argocd-redis:6379`
- `argocd-server` -> `argocd-repo-server:8081`
- `argocd-server` -> `argocd-dex-server:5556/5557/5558`
- `argocd-server` -> `10.96.0.1:443` for Kubernetes API
- `argocd-repo-server` -> external Git repositories on GitHub
- `argocd-application-controller` -> `argocd-repo-server`

Applied in cluster:
- `04-argocd-server-default-deny-egress.yaml`
- `05-allow-argocd-server-dns-egress.yaml`
- `06-allow-argocd-server-redis-egress.yaml`
- `07-allow-argocd-server-repo-server-egress.yaml`
- `08-allow-argocd-server-dex-egress.yaml`
- `09-allow-argocd-server-kubernetes-api-egress.yaml`

Still pending:
- repo-server egress audit
- controller egress audit
- Git / Helm / OCI outbound destination inventory
- safe `default-deny-egress` rollout

Post-apply validation:
- `argocd-server` still reached `argocd-repo-server`
- `argocd-server` still reached `argocd-dex-server`
- `argocd-server` still reached `kubernetes.default.svc`
- Argo CD VIP `10.40.10.52` still returned `307` from inside the cluster
- `argocd-server` produced no new `error` / `failed` / `timeout` log lines after apply
