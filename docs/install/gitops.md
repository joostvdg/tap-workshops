---
title: TAP GitOps Install
description: Install a TAP Profile via GitOps
author: joostvdg
tags: [tap, kubernetes, install, GitOps]
---


## Goal, Outcome & Checks

> Goal: Complete an installation of TAP for a Customer with GitOps and namespace management ready for production usage as an Operator

> Outcome:  TAP install and updates is controlled via chages in Git only

- [ ] I am able to manage a TAP installation with GitOPs

## Raw Notes

* h2o -> so we use SOPS
* Prerequisites
* Accept EULAs
* Install Cluster Essentials
* Install TAP with GitOps (SOPS)

### Install Cluster Essentials

* https://docs.vmware.com/en/Cluster-Essentials-for-VMware-Tanzu/1.5/cluster-essentials/deploy.html
* Login to Tanzu Network
* Download Cluster Essentials
* Upload to Jump Host
* Create Namespace
* Create ConfigMap for CA

```sh
kubectl create namespace kapp-controller
```

```sh
kubectl create secret generic kapp-controller-config \
   --namespace kapp-controller \
   --from-file caCerts=ssl/ca.crt
```

```sh
mkdir $HOME/tanzu-cluster-essentials
tar -xvf $HOME/scripts/scripts/tanzu-cluster-essentials-linux-amd64-1.5.0.tgz -C $HOME/tanzu-cluster-essentials
```

```sh
export INSTALL_BUNDLE=registry.tanzu.vmware.com/tanzu-cluster-essentials/cluster-essentials-bundle@sha256:79abddbc3b49b44fc368fede0dab93c266ff7c1fe305e2d555ed52d00361b446
export INSTALL_REGISTRY_HOSTNAME=registry.tanzu.vmware.com
export INSTALL_REGISTRY_USERNAME=TANZU-NET-USER
export INSTALL_REGISTRY_PASSWORD=TANZU-NET-PASSWORD
```

```sh
cd $HOME/tanzu-cluster-essentials
```

```sh
./install.sh --yes
```

NOTE: looks like my cluster already has the essentials installed -> part of TGKm?


## GitOps Install

* install Age -> https://github.com/FiloSottile/age#installation
* create new repo in Gitea -> `gitops-iterate-01`
* sync it
* download Reference Implementation
* ingest reference implmentation (script)
* run cluster config init script
* update git repo
* 

### Init Repo

```sh
mkdir -p $HOME/projects/gitops-iterate-01
cd $HOME/projects/gitops-iterate-01
git config --global init.defaultBranch main
```

```sh
touch README.md
git init
git checkout -b main
git add README.md
git commit -m "first commit"
git remote add origin https://gitea.services.h2o-2-9349.h2o.vmware.com/gitea/gitops-iterate-01.git
git push -u origin main
```

```sh
scp ~/Downloads/tanzu-gitops-ri-0.1.0.tgz ubuntu@10.220.13.174:/home/ubuntu
```

```sh
tar xvf $HOME/tanzu-gitops-ri-0.1.0.tgz  -C $HOME/projects/gitops-iterate-01
```

```sh
./setup-repo.sh iterate sops
```

```sh
git add . && git commit -m "Add iterate"
git push -u origin
```

```sh
cd clusters/iterate
less docs/README.md
```

### Create SOPS Key

```sh
mkdir -p $HOME/tmp-enc
chmod 700 $HOME/tmp-enc
cd $HOME/tmp-enc
```

```sh
age-keygen -o key.txt
```

* create `tap-sensitive-values.yaml`


```yaml
---
tap_install:
  sensitive_values:
```

```sh
export SOPS_AGE_RECIPIENTS=$(cat key.txt | grep "# public key: " | sed 's/# public key: //')
```

```sh
sops --encrypt tap-sensitive-values.yaml > tap-sensitive-values.sops.yaml
```

```sh
export SOPS_AGE_KEY_FILE=key.txt
```

```sh
sops --decrypt tap-sensitive-values.sops.yaml
```

```sh
cp tap-sensitive-values.sops.yaml $HOME/projects/gitops-iterate-01/clusters/iterate/cluster-config/values/
```

## Prepare Values

* use iterate: https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/multicluster-reference-tap-values-iterate-sample.html
* create `tap-non-sensitive-values.yaml`
    * at `<GIT-REPO-ROOT>/clusters/<CLUSTER-NAME>/cluster-config/values/tap-non-sensitive-values.yaml`
    * `vim clusters/iterate/cluster-config/values/tap-non-sensitive-values.yaml`


```yaml
tap_install:
  values:
    ceip_policy_disclosed: true
    excluded_packages:
    - policy.apps.tanzu.vmware.com
```

```yaml
tap_install:
  values:
    profile: iterate
    ceip_policy_disclosed: true
    shared:
      ingress_domain: "iterate.h2o-2-9349.h2o.vmware.com"
      ca_cert_data: | # To be passed if using custom certificates
        -----BEGIN CERTIFICATE-----
        MIID7jCCAtagAwIBAgIURv5DzXSDklERFu4gL2sQBNeRg+owDQYJKoZIhvcNAQEL
        BQAwgY4xCzAJBgNVBAYTAk5MMRgwFgYDVQQIEw9UaGUgTmV0aGVybGFuZHMxEDAO
        BgNVBAcTB1V0cmVjaHQxFTATBgNVBAoTDEtlYXJvcyBUYW56dTEdMBsGA1UECxMU
        S2Vhcm9zIFRhbnp1IFJvb3QgQ0ExHTAbBgNVBAMTFEtlYXJvcyBUYW56dSBSb290
        IENBMB4XDTIyMDMyMzE1MzUwMFoXDTI3MDMyMjE1MzUwMFowgY4xCzAJBgNVBAYT
        Ak5MMRgwFgYDVQQIEw9UaGUgTmV0aGVybGFuZHMxEDAOBgNVBAcTB1V0cmVjaHQx
        FTATBgNVBAoTDEtlYXJvcyBUYW56dTEdMBsGA1UECxMUS2Vhcm9zIFRhbnp1IFJv
        b3QgQ0ExHTAbBgNVBAMTFEtlYXJvcyBUYW56dSBSb290IENBMIIBIjANBgkqhkiG
        9w0BAQEFAAOCAQ8AMIIBCgKCAQEAyZXDL9W2vu365m//E/w8n1M189a5mI9HcTYa
        0xZhnup58Zp72PsgzujI/fQe43JEeC+aIOcmsoDaQ/uqRi8p8phU5/poxKCbe9SM
        f1OflLD9k2dwte6OV5kcSUbVOgScKL1wGEo5mdOiTFrEp5aLBUcbUeJMYz2IqLVa
        v52H0vTzGfmrfSm/PQb+5qnCE5D88DREqKtWdWW2bCW0HhxVHk6XX/FKD2Z0FHWI
        ChejeaiarXqWBI94BANbOAOmlhjjyJekT5hL1gh7BuCLbiE+A53kWnXO6Xb/eyuJ
        obr+uHLJldoJq7SFyvxrDd/8LAJD4XMCEz+3gWjYDXMH7GfPWwIDAQABo0IwQDAO
        BgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUwAwEB/zAdBgNVHQ4EFgQUfGU50Pe9
        YTv5SFvGVOz6R7ddPcUwDQYJKoZIhvcNAQELBQADggEBAHMoNDxy9/kL4nW0Bhc5
        Gn0mD8xqt+qpLGgChlsMPNR0xPW04YDotm+GmZHZg1t6vE8WPKsktcuv76d+hX4A
        uhXXGS9D0FeC6I6j6dOIW7Sbd3iAQQopwICYFL9EFA+QAINeY/Y99Lf3B11JfLU8
        jN9uGHKFI0FVwHX428ObVrDi3+OCNewQ3fLmrRQe6F6q2OU899huCg+eYECWvxZR
        a3SlVZmYnefbA87jI2FRHUPqxp4P2mDwj/RZxhgIobhw0zz08sqC6DW0Aj1OIJe5
        sDAm0uiUdqs7FZN2uKkLKekdTgW0QkTFEJTk5Yk9t/hOrjnHoWQfB+mLhO3vPhip
        vhs=
        -----END CERTIFICATE-----
    buildservice:
      pull_from_kp_default_repo: true
      exclude_dependencies: true
      kp_default_repository: "harbor.services.h2o-2-9349.h2o.vmware.com/buildservice/tbs-full-deps"

    supply_chain: basic
    ootb_supply_chain_basic:
      registry:
        server: harbor.services.h2o-2-9349.h2o.vmware.com
        repository: tap-apps

    image_policy_webhook:
      allow_unmatched_tags: true

    contour:
      envoy:
        service:
          type: LoadBalancer 

    cnrs:
      domain_name: "iterate.h2o-2-9349.h2o.vmware.com"

    appliveview_connector:
      backend:
        sslDeactivated: true
        ingressEnabled: true
        host: appliveview.view.h2o-2-9349.h2o.vmware.com
    excluded_packages:
    - policy.apps.tanzu.vmware.com
    - scanning.apps.tanzu.vmware.com
    - grype.scanning.apps.tanzu.vmware.com
```

```yaml
---
tap_install:
  sensitive_values:
    shared:
      image_registry:
        project_path: "harbor.services.h2o-2-9349.h2o.vmware.com/tap"
        username: ""
        password: ''
    buildservice:
      kp_default_repository_username: ""
      kp_default_repository_password: ''
```

## Setup Sync

### Git SSH


```sh
ssh-keyscan gitssh.h2o-2-9349.h2o.vmware.com > gitea-known-hosts.txt
```

### Sync Script

```sh
export INSTALL_REGISTRY_HOSTNAME=harbor.services.h2o-2-9349.h2o.vmware.com
export INSTALL_REGISTRY_USERNAME=admin
export INSTALL_REGISTRY_PASSWORD='VMware123!'
export GIT_SSH_PRIVATE_KEY=$(cat $HOME/.ssh/id_gitea)
export GIT_KNOWN_HOSTS=$(ssh-keyscan 1)
export SOPS_AGE_KEY=$(cat $HOME/tmp-enc/key.txt)
export TAP_PKGR_REPO=harbor.services.h2o-2-9349.h2o.vmware.com/tap/tap-packages
```

```sh
./tanzu-sync/scripts/configure.sh
```

```sh
git add cluster-config/ tanzu-sync/
git commit -m "Configure install of TAP 1.5.0"
git push
```

### Deploy Script

!!! Warning
    Run this with your Kubernetes context set to your target cluster!

    ```sh
    kubectx tap-iterate-admin@tap-iterate
    ```

```sh
./tanzu-sync/scripts/deploy.sh
```

## Setup Developer Namespaces

* https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/install-gitops-set-up-namespaces.html
* https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/namespace-provisioner-about.html
* https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/namespace-provisioner-customize-installation.html
* https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/namespace-provisioner-customize-installation.html#git-install

### Define GitOps For Namespace Provisioner

* create `desired-namespaces.yaml`
* create `namespaces.yaml`

```yaml title="desired-namespaces.yaml"
#@data/values
---
namespaces:
- name: dev
- name: qa
```

```yaml title="namespaces.yaml"
#@ load("@ytt:data", "data")
#! This loop will now loop over the namespace list in
#! in ns.yaml and will create those namespaces.
#@ for ns in data.values.namespaces:
---
apiVersion: v1
kind: Namespace
metadata:
  name: #@ ns.name
#@ end
```

### Configure Namespace Provisioner in TAP Install Values

* update `cluster-config/values/tap-non-sensitive-values.yaml`
* https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/namespace-provisioner-use-case3.html#git-private

```yaml
tap_install:
  values:
    ...
    namespace_provisioner:
      controller: false
      gitops_install:
        ref: origin/main
        subPath: clusters/iterate/dev-namespaces
        url: git@gitssh.h2o-2-9349.h2o.vmware.com:gitea/gitops-iterate-01.git
        secretRef:
          name: sync-git-ssh
          namespace: tanzu-sync
          create_export: true
```

## Tanzu Build Service Dependencies

* Inspired by VRabbi: https://vrabbi.cloud/post/tap-1-5-gitops-installation/

### Add TBS Dependencies Package and Repository

* create `cluster-config/config/tbs-install/package-repository.yaml`
* create `cluster-config/config/tbs-install/package-install.yaml`

```yaml title="package-repository.yaml"
apiVersion: packaging.carvel.dev/v1alpha1
kind: PackageRepository
metadata:
  name: tbs-full-deps-repository
  namespace: tap-install
  annotations:
    kapp.k14s.io/change-group: pkgr
spec:
  fetch:
    imgpkgBundle:
      image: harbor.services.h2o-2-9349.h2o.vmware.com/buildservice/tbs-full-deps:1.10.8
```

```yaml title="package-install.yaml"
apiVersion: packaging.carvel.dev/v1alpha1
kind: PackageInstall
metadata:
  name: full-tbs-deps
  namespace: tap-install
  annotations:
    kapp.k14s.io/change-group: tbs
    kapp.k14s.io/change-rule.0: "upsert after upserting pkgi"
    kapp.k14s.io/change-rule.1: "delete before deleting pkgi"
spec:
  serviceAccountName: tap-installer-sa
  packageRef:
    refName: full-tbs-deps.tanzu.vmware.com
    versionSelection:
      constraints: 1.10.8
```

## Test With Workload

Now that we have a Namespace to work in, we can define a Workload.

```sh
export TAP_DEVELOPER_NAMESPACE=dev
```

### Create Workload

We can then either use the CLI or the `Workload` CR to create our test workload.

=== "Tanzu CLI"
    ```sh
    tanzu apps workload create smoke-app \
      --git-repo https://github.com/sample-accelerators/tanzu-java-web-app.git \
      --git-branch main \
      --type web \
      --label app.kubernetes.io/part-of=smoke-app \
      --annotation autoscaling.knative.dev/minScale=1 \
      --yes \
      -n "$TAP_DEVELOPER_NAMESPACE"
    ```
=== "Kubernetes Manifest"
    ```sh
    echo "apiVersion: carto.run/v1alpha1
    kind: Workload
    metadata:
      labels:
        app.kubernetes.io/part-of: smoke-app
        apps.tanzu.vmware.com/workload-type: web
      name: smoke-app
      namespace: ${TAP_DEVELOPER_NAMESPACE}
    spec:
      params:
      - name: annotations
        value:
          autoscaling.knative.dev/minScale: \"1\"
      source:
        git:
          ref:
            branch: main
          url: https://github.com/sample-accelerators/tanzu-java-web-app.git
    " > workload.yml
    ```

    ```sh
    kubectl apply -f workload.yml
    ```

To verify the status of FluxCD checkout:

```sh
kubectl get gitrepo -A
```

Use `kubectl wait` to wait for the app to be ready.

```sh
kubectl wait --for=condition=Ready Workload smoke-app --timeout=10m -n "$TAP_DEVELOPER_NAMESPACE"
```

### Verify Workload

To see the logs:

```sh
tanzu apps workload tail smoke-app
```

To get the status:

```sh
tanzu apps workload get smoke-app
```

### Delete Workload

And then we can delete our test workload if want to.

```sh
tanzu apps workload delete smoke-app -y -n "$TAP_DEVELOPER_NAMESPACE"
```

## Links

* [Official Docs](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/install-gitops-intro.html)
* [VRabbi Blog](https://vrabbi.cloud/post/tap-1-5-gitops-installation/)
* [Mozilla SOPS](https://github.com/mozilla/sops)