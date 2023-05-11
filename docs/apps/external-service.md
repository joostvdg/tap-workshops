---
title: TAP App with External Service
description: TAP Demo App - Spring Boot with Database Using External Service
author: joostvdg
tags: [tap, kubernetes, spring, java, spring-boot, mysql, crossplane]
---

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

## TODO

* bitnami services one
* use spring boot 3
* use Java 17
* create via accelerator
* skip testcontainer -> explain what can be done
* create ssh secret
* go through the steps
    * create http secret
    * add
    * register in TAP GUI
    * show API
    * verify
    * update -> main controller & test
    * run local `mvn test` if you can
    * verify update

## Steps

* go to TAP GUI -> Create | use VSCode Extention
    * https://tap-gui.view.h2o-2-9349.h2o.vmware.com/create
* Possible Choices:
    * Tanzu Java Restful Web App
    * AppSSO Starter Java
    * Tanzu Java Web UI
    * Tanzu Java Web App

* create project via VSCode Extention
* open project (accept prompt)
* create project in Gitea
* add project to Gitea

### Config Used

* Project Name: tap-workload-demo-01
* Artifact Id: customer-profile
* Group Id: com.example
* Package Name: com.example.customerprofile
* Build Tool: maven
* Expose Open API Endpoint: Yes
* Update Boot 3: Yes
* Java Version: 17
* Database Type: postgres
* Database Name: customer-database
* Database Migration Tool: flyway
* Database Integration Test Type: in-memory
* Include Build Tool Wrapper: Yes
* Api System: profile-management
* Api Owner: customer-relations-department
* Api Description: Manage customer profiles
* Database Postgres Storage Class: default
* Live Update IDE Support: Yes
* Source Repository Prefix: dev.local


## Commands

```sh
git init
git add .
git commit -m "first commit"
git remote add origin https://gitea.services.h2o-2-9349.h2o.vmware.com/gitea/demo-02.git
git push -u origin main
```

```sh
export TAP_DEVELOPER_NAMESPACE=dev
```

```sh
kubectl create secret
```

```sh
tanzu apps workload create spring-boot-postgres-01 \
  --namespace dev \
  --git-repo ssh://git@172.16.50.201:22/gitea/spring-boot-postgres.git \
  --git-branch main \
  --type web \
  --label app.kubernetes.io/part-of=spring-boot-spring-01 \
  --label apps.tanzu.vmware.com/has-tests-needs-workspace=true \
  --annotation autoscaling.knative.dev/minScale=1 \
  --build-env BP_JVM_VERSION=17 \
  --param gitops_ssh_secret=gitea-ssh \
  --service-ref db=services.apps.tanzu.vmware.com/v1alpha1:ClassClaim:psql-1 \
  --yes

```