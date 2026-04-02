# Forgejo Runner Policies

Status: ingress hardening applied. Runner egress remains open by design for now.

Applied in cluster:
- `00-default-deny-ingress.yaml`

What is enforced:
- all pods in `forgejo-runner` are ingress-denied by default
- this does not affect `runner -> dind` communication because that traffic stays inside the same pod on `127.0.0.1:2375`
- this does not affect `runner -> forgejo` communication because that is egress, not ingress

Audited dependencies:
- runners register and poll `http://forgejo.forgejo.svc.cluster.local:3000`
- runner pods use a `docker:27-dind` sidecar
- job execution needs broad outbound internet access for image pulls, package installs, git clones, and arbitrary CI steps

Why egress is not restricted yet:
- a generic `default-deny-egress` would very likely break CI jobs
- outbound destinations vary by repository, action, base image, registry, package manager, and job script
- the safe next step would require repo-by-repo CI profiling, not a blind policy

Validation performed:
- all four runner deployments remained `Available`
- all runner pods remained `2/2 Running`
- no restart or storage change was required

Still pending:
- optional split between "trusted internal runners" and "internet-capable runners"
- repo-specific CI egress profiling if you want tighter outbound controls later
