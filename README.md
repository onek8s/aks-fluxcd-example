# aks-fluxcd-example

What Flux deploys into the `example` team namespace of the AKS clusters built by
[aks-terraform](https://github.com/olljanat-ai/aks-terraform). The platform
([aks-fluxcd-platform](https://github.com/olljanat-ai/aks-fluxcd-platform), `tenants/example`)
syncs `apps/` of this repository into `example`; everything else - the apps' source and images -
lives in the apps' own repositories.

## Layout

```
apps/
  kustomization.yaml     # the apps in the namespace, one directory each
  aks-hello/             # olljanat-ai/aks-hello, a .NET hello world website
    deployment.yaml
    service.yaml
    httproute.yaml       # attaches to the platform's `internal` Gateway
    poddisruptionbudget.yaml
```

Adding an app is a new directory under `apps/` and a line in `apps/kustomization.yaml`.

## Who owns what

The split follows the [Gateway API role model][roles]: the platform runs the gateway, the team
routes to its apps.

| | Where | |
| --- | --- | --- |
| `example` namespace, its quota, network policy and Pod Security Standard | aks-terraform | Platform |
| The Gateway API CRDs | aks-terraform | Platform |
| Traefik Hub API Gateway, its Azure internal load balancer, the `internal` Gateway, its hostnames and listeners | aks-fluxcd-platform | Platform |
| TLS: the certificate of each listener, kept in Key Vault | aks-fluxcd-platform | Platform |
| DNS: the hostname's record in Azure DNS, kept by external-dns | aks-fluxcd-platform | Platform |
| HTTP to HTTPS redirect | aks-fluxcd-platform | Platform |
| The Flux `GitRepository` and `Kustomization` that apply `apps/`, what they may do, and the `NetworkPolicy` that lets Traefik reach the pods | aks-fluxcd-platform (`tenants/example`) | Platform |
| The apps' `Deployment`, `Service`, `HTTPRoute` | this repository | Team |
| An app's source and image | the app's repository, e.g. [olljanat-ai/aks-hello](https://github.com/olljanat-ai/aks-hello) | Team |

An `HTTPRoute` names no hostname and no certificate. It attaches to the `https-example` listener of
the `internal` Gateway in `ingress-gateway`, which the platform restricts to routes from the
`example` namespace, and takes its hostname from there. Changing the hostname or the certificate is
a change to aks-fluxcd-platform; nothing here moves. The Gateway sits behind an Azure internal load
balancer, so the apps are reachable from inside the network only.

Flux applies `apps/` as the `flux-reconciler` service account of the `example` namespace, which
holds the `admin` role in that namespace and nothing outside it - so this repository can only ever
change the `example` namespace.

[roles]: https://gateway-api.sigs.k8s.io/concepts/roles-and-personas/

The hostname is set per cluster in aks-fluxcd-platform (`clusters/<cluster>/cluster-settings.yaml`).

## Releasing an app

An app's repository publishes its image; this repository decides which version runs. For
aks-hello: once a version is published to `ghcr.io/olljanat-ai/aks-hello` (see its README), change
the image tag in `apps/aks-hello/deployment.yaml` to it and merge. Flux rolls it out on its next
sync.

## Access to this private repository and the images

This repository and the image packages are private, so the cluster needs two secrets in the
`example` namespace. Neither is in Git:

- **`aks-fluxcd-example-git`** - read access to this repository, for Flux. A fine-grained personal
  access token (or a GitHub App) with read-only `Contents` on `olljanat-ai/aks-fluxcd-example`:

  ```sh
  kubectl -n example create secret generic aks-fluxcd-example-git \
    --from-literal=username=git --from-literal=password=github_pat_...
  ```

- **`ghcr-pull`** - read access to the images, for the nodes. A token with `read:packages`:

  ```sh
  kubectl -n example create secret docker-registry ghcr-pull \
    --docker-server=ghcr.io --docker-username=<github user> --docker-password=<token>
  ```

  Not needed once the packages are made public (package settings on GitHub); the reference in
  `apps/aks-hello/deployment.yaml` is harmless without the secret.
