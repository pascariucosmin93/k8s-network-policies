# Forgejo Policies

Status: ingress-only hardening applied.

Applied in cluster:
- `00-default-deny-ingress.yaml`
- `01-allow-public-web-and-ssh-ingress.yaml`

What is enforced:
- all pods in `forgejo` are ingress-denied by default
- Forgejo keeps public web ingress on pod port `3000`
- Forgejo keeps public SSH ingress on pod port `22` at the service layer

Validation performed:
- Forgejo VIP `10.30.10.85:3000` returned `200` from inside the cluster
- service still maps `22 -> 2222` and `3000 -> 3000`

Still pending:
- Forgejo core egress audit
- runner separation review
- runner egress restriction plan
