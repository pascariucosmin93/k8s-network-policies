# Forgejo Policies

Status: ingress hardening applied. Core Forgejo egress is now restricted to DNS and PostgreSQL only.

Applied in cluster:
- `00-default-deny-ingress.yaml`
- `01-allow-public-web-and-ssh-ingress.yaml`
- `02-default-deny-egress.yaml`
- `03-allow-dns-egress.yaml`
- `04-allow-postgres-egress.yaml`

What is enforced:
- all pods in `forgejo` are ingress-denied by default
- Forgejo keeps public web ingress on pod port `3000`
- Forgejo keeps public SSH ingress on service port `22`, which maps to pod port `2222`
- Forgejo pod egress is denied by default
- Forgejo may resolve DNS through `kube-dns`
- Forgejo may reach PostgreSQL only in namespace `postgress` on `5432`

Validation performed:
- Forgejo VIP `10.30.10.85:3000` returned `200` from inside the cluster
- service still exposes `22` and `3000`
- Forgejo service maps `22 -> 2222` and `3000 -> 3000`
- the Forgejo pod depends on `postgres.postgress.svc.cluster.local:5432`
- no pod restart or PVC change is required to apply these policies

Still pending:
- runner separation review
- runner egress restriction plan
- optional review for outbound webhooks / SMTP / package mirrors if you rely on them from Forgejo itself
