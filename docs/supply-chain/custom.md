---
title: Create Supply Chain
description: Create a custom TAP Supply Chain
author: joostvdg
tags: [tap, kubernetes, cartographer, tekton, supplychain]
---

This exercise teaches you to create a custom Cartographer Supply Chain.

We do so by introducing the Cartographer Supply Chain resources, then the Tekton resources, and then combining the two.

## Goals & Outcomes

* [ ] I have a basic understanding of Cartographer Supply Chain
* [ ] I have a basic understanding of Tekton
* [ ] I've written and implemented my own Tekton Task
* [ ] I've written and implemented my own Tekton Pipeline
* [ ] I've written and implemented my own Custom Supply Chain on a Cluster
* [ ] I've written and implemented my own Custom Supply Chain on a Cluster, using Tekton

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
* Checkout this repository, to avoid having to create the files yourself
  * The repository is in [GitHub.com/joostvdg/tap-workshops](https://github.com/joostvdg/tap-workshops)
  ```sh
  git clone https://github.com/joostvdg/tap-workshops.git
  ```

!!! Warning "Use Default Namespace"
    This exercise is standalone and we recommend cleaning up the resources after you're done.

    So we use the `default` Namespace for all resources here, to reduce typing and copy-pasting.

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
kubectl apply -f resources/cartographer/rbac.yaml
kubectl apply -f resources/cartographer/app-deploy-01/01-cluster-template.yaml
kubectl apply -f resources/cartographer/app-deploy-01/02-supply-chain.yaml
kuebctl apply -f resources/cartographer/app-deploy-01/03-workload.yaml
```

Once you have applied the resources, the workload should be valid:

```sh
kubectl get workload
```

And if you have the `kubectl tree` plugin, you can see the related resources:

```sh
kubectl tree workload hello
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
kubectl port-forward deployment/hello-deployment 8080:8080
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

Technically, we do not have to update the RBAC configuration, but its included for completeness.

```sh
kubectl apply -f resources/cartographer/rbac.yaml
kubectl apply -f resources/cartographer/app-deploy-02/01-cluster-template.yaml
kubectl apply -f resources/cartographer/app-deploy-02/02-supply-chain.yaml
kubectl apply -f resources/cartographer/app-deploy-02/03-workload.yaml
```

Once you have applied the resources, the workload should be valid:

```sh
kubectl get workload
```

To test the application, you can use `kubectl port-forward`:

```sh
kubectl port-forward deployment/hello-02-deployment 8080:8080
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
        namespace: default

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

We can now apply all the Resources to the Cluster:

```sh
kubectl apply -f resources/cartographer/app-deploy-03/00-rbac.yaml
kubectl apply -f resources/cartographer/app-deploy-03/01-cluster-template.yaml
kubectl apply -f resources/cartographer/app-deploy-03/02-supply-chain.yaml
kubectl apply -f resources/cartographer/app-deploy-03/03-workload.yaml
```

!!! Info

    The `01-cluster-template.yaml` file contains both the ClusterTemplate and the ClusterImageTemplate.

Once you have applied the resources, the workload should be valid:

```sh
kubectl get workload
```

Because we're now doing a build, you might want to follow allong with the logs:

```sh
tanzu apps workload tail source-code-01 --timestamp --since 1h
```

To test the application, you can use `kubectl port-forward`:

```sh
kubectl port-forward deployment/source-code-01-deployment 8080:8080
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

We'll examine the core resources of Tekton and how to use them.
So that later we can include them in our Cartographer Supply Chain.

### Core Resources

For those new to Tekton, here is a brief explanation of the core resources:

* **Step**: definition of a command to execute in a container image, part of the Task definition
* **Task**: template for a series of Steps with Workspaces, Outputs and (input) Parameters, more on those below
* **TaskRun**: a one-time runtime instantiation of a Task with its Workspaces and Parameters defined
* **Pipeline**: a template for a series of Tasks, with Workspaces and Parameters
* **PipelineRun**: a one-time runtime instantiation of a Pipeline with its Workspaces and Parameters defined
* **Workspace**: storage definition, essentially Kubernetes Volume definitions, conceptually in Task and Pipeline, specified in TaskRun and PipelineRun
* **Results**: Tasks can have outputs, named Results, which is a way of exporting information from a Task into the Pipeline, so it can be used by other Tasks
* **Params**: input parameters for a Task

### Task & TaskRun

A Tekton Task is a Kubernetes CR that contains one or more Steps, and lot of other [optional configuration](https://tekton.dev/docs/pipelines/tasks/)[^10].

We will ignore most the optional configuration for now, and focus on the Steps.

A Step has a `name`, `image`, and then either `args` and/or a `command` or a `script`, depending on the Image used.

Below is an example of a Task using an Image to execute a shell command:

```yaml title="resources/tekton/task/01-task-hello.yaml"
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello
spec:
  steps:
    - name: echo
      image: harbor.services.h2o-2-9349.h2o.vmware.com/library/alpine@sha256:c0669ef34cdc14332c0f1ab0c2c01acb91d96014b172f1a76f3a39e63d1f0bda
      script: |
        #!/bin/sh
        echo "Hello World"
```

You can add this to your cluster as follows:

```sh
kubectl apply -f resources/tekton/task/01-task-hello.yaml
```

You can verify it exists, by running:

```sh
kubectl get task
```

Which returns something like this:

```sh
NAME            AGE
hello           24h
```

You might wonder, now what?

Nothing, a Task is a template, it doesn't do anything by itself.

For that we need either a PipelineRun (when the Task is part of a Pipeline) or a **TaskRun**.

A TaskRun refers to a Task, satisfies its requirements and then instantiates a version of that Task.

In our case, a TaskRun looks like this:

```yaml title="resources/tekton/task/02-task-run.yaml"
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: hello-task-run
spec:
  taskRef:
    name: hello
```

We can add this to the cluster as follows:

```sh
kubectl apply -f resources/tekton/task/02-task-run.yaml
```

And then verify its status:

```sh
kubectl get taskrun
```

Which initially returns this:

```sh
NAME              SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
hello-task-run    Unknown     Pending     5s
```

And then this:

```sh
NAME              SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
hello-task-run    True        Succeeded   53s         47s
```

Because our TaskRun has a fixed name, we can loot at the Pod and request the logs of the container.
The container is named after the step, so our step `echo` becomes `step-echo`:

```sh
kubectl logs hello-task-run-pod -c step-echo
```

Which should return the following:

```sh
Hello World
```

Feel free to clean them up:

```sh
kubectl delete -f resources/tekton/task/02-task-run.yaml
kubectl delete -f resources/tekton/task/01-task-hello.yaml
```

Let us look at Pipeline and PipelineRun next.

### Pipeline & PipelineRun

Having a single Task with a fixed named TaskRun is a bit limiting.

If you need to support more flows, workspaces, some logic, or want to re-use existing Tasks you need a Pipeline.

A [Tekton Pipeline](https://tekton.dev/docs/pipelines/pipelines/)[^11] is a collection of Tasks with inputs, outputs, workspaces and built-in logic for CI/CD workflows.

We won't go into too much detail, but feel free to explore the options a Pipeline offers[^11].

For now, we'll stick the ability to combine a series of Tasks and supply them with their requirements (e.g., input parameters).

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: hello-goodbye
spec:
  params: []
  tasks: []
```

That's our Pipeline skeleton, let us add some tasks to it:

```yaml
spec:
  tasks:
    - name: hello
      taskRef:
        name: hello
    - name: goodbye
      runAfter:
        - hello
      taskRef:
        name: goodbye
      params:
      - name: username
        value: $(params.username)
```

As you can see, we can re-use our `hello` Task.
And there's is a new Task, called `goodbye`.

There are two new things here:

* `runAfter`: Tasks might rely on outputs (Results) from other tasks, or have a logical sequential order, so we can specify Task B runs after Task A completed (successfully)
* `params`: our second Task, `goodbye`, requires an input parameter named `username`

As you can see in the `params` section of the `goodbye` Task, we supply it a value from the Pipeline `params` object.
Let's add that to our Pipeline:

```yaml
spec:
  params:
  - name: username
    type: string
```

Our Pipeline now looks like this:

```yaml title="resources/tekton/pipeline/03-pipeline.yaml"
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: hello-goodbye
spec:
  params:
  - name: username
    type: string
  tasks:
    - name: hello
      taskRef:
        name: hello
    - name: goodbye
      runAfter:
        - hello
      taskRef:
        name: goodbye
      params:
      - name: username
        value: $(params.username)
```

We can add our two Tasks and the Pipeline to the cluster, and Tekton verifies all Tasks are accounted for.

```sh
kubectl apply -f resources/tekton/pipeline/01-task-hello.yaml
kubectl apply -f resources/tekton/pipeline/02-task-goodbye.yaml
kubectl apply -f resources/tekton/pipeline/03-pipeline.yaml
```

And verify the Pipeline is healthy:

```sh
kubectl get pipeline
```

Which should result in:

```sh
NAME                 AGE
hello-goodbye        24h
```

Now that we have a Pipeline, we have to ***Run*** it.
It is probably not a surpise we do this via a **PipelineRun**.

Ignoring all the other bells and whistles Tekton offers, instantiating a Pipeline is straightforward.
We create a **PipelineRun** manifest, reference the **Pipeline** and supply it its requirements (e.g., `params`).

```yaml title="resources/tekton/pipeline/04-pipeline-run-static.yaml"
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: hello-goodbye-run
spec:
  pipelineRef:
    name: hello-goodbye
  params:
  - name: username
    value: "Tekton"
```

As you can see, we refer to our Pipeline via `pipelineRef.name`.
And we supply the input parameters via `params`.

Let's apply this to the cluster, and initiate our first Pipeline ***Run***.

```sh
kubectl apply \
  -f resources/tekton/pipeline/04-pipeline-run-static.yaml
```

And verify the state:

```sh
kubectl get pipelinerun
```

Which should yield something like this at first:

```sh
NAME                           SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
hello-goodbye-run              Unknown     Running     6s
```

And then:

```sh
NAME                           SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
hello-goodbye-run              True        Succeeded   64s         45s
```

And if we look for the TaskRuns:

```sh
kubectl get taskrun
```

We should see the following

```sh
NAME                           SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
hello-goodbye-run-goodbye      True        Succeeded   76s         63s
hello-goodbye-run-hello        True        Succeeded   82s         76s
```

There is a downside to running Pipelines like this, we can not have more than one Run.

The simple solution to this, is the replace the `metadata.name` property by `metadata.generateName`.
Where the convention is to end in a `-`, so a generated hash is included.

```yaml title="resources/tekton/pipeline/05-pipeline-run-dynamic.yaml"
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: hello-goodbye-run-
spec:
  pipelineRef:
    name: hello-goodbye
  params:
  - name: username
    value: "Tekton"
```

We cannot apply this resource to the cluster, we have to use `kubectl create` instead (because of the `generatedName`):

```sh
kubectl create -f  resources/tekton/pipeline/05-pipeline-run-dynamic.yaml
```

If we now look at the PipelineRun and TaskRuns:

```sh
kubectl get taskrun,pipelinerun
```

We get the following:

```sh
NAME                                                            SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
taskrun.tekton.dev/hello-goodbye-run-goodbye                    True        Succeeded   6m54s       6m41s
taskrun.tekton.dev/hello-goodbye-run-hello                      True        Succeeded   7m          6m54s
taskrun.tekton.dev/hello-goodbye-run-vhswg-goodbye              True        Succeeded   29s         21s
taskrun.tekton.dev/hello-goodbye-run-vhswg-hello                True        Succeeded   36s         29s

NAME                                                  SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
pipelinerun.tekton.dev/hello-goodbye-run              True        Succeeded   7m          6m41s
pipelinerun.tekton.dev/hello-goodbye-run-vhswg        True        Succeeded   36s         21s
```

The Run objects now include a hash in their name, allowing more than one to exists at once.

## Tekton Pipeline With Workspace

One common thing CI/CD Pipelines share, is the need to share data between Tasks.
For example, you might have a Git Clone task to collect your source code, and then re-use that in the rest of the pipeline.

Before going into how we work with Workspaces, let's highlight another concept.
Tasks are re-usable templates, as such, there is a large collection of them maintained by the community.
This is called the [Tekton Catalog](https://github.com/tektoncd/catalog/tree/main/task)[^12], in which you'll find common tasks such a GitClone, building an Image with Kaniko and so on.

### Setting Up The Tasks

We will re-use the existing [GitClone Task](https://github.com/tektoncd/catalog/tree/main/task/git-clone)[^13], although with one minor modification.

Which is included in the resources:

```sh
kubectl apply \
  -f resources/tekton/pipeline-w-workspace/02-task-git-clone-0.10.yaml
```

This Task let's us Clone a Git repository, with practically all common Git clone configuration options.

It defines a list of Workspaces:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: git-clone
spec:
  workspaces:
    - name: output
      description: The git repo will be cloned onto the volume backing this Workspace.
    - name: ssh-directory
      optional: true
      description: |
        A .ssh directory with private key, known_hosts, config, etc. Copied to
        the user's home before git commands are executed. Used to authenticate
        with the git remote when performing the clone. Binding a Secret to this
        Workspace is strongly recommended over other volume types.
    - name: basic-auth
      optional: true
      description: |
        A Workspace containing a .gitconfig and .git-credentials file. These
        will be copied to the user's home before any git commands are run. Any
        other files in this Workspace are ignored. It is strongly recommended
        to use ssh-directory over basic-auth whenever possible and to bind a
        Secret to this Workspace over other volume types.
    - name: ssl-ca-directory
      optional: true
      description: |
        A workspace containing CA certificates, this will be used by Git to
        verify the peer with when fetching or pushing over HTTPS.
```

Most of which have `optional: true`, except for `output`.
Which means we'll need to supply that in our **Pipeline** later.

It also has many paramaters, of which `url` and `revision` are required.
We'll need to supply those as well.

Next up is a Task that uses this Git checkout, to determine the next Git tag (e.g., application's release version).

```sh
kubectl apply -f resources/tekton/pipeline-w-workspace/01-task-git-next-tag.yaml
```

This task also required a Workspace:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: git-next-tag
spec:
  workspaces:
    - name: source
```

It requires one parameter, `base`, which is the SemVer base (e.g., `1.0.*`).

And it provided an Output:

```yaml
spec:
  results:
    - name: NEXT_TAG
      description: Next version for Git Tag.
```

We'll look at this result later.

### Creating the Pipeline with Workspace

So far, our two Tasks have a "shopping list" of required items:

* Workspace (`source` for git-next-tag, and `output` for git-clone)
* Parameters: (Git) `url`, (Git) `revision`, and (SemVer Tag) `base`

This is how our Pipeline looks like, satisfying those requirements:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: clone-git-next-tag
spec:
  params:
    - name: repo-url
      type: string
    - name: base
      type: string
    - name: gitrevision
  workspaces:
    - name: shared-data
```

Next, we add the Tasks and configure their requirements via these properties:

```yaml
spec:
  tasks:
    - name: fetch-source
      taskRef:
        name: git-clone
      workspaces:
        - name: output
          workspace: shared-data
      params:
        - name: url
          value: $(params.repo-url)
        - name: revision
          value: $(params.gitrevision)
    - name: git-next-tag
      runAfter: ["fetch-source"]
      taskRef:
        name: git-next-tag
      workspaces:
        - name: source
          workspace: shared-data
      params:
        - name: base
          value: $(params.base)
```

As you might have spotted, we re-use the Workspace.
Supplying the same single Workspace we defined in the Pipeline to both Tasks!

This is intended, else we can't re-use our Git clone.

Apply the Pipeline to the cluster.

```sh
kubectl apply \
  -f resources/tekton/pipeline-w-workspace/03-pipeline.yaml
```

??? Example "Complete Pipeline Example"

    ```yaml title="resources/tekton/pipeline-w-workspace/03-pipeline.yaml"
    apiVersion: tekton.dev/v1beta1
    kind: Pipeline
    metadata:
      name: clone-git-next-tag
    spec:
      description: |
        This pipeline clones a git repo, then echoes the README file to the stdout.
      params:
        - name: repo-url
          type: string
          description: The git repo URL to clone from.
        - name: base
          description: version Base to query Git tags for (e.g., v2.1.*)
          type: string
        - name: gitrevision
          description: git revision to checkout
      workspaces:
        - name: shared-data
          description: |
            This workspace contains the cloned repo files, so they can be read by the
            next task.
      tasks:
        - name: fetch-source
          taskRef:
            name: git-clone
          workspaces:
            - name: output
              workspace: shared-data
          params:
            - name: url
              value: $(params.repo-url)
            - name: revision
              value: $(params.gitrevision)
        - name: git-next-tag
          runAfter: ["fetch-source"]
          taskRef:
            name: git-next-tag
          workspaces:
            - name: source
              workspace: shared-data
          params:
            - name: base
              value: $(params.base)
    ```

### Create the PipelineRun

To instantiate our Pipeline, we'll create a PipelineRun.

In this PipelineRun, we need to do the following:

* reference the Pipeline
* supply a Workspace
* supply the Parameters

To make it re-usable, we'll start with a `generateName` style:

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: clone-git-next-tag-run-
```

To ensure all containers can read the filesystem of the volume, we set a specific `fsGroup`:

```sh
spec:
  podTemplate:
    securityContext:
      fsGroup: 65532
```

And then we create a Workspace via a `volumeClaimTemplate`:

```yaml
spec:
  workspaces:
    - name: shared-data
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 50Mi
          volumeMode: Filesystem
```

Note, the name `shared-data` is the name specified in the Pipeline.
The Pipeline definition ensures each Task gets it supplied as however it named it.

And the parameters:

```yaml
spec:
  params:
    - name: repo-url
      value: https://github.com/joostvdg/go-demo.git
    - name: base
      value: "v2.1"
    - name: gitrevision
      value: main
```

??? Example "Full PipelineRun Example"

    ```yaml
    apiVersion: tekton.dev/v1beta1
    kind: PipelineRun
    metadata:
      generateName: clone-git-next-tag-run-
    spec:
      pipelineRef:
        name: clone-git-next-tag
      podTemplate:
        securityContext:
          fsGroup: 65532
      workspaces:
        - name: shared-data
          volumeClaimTemplate:
            spec:
              accessModes:
                - ReadWriteOnce
              resources:
                requests:
                  storage: 50Mi
              volumeMode: Filesystem
      params:
        - name: repo-url
          value: https://github.com/joostvdg/go-demo.git
        - name: base
          value: "v2.1"
        - name: gitrevision
          value: main
    ```

### Excercise 4

Just in case you haven't applied all files yet, here's the whole list again:

```sh
kubectl apply -f resources/tekton/pipeline-w-workspace/02-task-git-clone-0.10.yaml
kubectl apply -f resources/tekton/pipeline-w-workspace/01-task-git-next-tag.yaml
kubectl apply -f resources/tekton/pipeline-w-workspace/03-pipeline.yaml
```

Verify the Pipeline is valid, and then create the PipelineRun:

```sh
kubectl create \
  -f resources/tekton/pipeline-w-workspace/04-pipeline-run.yaml
```

Once created, you can verify the status:

```sh
kubectl get taskrun,pipelinerun
```

If you want the logs, you'll now have to find the appropriate Pod name, as its name is generated.

```sh
kubectl get pod
```

```sh
POD_NAME=
```

```sh
kubectl logs ${POD_NAME}
```

You can also use the (automatic) Labels to query them:

```sh
kubectl get taskrun -l tekton.dev/task=git-next-tag
```

And then you can find the output of the `Result` from the `git-next-tag` Task in its `status.taskResults` field:

```sh
kubectl get taskrun \
  -l tekton.dev/task=git-next-tag -ojson \
  | jq '.items | map(.status.taskResults)'
```

## Tekton In Supply Chain

Now that we know how to build a Cartographer Supply Chain and a Tekton Pipeline, let's combine the two!

As with the other steps, we're reusing an [existing tutorial](https://cartographer.sh/docs/v0.7.0/tutorials/lifecycle/)[^14].

The goal is to create a Supply Chain that validates source code, then builds an Image and then deploys that Image.

It is in the Source Code Validation step we'll use Tekton.

### Tekton Task Markdown Lint

The Markdown Lint Task is [available in the Tekton Catalog](https://github.com/tektoncd/catalog/tree/main/task/markdown-lint/0.1)[^15], and only needs a Workspace to do its work.

```yaml
workspaces:
  - name: shared-workspace
    description: A workspace that contains the fetched git repository.
```

The task doesn't contain anything new, so we'll go on to the Pipeline.

??? Example "Markdown Lint Task"

    ```yaml title="resources/cartographer/app-deploy-04/01-tekton-task-markdown-lint-0.1.yaml"
    apiVersion: tekton.dev/v1beta1
    kind: Task
    metadata:
      name: markdown-lint
      labels:
        app.kubernetes.io/version: "0.1"
      annotations:
        tekton.dev/pipelines.minVersion: "0.12.1"
        tekton.dev/categories: Code Quality
        tekton.dev/tags: linter
        tekton.dev/displayName: "Markdown linter"
        tekton.dev/platforms: "linux/amd64"
    spec:
      description: >-
        This task can be used to perform lint check on Markdown files
      workspaces:
        - name: shared-workspace
          description: A workspace that contains the fetched git repository.
      params:
        - name: args
          type: array
          description: extra args needs to append
          default: ["--help"]
      steps:
        - name: lint-markdown-files
          image: harbor.services.h2o-2-9349.h2o.vmware.com/other/markdownlint@sha256:399a199c92f89f42cf3a0a1159bd86ca5cdc293fcfd39f87c0669ddee9767724 #tag: 0.11.0
          workingDir: $(workspaces.shared-workspace.path)
          command:
            - mdl
          args:
            - $(params.args)
    ```

!!! Info
    You might be wondering: "If they are all available in the Catalog, why not use them from there?".

    Which is a valid question to ask. 
    The answer: the images used are relocated and the Task in the resources uses that relocated Image.

### Tekton Pipeline

The Pipeline uses two tasks. The Markdown Lint[^15] Task and the [Git Clone](https://github.com/tektoncd/catalog/tree/main/task/git-clone/0.3)[^16] Task.
The tutorial uses Git Clone version `0.3`, so we've included that in the Resources.

As the Markdown Lint task requires the Source Code, it has to run after the Git Clone (i.e., `fetch-repository`) Task.

```yaml
spec:
  tasks:
    - name: fetch-repository
      taskRef:
        name: git-clone
    - name: md-lint-run
      taskRef:
        name: markdown-lint
      runAfter:
        - fetch-repository
```

As with our previous Pipeline, we need to supply the Git Repository URL and the (Git) Revision.
So we specify appropriate `params` at `spec.params` for the Pipeline.

We also have to specify a Workspace, which is then given to both tasks.

```yaml
spec:
  params:
    - name: repository
      type: string
    - name: revision
      type: string
  workspaces:
    - name: shared-workspace
```

We then make sure the Workspace and the Parameters are applied to the Tasks.

See the complete example below.

??? Example "Complete Pipeline Example"

    ```yaml title="resources/cartographer/app-deploy-04/02-tekton-pipeline-markdown-lint.yaml"
    apiVersion: tekton.dev/v1beta1
    kind: Pipeline
    metadata:
      name: linter-pipeline
    spec:
      params:
        - name: repository
          type: string
        - name: revision
          type: string
      workspaces:
        - name: shared-workspace
      tasks:
        - name: fetch-repository
          taskRef:
            name: git-clone
          workspaces:
            - name: output
              workspace: shared-workspace
          params:
            - name: url
              value: $(params.repository)
            - name: revision
              value: $(params.revision)
            - name: subdirectory
              value: ""
            - name: deleteExisting
              value: "true"
        - name: md-lint-run #lint markdown
          taskRef:
            name: markdown-lint
          runAfter:
            - fetch-repository
          workspaces:
            - name: shared-workspace
              workspace: shared-workspace
          params:
            - name: args
              value: ["."]
    ```

### Tekton PipelineRun in Cartographer

For Cartographer to instantiate a Tekton Pipeline, it needs to produce a PipelineRun.

As you probably expect by now, we use one of the Template types of Cartographer.

Perhaps a bit counterintuitive, but as the Pipeline does a Git checkout we use the `ClusterSourceTemplate`.

```yaml
apiVersion: carto.run/v1alpha1
kind: ClusterSourceTemplate
metadata:
  name: source-linter
spec:
  template: {}
```

The Supply Chain runs more than once.
In order for that to work well with the Tekton resources we need to do the following:

* use the `metadata.generateName` way of naming the Tekton resources
* set the **ClusterSourceTemplate**'s `lifecycle` property to `tekton`

```yaml
spec:
  lifecycle: tekton
```

In the `spec.template`, we write the **PipelineRun**.

```yaml
spec:
  template:
    apiVersion: tekton.dev/v1beta1
    kind: PipelineRun
    metadata:
      generateName: linter-pipeline-run-
    spec:
      pipelineRef:
        name: linter-pipeline
```

Our Pipeline requires a Workspace, which we'll define in the way we've done before:

```yaml
spec:
  template:
    apiVersion: tekton.dev/v1beta1
    kind: PipelineRun
    spec:
      workspaces:
        - name: shared-workspace
          volumeClaimTemplate:
            spec:
              accessModes:
                - ReadWriteOnce
              resources:
                requests:
                  storage: 256Mi
```

The Pipeline also requires two parameters.
The Git Repository URL (`repository`) and Revision (`revision`).

```yaml
spec:
  template:
    apiVersion: tekton.dev/v1beta1
    kind: PipelineRun
    spec:
      params:
        - name: repository
          value: $(workload.spec.source.git.url)$
        - name: revision
          value: $(workload.spec.source.git.ref.branch)$
```

Here there is a change.

We specify the Parameters so they are copied from the **Workload**.

The **ClusterSourceTemplate** also expects to ouput the Source URL and Revision.
Which also allows us to show how you retrieve values from the Tekton PipelineRun.

```yaml
spec:
  urlPath: .status.pipelineSpec.tasks[0].params[0].value
  revisionPath: .status.pipelineSpec.tasks[0].params[1].value
```

That is our ClusterSourceTemplate.

??? Example "Complete ClusterSourceTemplate"

    ```yaml title="resources/cartographer/app-deploy-04/03-cluster-source-template.yaml"
    apiVersion: carto.run/v1alpha1
    kind: ClusterSourceTemplate
    metadata:
      name: source-linter
    spec:
      template:
        apiVersion: tekton.dev/v1beta1
        kind: PipelineRun
        metadata:
          generateName: linter-pipeline-run-
        spec:
          pipelineRef:
            name: linter-pipeline
          params:
            - name: repository
              value: $(workload.spec.source.git.url)$
            - name: revision
              value: $(workload.spec.source.git.ref.branch)$
          workspaces:
            - name: shared-workspace
              volumeClaimTemplate:
                spec:
                  accessModes:
                    - ReadWriteOnce
                  resources:
                    requests:
                      storage: 256Mi
      urlPath: .status.pipelineSpec.tasks[0].params[0].value
      revisionPath: .status.pipelineSpec.tasks[0].params[1].value
      lifecycle: tekton
    ```

### Image Building Templates

As we're extending the previous Supply Chain, we're reusing the templates from before.
Namely, the **ClusterTemplate** and the **ClusterImageTemplate**.

As you probably expect from me, I've renamed them, so they can be used next to the other ones.

```yaml
apiVersion: carto.run/v1alpha1
kind: ClusterTemplate
metadata:
  name: app-deploy-from-sc-image-04
```

And:

```yaml
apiVersion: carto.run/v1alpha1
kind: ClusterImageTemplate
metadata:
  name: image-builder-04
```

For the rest they are the same.

??? Example "Complete Templates"

    ```yaml title="resources/cartographer/app-deploy-04/04-cluster-template.yaml"
    apiVersion: carto.run/v1alpha1
    kind: ClusterTemplate
    metadata:
      name: app-deploy-from-sc-image-04
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
                  image: $(images.built-image.image)$
                  env:
                    - name: $(params.env_key)$
                      value: $(params.env_value)$
      params:
        - name: env_key
          default: "FOO"
        - name: env_value
          default: "BAR"

    ---
    apiVersion: carto.run/v1alpha1
    kind: ClusterImageTemplate
    metadata:
      name: image-builder-04
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

### Update Supply Chain

We're almost there, let's update (and rename) the Supply Chain.

```yaml
apiVersion: carto.run/v1alpha1
kind: ClusterSupplyChain
metadata:
  name: source-code-supply-chain-04
spec:
  selector:
    workload-type: source-code-04
```

We now have three resources:

* **ClusterSourceTemplate**: the Tekton PipelineRun
* **ClusterImageTemplate**: our KPack Image build
* **ClusterTemplate**: our deployment

```yaml
  resources:
    - name: lint-source
      templateRef:
        kind: ClusterSourceTemplate
        name: source-linter
    - name: build-image
      templateRef:
        kind: ClusterImageTemplate
        name: image-builder-04
      sources:
        - resource: lint-source
          name: source
    - name: deploy
      templateRef:
        kind: ClusterTemplate
        name: app-deploy-from-sc-image-04
      images:
        - resource: build-image
          name: built-image
```

We also need to update the permission for our SA, wich we'll do next.
The SA config itself has not changed.

```yaml
spec:
  serviceAccountRef:
    name: cartographer-from-source-sa
```

??? Example "Complete Supply Chain"

    ```yaml title="resources/cartographer/app-deploy-04/05-supply-chain.yaml"
    apiVersion: carto.run/v1alpha1
    kind: ClusterSupplyChain
    metadata:
      name: source-code-supply-chain-04
    spec:
      selector:
        workload-type: source-code-04

      resources:
        - name: lint-source
          templateRef:
            kind: ClusterSourceTemplate
            name: source-linter
        - name: build-image
          templateRef:
            kind: ClusterImageTemplate
            name: image-builder-04
          sources:
            - resource: lint-source
              name: source
        - name: deploy
          templateRef:
            kind: ClusterTemplate
            name: app-deploy-from-sc-image-04
          images:
            - resource: build-image
              name: built-image

      serviceAccountRef:
        name: cartographer-from-source-sa
    ```


### Update Workload

The only change to the Workload is the name.

```yaml
apiVersion: carto.run/v1alpha1
kind: Workload
metadata:
  name: hello-04
  labels:
    workload-type: source-code-04
```

??? Example "Updated Workload"

    ```yaml title="resources/cartographer/app-deploy-04/06-workload.yaml"
    apiVersion: carto.run/v1alpha1
    kind: Workload
    metadata:
      name: hello-04
      labels:
        workload-type: source-code-04
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

### Exercise 5

Just in case you haven't applied all files yet, here's the whole list again:

```sh
kubectl apply -f resources/cartographer/app-deploy-04/00-rbac.yaml
kubectl apply -f resources/cartographer/app-deploy-04/01-tekton-task-git-clone-0.3.yaml
kubectl apply -f resources/cartographer/app-deploy-04/01-tekton-task-markdown-lint-0.1.yaml
kubectl apply -f resources/cartographer/app-deploy-04/02-tekton-pipeline-markdown-lint.yaml
kubectl apply -f resources/cartographer/app-deploy-04/03-cluster-source-template.yaml
kubectl apply -f resources/cartographer/app-deploy-04/04-cluster-template.yaml
kubectl apply -f resources/cartographer/app-deploy-04/05-supply-chain.yaml
kubectl apply -f resources/cartographer/app-deploy-04/06-workload.yaml
```

Once created, you can verify the status:


```sh
kubectl get workload
```

As there's a few things going on, we'll see the Workload have several different statusses.

For example:

```sh
NAME             SOURCE                               SUPPLYCHAIN                   READY   REASON                                                AGE
hello-04         https://github.com/joostvdg/go-demo  source-code-supply-chain-04   False   SetOfImmutableStampedObjectsIncludesNoHealthyObject   11s
```

The `Unknown` status with Reason `MissingValueAtPath` is quite common, it usually means a Resource B expects a value from another Resource A.
While the Resource A is not finished with its work, Resource B cannot read the value and thus reports `MissingValueAtPath`.

```sh
NAME             SOURCE                               SUPPLYCHAIN                   READY     REASON                        AGE
hello-04         https://github.com/joostvdg/go-demo  source-code-supply-chain-04   Unknown   MissingValueAtPath            68s
```

Eventually the Workload will be finished successfully.

```sh
NAME             SOURCE                               SUPPLYCHAIN                   READY     REASON                        AGE
hello-04         https://github.com/joostvdg/go-demo  source-code-supply-chain-04   True      Ready                         6m2s
```

We can then also take a look at the Tekton resources:

```sh
kubectl get taskrun,pipelinerun
```

```sh
NAME                                                            SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
taskrun.tekton.dev/linter-pipeline-run-xn5vg-fetch-repository   True        Succeeded   6m55s       6m40s
taskrun.tekton.dev/linter-pipeline-run-xn5vg-md-lint-run        True        Succeeded   6m40s       6m34s

NAME                                                  SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
pipelinerun.tekton.dev/linter-pipeline-run-xn5vg      True        Succeeded   6m55s       6m34s
```

For more details, see the commands from the previous Exercises.

!!! Info

    In case you are wondering how this hierarchy now looks:

    ```sh
    * ClusterSupplyChain
      * ClusterSourceTemplate
        * ClusterRunTemplate
          * PipelineRun
            * Task
              * TaskRun (Generated)
                * Pod (Generated)
                  * InitContainer -> Shell (Generated)
    ```

## OOTB Pipeline Appendix

!!! Warning "Source in TAP"
    In TAP, we don't have to clone our sources from Git, we can download them from FluxCD.

    The way TAP works, it that the trigger for a SupplyChain goes through FluxCD's GitRepository management.
    So below is a way of codifying that process into a Tekton Task:

    ```yaml
    apiVersion: tekton.dev/v1beta1
    kind: Task
    metadata:
      name: fluxcd-repo-download
    spec:
      params:
        - name: source-url
          type: string
          description: |
            the source url to download the code from, 
            in the form of a FluxCD repository checkout .tar.gz
      workspaces:
        - name: output
          description: The git repo will be cloned onto the volume backing this Workspace.
      steps:
        - name: download-source
          image: public.ecr.aws/docker/library/gradle:jdk17-focal
          script: |
            #!/usr/bin/env sh
            cd $(workspaces.output.path)
            wget -qO- $(params.source-url) | tar xvz -m
    ```

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
[^10]: [Tekton - Task details](https://tekton.dev/docs/pipelines/tasks/)
[^11]: [Tekton - Pipeline details](https://tekton.dev/docs/pipelines/pipelines/)
[^12]: [Tekton - Task Catalog](https://github.com/tektoncd/catalog/tree/main/task)
[^13]: [Tekton Catalog - Git Clone 0.3](https://github.com/tektoncd/catalog/tree/main/task/git-clone)
[^14]: [Cartographer - Supply Chain with Tekton Pipeline](https://cartographer.sh/docs/v0.7.0/tutorials/lifecycle/)
[^15]: [Tekton Catalog - Markdown Lint 0.1](https://github.com/tektoncd/catalog/tree/main/task/markdown-lint/0.1)
[^16]: [Tekton Catalog - Git Clone 0.3](https://github.com/tektoncd/catalog/tree/main/task/git-clone/0.3)
[^17]: [Cartographer - Lifecycle Tutorial](https://cartographer.sh/docs/v0.7.0/tutorials/lifecycle/)
