---
title: TAP App with External Service
description: TAP Demo App - Spring Boot with Database Using External Service
author: joostvdg
tags: [tap, kubernetes, spring, java, spring-boot, mysql, crossplane]
---

In this workshop, we onboard an Application into TAP that depends on an external database.

For this to work well, we need to support the following:
* Ability to have a database during testing
* Ability to inject connection details to external database

For the first, we need to customize our Supply Chain.
The latter is supported in TAP via the Services Toolkit component, leveraging Crossplane and Service Binding Spec to do so.

## Checks

- [ ] I am able to register API Documentation for a Workload in TAP GUI
- [ ] I am able to stand up shared services and claim resources on a TAP Run Cluster
- [ ] I can confgure API Auto Registration within a Supply Chain for a Workload
- [ ] I can configure API Scoring and Validation within TAP GUI
- [ ] I can configure API Portal with TAP
- [ ] I can install Shared Services on a TAP Cluster (Services Toolkit)
- [ ] I can claim a shared service on a TAP Cluster (Service Bindings)
- [ ] I can use a postgres resource claim with sample Spring Boot Application Pet Clinic Accelerator
- [ ] Expose Accerators Endpoint for use in Tanzu Accelerator Plugins

## Steps

* update supply chain
    * create new Tekton Tasks
    * create new Tekton Pipeline that uses these Tasks
    * create copy from OOTB Supply Chain that uses Tekton Pipeline
* fork existing application
* create Workload and see tests succeed
* deployment fails -> we need a database
* create database via Bitnami Services
    * static vs. dynamic
    * we do static for now
    * verify database exists
* use services toolkit to bind
    * create claim
    * update workload to use claim

## Update Supply Chain

As stated in the preamble, we supply our application with a database during testing.

To be precise, we will provide it with a means to do so itself, via Testcontainers.

To do so, we need to give it a Docker host to talk to, to spin up arbitrary containers.

!!! Danger "Not Production Ready"

    There are better ways of giving the application a Docker host to talk to, so that it can leverage Test Containers.

    You can run the build on specific VMs or Micro VMs with Docker, or expose the Docker daemon on those VMs.

    To solution below uses Docker In Docker, or DinD.
    While supported by Docker, it is not a best practice.

To make this Supply Chain leverage Tekton in a good way, we will also introduce a (Tekton) Workspace to share code between Tasks.

!!! Important "Create the files below"

    The Files you see below need to be applied to the cluster.

    So please create them on your machine, and follow the instructions after.

### Checkout Task

A proper Pipeline starts with checking out the code.
Below is a Task that uses the information provided by the OOTB Supply Chain to copy the code from FluxCD.

It stores this on a Workspace that is shared with any next Task in the (Tekton) Pipeline.

```yaml title="task-fluxcd-repo-download.yaml"
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: fluxcd-repo-download
spec:
  params:
  - description: "the source url to download the code from, \nin the form of a FluxCD
      repository checkout .tar.gz\n"
    name: source-url
    type: string
  steps:
  - image: harbor.services.h2o-2-9349.h2o.vmware.com/dockerhub/library/gradle:jdk17-focal
    name: download-source
    resources: {}
    script: |
      #!/usr/bin/env sh
      cd $(workspaces.output.path)
      wget -qO- $(params.source-url) | tar xvz -m
  workspaces:
  - description: The git repo will be cloned onto the volume backing this Workspace.
    name: output
```

### Maven TestContainers Task

Below is a Task that supports a Maven build that requires a Docker daemon.

One such type of build that requires a Docker daemon, is the use of TestContainers.

TestContainers provides a build with ad-hoc creation of a Docker container required for tests, and then handles the cleanup as well.

```yaml title="task-maven-test-containers.yaml"
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: maven-test-containers
spec:
  sidecars:
  - image: harbor.services.h2o-2-9349.h2o.vmware.com/library/docker:20.10-dind
    name: docker
    resources: {}
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /var/lib/docker
      name: dind-storage
    - mountPath: /var/run/
      name: dind-socket
  steps:
  - image: harbor.services.h2o-2-9349.h2o.vmware.com/library/eclipse-temurin:17.0.3_7-jdk-alpine
    name: read
    resources: {}
    script: ./mvnw test
    volumeMounts:
    - mountPath: /var/run/
      name: dind-socket
    workingDir: $(workspaces.output.path)
  volumes:
  - emptyDir: {}
    name: dind-storage
  - emptyDir: {}
    name: dind-socket
  workspaces:
  - description: The workspace consisting of maven project.
    name: output
```

### Tekton Pipeline FluxCD Maven Test

Here is the Pipeline that combines the two Tasks.

```yaml title="pipeline-fluxcd-maven-test.yaml"
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  labels:
    apps.tanzu.vmware.com/pipeline: test
  name: fluxcd-maven-test
spec:
  params:
  - name: source-url
    type: string
  - name: source-revision
    type: string
  tasks:
  - name: fetch-repository
    params:
    - name: source-url
      value: $(params.source-url)
    taskRef:
      kind: Task
      name: fluxcd-repo-download
    workspaces:
    - name: output
      workspace: shared-workspace
  - name: maven-run
    params:
    - name: GOALS
      value:
      - clean
      - verify
    runAfter:
    - fetch-repository
    taskRef:
      kind: Task
      name: maven-test-containers
    workspaces:
    - name: output
      workspace: shared-workspace
  workspaces:
  - name: shared-workspace
  - name: maven-settings
```

### ClusterTemplate For New Pipeline

A Tekton Pipeline needs to be triggered via a **PipelineRun**.

In our Supply Chain, we use a **ClusterRunTemplate** to generate a **PipelineRun** with the appropriate parameters.

```yaml title="tekton-source-pipelinerun-workspace.yaml"
apiVersion: carto.run/v1alpha1
kind: ClusterRunTemplate
metadata:
  name: tekton-source-pipelinerun-workspace
spec:
  outputs:
    revision: spec.params[?(@.name=="source-revision")].value
    url: spec.params[?(@.name=="source-url")].value
  template:
    apiVersion: tekton.dev/v1beta1
    kind: PipelineRun
    metadata:
      generateName: $(runnable.metadata.name)$-
      labels: $(runnable.metadata.labels)$
    spec:
      params: $(runnable.spec.inputs.tekton-params)$
      pipelineRef:
        name: $(selected.metadata.name)$
      podTemplate:
        securityContext:
          fsGroup: 65532
      workspaces:
      - name: maven-settings
        emptyDir: {}
      - name: shared-workspace
        volumeClaimTemplate:
          spec:
            accessModes:
            - ReadWriteOnce
            resources:
              requests:
                storage: 500Mi
            volumeMode: Filesystem
```

### Add New Components to the Cluster

We can now add the resources to the cluster.

The Tekton resources are namespaced, the Cartographer **ClusterRunTemplate**, as the name implies, is not.

```sh
kubectl apply -f task-fluxcd-repo-download.yaml \
    -n ${TAP_DEVELOPER_NAMESPACE}
kubectl apply -f task-maven-test-containers.yaml \
    -n ${TAP_DEVELOPER_NAMESPACE}
kubectl apply -f pipeline-fluxcd-maven-test.yaml \
        -n ${TAP_DEVELOPER_NAMESPACE}
kubectl apply -f tekton-source-pipelinerun-workspace.yaml 
```

!!! Danger "There Can Be Only One"
    Make sure there is one **Pipeline** with the label `apps.tanzu.vmware.com/pipeline=test`.

    Verify this with this command:

    ```sh
    get pipeline -l apps.tanzu.vmware.com/pipeline=test \
      -n $TAP_DEVELOPER_NAMESPACE
    ```

    If there is more than one, e.g., you still have `developer-defined-tekton-pipeline`, as below:

    ```sh
    NAME                                AGE
    developer-defined-tekton-pipeline   14h
    fluxcd-maven-test                   87m
    ```

    Please remove that one:

    ```sh
    kubectl delete pipeline developer-defined-tekton-pipeline \
      -n $TAP_DEVELOPER_NAMESPACE
    ```

### Copy Existing Supply Chain Components

And then customize the **ClusterSourceTemplate** name `testing-pipeline` to use our new **ClusterRunTemplate** instead.

```sh
kubectl get ClusterSourceTemplate testing-pipeline \
  -o yaml > testing-pipeline-workspace.yaml
```

We first strip away all the things from the `metadata` that we don't need, and rename it in a single `yq` command.

```sh
yq e -i '.metadata = {"name": "testing-pipeline-workspace"}' testing-pipeline-workspace.yaml
```

The next step is a bit more difficult, so we need some `sed` magic.
There is an inlined YTT template, in which we need to rename the Tekton Pipeline we refer to the one we just created.

```sh
sed -i -e "s/tekton-source-pipelinerun/tekton-source-pipelinerun-workspace/g" testing-pipeline-workspace.yaml
```

??? Example

    ```sh
    cat testing-pipeline-workspace.yaml
    ```

    ```yaml title="testing-pipeline-workspace.yaml"
    apiVersion: carto.run/v1alpha1
    kind: ClusterSourceTemplate
    metadata:
      name: testing-pipeline-workspace
    spec:
      healthRule:
        singleConditionType: Ready
      lifecycle: mutable
      params:
        - default:
            apps.tanzu.vmware.com/pipeline: test
          name: testing_pipeline_matching_labels
      revisionPath: .status.outputs.revision
      urlPath: .status.outputs.url
      ytt: "#@ load(\"@ytt:data\", \"data\")\n\n#@ def merge_labels(fixed_values):\n#@   labels = {}\n#@   if hasattr(data.values.workload.metadata, \"labels\"):\n#@     labels.update(data.values.workload.metadata.labels)\n#@   end\n#@   labels.update(fixed_values)\n#@   return labels\n#@ end\n\n#@ def merged_tekton_params():\n#@   params = []\n#@   if hasattr(data.values, \"params\") and hasattr(data.values.params, \"testing_pipeline_params\"):\n#@     for param in data.values.params[\"testing_pipeline_params\"]:\n#@       params.append({ \"name\": param, \"value\": data.values.params[\"testing_pipeline_params\"][param] })\n#@     end\n#@   end\n#@   params.append({ \"name\": \"source-url\", \"value\": data.values.source.url })\n#@   params.append({ \"name\": \"source-revision\", \"value\": data.values.source.revision })\n#@   return params\n#@ end\n---\napiVersion: carto.run/v1alpha1\nkind: Runnable\nmetadata:\n  name: #@ data.values.workload.metadata.name\n  labels: #@ merge_labels({ \"app.kubernetes.io/component\": \"test\" })\nspec:\n  #@ if/end hasattr(data.values.workload.spec, \"serviceAccountName\"):\n  serviceAccountName: #@ data.values.workload.spec.serviceAccountName\n\n  runTemplateRef:\n    name: tekton-source-pipelinerun-workspace\n    kind: ClusterRunTemplate\n\n  selector:\n    resource:\n      apiVersion: tekton.dev/v1beta1\n      kind: Pipeline\n\n    #@ not hasattr(data.values, \"testing_pipeline_matching_labels\") or fail(\"testing_pipeline_matching_labels param is required\")\n    matchingLabels: #@ data.values.params[\"testing_pipeline_matching_labels\"] or fail(\"testing_pipeline_matching_labels param cannot be empty\")\n\n  inputs: \n    tekton-params: #@ merged_tekton_params()\n"
    ```

Again, this is a Cluster resource, so no need to supply a namespace:

```sh
kubectl apply -f testing-pipeline-workspace.yaml
```

Next, let's create a new ClusterSupplyChain, wich uses the resources we just created.
This way, our existing Supply Chain and the Applications depending on it, continue to function.

First, retrieve the existing the **ClusterSupplyChain** as a start:

```sh
kubectl get ClusterSupplyChain source-test-scan-to-url \
  -o yaml > source-test-scan-to-url.yaml
```

We first strip away all the things from the `metadata` that we don't need, and rename it in a single `yq` command.

```sh
yq e -i '.metadata = {"name": "source-test-scan-to-url-workspace"}' source-test-scan-to-url.yaml
```

Clear the Status, just in case:

```sh
yq e -i '.status = {}' source-test-scan-to-url.yaml
```

Change to Source Template to the one we just created:

```sh
sed -i -e "s/testing-pipeline/testing-pipeline-workspace/g" source-test-scan-to-url.yaml
```

Set Selector to:

```yaml
selector:
  apps.tanzu.vmware.com/has-tests-needs-workspace: "true"
```

```sh
sed -i -e "s/has-tests/has-tests-needs-workspace/g" source-test-scan-to-url.yaml
```

Verify the SupplyChain looks valid:

```sh
cat source-test-scan-to-url.yaml
```

??? Example

    ```yaml
    apiVersion: carto.run/v1alpha1
    kind: ClusterSupplyChain
    metadata:
      name: source-test-scan-to-url-workspace
    spec:
      params:
        - name: maven_repository_url
          value: https://repo.maven.apache.org/maven2
        - default: main
          name: gitops_branch
        - default: supplychain
          name: gitops_user_name
        - default: supplychain
          name: gitops_user_email
        - default: supplychain@cluster.local
          name: gitops_commit_message
        - default: ""
          name: gitops_ssh_secret
      resources:
        - name: source-provider
          params:
            - default: default
              name: serviceAccount
            - default: go-git
              name: gitImplementation
          templateRef:
            kind: ClusterSourceTemplate
            name: source-template
        - name: source-tester
          sources:
            - name: source
              resource: source-provider
          templateRef:
            kind: ClusterSourceTemplate
            name: testing-pipeline-workspace
        - name: source-scanner
          params:
            - default: scan-policy
              name: scanning_source_policy
            - default: blob-source-scan-template
              name: scanning_source_template
          sources:
            - name: source
              resource: source-tester
          templateRef:
            kind: ClusterSourceTemplate
            name: source-scanner-template
        - name: image-provider
          params:
            - default: default
              name: serviceAccount
            - name: registry
              value:
                ca_cert_data: |-
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
                repository: tap-apps
                server: harbor.services.h2o-2-9349.h2o.vmware.com
            - default: default
              name: clusterBuilder
            - default: ./Dockerfile
              name: dockerfile
            - default: ./
              name: docker_build_context
            - default: []
              name: docker_build_extra_args
          sources:
            - name: source
              resource: source-scanner
          templateRef:
            kind: ClusterImageTemplate
            options:
              - name: kpack-template
                selector:
                  matchFields:
                    - key: spec.params[?(@.name=="dockerfile")]
                      operator: DoesNotExist
              - name: kaniko-template
                selector:
                  matchFields:
                    - key: spec.params[?(@.name=="dockerfile")]
                      operator: Exists
        - images:
            - name: image
              resource: image-provider
          name: image-scanner
          params:
            - default: scan-policy
              name: scanning_image_policy
            - default: private-image-scan-template
              name: scanning_image_template
          templateRef:
            kind: ClusterImageTemplate
            name: image-scanner-template
        - images:
            - name: image
              resource: image-scanner
          name: config-provider
          params:
            - default: default
              name: serviceAccount
          templateRef:
            kind: ClusterConfigTemplate
            name: convention-template
        - configs:
            - name: config
              resource: config-provider
          name: app-config
          templateRef:
            kind: ClusterConfigTemplate
            options:
              - name: config-template
                selector:
                  matchLabels:
                    apps.tanzu.vmware.com/workload-type: web
              - name: server-template
                selector:
                  matchLabels:
                    apps.tanzu.vmware.com/workload-type: server
              - name: worker-template
                selector:
                  matchLabels:
                    apps.tanzu.vmware.com/workload-type: worker
        - configs:
            - name: app_def
              resource: app-config
          name: service-bindings
          templateRef:
            kind: ClusterConfigTemplate
            name: service-bindings
        - configs:
            - name: app_def
              resource: service-bindings
          name: api-descriptors
          templateRef:
            kind: ClusterConfigTemplate
            name: api-descriptors
        - configs:
            - name: config
              resource: api-descriptors
          name: config-writer
          params:
            - default: default
              name: serviceAccount
            - name: registry
              value:
                ca_cert_data: |-
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
                repository: tap-apps
                server: harbor.services.h2o-2-9349.h2o.vmware.com
          templateRef:
            kind: ClusterTemplate
            name: config-writer-template
        - name: deliverable
          params:
            - name: registry
              value:
                ca_cert_data: |-
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
                repository: tap-apps
                server: harbor.services.h2o-2-9349.h2o.vmware.com
            - default: go-git
              name: gitImplementation
          templateRef:
            kind: ClusterTemplate
            name: deliverable-template
      selector:
        apps.tanzu.vmware.com/has-tests-needs-workspace: "true"
      selectorMatchExpressions:
        - key: apps.tanzu.vmware.com/workload-type
          operator: In
          values:
            - web
            - server
            - worker
    status: {}
    ```

And then apply it to the cluster.

```sh
kubectl apply -f source-test-scan-to-url.yaml
```

Then we can verify if our new Supply Chain is valid:

```sh
kubectl get ClusterSupplyChain
```

Which should yield:

```sh
NAME                                READY   REASON   AGE
scanning-image-scan-to-url          True    Ready    13h
source-test-scan-to-url             True    Ready    13h
source-test-scan-to-url-workspace   True    Ready    8s
```

## Create External Service with Bitnami Services

https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/bitnami-services-tutorials-working-with-bitnami-services.html

Starting from TAP 1.5, TAP includes Crossplane to help with managing External Services, such as Databases.

By default, it includes prepared Crossplane packages (called Configurations) for several Bitnami Helm Charts.
These are managed via the App `bitnami-services`, which is part of the Iterate, Run, and Full profile.

We will use these services, to create a "production" database for our application.

### View Existing Services

Let's look at the services that we have available.

```sh
tanzu service class list
```

Which should yield the following:

```sh
  NAME                  DESCRIPTION
  mysql-unmanaged       MySQL by Bitnami
  postgresql-unmanaged  PostgreSQL by Bitnami
  rabbitmq-unmanaged    RabbitMQ by Bitnami
  redis-unmanaged       Redis by Bitnami
```

We can explore the services with the Tanzu CLI, for example, the Postgresql:

```sh
tanzu service class get postgresql-unmanaged
```

Which gives a few details, such as the parameters:

```sh
NAME:           postgresql-unmanaged
DESCRIPTION:    PostgreSQL by Bitnami
READY:          true

PARAMETERS:
  KEY        DESCRIPTION                                                  TYPE     DEFAULT  REQUIRED
  storageGB  The desired storage capacity of the database, in Gigabytes.  integer  1        false
```

In this case, there is only one parameter, `storageGB`, which is optional.

### Claim External Service

Let's create an external service to be consumed by a Workload.

```sh
export SQL_CLASS_CLAIM_NAME=psql-1
export TAP_DEVELOPER_NAMESPACE=
```

!!! Important
    If binding to more than one application workload then all application workloads must exist in the same namespace. This is a known limitation. For more information, see [Cannot claim and bind to the same service instance from across multiple namespaces](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/services-toolkit-reference-known-limitations.html#multi-workloads)[^2].

We then create a Class Claim, which means we create a Service of "Class".
This is a static generation, but when defining them dynamically, a Class works like Storage Classes.

For now, we can ignore those details, and we crate our Service by creating the Class Claim.

```sh
tanzu service class-claim create ${SQL_CLASS_CLAIM_NAME} \
  --class postgresql-unmanaged \
  --parameter storageGB="3" \
  --namespace ${TAP_DEVELOPER_NAMESPACE}
```

We can view our Class Claim:

```sh
tanzu services class-claims list -n  ${TAP_DEVELOPER_NAMESPACE}
```

Which shows it is ready:

```sh
  NAME    CLASS                 READY  REASON
  psql-1  postgresql-unmanaged  True   Ready
```

For more details, use the `get` command:


```sh
tanzu services class-claims get ${SQL_CLASS_CLAIM_NAME} \
  --namespace ${TAP_DEVELOPER_NAMESPACE}
```

Which gives this details repsonse:

```sh
Name: psql-1
Namespace: dev
Claim Reference: services.apps.tanzu.vmware.com/v1alpha1:ClassClaim:psql-1
Class Reference:
  Name: postgresql-unmanaged
Parameters:
  storageGB: 3
Status:
  Ready: True
  Claimed Resource:
    Name: 7af21ebe-bf15-4e88-9607-72bfcf9d3cb7
    Namespace: dev
    Group:
    Version: v1
    Kind: Secret
```

If you're curious, you can verify there isn't currently running any database or any pod for that matter, in that Namespace.

```sh
kubectl get po -n $DEV_TEAM_NAMESPACE
```

Which should result into this:

```sh
No resources found in dev-team-1 namespace.
```

## Fork Test Application

Go to the Gitea instance, and find the `shared/spring-boot-postgres` instance.

Or go directly to [it here](https://gitea.services.h2o-2-9349.h2o.vmware.com/shared/spring-boot-postgres).

Then click Fork, to add it to your User.

## Create Workload

Now we can create a Workload that consumes this service.

Assuming the Pipeline and SupplyChain we defined is valid, this will use a Testcontainer database for testing, and a "real" database when deployed.

```sh
export TAP_DEVELOPER_NAMESPACE=dev
export SSH_SECRET=ssh-credentials
export LAB=
```

```sh
tanzu apps workload create spring-boot-postgres-01 \
  --namespace ${TAP_DEVELOPER_NAMESPACE} \
  --git-repo ssh://git@gitssh.h2o-2-9349.h2o.vmware.com/${LAB}/spring-boot-postgres.git \
  --git-branch main \
  --type web \
  --label app.kubernetes.io/part-of=spring-boot-spring-01 \
  --label apps.tanzu.vmware.com/has-tests-needs-workspace=true \
  --annotation autoscaling.knative.dev/minScale=1 \
  --build-env BP_JVM_VERSION=17 \
  --param gitops_ssh_secret=${SSH_SECRET} \
  --service-ref db=services.apps.tanzu.vmware.com/v1alpha1:ClassClaim:${SQL_CLASS_CLAIM_NAME} \
  --yes
```

!!! Warning "SourceScan Error with no Data"

    Unfortunately, the ScanTemplate setup doesn't always handle to secrets correctly.

    The Scan Templates are generated by TAP when you install the profile.

    They need to contact the Metadata Store, and need to trust its Certificate.

    It does so by importing the `app-tls-cert` from the `metadata-store` Namespace.

    Sometimes this fails, and then the Source Scan returns this data:

    ```sh
    kubectl get sourcescan -n $TAP_DEVELOPER_NAMESPACE
    ```

    ```sh
    NAME          PHASE   SCANNEDREVISION   SCANNEDREPOSITORY   AGE   CRITICAL   HIGH   MEDIUM   LOW   UNKNOWN   CVETOTAL
    tap-demo-04   Error                                         19m
    ```

    Read [this paragraph](/tap-workshops/supply-chain/basic-to-test-scan/#create-tls-app-cert-secret-separately) for a more permanent solution.
    
    To remedy this, we can do the secret export and import ourselves with the **SecretGen Controller**:

    ```yaml title="metadata-tls-secret-import-and-export.yaml"
    ---
    apiVersion: secretgen.carvel.dev/v1alpha1
    kind: SecretExport
    metadata:
      name: app-tls-cert
      namespace: metadata-store
    spec:
      toNamespace: dev
    ---
    apiVersion: secretgen.carvel.dev/v1alpha1
    kind: SecretImport
    metadata:
      name: app-tls-cert
      namespace: dev
    spec:
      fromNamespace: metadata-store
    ```

    And then apply it to the cluster.

    ```sh
    kubectl apply -f metadata-tls-secret-import-and-export.yaml \
      -n $TAP_DEVELOPER_NAMESPACE
    ```

## Security Scan Failed

If you look at the Supply Chain view in TAP GUI, you will notice the Source Scan failed.

That is because this application has a few more violations.

> Failed because of 4 violations: CVE postgresql CVE-2015-0244 {"Critical"}. CVE postgresql CVE-2015-3166 {"Critical"}. CVE postgresql CVE-2018-1115 {"Critical"}. CVE postgresql CVE-2019-10211 {"Critical"}

We can add them to the list of ingored CVE's in our current ScanPolicy.

```sh
ignoreCves := ["GHSA-36p3-wjmg-h94x", "CVE-2015-0244", "CVE-2015-3166", "CVE-2018-1115", "CVE-2019-10211", "CVE-2016-1000027"]
```

Or, we can create a new policy, just for this application.

```yaml title="spring-boot-postgres-policy.yaml"
apiVersion: scanning.apps.tanzu.vmware.com/v1beta1
kind: ScanPolicy
metadata:
  labels:
    app.kubernetes.io/part-of: enable-in-gui
  name: scan-policy-spring-boot-postgres
spec:
  regoFile: |
    package main

    # Accepted Values: "Critical", "High", "Medium", "Low", "Negligible", "UnknownSeverity"
    notAllowedSeverities := ["Critical"]
    ignoreCves := ["GHSA-36p3-wjmg-h94x", "CVE-2015-0244", "CVE-2015-3166", "CVE-2018-1115", "CVE-2019-10211", "CVE-2016-1000027"]

    contains(array, elem) = true {
      array[_] = elem
    } else = false { true }

    isSafe(match) {
      severities := { e | e := match.ratings.rating.severity } | { e | e := match.ratings.rating[_].severity }
      some i
      fails := contains(notAllowedSeverities, severities[i])
      not fails
    }

    isSafe(match) {
      ignore := contains(ignoreCves, match.id)
      ignore
    }

    deny[msg] {
      comps := { e | e := input.bom.components.component } | { e | e := input.bom.components.component[_] }
      some i
      comp := comps[i]
      vulns := { e | e := comp.vulnerabilities.vulnerability } | { e | e := comp.vulnerabilities.vulnerability[_] }
      some j
      vuln := vulns[j]
      ratings := { e | e := vuln.ratings.rating.severity } | { e | e := vuln.ratings.rating[_].severity }
      not isSafe(vuln)
      msg = sprintf("CVE %s %s %s", [comp.name, vuln.id, ratings])
    }
```

```sh
kubectl apply -f spring-boot-postgres-policy.yaml\
  -n $TAP_DEVELOPER_NAMESPACE
```


We will [update our Workload](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/namespace-provisioner-use-case1.html?hWord=N4IghgNiBcIMoGMwDsAEAFA9hAlggniAL5A)[^1] to use this specific Policy for its Source and Image scan:

```sh
--param scanning_source_policy="scan-policy-spring-boot-postgres" \
--param scanning_image_policy="scan-policy-spring-boot-postgres" \
```

To update an existing Workload, we can use `tanzu apps workload apply`:

```sh
tanzu apps workload apply spring-boot-postgres-01 \
  --namespace ${TAP_DEVELOPER_NAMESPACE} \
  --git-repo ssh://git@gitssh.h2o-2-9349.h2o.vmware.com/${LAB}/spring-boot-postgres.git \
  --git-branch main \
  --type web \
  --label app.kubernetes.io/part-of=spring-boot-spring-01 \
  --label apps.tanzu.vmware.com/has-tests-needs-workspace=true \
  --annotation autoscaling.knative.dev/minScale=1 \
  --build-env BP_JVM_VERSION=17 \
  --param gitops_ssh_secret=${SSH_SECRET} \
  --param scanning_source_policy="scan-policy-spring-boot-postgres" \
  --param scanning_image_policy="scan-policy-spring-boot-postgres" \
  --service-ref db=services.apps.tanzu.vmware.com/v1alpha1:ClassClaim:${SQL_CLASS_CLAIM_NAME} \
  --yes
```

Push a change the repository to trigger a new build.

### Test The Application

```sh
kubectl get httpproxy -n dev
```

Collect the URL, something like `spring-boot-postgres-01.dev.lab02.h2o-2-9349.h2o.vmware.com`

```sh
export APP_URL=
```

```sh
curl -lk "https://${APP_URL}"
```

Which returns an empty list: 

```sh
[]
```

We can verify the database works by adding an entry then retrieving the results again.

```sh
curl -X POST -lk "https://${APP_URL}/create" -d '{"name": "piet"}' \
  -H "content-type: application/json"
```

Querying the server again:

```sh
curl -lk "https://${APP_URL}"
```

Now results in a value:

```json
[{"id":1,"userId":null,"name":"piet","creationDate":"2023-05-17T11:01:02.136+00:00"}]
```

## References

[^1]: [TAP 1.5 - More Tekton and Scan Policy resources per Namespace](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/namespace-provisioner-use-case1.html?hWord=N4IghgNiBcIMoGMwDsAEAFA9hAlggniAL5A)
[^2]: [TAP 1.5 - Know Limitations With Class Claims](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/services-toolkit-reference-known-limitations.html#multi-workloads)
