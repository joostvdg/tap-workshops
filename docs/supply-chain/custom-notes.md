# Notes On Workshop

## Cartographer

* harbor.services.h2o-2-9349.h2o.vmware.com/tap-apps/tap-demo-03-dev@sha256:5e7615dd7e9e02dc32b96e089a87870c19d55d143ecb9eea3eb701ad8cba8013
git
* ghcr.io/joostvdg/go-demo:v2.1.16
* https://github.com/joostvdg/go-demo/blob/main/k8s/deployment.yaml

To construct and use a Supply Chain, we need the following ingredients:

1. One or more ***Resources***, usually Cartographer templates
1. A SupplyChain definition, using those Resources
1. A Workload definition that _selects_ the Supply Chain

### My First Template

!!! Tip "Relies on Workload Definition"

    Cartographer works from a **Workload** definition.

    So any template, like the **ClusterTemplate** below, works because the trigger is a **Workload** CR.

```yaml title="cluster-template-app-deploy.yaml"
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

For the Workload definition, it needs to supply the following fields:

* `metadata.name`
* `spec.image`

### Supply Chain Definition

> The supply chain has three top level fields in its spec, the resources, a service account reference and a selector for workloads.

1. **Resources**: Which CRs are used (and how)
1. **ServiceAccount Reference**: reference to a ServiceAccount that can use the Templates/Resources
1. **Selector**: Set of Kubernetes **Labels** which you have to specify on a **Workload** to trigger this Supply Chain


```yaml title="my-first-supply-chain.yaml"
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

Remember, we need to specify the following fields:

* `metadata.name`
* `spec.image`

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

```yaml title="rbac.yaml"
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

### Putting it together

```sh
kubectl apply -f rbac.yaml
kubectl apply -f cluster-template-app-deploy.yaml
kubectl apply -f my-first-supply-chain.yaml
kuebctl apply -f workload-pre-built-hello.yaml
```

```sh
kubectl tree workload hello
```

```sh
NAMESPACE  NAME                                       READY  REASON  AGE
default    Workload/hello                             True   Ready   98m
default    └─Deployment/hello-deployment              -              98m
default      └─ReplicaSet/hello-deployment-cfdf74d6   -              98m
default        ├─Pod/hello-deployment-cfdf74d6-kjjp5  True           98m
default        ├─Pod/hello-deployment-cfdf74d6-wgtvl  True           98m
default        └─Pod/hello-deployment-cfdf74d6-x2pmc  True           98m
```

```sh
kubectl port-forward deployment/hello-deployment 8080:8080
```

```sh
curl http://localhost:8080/
```

## With Params

* https://cartographer.sh/docs/v0.7.0/tutorials/using-params/

```sh
kubectl port-forward deployment/hello-deployment 8080:8080
```

```sh
curl http://localhost:8080/
```

## Extending

* https://cartographer.sh/docs/v0.7.0/tutorials/extending-a-supply-chain/
* https://github.com/paketo-buildpacks/go-build
* https://github.com/joostvdg/go-demo/
* https://github.com/pivotal/kpack/blob/main/docs/image.md


```sh
tanzu apps workload tail source-code-01 --namespace dev --timestamp --since 1h
```

## Tekton LifeCycle


## OOTB Supply Chain

### Build Cluster

```sh
kubectl tree workload -n dev tap-demo-03   
```

```sh
NAMESPACE  NAME                                                READY  REASON              AGE
dev        Workload/tap-demo-03                                True   Ready               5d2h
dev        ├─ConfigMap/tap-demo-03                             -                          5d
dev        ├─ConfigMap/tap-demo-03-deliverable                 -                          5d2h
dev        ├─ConfigMap/tap-demo-03-with-api-descriptors        -                          5d
dev        ├─ConfigMap/tap-demo-03-with-claims                 -                          5d
dev        ├─GitRepository/tap-demo-03                         True   Succeeded           5d2h
dev        ├─Image/tap-demo-03                                 True                       5d1h
dev        │ ├─Build/tap-demo-03-build-1                       -                          5d1h
dev        │ │ └─Pod/tap-demo-03-build-1-build-pod             False  PodCompleted        5d1h
dev        │ ├─Build/tap-demo-03-build-2                       -                          5d
dev        │ │ └─Pod/tap-demo-03-build-2-build-pod             False  PodCompleted        5d
dev        │ ├─Build/tap-demo-03-build-3                       -                          5d
dev        │ │ └─Pod/tap-demo-03-build-3-build-pod             False  PodCompleted        5d
dev        │ ├─PersistentVolumeClaim/tap-demo-03-cache         -                          5d1h
dev        │ └─SourceResolver/tap-demo-03-source               True                       5d1h
dev        ├─ImageScan/tap-demo-03                             -                          5d1h
dev        │ ├─TaskRun/scan-tap-demo-03-29z7x                  -                          5d
dev        │ │ └─Pod/scan-tap-demo-03-29z7x-pod                False  PodCompleted        5d
dev        │ ├─TaskRun/scan-tap-demo-03-2hwn7                  -                          5d
dev        │ │ └─Pod/scan-tap-demo-03-2hwn7-pod                False  PodCompleted        5d
dev        │ ├─TaskRun/scan-tap-demo-03-bzrb9                  -                          5d
dev        │ │ └─Pod/scan-tap-demo-03-bzrb9-pod                False  PodCompleted        5d
dev        │ └─TaskRun/scan-tap-demo-03-phsmz                  -                          5d1h
dev        │   └─Pod/scan-tap-demo-03-phsmz-pod                False  PodCompleted        5d1h
dev        ├─PodIntent/tap-demo-03                             True   ConventionsApplied  5d
dev        ├─Runnable/tap-demo-03                              True   Ready               5d2h
dev        │ ├─PipelineRun/tap-demo-03-46rnd                   -                          5d
dev        │ │ ├─PersistentVolumeClaim/pvc-16560cea6b          -                          5d
dev        │ │ ├─TaskRun/tap-demo-03-46rnd-fetch-repository    -                          5d
dev        │ │ │ └─Pod/tap-demo-03-46rnd-fetch-repository-pod  False  PodCompleted        5d
dev        │ │ └─TaskRun/tap-demo-03-46rnd-maven-run           -                          5d
dev        │ │   └─Pod/tap-demo-03-46rnd-maven-run-pod         False  PodCompleted        5d
dev        │ ├─PipelineRun/tap-demo-03-66fp8                   -                          5d
dev        │ │ ├─PersistentVolumeClaim/pvc-8bd0367a05          -                          5d
dev        │ │ ├─TaskRun/tap-demo-03-66fp8-fetch-repository    -                          5d
dev        │ │ │ └─Pod/tap-demo-03-66fp8-fetch-repository-pod  False  PodCompleted        5d
dev        │ │ └─TaskRun/tap-demo-03-66fp8-maven-run           -                          5d
dev        │ │   └─Pod/tap-demo-03-66fp8-maven-run-pod         False  PodCompleted        5d
dev        │ ├─PipelineRun/tap-demo-03-6xtlf                   -                          5d
dev        │ │ ├─PersistentVolumeClaim/pvc-8bd736ceef          -                          5d
dev        │ │ ├─TaskRun/tap-demo-03-6xtlf-fetch-repository    -                          5d
dev        │ │ │ └─Pod/tap-demo-03-6xtlf-fetch-repository-pod  False  PodCompleted        5d
dev        │ │ └─TaskRun/tap-demo-03-6xtlf-maven-run           -                          5d
dev        │ │   └─Pod/tap-demo-03-6xtlf-maven-run-pod         False  PodFailed           5d
dev        │ ├─PipelineRun/tap-demo-03-ff5dn                   -                          5d1h
dev        │ │ ├─PersistentVolumeClaim/pvc-b5da4a07af          -                          5d1h
dev        │ │ ├─TaskRun/tap-demo-03-ff5dn-fetch-repository    -                          5d1h
dev        │ │ │ └─Pod/tap-demo-03-ff5dn-fetch-repository-pod  False  PodCompleted        5d1h
dev        │ │ └─TaskRun/tap-demo-03-ff5dn-maven-run           -                          5d1h
dev        │ │   └─Pod/tap-demo-03-ff5dn-maven-run-pod         False  PodFailed           5d1h
dev        │ └─PipelineRun/tap-demo-03-wfswt                   -                          5d2h
dev        │   ├─PersistentVolumeClaim/pvc-29e80c6f81          -                          5d2h
dev        │   ├─TaskRun/tap-demo-03-wfswt-fetch-repository    -                          5d2h
dev        │   │ └─Pod/tap-demo-03-wfswt-fetch-repository-pod  False  PodCompleted        5d2h
dev        │   └─TaskRun/tap-demo-03-wfswt-maven-run           -                          5d2h
dev        │     └─Pod/tap-demo-03-wfswt-maven-run-pod         False  PodCompleted        5d2h
dev        ├─Runnable/tap-demo-03-config-writer                True   Ready               5d
dev        │ ├─TaskRun/tap-demo-03-config-writer-gllk2         -                          5d
dev        │ │ └─Pod/tap-demo-03-config-writer-gllk2-pod       False  PodCompleted        5d
dev        │ └─TaskRun/tap-demo-03-config-writer-qgsk5         -                          5d
dev        │   └─Pod/tap-demo-03-config-writer-qgsk5-pod       False  PodCompleted        5d
dev        └─SourceScan/tap-demo-03                            -                          5d1h
dev          ├─TaskRun/scan-tap-demo-03-7dnjg                  -                          5d
dev          │ └─Pod/scan-tap-demo-03-7dnjg-pod                False  PodCompleted        5d
dev          ├─TaskRun/scan-tap-demo-03-9vlxv                  -                          5d
dev          │ └─Pod/scan-tap-demo-03-9vlxv-pod                False  PodCompleted        5d
dev          ├─TaskRun/scan-tap-demo-03-kv2sv                  -                          5d
dev          │ └─Pod/scan-tap-demo-03-kv2sv-pod                False  PodCompleted        5d
dev          └─TaskRun/scan-tap-demo-03-nf6j6                  -                          5d1h
dev            └─Pod/scan-tap-demo-03-nf6j6-pod                False  PodCompleted        5d1h
```

### Run Cluster

```sh
kubectl tree deliverable tap-demo-03 -n apps
```

```sh
NAMESPACE  NAME                                    READY  REASON  AGE
apps       Deliverable/tap-demo-03                 True   Ready   4d23h
apps       ├─App/tap-demo-03                       -              4d23h
apps       └─ImageRepository/tap-demo-03-delivery  True   Ready   4d23h
```

### Iterate Cluster

```sh
kubectl tree workload -n dev smoke-app
```

```sh
NAMESPACE  NAME                                         READY  REASON              AGE
dev        Workload/smoke-app                           True   Ready               2d23h
dev        ├─ConfigMap/smoke-app                        -                          2d23h
dev        ├─ConfigMap/smoke-app-with-api-descriptors   -                          2d23h
dev        ├─ConfigMap/smoke-app-with-claims            -                          2d23h
dev        ├─Deliverable/smoke-app                      True   Ready               2d23h
dev        │ ├─App/smoke-app                            -                          2d23h
dev        │ └─ImageRepository/smoke-app-delivery       True   Ready               2d23h
dev        ├─GitRepository/smoke-app                    True   Succeeded           2d23h
dev        ├─Image/smoke-app                            True                       2d23h
dev        │ ├─Build/smoke-app-build-1                  -                          2d23h
dev        │ │ └─Pod/smoke-app-build-1-build-pod        False  PodCompleted        2d23h
dev        │ ├─PersistentVolumeClaim/smoke-app-cache    -                          2d23h
dev        │ └─SourceResolver/smoke-app-source          True                       2d23h
dev        ├─PodIntent/smoke-app                        True   ConventionsApplied  2d23h
dev        └─Runnable/smoke-app-config-writer           True   Ready               2d23h
dev          └─TaskRun/smoke-app-config-writer-wdlzc    -                          2d23h
dev            └─Pod/smoke-app-config-writer-wdlzc-pod  False  PodCompleted        2d23h
```

```sh
kubectl tree deliverable -n dev smoke-app
```

```sh
NAMESPACE  NAME                                  READY  REASON  AGE
dev        Deliverable/smoke-app                 True   Ready   2d23h
dev        ├─App/smoke-app                       -              2d23h
dev        └─ImageRepository/smoke-app-delivery  True   Ready   2d23h
```

## Modify OOTB Supply Chains

* https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/scc-authoring-supply-chains.html#modifying-an-out-of-the-box-supply-chain-2

To change the shape of a supply chain or the template that it points to, do the following:

1. Copy one of the reference supply chains.
1. Remove the old supply chain. See preventing Tanzu Application Platform supply chains from being installed.
1. Edit the supply chain object.
1. Submit the modified supply chain to the cluster

## Tekton Tutorial

* https://tekton.dev/docs/getting-started/
* https://tekton.dev/docs/getting-started/tasks/
* https://cartographer.sh/docs/v0.7.0/tutorials/lifecycle/
* https://tekton.dev/docs/results/
* https://github.com/tektoncd/catalog/tree/main/task
* Adding custom behavior to Supply Chains: https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/scc-authoring-supply-chains.html#adding-custom-behavior-to-supply-chains-7

### Tekton Hello World

```sh
export TAP_DEVELOPMENT_NAMESPACE=dev
```

```sh
kubectl apply -f 01-task-hello.yaml -n ${TAP_DEVELOPMENT_NAMESPACE}
```

```sh
kubectl apply -f 02-task-run.yaml -n ${TAP_DEVELOPMENT_NAMESPACE}
```

```sh
kubectl get taskrun hello-task-run -n ${TAP_DEVELOPMENT_NAMESPACE}
```

```sh
kubectl -n ${TAP_DEVELOPMENT_NAMESPACE} logs hello-task-run-pod -c step-echo
```

```sh
kubectl delete -f 01-task-hello.yaml -n ${TAP_DEVELOPMENT_NAMESPACE}
kubectl delete -f 02-task-run.yaml -n ${TAP_DEVELOPMENT_NAMESPACE}
```

### Tekton Pipeline

```sh
kubectl -n dev logs hello-goodbye-run-goodbye-pod -c step-goodbye
```

```sh
kubectl create -f 04-pipeline-run-dynamic.yaml -n ${TAP_DEVELOPMENT_NAMESPACE}
```

```sh
kubectl get taskrun,pipelinerun -n dev
```

### Tekton Pipeline With Workspace

```sh
kubectl get taskrun,pipelinerun -n dev
```

```sh
kubectl get pod  -n ${TAP_DEVELOPMENT_NAMESPACE}
```

```sh
kubectl -n ${TAP_DEVELOPMENT_NAMESPACE} logs  clone-git-next-tag-run-z5vrr-git-next-tag-pod
```

``sh
kubectl get taskrun -n dev -l tekton.dev/task=git-next-tag
```

```sh
kubectl get taskrun -n dev -l tekton.dev/task=git-next-tag -ojson | jq '.items | map(.status.taskResults)'
```


### App Deploy 04

```sh
kubectl tree workload -n dev hello-again
```

```sh
tanzu apps workload tail hello-04 --namespace dev --timestamp --since 1h
```


```sh
kubectl port-forward deployment/hello-deployment 8080:8080
```

```sh
curl http://localhost:8080/
```


### Examples

* ClusterSupplyChain
  * ClusterSourceTemplate
    * ClusterRunTemplate
      * PipelineRun
        * Task
          * TaskRun (Generated)
            * Pod (Generated)
              * InitContainer -> Shell (Generated)

```yaml
apiVersion: carto.run/v1alpha1
kind: ClusterSupplyChain
spec:
  resources:
    - name: source-tester
      sources:
        - name: source
          resource: source-provider
      templateRef:
        kind: ClusterSourceTemplate
        name: testing-pipeline-workspace
```

```yaml
apiVersion: carto.run/v1alpha1
kind: ClusterSourceTemplate
spec:
  ytt: ...
```

```yaml title="ClusterRunTemplate-tektonsource-pipelinerun.yaml"
apiVersion: carto.run/v1alpha1
kind: ClusterRunTemplate
metadata:
  annotations:
    kapp.k14s.io/identity: v1;/carto.run/ClusterRunTemplate/tekton-source-pipelinerun;carto.run/v1alpha1
    kapp.k14s.io/original: '{"apiVersion":"carto.run/v1alpha1","kind":"ClusterRunTemplate","metadata":{"labels":{"kapp.k14s.io/app":"1683272492779471078","kapp.k14s.io/association":"v1.72b4cf08dac7ed1e3cac533f6a62ff76"},"name":"tekton-source-pipelinerun"},"spec":{"outputs":{"revision":"spec.params[?(@.name==\"source-revision\")].value","url":"spec.params[?(@.name==\"source-url\")].value"},"template":{"apiVersion":"tekton.dev/v1beta1","kind":"PipelineRun","metadata":{"generateName":"$(runnable.metadata.name)$-","labels":"$(runnable.metadata.labels)$"},"spec":{"params":"$(runnable.spec.inputs.tekton-params)$","pipelineRef":{"name":"$(selected.metadata.name)$"}}}}}'
    kapp.k14s.io/original-diff-md5: c6e94dc94aed3401b5d0f26ed6c0bff3
  creationTimestamp: "2023-05-05T07:41:34Z"
  generation: 1
  labels:
    kapp.k14s.io/app: "1683272492779471078"
    kapp.k14s.io/association: v1.72b4cf08dac7ed1e3cac533f6a62ff76
  name: tekton-source-pipelinerun
  resourceVersion: "10224684"
  uid: 34c133ce-228d-4f0d-85df-fe60bae905f7
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
```
