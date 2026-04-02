# Forgejo Policies

Status: not applied yet.

Reason:
- CI runners usually require broad and variable outbound access
- this namespace needs a dedicated egress audit before enforcement

Planned next step:
- separate Forgejo core traffic from runner traffic
- document SSH / HTTP ingress requirements
- restrict runner egress incrementally after validating jobs
