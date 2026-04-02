# K8s Network Policies

## Goal

This folder documents and stages Kubernetes network policy hardening for the cluster.

Current priority:
- keep BGP-based external exposure stable
- do not change router `.100` NAT / port-forward behavior
- do not change the current `LoadBalancer` VIP model during the first phase
- improve only internal cluster networking controls first

That means the first implementation phase focuses on:
- namespace isolation
- pod-to-pod traffic restrictions
- explicit DNS allowances
- explicit app dependency allowances
- safer service classification

## Current State

Observed cluster characteristics:
- Cilium is the CNI and kube-proxy replacement
- Hubble is enabled
- BGP control plane is enabled
- routing mode is `tunnel` with `vxlan`
- there are 2 Cilium `LoadBalancer` IP pools:
  - `lb-pool-1` -> `10.30.10.0/24`
  - `lb-pool-2` -> `10.40.10.0/24`
- external access is not direct internet exposure from the cluster
- `LoadBalancer` VIPs are announced through BGP to router `.100`
- router `.100` performs NAT / port forwarding toward cluster VIPs

Important constraint:
- we should not touch BGP peering, router advertisement behavior, or external service exposure in the first hardening phase

## Main Risks Found

### 1. Very few network policies exist

At the time of review, the cluster had almost no `CiliumNetworkPolicy` coverage.

Practical impact:
- east-west traffic is too permissive
- compromise of one workload can lead to lateral movement
- troubleshooting is harder because "allowed by default" hides bad dependency design

### 2. LoadBalancer pool separation is not yet trustworthy

There are 2 pools, but actual service placement is not fully aligned with the intended logical split.

Practical impact:
- pool labels alone should not be treated as enforcement
- service classification must be documented explicitly
- pool usage should be cleaned up later, but without mixing it into the first hardening step

### 3. Hubble loses some events

Hubble flow visibility is useful, but not perfect during heavier activity.

Practical impact:
- logs and metrics are good enough for rollout validation
- they should not be treated as complete forensic truth during every incident

## Scope of Phase 1

Phase 1 changes only internal traffic rules.

Included:
- default deny policies per namespace
- allow DNS egress
- allow ingress from approved in-cluster entrypoints
- allow explicit app dependencies only
- allow monitoring flows only where required

Excluded:
- BGP changes
- router `.100` NAT changes
- `LoadBalancer` IP reallocation
- ingress VIP redesign
- native routing / tunnel redesign
- `exportPodCIDR` changes

## Rollout Status

Already applied in cluster:
- `cv`
- `employee-leave`
- `todo-app`
- `immich`
- `gaz`

Validated after apply:
- `cv` VIP `10.30.10.3` returned `200`
- `employee-leave` VIP `10.30.10.9` returned `200`
- `todo-app` VIP `10.30.10.10` returned `200`
- `immich` VIP `10.30.10.7` returned `200`
- `gaz` VIP `10.30.10.6` returned `200`
- `gaz` gateway VIP `10.30.10.8` returned `200`

Not applied yet:
- `monitoring`
- `argocd`
- `forgejo`

Reason these are still pending:
- they have more sensitive or variable outbound dependencies
- a blind `default-deny-egress` would carry a higher outage risk

## Rollout Strategy

### Rule 1

Do not start in `kube-system`.

### Rule 2

Do not start with egress-heavy namespaces like `argocd`, `forgejo`, or `monitoring`.

### Rule 3

Start with smaller application namespaces first, then expand.

Recommended order:
1. `cv`
2. `todo-app`
3. `employee-leave`
4. `gaz`
5. `immich`
6. `monitoring`
7. `argocd`
8. `forgejo`

## Baseline Policy Model

Each application namespace should start with this model:

1. `default-deny-ingress`
2. `default-deny-egress`
3. `allow-dns-egress`
4. `allow-ingress-from-approved-entrypoint`
5. `allow-app-specific-dependencies`

This avoids two common rollout mistakes:
- relying on one giant shared policy
- mixing ingress hardening and dependency discovery in a single change

## Namespace Policy Recommendations

### `cv`

Recommended baseline:
- deny all ingress by default
- deny all egress by default
- allow DNS to `kube-system/kube-dns`
- allow ingress only from the namespace/pods that terminate public traffic for this app
- allow egress only to explicitly required backends

Expected shape:
- internet -> router `.100` -> VIP -> ingress/gateway -> `cv`
- no arbitrary direct pod-to-pod access from unrelated namespaces

### `todo-app`

Recommended baseline:
- default deny ingress
- default deny egress
- allow DNS
- allow frontend -> backend
- deny unrelated namespace access

### `employee-leave`

Recommended baseline:
- default deny ingress
- default deny egress
- allow DNS
- allow frontend -> backend
- allow backend -> database only if needed

### `gaz`

Recommended baseline:
- default deny ingress
- default deny egress
- allow DNS
- allow gateway/ingress -> exposed services only
- allow only required service-to-service flows
- allow database/cache/queue access only from the services that need it

This namespace should be treated as the main networking hardening target because it contains multiple services and therefore the highest lateral movement risk.

### `immich`

Recommended baseline:
- default deny ingress
- default deny egress
- allow DNS
- allow ingress only from approved entrypoint
- allow app -> postgres
- allow app -> redis

### `monitoring`

Recommended baseline:
- default deny ingress
- default deny egress
- allow DNS
- allow Grafana -> Prometheus
- allow Grafana -> Loki
- allow Alloy -> Loki
- allow Prometheus scrape targets only where needed

This namespace should be hardened carefully because it is operationally sensitive.

### `argocd`

Recommended baseline:
- default deny ingress
- default deny egress
- allow DNS
- allow UI access only from the approved entrypoint path
- allow repo-server/controller outbound access only to required Git / Helm / OCI / Kubernetes API destinations

### `forgejo`

Recommended baseline:
- default deny ingress
- default deny egress
- allow DNS
- allow only required web and SSH ingress
- restrict runner egress as tightly as practical

This namespace is high risk because CI runners usually need broad outbound access unless deliberately constrained.

## Safe Implementation Pattern

For each namespace:

1. document expected traffic
2. apply `default-deny-ingress`
3. apply `allow-dns-egress`
4. validate app still resolves dependencies
5. apply `default-deny-egress`
6. add only the minimum required allow rules
7. validate again with Hubble, Loki, app health checks, and manual smoke tests

## Validation Checklist

Each rollout should validate:
- service health checks still pass
- app homepage / API still works through the current external path
- DNS resolution still works
- internal dependency calls still succeed
- Hubble does not show unexpected drops for intended traffic
- Loki shows only expected deny events during testing

## What Not To Change Yet

To avoid downtime, do not combine namespace policy rollout with:
- BGP peering changes
- router `.100` changes
- pool rebalance
- VIP renumbering
- ingress architecture changes
- `exportPodCIDR` cleanup
- tunnel/native routing changes

These are separate follow-up tasks.

## Follow-Up After Phase 1

Once namespace isolation is stable, the next networking tasks should be:
- clean up real `LoadBalancer` pool assignment and enforcement
- audit which services belong in internal vs externally forwarded VIP ranges
- review whether `exportPodCIDR: true` is actually required
- improve Hubble/Loki flow troubleshooting for incident response

## Folder Layout

This repo should keep networking hardening material here first, independent of GitOps rollout.

Suggested structure:

```text
k8s-network-policies/
  README.md
  cv/
  todo-app/
  employee-leave/
  gaz/
  immich/
  monitoring/
  argocd/
  forgejo/
```

Each namespace folder should eventually contain:
- baseline deny policies
- DNS allow policy
- namespace-specific allow rules
- short notes about expected traffic

## Current Implementation Notes

- `cv` currently keeps public ingress open on HTTP because it is exposed directly through its own `LoadBalancer` service
- `employee-leave` and `todo-app` use frontend/backend separation with explicit frontend -> backend allowance
- `immich` allows only server -> postgres and server -> redis internally
- `gaz` currently keeps same-namespace service traffic open while restricting namespace boundaries and preserving gateway/backend traffic

## Recommendation

Immediate next step:
- implement and test baseline policies first for `cv`, `todo-app`, and `employee-leave`
