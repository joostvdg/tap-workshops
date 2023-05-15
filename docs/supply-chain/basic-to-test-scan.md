---
title: Test & Scan Supply Chain
description: Upgrade Basic to Test & Scan Supply Chain
author: joostvdg
tags: [tap, kubernetes, install]
---


* https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/getting-started-add-test-and-security.html

## What you will do

* Install OOTB Supply Chain with Testing.
* Add a Tekton pipeline to the cluster and update the workload to point to the pipeline and resolve errors.
* Install OOTB Supply Chain with Testing and Scanning.
* Update the workload to point to the Tekton pipeline and resolve errors.
* Query for vulnerabilities and dependencies

## Steps

* Scanning Pre-requisites
* Update TAP Profile Config file
* Add Tekton Pipeline
* Add ScanPolicy
* Update Workload

## Scanning Prerequisites

* https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/tap-gui-plugins-scc-tap-gui.html#scan

## Update Profile Config

* Supply Chain
* TAP GUI Config
* enable Test and Scaning components

```diff
- supply_chain: testing
+ supply_chain: testing_scanning

- ootb_supply_chain_testing:
+ ootb_supply_chain_testing_scanning:
    registry:
      server: "<SERVER-NAME>"
      repository: "<REPO-NAME>"
```

```yaml
grype:
  db:
    dbUpdateUrl: https://minio.services.h2o-2-9349.h2o.vmware.com/grype/databases/listing.json
```

```yaml
scanning:
  metadataStore:
    url: ""
```

### Updated Profile Template

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

grype:
  db:
    dbUpdateUrl: https://minio.services.h2o-2-9349.h2o.vmware.com/grype/databases/listing.json

scanning:
  metadataStore:
    url: ""

ceip_policy_disclosed: true
excluded_packages:
  - eventing.tanzu.vmware.com #! not used, so removing to reduce memory/cpu footprint
  - tap-telemetry.tanzu.vmware.com #! not used, so removing to reduce memory/cpu footprint
```


## Verify Components

* metadata-store

```sh
tanzu package installed get metadata-store -n tap-install
tanzu package installed get scanning -n tap-install
tanzu package installed get grype -n tap-install
```

## Query For Vulnerabilities

* https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/getting-started-add-test-and-security.html#query-for-vulnerabilities-8

```sh
tanzu insight image get --digest DIGEST
tanzu insight image vulnerabilities --digest  DIGEST
```