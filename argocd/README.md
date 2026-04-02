# Argo CD Policies

Status: ingress-only hardening planned.

Reason:
- this namespace needs outbound access to Git / Helm / OCI / Kubernetes API paths
- a blind `default-deny-egress` risks deployment failures
- safe first step is ingress-only hardening

Planned next step:
- apply `default-deny-ingress`
- allow same-namespace traffic
- allow public ingress only to `argocd-server`
- allow monitoring namespace scrapes to Argo CD metrics ports
- audit repo-server and controller outbound destinations
- define minimum ingress for the UI
- apply policies after dependency validation
