---
title: Create Supply Chain
description: Create a custom TAP Supply Chain
author: joostvdg
tags: [tap, kubernetes, cartographer, tekton, supplychain]
---

## Questions

* FluxCD Webhook instead of Polling
  * Polling must die
* Git metadata (e.g., Git Webhook data)?
* Other branches, tags, PRs?
* Handle multiple commits to same revision in sequence?
* Build caching?
* Tekton GUI?
* Carto Live Editor?

## Goals & Outcomes

* I've written and implemented my own Custom Supply Chain on a Cluster
* ?

## Steps

* Cartographer Introduction
* Tekton Introduction
* Integrating Tekton into Cartographer
* Compare with OOTB Supply Chains

We start with exploring the core components of Cartographer.
First with hardcoded values, then with some templating involved.

Eventually, for a Supply Chain, we need something to run commands in a container.
For this, we rely on Tekton.

Before we add the Tekton bits to the Cartographer Supply Chain, we take a look a Tekton's core components.
We then join the two in a more elaborate Supply Chain.

And lastly, you should now be able to read and comprehend the OOTB Supply Chains.

## Prerequisites

* TAP Profile installed which includes Tekton and Cartographer
  * Full, Iterate, or Build
* TAP Developer Namespace setup
  * for RBAC and Repository Credentials
* Checkout this repository

## Cartographer Introduction

The Cartographer docs have a [good tutorial](https://cartographer.sh/docs/v0.7.0/tutorials/first-supply-chain/)[^1], which we'll follow in condensed form.

If you'd rather follow the tutorial from there, the files included in this repository have minor changes, it is recommend to compare them.
Just make sure to apply the files into a namespace that is configured as a TAP Developer Namespace.

We'll start with a the minimal configuration and then build it out:

1. My First Supply Chain
1. Include Parameters
1. Git Checkout and Image Build

To construct and use a Supply Chain, we need the following ingredients:

1. One or more ***Resources***, usually Cartographer templates
1. A SupplyChain definition, using those Resources
1. A Workload definition that _selects_ the Supply Chain

### My First Template

!!! Tip "Relies on Workload Definition"

    Cartographer works from a **Workload** definition.

    So any template, like the **ClusterTemplate** below, works because the trigger is a **Workload** CR.

As the name implies, a ClusterTemplate is a resource that generates another resource.
You define that other resource via the `spec.template` field.

```yaml
apiVersion: carto.run/v1alpha1
kind: ClusterTemplate
metadata:
  name: app-deploy
spec:
  template:
```

In the case of our example, we will templatize a Deployment.

Parameters for the template come from upstream resources, such as the **Workload** that triggers a Supply Chain.
We define the parameter as `$(source.propery)$`, note the double `$`.

```yaml
template:
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: $(workload.metadata.name)$-deployment
```

A **Workload** is a namespaced Kubernetes CR, which we can leverage to fill in all required (and unique) fields of a **Deployment**.

??? Example "ClusterTemplate"

    ```yaml title="resources/cartographer/app-deploy-01/01-cluster-template.yaml"
    apiVersion: carto.run/v1alpha1
    kind: ClusterTemplate
    metadata:
      name: app-deploy
    spec:
      template:
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: $(workload.metadata.name)$-deployment
          labels:
            app: $(workload.metadata.name)$
        spec:
          replicas: 3
          selector:
            matchLabels:
              app: $(workload.metadata.name)$
          template:
            metadata:
              labels:
                app: $(workload.metadata.name)$
            spec:
              containers:
                - name: $(workload.metadata.name)$
                  image: $(workload.spec.image)$
    ```

Note the property `spec.image`. This is something we must define within the Workload to make this Template work.

### Supply Chain Definition

> The supply chain has three top level fields in its spec, the resources, a service account reference and a selector for workloads.

1. **Resources**: Which CRs are used (and how)
1. **ServiceAccount Reference**: reference to a ServiceAccount that can use the Templates/Resources
1. **Selector**: Set of Kubernetes **Labels** which you have to specify on a **Workload** to trigger this Supply Chain

The most minimal Supply Chain contains a single Resource.
The Cluster Template we just created is one of those resources.

There are five types of [Resources](https://cartographer.sh/docs/v0.7.0/tutorials/extending-a-supply-chain/#app-operator-steps)[^2]:

* **ClusterTemplate**: instructs the supply chain to instantiate a Kubernetes object that has no outputs to be supplied to other objects in the chain
* **ClusterSourceTemplates**: indicates how the supply chain could instantiate an object responsible for providing source code
* **ClusterImageTemplates**: instructs how the supply chain should instantiate an object responsible for supplying container images
* **ClusterDeploymentTemplate**: indicates how the delivery should configure the environment (namespace/cluster)
* **ClusterConfigTemplates**: Instructs the supply chain how to instantiate a Kubernetes object that knows how to make Kubernetes configurations available to further resources in the chain

You can find the details of each resource [in the docs](https://cartographer.sh/docs/v0.7.0/reference/template/)[^3]

One of the goals of Cartographer is to be the abstraction layer above CI/CD Pipelines.
As such, the majority of the resources make the most sense with a Cluster scope.

To make the disctintion clear, all Cluster scoped CRs start with `Cluster`.
This includes the Supply Chain, so our empty Supply Chain looks like this:

```yaml
apiVersion: carto.run/v1alpha1
kind: ClusterSupplyChain
metadata:
  name: my-first-supply-chain
spec:
  resources: []
  serviceAccountRef: {}
  selector: {}
```

Each resource has a `name` and `templateRef` field.
And the `templateRef` field maps to a Kubernetes CR with `kind` and `name`.

For example, here we map our Cluster Template from earlier:

```yaml
resources:
  - name: deploy
    templateRef:
      kind: ClusterTemplate
      name: app-deploy
```

The `serviceAccountRef` requires the `name` and `namespace` of the **ServiceAccount** Cartographer uses.

The `selector` contains a Kubernetes **Label**:

```yaml
selector:
  workload-type: pre-built
```

See the complete example below.

??? Example "Supply Chain"

    ```yaml title="resources/cartographer/app-deploy-01/02-supply-chain.yaml"
    apiVersion: carto.run/v1alpha1
    kind: ClusterSupplyChain
    metadata:
      name: my-first-supply-chain
    spec:
      resources:
        - name: deploy
          templateRef:
            kind: ClusterTemplate
            name: app-deploy

      serviceAccountRef:
        name: cartographer-pre-built-sa
        namespace: default

      selector:
        workload-type: pre-built
    ```

### Workload Definition

Remember, we need to specify the `spec.image` field.

And if we want to trigger our Supply Chain, our Workload needs to contain the Label `workload-type=pre-built`.

```yaml title="workload-pre-built-hello.yaml"
apiVersion: carto.run/v1alpha1
kind: Workload
metadata:
  name: hello
  labels:
    workload-type: pre-built
spec:
  image: ghcr.io/joostvdg/go-demo:v2.1.16
```

### RBAC

Before we apply the resources to the cluster, we need to make sure our ServiceAccount exists and has the required permissions.

??? Example "ServiceAccount, Roles, and RoleBindings"

    ```yaml title="resources/cartographer/rbac.yaml"
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: cartographer-pre-built-sa
      namespace: default

    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: deploy-image-role
    rules:
      - apiGroups:
          - apps
        resources:
          - deployments
        verbs:
          - list
          - create
          - update
          - delete
          - patch
          - watch
          - get

    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: cartographer-prebuilt-role-binding
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: deploy-image-role
    subjects:
      - kind: ServiceAccount
        name: cartographer-pre-built-sa
    ```

### Excercise 1

We should now have four files, which you can apply to the Cluster and the appropriate Namespace.

```sh
export DEV_NAMESPACE=dev
```

```sh
kubectl apply -f resources/cartographer/rbac.yaml -n ${DEV_NAMESPACE}
kubectl apply -f resources/cartographer/app-deploy-01/01-cluster-template.yaml
kubectl apply -f resources/cartographer/app-deploy-01/02-supply-chain.yaml
kuebctl apply -f resources/cartographer/app-deploy-01/03-workload.yaml -n ${DEV_NAMESPACE}
```

Once you have applied the resources, the workload should be valid:

```sh
kubectl get workload -n $DEV_NAMESPACE
```

And if you have the `kubectl tree` plugin, you can see the related resources:

```sh
kubectl tree workload hello -n ${DEV_NAMESPACE}
```

Which should show something like this:

```sh
NAMESPACE  NAME                                       READY  REASON  AGE
default    Workload/hello                             True   Ready   98m
default    └─Deployment/hello-deployment              -              98m
default      └─ReplicaSet/hello-deployment-cfdf74d6   -              98m
default        ├─Pod/hello-deployment-cfdf74d6-kjjp5  True           98m
default        ├─Pod/hello-deployment-cfdf74d6-wgtvl  True           98m
default        └─Pod/hello-deployment-cfdf74d6-x2pmc  True           98m
```

To test the application, you can use `kubectl port-forward`:

```sh
kubectl port-forward deployment/hello-deployment -n $DEV_NAMESPACE 8080:8080
```

And then Curl:

```sh
curl http://localhost:8080/
```

## Adding Additional Parameters

A next step is to add more templating to our Supply Chain[^4].

One of the ways we can do this is via additional parameters.

### Add Paramters To ClusterTemplate

We extend the ClusterTemplate from the previous example.

I tend the rename updated examples:

```yaml
apiVersion: carto.run/v1alpha1
kind: ClusterTemplate
metadata:
  name: app-deploy-02
```

Parameters are a top level spec resource (`params`), which you can verify with `kubectl explain`.

```sh
kubectl explain ClusterTemplate.spec
```

Which lists among other things:

```sh
params	<[]Object>
  Additional parameters. See:
  https://cartographer.sh/docs/latest/architecture/#parameter-hierarchy
```

We'll add a single environment variable to our Deployment.
For this, we create two `params` entries:

```yaml
params:
  - name: env_key
    default: "FOO"
  - name: env_value
    default: "BAR"
```

Which we can use in our `spec.template` via `$(params.<nameOfParam>)$`:

```yaml
containers:
  - name: $(workload.metadata.name)$
    image: $(workload.spec.image)$
    env:
      - name: $(params.env_key)$
        value: $(params.env_value)$
```

??? Example "Updated ClusterTemplate"

    ```yaml title="resources/cartographer/app-deploy-02/01-cluster-template.yaml"
    apiVersion: carto.run/v1alpha1
    kind: ClusterTemplate
    metadata:
      name: app-deploy-02
    spec:
      template:
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: $(workload.metadata.name)$-deployment
          labels:
            app: $(workload.metadata.name)$
        spec:
          replicas: 3
          selector:
            matchLabels:
              app: $(workload.metadata.name)$
          template:
            metadata:
              labels:
                app: $(workload.metadata.name)$
            spec:
              containers:
                - name: $(workload.metadata.name)$
                  image: $(workload.spec.image)$
                  env:
                    - name: $(params.env_key)$
                      value: $(params.env_value)$
      params:
        - name: env_key
          default: "FOO"
        - name: env_value
          default: "BAR"
    ```

### Update Supply Chain To Use Updated Template

The parameters we added to the **ClusterTemplate** need to be supplied by the **Workload**.

So technically we do not have to change our **ClusterSupplyChain**.
I do so anyway, so we can have both Supply Chains in the cluster at the same time.

We change the Template Ref:

```yaml
- name: deploy
  templateRef:
    kind: ClusterTemplate
    name: app-deploy-02
```

If we want Workloads to select the new Supply Chain, we should also update the Selector:

```yaml
selector:
  workload-type: pre-built-02
```

??? Example "Updated Supply Chain"

    ```yaml title="resources/cartographer/app-deploy-02/02-supply-chain.yaml"
    apiVersion: carto.run/v1alpha1
    kind: ClusterSupplyChain
    metadata:
      name: supply-chain-02
    spec:
      resources:
        - name: deploy
          templateRef:
            kind: ClusterTemplate
            name: app-deploy-02

      serviceAccountRef:
        name: cartographer-pre-built-sa
        namespace: default

      selector:
        workload-type: pre-built-02
    ```

### Add Parameters To Workload

In the Workload we change a few more things.

We rename it, to `hello-02`, to differentiate from our previous Workload.

We update the Label to `workload-type: pre-built-02` to reflect the updated Supply Chain.

And, last but not least, we add the parameters!

We do so similarly as we did for the Cluster Template:

```yaml
params:
  - name: env_key
    value: "K_SERVICE"
  - name: env_value
    value: "carto-hello"
```

!!! Tip

    For those wondering what this parameter does.

    When run via Knative Serving (as is done in TAP) the `k_SERVICE` env variable is printed in the http get on the `/` endpoint.

    So setting this, let's us directly see our value in the output.

    ```sh
    Chart Version: ; Image Version: ; Release: unknown, SemVer: , GitCommit: ,Host: hello-04-deployment-7645c7c549-dpn8v, Revision: , Service: carto-hello-source
    ```

??? Example "Updated Workload"

    ```yaml title="resources/cartographer/app-deploy-02/03-workload.yaml"
    apiVersion: carto.run/v1alpha1
    kind: Workload
    metadata:
      name: hello-02
      labels:
        workload-type: pre-built-02
    spec:
      image: ghcr.io/joostvdg/go-demo:v2.1.16
      params:
        - name: env_key
          value: "K_SERVICE"
        - name: env_value
          value: "carto-hello"
    ```

### Excercise 2

```sh
export DEV_NAMESPACE=dev
```

Technically, we do not have to update the RBAC configuration, but its included for completeness.

```sh
kubectl apply -f resources/cartographer/rbac.yaml -n ${DEV_NAMESPACE}
kubectl apply -f resources/cartographer/app-deploy-02/01-cluster-template.yaml
kubectl apply -f resources/cartographer/app-deploy-02/02-supply-chain.yaml
kubectl apply -f resources/cartographer/app-deploy-02/03-workload.yaml  -n ${DEV_NAMESPACE}
```

Once you have applied the resources, the workload should be valid:

```sh
kubectl get workload -n $DEV_NAMESPACE
```

To test the application, you can use `kubectl port-forward`:

```sh
kubectl port-forward deployment/hello-02-deployment -n $DEV_NAMESPACE 8080:8080
```

And then Curl:

```sh
curl http://localhost:8080/
```

## Extending The Supply Chain

So far our Supply Chain deploys our application by generating a **Deployment**.

Now it is time to add more steps to our Supply Chain[^5], so we, you know, a Chain of steps instead of just one.

To stay in that theme, the **Deployment** needs to be supplied with a (container) **Image**.
So our next goal is to build that **Image** and supply it to the **Deployment**.

### Add Image Template

As doing CI/CD in Kubernetes often involves creating Container Images, Cartographer has a first-class support for this.

This capability is supplied by the CR **ClusterImageTemplate**[^6].

```yaml
apiVersion: carto.run/v1alpha1
kind: ClusterImageTemplate
metadata:
  name: image-builder-01
spec:
  template: {}
```

Cartographer doesn't actually build the image for us, it relies on third party tools.
One such tool, which matches really well with Cartographer, is [KPack](https://buildpacks.io/docs/tools/kpack/)[^7].

??? Example "KPack Image CR example"

    ```yaml
    apiVersion: kpack.io/v1alpha1
    kind: Image
    metadata:
      name: example-image
      namespace: default
    spec:
      tag: <DOCKER-IMAGE-TAG>
      serviceAccount: <SERVICE-ACCOUNT>
      builder:
        name: <BUILDER>
        kind: Builder
      source:
        git:
          url: <APPLICATION-SOURCE-REPOSITORY>
          revision: <APPLICATION-SOURCE-REVISION>
    ```

KPack has an **Image** CR, that requires us the provide the following values:

* **Tag**: the name of the Image to build
* **Builder**: a KPack Builder, which is a Kubernetes CR that configures [Cloud Native Buildpacks](https://buildpacks.io/)[^8]
* **Source**: the source code input for the image building process

The Tag speaks for itself, let's look at the Builder.

When installing TAP, it also includes Tanzu Build Service (TBS).
TBS installs and configures KPack, and thus we already have a set of KPack Builders available.

We can retrieve them as follows:

```sh
kubectl get clusterbuilder
```

Which in my environment returns the following (some values are abbreviated):

```sh
NAME         LATESTIMAGE                                                    READY
base         h.s.h.v.com/b../tbs-full-deps:clusterbuilder-base@s.......85   True
base-jammy   h.s.h.v.com/b../tbs-full-deps:clusterbuilder-base-jammy@..70   True
default      h.s.h.v.com/b../tbs-full-deps:clusterbuilder-default@.....57   True
full         h.s.h.v.com/b../tbs-full-deps:clusterbuilder-full@........2c   True
full-jammy   h.s.h.v.com/b../tbs-full-deps:clusterbuilder-full-jammy@..d0   True
tiny         h.s.h.v.com/b../tbs-full-deps:clusterbuilder-tiny@........96   True
tiny-jammy   h.s.h.v.com/b../tbs-full-deps:clusterbuilder-tiny-jammy@..9f   True
```

I like small Images (and I cannot lie), so I choose `tiny`:

```yaml
builder:
  kind: ClusterBuilder
  name: tiny
```

!!! Warning "Missing API Schema"
    Unfortunately, the KPack CRs do not contain the OpenAPI schema.

    So we cannot use `kubectl explain` to explore the schema.

For the source, we assume it is a Git repository, and require the **Workload** to specify it.

```yaml
source:
  git:
    url: $(workload.spec.source.git.url)$
    revision: $(workload.spec.source.git.ref.branch)$
```

For the **ClusterImageTemplate** itself, we need to specify two more things:

* **params*: an input parameter for the `image_prefix` for the **Tag**
  * `tag: $(params.image_prefix)$$(workload.metadata.name)$`
* **imagePath**: we need to specify where the template can retrieve the URI of the built Image, to supply it to the next steps

```yaml
params:
  - name: image_prefix
    default: harbor.services.h2o-2-9349.h2o.vmware.com/tap-apps/test
imagePath: .status.latestImage
```

??? Example "Full ClusterImageTemplate"

    ```yaml title="resources/cartographer/app-deploy-03/01-cluster-template.yaml"
    apiVersion: carto.run/v1alpha1
    kind: ClusterImageTemplate
    metadata:
      name: image-builder-01
    spec:
      template:
        apiVersion: kpack.io/v1alpha2
        kind: Image
        metadata:
          name: $(workload.metadata.name)$
        spec:
          tag: $(params.image_prefix)$$(workload.metadata.name)$
          builder:
            kind: ClusterBuilder
            name: tiny
          source:
            git:
              url: $(workload.spec.source.git.url)$
              revision: $(workload.spec.source.git.ref.branch)$
      params:
        - name: image_prefix
          default: harbor.services.h2o-2-9349.h2o.vmware.com/tap-apps/test
      imagePath: .status.latestImage
```

### Update Cluster Template

As with the other examples, we rename the **ClusterTemplate** so it does not replace our previous efforts.

```yaml title="resources/cartographer/app-deploy-03/01-cluster-template.yaml"
apiVersion: carto.run/v1alpha1
kind: ClusterTemplate
metadata:
  name: app-deploy-from-sc-image-01
```

We want our **Image** to come from within the Supply Chain, and no longer supplied by our **Workload**.

Later we'll declare in the Supply Chain how and where the Image comes from.
For now, let's assume we'll end up with the value supplied in the struct `images.built-image.image`.

```yaml
containers:
  - name: $(workload.metadata.name)$
    image: $(images.built-image.image)$
```

### Update RBAC

The SA we use needs permissions to use the KPack resources.

So we need to update our RBAC configuration:

??? "Updated RBAC Config"

    ```yaml title="resources/cartographer/app-deploy-03/00-rbac.yaml"
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: cartographer-from-source-sa

    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: deploy-image-role
    rules:
      - apiGroups:
          - apps
        resources:
          - deployments
        verbs:
          - list
          - create
          - update
          - delete
          - patch
          - watch
          - get

    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: cartographer-deploy-role-binding
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: deploy-image-role
    subjects:
      - kind: ServiceAccount
        name: cartographer-from-source-sa

    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: build-image-role
    rules:
      - apiGroups:
          - kpack.io
        resources:
          - images
        verbs:
          - list
          - create
          - update
          - delete
          - patch
          - watch
          - get

    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: cartographer-build-image-role-binding
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: build-image-role
    subjects:
      - kind: ServiceAccount
        name: cartographer-from-source-sa
    ```

### Update Supply Chain

Oke. so we created a second Cartographer **Resource**, the **ClusterImageTemplate**.

Let's add it to our Supply Chain.

We add it to the list of **Resources**:

```yaml
resources:
  - name: build-image
    templateRef:
      kind: ClusterImageTemplate
      name: image-builder-01
```

Our Deployment Template now needs to receive the Image URI from the Supply Chain.
So we need to add that information to the Template's Resource definition.

In case you don't remember, we assumed it would be defined within `images.built-image.image`.

You can see below how we can express that.
We add the `images` list, and link it to the output `built-image` from the ***Resource*** `build-image`.

The Resource `build-image` is the _Resource_ name of our **ClusterImageTemplate**. 

```yaml
- name: deploy
  templateRef:
    kind: ClusterTemplate
    name: app-deploy-from-sc-image-01
  images:
    - resource: build-image
      name: built-image
```

Below is the complete updated Supply Chain.

??? Example "Updated Supply Chain"

    ```yaml title="resources/cartographer/app-deploy-03/02-supply-chain.yaml"
    apiVersion: carto.run/v1alpha1
    kind: ClusterSupplyChain
    metadata:
      name: source-code-supply-chain-01
    spec:
      resources:
        - name: build-image
          templateRef:
            kind: ClusterImageTemplate
            name: image-builder-01
        - name: deploy
          templateRef:
            kind: ClusterTemplate
            name: app-deploy-from-sc-image-01
          images:
            - resource: build-image
              name: built-image

      serviceAccountRef:
        name: cartographer-from-source-sa
        namespace: dev

      selector:
        workload-type: source-code-01
    ```

### Update Workload

As the Image for our Deployment is now generated by the Supply Chain, we remove it from our Workload manifest.

Instead, we now need to supply the (Git) Source of our Workload.

```yaml
source:
  git:
    ref:
      branch: main
    url: https://github.com/joostvdg/go-demo
```

Below is the complete updated Workload manifest:

??? Example "Updated Workload"

    ```yaml title="resources/cartographer/app-deploy-03/03-workload.yaml"
    apiVersion: carto.run/v1alpha1
    kind: Workload
    metadata:
      name: source-code-01
      labels:
        workload-type: source-code-01
    spec:
      params:
        - name: env_key
          value: "K_SERVICE"
        - name: env_value
          value: "carto-hello-source"
      source:
        git:
          ref:
            branch: main
          url: https://github.com/joostvdg/go-demo
    ```

### Excercise 3

```sh
export DEV_NAMESPACE=dev
```

We can now apply all the Resources to the Cluster:

```sh
kubectl apply -f resources/cartographer/app-deploy-03/00-rbac.yaml -n ${DEV_NAMESPACE}
kubectl apply -f resources/cartographer/app-deploy-03/01-cluster-template.yaml
kubectl apply -f resources/cartographer/app-deploy-03/02-supply-chain.yaml
kubectl apply -f resources/cartographer/app-deploy-03/03-workload.yaml  -n ${DEV_NAMESPACE}
```

!!! Info

    The `01-cluster-template.yaml` file contains both the ClusterTemplate and the ClusterImageTemplate.

Once you have applied the resources, the workload should be valid:

```sh
kubectl get workload -n $DEV_NAMESPACE
```

Because we're now doing a build, you might want to follow allong with the logs:

```sh
tanzu apps workload tail source-code-01 --namespace dev --timestamp --since 1h
```

To test the application, you can use `kubectl port-forward`:

```sh
kubectl port-forward deployment/source-code-01-deployment -n $DEV_NAMESPACE 8080:8080
```

And then Curl:

```sh
curl http://localhost:8080/
```

## Tekton Introduction

At some point a Supply Chain needs to perform actions that are specific to your application or tech stack.

Usually by means of running a particular Container with some shell commands.

While you could use the ClusterTemplate to run a Pod with specific commands, there is a better tool for the job: [Tekton](https://tekton.dev/docs/getting-started/)[^9]

> Tekton is an open-source cloud native CICD (Continuous Integration and Continuous Delivery/Deployment) solution



## Tekton Pipeline With Workspace



## References

[^1]: [Cartographer - First Supply Chain Tutorial](https://cartographer.sh/docs/v0.7.0/tutorials/first-supply-chain/)
[^2]: [Cartographer - Types of Resources](https://cartographer.sh/docs/v0.7.0/tutorials/extending-a-supply-chain/#app-operator-steps)
[^3]: [Cartographer - Resource Definitions](https://cartographer.sh/docs/v0.7.0/reference/template/)
[^4]: [Cartographer - Parameters Tutorial](https://cartographer.sh/docs/v0.7.0/tutorials/using-params/)
[^5]: [Cartographer - Extend A Supply Chain Tutorial](https://cartographer.sh/docs/v0.7.0/tutorials/extending-a-supply-chain/)
[^6]: [Cartographer - ClusterImageTemplate resource spec](https://cartographer.sh/docs/v0.7.0/reference/template/#clusterimagetemplate)
[^7]: [KPack - Kubernetes solution for running Cloud Native Buildpacks](https://buildpacks.io/docs/tools/kpack/)
[^8]: [Cloud Native Build Packs](https://buildpacks.io/)
[^9]: [Tekton - Open-source cloud native CICD](https://tekton.dev/docs/getting-started/)
[^10]: []()
[^11]: []()
[^12]: []()
