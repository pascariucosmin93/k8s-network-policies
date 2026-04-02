# Forgejo Policies

Status: ingress-only hardening planned.

Reason:
- CI runners usually require broad and variable outbound access
- this namespace needs a dedicated egress audit before enforcement
- safe first step is ingress-only hardening

Planned next step:
- apply `default-deny-ingress`
- allow public web and SSH ingress to Forgejo itself
- separate Forgejo core traffic from runner traffic
- document SSH / HTTP ingress requirements
- restrict runner egress incrementally after validating jobs
