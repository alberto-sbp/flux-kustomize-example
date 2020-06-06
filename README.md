# Using Flux with Kustomize


## Scenario and Goals

The following makes use of Flux's manifest-generation feature
together with [Kustomize](https://github.com/kubernetes-sigs/kustomize) (and other such tooling, in theory).

For this project we consider an scenario with three clusters, `test`, `acceptance` and
`production`. The goal is to levarage the full functionality of Flux (including
automatic releases and supporting all `fluxctl` commands) to manage all
clusters while minimizing multiple declarations.

`test`, `staging` and `production` are almost identical.

## Create a kind cluster

```bash
curl https://raw.githubusercontent.com/schubergphilis/k8s-team-ckad-training/master/cluster-config.yml --silent --output cluster-config.yml

kind create cluster --config cluster-config.yml
```

Deploy calico overlay network (required for the network policy)

```bash
kubectl apply -f https://raw.githubusercontent.com/schubergphilis/k8s-team-ckad-training/master/calico.yml
kubectl -n kube-system set env daemonset/calico-node FELIX_IGNORELOOSERPF=true
```

## How to run the example

In order to run this example, you need to:

1. Deploy Flux version 1.13.0 or newer.

2. Make sure to pass the flag `--manifest-generation=true` to fluxd, in its container spec.

3. Fork this repository and add the fork's URL as the `--git-url` flag for the fluxd container.

4. Pick an environment to run (`staging` or `production`) and ask Flux to use
that environment by passing flag `--git-path=staging` or `--git-path=production`

5. As usual, you need to make sure that the ssh key shown by `fluxctl identity`
is added to the your github fork.

```bash

kubectl create ns flux

helm upgrade -i helm-operator fluxcd/helm-operator \
--namespace flux \
--set helm.versions=v3

helm upgrade -i flux fluxcd/flux --wait \
--namespace flux \
--set manifestGeneration=true \
--set git.path=test \
--set syncGarbageCollection.enabled=true \
--set git.url=git@github.com:alberto-sbp/flux-kustomize-example

```

## How does this example work?

```
├── .flux.yaml
├── base
│   ├── consul.yaml
│   ├── kustomization.yaml
├── test
│   ├── flux-patch.yaml
│   └── kustomization.yaml
|   └── consul.yaml
├── acceptance
│   ├── flux-patch.yaml
│   └── kustomization.yaml
|   └── consul.yaml
└── production
    ├── flux-patch.yaml
    ├── kustomization.yaml
    └── consul.yaml
```

* `base` contains the base manifests. The resources to be deployed in `test`,
  `staging` and `production` are almost identical to the ones described here.
* the `staging` and `production` directories make use of `base`, with a few patches, 
  to generate the final manifests for each environment:
    * `staging/kustomization.yaml` and `production/kustomization.yaml`
       are Kustomize config files which indicate how to apply the patches.
    * `staging/flux-patch.yaml` and `production/flux-patch.yaml` contain
       environment-specific Flux [annotations](https://docs.fluxcd.io/en/latest/tutorials/driving-flux/)
       and the container images to be deployed in each environment.
    * `production/replicas-patch.yaml` increases the number of replicas of podinfo in production.
* `.flux.yaml` is used by Flux to generate and update manifests. 
  Its commands are run in the directory (`staging` or `production`). 
  In this particular case, `.flux.yaml` tells Flux to generate manifests running 
  `kustomize build` and update policy annotations and container images by editing 
  `flux-patch.yaml`, which will implicitly applied to the manifests generated with
  `kustomize build`.
