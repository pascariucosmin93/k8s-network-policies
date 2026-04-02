# employee-leave Policies

Status: applied in cluster.

Applied baseline:
- `default-deny-ingress`
- `default-deny-egress`
- `allow-dns-egress`
- `allow-frontend-http-ingress`
- `allow-frontend-to-backend`
- `allow-backend-postgres-egress`

Validation after apply:
- VIP `10.30.10.9` returned `200`

Notes:
- frontend remains publicly reachable on its app port
- backend accepts traffic only from frontend
- backend keeps outbound access for PostgreSQL on `tcp/5432`
- the namespace already had older policies before this rollout; they were not removed
