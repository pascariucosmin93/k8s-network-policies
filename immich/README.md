# immich Policies

Status: applied in cluster.

Applied baseline:
- `default-deny-ingress`
- `default-deny-egress`
- `allow-dns-egress`
- `allow-server-http-ingress`
- `allow-server-egress`
- `allow-postgres-from-server`
- `allow-redis-from-server`

Validation after apply:
- VIP `10.30.10.7` returned `200`

Notes:
- `immich-server` remains publicly reachable on its HTTP port
- internal traffic is narrowed to:
  - `immich-server` -> `immich-postgres`
  - `immich-server` -> `immich-redis`
