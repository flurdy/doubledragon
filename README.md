# Double Dragon
Flux v2 scaffold

Kubernetes cluster configuration that uses GitOps to manage state.

Includes Flux, Helm, cert-manager, Nginx Ingress and Sealed Secrets.

* [fluxcd.io](https://fluxcd.io)
* [helm.sh](https://helm.sh)
* [kubernetes.github.io/ingress-nginx/](https://kubernetes.github.io/ingress-nginx/)
* [github.com/bitnami-labs/sealed-secrets](https://github.com/bitnami-labs/sealed-secrets)
* [github.com/jetstack/cert-manager](https://github.com/jetstack/cert-manager)
* [kustomize.io](https://kustomize.io)

![Double Dragon](https://static.wixstatic.com/media/cf1e64_4736286d1baa49ee99802212ada59dee~mv2.png/v1/fill/w_498,h_664,al_c,usm_0.66_1.00_0.01/cf1e64_4736286d1baa49ee99802212ada59dee~mv2.png "arcade!")

### Contributors

* Ivar Abrahamsen : [@flurdy](https://twitter.com/flurdy) : [github.com/flurdy](https://github.com/flurdy) : [eray.uk](https://eray.uk)


### Flux version

* For a Flux v1 setup, please follow the older [Lemmings repository](https://github.com/flurdy/lemmings/)
* For a Flux v2 setup, please follow this, the [Double Dragon repository](https://github.com/flurdy/doubledragon/)

### Contents

1. [Introduction](#doubledragon)
1. [Pre requisites](#contributors)
1. [Double Dragon install](#double-dragon-install)
   1. [Clone repository](#forkclone-repository)
   1. [Cluster environment variables](#cluster-environment-variables)
   1. [Install Flux CLI](#install-flux-cli)
   1. [Bootstrap Flux](#bootstrap-flux-on-your-cluster)
   1. [File structure](#file-structure)
   1. [Namespaces](#namespaces)
   1. [Sealed Secrets](#sealed-secrets)
   1. [Nginx Ingress](#nginx-ingress)
   1. [Cert Manager](#cert-manager)
   1. [Container registries](#container-registries)
1. [Applications](#applications)
   1. [Hello world application](#hello-world-application)
   1. [Your first application](#your-first-application)
   1. [Automatic image updates](#automatic-image-updates)
1. [Go wild](#go-wild)
1. [Further information](#further-information)
   1. [Advise: Don't touch](#advise-dont-touch)
   1. [Troubleshooting](#troubleshooting)
   1. [Add another cluster](#add-another-cluster)
   1. [Providers and tools](#providers-tools)
   1. [License](#license)

## Pre requisites

### Kubernetes tools and cluster

*     brew install kubectl
* Create Kubernetes cluster (see [Kubernetes as a Service providers](#Kubernetes-as-a-Service) below)
* Set up Kubernetes context (see [provider CLIs](#Kubernetes-as-a-Service) and [kubectx CLI](#Kubernetes-as-a-Service)  below)
* Test cluster connection:

      kubectl cluster-info



### Github token

Flux uses your [Github Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) to access your repos.
If you need to create a new one you need to make sure it has the required accesses ticked. For example Flux will need to access to the deploy key for the repo which require *Admin* access.

In your dotfiles make sure you expose it as `GITHUB_TOKEN`.
At the same time set the `GITHUB_USER` env-var to your github username.
(Nudge: [direnv](https://direnv.net/))

## Double Dragon install

---
### Fork/Clone repository

* Initialize an empty Double Dragon repo for your setup.

      git clone git@github.com:flurdy/doubledragon.git;
      mkdir doubledragon-fleet;
      cp doubledragon/README.md doubledragon-fleet/;
      cp doubledragon/LICENSE doubledragon-fleet/;
      cd doubledragon-fleet;
      git init;
      git add README.md LICENSE;
      git commit -m "Starting our double dragon fleet";

  Replace _doubledragon-fleet_ with whatever you want to call your repository.

  You may wish use the original `doubledragon` repo to compare.

* Create a private github repository

  Manually create a private `doubledragon-fleet` repo via [github.com](https://github.com)
  or with the [Github CLI](https://cli.github.com/)

      brew install gh;
      gh auth login;
      gh repo create --private doubledragon-fleet -r origin -s .

  Note, the CLI for some reason does not like if Github PAT env-var is set so you may have to temporarily unset when using it.

   In Bash:

      unset GITHUB_TOKEN

   In Fish:

      set -e GITHUB_TOKEN

  Make sure you set the `GITHUB_TOKEN` env-var again afterwards.

* And push your local repo to the github repo

      git push -u origin main

* Edit the `README.md` as you see fit.

* Edit the `LICENSE` as you see fit.

* Note, Flux can also talk to Bitbucket, Gitlab, Github Enterprise and self-hosted git repositories

### Cluster environment variables

* Lets temporarily add the repo and cluster name as environment variables so that most commands in this howto can be copy-pasted directly

  In Bash:

      export DOUBLEDRAGON_REPO=doubledragon-fleet;
      export DOUBLEDRAGON_NAME=doubledragon;
      export DOUBLEDRAGON_CLUSTER=doubledragon-01

  In Fish:

      set -x DOUBLEDRAGON_REPO doubledragon-fleet;
      set -x DOUBLEDRAGON_NAME doubledragon;
      set -x DOUBLEDRAGON_CLUSTER doubledragon-01

  Replace _doubledragon-fleet, doubledragon, doubledragon-01_ with whatever you decide to call your repository and cluster

### Install Flux CLI

*
      brew install fluxcd/tap/flux

* Test if Flux ir ready to be installed on your cluster

      flux check --pre

### Bootstrap Flux on your cluster

    flux bootstrap github \
	   --token-auth=false \
      --components-extra=image-reflector-controller,image-automation-controller \
      --owner=$GITHUB_USER \
      --repository=$DOUBLEDRAGON_REPO \
      --branch=main \
      --path=./clusters/$DOUBLEDRAGON_CLUSTER \
      --read-write-key=true \
      --personal

* This will create a github repo set in `$DOUBLEDRAGON_REPO` if it does not already exist.
   And names your initial cluster as set in `$DOUBLEDRAGON_CLUSTER`.

* Update your local repo with the _origin_ changes

      git pull

### File structure

Unlike Flux v1 which was a simpler one repo per cluster,
Flux v2 is more flexible with potentially many clusters per repo and more abstractions if desired.

Flux v2 also prefer to use [Kustomize](https://kustomize.io) for templating,
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
* `apps/overlays/doubledragon` is where you choose which apps a cluster has,
  and any customization specific to that cluster.
* `cluster/doubledragon-01` with links to apps and infrastructure active in your specific cluster.
* `infrastructure/sources` where to find images from registries, Helm repos, etc.

* Create some of these folders:

      mkdir -p apps/base;
      mkdir -p apps/overlays/$DOUBLEDRAGON_NAME;
      mkdir -p infrastructure/sources

* Check the file structure

      tree


### Namespaces

* Lets separate some of our resources into two namespaces.

  You may go with further specific namespaces if you prefer.

  (For some reason _kustomize_ does not let you add several namespaces
  in one _kustomization_ so we will add plain files to the cluster)

* Edit `clusters/$DOUBLEDRAGON_CLUSTER/namespaces.yaml`

      apiVersion: v1
      kind: Namespace
      metadata:
        name: infrastructure
      ---
      apiVersion: v1
      kind: Namespace
      metadata:
        name: apps

* Push to _Flux_

      git add clusters/$DOUBLEDRAGON_CLUSTER/namespaces.yaml;
      git commit -m "Namespaces";
      git push

### Sealed Secrets

Safely store encrypted secrets in the git repository.

There are several alternative encrypted secrets solutions, such as [Mozilla's SOPS](https://fluxcd.io/flux/guides/mozilla-sops/),
but Sealed Secrets works well for me.

* Add a Helm repository for Sealed Secrets

      flux create source helm sealed-secrets-source \
        --interval=1h \
        --namespace=infrastructure \
        --url=https://bitnami-labs.github.io/sealed-secrets \
        --export > infrastructure/sources/sealed-secrets-source.yaml

* Install Helm chart

      mkdir -p infrastructure/sealed-secrets;

      flux create helmrelease sealed-secrets \
        --interval=1h \
        --release-name=sealed-secrets-controller \
        --target-namespace=infrastructure \
        --source=HelmRepository/sealed-secrets-source \
        --chart=sealed-secrets \
        --chart-version=">=1.15.0-0" \
        --crds=CreateReplace \
        --export > infrastructure/sealed-secrets/sealed-secrets.yaml

* Add to git so Flux can act on it

      git add infrastructure/sources/sealed-secrets-source.yaml \
        infrastructure/sealed-secrets/sealed-secrets.yaml;
      git commit -m "Added Sealed Secrets"

* Next we need to add simple _Kustomization_ files that activates Sealed Secrets for our cluster.
These will be very simple for now, later on they will be more elaborate and helpful


* Edit   `infrastructure/sealed-secrets/kustomization.yaml`

      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization
      resources:
      - sealed-secrets.yaml

* Edit   `infrastructure/sources/kustomization.yaml`

      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization
      resources:
      - sealed-secrets-source.yaml

* Append to `infrastructure/kustomization.yaml`

      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization
      resources:
      - sources
      - sealed-secrets

* Create a pollable link in our cluster for all _infrastructure_:

      flux create kustomization infrastructure \
        --target-namespace=infrastructure \
        --source=flux-system \
        --path="./infrastructure" \
        --prune=true \
        --interval=10m \
        --export > clusters/$DOUBLEDRAGON_CLUSTER/infrastructure.yaml


* Push to git

      git add infrastructure/sources/kustomization.yaml;
      git add infrastructure/sealed-secrets/kustomization.yaml;
      git add infrastructure/kustomization.yaml;
      git add clusters/$DOUBLEDRAGON_CLUSTER/infrastructure.yaml;
      git commit -m "Activated Sealed Secrets";
      git push

* Flux should pick this up and install the Helm chart for Sealed Secrets

#### Using Sealed Secrets

* Install `kubeseal` CLI

      brew install kubeseal

* Retrieve public key from this cluster

      mkdir -p clusters/$DOUBLEDRAGON_CLUSTER/secrets;

      kubeseal --fetch-cert \
        --controller-name=sealed-secrets-controller \
        --controller-namespace=infrastructure \
        > clusters/$DOUBLEDRAGON_CLUSTER/secrets/sealed-secrets-cert.pem

  * Some cluster setups may block access to your sealed-secrets-controller,
e.g. a GKE cluster.

     So instead we can temporarily proxy that locally like this:

        kubectl --namespace infrastructure port-forward \
          service/sealed-secrets-controller 8081:8080

  * And use `curl` to download the certificate instead:

        curl localhost:8081/v1/cert.pem \
          > clusters/$DOUBLEDRAGON_CLUSTER/secrets/sealed-secrets-cert.pem

* Add it to source control

      git add clusters/$DOUBLEDRAGON_CLUSTER/secrets/sealed-secrets-cert.pem;
      git commit -m "Sealed Secret public key";
      git push

### Nginx Ingress

* Add a Helm repository

      flux create source helm ingress-nginx-source \
        --interval=1h \
        --namespace=infrastructure \
        --url=https://kubernetes.github.io/ingress-nginx \
        --export > infrastructure/sources/ingress-nginx-source.yaml

* Append it to the exiting sources kustomization `infrastructure/sources/kustomization.yaml`

      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization
      resources:
      - sealed-secrets-source.yaml
      - ingress-nginx-source.yaml

* Install the ingress controller with the Helm chart

      mkdir -p infrastructure/ingress-nginx;

      flux create helmrelease ingress-nginx \
        --interval=1h \
        --release-name=ingress-nginx \
        --target-namespace=apps \
        --namespace=infrastructure \
        --source=HelmRepository/ingress-nginx-source \
        --chart=ingress-nginx \
        --chart-version=">=1.0-4" \
        --crds=CreateReplace \
        --export > infrastructure/ingress-nginx/ingress-nginx.yaml

* Add kustomization `infrastructure/ingress-nginx/kustomization.yaml`

      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization
      resources:
      - ingress-nginx.yaml


* Append it to `infrastructure/kustomization.yaml`

      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization
      resources:
      - sources
      - sealed-secrets
      - ingress-nginx

* Add to git and push

      git add infrastructure/sources/ingress-nginx-source.yaml;
      git add infrastructure/sources/kustomization.yaml;
      git add infrastructure/ingress-nginx/ingress-nginx.yaml;
      git add infrastructure/ingress-nginx/kustomization.yaml;
      git add infrastructure/kustomization.yaml;
      git commit -m "Nginx Ingress";
      git push

### Cert manager
#### Install cert-manager

* Add a Jetstack source repo

      flux create source helm jetstack-source \
        --interval=1h \
        --namespace=infrastructure \
        --url=https://charts.jetstack.io \
        --export > infrastructure/sources/jetstack-source.yaml

  * Append it to the exiting sources kustomization `infrastructure/sources/kustomization.yaml`

        apiVersion: kustomize.config.k8s.io/v1beta1
        kind: Kustomization
        resources:
        - sealed-secrets-source.yaml
        - ingress-nginx-source.yaml
        - jetstack-source.yaml

* Define Cert Manager Helm properties


      mkdir infrastructure/cert-manager;

  * Edit `infrastructure/cert-manager/values.yaml`

        crds:
          enabled: true
        

* Install Cert Manager Helm

      flux create helmrelease cert-manager \
        --chart=cert-manager \
        --source=HelmRepository/jetstack-source.infrastructure \
        --release-name=cert-manager \
        --namespace=infrastructure \  
        --target-namespace cert-manager \
        --create-target-namespace \
        --chart-version=">=1.12.0" \
        --interval=1h \
        --values=infrastructure/cert-manager/values.yaml \
        --export > infrastructure/cert-manager/cert-manager.yaml


  * Create kustomization `infrastructure/cert-manager/kustomization.yaml`

        apiVersion: kustomize.config.k8s.io/v1beta1
        kind: Kustomization
        resources:
        - cert-manager.yaml

  * Append it to infrastructure kustomization `infrastructure/kustomization.yaml`

        apiVersion: kustomize.config.k8s.io/v1beta1
        kind: Kustomization
        resources:
        - sources
        - sealed-secrets
        - ingress-nginx
        - cert-manager

* Add to repo and push

      git add infrastructure/sources/jetstack-source.yaml;
      git add infrastructure/sources/kustomization.yaml;
      git add infrastructure/cert-manager/cert-manager.yaml;
      git add infrastructure/cert-manager/kustomization.yaml;
      git add infrastructure/kustomization.yaml;
      git commit -m "Cert-manager";
      git push

* Verify Cert manager works

  * Install the cert-manager CLI

    Optional but handy

        brew install cmctl

  * Verify

        cmctl check api

    Hopefully that will return "`The cert-manager API is ready`"


#### Certificate issuers

Lets create a _staging_ and _production_ certificate issuers with [Lets Encrypt](https://letsencrypt.org/), so that testing in _staging_ does not flood the _prod_ instance.

    mkdir -p clusters/$DOUBLEDRAGON_CLUSTER/certificate-issuers

* Create and edit the staging issuer at

  `clusters/$DOUBLEDRAGON_CLUSTER/certificate-issuers/letsencrypt-issuer-staging.yaml`

      apiVersion: cert-manager.io/v1
      kind: ClusterIssuer
      metadata:
        name: letsencrypt-staging
      spec:
        acme:
          server: https://acme-staging-v02.api.letsencrypt.org/directory
          email: youremail@example.com
          privateKeySecretRef:
            name: letsencrypt-staging-secret
          solvers:
          - http01:
              ingress:
                class: nginx

* Replace `youremail@example.com` with an email address you have access to

* Add to _flux_ and watch till active

      git add clusters/$DOUBLEDRAGON_CLUSTER/certificate-issuers/letsencrypt-issuer-staging.yaml;
      git commit -m "Staging issuer";
      git push;
      kubectl get clusterissuer -A --watch

* Secure an app

  I.e. add a TLS certificate to an ingress.

  > [!NOTE] 
  > This step may have to wait until you add your own apps later on in the tutorial.

  * Edit your app's _ingress_ `apps/base/someapp/ingress.yaml`

    Add the annotation and _tls_ sections

        apiVersion: networking.k8s.io/v1
        kind: Ingress
        metadata:
          annotations:
            cert-manager.io/cluster-issuer: letsencrypt-staging
          name: someapp-ingress
          namespace: apps
        spec:
          rules:
          - host: someapp.example.com
            http:
              paths:
              - pathType: Prefix
                path: /
                backend:
                  service:
                    name: someapp-service
                    port:
                      number: 80
          tls:
          - hosts:
            - someapp.example.com
            secretName: someapp-cert-staging

  * Add to git

        git add apps/base/someapp/ingress.yaml;
        git commit -m "Secured someapp";
        git push;
        kubectl get ingress -n apps --watch

    Soon the `someapp.example.com` line will show 443 as available port. It all ok.

    Note, your browser will throw a warning when accessing this site
    as the certificate for staging is not signed. Unlike prod.

* Now lets add a prod issuer, create and edit

  `clusters/$DOUBLEDRAGON_CLUSTER/certificate-issuers/letsencrypt-issuer-prod.yaml`

      apiVersion: cert-manager.io/v1
      kind: ClusterIssuer
      metadata:
        name: letsencrypt-prod
      spec:
        acme:
          server: https://acme-v02.api.letsencrypt.org/directory
          email: youremail@example.com
          privateKeySecretRef:
            name: letsencrypt-prod-secret
          solvers:
          - http01:
              ingress:
                class: nginx

* Add to _flux_ and watch till active

      git add clusters/$DOUBLEDRAGON_CLUSTER/certificate-issuers/letsencrypt-issuer-prod.yaml;
      git commit -m "Prod issuer";
      git push;
      kubectl get clusterissuer -A --watch

* Update the certificate for your app

  > [!NOTE] 
  > Again, only after you have added your own apps later on in the tutorial.

  Change the `cluster-issuer` annotation and `secretName` in

  `apps/base/someapp/ingress.yaml`

      apiVersion: networking.k8s.io/v1
      kind: Ingress
      metadata:
        annotations:
          cert-manager.io/cluster-issuer: letsencrypt-prod
        name: someapp-ingress
        namespace: apps
      spec:
        rules:
        - host: someapp.example.com
          http:
            paths:
            - pathType: Prefix
              path: /
              backend:
                service:
                  name: someapp-service
                  port:
                    number: 80
        tls:
        - hosts:
          - someapp.example.com
          secretName: someapp-cert-prod

  * Push and check when the certificate change goes live

        git add apps/base/someapp/ingress.yaml;
        git commit -m "Secured someapp with prod cert";
        git push;
        kubectl describe ingress someapp-ingress -n apps --watch
        
### Container Registries

To access private Docker container image repositories
we need to setup some more sources, image sources.
And some secrets to access those.


* Please follow [flurdy's 'kubernetes-docker-registry guide'](https://flurdy.com/docs/kubernetes/registry/kubernetes-docker-registry.html) for your relevant registries.

#### GCR Google Container Registry

>[!WARNING]
> GCR has been deprecated and replaced by Google Artifact Registry.
> This is only as an example of how you would seal and add any registry to your cluster

* For example if you needed [GCR](https://cloud.google.com/container-registry/),
  and have followed the guide above and got a `gcr-registry.yml` file (or `.yaml`),

  and maybe the initial `gcp-service-account.json` source as well.

* Make sure the raw secrets do not get added to *git* by accident

      echo gcp-service-account.json >> .gitignore;
      echo gcr-registry.yml >> .gitignore;
      git add .gitignore

* Seal the secrets

      mkdir -p clusters/$DOUBLEDRAGON_CLUSTER/registries/apps;
      mkdir -p clusters/$DOUBLEDRAGON_CLUSTER/registries/infrastructure

  A registry secret for both the _apps_ namespace

      kubeseal --format=yaml --namespace=apps \
      --cert=clusters/$DOUBLEDRAGON_CLUSTER/secrets/sealed-secrets-cert.pem \
      < gcr-registry.yml \
      > clusters/$DOUBLEDRAGON_CLUSTER/registries/apps/sealed-gcr-registry.yaml

   and the _infrastructure_ namespace

      kubeseal --format=yaml --namespace=infrastructure \
      --cert=clusters/$DOUBLEDRAGON_CLUSTER/secrets/sealed-secrets-cert.pem \
      < gcr-registry.yml \
      > clusters/$DOUBLEDRAGON_CLUSTER/registries/infrastructure/sealed-gcr-registry.yaml

* Add the secrets to Flux

      git add clusters/$DOUBLEDRAGON_CLUSTER/registries/apps/sealed-gcr-registry.yaml;
      git add clusters/$DOUBLEDRAGON_CLUSTER/registries/infrastructure/sealed-gcr-registry.yaml;
      git commit -m "GCR registry";
      git push

   You may later need more for other and future namespaces, e.g. `default` and `flux-system`

* Remove `gcr-registry.yml` (and `gcp-service-account.json`)

  Later on when you have tested the registry by confirming that the cluster can download actual deployment images for your apps,
  you should delete the unencrypted registry files

      rm gcr-registry.yml gcp-service-account.json


* Lets set up repo image scanning

  To check when a new repo tag and image has been added to a registry.

  For example if you have an app that stores its images in a private repo like _GCR_.

  Otherwise you can wait to do this step later.

      flux create image repository someapp-source \
      --image=ghcr.io/someorg/someuser/somerepo \
      --interval=5m \
      --namespace=infrastructure \
      --secret-ref=gcr-registry \
      --export > infrastructure/sources/someapp-source.yaml

  * (Change _somerepo_ to your app name.
  And use the correct GCR image path.)

  * This example refers to the `gcr-registry` sealed secret

  * Append this to the YAML in `infrastructure/sources/someapp-source.yaml`:

        accessFrom:
          namespaceSelectors:
            - matchLabels:
                kubernetes.io/metadata.name: apps

    Note, indentation is under `spec`.

* Note, for _GCR_ there is the alternative option of a more secure short-lived _acccess token_ instead.

  This can be done with Flux. [You need to set up a cronjob to refresh it](https://fluxcd.io/flux/guides/cron-job-image-auth/).

* Add these to `infrastructure/sources/kustomization.yaml`

      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization
      resources:
      ...
      - someapp-source.yaml

* Add to git/flux

      git add infrastructure/sources/someapp-source.yaml;
      git add infrastructure/sources/kustomization.yaml;
      git commit -m "Sources for someapp"
      git push

## Applications

---
## Hello world application

Lets create a Hello World app.

### Base layer

* First lets create a _base layer_

      mkdir -p apps/base/hello

* And an initial deployment yaml for an _Hello_ app

      kubectl create deployment hello-deployment \
      --image=nginxdemos/hello:0.3 \
      --namespace=apps \
      --dry-run=client -o yaml \
      > apps/base/hello/deployment.yaml

* Lets prune the output a bit: `apps/base/hello/deployment.yaml`,
and change the app labels to just `hello`

      apiVersion: apps/v1
      kind: Deployment
      metadata:
        labels:
          app: hello
        name: hello-deployment
        namespace: apps
      spec:
        replicas: 1
        selector:
          matchLabels:
            app: hello
        template:
          metadata:
            labels:
              app: hello
          spec:
            containers:
              - image: nginxdemos/hello:0.3
                name: hello

* Add a service at `apps/base/hello/service.yaml`

      apiVersion: v1
      kind: Service
      metadata:
        name: hello-service
        namespace: apps
      spec:
        selector:
          app: hello
        ports:
          - protocol: TCP
            port: 80
            targetPort: 80

* And an ingress at `apps/base/hello/ingress.yaml`

      apiVersion: networking.k8s.io/v1
      kind: Ingress
      metadata:
        name: hello-ingress
        namespace: apps
        annotations:
          kubernetes.io/ingress.class: nginx
      spec:
        rules:
          - host: hello.example.com
            http:
              paths:
                - path: /
                  pathType: Prefix
                  backend:
                    service:
                      name: hello-service
                      port:
                        number: 80

* Bundle these in `apps/base/hello/kustomization.yaml`

      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization
      namespace: apps
      resources:
        - deployment.yaml
        - service.yaml
        - ingress.yaml

### Hello app overlay

     mkdir -p apps/overlays/$DOUBLEDRAGON_NAME/hello

* Edit `apps/overlays/$DOUBLEDRAGON_NAME/hello/kustomization.yaml`

  In more complicated apps this may have some overrides but for now very simple.

      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization
      namespace: apps
      bases:
        - ../../../base/hello


* Edit `apps/overlays/$DOUBLEDRAGON_NAME/kustomization.yaml`

      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization
      namespace: apps
      bases:
        - hello

* Add it all to the repo

      git add apps/base/hello/deployment.yaml;
      git add apps/base/hello/service.yaml;
      git add apps/base/hello/ingress.yaml;
      git add apps/base/hello/kustomization.yaml;
      git add apps/overlays/$DOUBLEDRAGON_NAME/hello/kustomization.yaml;
      git add apps/overlays/$DOUBLEDRAGON_NAME/kustomization.yaml;
      git commit -m "Hello app files";
      git push

### Add Hello app to cluster

* Create a _kustomization_ for all apps in the overlay

      flux create kustomization apps \
        --target-namespace=apps \
        --source=flux-system \
        --path="./apps/overlays/$DOUBLEDRAGON_NAME" \
        --depends-on=infrastructure \
        --prune=true \
        --interval=10m \
        --export > clusters/$DOUBLEDRAGON_CLUSTER/apps.yaml

* Add update the repo

      git add clusters/$DOUBLEDRAGON_CLUSTER/apps.yaml;
      git commit -m "Adding apps to the cluster";
      git push
### Test Hello

* Find the ingress controller's `External IP`

      kubectl get services -n apps ingress-nginx-controller

* Use `curl` to resolve the URL. Replace `11.22.33.44` with the external IP,
  and `lynx` to view it

      curl -H "Host: hello.example.com" \
        --resolve hello.example.com:80:11.22.33.44 \
        --resolve hello.example.com:443:11.22.33.44 \
        http://hello.example.com | lynx -stdin

* This should show a basic hello world page, with an Nginx logo and some server address, name and date details.

### Delete Hello

* Remove / comment out the app

  Normally you just edit `apps/overlay/$DOUBLEDRAGON_NAME/kustomizaton.yaml`
  And comment out

      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization
      namespace: apps
      bases:
      #  - hello

   But since this is the only app in there it might break the YAML.

   So the easiert is to delete cluster's `apps.yaml` file

      git rm clusters/$DOUBLEDRAGON_CLUSTER/apps.yaml;
      git commit -m "Removing apps from the cluster";
      git push

  That should cascade the changes via the aggregated _kustomize_, and remove the ingress, service and deployment from the live cluster.

* If permanent, remove the app's `apps/overlays` and `apps/base` folders as well as a tidy-up chore.


## Your first application

Ok, so a _Hello world_ is not what you intended to use your cluster for.
Lets see how you could add a real application to the cluster.

For simplification lets call the app `myfirstapp`.
Replace any reference to it with a correct name.

### Deploy, Service, Ingress, Overlay

These will be very similar to the _Hello_ app.

* Names and labels

  Change all the names and lables from `hello` to `myfirstapp`.

* Deployment

  One thing to change is the image name and tag, and adding a registry secret

  E.g.:

      spec:
        containers:
          - image: gcr.io/somethingsomething:1.2.3
            name: myfirstapp-container
         ...
        imagePullSecrets:
          - name: gcr-registry


* More _deployment_ config

  For the _Hello_ deployment the basic config was sufficient,
  but for more normal workflows you will most likely add more e.g. resource limits, env-vars, secrets etc.

  E.g.:

      spec:
        containers:
          - image: gcr.io/somethingsomething:1.2.3
            ...
            resources:
              requests:
                memory: "250Mi"
                cpu: "50m"
              limits:
                memory: "800Mi"
                cpu: "250m"
            env:
              - name: SOMEVAR
                value: "5"
            envFrom:
              - secretRef:
                  name: some-secret

   These are out-of-scope for this tutorial though.

* Ingress

  You will need to change the hostname in `hosts`, and possibly the paths if necessary.

* Overlay

  You could be clever with the overlay but for now just copy what _Hello_ did

* Add it all to git

      git add apps/base/myfirstapp/deployment.yaml;
      git add apps/base/myfirstapp/service.yaml;
      git add apps/base/myfirstapp/ingress.yaml;
      git add apps/base/myfirstapp/kustomization.yaml;
      git add overlay/$DOUBLE_DRAGON_CLUSTER/myfirstapp/kustomization.yaml;
      git add overlay/$DOUBLE_DRAGON_CLUSTER/kustomization.yaml;
      git commit -m "Added myfirstapp"

### Registry and image repository

* Registry

  You would probably store your app's images in a private Docker registry.

  Revisit the [Container registries](#container-registries) section to add relevant registry secrets.

* Image repository

  Lets add an __Image Repository__ for your application.
  So that the _deployment_ below can scan and find the _Docker_ image it requires

      flux create image repository myfirstapp-source \
        --image=gcr.io/somethingsomething \
        --interval=5m \
        --namespace=infrastructure \
        --export > infrastructure/sources/myfirstapp-source.yaml

* Append this to the YAML in infrastructure/sources/myfirstapp-source.yaml:

      ...
      accessFrom:
        namespaceSelectors:
          - matchLabels:
              kubernetes.io/metadata.name: apps

* Append this source to the _Kustomization_ at `infrastructure/sources/kustomization.yaml`

      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization
      resources:
        ...
        - myfirstapp-source.yaml

* Add to git

      git add infrastructure/sources/myfirstapp-source.yaml;
      git add infrastructure/sources/kustomization.yaml;
      git commit -m "myfirstapp source";
      git push


### Test the application

* Like _hello_ use `curl`

      curl -H "Host: myfirstapp.example.com" \
        --resolve myfirstapp.example.com:80:11.22.33.44 \
        --resolve myfirstapp.example.com:443:11.22.33.44 \
        http://myfirstapp.example.com | lynx -stdin

   Replace `myfirstapp.example.com` with your hostname

That should be the basics to get your first application added

## Automatic image updates

Letting _Flux_ automatically update the image if a newer one get uploaded to the _docker registry_ is a handy feature.

_Flux_ allows different policies on when to update it, such as only when approved, only on major version upgrades and more.

Here is how to update on every new [semver](https://semver.org/) tag:


* Image repository

  As shown above in the [Your first application](#your-first-application) section, you need an _image repository_ source(s) for your application images

* Image policies

  Similarily we need a policy on how to act on any changes to the source repository

      flux create image policy myfirstapp-policy \
        --image-ref=myfirstapp-source \
        --namespace=apps \
        --select-semver=0.3.x \
        --export > ./apps/base/myfirstapp/image-policy.yaml

  This policy allows updates on any new `0.3.x` semver versions.
  So if on `0.3.1` if `0.3.2` gets uploaded it will trigger this policy.
  A new lower version of `0.3.0` would not.

  However `0.4.0` will not get uploaded. Nor would `1.0.0`.
  For that the `--select-semver` would have to be `0.x` or just `x` I think

* Add source namespace to the policy

  The CLI does not have that option so please change the policy to refer to the "infrastructure" namespace:

      ---
      apiVersion: image.toolkit.fluxcd.io/v1beta1
      kind: ImagePolicy
      metadata:
        name: myfirstapp-policy
        namespace: apps
      spec:
        imageRepositoryRef:
          name: myfirstapp-source
          namespace: infrastructure
        policy:
          semver:
            range: 0.3.x


* Append this policy to the _Kustomization_ of the app `apps/base/myfirstapp/kustomization.yaml`

      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization
      resources:
        ...
        - image-policy.yaml

* Apply the policy

  We need to tell _Flux_ where this policy would apply

  Edit `apps/base/myfirstapp/deployment.yaml`
  and modify the `image` line by appending the policy name

      ...
      spec:
        containers:
          - image: gcr.io/something:0.3.1 # {"$imagepolicy": "apps:myfirstapp-policy "}
            ...

  This seems a bit hacky but this is how it works

* Image Update

  We also need to tell the `apps` kustomization about how to update any the images.
  I.e. create a _git commit_.

      flux create image update apps-image-update --namespace=apps \
        --git-repo-ref=flux-system \
        --git-repo-namespace=flux-system \
        --git-repo-path="./" \
        --checkout-branch=main \
        --push-branch=main \
        --author-name=fluxcdbot \
        --author-email=fluxcdbot@users.noreply.github.com \
        --commit-template="{{range .Updated.Images}}{{println .}}{{end}}" \
        --export > apps/overlays/$DOUBLE_DRAGON_NAME/image-updates.yaml

* Append to `apps/overlays/$DOUBLE_DRAGON_NAME/kustomization.yaml`

      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization
      bases:
        - myfirstapp
        ...
      resources:
        ...
        - image-updates.yaml

* Add to the repo

      git add apps/base/myfirstapp/deployment.yaml;
      git add apps/base/myfirstapp/image-policy.yaml;
      git add apps/base/myfirstapp/kustomization.yaml;
      git add apps/overlays/$DOUBLE_DRAGON_NAME/image-updates.yaml;
      git add apps/overlays/$DOUBLE_DRAGON_NAME/kustomization.yaml;
      git commit -m "myfirstapp image policy";
      git push


* New versions

  Any new images in the docker repository that is in the chosen semver version range will now automatically update the deployment.

  Note: You need to wait for the various polling of the image repository, the policy, flux itself etc. So changes might take awhile.

  An alternative is that _Flux_ also let policy changes be prompted via webhook, which is also possible.

* Tail if the policy has updated

      flux get image policy -n apps --watch

  Or any image states

      flux get images all --all-namespaces
## Go wild

* Add/update your deployments, services, charts, docker registries, secrets, kustomizations etc

### Usual steps for a simple web app

1. Add source if in a private repo. And add/append to source _kustomization_.

   * `infrastructure/sources/someapp-source.yaml`
   * `infrastructure/sources/kustomization.yaml`
   * `infrastructure/kustomization.yaml`

1. Add deploy, service, ingress to new app base folder.

   * `apps/base/someapp/deployment.yaml`
   * `apps/base/someapp/service.yaml`
   * `apps/base/someapp/ingress.yaml`
   * `apps/base/someapp/image-policy.yaml`

1. Add/append to apps _kustomization_ and overlay.

   * `apps/base/someapp/kustomization.yaml`
   * `apps/base/kustomization.yaml`
   * `apps/overlay/somecluster/someapp/kustomization.yaml`
   * `apps/overlay/somecluster/kustomization.yaml`

   (Some of the _Kustomization_ files can be short-cutted if they do nothing but redirect)

## Further information

---
## Advise: Don't touch

* Once Flux is running, by convention avoid using `kubectl create|apply` etc.
* And by the same convention avoid using `flux create`.

  i.e avoid acting directly for any _write_ operations.

  Nearly all changes should be via Git. Export any changed YAML to Git as above.

  Otherwise the git source and cluster state will start to diverge and hard to recreate.

* Any `kubectl` and  `flux` interaction should be read only. Those are fine.

* Sometimes whilst troubleshooting you will have to use the scalpel and use `kubectl create|apply|delete` or `flux create`.

   But minimise the usage, and try to update the yaml to reflect any permanent changes.

## Troubleshooting

Frequent issues and how to monitor.

### 1 Tail the logs

* _Flux_ logs

      flux logs -Af --since 3h

* _Kubernetes_ logs

      kubectl logs podname -f

### 2 Watch statuses

* _Flux_ kustomization status

      flux get kustomizations --watch

* _Kubernetes_ status

      kubectl get deploy,service,ingress,pods,secret,imagerepository,clusterissuer -n apps

  Or watch a single resource type

      kubectl get deploy -A --watch

### 3 Known possible issues

* Certain operation takes a few minutes, e.g. pod creation, waiting on Flux scan polling

* Nginx: `x509 certificate is not valid`

  Happens sometimes with the Nginx ingresses.
  Seems to be a known problem that require [manual patching](https://github.com/jet/kube-webhook-certgen#patch).
  Or as I fix it:

  * Comment out the _nginx controller_ and the ingresses from the _kustomization_ files.
  * Wait until _Flux_ has removed them from the cluster.
  * Uncomment and add them back in.
  * Note this may change the external IP assigned to the cluster's load balancer.


### 4 Remove and add

* Fixed typos, and nothing changes?

  Sometimes some resources gets added with a typo,
  but you fixed it and pushed the change to the repo,
  yet _Flux_ or _Kubernetes_ do not pick up the change?

  Most of the time _Flux_ and _Kubernetes_ notices and changes the resources. But sometimes not.

* Force the change.

  Simply remove the resource, push to git, let the system catch up, add it back with the typo corrected, and the change gets picked up

  Most of the time the _"removal"_ can be done by commenting out the reference to it in a `kustomization.yaml` file.
  Instead of removing actual _deployment_ etc git files and history.

* Scale the deployments down and back up

      kubectl scale deploy -n apps --replicas=0 myfirstapp-deployment

  Wait until pods are destroyed

      kubectl get pods -n apps --watch

  Then scale back up

      kubectl scale deploy -n apps --replicas=2 myfirstapp-deployment

  Note, sometimes _flux_ notices the inconsistency and scales the app back up as well before you do.

* Force _flux_ do reconcile the cluster state and the repository state

  If impatient

       flux reconcile kustomization apps --with-source

## Removing _Flux_

The nuclear option. But sometimes neccessary.

    flux uninstall --namespace=flux-system

Though usually I just spin up another cluster instead.

Note, any encrypted secrets will have to be re-sealed when re-installing _flux_ with the new certificate

## Add another cluster

* Now that you have a working cluster, scrap it. If you want to.

  Create a new cluster without all the mistakes from setting up the first cluster.

* Or when you just need another cluster naturally, you can do the same.

* In only a few steps, you do not have to do it all again

### Create and bootstrap another cluster

* Create the cluster with your provider

* Authenticate `kubectl` with the new cluster

* Set as the current kubernetes context

* Maybe export a new env-var (and old ones if no longer set)

  In Bash:

       export DOUBLEDRAGON_REPO=doubledragon-fleet;
       export DOUBLEDRAGON_CLUSTER=doubledragon-01;
       export DOUBLEDRAGON_CLUSTER_NEW=doubledragon-02

  In Fish:

       set -x DOUBLEDRAGON_REPO doubledragon-fleet;
       set -x DOUBLEDRAGON_CLUSTER doubledragon-01;
       set -x DOUBLEDRAGON_CLUSTER_NEW doubledragon-02

* Bootstrap the new cluster with `doubledragon-02` or `$DOUBLEDRAGON_CLUSTER_NEW` as the name

      flux bootstrap github \
        --components-extra=image-reflector-controller,image-automation-controller \
        --owner=$GITHUB_USER \
        --repository=$DOUBLEDRAGON_REPO \
        --branch=main \
        --path=./clusters/$DOUBLEDRAGON_CLUSTER_NEW \
        --read-write-key \
        --personal

   After a while pull the changes

      git pull

### Copy and re-initialise infrastructure

* Add namespaces to the new cluster

      cp clusters/$DOUBLEDRAGON_CLUSTER/namespaces.yaml clusters/$DOUBLEDRAGON_CLUSTER_NEW/;
      git add clusters/$DOUBLEDRAGON_CLUSTER_NEW/namespaces.yaml;
      git commit -m "Double Dragon II namespaces";
      git push

* Copy the `infrastructure.yaml` kustomization to the new cluster

      cp clusters/$DOUBLEDRAGON_CLUSTER/infrastructure.yaml clusters/$DOUBLEDRAGON_CLUSTER_NEW/;
      git add clusters/$DOUBLEDRAGON_CLUSTER_NEW/infrastructure.yaml;
      git commit -m "Double Dragon II infrastructure";
      git push

  This will add the _Sealed Secrets_, _Nginx_, and everything in _sources_ to the new cluster.
  And more if you have extended it.

  The _sources_ may cause issues initially until we re-encrypt any secrets.

* Download the _Sealed Secrets_ public key for this cluster

      mkdir -p clusters/$DOUBLEDRAGON_CLUSTER_NEW/secrets;

      kubeseal --fetch-cert \
      --controller-name=sealed-secrets-controller \
      --controller-namespace=infrastructure \
      > clusters/$DOUBLEDRAGON_CLUSTER_NEW/secrets/sealed-secrets-cert.pem;

      git add clusters/$DOUBLEDRAGON_CLUSTER_NEW/secrets/sealed-secrets-cert.pem;
      git commit -m "Sealed Secret public key";
      git push

* Re-encrypt secrets such as the GCR registry secret if needed

  E.g.

      mkdir -p clusters/$DOUBLEDRAGON_CLUSTER_NEW/registries/apps;
      mkdir -p clusters/$DOUBLEDRAGON_CLUSTER_NEW/registries/infrastructure;

      kubeseal --format=yaml --namespace=apps \
       --cert=clusters/$DOUBLEDRAGON_CLUSTER_NEW/secrets/sealed-secrets-cert.pem \
       < gcr-registry.yml \
       > clusters/$DOUBLEDRAGON_CLUSTER_NEW/registries/apps/sealed-gcr-registry.yml;

      kubeseal --format=yaml --namespace=infrastructure \
       --cert=clusters/$DOUBLEDRAGON_CLUSTER_NEW/secrets/sealed-secrets-cert.pem \
       < gcr-registry.yml \
       > clusters/$DOUBLEDRAGON_CLUSTER_NEW/registries/infrastructure/sealed-gcr-registry.yml;

      git add clusters/$DOUBLEDRAGON_CLUSTER_NEW/registries/apps/sealed-gcr-registry.yml;
      git add clusters/$DOUBLEDRAGON_CLUSTER_NEW/registries/infrastructure/sealed-gcr-registry.yml;
      git commit -m "GCR registry for cluster DD-02 ns";
      git push

### Copy/tweak over other cluster specifics

* Such as *certificate-manager* issuers

      mkdir -p clusters/$DOUBLEDRAGON_CLUSTER_NEW/certificate-issuers;
      cp clusters/$DOUBLEDRAGON_CLUSTER/certificate-issuers/* \
         clusters/$DOUBLEDRAGON_CLUSTER_NEW/certificate-issuers/;

      git add clusters/$DOUBLEDRAGON_CLUSTER_NEW/certificate-issuers/*;
      git commit -m "Staging and Prod issuer";
      git push

### Copy/tweak apps overlay

* Optionally create a new overlay

  Or share the same common one in `apps/overlays/doubledragon`
  linked in `apps.yaml`

      cp clusters/$DOUBLEDRAGON_CLUSTER/apps.yaml clusters/$DOUBLEDRAGON_CLUSTER_NEW/;
      git add clusters/$DOUBLEDRAGON_CLUSTER_NEW/apps.yaml;
      git commit -m "Apps overlay for cluster DD-02";
      git push

* And that will be it

* Note, the exposed load balancer external IP will be different.

## Providers and tools

### Kubernetes as a Service

* Cloud providers

  * Amazon AWS EKS: [aws.amazon.com/eks/](https://aws.amazon.com/eks/)
  * Google Cloud GKE: [cloud.google.com/kubernetes-engine/](https://cloud.google.com/kubernetes-engine/)
  * Microsoft Azure AKS: [azure.microsoft.com/en-us/services/kubernetes-service/](https://azure.microsoft.com/en-us/services/kubernetes-service/)
  * DigitalOcean Kubernetes: [www.digitalocean.com/products/kubernetes/](https://www.digitalocean.com/products/kubernetes/)

* Cloud provider CLIs

  * [cloud.google.com/sdk/](https://cloud.google.com/sdk/)

        brew cask install google-cloud-sdk

  * [github.com/digitalocean/doctl](https://github.com/digitalocean/doctl)

        brew install doctl
  * [eksctl.io](https://eksctl.io/)

        brew tap weaveworks/tap
        brew install weaveworks/tap/eksctl

  * [github.com/Azure/azure-cli](https://github.com/Azure/azure-cli)

        brew install azure-cli

### Tools

* [github.com/ahmetb/kubectx](https://github.com/ahmetb/kubectx)

      brew install kubectx

* [github.com/vmware-tanzu/octant](https://github.com/vmware-tanzu/octant)

      brew install octant

* [k9ss.io](https://k9ss.io)

      brew install derailed/k9s/k9s

* [keel.sh](https://keel.sh)

* [headlamp.dev](https://headlamp.dev)
   
  * [headlamp.dev/blog/2025/03/26/flux-ui-updates](https://headlamp.dev/blog/2025/03/26/flux-ui-updates)

* [github.com/stern/stern](https://github.com/stern/stern)

      brew install stern

* [flurdy.com/docs/kubernetes/registry/kubernetes-docker-registry.html](https://flurdy.com/docs/kubernetes/registry/kubernetes-docker-registry.html)
* [ramitsurana.github.io/awesome-kubernetes/](https://ramitsurana.github.io/awesome-kubernetes/)

Client tools are also available on Linux, Windows and more.

### License


The _Lemmings_ and _Double Dragon_ code bases are licensed under the _MIT license_ which lets you pretty much do as you please with it.

Though please attribute back if possible.

### Attributions

* This guide heavily used the docs and example project available on the official Flux website
   * [fluxcd.io/flux](https://fluxcd.io/flux/)
* Kubernetes official docs
   * [kubernetes.io/docs/](https://kubernetes.io/docs/)
* [cert-manager.io/docs/](https://cert-manager.io/docs/installation/continuous-deployment-and-gitops/)

### Versions

* 2025-03-25 Updated for Flux 2.5
* 2023-03-21 Your first app and image updates
* 2023-02-09 Double Dragon tweaks and env-vars
* 2022-11-10 Double Dragon refreshed
* 2021-07-10 Flux 2. Lemmings => Double Dragon
* 2020-02-13 Flux 1.1, fluxcd.io annotations, and Helm 3
* 2019-11-07 Flux 0.16, flux.weave.works annotations and Helm 2
