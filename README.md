# idp-cluster

This runbook builds the `idp-cluster` kind cluster and hands it to Flux CD.
Flux CD is a GitOps tool that reads this Git repo and applies what it finds to
the cluster.

A GitOps setup uses the Git repo as the source of truth. The cluster matches
the repo, and the repo changes through normal Git commits.

## 1. Create the cluster

Run this from the repo root:

```bash
kind create cluster --name idp-cluster --config kind-config.yaml
```

The `kind-config.yaml` file sets `disableDefaultCNI: true`. A CNI is the
plugin that routes traffic between pods. The cluster has no CNI after this
step.

## 2. Install Cilium out-of-band

Run this from the repo root:

```bash
./cilium/install.sh
```

Out-of-band means by hand, before Flux starts. The script loads the
`cilium/cilium:v1.20.4` image into the kind nodes. Then it installs Cilium
with Helm into the `kube-system` namespace, using the values in
`cilium/values.yaml`.

You must install Cilium before Flux. The Flux controllers need a working
network to start, and this cluster has no other CNI.

## 3. Bootstrap Flux

Run this from the repo root:

```bash
flux bootstrap github \
  --token-auth \
  --owner=axolotlite \
  --repository=flux-test \
  --branch=main \
  --path=flux/00-flux-system \
  --personal
```

Bootstrap installs Flux in the `flux-system` namespace and points it at this
repo. The `--path` flag names the directory Flux applies first. That directory
holds two Kustomizations: `00-infra.yaml` and `01-apps.yaml`.

A Kustomization is a Flux object that points at a path in the repo and applies
what it finds there.

## 4. How the order works

The directories are numbered by importance, from `00-flux-system` to
`01-infra` to `02-apps`. The numbers are a reading aid. Flux does not read
them.

Flux applies top-level Kustomizations in parallel. It uses `dependsOn` to
force an order. Here only `apps` depends on `infra`, so apps deploy after
infra. The `infra` and `flux-system` groups have no `dependsOn`.

`flux bootstrap` applies `00-flux-system` first by construction. It creates the
two sync Kustomizations in that directory. Those two then pull in `01-infra`
and `02-apps`.

Flux does not wait for a CNI health check. You install Cilium out-of-band in
step 2, so the network exists before Flux starts.

## What is deployed

Flux applies everything from this repo. You do not run `flux create` by hand.

`02-apps/podinfo/` deploys podinfo. A `GitRepository` object points at
`github.com/stefanprodan/podinfo`, branch `master`. A `Kustomization` object
applies the manifests from the `./kustomize` path in that repo into the
`default` namespace. It sets the HPA `minReplicas` to `1` with a patch.

`01-infra/00-cilium/` manages Cilium. A `HelmRepository` object points at the
chart index. A `HelmRelease` object installs the chart into `kube-system`.
This release adopts the Cilium install from step 2. From then on, Flux handles
upgrades and value changes.

To add a component, add a directory under `02-apps/` or `01-infra/`. Then add
that directory to the group's `kustomization.yaml`.

The old `clusters/idp-cluster/` layout is replaced by `flux/`. You can delete
it.
