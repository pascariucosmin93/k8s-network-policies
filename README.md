# k8s-network-policies

Production-ready GitOps repository for Kubernetes network security using Cilium and Argo CD.

## Audit Findings

The previous repository was closer to a demo than a safe production baseline. The main issues were:

- Missing explicit `metadata.namespace` in most policy manifests. The repo relied on Kustomize namespace injection, which makes raw YAML review harder and increases the chance of applying a policy into the wrong namespace during manual operations.
- Default deny existed, but the shared baseline did not include a controlled intra-namespace policy or a reusable kube-apiserver policy. This makes adoption harder and encourages teams to add one-off exceptions.
- `gaz` had ingress-only rules for several services but no matching egress from the caller. With a namespace default deny, that silently breaks east-west calls.
- `gaz` and `monitoring` used some destination selectors that were too broad or incomplete. Example: Prometheus egress to the whole `monitoring` namespace on multiple ports is wider than least privilege.
- DNS allow existed, but it matched only `k8s-app: kube-dns`. Many clusters use CoreDNS labels or both label sets during migrations.
- kube-apiserver access existed in two forms: a reusable module and app-specific copies. The repo did not clearly distinguish when API access should be namespace-wide versus workload-specific.
- Argo CD was configured with `automated.prune: true` from the start. That is unsafe for first-time GitOps adoption because any drift or missing manifest in Git can delete live policies immediately.
- The repository structure mixed reusable and app-specific policy concerns and used `policies.yaml` names inconsistently, which makes large-scale GitOps promotion harder to reason about.

## Target Layout

```text
.
├── argocd
│   ├── application.yaml
│   └── applicationset.yaml
└── network-policies
    ├── apps
    │   ├── gaz
    │   │   ├── kustomization.yaml
    │   │   └── policy.yaml
    │   └── monitoring
    │       ├── kustomization.yaml
    │       └── policy.yaml
    ├── base
    │   ├── allow-dns.yaml
    │   ├── allow-internal-namespace.yaml
    │   ├── allow-kube-api.yaml
    │   ├── deny-all.yaml
    │   └── kustomization.yaml
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

## Baseline Policy Model

- `deny-all.yaml`
  Zero-trust baseline. Nothing talks until explicitly allowed.
- `allow-dns.yaml`
  Shared DNS egress to CoreDNS or kube-dns on TCP/UDP 53.
- `allow-kube-api.yaml`
  Reusable Cilium policy for workloads that must call the Kubernetes API. Keep this opt-in unless a namespace is controller-heavy.
- `allow-internal-namespace.yaml`
  Optional namespaced east-west allow for legacy or tightly-coupled workloads. Do not enable it by default for sensitive apps.

## App Policy Model

- `apps/gaz/policy.yaml`
  Demonstrates zero-trust app isolation with explicit gateway ingress, explicit service-to-service egress, database-only access for the calculator, and partner API egress restricted by DNS name and port.
- `apps/monitoring/policy.yaml`
  Demonstrates platform namespace controls with ingress to Grafana only from the ingress path, Grafana to Loki on L7 HTTP, and explicit kube-apiserver access only for Prometheus and Alloy.

## Argo CD Adoption

The bootstrap `Application` and the fleet `ApplicationSet` intentionally use:

- `automated.selfHeal: true`
- `automated.prune: false`

Use `prune: false` for the initial GitOps adoption phase. Enable `prune: true` only after:

1. all live policies are exported and committed to Git,
2. every Argo app is `Synced` and `Healthy`,
3. you have validated that no unmanaged emergency policy is still required,
4. you have a rollback path for each namespace.

## Safe Migration Sequence

1. Export live policies from each namespace:

   ```bash
   kubectl get networkpolicy,ciliumnetworkpolicy -A -o yaml > existing-network-policies.yaml
   ```

2. Compare live state with Git and normalize names, labels, and namespaces before enabling Argo CD ownership.

3. Render each overlay locally and review:

   ```bash
   kubectl kustomize network-policies/envs/dev/gaz
   kubectl kustomize network-policies/envs/dev/monitoring
   ```

4. Apply the bootstrap Argo `Application`.

5. Let Argo sync with `prune: false`, then verify:
   - DNS resolution works from restricted pods
   - app-to-app traffic works only on intended paths and ports
   - metrics and dashboards still function
   - no unexpected denied flows appear in Hubble

6. After a clean burn-in period, switch Argo CD apps to `prune: true` and remove orphaned manual policies.

## Validation Commands

```bash
kubectl kustomize network-policies/envs/dev/gaz
kubectl kustomize network-policies/envs/dev/monitoring
kubectl get ciliumnetworkpolicies,networkpolicies -A
hubble observe --verdict DROPPED --last 50
```
