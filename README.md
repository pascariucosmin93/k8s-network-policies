# k8s-network-policies

Kubernetes network policies for a self-hosted homelab cluster running Cilium as CNI with kube-proxy replacement and BGP control plane.

Each namespace follows a default-deny model with explicit allow rules for each required traffic flow.

## Cluster Context

- **CNI**: Cilium (kube-proxy replacement mode, `bpf-lb-sock: false`)
- **Routing**: BGP via FRR, LoadBalancer IPs announced to upstream routers
- **Observability**: Hubble + Loki for flow visibility and policy validation

> **Cilium note:** Standard `NetworkPolicy` with `ipBlock` does not work for Kubernetes node IPs or the API server when Cilium runs in kube-proxy replacement mode — DNAT happens before policy evaluation. Node-level and API server access uses `CiliumNetworkPolicy` with `toEntities` instead.

## Structure

```
.
├── argocd/
├── cv/
├── employee-leave/
├── forgejo/
├── forgejo-runner/
├── gaz/
├── immich/
├── monitoring/
└── todo-app/
```

## Namespace Coverage

| Namespace | Default Deny Ingress | Default Deny Egress | Notes |
|-----------|:--------------------:|:-------------------:|-------|
| `cv` | ✓ | ✓ | HTTP ingress from LoadBalancer only |
| `employee-leave` | ✓ | ✓ | Frontend → backend → postgres |
| `todo-app` | ✓ | ✓ | Frontend → backend → postgres |
| `gaz` | ✓ | ✓ | Microservices with gateway pattern |
| `immich` | ✓ | ✓ | Server → postgres + redis |
| `monitoring` | ✓ | ✓ | Grafana, Prometheus, Alloy, Loki |
| `argocd` | ✓ | ✓ | Server + application-controller egress |
| `forgejo` | ✓ | ✓ | Web + SSH ingress, postgres egress |
| `forgejo-runner` | ✓ | open | CI runners require broad outbound access |

## Policy Model

Every namespace starts with:

```
00-default-deny-ingress     # block all inbound traffic
01-default-deny-egress      # block all outbound traffic
02-allow-dns-egress         # allow kube-dns only
...                         # explicit allow rules per dependency
```

Files are numbered to make the order and intent immediately readable.

## Monitoring Namespace — Cilium-Specific Fixes

Standard `ipBlock` rules for Kubernetes API access silently fail with Cilium kube-proxy replacement because the packet-level DNAT translates the ClusterIP to the real node IP before policy evaluation.

Fixed by using `CiliumNetworkPolicy` with `toEntities`:

```yaml
# works with Cilium kube-proxy replacement
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
spec:
  egress:
    - toEntities:
        - kube-apiserver   # for API server access (Alloy, Prometheus)
    - toEntities:
        - remote-node      # for node-level scraping (Prometheus → node-exporter, cilium-agent)
        - host
```

## Apply

Apply all policies for a namespace:

```bash
kubectl apply -f monitoring/
kubectl apply -f cv/
kubectl apply -f gaz/
```

Apply everything at once:

```bash
for ns in cv employee-leave todo-app gaz immich monitoring argocd forgejo forgejo-runner; do
  kubectl apply -f $ns/
done
```

## Validate

After applying, verify with Hubble:

```bash
hubble observe --namespace cv --verdict DROPPED
hubble observe --namespace monitoring --verdict DROPPED
```

Or check Prometheus targets are still up:

```bash
kubectl exec -n monitoring deploy/prometheus-server -c prometheus-server -- \
  wget -qO- http://localhost:9090/api/v1/targets | \
  python3 -c "import json,sys; [print(t['labels']['job'], t['health']) for t in json.load(sys.stdin)['data']['activeTargets']]"
```
