# doubledragon
Flux v2 scaffold

Kubernetes cluster configuration that uses GitOps to manage state.

Includes Flux, Helm, cert-manager, Nginx Ingress and Sealed Secrets.

* [fluxcd.io](https://fluxcd.io)
* [helm.sh](https://helm.sh)
* [github.com/jetstack/cert-manager](https://github.com/jetstack/cert-manager)
* [kubernetes.github.io/ingress-nginx/](https://kubernetes.github.io/ingress-nginx/)
* [github.com/bitnami-labs/sealed-secrets](https://github.com/bitnami-labs/sealed-secrets)
* [kustomize.io](https://kustomize.io)

![Double Dragon](https://static.wixstatic.com/media/cf1e64_4736286d1baa49ee99802212ada59dee~mv2.png/v1/fill/w_498,h_664,al_c,usm_0.66_1.00_0.01/cf1e64_4736286d1baa49ee99802212ada59dee~mv2.png "arcade!")

## Contributors

* Ivar Abrahamsen : [@flurdy](https://twitter.com/flurdy) : [github.com/flurdy](https://github.com/flurdy) : [eray.uk](https://eray.uk)


## Flux version

* For a Flux v1 setup, please follow the older [Lemmings repository](https://github.com/flurdy/lemmings/)
* For a Flux v2 setup, please follow this, the [Double Dragon repository](https://github.com/flurdy/doubledragon/)

## Pre requisite

### Kubernetes tools and cluster

*     brew install kubectl
* Create Kubernetes cluster (see *Kubernetes as a Service providers* below)
* Set up Kubernetes context (see provider CLIs and `kubectx` below)
* Test cluster connection:

      kubectl cluster-info



### Github token

Flux uses your [Github Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) to access your repos.
If you need to create a new one you need to make sure it has the required accesses ticked. For example Flux will need to access to the deploy key for the repo which require *Admin* access.

In your dotfiles make sure you expose it as `GITHUB_TOKEN`.
At the same time set the `GITHUB_USER` env-var to your github username.
(Nudge: [direnv](https://direnv.net/))

### Github CLI

This is optional. Instead you can do this manually via the github website. But the examples for clarity uses the [CLI](https://cli.github.com/).


      brew install gh;
      gh auth login

Note, the CLI for some reason does not like if Github PAT env-var is set so you may have to temporarily unset when using it.

     # In Bash:
     unset GITHUB_TOKEN
     # In Fish:
     set -e GITHUB_TOKEN

Make sure you set the `GITHUB_TOKEN` env-var again afterwards.
## Double Dragon install

### Fork/Clone repository

* Initialize an empty Double Dragon repo for your setup.

      git clone git@github.com:flurdy/doubledragon.git \
         doubledragon-fleet;
      cd doubledragon-fleet;
      rm -rf .git;
      git init;
      git add README.md LICENSE;
      git commit -m "Starting our double dragon fleet";

* And push it to github as a private repo

      gh repo create --private doubledragon-fleet -r origin;
      git push -u origin main

* Replace _doubledragon-fleet_ with whatever you want to call your repository

Note, Flux can also talk to Bitbucket, Gitlab, Github Enterprise and self-hosted git repositories

### Install Flux CLI

*
      brew install fluxcd/tap/flux

* Test if Flux ir ready to be installed on your cluster

      flux check --pre

### Bootstrap Flux on your cluster

*
      flux bootstrap github \
        --owner=$GITHUB_USER \
        --repository=doubledragon-fleet \
        --branch=main \
        --path=./clusters/doubledragon-01 \
        --personal

* This assumes the repo is called _doubledragon-fleet_ (it will create it if it does not exist).
   And names your initial cluster as _doubledragon-01_.
   Change as appropriate

* Note previously I also included:

      --components-extra=image-reflector-controller,image-automation-controller \
   but that is under review if still needed for image scanning

* Update your local repo with the _origin_ changes

      git pull

### File structure

Unlike Flux v1 which was a simpler one repo per cluster.
Flux v2 is more flexible with potentially many clusters per repo and more.

Flux v2 also prefer to use [Kustomize](https://kustomize.io),
but you do not have to use it.

Flux can act directly in your cluster,
but this setup does everything via config files and git.
That way we have a replayable paper trail.

So a Flux repo may look like this at the start:

    |-- apps
    |   |-- base
    |   |-- overlays
    |   |   |-- doubledragon
    |-- clusters
    |   |-- doubledragon-01
    |-- infrastructure
    |   |-- sources


* `apps/base` is where you define your apps; deployments, services, etc.
* `apps/overlays/clustername` is where you choose which apps a cluster has.
* `cluster/clustername` with links to apps and infrastructure active in your specific cluster.
* `infrastructure/sources` where to find images from registries, Helm repos, etc.

* Create some of these folders:

      mkdir -p apps/base;
      mkdir -p apps/overlays/doubledragon;
      mkdir -p infrastructure/sources

* Check the file structure

      tree

### Sealed Secrets

* Add a Helm repository for Sealed Secrets

      flux create source helm sealed-secrets \
        --interval=1h \
        --url=https://bitnami-labs.github.io/sealed-secrets \
        --export > infrastructure/sources/sealed-secrets-source.yaml

* Install Helm chart

      mkdir -p infrastructure/sealed-secrets;

      flux create helmrelease sealed-secrets \
        --interval=1h \
        --release-name=sealed-secrets-controller \
        --target-namespace=flux-system \
        --source=HelmRepository/sealed-secrets \
        --chart=sealed-secrets \
        --chart-version=">=1.15.0-0" \
        --crds=CreateReplace \
        --export > infrastructure/sealed-secrets/sealed-secrets.yaml;

      touch infrastructure/sealed-secrets/kustomization.yaml

* Push to git so Flux can act on it

      git add infrastructure/sources/sealed-secrets-source.yaml \
        infrastructure/sealed-secrets/sealed-secrets.yaml \
        infrastructure/sealed-secrets/kustomization.yaml;
      git commit -m "Added Sealed Secrets";
      git push

* Install CLI

      brew install kubeseal

* Retrieve public key

      mkdir /clusters/my-doubledragon-01/secrets;

      kubeseal --fetch-cert \
        --controller-name=sealed-secrets-controller \
        --controller-namespace=flux-system \
        > /clusters/my-doubledragon-01/secrets/pub-sealed-secrets.pem

  * Some cluster setups may block access to your sealed-secrets-controller,
e.g. a GKE cluster.

     So instead we can temporarily proxy that locally like this:

        kubectl --namespace flux-system port-forward \
          service/sealed-secrets-controller 8081:8080

  * And use `curl` to download the certificate instead:

        curl localhost:8081/v1/cert.pem \
          > /clusters/my-doubledragon-01/secrets/pub-sealed-secrets.pem

* Add it to source control

      git add clusters/my-doubledragon-01/secrets/pub-sealed-secrets.pem

### Nginx Ingress

* Add a Helm repository

      flux create source helm sealed-secrets \
        --interval=1h \
        --url=https://kubernetes.github.io/ingress-nginx \
        --export > infrastructure/sources/ingress-nginx-source.yaml

* Install Helm chart

      mkdir -p infrastructure/ingress-nginx;

      flux create helmrelease ingress-nginx \
        --interval=1h \
        --release-name=ingress-nginx \
        --target-namespace=flux-system \
        --source=HelmRepository/ingress-nginx-source \
        --chart=ingress-nginx \
        --chart-version=">=1.0-4" \
        --crds=CreateReplace \
        --export > infrastructure/ingress-nginx/ingress-nginx.yaml;



### Cert Manager

* Coming soon...
### Container Registries

To access private Docker container image repositories
we need to setup some more sources, image sources.
And some secrets to access those.


* [flurdy's 'kubernetes-docker-registry.html](https://flurdy.com/docs/kubernetes/registry/kubernetes-docker-registry.html)



* Create



__Your GitOps based Kubernetes cluster is live!__

If you waited a few minutes then your GitOps configured Kubernetes cluster is now live.

But with no apps running.

## Your first application

### Launch application

### Test application

* Find the ingress controller's `External IP`

      kubectl get services nginx-ingress-controller

* Use `curl` to resolve the URL. Replace `11.22.33.44` with the external IP.
* And `lynx` to view it

      curl -H "Host: hello.example.com" \
        --resolve hello.example.com:80:11.22.33.44 \
        --resolve hello.example.com:443:11.22.33.44 \
        http://hello.example.com | lynx -stdin
* This should show a basic hello world page, with an Nginx logo and some server address, name and date details.


### Update application

### Delete application

## Go wild

* Add/update your deployments, services, charts, docker registries, secrets, etc
## Advise: Don't touch

* Once Flux is running, by convention avoid using `kubectl create|apply` etc.
* Nearly all changes should be via Git and Flux.
* Any `kubectl` interaction should be read only.

## More information, alternatives, suggestions

* Kubernetes as a Service
  * Amazon AWS EKS: [aws.amazon.com/eks/](https://aws.amazon.com/eks/)
  * Google Cloud GKE: [cloud.google.com/kubernetes-engine/](https://cloud.google.com/kubernetes-engine/)
  * Microsoft Azure AKS: [azure.microsoft.com/en-us/services/kubernetes-service/](https://azure.microsoft.com/en-us/services/kubernetes-service/)
  * DigitalOcean Kubernetes: [www.digitalocean.com/products/kubernetes/](https://www.digitalocean.com/products/kubernetes/)
* Cloud provider CLI
  * [cloud.google.com/sdk/](https://cloud.google.com/sdk/)

        brew cask install google-cloud-sdk

  * [github.com/digitalocean/doctl](https://github.com/digitalocean/doctl)

        brew install doctl
  * [eksctl.io](https://eksctl.io/)

        brew tap weaveworks/tap
        brew install weaveworks/tap/eksctl

  * [github.com/Azure/azure-cli](https://github.com/Azure/azure-cli)

        brew install azure-cli
* Tools
  * [github.com/ahmetb/kubectx](https://github.com/ahmetb/kubectx)

        brew install kubectx

   * [github.com/vmware-tanzu/octant](https://github.com/vmware-tanzu/octant)

         brew install octant

   * [k9ss.io](https://k9ss.io)

         brew install derailed/k9s/k9s

   * [keel.sh](https://keel.sh)

* [flurdy.com/docs/kubernetes/registry/kubernetes-docker-registry.html](https://flurdy.com/docs/kubernetes/registry/kubernetes-docker-registry.html)
* [ramitsurana.github.io/awesome-kubernetes/](https://ramitsurana.github.io/awesome-kubernetes/)

### Notes:

* Certain operation takes a few minutes, e.g. pod creation.
* Client tools are also available on Linux, Windows and more.



## Versions

* 2021-07-10 Flux 2. Lemmings => Double Dragon
* 2020-02-13 Flux 1.1, fluxcd.io annotations, and Helm 3
* 2019-11-07 Flux 0.16, flux.weave.works annotations and Helm 2