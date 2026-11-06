# flux

This directory holds the Flux setup for the `idp-cluster` kind cluster. It is
ordered by importance: `flux-system`, then `infra`, then `apps`. This
directory lives inside the `idp-cluster` Git repo.

```
flux/
├── 00-flux-system/
│   ├── 00-infra.yaml   # Kustomization "infra" -> ./flux/01-infra
│   └── 01-apps.yaml    # Kustomization "apps"  -> ./flux/02-apps, dependsOn: [infra]
├── 01-infra/
│   ├── kustomization.yaml
│   └── 00-cilium/
│       ├── kustomization.yaml
│       ├── helmrepository.yaml
│       └── helmrelease.yaml
└── 02-apps/
    ├── kustomization.yaml
    └── podinfo/
        ├── kustomization.yaml
        ├── gitrepository.yaml
        └── fluxcd.yaml
```

## How the order works

The `00/01/02` prefixes are a reading aid. Flux does not read them.

Flux applies top-level Kustomizations in parallel. It uses `dependsOn` to
force an order. Only `apps` depends on `infra`, so apps deploy after infra
reconciles.

`00-flux-system` applies first by construction. `flux bootstrap` creates the
two sync Kustomizations in that directory. They pull in `01-infra` and
`02-apps`.

Flux does not wait for a CNI health check. You install Cilium out-of-band
before bootstrap, so the network exists when Flux starts.

## File conventions

Each Flux object lives in one file. The file name matches the object kind in
lowercase. For example, `helmrelease.yaml` holds a `HelmRelease`.

The `kustomization.yaml` in each directory is the kustomize build file. It
lists the files in that directory. It is not a Flux object.

The `fluxcd.yaml` file in `02-apps/podinfo/` holds the Flux `Kustomization`
that applies podinfo. It is named `fluxcd.yaml` so it does not clash with the
kustomize `kustomization.yaml` in the same directory.

The `dependsOn` field uses the object name, not the file name. Here `apps`
depends on `infra`.

## Path rules

A Kustomization `spec.path` is relative to the repo root. It carries the
`flux/` prefix, so the two sync paths are `./flux/01-infra` and
`./flux/02-apps`.

The podinfo `Kustomization` is the one exception. Its `path: ./kustomize` is
relative to the external `stefanprodan/podinfo` Git repo, not to this repo.

Workloads land in their own namespace. podinfo goes to `default`. Cilium goes
to `kube-system`.

To add a component, add a directory under `02-apps/` or `01-infra/`. Then add
that directory to the group's `kustomization.yaml`.

## Bootstrap order

The `kind-config.yaml` file sets `disableDefaultCNI: true`. The cluster has no
CNI until Cilium is up. The Flux controllers need a network to start, so
Cilium comes up out-of-band first. The `HelmRelease` in `01-infra` then takes
over and manages it.

Run these from the repo root:

```bash
# 1. Create the kind cluster (no default CNI)
kind create cluster --name idp-cluster --config kind-config.yaml

# 2. Install Cilium out-of-band (loads the image, then runs helm install)
./cilium/install.sh

# 3. Bootstrap Flux. The path points at the dir with the two sync Kustomizations.
flux bootstrap github \
  --token-auth \
  --owner=axolotlite \
  --repository=flux-test \
  --branch=main \
  --path=flux/00-flux-system \
  --personal
```

If you later move `flux/` into its own repo, change the path to
`--path=00-flux-system`. Also drop the `flux/` prefix from the two
Kustomization paths.

## Cilium values

Two sets of Cilium values exist, and each one serves one phase.

`cilium/values.yaml` holds the bootstrap values. The script
`cilium/install.sh` reads them during the out-of-band install, before Flux
exists. Their job is to start the CNI so the Flux controllers can run.

The `helmrelease.yaml` file holds the managed values. Once Flux is up, Flux
owns the Cilium release and manages it with these values. Edit them to change
Cilium, and Flux applies the change as a Helm upgrade.

When Flux first takes over, it moves the release to match the managed values.
If the two sets differ at that moment, Flux runs one upgrade to align them.
This is expected. The chart version stays pinned to `1.20.4`, the same image
`install.sh` loads.

The bootstrap values set two options:

- `image.pullPolicy: IfNotPresent`: kind preloads the Cilium image into each
  node, so pulling from the registry would fail (no CNI yet).
- `ipam.mode: kubernetes`: Cilium uses the node's existing IP pool. This is
  required for a kind cluster, and it keeps the install simple.

## Notes

`infra` uses `wait: false`. That is correct because the CNI is installed
out-of-band. Apps still depend on infra. They start once the infra objects are
applied, not after a CNI health check.

`prune: true` is set on `infra`. If you delete the Cilium directory, Flux
removes Cilium. Be deliberate before you do that.

`cilium/install.sh` is the first, out-of-band install. After that, Flux owns
upgrades and value changes through the `HelmRelease`.
