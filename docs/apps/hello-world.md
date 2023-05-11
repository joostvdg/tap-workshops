---
title: TAP Workload Demo
description: TAP Demo App - Absolute minimal
author: joostvdg
tags: [tap, kubernetes, spring, java]
---

Coming soon.

## Checks

- [ ] I'm able to use an accelerator to generate a Web TAP Workload
- [ ] I applied a TAP Web Workload and register it within TAP GUI
- [ ] I'm able to do basic troubleshooting of a TAP Workload
- [ ] I've configured App Live View in TAP GUI for a TAP Web Workload
- [ ] I've configured Tanzu Build Service and it works for a Spring Boot Application to produce an Image
- [ ] I am able to view and verify a TAP Workload displays in properly in Supply Chain view of TAP GUI
- [ ] I know how to configure an integration to a private Git provider in TAP GUI
- [ ] I know how to configure access to private repositories (SCM and Container) for TAP Workloads
- [ ] I have configured custom CA's for TAP

To add:

- [ ] I am able to view and verify results for the Security Analysis within TAP GUI

## Java Web

### Steps

* go to TAP GUI -> Create | use VSCode Extention
    * https://tap-gui.view.h2o-2-9349.h2o.vmware.com/create
    * Tanzu Java Web UI
* create project in Gitea
    * make the project public (should be the default)
* Push project to Gitea
* create secret for Gitea with Credentials & CA Cert
    * https://fluxcd.io/flux/components/source/gitrepositories/
* create workload
* (optional) copy Deliverable to Run cluster
* verify application runs

### Commands

```sh
git init
git add .
git commit -m "first commit"
git remote add origin https://gitea.services.h2o-2-9349.h2o.vmware.com/gitea/tap-demo-03.git
git push -u origin main
```

```sh
export TAP_DEVELOPER_NAMESPACE=dev
```

* https://fluxcd.io/flux/components/source/gitrepositories/

```sh
kubectl create secret generic https-credentials \
   --namespace $TAP_DEVELOPER_NAMESPACE \
   --from-file caFile=ssl/ca.crt \
   --from-literal username=gitea \
   --from-literal password=gitea
```


* in existing env we need ` --label apps.tanzu.vmware.com/has-tests-needs-workspace=true \`

```sh
export APP_NAME=tap-demo-03
```

```sh
tanzu apps workload create ${APP_NAME} \
  --namespace ${TAP_DEVELOPER_NAMESPACE} \
  --git-repo https://gitea.services.h2o-2-9349.h2o.vmware.com/gitea/${APP_NAME}.git \
  --git-branch main \
  --type web \
  --label app.kubernetes.io/part-of=${APP_NAME} \
  --label apps.tanzu.vmware.com/has-tests-needs-workspace=true \
  --annotation autoscaling.knative.dev/minScale=1 \
   --param gitops_ssh_secret=https-credentials \
  --yes
```

```sh
kubectl get workload -A
```

```sh
kubectl get GitRepository -A
```

```sh
tanzu apps workload get ${APP_NAME} --namespace dev
```

```sh
tanzu apps workload tail ${APP_NAME} --namespace dev --timestamp --since 1h
```

```sh
export DELIVERABLE_FILE="${APP_NAME}-deliverable.yaml"
```

```sh
kubectl get configmap "${APP_NAME}-deliverable" -n ${TAP_DEVELOPER_NAMESPACE} \
    -o go-template='{{.data.deliverable}}' \
    > ${DELIVERABLE_FILE}

echo "DELIVERABLE_FILE=${DELIVERABLE_FILE}"
```

* Change to Run cluster

```sh
kubectl apply -f ${DELIVERABLE_FILE} -n apps
```

* register application in TAP GUI
* use `./catalog/catalog-info.yaml`
* use the raw version, for `tap-demo-03` URL=https://gitea.services.h2o-2-9349.h2o.vmware.com/gitea/tap-demo-03/raw/branch/main/catalog/catalog-info.yaml

```sh
kubectl get deliverable -n apps
```

```sh
kubectl get httpproxy -A
```

```sh
curl -k "https://tap-demo-03.apps.run-01.h2o-2-9349.h2o.vmware.com/"
```

