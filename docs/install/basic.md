---
title: TAP Basic Install
description: TAP Single Cluster with Supply Chain Basic Installation
author: joostvdg
tags: [tap, kubernetes, install]
---

!!! Important
    The Goals and Outcomes is the work of Rick Farmer.

## Goals & Outcomes Operator

1. I can do a basic install of TAP within a customer environment
1. I have a basic understanding of the core features of TAP such that I can explain these to a customer
1. I’m able to demo TAP deployment via an OOTB accelerator and can explain the benefits of accelerator

### Checks

- [ ] Know the relevant parts of the docs
- [ ] Understand the prerequisites: TAP install
- [ ] Understand the prerequisites: Jump Host
- [ ] Can configure a TAP Full Profile config file
- [ ] Can discover how to configure TAP Packages
- [ ] Understand how to configure custom CA
- [ ] Can install TAP Full Profile
- [ ] I manually installed TAP
- [ ] I'm able to use an accelerator to generate a Web TAP Workload
- [ ] I'm able to manually setup a single developer namespace
- [ ] I applied a TAP Web Workload and register it within TAP GUI
- [ ] I'm able to do basic troubleshooting of a TAP Workload
- [ ] I'm able to do basic troubleshooting of a TAP Installation
- [ ] I've configured App Live View in TAP GUI for a TAP Web Workload
- [ ] I can update TAP values and reconcile changes on the cluster
- [ ] I understand TAP GUI Catalog System
- [ ] I know how to configure an integration to a private Git provider in TAP GUI
- [ ] I know how to configure access to private container registries in TAP Values

### Optional

- [ ] I'm able to install Learning Center and use the sample workshop.
- [ ] I understand the different Workload Types available in TAP
- [ ] I can configure a public Git provider for TAP GUI in TAP Values
- [ ] I can configure DNS for TAP Endpoints
- [ ] I have relocated TAP Images to my own container registry and understand the imgpkg utility.
- [ ] I understand Kapp Controller and Secret Gen Controller in the context of TAP
- [ ] I've able to configure and install the OOTB Testing and Scanning Supply Chain and verify a Workload
- [ ] I am able to configure a Scan Policy for a Source and Image Scan
- [ ] I am able to create a Tekton Task to execute Unit Tests within a Supply Chain for a Developer Namespace
- [ ] I know how to configure an Auth Provider for TAP GUI access
- [ ] I'm able to configure TAP GUI to read from Metadata Store.
- [ ] I know how to configure access to private repositories (SCM and Container) for TAP Workloads
- [ ] I am able to register API Documentation for a Workload in TAP GUI
- [ ] I am able to view and verify results for the Security Analysis within TAP GUI

## Steps

* Verify Prerequisites are met
* Install Cluster Essentials
* Choose TAP setup (profile, installation type)
* Review TAP packages configuration options
    * e.g., TAP values schema -> map value to other package -> other package values schema
* Install TAP Profile
* Install Test Workload

## Verify Prerequisites are met

The [prerequisites](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/prerequisites.html)[^1] we need:

* Credentials for Tanzu Net
* Credentials for local Container Image Registry
* Accept Tanzu Application Platform EULAs
* DNS Records for the clusters
* Suitable Kubernetes clusters
* Jump Host or other machine with the required tools in place
* Relocate TAP Images to local Container Image Registry

### Required Tools

* kubectl
* yq
* jq
* Tanzu CLI
    * with TAP plugins
* curl
* ...?

### Suitable Kubernetes clusters

TAP 1.5 [supports Kubernetes 1.24, 1.25 and 1.26](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/prerequisites.html#kubernetes-cluster-requirements-3)[^2].

Indicative cluster resource requirements of TAP per profile: (TODO: verify these numbers)

| Profile 	    | Memory/Node 	| Storage/Node 	| vCPU Total 	| Memory Total |
|-------------	|-------------	|--------------	|------------	| ------------ |
| **Iterate** 	|  8GB         	| 150GB        	| 12         	| 16GB         |
| **Build**   	| 16GB         	| 150GB        	| 12         	| 12GB         |
| **View**    	|  8GB         	|  50GB        	| 8          	|  8GB         |
| **Run**     	|  8GB         	| 100GB        	| 12         	|  8GB         |
| **Full**    	| 16GB        	| 150BG        	| 16         	| 20GB         |

!!! Warning
    These are requirements to run the TAP components.
    This does **not** include the applications or builds run in the cluster.

    For example, let's look at an application in a Run cluster.
    If your application requires 10vCPU and 40GB memory, you add that on top of the TAP requirements.
    Your Run cluser now needs a minimum of 22vCPU and 48GB of memory.

### TAP Images Relocated

TODO: provide information on what has been relocated and to where for the LAB environment
[^3]

## Optional: Install Cluster Essentials

!!! Danger "Optional"
    The intention is that your Lab environments has them installed already.

    You can verify this:

    ```sh
    kubectl get po -n secretgen-controller
    ```

    ```sh
    NAME                                   READY   STATUS    RESTARTS   AGE
    secretgen-controller-b9d795c44-ml5pk   1/1     Running   0          42m
    ```

    And, to be sure:

    ```sh
    kubectl get pod -n tkg-system
    ```

    ```sh
    NAME                                                     READY   STATUS    RESTARTS   AGE
    kapp-controller-55d5dd6486-b47mn                         2/2     Running   0          43m
    tanzu-capabilities-controller-manager-7cdd959657-sb9x6   1/1     Running   0          42m
    ```

    If these are not there and both commands return empty, verify you are talking to the correct cluster.
    And if required, below are the instructions for installing the Cluster Essentials.

    If they are installed, [proceed to Choose TAP setup](/tap-workshops/install/basic/#choose-tap-setup-profile-installation-type)

[Cluster Essentials](https://docs.vmware.com/en/Cluster-Essentials-for-VMware-Tanzu/1.5/cluster-essentials/deploy.html)[^4] essentially (pun intended) boils down to two components:

* [KAPP Controller](https://carvel.dev/kapp-controller/)[^5]
* [SecretGen Controller](https://github.com/carvel-dev/secretgen-controller)[^6]

Please install the [Cluster Essentials](https://docs.vmware.com/en/Cluster-Essentials-for-VMware-Tanzu/1.5/cluster-essentials/deploy.html)[^4] according to the docs.

!!! Important
    While not explicitly mentioned, for each TAP minor version (e.g., 1.4, 1.5) there is an associated Cluster Essentials[^4].

    A minimum version of either controller is required, although there is no explicit version of either known at this point in time (April 2023).

    So unless there is a strong reason not too, use the Cluster Essentials referenced in the TAP docs.

## Choose TAP setup (profile, installation type)

For almost every customer, the TAP installation covers multiple environments, multiple clusters, and various profiles.

Some customers want a Run cluster per application, others assume multi-tenancy is oke.

It is generally recommended to assume different environments with their own purpose. For example:

* **Test**: contains one or more clusters, regularly re-created, used for experimentation and learning. Only used by the Platform Team.
* **Staging**: multiple clusters and multiple profiles, used for testing specific features and upgrades. Used by the Platform Team, and a handful of "beta testers".
* **Production**: multiple clusters and multiple profiles, including an iterate cluster for learning for end-users.

### Full Profile For Workshop

For the sake of brevity for this workshop, we'll stick to a single cluster with the **Full** profile.

While this does not represent a typical installation, it let's you go through the steps of installing and using TAP.
It also shows you all the components and let's you customize and interact with them, without having to go multiple times the same process.

The **Full** profile contains all the components of TAP, which is what the name implies (not always true, but this time it is).

### Install Type

Next we choose how we want to install TAP.

There are currently three options:

1. Traditional manual KAPP package install, online
1. Traditional manual KAPP package install, **offline**
1. GitOps install, ***beta***

It is likely that the GitOps installation type will be the default in the future.

For now the most common (and supported) installation is the Traditional **Offline** install.

It is good to understand what steps need to be taken before automating them.
So this is the type we'll use for this workshop.

### Version

Ideally we want to use the latest (supported) version.

Which at this time of writing is TAP **1.5**.

TAP 1.5 requires Kubernetes 1.24, so this is currently (April 2023) not supported on TGKs based customers, as they can only go to Kubernetes 1.23.

Assuming that in due time vSphere 8 with TGKs does support 1.24 (the Supervisor cluster already does), TAP 1.5 is a relatively safe bet.

TAP 1.6, not yet released, will require Kubernetes 1.25, which won't be supported anytime soon with TGKs or TGKm.

So let's stick to TAP **1.5**.

### Conclusion

Our TAP environment will be as follows:

* TAP 1.5
* single Kubernetes cluster of 1.24
* using the Full profile
* installed via the traditional KAPP package, assuming a internet restricted environment

## Install Package Repository

Now that we know what version of TAP we want to install, and how we want to install it, we install the Package Repository.

Assumptions:
* All relevant TAP packages are relocated
* We have read credentials to Image Registry containing the TAP apps
* We have Kubernetes cluster
  * Which runs a version that TAP supports (e.g., 1.24)
  * The nodes trust the CA of Image Registry

### Create Required Secrets

The TAP Package Repository comes from either Tanzu Network or the Registry you relocated the TAP images to.

In either case, you need a Registry ***read*** credential in the Namespace we install the Package Repository in.

```sh
export TAP_INSTALL_NAMESPACE=tap-install
export TAP_INSTALL_REGISTRY_SECRET=tap-registry
export TAP_INSTALL_REGISTRY_HOSTNAME=
export TAP_INSTALL_REGISTRY_USERNAME=
export TAP_INSTALL_REGISTRY_PASSWORD=
```

First, create the Namespace:

```sh
kubectl create namespace ${TAP_INSTALL_NAMESPACE} || true
```

And then use the Tanzu CLI to create a credential via the SecretGen Controller:

```sh
tanzu secret registry add ${TAP_INSTALL_REGISTRY_SECRET} \
    --server    $TAP_INSTALL_REGISTRY_HOSTNAME \
    --username  $TAP_INSTALL_REGISTRY_USERNAME \
    --password  $TAP_INSTALL_REGISTRY_PASSWORD \
    --namespace ${TAP_INSTALL_NAMESPACE} \
    --export-to-all-namespaces \
    --yes 
```

!!! Important "Registry Write Secret"
    TAP profiles `Iterate`, `Full`, and `Build`, also need a Registry ***write*** secret.

    This secret is used to write built images of the applications going throuhg the Supply Chains.
    Define the appropriate environment variables:

    ```sh
    export TAP_INSTALL_NAMESPACE=tap-install
    export TAP_BUILD_REGISTRY_SECRET=registry-credentials
    export TAP_BUILD_REGISTRY_HOSTNAME=
    export TAP_BUILD_REGISTRY_USERNAME=
    export TAP_BUILD_REGISTRY_PASSWORD=
    ```

    And then we use the Tanzu CLI to create the secret:

    ```sh
    tanzu secret registry add ${TAP_BUILD_REGISTRY_SECRET} \
    --username ${TAP_BUILD_REGISTRY_USERNAME} \
    --password ${TAP_BUILD_REGISTRY_PASSWORD} \
    --server   ${TAP_BUILD_REGISTRY_HOSTNAME} \
    --namespace ${TAP_INSTALL_NAMESPACE} \
    --export-to-all-namespaces \
    --yes
    ```

### Create Package Repository

To make the values less abstract, let's look at an example.

Assume we have a internal Harbor registry, with the hostname `harbor.example.com`.
We create a project in this Harbor instance called `tap`, and relocated the TAP packages to this project as `tap-packages`.

The complete URL will now be: `harbor.example.com/tap/tap-packages:1.5.0`.

And our environment variables will be:

* `INSTALL_REGISTRY_REPO=tap`
* `INSTALL_REGISTRY_HOSTNAME=harbor.example.com`
* `TAP_VERSION=1.5.0`

Set the environment variables to values appropriate for your environment.

```sh
export TAP_INSTALL_NAMESPACE=tap-install
export TAP_VERSION=1.5.0
export INSTALL_REGISTRY_REPO=tap
export INSTALL_REGISTRY_HOSTNAME=${TAP_INSTALL_REGISTRY_HOSTNAME}
```

And use the Tanzu CLI to create the Package Repository for TAP in the TAP install Namespace (usually `tap-install`).

```sh
tanzu package repository add tanzu-tap-repository \
  --url ${INSTALL_REGISTRY_HOSTNAME}/${INSTALL_REGISTRY_REPO}/tap-packages:${TAP_VERSION} \
  --namespace ${TAP_INSTALL_NAMESPACE}
```

Run the command below to verify the Package Repository is reconciled successfully:

```sh
tanzu package repository list -n ${TAP_INSTALL_NAMESPACE}
```

This should return something like this:

```sh
  NAME                      SOURCE                                                                            STATUS
  tanzu-tap-repository      (imgpkg) harbor.services.h2o-2-9349.h2o.vmware.com/tap/tap-packages:1.5.0         Reconcile succeeded
```

### Configure Certificate Authority Bundle For Crossplane

The use of external services via the [Bitnami Services](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/bitnami-services-tutorials-working-with-bitnami-services.html)[^14] feature relies on [Crossplane](https://www.crossplane.io/)[^15].

Unfortunately, in the `1.5.0` release of TAP, the Crossplane package does not pickup the `shared.ca_cert_data` property.

This means we must configure Crossplane ourselves.
Crossplane expects [a ConfigMap with a ca-bundle property](https://docs.crossplane.io/knowledge-base/guides/self-signed-ca-certs/)[^13], which we later configure when installing TAP Profile.

First, verify the `crossplane-system` namespace exists:

```sh
kubectl create namespace crossplane-system || true
```

And then create the ConfigMap with the expected name and value.

```sh
kubectl -n crossplane-system create cm ca-bundle-config \
  --from-file=ca-bundle=ca.crt
```

## Review TAP packages configuration options

Before we can install our TAP Profile as desired, we need to understand how to configure it.

We can take a look at the available packages and specifically the TAP package itself to discover what we can configure.

### Explore Packages

The first step, is to discover the packages available to us.

```sh
tanzu package available list --namespace ${TAP_INSTALL_NAMESPACE}
```

This is a large list, so we'll limit the expected output:

```sh
  NAME                                                 DISPLAY-NAME
  accelerator.apps.tanzu.vmware.com                    Application Accelerator for VMware Tanzu
  api-portal.tanzu.vmware.com                          API portal
  apis.apps.tanzu.vmware.com                           API Auto Registration for VMware Tanzu
  ...
```

To see what we can configure with TAP, we have to take a few steps:

```sh
tanzu package available get tap.tanzu.vmware.com --namespace ${TAP_INSTALL_NAMESPACE}
```

This returns the versions of the TAP main package available to us.

You can also get a list of available packages and their version via `kubectl`:

```sh
kubectl get package -n ${TAP_INSTALL_NAMESPACE}
```

Which results in something like this:

```sh
NAME                                                            PACKAGEMETADATA NAME                                  VERSION                         AGE
antrea.tanzu.vmware.com.1.7.2+vmware.1-tkg.1-advanced           antrea.tanzu.vmware.com                               1.7.2+vmware.1-tkg.1-advanced   646h52m30s
capabilities.tanzu.vmware.com.0.28.1+vmware.1                   capabilities.tanzu.vmware.com                         0.28.1+vmware.1                 646h52m34s
kube-vip-cloud-provider.tanzu.vmware.com.0.0.4+vmware.2-tkg.1   kube-vip-cloud-provider.tanzu.vmware.com              0.0.4+vmware.2-tkg.1            646h52m33s
...

```

Eitherway, we now know the version of the TAP package available to us, `1.5.0`.
We need this information if we want to see the possible values for the TAP package.

We add the version to the package name (`<packageName>/<packageVersion>`), and then add the `--values-schema` flag.

```sh
tanzu package available get tap.tanzu.vmware.com/1.5.0 \
  --namespace ${TAP_INSTALL_NAMESPACE} \
  --values-schema
```

This results in a long list of possible values:

```sh
  KEY                                                  DEFAULT                 TYPE    DESCRIPTION
  appliveview                                                                  object  App Live View configuration
  contour.envoy.service.type                           LoadBalancer            string  Set to LoadBalancer by default; valid values are LoadBalancer, NodePort and
                                                                                       ClusterIP(except for contour.infrastructure_provider=vsphere)
  crossplane                                                                   object  Crossplane configuration
  image_policy_webhook                                                         object  Image Policy Webhook configuration
  learningcenter                                                               object  Learning Center configuration
  ...

```

As you can probably tell, many of these values are not explained at this level.

For example, the `TYPE` of `crossplane` is `object`.
We will have to explore the values schema of the Crossplane package to discover what we can configure here if we need to.

There is a lot of overlap between the packages, especially with configuration options such as a Domain name, secrets, or a custom CA.

These are conventient configurable via the `shared` configuration key:

```sh
tanzu package available get tap.tanzu.vmware.com/1.5.0 \
  --namespace ${TAP_INSTALL_NAMESPACE} \
  --values-schema | grep shared.
```

This gives the following list:

```sh
  shared.image_registry.password                       ""                      string  Optional: Password for the image registry. Mutually exclusive with
                                                                                       shared.image_registry.secret.name/namespace.
  shared.image_registry.project_path                   ""                      string  Optional: Project path in the image registry server used for builder and
  shared.image_registry.secret.name                    ""                      string  Optional: Secret name for the image registry credentials of
                                                                                       shared.image_registry.username/password.
  shared.image_registry.secret.namespace               ""                      string  Optional: Secret namespace for the image registry credentials. Mutually
                                                                                       exclusive with shared.image_registry.username/password.
  shared.image_registry.username                       ""                      string  Optional: Username for the image registry. Mutually exclusive with
                                                                                       shared.image_registry.secret.name/namespace.
  shared.ingress_domain                                ""                      string  Optional: Domain name to be used in service routes and hostnames for instances
  shared.ingress_issuer                                tap-ingress-selfsigned  string  Optional: A cert-manager.io/v1/ClusterIssuer for issuing TLS certificates to TAP
  shared.kubernetes_distribution                       ""                      string  Optional: Type of K8s infrastructure being used. Can be used in coordination
  shared.kubernetes_version                            ""                      string  Optional: K8s version. Can be used independently or in coordination with
  shared.activateAppLiveViewSecureAccessControl                                bool    Optional: Enable Secure Access Connection between App Live View Components
  shared.ca_cert_data                                  ""                      string  Optional: PEM Encoded certificate data to trust TLS connections with a private
```

Let's dive into the Full profile next.

### Full Profile Example From Docs

The documentation has an annotated [Full profile configuration file](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/install.html#full-profile-3)[^7] example, you can expand the example below if you want to see it.

Let's look at how we use YTT to template the Profile configuration.

??? Example "Full Profile Example From Docs"

    ```yaml title="full-profile.yaml"
    shared:
      ingress_domain: "INGRESS-DOMAIN"
      ingress_issuer: # Optional, can denote a cert-manager.io/v1/ClusterIssuer of your choice. Defaults to "tap-ingress-selfsigned".

      image_registry:
        project_path: "SERVER-NAME/REPO-NAME"
        secret:
          name: "KP-DEFAULT-REPO-SECRET"
          namespace: "KP-DEFAULT-REPO-SECRET-NAMESPACE"

      kubernetes_distribution: "K8S-DISTRO" # Only required if the distribution is OpenShift and must be used with the following kubernetes_version key.

      kubernetes_version: "K8S-VERSION" # Required regardless of distribution when Kubernetes version is 1.25 or later.

      ca_cert_data: | # To be passed if using custom certificates.
          -----BEGIN CERTIFICATE-----
          MIIFXzCCA0egAwIBAgIJAJYm37SFocjlMA0GCSqGSIb3DQEBDQUAMEY...
          -----END CERTIFICATE-----

    ceip_policy_disclosed: FALSE-OR-TRUE-VALUE # Installation fails if this is not set to true. Not a string.

    #The above keys are minimum numbers of entries needed in tap-values.yaml to get a functioning TAP Full profile installation.

    #Below are the keys which may have default values set, but can be overridden.

    profile: full # Can take iterate, build, run, view.

    supply_chain: basic # Can take testing, testing_scanning.

    ootb_supply_chain_basic: # Based on supply_chain set above, can be changed to ootb_supply_chain_testing, ootb_supply_chain_testing_scanning.
      registry:
        server: "SERVER-NAME" # Takes the value from the shared section by default, but can be overridden by setting a different value.
        repository: "REPO-NAME" # Takes the value from the shared section by default, but can be overridden by setting a different value.
      gitops:
        ssh_secret: "SSH-SECRET-KEY" # Takes "" as value by default; but can be overridden by setting a different value.

    contour:
      envoy:
        service:
          type: LoadBalancer # This is set by default, but can be overridden by setting a different value.

    buildservice:
      # Takes the value from the shared section by default, but can be overridden by setting a different value.
      kp_default_repository: "KP-DEFAULT-REPO"
      kp_default_repository_secret: # Takes the value from the shared section above by default, but can be overridden by setting a different value.
        name: "KP-DEFAULT-REPO-SECRET"
        namespace: "KP-DEFAULT-REPO-SECRET-NAMESPACE"

    tap_gui:
      service_type: ClusterIP # If the shared.ingress_domain is set as earlier, this must be set to ClusterIP.
      metadataStoreAutoconfiguration: true # Create a service account, the Kubernetes control plane token and the requisite app_config block to enable communications between Tanzu Application Platform GUI and SCST - Store.
      app_config:
        catalog:
          locations:
            - type: url
              target: https://GIT-CATALOG-URL/catalog-info.yaml

    metadata_store:
      ns_for_export_app_cert: "MY-DEV-NAMESPACE"
      app_service_type: ClusterIP # Defaults to LoadBalancer. If shared.ingress_domain is set earlier, this must be set to ClusterIP.

    scanning:
      metadataStore:
        url: "" # Configuration is moved, so set this string to empty.

    grype:
      namespace: "MY-DEV-NAMESPACE"
      targetImagePullSecret: "TARGET-REGISTRY-CREDENTIALS-SECRET"
      # In a single cluster, the connection between the scanning pod and the metadata store happens inside the cluster and does not pass through ingress. This is automatically configured, you do not need to provide an ingress connection to the store.

    policy:
      tuf_enabled: false # By default, TUF initialization and keyless verification are deactivated.
    tap_telemetry:
      customer_entitlement_account_number: "CUSTOMER-ENTITLEMENT-ACCOUNT-NUMBER" # (Optional) Identify data for creating the Tanzu Application Platform usage reports.
    ```

### Full Profile Template

Usually you will have more than one environment and possibly more than one of the same profile in the same environment.
So you end up installing the same Profile more than once.

It is recommended to use some kind of templating.

In the world of Tanzu (and TAP), it makes sense to use YTT.

```yaml title="full-profile.ytt.yaml"

#@ load("@ytt:data", "data")
#@ dv = data.values
#@ kpRegistry = "{}/{}".format(dv.buildRegistry, dv.tbsRepo)
---
profile: full

shared:
  ingress_domain: #@ dv.domainName
  ca_cert_data: #@ dv.caCert
  image_registry:
    secret:
      name: #@ dv.buildRegistrySecret
      namespace: tap-install

buildservice:
  pull_from_kp_default_repo: true
  exclude_dependencies: true
  kp_default_repository: #@ kpRegistry
  kp_default_repository_secret:
      name: #@ dv.buildRegistrySecret
      namespace: tap-install

supply_chain: basic
ootb_supply_chain_basic:
  registry:
    server: #@ dv.buildRegistry
    repository: #@ dv.buildRepo

appliveview_connector:
  backend:
    sslDeactivated: true
    ingressEnabled: true
    host: #@ "appliveview."+dv.domainName

appliveview:
  ingressEnabled: true
  server:
    tls:
      enabled: false

tap_gui:
  service_type: ClusterIP
  app_config:
    auth:
      allowGuestAccess: true
    customize:
      custom_name: 'Portal McPortalFace'
    organization:
      name: 'Org McOrg Face'
    catalog:
      locations:
        - type: url
          target: https://github.com/joostvdg/tap-catalog/blob/main/catalog-info.yaml
        - type: url
          target: https://github.com/joostvdg/tap-hello-world/blob/main/catalog/catalog-info.yaml

crossplane:
  registryCaBundleConfig:
    name: ca-bundle-config
    key: ca-bundle

#! reduces memory and CPU requirements, not recommended for production
#! but our Lab environments have resource restrictions
cnrs:
  lite:
    enable: true 

contour:
  envoy:
    service:
      type: LoadBalancer

ceip_policy_disclosed: true
excluded_packages:
  - scanning.apps.tanzu.vmware.com #! disabled for now, enabled when we upgrade OOTB Basic to Test & Scanning
  - grype.scanning.apps.tanzu.vmware.com #! disabled for now, enabled when we upgrade OOTB Basic to Test & Scanning
  - policy.apps.tanzu.vmware.com #! disabled for now, enabled when we upgrade OOTB Basic to Test & Scanning
  - eventing.tanzu.vmware.com #! not used, so removing to reduce memory/cpu footprint
  - tap-telemetry.tanzu.vmware.com.0.5.0-build #! not used, so removing to reduce memory/cpu footprint
```

!!! Tip "Template Explained"

    `shared`: values that are to be shared across any package that has the same configuration option. 
      It is here we configure Install Registry credentials and custom CA.

    `buildservice`: if your environment has no or restricted access to the internet, you need to manage the depencies yourself.
      It can re-use the repository from the shared configuration. Here we do that, and tell it to exclude the depencies as we install them ourselves.

    `supply_chain`: TAP can install one Cartographer Supply Chain fully*. You specify the Supply Chain by name,
      and then configure the Supply Chain by its own name**, in our case `basic` and `ootb_supply_chain_basic`. 
      Where we override the default Registry and Repository; this is where our built images are pushed.

    `appliveview`: the App Live View component let's use view registered application's resources existing across TAP cluster in the TAP GUI.
      Note, this is a TAP GUI component and that is where you will find it.

    `tap_gui`: the TAP GUI is the way to discover APIs, applications, accelerators and the like. It is Backstage with a few VMware Tanzu plugins included.
      It has a lot of configuration options, most of which can be ignored for an initial installation. 
      To use TAP GUI, you have to specify an authorization provider. In the event you do not have one (ready), you set **app_config.auth.allowGuestAccess** to true.

    `crossplane`: Crossplane is a new package for TAP 1.5, and unfortunately does not (yet) pick up the Shared CA configuration. So we have to configure this ourselves.
      TAP provides external services for use with the Service Binding Specification. By default it includes Crossplane packages for several Bitnami Helm Charts (packed as Bitnami Services).

    `ceip_policy_disclosed`: if you accept the policy or not. If you set this to false, you cannot install TAP

    `excluded_packages`: the packages we exclude are from the Test & Scanning Supply Chain. They require some configuration, especially if you are in a restricted internet access environment.
      Because we are using the Basic Supply Chain, these packages are not required and we can exclude them.

    * = the Test and Test & Scanning Supply Chains do miss a Tekton task, but are complete beyond that

    ** = each Supply Chain is its own Carvel Package part of the TAP Packages Repository

### Tanzu Build Service

TAP includes a version of Tanzu Build Service (TBS).
One of the things TBS needs, is a bunch of container images for the various Build Packs and the various versions of tech stacks they support.

A full set of depencies is ~12 GB, and TBS starts synchronizing these to the nodes once it is up and running.
There are two complications with this, one, thats a lot of data to synchronize from the outside, and two, you might not be able to reach the outside.

So what we do instad, is to [relocate the TBS depencies](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/install.html#optional-additional-build-service-configurations-4)[^8], in the same way as we do with the TAP packages.

We configured this in the TAP Profile already, by specifying the following:

```yaml
buildservice:
  exclude_dependencies: true
```

We cannot install the TAP TBS Dependencies (a TAP specific bundle of TBS) until we install TAP.
The TAP TBS Dependencies package depends on CRDs that are installed by TAP. So we will continue this later.

## Install TAP Profile

At this point, we have the following:

* TAP Packages available in a local Image Registry
* TAP TBS Dependencies available in a local Image Registry
* A Namespace to install TAP into (the concention is `tap-install`)
* Read access secret to the local Image Registry to retrieve the TAP packages
* Write access secret to the local Image Registry to write application build images to
* Profile Template for our Profile of choice (`Full`)

We now take the following steps:

1. Generate a Profile config file from the YTT Template
1. Install the TAP package
1. Install the TBS Dependencies Package Repository
1. Install the TBS Dependencies Package

### Generate Profile Config FIle

First, we set the required variables:

!!! Warning
    You should have received the address of the Build Registry and the Domain specific to your cluster!

```sh
export TAP_BUILD_REGISTRY_SECRET=registry-credentials
export BUILD_REGISTRY_REPO=tap-apps
export TBS_REPO=buildservice/tbs-full-deps
export CA_CERT=$(cat ca.crt)
export BUILD_REGISTRY=
export DOMAIN_NAME=
```

And then we run YTT to generate our Profile configuration file.

```sh
ytt -f full-profile.ytt.yaml \
  -v buildRegistry="$BUILD_REGISTRY" \
  -v buildRegistrySecret="$BUILD_REGISTRY_SECRET" \
  -v buildRepo="$BUILD_REGISTRY_REPO" \
  -v tbsRepo="$TBS_REPO" \
  -v domainName="$DOMAIN_NAME" \
  -v caCert="${CA_CERT}" \
  > "tap-values-full.yml"
```

We recommend you inspect the generated file:

```sh
cat tap-values-full.yml
```

### Install TAP Package

We are now ready to install TAP.

In case you don't remember or your environment variables are no longer set, set them to the appropriate values:

```sh
export TAP_INSTALL_NAMESPACE=tap-install
export TAP_VERSION=1.5.0
```

We can then [install TAP](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/install.html#install-your-tanzu-application-platform-package-5)[^9] via the Tanzu CLI.

```sh
tanzu package install tap \
  -p tap.tanzu.vmware.com \
  -v $TAP_VERSION \
  --values-file tap-values-full.yml \
  -n ${TAP_INSTALL_NAMESPACE}
```

You can get an overview of the Packages that are installed via the Tanzu CLI:

```sh
tanzu package installed list -n ${TAP_INSTALL_NAMESPACE}
```

Or via `kubectl`:

```sh
kubectl get app -n ${TAP_INSTALL_NAMESPACE}
```

If there is an issue, you can debug the package installs via `kubectl` commands:

```sh
kubectl describe app -n ${TAP_INSTALL_NAMESPACE} ${APP}
```

!!! Hint "Carvel Tips"

    Some Carvel tips worth mentioning:

    1. If you need to update an installation, you can run `tap package install` again. The Tanzu CLI handles the difference between Install and Update for you.
    1. KAPP works with asynchronous loops. This can cause the Client (Tanzu CLI) to break out of the watch loop when it detects the faillure of the **previous** reconcilliation. Don't be alarmed, and check with `kubectl get app` or Tanzu CLI (below)  to verify if the update succeeded.
      ```sh
      tanzu package installed list
      ```
    1. If you are sure there's nothing holding back a successful reconciliation, but it isn't updating (yet), you can force a reconcilliation with `kick`: 
      ```sh
      tanzu package installed kick ${APP} -n ${TAP_INSTALL_NAMESPACE}
      ```
    1. If you delete the TAP install, and then re-install it, remember that an uninstall removes all associated Namespaces as well. So you will have to do the [Crossplane prep](/tap-workshops/install/basic/#configure-certificate-authority-bundle-for-crossplane) again.

If everything is still reconciling and you want to wait on the TAP app to succeed:

```sh
kubectl wait --for=condition=ReconcileSucceeded app \
  -n ${TAP_INSTALL_NAMESPACE} tap \
  --timeout=10m
```

!!! Warning "Takes about 7-8 minutes"

    ```sh
    NAME  DESCRIPTION           SINCE-DEPLOY   AGE
    tap   Reconcile succeeded   4m10s          7m6s
    ```

### Determine TBS Version

Now that we have the TAP package installed, we can install the TBS dependencies.

First, we need to verify which version of TBS TAP shipped[^8].

```sh
tanzu package available list buildservice.tanzu.vmware.com \
   --namespace ${TAP_INSTALL_NAMESPACE}
```

Which for TAP `1.5.0` gives the following result:

```sh
  NAME                           VERSION  RELEASED-AT
  buildservice.tanzu.vmware.com  1.10.8   -
```

We then set the environment variable, so we can re-use it:

```sh
export TBS_VERSION=1.10.8
```

!!! Note
    Your environment should already have the TBS Dependencies relocated for you.

    If you want to know how to do so, you can find it documented [here](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/install.html#optional-additional-build-service-configurations-4)[^8]

### Install TBS Depencies Package Repository

In case you have not yet, set the environment variables.

```sh
export TBS_REPO=buildservice/tbs-full-deps
```

And we then we can install the TBS Dependencies Package Repository:

```sh
tanzu package repository add tbs-full-deps-repository \
  --url ${INSTALL_REGISTRY_HOSTNAME}/${TBS_REPO}:${TBS_VERSION} \
  --namespace ${TAP_INSTALL_NAMESPACE}
```

We can verify the status via the Tanzu CLI:

```sh
tanzu package repository list -n ${TAP_INSTALL_NAMESPACE}
```

Which should now yield this:

```sh
  NAME                      SOURCE                                                                            STATUS
  tanzu-tap-repository      (imgpkg) harbor.services.h2o-2-9349.h2o.vmware.com/tap/tap-packages:1.5.0         Reconcile succeeded
  tbs-full-deps-repository  (imgpkg)                                                                          Reconcile succeeded
                            harbor.services.h2o-2-9349.h2o.vmware.com/buildservice/tbs-full-deps:1.10.8
```

### Install TBS Full Dependencies Package

We can now install the TBS Full Dependencies package[^8]:

```sh
tanzu package install full-tbs-deps \
  -p full-tbs-deps.tanzu.vmware.com \
  -v ${TBS_VERSION} \
  -n ${TAP_INSTALL_NAMESPACE}
```

And then verify the package is installed and reconciled correctly:

```sh
tanzu package installed get full-tbs-deps -n $TAP_INSTALL_NAMESPACE
```

Which should return something like this:

```sh
NAMESPACE:          tap-install
NAME:               full-tbs-deps
PACKAGE-NAME:       full-tbs-deps.tanzu.vmware.com
PACKAGE-VERSION:    1.10.8
STATUS:             Reconcile succeeded
CONDITIONS:         - type: ReconcileSucceeded
  status: "True"
  reason: ""
  message: ""
```

## Install Test Application

* explain workloads
* CLI
* namespace & namespace provisioner
* ?

One of the goals of TAP is to be flexible, to support the various ways people build and run applications.

Cartographer's Supply Chains let you define any workflow you want for any kind of resource you can express in Kubernetes.

While that is interesting, starting with the basics, we focus on the `batteries included` part.
Which gives you three main workflows:

* build an application
* run an application
* both build & run an application

Each of the three Out-Of-The-Box (OOTB) Supply Chains comes with two `ClusterSupplyChain` CRs.

* `x-image-to-url`
* `source-x-to-url`

The Source to Image journey is defined by the `Workload` CR.
The Image to URL journey is defined by the `Deliverable` CR.

When you start at the Source, the OOTB Supply Chains generate the `Deliverable` CR for you.

To test if our TAP machinery works as intended, we'll stick to creating a `Workload` that uses the `source-to-url` Supply Chain.

In order to the tools used by the Supply Chain to do their work, they need the appropriate Secrets and RBAC permissions.
So we start with setting up a Developer Namespace.

### Set Up Developer Namespace

Starting with TAP 1.5, it includes a package called the [Namespace Provisioner](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/namespace-provisioner-about.html)[^10].

This let's us configure Developer Namespaces by adding Label.

```sh
export TAP_DEVELOPER_NAMESPACE=dev
```

=== "Via kubectl"

    ```sh
    kubectl create namespace ${TAP_DEVELOPER_NAMESPACE}
    
    kubectl label namespace ${TAP_DEVELOPER_NAMESPACE} \
      apps.tanzu.vmware.com/tap-ns=""
    ```
=== "Via Manifest"

    ```sh
    cat <<EOF > tap-dev-namespace.yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      labels:
        apps.tanzu.vmware.com/tap-ns: ""
        kubernetes.io/metadata.name: ${TAP_DEVELOPER_NAMESPACE}
      name: ${TAP_DEVELOPER_NAMESPACE}
    EOF
    ```

    ```sh
    kubectl apply -f tap-dev-namespace.yaml
    ```

If we wait a few moments, we can then see the Namespace contains Secrets and RoleBindings:

```sh
kubectl get secret,rolebinding -n $TAP_DEVELOPER_NAMESPACE
```

Which shows something like this:

```sh
NAME                            TYPE                             DATA   AGE
secret/registries-credentials   kubernetes.io/dockerconfigjson   1      25d

NAME                                                            ROLE                   AGE
rolebinding.rbac.authorization.k8s.io/default-permit-workload   ClusterRole/workload   25d
```

### Create Workload

Now that we have a Namespace to work in, we can define a Workload.

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

Use `kubectl wait` to wait for the app to be ready.

```sh
kubectl wait --for=condition=Ready Workload smoke-app --timeout=10m -n ${$TAP_DEVELOPER_NAMESPACE}
```

### Verify Workload

To see the logs:

=== "Tanzu CLI"
    ```sh
    tanzu apps workload tail smoke-app --timestamp --since 1h \
      -n ${TAP_DEVELOPER_NAMESPACE}
    ```
=== "Stern"
    ```sh
    stern -n ${TAP_DEVELOPER_NAMESPACE} -l app.kubernetes.io/part-of=smoke-app
    ```

To get the status:

=== "Tanzu CLI"
    ```sh
    tanzu apps workload get smoke-app \
        -n ${TAP_DEVELOPER_NAMESPACE}
    ```
=== "Kubectl"
    ```sh
    kubectl get workload smoke-app \
      -n ${TAP_DEVELOPER_NAMESPACE}
    ```

And to verify the Image build (kpack):

```sh
kubectl describe images.kpack.io smoke-app\
   -n ${TAP_DEVELOPER_NAMESPACE}
```

### Call App Endpoint

Collect the endpoint:

```sh
kubectl get httpproxy -n ${TAP_DEVELOPER_NAMESPACE}
```

Which should return something like this:

```sh
NAME                                                              FQDN                                            TLS SECRET                                       STATUS   STATUS DESCRIPTION
smoke-app-contour-smoke-app.dev                                   smoke-app.dev                                                                                    valid    Valid HTTPProxy
smoke-app-contour-smoke-app.dev.lab02.h2o-2-9349.h2o.vmware.com   smoke-app.dev.lab02.h2o-2-9349.h2o.vmware.com   dev/route-337ba674-14e1-470c-9733-0b1bab8922b4   valid    Valid HTTPProxy
smoke-app-contour-smoke-app.dev.svc                               smoke-app.dev.svc                                                                                valid    Valid HTTPProxy
smoke-app-contour-smoke-app.dev.svc.cluster.local                 smoke-app.dev.svc.cluster.local                                                                  valid    Valid HTTPProxy
```

Copy the URL that has an FQDN with this structure: `<app>.<namespace>.<labName>.h2o-2-9349.h2o.vmware.com`:

```sh
export URL=
```

And then you can curl:

```sh
curl -lk "http://${URL}"
```

## Register and View App in TAP GUI

* Open TAP GUI
* View Workload's Supply Chain
* Update TAP GUI Config / TAP install
* Register Application
* View Workload in App Live View
* Point to Hello Workload for managing an App with updates to source code (via Gitea)

## Cleanup

### Delete Workload

And then we can delete our test workload if want to.

```sh
tanzu apps workload delete smoke-app -y -n ${TAP_DEVELOPER_NAMESPACE}
```

## Links

[^1]: [TAP 1.5 - Prerequisites](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/prerequisites.html)
[^2]: [TAP 1.5 - Supported Kubernetes versions](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/prerequisites.html#kubernetes-cluster-requirements-3)
[^3]: [TAP 1.5 - Relocate images to a registry](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/install.html#relocate-images-to-a-registry-0)
[^4]: [TAP 1.5 - Cluster Essentials](https://docs.vmware.com/en/Cluster-Essentials-for-VMware-Tanzu/1.5/cluster-essentials/deploy.html)
[^5]: [KAPP Controller - Overview](https://carvel.dev/kapp-controller/)
[^6]: [SecretGen Controller - GitHub page](https://github.com/carvel-dev/secretgen-controller)
[^7]: [TAP 1.5 - Full profile example](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/install.html#full-profile-3)
[^8]: [TAP 1.5 - Handle TBS Depencies](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/install.html#optional-additional-build-service-configurations-4)
[^9]: [TAP 1.5 - Install TAP Package](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/install.html#install-your-tanzu-application-platform-package-5)
[^10]: [TAP 1.5 - Namespace Provisioner](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/namespace-provisioner-about.html)
[^11]: [TAP 1.5 - Trust CA for Workload specifically](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/app-sso-service-operators-workload-trust-custom-ca.html)
[^12]: [KAPP Controller - Configure Controller to Trust custom CA](https://carvel.dev/kapp-controller/docs/v0.43.2/controller-config/#controller-configuration-spec)
[^13]: [Crossplane - Configure Self-signed CA Certs support](https://docs.crossplane.io/knowledge-base/guides/self-signed-ca-certs/)
[^14]: [TAP 1.5 - Bitnami Services](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/bitnami-services-tutorials-working-with-bitnami-services.html)
[^15]: [Crossplane - The Cloud Native Control Plane Framework](https://www.crossplane.io/)
