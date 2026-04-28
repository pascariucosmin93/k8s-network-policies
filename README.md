# k8s-network-policies

Production-ready GitOps repository for Kubernetes network policies on clusters running Cilium and managed with Argo CD. The current layout is optimized for higher-scale microservice environments where policy count, Envoy/L7 overhead, and GitOps reconciliation cost matter.

## What Was Wrong In The Original Repo

- The repository was organized as flat namespace folders, which made promotion across environments difficult and encouraged copy/paste drift.
- Several policies allowed traffic by port only, without restricting destinations. Examples included public ingress on app ports and `443` egress with no target constraint.
- Kubernetes API access was sometimes modeled as plain `NetworkPolicy` egress on `443` or `6443`, which is unreliable with Cilium kube-proxy replacement and too broad from a zero-trust perspective.
- DNS was handled per namespace, but not as a reusable baseline.
- Namespace naming and structure were inconsistent. There were typo-prone directories such as `dasboard`, `clouadfare`, and `postgress`.
- There was no Argo CD layout for safe GitOps promotion, pruning, or self-healing.

## What Improved

- Policies are now split into reusable shared baselines and app-specific modules.
- Shared controls are Cilium-aware:
  - `default-deny`
  - DNS allow with DNS L7 inspection
- App policies demonstrate zero-trust controls with:
  - explicit ingress sources
  - explicit egress destinations
  - selective HTTP L7 method/path restrictions only on sensitive edges
  - FQDN-based egress for external APIs
- kube-apiserver access is no longer global. It is opt-in through a reusable module, and the provided app examples use workload-specific API rules instead of namespace-wide API access.
- Hot paths use identity-based L3/L4 rules where possible to reduce proxy overhead under load.
- Environment overlays use Kustomize so Argo CD can deploy each namespace independently.
- Argo CD includes both a single `Application` example and an `ApplicationSet` for fleet rollout with automated sync, prune, self-heal, `PruneLast`, and `ServerSideApply`.

## Repository Layout

```text
.
├── argocd
│   ├── application.yaml
│   └── applicationset.yaml
└── network-policies
    ├── apps
    │   ├── gaz
    │   │   ├── kustomization.yaml
    │   │   └── policies.yaml
    │   └── monitoring
    │       ├── kustomization.yaml
    │       └── policies.yaml
    ├── base
    │   ├── allow-dns.yaml
    │   ├── deny-all.yaml
    │   └── kustomization.yaml
    ├── common
    │   └── kube-api
    │       ├── allow-kube-api.yaml
    │       └── kustomization.yaml
    └── envs
        ├── dev
        │   ├── gaz
        │   │   └── kustomization.yaml
        │   └── monitoring
        │       └── kustomization.yaml
        └── prod
            ├── gaz
            │   └── kustomization.yaml
            └── monitoring
                └── kustomization.yaml
```

## Core YAML Examples

### Shared Baseline: `deny-all`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

### Shared Baseline: `allow-dns`

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-dns
spec:
  endpointSelector: {}
  egress:
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s:k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: UDP
            - port: "53"
              protocol: TCP
          rules:
            dns:
              - matchPattern: "*"
```

### Optional Shared Module: `allow-kube-api`

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-kube-api
spec:
  endpointSelector:
    matchExpressions:
      - key: io.kubernetes.pod.namespace
        operator: Exists
  egress:
    - toEntities:
        - kube-apiserver
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
            - port: "6443"
              protocol: TCP
```

### App Example: `gaz`

The `gaz` app module demonstrates:

- ingress only from the Cilium ingress path
- service-account-based microservice identity
- L7 only on north-south and sensitive billing edges
- L4-only rules on hot east-west paths for lower proxy cost
- FQDN-based egress for external partners
- explicit database access

See [network-policies/apps/gaz/policies.yaml](/home/cosmin/k8s-network-policies/network-policies/apps/gaz/policies.yaml:1).

## Argo CD Deployment

1. Install Cilium policy CRDs and Argo CD in the cluster.
2. Apply the bootstrap application:

```bash
kubectl apply -f argocd/application.yaml
```

3. Apply the fleet `ApplicationSet`:

```bash
kubectl apply -f argocd/applicationset.yaml
```

4. Argo CD will create one child application per environment/namespace overlay from `network-policies/envs/*/*`.
5. Validate rendered manifests before promotion:

```bash
kubectl kustomize network-policies/envs/dev/gaz
kubectl kustomize network-policies/envs/prod/monitoring
```

## Operational Notes

- `allow-kube-api` is opt-in. Include `network-policies/common/kube-api` only for namespaces that actually need Kubernetes API access, such as observability, GitOps controllers, operators, or service meshes.
- Intra-namespace east-west traffic is intentionally not globally allowed. Add a dedicated app policy only where required.
- External FQDN rules in app policies should be reviewed per environment before production rollout.
- For 10k-user scale, prefer this policy pattern:
  - baseline deny everywhere
  - DNS globally per namespace
  - kube API only for controllers
  - L7 on ingress and high-risk APIs
  - L3/L4 plus workload identity on high-throughput service-to-service paths
- Ensure workloads use stable labels and dedicated service accounts, otherwise the identity-based rules in the app modules will not match.
