# cv Policies

Status: applied in cluster.

Applied baseline:
- `default-deny-ingress`
- `default-deny-egress`
- `allow-dns-egress`
- `allow-http-ingress`

Validation after apply:
- VIP `10.30.10.3` returned `200`

Notes:
- `cv` is exposed directly through its own `LoadBalancer` service
- ingress is currently kept open on `tcp/80` to avoid downtime
- main hardening gain here is blocking unexpected egress and non-HTTP ingress
