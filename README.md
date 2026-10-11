# aks-fluxcd-example

An application team's repository in [OneK8s](https://onek8s.github.io/) - one Kubernetes platform for
every cloud: what Flux deploys into the `example` team namespace of the clusters built by
[aks-terraform](https://github.com/onek8s/aks-terraform). Nothing here names a cloud: plain
Kubernetes and Gateway API manifests, which move to another cloud's clusters unchanged. aks-terraform gives the namespace a
Flux configuration of its own, also named `example` (`managed_namespaces.example.flux`), that syncs
`apps/` of this repository into it - its status is on the cluster's GitOps page in the Azure portal.
Everything else - the apps' source and images - lives in the apps' own repositories.

The stateless clusters, aks-dev-stateless-az1 and aks-dev-stateless-az2, have no persistent storage
at all, so they sync `stateless/` instead: the apps of `apps/` that keep no state - aks-hello, not
boot-log.

## Layout

```
apps/
  kustomization.yaml     # the apps in the namespace, one directory each
  aks-hello/             # onek8s/aks-hello, a .NET hello world website
    deployment.yaml
    service.yaml
    httproute.yaml       # attaches to the platform's `public` and `internal` Gateways
    poddisruptionbudget.yaml
  boot-log/              # a stateful example: a log of its own starts, on a portable-disk volume
    persistentvolumeclaim.yaml
    deployment.yaml
    service.yaml
    httproute.yaml       # /boot-log/ of the same hostnames as aks-hello
stateless/
  kustomization.yaml     # the apps that keep no state, for the stateless clusters: aks-hello only
```

Adding an app is a new directory under `apps/` and a line in `apps/kustomization.yaml` - and in
`stateless/kustomization.yaml`, as `../apps/<app>`, if it needs no volume.

## Who owns what

The split follows the [Gateway API role model][roles]: the platform runs the gateway, the team
routes to its apps.

| | Where | |
| --- | --- | --- |
| `example` namespace, its quota, network policy and Pod Security Standard | aks-terraform | Platform |
| The Gateway API CRDs | aks-terraform | Platform |
| Traefik Proxy, its internal and public load balancers, the `internal` and `public` Gateways, their hostnames and listeners | aks-fluxcd-platform | Platform |
| TLS: the certificate of each listener, kept in Key Vault | aks-fluxcd-platform | Platform |
| DNS: the hostnames' records in Azure DNS and Azure Private DNS, kept by external-dns | aks-fluxcd-platform | Platform |
| HTTP to HTTPS redirect | aks-fluxcd-platform | Platform |
| The Flux configuration that applies `apps/` | aks-terraform (`managed_namespaces.example.flux`) | Platform |
| The token it reads this repository with, and the `NetworkPolicy` that lets Traefik reach the pods | aks-fluxcd-platform (`tenants/example`) | Platform |
| The container registry the images are pulled from, as the cluster's kubelet identity | aks-terraform (`shared/`, `container_registry_name`) | Platform |
| The apps' `Deployment`, `Service`, `HTTPRoute` | this repository | Team |
| An app's source and image | the app's repository, e.g. [onek8s/aks-hello](https://github.com/onek8s/aks-hello) | Team |

An `HTTPRoute` names no hostname and no certificate. It attaches to the listeners the platform
gives the team - restricted to routes from the `example` namespace - and takes their hostnames:
`https-example` of the `public` Gateway, on the internet (aks-hello: `hello-<cluster>.onek8s.lol`),
`https-example-global` of the same Gateway, the name every stateless cluster serves at once
(`hello.onek8s.lol`, behind aks-terraform's Traffic Manager - the page says which cluster answered),
and `https-example-internal` of the `internal` Gateway, behind an Azure internal load balancer and
resolving inside the network only (`hello-<cluster>.internal.onek8s.lol`). Both serve the platform's wildcard
certificate. An app that should not be on the internet attaches to the internal listener only.
Changing a hostname or the certificate is a change to aks-fluxcd-platform; nothing here moves.

Flux applies `apps/` as the AKS Flux extension's `flux-applier` service account of the `example`
namespace, and the extension refuses references across namespaces - so this repository can only
ever change the `example` namespace. `apps/kustomization.yaml` sets `namespace: example` on
everything, since the configuration has no `targetNamespace` to do it.

[roles]: https://gateway-api.sigs.k8s.io/concepts/roles-and-personas/

The hostnames are set per cluster in aks-fluxcd-platform (`clusters/<cluster>/cluster-settings.yaml`).

## Persistent volumes

A volume is a `PersistentVolumeClaim` on the platform's **`portable-disk`** StorageClass - the only
one the namespace may use. Its disk is created in the platform's shared resource group, not in the
cluster's node resource group, so it outlives the cluster and can be moved to the next one. A claim
naming another class, or none (which would take AKS's `default`), is refused by the platform's
`portable-storage-only` admission policy, as is a pod with an inline `csi` volume. See
aks-fluxcd-platform's README, "Persistent volumes".

### Moving boot-log to another cluster

`boot-log` appends `<time> <node> <pod>` to `log.txt` on its volume every time it starts, and serves
it at `https://<example hostname>/boot-log/log.txt` - `hello-dev-stateful.onek8s.lol` on
aks-dev-stateful.

The stateless clusters - no disk CSI driver, no `portable-disk` StorageClass - do not run boot-log
(they sync `stateless/`), so the steps below need another cluster with portable disks to move the
disk to. They are kept for that: with a second stateful cluster, they move aks-dev-stateful's disk to
it.

1. **Stop it in the old cluster.** Suspend the namespace's Flux configuration, so the next sync does
   not start the pod again, and scale the app down - the disk detaches once the pod is gone. Note
   the disk's resource ID.

   ```sh
   az k8s-configuration flux update -g rg-aks-dev -c aks-dev-stateful -t managedClusters -n example --suspend true
   kubectl -n example scale deployment boot-log --replicas=0
   kubectl get pv "$(kubectl -n example get pvc boot-log -o jsonpath='{.spec.volumeName}')" \
     -o jsonpath='{.spec.csi.volumeHandle}'
   ```

2. **Hand the disk to the new cluster.** Suspend its configuration too, and remove its own claim -
   its disk is kept (`Retain`), and is deleted in Azure later if nobody wants it. Then a cluster
   admin creates a PersistentVolume for the moved disk, reserved for the `boot-log` claim (a
   PersistentVolume is cluster-wide, which the team cannot create):

   ```sh
   az k8s-configuration flux update -g rg-aks-dev -c aks-dev-stateful-2 -t managedClusters -n example --suspend true
   kubectl -n example scale deployment boot-log --replicas=0
   kubectl -n example delete pvc boot-log
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
       namespace: example
       name: boot-log
     csi:
       driver: disk.csi.azure.com
       volumeHandle: <the disk's resource ID from step 1>
   PV
   ```

3. **Start it there.** Resume the new cluster's configuration. Flux creates the `boot-log` claim
   again, it binds to the reserved volume, and the pod starts on the moved disk:

   ```sh
   az k8s-configuration flux update -g rg-aks-dev -c aks-dev-stateful-2 -t managedClusters -n example --suspend false
   curl https://hello-dev-stateful-2.onek8s.lol/boot-log/log.txt
   ```

   The lines written on aks-dev-stateful's node are there, followed by one from this cluster's.

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

Any secret an app needs goes in the vault as `example--<name>`, with an `ExternalSecret` in `apps/`
against the namespace's `key-vault` SecretStore - the namespace's identity reads only the secrets
under that prefix.

The manifests can use the post-build variable `${cluster_name}` - the name of the cluster they are
applied in, which aks-terraform gives every team's Flux configuration. aks-hello shows it on its page.
