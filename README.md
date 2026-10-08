# aks-fluxcd-example

What Flux deploys into the `example` team namespace of the AKS clusters built by
[aks-terraform](https://github.com/olljanat-ai/aks-terraform). aks-terraform gives the namespace a
Flux configuration of its own, also named `example` (`managed_namespaces.example.flux`), that syncs
`apps/` of this repository into it - its status is on the cluster's GitOps page in the Azure portal.
Everything else - the apps' source and images - lives in the apps' own repositories.

## Layout

```
apps/
  kustomization.yaml     # the apps in the namespace, one directory each
  aks-hello/             # olljanat-ai/aks-hello, a .NET hello world website
    deployment.yaml
    service.yaml
    httproute.yaml       # attaches to the platform's `public` and `internal` Gateways
    poddisruptionbudget.yaml
  boot-log/              # a stateful example: a log of its own starts, on a portable-disk volume
    persistentvolumeclaim.yaml
    deployment.yaml
    service.yaml
    httproute.yaml       # /boot-log/ of the same hostnames as aks-hello
```

Adding an app is a new directory under `apps/` and a line in `apps/kustomization.yaml`.

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
| The apps' `Deployment`, `Service`, `HTTPRoute` | this repository | Team |
| An app's source and image | the app's repository, e.g. [olljanat-ai/aks-hello](https://github.com/olljanat-ai/aks-hello) | Team |

An `HTTPRoute` names no hostname and no certificate. It attaches to the listeners the platform
gives the team - restricted to routes from the `example` namespace - and takes their hostnames:
`https-example` of the `public` Gateway, on the internet (aks-hello: `hello.onek8s.lol`), and
`https-example-internal` of the `internal` Gateway, behind an Azure internal load balancer and
resolving inside the network only (`hello.internal.onek8s.lol`). Both serve the platform's wildcard
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
it at `https://<example hostname>/boot-log/log.txt` - `hello.onek8s.lol` on aks-prototype-a,
`hello-b.onek8s.lol` on aks-prototype-b (the team's stable name is on aks-prototype-a; see
aks-fluxcd-platform's README, "Moving a workload to another cluster"). Both clusters sync this
repository, so each starts with a disk of its own. To move aks-prototype-a's disk to aks-prototype-b:

1. **Stop it in the old cluster.** Suspend the namespace's Flux configuration, so the next sync does
   not start the pod again, and scale the app down - the disk detaches once the pod is gone. Note
   the disk's resource ID.

   ```sh
   az k8s-configuration flux update -g rg-aks-prototype -c aks-prototype-a -t managedClusters -n example --suspend true
   kubectl -n example scale deployment boot-log --replicas=0
   kubectl get pv "$(kubectl -n example get pvc boot-log -o jsonpath='{.spec.volumeName}')" \
     -o jsonpath='{.spec.csi.volumeHandle}'
   ```

2. **Hand the disk to the new cluster.** Suspend its configuration too, and remove its own claim -
   its disk is kept (`Retain`), and is deleted in Azure later if nobody wants it. Then a cluster
   admin creates a PersistentVolume for the moved disk, reserved for the `boot-log` claim (a
   PersistentVolume is cluster-wide, which the team cannot create):

   ```sh
   az k8s-configuration flux update -g rg-aks-prototype -c aks-prototype-b -t managedClusters -n example --suspend true
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
   az k8s-configuration flux update -g rg-aks-prototype -c aks-prototype-b -t managedClusters -n example --suspend false
   curl https://hello-b.onek8s.lol/boot-log/log.txt
   ```

   The lines written on aks-prototype-a's node are there, followed by one from this cluster's.

The old cluster stays suspended for as long as the disk is the new one's; resuming it there would
start the pod on a claim whose disk is attached elsewhere.

## Releasing an app

An app's repository publishes its image; this repository decides which version runs. For
aks-hello: once a version is published to `ghcr.io/olljanat-ai/aks-hello` (see its README), change
the image tag in `apps/aks-hello/deployment.yaml` to it and merge. Flux rolls it out on its next
sync.

## Access to this private repository and the images

This repository and the image packages are private, so the cluster needs two secrets in the
`example` namespace. Neither is in Git: both are kept in the `example` namespace's share of the
cluster's Key Vault - the secrets named `example--...`, which this namespace and no other can read -
and synced into the namespace by the platform (aks-fluxcd-platform, `tenants/example/key-vault.yaml`):

- **`example--aks-fluxcd-example-git`** - read access to this repository, for Flux. A fine-grained
  personal access token (or a GitHub App) with read-only `Contents` on
  `olljanat-ai/aks-fluxcd-example`. Synced as `aks-fluxcd-example-git`.

  ```sh
  az keyvault secret set --vault-name <key vault> --name example--aks-fluxcd-example-git --value github_pat_...
  ```

- **`example--ghcr-pull`** - read access to the images, for the nodes: a Docker config JSON for a
  token with `read:packages`. Synced as `ghcr-pull`.

  ```sh
  az keyvault secret set --vault-name <key vault> --name example--ghcr-pull --value "$(
    kubectl create secret docker-registry ghcr-pull --dry-run=client -o jsonpath='{.data.\.dockerconfigjson}' \
      --docker-server=ghcr.io --docker-username=<github user> --docker-password=<token> |
    base64 -d)"
  ```

The team's writers can set these themselves. Any other secret an app needs goes in the vault the
same way, as `example--<name>`, with an `ExternalSecret` in `apps/` against the namespace's
`key-vault` SecretStore.
