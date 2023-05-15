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
* 

```diff
- supply_chain: testing
+ supply_chain: testing_scanning

- ootb_supply_chain_testing:
+ ootb_supply_chain_testing_scanning:
    registry:
      server: "<SERVER-NAME>"
      repository: "<REPO-NAME>"

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