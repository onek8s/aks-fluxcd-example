# OneK8s example application - notes for Claude sessions

The shared working notes for all OneK8s repos are in `onek8s/aks-terraform/CLAUDE.md`. Read that first.

This is an application team's repo, app `aks-hello` (image amd64 only). Placement is stated with pod labels
only: `platform/arch: amd64` and `platform/capacity: spot`; the platform's Azure Policy mutations
(azure-landing-zone `stacks/aks-policy`) add the arch nodeSelector, the spot toleration/preference and the
zone's nodeSelector + toleration from the namespace. Validate with `kustomize build` and the fleet's
`check-egress.py` (the CI workflow does both).

- `firewall-egress-int-fi/` is the same for `int-fi-example` (Finland's zone, in az3 since 2026-10-20: Helsinki's forecast; a
  second zone in the cell to test the isolation between zones).
- `firewall-egress/` is what the lab's one cell, az3, syncs into `int-se-example` (Sweden's zone): aks-hello on
  the zone's own nodes with a Stockholm forecast from api.open-meteo.com and its public IP from api.ipify.org.
  No Cilium FQDN policy: the hub firewall decides, per application, from azure-landing-zone's
  `firewall-applications/dev-int/int-se-example.yaml` - proposed by the egress agent (aks-ai-agents), merged
  by a human.
- `apps/` is the whole namespace (aks-hello + boot-log on a portable disk) for `int-fi-example` in the
  stateful cluster (defined, never deployed); `apps/aks-hello-global` is the Traffic Manager name of an
  external zone (no external cell runs today).
- **Lab replica switch**: `lab/scale/kustomization.yaml` (`replicas: aks-hello count`), included by
  `firewall-egress/`: **2 = running, what Git says** (keep it so: aks-terraform's `Lab power` start resumes
  Flux, which puts this number back); 0 = off in Git with the cluster running (the rollout guardrails allow 0).
  While the cell is stopped or Flux is suspended, nothing merged here reaches it until the start.
- **HTTP/2 upstream (design D69)**: aks-hello's Service carries `appProtocol: kubernetes.io/h2c` and targets its
  h2c port 8081 (probes stay HTTP/1.1 on 8080); the platform's policy `h2c-upstream` requires it on every routed
  Service port. boot-log's Service has none on purpose (busybox httpd is HTTP/1.1 only, the known audit finding).
- aks-hello 0.6.0 runs on the Ubuntu ASP.NET Core image with a shell (`kubectl exec -it <pod> -- bash`).
- aks-hello 0.7.0 adds ifconfig, ping, curl and wget to the image, all usable as UID 1654 with every capability
  dropped. ping needs the pod's sysctl `net.ipv4.ping_group_range: "1654 1654"` (deployment.yaml; already there
  while the image is 0.6.0) - keep it while ping should work. The image tag moves to 0.7.0 only after the registry
  has built it (aks-hello README, Releasing), and always with its digest (`deployment.yaml` and `firewall-egress/`'s
  `images:`): the platform admits images from the registry by digest only (`k8s-allowed-images`).
