# todo-app Policies

Status: applied in cluster.

Applied baseline:
- `default-deny-ingress`
- `default-deny-egress`
- `allow-dns-egress`
- `allow-frontend-http-ingress`
- `allow-frontend-to-backend`
- `allow-frontend-egress-to-backend`
- `allow-backend-postgres-egress`

Validation after apply:
- VIP `10.30.10.10` returned `200`

Notes:
- frontend remains publicly reachable on HTTP
- frontend/backend traffic is now explicit
- backend keeps outbound access for PostgreSQL on `tcp/5432`
