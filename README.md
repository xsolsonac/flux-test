# Flux

 * [Documentation](https://fluxcd.io/flux/)
 * [Github repository](https://github.com/fluxcd/flux2?tab=readme-ov-file)

## Install

 ```bash
 curl -s https://fluxcd.io/install.sh | sudo bash

# To configure your shell to load flux bash completions add to your profile:
. <(flux completion bash)
 ```

# flux-test

Creem un repositori git (buit) https://github.com/xsolsonac/flux-test

```bash
# preparem variables entorn GITHUB
export GITHUB_USER=xsolsonac

# flux bootstrap
# The flux bootstrap github command deploys the Flux controllers on a Kubernetes cluster and configures the controllers to sync the cluster state from a GitHub repository

flux bootstrap github --help

flux bootstrap github \
  --context=kind-k8s-localtest  \
  --components-extra=image-reflector-controller,image-automation-controller \
  --token-auth  \
  --owner=$GITHUB_USER  \
  --repository=flux-test  \
  --branch=main  \
  --path=clusters/dev  \
  --personal
```

Afegim el repositori on es troba l'aplicació a Flux, usem podinfo d'exemple

```bash
# Create a GitRepository manifest pointing to podinfo repository’s master branch:
flux create source git podinfo \
  --url=https://github.com/stefanprodan/podinfo \
  --branch=master \
  --interval=1m \
  --export > ./clusters/dev/podinfo-source.yaml
```

Afegim també la kustomització i apliquem al k8s:

```bash
kubectl apply -f podinfo-source.yaml
kubectl apply -f podinfo-kustomization.yaml
```


# Desplegament via HelmRelease i HelmRepository

* Disposem d'un repositori Helm a `https://xsolsonac.github.io/tercers-api` a [Github Pages](https://pages.github.com/). El repositori ha de contenir els fitxers .tgz i un index.yaml

```bash
# Creació dels fitxers .tgz a partir del Chart de Helm
helm package <chart_name/>

# Actualització de l'index.yaml
helm repo index .
```
* Creem manifest amb components **HelmRelease** i **HelmRepository** a `tercers-api-chart-helmrepo.yaml`
  * El *HelmRepository* apunta al repositori helm creat
  * El *HelmRelease* usa el chart <chart_name/> del *HelmRepository*

```bash
cat > tercers-api-chart-helmrepo.yaml <<- EOM
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: tercers-api
  namespace: default
spec:
  interval: 1m
  url: https://xsolsonac.github.io/tercers-api
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: tercers-api
  namespace: default
spec:
  interval: 5m
  timeout: 2m
  chart:
    spec:
      chart: hercules-udl-mi
      version: '1.0.*'
      sourceRef:
        kind: HelmRepository
        name: tercers-api
      interval: 5m
  releaseName: hercules-udl-mi
  install:
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
  driftDetection:
    mode: enabled
    ignore:
    - paths: ["/spec/replicas"]
      target:
        kind: Deployment
  values:
    replicaCount: 1
EOM
```

* Apliquem manifest al k8s i comprovem disponibilitat

```bash
# apliquem instruccions de deploy del chart
kubectl apply -f tercers-api-chart-helmrepo.yaml

# comprovem disponibilitat del chart
kubectl get helmrepository
kubectl get helmrelease

kubectl describe helmrelease tercers-api
```


# Desplegament via HelmRelease i GitRepository

* Disposem d'un repositori a Github amb un Helm Chart, per exemple el repositori `sicudl/hercules-udl-mi` amb la carpeta `helm`.


* Creem manifest amb components **HelmRelease** i **GitRepository** a `tercers-api-chart-gitrepo.yaml`
  * El *GitRepository* apunta a la carpeta *helm* del repositori de Github `sicudl/hercules-udl-mi`, com que és un repositori privat necessirem autenticació en aquest cas amb un autenticació bàsica amb PAT.
    * [flux create secret git](https://fluxcd.io/flux/cmd/flux_create_secret_git/)
    * [gitRepository authentication](https://fluxcd.io/flux/components/source/gitrepositories/#basic-access-authentication)
  * El *HelmRelease* usa el chart del *HelmRepository*


```bash

# Creem un create un Secret a k8s per a poder-lo usar posteriorment en l'autenticació al GitRepository via PAT

export GITHUB_USER=xxxxxxxx
export GITHUB_TOKEN=xxxxxxxxxx
flux -n default create secret git hercules-udl-git-auth \
--url=https://github.com/sicudl/hercules-udl-mi.git \
--username=$GITHUB_USER \
--password=$GITHUB_TOKEN

cat > tercers-api-chart-gitrepo.yaml <<- EOM
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: tercers-api-gitrepo
  namespace: default
spec:
  secretRef:
    name: hercules-udl-git-auth
  interval: 1m
  url: https://github.com/sicudl/hercules-udl-mi.git
  ref:
    branch: master
  ignore: |
    # exclude all
    /*
    # include charts directory
    !/helm/
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: tercers-api-gitrepo
  namespace: default
spec:
  interval: 5m
  timeout: 2m
  chart:
    spec:
      chart: helm/hercules-udl-mi
      version: '1.0.*'
      sourceRef:
        kind: GitRepository
        name: tercers-api-gitrepo
      interval: 5m
  releaseName: hercules-udl-mi
  install:
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
  driftDetection:
    mode: enabled
    ignore:
    - paths: ["/spec/replicas"]
      target:
        kind: Deployment
  values:
    replicaCount: 1
EOM
```

* Apliquem manifest al k8s i comprovem disponibilitat

```bash
# apliquem instruccions de deploy del chart
kubectl apply -f tercers-api-chart-gitrepo.yaml

# comprovem disponibilitat del chart i del gitrepo
kubectl get gitrepository
kubectl get helmrelease

kubectl describe helmrelease tercers-api-gitrepo

# forcem reconcile per
flux reconcile hr tercers-api-gitrepo -n default --with-source --force
```

# Creació estructura monorepo per a gestionar configuracions dels diversos entorns

* [Flux - Ways of structuring your repositories](https://fluxcd.io/flux/guides/repository-structure/)

Estructura similar a la següent

```bash
├── apps
│   ├── base
│   ├── production
│   └── staging
├── infrastructure
│   ├── base
│   ├── production
│   └── staging
└── clusters
    ├── production
    └── staging
```