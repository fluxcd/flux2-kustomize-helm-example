# flux2-kustomize-helm-example

[![test](https://github.com/fluxcd/flux2-kustomize-helm-example/workflows/test/badge.svg)](https://github.com/fluxcd/flux2-kustomize-helm-example/actions)
[![e2e](https://github.com/fluxcd/flux2-kustomize-helm-example/workflows/e2e/badge.svg)](https://github.com/fluxcd/flux2-kustomize-helm-example/actions)
[![license](https://img.shields.io/github/license/fluxcd/flux2-kustomize-helm-example.svg)](https://github.com/fluxcd/flux2-kustomize-helm-example/blob/main/LICENSE)

For this example we assume a scenario with two clusters: staging and production.
The end goal is to leverage Flux and Kustomize to manage both clusters while minimizing duplicated declarations.

We will configure Flux to install, test and upgrade a demo app using
`HelmRepository` and `HelmRelease` custom resources.
Flux will monitor the Helm repository, and it will automatically
upgrade the Helm releases to their latest chart version based on semver ranges.

## Prerequisites

You will need a Kubernetes cluster version 1.33 or newer.
For a quick local test, you can use [Kubernetes kind](https://kind.sigs.k8s.io/docs/user/quick-start/).
Any other Kubernetes setup will work as well though.

In order to follow the guide you'll need a GitHub account and a
[personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line)
that can create repositories (check all permissions under `repo`).

Install the Flux CLI on macOS or Linux using Homebrew:

```sh
brew install fluxcd/tap/flux
```

Or install the CLI by downloading precompiled binaries using a Bash script:

```sh
curl -s https://fluxcd.io/install.sh | sudo bash
```

## Repository structure

The Git repository contains the following top directories:

- **apps** dir contains Helm releases with a custom configuration per cluster
- **infrastructure** dir contains common infra tools such as ingress-nginx and cert-manager
- **clusters** dir contains the Flux configuration per cluster

```
├── apps
│   ├── base
│   ├── production 
│   └── staging
├── infrastructure
│   ├── configs
│   └── controllers
└── clusters
    ├── production
    └── staging
```

### Applications

The apps configuration is structured into:

- **apps/base/** dir contains namespaces and Helm release definitions
- **apps/production/** dir contains the production Helm release values
- **apps/staging/** dir contains the staging values

```
./apps/
├── base
│   └── podinfo
│       ├── kustomization.yaml
│       ├── namespace.yaml
│       ├── release.yaml
│       └── repository.yaml
├── production
│   ├── kustomization.yaml
│   └── podinfo-patch.yaml
└── staging
    ├── kustomization.yaml
    └── podinfo-patch.yaml
```

In **apps/base/podinfo/** dir we have a Flux `HelmRelease` with common values for both clusters:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: podinfo
  namespace: podinfo
spec:
  releaseName: podinfo
  chart:
    spec:
      chart: podinfo
      sourceRef:
        kind: HelmRepository
        name: podinfo
        namespace: flux-system
  interval: 50m
  values:
    ingress:
      enabled: true
      className: nginx
```

In **apps/staging/** dir we have a Kustomize patch with the staging specific values:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: podinfo
spec:
  chart:
    spec:
      version: ">=1.0.0-alpha"
  test:
    enable: true
  values:
    ingress:
      hosts:
        - host: podinfo.staging
```

Note that with `version: ">=1.0.0-alpha"` we configure Flux to automatically upgrade
the `HelmRelease` to the latest chart version including alpha, beta and pre-releases.

In **apps/production/** dir we have a Kustomize patch with the production specific values:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: podinfo
  namespace: podinfo
spec:
  chart:
    spec:
      version: ">=1.0.0"
  values:
    ingress:
      hosts:
        - host: podinfo.production
```

Note that with ` version: ">=1.0.0"` we configure Flux to automatically upgrade
the `HelmRelease` to the latest stable chart version (alpha, beta and pre-releases will be ignored).

### Infrastructure

The infrastructure is structured into:

- **infrastructure/controllers/** dir contains namespaces and Helm release definitions for Kubernetes controllers
- **infrastructure/configs/** dir contains Kubernetes custom resources such as cert issuers and networks policies

```
./infrastructure/
├── configs
│   ├── cluster-issuers.yaml
│   └── kustomization.yaml
└── controllers
    ├── cert-manager.yaml
    ├── ingress-nginx.yaml
    └── kustomization.yaml
```

In **infrastructure/controllers/** dir we have the Flux definitions such as:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: OCIRepository
metadata:
  name: cert-manager
  namespace: cert-manager
spec:
  interval: 24h
  url: oci://quay.io/jetstack/charts/cert-manager
  layerSelector:
    mediaType: "application/vnd.cncf.helm.chart.content.v1.tar+gzip"
    operation: copy
  ref:
    semver: "1.x"
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: cert-manager
  namespace: cert-manager
spec:
  interval: 12h
  chartRef:
    kind: OCIRepository
    name: cert-manager
  values:
    crds:
      enabled: true
      keep: false
```

Note that in the `OCIRepository` we configure Flux to check for new chart versions every 24 hours.
If a newer chart is found that matches the `semver: 1.x` constraint, Flux will upgrade the release accordingly.

In **infrastructure/configs/** dir we have Kubernetes custom resources, such as the Let's Encrypt issuer:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    # Replace the email address with your own contact email
    email: fluxcdbot@users.noreply.github.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-nginx
    solvers:
      - http01:
          ingress:
            class: nginx
```

In **clusters/production/infrastructure.yaml** we replace the Let's Encrypt server value to point to the production API:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infra-configs
  namespace: flux-system
spec:
  # ...omitted for brevity
  dependsOn:
    - name: infra-controllers
  patches:
    - patch: |
        - op: replace
          path: /spec/acme/server
          value: https://acme-v02.api.letsencrypt.org/directory
      target:
        kind: ClusterIssuer
        name: letsencrypt
```

Note that with `dependsOn` we tell Flux to first install or upgrade the controllers and only then the configs.
This ensures that the Kubernetes CRDs are registered on the cluster, before Flux applies any custom resources.

### Clusters

A cluster is configured inside its own directory under **clusters/** dir, containing:

- **artifacts.yaml** contains an `ArtifactGenerator` that splits the monorepo into infrastructure and apps artifacts
- **infrastructure.yaml** contains the Flux `Kustomization` definitions for reconciling the infrastructure controllers and configs
- **apps.yaml** contains the Flux `Kustomization` definition for reconciling the apps Kustomize overlay for the specific cluster

```
./clusters/
├── production
│   ├── apps.yaml
│   ├── artifacts.yaml
│   └── infrastructure.yaml
└── staging
    ├── apps.yaml
    ├── artifacts.yaml
    └── infrastructure.yaml
```

In **clusters/staging/** dir we have the Flux Kustomization definitions, for example:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  dependsOn:
    - name: infra-configs
  interval: 1h
  retryInterval: 2m
  timeout: 5m
  sourceRef:
    kind: ExternalArtifact
    name: apps
  path: ./staging
  prune: true
  wait: true
```

With `path: ./staging` we configure Flux to sync the apps staging Kustomize overlay and 
with `dependsOn` we tell Flux to wait for the infrastructure configs to be installed before applying the apps.

Note that the `ExternalArtifact` source is generated by the `ArtifactGenerator`
from the contents of the **apps/base** and **apps/staging** dirs.
The `ArtifactGenerator` allows us to split the monorepo into smaller artifacts that can be synced independently.
Changes to files outside the **apps/** dirs will not trigger a reconciliation of the apps Kustomization.

## Bootstrap

Fork this repository on your personal GitHub account and export your GitHub access token, username and repo name:

```sh
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
export GITHUB_REPO=<repository-name>
```

Verify that your staging cluster satisfies the prerequisites with:

```sh
flux check --pre
```

Set the kubectl context to your staging cluster and bootstrap Flux:

```sh
flux bootstrap github \
    --components-extra=source-watcher \
    --context=staging \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/staging
```

The bootstrap command commits the manifests for the Flux components in `clusters/staging/flux-system` dir
and creates a deploy key with read-only access on GitHub, so it can pull changes inside the cluster.

Watch for the Helm releases being installed on staging:

```console
$ watch flux get helmreleases --all-namespaces

NAMESPACE    	NAME         	REVISION	SUSPENDED	READY	MESSAGE 
cert-manager 	cert-manager 	1.19.1   	False    	True 	Helm install succeeded
ingress-nginx	ingress-nginx	4.13.4   	False    	True 	Helm install succeeded
podinfo      	podinfo      	6.9.2   	False    	True 	Helm install succeeded
```

Verify that the demo app can be accessed via ingress:

```console
$ kubectl -n ingress-nginx port-forward svc/ingress-nginx-controller 8080:80 &

$ curl -H "Host: podinfo.staging" http://localhost:8080
{
  "hostname": "podinfo-59489db7b5-lmwpn",
  "version": "6.9.2"
}
```

Bootstrap Flux on production by setting the context and path to your production cluster:

```sh
flux bootstrap github \
    --components-extra=source-watcher \
    --context=production \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/production
```

Watch the production reconciliation:

```console
$ flux get kustomizations --watch

NAME                    REVISION                    READY   MESSAGE
flux-system             main@sha1:a7be7dff          True    Applied revision: main@sha1:a7be7dff
infra-controllers       latest@sha256:c0ac3648      True    Applied revision: latest@sha256:c0ac3648
infra-configs           latest@sha256:c0ac3648      True    Applied revision: latest@sha256:c0ac3648
apps                    latest@sha256:26785ee4      True    Applied revision: latest@sha256:26785ee4
```

### Add clusters

If you want to add a cluster to your fleet, first clone your repo locally:

```sh
git clone https://github.com/${GITHUB_USER}/${GITHUB_REPO}.git
cd ${GITHUB_REPO}
```

Create a dir inside `clusters` with your cluster name:

```sh
mkdir -p clusters/dev
```

Copy the sync manifests from staging:

```sh
cp clusters/staging/artifacts.yaml clusters/dev
cp clusters/staging/infrastructure.yaml clusters/dev
cp clusters/staging/apps.yaml clusters/dev
```

You could create a dev overlay inside `apps`, make sure
to change the `spec.path` inside `clusters/dev/apps.yaml` to `path: ./apps/dev`. 

Push the changes to the main branch:

```sh
git add -A && git commit -m "add dev cluster" && git push
```

Set the kubectl context and path to your dev cluster and bootstrap Flux:

```sh
flux bootstrap github \
    --components-extra=source-watcher \
    --context=dev \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/dev
```

## Testing

Any change to the Kubernetes manifests or to the repository structure should be validated in CI before
a pull requests is merged into the main branch and synced on the cluster.

This repository contains the following GitHub CI workflows:

* the [test](./.github/workflows/test.yaml) workflow validates the Kubernetes manifests and Kustomize overlays with [kubeconform](https://github.com/yannh/kubeconform)
* the [e2e](./.github/workflows/e2e.yaml) workflow starts a Kubernetes cluster in CI and tests the staging setup by running Flux in Kubernetes Kind
