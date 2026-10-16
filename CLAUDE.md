# OneK8s example application - notes for Claude sessions

The shared working notes for all OneK8s repos are in `onek8s/aks-terraform/CLAUDE.md`. Read that first.

This is an application team's repo: namespace `ext-fi-example` in isolation zone ext-fi, app
`aks-hello` (image amd64 only). Placement is stated with pod labels only: `platform/arch: amd64` and
`platform/capacity: spot`; the platform's Azure Policy mutations (azure-landing-zone
`stacks/aks-policy`) add the arch nodeSelector, the spot toleration/preference and the zone's
nodeSelector + toleration from the namespace, so it runs on spot when available and on on-demand
otherwise. **Lab cost exception:** `stateless/kustomization.yaml` patches it onto the cell's system
node (`kubernetes.azure.com/mode: system`, toleration `CriticalAddonsOnly`) with the label
`platform/zone-placement: "off"`, which the zone mutation skips (the zone validation still reports it,
in audit), so dev runs no zone nodes at all; remove that patch to go back to the zone's nodes. The same file leaves out aks-hello's `VerticalPodAutoscaler`: the dev cells run without the VPA add-on (one 8 GiB system node, 2026-10-14); drop that patch when the add-on is back. Routes: `hello-${cluster_name}.${zone_domain}` per cell and the global
`hello.${zone_domain}` behind Traffic Manager (annotation
`external-dns.kubernetes.io/controller: traffic-manager`). No GitHub CI runs here: validate with
`kustomize build` and the fleet's `check-egress.py`.

**Firewall-egress cell (design D59, 2026-10-15):** `firewall-egress/` is what the lab's az3 cell syncs into
`int-se-example` (Sweden's zone): aks-hello 0.5.0 on the zone's own nodes (no system-node patch: a cell without an
overlay identifies the zone at the hub firewall by the pod's subnet), with a Stockholm weather forecast from
api.open-meteo.com. No Cilium FQDN policy: the hub firewall decides that egress, and denies it until
azure-landing-zone's `firewall-applications/dev-int/int-se-example.yaml` allows it - proposed by the egress agent
(aks-ai-agents) from the denied flows, merged by a human.


**Overlay + firewall-only cell (design D61, 2026-10-16):** `firewall-egress-int-fi/` is what the lab's az2 cell syncs into
`int-fi-example`: aks-hello 0.5.0 (Helsinki forecast from api.open-meteo.com, public IP from api.ipify.org) on int-fi's own zone
nodes. az2 has no ACNS, so no Cilium FQDN policy: the hub firewall decides, per zone (it sees the zone's node addresses),
from azure-landing-zone's `firewall-applications/dev-int/int-fi-example.yaml` - proposed by the egress agent, merged by a human.
