# Argo CD Policies

Status: ingress hardening applied. Targeted egress hardening remains active only for `argocd-repo-server`. `argocd-server` egress hardening was rolled back after it broke application details in the UI.

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
- `argocd-application-controller` -> `argocd-repo-server:8081`
- `argocd-application-controller` -> `argocd-redis:6379`
- `argocd-application-controller` -> `10.96.0.1:443` for Kubernetes API

Applied in cluster:
- `15-argocd-repo-server-github-egress-cnp.yaml`

Still pending:
- application-controller egress hardening
- applicationset-controller egress audit
- notifications-controller egress audit
- future expansion if Argo CD starts using non-GitHub repos or OCI registries

Post-apply validation:
- Argo CD VIP `10.40.10.52` still returned `307` from inside the cluster
- `argocd-repo-server` still completed `git ls-remote` to GitHub successfully
- `argocd-repo-server` needed explicit Redis access for git ref cache after the initial FQDN-only policy

Prepared but rolled back:
- `04-argocd-server-default-deny-egress.yaml`
- `05-allow-argocd-server-dns-egress.yaml`
- `06-allow-argocd-server-redis-egress.yaml`
- `07-allow-argocd-server-repo-server-egress.yaml`
- `08-allow-argocd-server-dex-egress.yaml`
- `09-allow-argocd-server-kubernetes-api-egress.yaml`
- `10-argocd-application-controller-default-deny-egress.yaml`
- `11-allow-argocd-application-controller-dns-egress.yaml`
- `12-allow-argocd-application-controller-redis-egress.yaml`
- `13-allow-argocd-application-controller-repo-server-egress.yaml`
- `14-allow-argocd-application-controller-kubernetes-api-egress.yaml`

Rollback reason:
- `argocd-server` started timing out on Kubernetes API watches and application GETs, which surfaced in the UI as `Unable to load data: permission denied`
- `argocd-application-controller` started timing out on Kubernetes API watches during the first restrictive rollout
