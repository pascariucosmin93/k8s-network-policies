# Nginx IP Monitoring - Documentation

## Architecture

```
Internet → Cloudflare Tunnel → nginx (app1/app2) → logs JSON
                                                         ↓
                                                    Alloy (DaemonSet)
                                                         ↓
                                                    Loki (10.90.90.32:3100)
                                                         ↓
                                                    Grafana (10.90.90.101:8087)
```

---

## Components

### 1. Nginx applications

**Files:** `k8s/nginx-app1.yaml`, `k8s/nginx-app2.yaml`

Two nginx deployments (2 replicas each) in namespaces `app1` and `app2`.

**JSON log format** configured in `nginx.conf`:
```nginx
map $http_cf_connecting_ip $real_ip {
    ""      $remote_addr;       # fallback when it does not come through Cloudflare
    default $http_cf_connecting_ip;
}

log_format json_cf escape=json
    '{"time":"$time_iso8601",'
    '"ip":"$real_ip",'           # real IP (CF or direct)
    '"cf_ip":"$http_cf_connecting_ip",'
    '"country":"$http_cf_ipcountry",'
    '"ray":"$http_cf_ray",'
    '"method":"$request_method",'
    '"path":"$request_uri",'
    '"status":$status,'
    '"ua":"$http_user_agent"}';
```

`ip` field:
- Through **Cloudflare Tunnel** -> real client IP (from `CF-Connecting-IP`)
- **Direct** (without CF) -> `$remote_addr` (connected IP)

---

### 2. Cilium Gateway

**File:** `k8s/nginx-gateway.yaml`

Dedicated gateway in namespace `nginx-demo`, IP: `10.30.10.0`.

| Hostname | Namespace backend |
|----------|------------------|
| `app1.local` | `app1` |
| `app2.local` | `app2` |

Hostname-based routing via `HTTPRoute`.

---

### 3. Cloudflare Tunnel

The existing tunnel in namespace `clouadfare` (deployment `cloudflared`) exposes the applications publicly.

Configuration in Cloudflare Dashboard -> Tunnels -> Public Hostnames:

| Public Hostname | Service (inside cluster) |
|----------------|--------------------------|
| `app1.cosmin-lab.com` | `http://nginx-app1.app1.svc.cluster.local` |
| `app2.cosmin-lab.com` | `http://nginx-app2.app2.svc.cluster.local` |

> **Important:** It points directly to the ClusterIP service, not the Gateway, because cloudflared runs inside the cluster and can resolve internal DNS.

---

### 4. Grafana Alloy

**ConfigMap:** `monitoring/alloy` (config.alloy)

Collects logs from **all** pods in the cluster and sends them to Loki.

**Changes from the original config:**
- Corrected Loki URL: `10.13.13.30:3100` -> `10.90.90.32:3100`
- Added `loki.process` stage with JSON parsing to extract `ip` and `country` fields as Loki labels (useful for quick filtering in Grafana)

**Pipeline:**
```
loki.source.kubernetes → loki.process (JSON parse) → loki.write → Loki
loki.source.file (hubble) ─────────────────────────────────────→ Loki
```

---

### 5. CiliumNetworkPolicy - Alloy Egress

**Resource:** `monitoring/alloy-egress-policy`

**Resolved issue:** The original policy allowed Alloy to send to `10.13.13.30:3100` (old, unreachable Loki). Cilium was silently dropping all traffic to `10.90.90.32:3100`.

**Applied fix:**
```yaml
- toCIDR:
  - 10.90.90.32/32       # was 10.13.13.30/32
  toPorts:
  - ports:
    - port: "3100"
      protocol: TCP
```

---

### 6. Grafana Dashboard

**URL:** `http://10.90.90.101:8087/d/fe273626-9df0-47d3-9398-130cd2e6a1b8/nginx-ip-monitor`

**Panels:**
| Panel | Type | Query |
|-------|-----|-------|
| Log stream live | Logs | `{namespace=~"app1\|app2"} \| json \| ip != ""` |
| Top IPs | Bar chart | `topk(10, sum by (ip) (...))` |
| Top Countries | Pie chart | `sum by (country) (...)` |
| Requests over time | Timeseries | `sum by (namespace) (count_over_time(...))` |

Auto-refresh: 10s.

---

## Useful queries in Grafana Explore (Loki)

```logql
# All requests with real IP
{namespace=~"app1|app2"} | json | ip != ""

# Simple formatted output
{namespace=~"app1|app2"} | json | line_format "{{.ip}} [{{.country}}] {{.status}} {{.path}}"

# Top 10 IPs (last 24h)
topk(10, sum by (ip) (count_over_time({namespace=~"app1|app2"} | json | ip != "" [24h])))

# Filter by specific IP
{namespace=~"app1|app2"} | json | ip = "1.2.3.4"

# Requests per country
sum by (country) (count_over_time({namespace=~"app1|app2"} | json | country != "" [1h]))
```

---

## Cluster access

```bash
ssh root@10.90.90.9       # router/jump host
ssh devops@192.168.70.20  # k8s control plane node
```

Or in a single step:
```bash
ssh -J root@10.90.90.9 devops@192.168.70.20 "kubectl get nodes"
```
