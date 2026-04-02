# Argo CD Policies

Status: not applied yet.

Reason:
- this namespace needs outbound access to Git / Helm / OCI / Kubernetes API paths
- a blind `default-deny-egress` risks deployment failures

Planned next step:
- audit repo-server and controller outbound destinations
- define minimum ingress for the UI
- apply policies after dependency validation
