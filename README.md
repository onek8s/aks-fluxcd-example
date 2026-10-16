# aks-fluxcd-example

An application team's repository in [OneK8s](https://onek8s.github.io/) - one Kubernetes platform for
every cloud: what Flux deploys into the team's namespace in the **int-fi** isolation zone,
`int-fi-example`, of the clusters built by [aks-terraform](https://github.com/onek8s/aks-terraform).
Nothing here names a cloud: plain Kubernetes and Gateway API manifests, which move to another cloud's
clusters unchanged. aks-terraform gives the namespace a Flux configuration of its own, also named
`int-fi-example` (`managed_namespaces.int-fi-example.flux`), that syncs `apps/` of this repository
into it - its status is on the cluster's GitOps page in the Azure portal. Everything else - the apps'
source and images - lives in the apps' own repositories.

The stateless clusters, aks-dev-ext-stateless-az1 and aks-dev-ext-stateless-az2 - the design's cells - have
no persistent storage at all, so they sync `stateless/` instead: the apps of `apps/` that keep no
state - aks-hello, not boot-log.

## Layout

```
apps/
  kustomization.yaml     # the apps in the namespace, one directory each
  aks-hello/             # onek8s/aks-hello, a .NET hello world website
    deployment.yaml      # amd64, spot by choice (labels platform/arch, platform/capacity)
    service.yaml
    httproute.yaml       # the app's names on the int-fi zone's Gateway
    poddisruptionbudget.yaml
  boot-log/              # a stateful example: a log of its own starts, on a portable-disk volume
    persistentvolumeclaim.yaml
    deployment.yaml
    service.yaml
    httproute.yaml       # /boot-log/ of aks-hello's name in the cluster
stateless/
  kustomization.yaml     # the apps that keep no state, for the stateless clusters: aks-hello only
lab/
  scale/                 # the lab's replica switch for aks-hello in every cell: 0 (off) or 2 (running)
```

Adding an app is a new directory under `apps/` and a line in `apps/kustomization.yaml` - and in
`stateless/kustomization.yaml`, as `../apps/<app>`, if it needs no volume.

## Switching the lab's copies off and on

A pause of the whole lab is aks-terraform's **Lab power** workflow: its `stop` suspends the cells'
Flux configurations, scales the apps down in the clusters, turns NAP off and stops the clusters (the
hub's firewall last); its `start` reverts that, and Flux puts the state of Git back. Nothing here changes.

`lab/scale/kustomization.yaml` is a Git-only switch next to it: one kustomize component every cell
directory (`firewall-egress/`, `firewall-egress-int-fi/`, `stateless/`) includes, with one number -
aks-hello's replicas in every cell. `2` is running; `0` switches the application off with the clusters
running - its pods go and NAP removes the zones' nodes within minutes (the platform's rollout
guardrails allow 0 and at least 2, nothing between). Keep it at 2 for a Lab power `start` to bring
the app back. Change it in a pull request; the cells' Flux configurations apply `main` within a minute.

## Who owns what

The split follows the [Gateway API role model][roles]: the platform runs the zone's gateway, the team
routes to its apps.

| | Where | |
| --- | --- | --- |
| `int-fi-example` namespace in the int-fi zone, its quota, default network policies and Pod Security Standard | aks-terraform | Platform |
| The zone's Cilium baseline: in from the zone's gateway only, out to cluster DNS only | aks-fluxcd-platform (`infrastructure/zones`) | Platform |
| The Gateway API CRDs | aks-terraform | Platform |
| The int-fi gateway: the Gateway's App Routing (Envoy) pods, its load balancer, the Gateway `int-fi`, its HTTPS listener for `*.int-fi.dev.onek8s.lol` and the zone's wildcard certificate | aks-fluxcd-platform (`infrastructure/zones`) | Platform |
| DNS: the routes' hostnames in the zone's Azure Private DNS (`int-fi.dev.onek8s.lol`), kept by external-dns-int. In an external zone also `hello.<zone domain>` in Azure Traffic Manager (`apps/aks-hello-global`; aks-terraform `shared/`: `global_hostnames`) - none for an internal zone | aks-fluxcd-platform, aks-terraform | Platform |
| The Flux configuration that applies `apps/` | aks-terraform (`managed_namespaces.int-fi-example.flux`) | Platform |
| The namespace's share of the int-fi zone's Key Vault (no webhook Receiver in the internal cell: Flux polls) | aks-fluxcd-platform (`tenants/int-fi-example`) | Platform |
| The container registry the images are pulled from, as the cluster's kubelet identity | aks-terraform (`shared/`, `container_registry_name`) | Platform |
| The apps' `Deployment`, `Service`, `HTTPRoute` and egress policy | this repository | Team |
| An app's source and image | the app's repository, e.g. [onek8s/aks-hello](https://github.com/onek8s/aks-hello) | Team |

A route attaches to the zone's Gateway (`${zone}` in `${zone}-gateway`: `int-fi` in `int-fi-gateway`) and names its hostnames, below
the zone's domain - `${zone_domain}`, `int-fi.dev.onek8s.lol`, which aks-terraform gives the
namespace's Flux configuration together with `${cluster_name}`. aks-hello has two:

- `hello-${cluster_name}.${zone_domain}` - its name in each cluster, which the cluster's external-dns
  publishes as an A record of the cluster's int-fi gateway;
- `hello.${zone_domain}` - its name in every cell at once, behind aks-terraform's Azure Traffic
  Manager, which hands out one cell or another (the page says which cluster answered). That route
  carries `external-dns.kubernetes.io/controller: traffic-manager`, so external-dns leaves the name
  to Traffic Manager.

Both are served with the zone's wildcard certificate, on HTTPS only - there is no plain HTTP.

**Placement.** The pods state only what is the application's choice, as labels (design R17):
`platform/arch` (`amd64` default, `arm64`, `multi`) and `platform/capacity` (`on-demand` default,
`spot`). The platform's Azure Policy mutations (onek8s/azure-landing-zone `stacks/aks-policy`) turn
them into the architecture nodeSelector and the spot toleration and preference, and pin every pod of
`int-fi-*` to the int-fi zone's nodes; validations report a pod that selects nodes any other way.
aks-hello is an amd64 image and may run on spot.

**Egress.** Nothing leaves the namespace but cluster DNS unless an app declares it: a
`CiliumNetworkPolicy` named `egress` listing every outside destination by FQDN with its ports, and the
same names in its DNS rule (design section 16). aks-hello needs nothing outside the cluster and has
none. CI checks every egress policy against the platform's rules and the int-fi zone's allow-list
(`.github/workflows/ci.yml`).

Flux applies `apps/` as the AKS Flux extension's `flux-applier` service account of the
`int-fi-example` namespace, and the extension refuses references across namespaces - so this
repository can only ever change that namespace. `apps/kustomization.yaml` sets `namespace:
int-fi-example` on everything, since the configuration has no `targetNamespace` to do it.

[roles]: https://gateway-api.sigs.k8s.io/concepts/roles-and-personas/

## Persistent volumes

A volume is a `PersistentVolumeClaim` on the platform's **`portable-disk`** StorageClass - the only
one the namespace may use. Its disk is created in the platform's shared resource group, not in the
cluster's node resource group, so it outlives the cluster and can be moved to the next one. A claim
naming another class, or none (which would take AKS's `default`), is refused by the platform's
`portable-storage-only` admission policy, as is a pod with an inline `csi` volume. See
aks-fluxcd-platform's README, "Persistent volumes".

### Moving boot-log to another cluster

`boot-log` appends `<time> <node> <pod>` to `log.txt` on its volume every time it starts, and serves
it at `https://hello-<cluster>.int-fi.dev.onek8s.lol/boot-log/log.txt` on
aks-dev-ext-stateful.

The stateless clusters - no disk CSI driver, no `portable-disk` StorageClass - do not run boot-log
(they sync `stateless/`), so the steps below need another cluster with portable disks to move the
disk to. They are kept for that: with a second stateful cluster, they move aks-dev-ext-stateful's disk to
it.

1. **Stop it in the old cluster.** Suspend the namespace's Flux configuration, so the next sync does
   not start the pod again, and scale the app down - the disk detaches once the pod is gone. Note
   the disk's resource ID.

   ```sh
   az k8s-configuration flux update -g rg-aks-dev -c aks-dev-ext-stateful -t managedClusters -n int-fi-example --suspend true
   kubectl -n int-fi-example scale deployment boot-log --replicas=0
   kubectl get pv "$(kubectl -n int-fi-example get pvc boot-log -o jsonpath='{.spec.volumeName}')" \
     -o jsonpath='{.spec.csi.volumeHandle}'
   ```

2. **Hand the disk to the new cluster.** Suspend its configuration too, and remove its own claim -
   its disk is kept (`Retain`), and is deleted in Azure later if nobody wants it. Then a cluster
   admin creates a PersistentVolume for the moved disk, reserved for the `boot-log` claim (a
   PersistentVolume is cluster-wide, which the team cannot create):

   ```sh
   az k8s-configuration flux update -g rg-aks-dev -c aks-dev-ext-stateful-2 -t managedClusters -n int-fi-example --suspend true
   kubectl -n int-fi-example scale deployment boot-log --replicas=0
   kubectl -n int-fi-example delete pvc boot-log
   kubectl apply -f - <<'PV'
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: example-boot-log
   spec:
     capacity:
       storage: 1Gi
     accessModes: [ReadWriteOnce]
     persistentVolumeReclaimPolicy: Retain
     storageClassName: portable-disk
     claimRef:
       namespace: int-fi-example
       name: boot-log
     csi:
       driver: disk.csi.azure.com
       volumeHandle: <the disk's resource ID from step 1>
   PV
   ```

3. **Start it there.** Resume the new cluster's configuration. Flux creates the `boot-log` claim
   again, it binds to the reserved volume, and the pod starts on the moved disk:

   ```sh
   az k8s-configuration flux update -g rg-aks-dev -c aks-dev-ext-stateful-2 -t managedClusters -n int-fi-example --suspend false
   curl https://hello-aks-dev-ext-stateful-2.int-fi.dev.onek8s.lol/boot-log/log.txt
   ```

   The lines written on aks-dev-ext-stateful's node are there, followed by one from this cluster's.

The old cluster stays suspended for as long as the disk is the new one's; resuming it there would
start the pod on a claim whose disk is attached elsewhere.

## Releasing an app

An app's repository publishes its image; this repository decides which version runs. For
aks-hello: once the environment's registry has built a version (`acronek8sdev.azurecr.io/aks-hello`,
see aks-hello's README), change the image tag in `apps/aks-hello/deployment.yaml` to it and merge. Flux rolls it out on its next
sync.

## Access to this repository and the images

This repository is public, so Flux reads it without a token. A team whose repository is private keeps
a read-only token in its namespace's share of the Key Vault - `<namespace>--<name>`, synced into the
namespace by an `ExternalSecret` of the platform's - and names the synced Secret in aks-terraform's
`managed_namespaces.<team>.flux.secret_name`.

The images need no secret: they are in the environment's Azure Container Registry, and each cluster
pulls them as its kubelet identity, which has `AcrPull` there (aks-terraform).

Any secret an app needs goes in the int-fi zone's vault as `int-fi-example--<name>`, with an `ExternalSecret` in `apps/`
against the namespace's `key-vault` SecretStore - the namespace's identity reads only the secrets
under that prefix.

The manifests can use the post-build variables `${cluster_name}` - the name of the cluster they are
applied in, which aks-terraform gives every team's Flux configuration; aks-hello shows it on its page -
and `${zone_domain}`, the domain of the namespace's isolation zone.
