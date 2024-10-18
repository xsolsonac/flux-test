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
  --token-auth  \
  --owner=$GITHUB_USER  \
  --repository=flux-test  \
  --branch=main  \
  --path=clusters/dev  \
  --personal
```


