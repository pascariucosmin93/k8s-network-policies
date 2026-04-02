# gaz Policies

Status: applied in cluster.

Applied baseline:
- `default-deny-ingress`
- `default-deny-egress`
- `allow-dns-egress`
- `allow-same-namespace-traffic`
- `allow-calculator-public-ingress`
- `allow-gateway-to-backends`
- `allow-calculator-db-egress`
- `allow-calculator-https-egress`

Validation after apply:
- VIP `10.30.10.6` returned `200`
- gateway VIP `10.30.10.8` returned `200`

Notes:
- this namespace is the most complex application namespace in the cluster
- same-namespace traffic is currently kept open to avoid breaking service-to-service calls
- gateway/backend connectivity is explicitly preserved for `cilium-envoy`
- `calculatorgaz` keeps database egress on `tcp/5432`
- `calculatorgaz` keeps outbound HTTPS for external integrations
- the namespace already had older policies before this rollout; they were not removed
