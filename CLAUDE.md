# OneK8s example application - notes for Claude sessions

The shared working notes for all OneK8s repos are in `onek8s/aks-terraform/CLAUDE.md`. Read that first.

This is an application team's repo: namespace `ext-fi-example` in isolation zone ext-fi, app
`aks-hello` (image amd64 only, so it pins `kubernetes.io/arch: amd64`). It selects the zone's nodes
(`platform/zone: ext-fi` + toleration) and tolerates spot, so it runs on spot when available and on
on-demand otherwise. Routes: `hello-${cluster_name}.${zone_domain}` per cell and the global
`hello.${zone_domain}` behind Traffic Manager (annotation
`external-dns.kubernetes.io/controller: traffic-manager`). No GitHub CI runs here: validate with
`kustomize build` and the fleet's `check-egress.py`.
