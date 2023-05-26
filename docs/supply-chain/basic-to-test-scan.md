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

One of the key components of the Scanning and Testing Supply Chain, is [Grype](https://github.com/anchore/grype)[^1].

It cross references packages from SBOM files with CVE databases.

In restricted environments, Grype cannot retrieve these databases. So we bring it to Grype instead.

If you want to learn how you can relocate the Grype database, [read this](https://joostvdg.github.io/tanzu/grype-airgapped/#relocate-grype-database)[^2].

??? Example "Grype Database Relocation/Update Script"

    ```sh
    #!/bin/bash
    set -euo pipefail
    pushd $(dirname $0)

    MINIO_HOSTNAME=${MINIO_HOSTNAME:-"localhost"}

    echo "> Removing existing listing.json"
    rm listing.json || true

    echo "> Downloading new listing.json from Grype's databases repo"
    http --download https://toolbox-data.anchore.io/grype/databases/listing.json

    echo "> Stripping listing to latest file only"
    cp listing.json listing_original.json
    echo '{"available": {"5": [' > listing_tmp.json
    cat listing_original.json | jq '.available."5"[0]' >> listing_tmp.json
    echo ']}}' >> listing_tmp.json
    cat listing_tmp.json | jq > listing.json

    echo "> Generate download script"
    echo "#!/bin/bash" > grype_down.sh
    echo "set -euo pipefail" >> grype_down.sh

    cat listing.json |jq -r '.available[] | values[].url' \
      | awk '{print "http --download " $1}' >> grype_down.sh
    chmod +x grype_down.sh

    echo "> Removing existing database files"
    rm *.tar.gz || true

    echo "> Downloading new database files"
    ./grype_down.sh

    echo "> Uploading database files to MinIO"
    mc cp *.tar.gz minio_h20/grype/databases/

    echo "> Update listing file"
    cp listing.json listing_copy.json
    sed -i -e \
      "s/https:\/\/toolbox-data.anchore.io\/grype/https:\/\/$MINIO_HOSTNAME\/grype/g" \
      listing.json

    echo "> Upload updated listing file"
    mc cp listing.json minio_h20/grype/databases/

    echo "> View folder in MinIO"
    mc ls minio_h20/grype/databases/
    ```

To shortcut the configuration, we've already configured an "offline" storage of the Grype database.
To use it, we use the snippet below.

```yaml
grype:
  db:
    dbUpdateUrl: https://minio.services.h2o-2-9349.h2o.vmware.com/grype/databases/listing.json
```

!!! Warning "TAP GUI config for Gitea"
    If you skipped the Hello World Workload Lab, you're missing the Gitea config for the TAP GUI.

    Which looks as follows:

    ```yaml
    tap_gui:
      app_config:
        backend:
          reading:
            allow:
              - host: #@ dv.gitServer
        integrations:
          gitea:
            - host: #@ dv.gitServer
              username: #@ dv.gitUser
              password: #@ dv.gitPassword
    ```

    For completeness, we assume you want this configuration regardless.

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

supply_chain: testing_scanning
ootb_supply_chain_testing_scanning:
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
    backend:
      reading:
        allow:
          - host: #@ dv.gitServer
    integrations:
      gitea:
        - host: #@ dv.gitServer
          username: #@ dv.gitUser
          password: #@ dv.gitPassword
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

ceip_policy_disclosed: true
excluded_packages:
  - eventing.tanzu.vmware.com #! not used, so removing to reduce memory/cpu footprint
  - tap-telemetry.tanzu.vmware.com #! not used, so removing to reduce memory/cpu footprint
```

### Generate updated profile

```sh
export GIT_SERVER=gitea.services.h2o-2-9349.h2o.vmware.com
export GIT_USER=gitea
export GIT_PASSWORD='VMware123!'
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
  -v buildRegistrySecret="$TAP_BUILD_REGISTRY_SECRET" \
  -v buildRepo="$BUILD_REGISTRY_REPO" \
  -v tbsRepo="$TBS_REPO" \
  -v domainName="$DOMAIN_NAME" \
  -v caCert="${CA_CERT}" \
  -v gitUser="${GIT_USER}" \
  -v gitPassword="${GIT_PASSWORD}" \
  -v gitServer="${GIT_SERVER}" \
  > "tap-values-full.yml"
```

We recommend you inspect the generated file:

```sh
cat tap-values-full.yml
```

### Install Updated Profile


We then update the TAP installation via the same command.

```sh
export TAP_INSTALL_NAMESPACE=tap-install
export TAP_VERSION=1.5.0
```

```sh
tanzu package install tap \
  -p tap.tanzu.vmware.com \
  -v $TAP_VERSION \
  --values-file tap-values-full.yml \
  -n ${TAP_INSTALL_NAMESPACE}
```

## Verify Components

* metadata-store
* scanning
* grype

```sh
tanzu package installed get metadata-store -n tap-install
tanzu package installed get scanning -n tap-install
tanzu package installed get grype -n tap-install
```
## Add Scan Policy

We cannot yet run our Testing & Scanning pipeline, we need a Scan Policy!

The Scan Policy contains the rules by which to judge the outcome of the vulnerability scans.

For more information about this policy, refer to [the TAP docs](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/scc-ootb-supply-chain-testing-scanning.html)[^3].

Let's create a Policy:

```yaml title="scan-policy.yaml"
apiVersion: scanning.apps.tanzu.vmware.com/v1beta1
kind: ScanPolicy
metadata:
  name: scan-policy
  labels:
    'app.kubernetes.io/part-of': 'enable-in-gui'
spec:
  regoFile: |
    package main

    # Accepted Values: "Critical", "High", "Medium", "Low", "Negligible", "UnknownSeverity"
    notAllowedSeverities := ["Critical", "High", "UnknownSeverity"]
    ignoreCves := []

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

And apply this to the TAP Developer Namespace.

```sh
export TAP_DEVELOPER_NAMESPACE=dev
```

```sh
kubectl apply -f scan-policy.yaml \
  -n $TAP_DEVELOPER_NAMESPACE
```

!!! Info "Update The Policy To Reflect Reality"

    It is better to fix the leak before attempting to clear the bucket.

    So you might want to setup lest strict rules to start with, so that people get time to resolve them.

    For example, in our test application (see next section) there are some vulnerabilities.

    As I don't care too much about those at this point in time, I will update the policy.

    First, I'll restrict the `notAllowedSeverities` to `Critical` only.

    And then I add the known vulnerabilities in that category. If new Criticals show up, it will fail, but for now, we can start the pipeline

    ```sh
    notAllowedSeverities := ["Critical"]
    ignoreCves := ["CVE-2016-1000027", "CVE-2016-0949","CVE-2017-11291","CVE-2018-12805","CVE-2018-4923","CVE-2021-40719","CVE-2018-25076","GHSA-45hx-wfhj-473x","GHSA-jvfv-hrrc-6q72","CVE-2018-12804","GHSA-36p3-wjmg-h94x","GHSA-36p3-wjmg-h94x","GHSA-6v73-fgf6-w5j7"]
    ```

    Read the [Triaging and Remediating CVEs guide](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/scst-scan-triaging-and-remediating-cves.html)[^4] for more information.

## Add Tekton Pipeline

The **Test** in the Testing and Scanning Supply Chain is executed by a Tekton Pipeline.

The Supply Chain contains a **PipelineRun** template which assumes a Tekton **Pipeline** with a specific name and input parameters exist.

This **Pipeline** is not included out of the box, though [the docs do have an example](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/getting-started-add-test-and-security.html#tekton-pipeline-config-example-3)[^5].

We start with using this example (except for the image,  to avoid Dockerhub rate limiting), as it fits our example application.

Later, if you want, we'll guide you to a more elaborate solution.

```yaml title="tekton-pipeline-test.yaml"
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: developer-defined-tekton-pipeline
  labels:
    apps.tanzu.vmware.com/pipeline: test     # (!) required
spec:
  params:
    - name: source-url                       # (!) required
    - name: source-revision                  # (!) required
  tasks:
    - name: test
      params:
        - name: source-url
          value: $(params.source-url)
        - name: source-revision
          value: $(params.source-revision)
      taskSpec:
        params:
          - name: source-url
          - name: source-revision
        steps:
          - name: test
            image: harbor.services.h2o-2-9349.h2o.vmware.com/dockerhub/library/gradle
            script: |-
              cd `mktemp -d`

              wget -qO- $(params.source-url) | tar xvz -m
              ./mvnw test
```

Once you've created the file, apply it to the cluster.
As it is a Namespaced resource, add it to your TAP Developer Namespace.

```sh
kubectl apply -f tekton-pipeline-test.yaml \
  -n $TAP_DEVELOPER_NAMESPACE
```

## Update Workload

Before, the Basic Supply Chain would pick any Workload.
The Testing & Scanning Supply Chain only picks up Workloads that contain the label `apps.tanzu.vmware.com/has-tests=true`.

if you look at your existing Workloads, you'll see that they no longer match a valid Supply Chain:

```sh
tanzu apps workload list \
  -n $TAP_DEVELOPER_NAMESPACE
```

Which shows:

```sh
NAME                 TYPE   APP                  READY                 AGE
tanzu-java-web-app   web    tanzu-java-web-app   SupplyChainNotFound   13h
tap-demo-04          web    tap-demo-04          SupplyChainNotFound   165m
```

Let's change this, by adding the missing label to our Workload.

=== "Existing Workload with Tanzu CLI"
    ```sh
    export APP_NAME=
    ```

    ```sh
    tanzu apps workload update $APP_NAME \
      -n $TAP_DEVELOPER_NAMESPACE \
      --label "apps.tanzu.vmware.com/has-tests=true"
    ```

=== "Existing Workload with Manifest"
    ```sh title="config/workload.yaml" hl_lines="7"
    apiVersion: carto.run/v1alpha1
    kind: Workload
    metadata:
      name: tap-demo-04
      labels:
        apps.tanzu.vmware.com/workload-type: web
        apps.tanzu.vmware.com/has-tests: "true"
        apps.tanzu.vmware.com/auto-configure-actuators: "true"
        app.kubernetes.io/part-of: tap-demo-04
    spec:
      build:
        env:
          - name: BP_JVM_VERSION
            value: "17"
      params:
      - name: gitops_ssh_secret
        value: ssh-credentials
      - name: annotations
        value:
          autoscaling.knative.dev/minScale: "1"
      source:
        git:
          url: ssh://git@gitssh.h2o-2-9349.h2o.vmware.com/lab02/tap-demo-04.git
          ref:
            branch: main
    ```

=== "New Sample Workload with Tanzu CLI"
    ```sh
    tanzu apps workload create smoke-app \
      --git-repo https://github.com/sample-accelerators/tanzu-java-web-app.git \
      --git-branch main \
      --type web \
      --label app.kubernetes.io/part-of=smoke-app \
      --label apps.tanzu.vmware.com/has-tests=true \
      --annotation autoscaling.knative.dev/minScale=1 \
      --yes \
      -n "$TAP_DEVELOPER_NAMESPACE"
    ```

Once this is done, verify our Tekton Pipeline was run correctly:

```sh
kubectl get pipelinerun,taskrun \
  -n $TAP_DEVELOPER_NAMESPACE
```

Which should yield something like this:

```sh
NAME                                       SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
pipelinerun.tekton.dev/tap-demo-04-f62k8   True        Succeeded   3m53s       2m28s

NAME                                                        SUCCEEDED   REASON                   STARTTIME   COMPLETIONTIME
taskrun.tekton.dev/scan-tap-demo-04-9xjcf                   False       CouldntGetTask           54s         53s
taskrun.tekton.dev/tap-demo-04-f62k8-test                   True        Succeeded                3m53s       2m28s
```

And verify the scans:

```sh
kubectl get sourcescan,imagescan \
  -n $TAP_DEVELOPER_NAMESPACE
```

Which should yield something like this:

```sh
NAME                                                    PHASE       SCANNEDREVISION                                 SCANNEDREPOSITORY                                                                                                                              AGE   CRITICAL   HIGH   MEDIUM   LOW   UNKNOWN   CVETOTAL
sourcescan.scanning.apps.tanzu.vmware.com/smoke-app     Completed   main/6db88c7a7e7dec1843809b058195b68480c4c12a   http://fluxcd-source-controller.flux-system.svc.cluster.local./gitrepository/dev/smoke-app/6db88c7a7e7dec1843809b058195b68480c4c12a.tar.gz     16m   1          0      0        0     0         1
sourcescan.scanning.apps.tanzu.vmware.com/tap-demo-04   Completed   main/c3a1da9983f56c0ff3d595077b2f12ff238cb92a   http://fluxcd-source-controller.flux-system.svc.cluster.local./gitrepository/dev/tap-demo-04/c3a1da9983f56c0ff3d595077b2f12ff238cb92a.tar.gz   10h   1          0      0        0     0         1

NAME                                                   PHASE       SCANNEDIMAGE                                                                                                                                 AGE   CRITICAL   HIGH   MEDIUM   LOW   UNKNOWN   CVETOTAL
imagescan.scanning.apps.tanzu.vmware.com/smoke-app     Completed   harbor.services.h2o-2-9349.h2o.vmware.com/tap-apps/smoke-app-dev@sha256:9c67da3d7b6ff8bc4e622aa6f213e4a8acd3c602fa79c06dd517e3d1c50b80a1     14m   1          8      10       16    0         35
imagescan.scanning.apps.tanzu.vmware.com/tap-demo-04   Completed   harbor.services.h2o-2-9349.h2o.vmware.com/tap-apps/tap-demo-04-dev@sha256:c63c3003e8f57c0bd6583d90e2f35a28025c85a93ddb9a2c67542f39cc07ffff   16m   0          3      5        16    0         24
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

    To remedy this, we can do the secret export and import ourselves with the **SecretGen Controller**:

    Note, this is a **temporary** solution, as TAP will remove the secret in the `dev` namespace within a few minutes.

    Read [this paragraph](/tap-workshops/supply-chain/basic-to-test-scan/#create-tls-app-cert-secret-separately) for a more permanent solution.

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

## Query For Vulnerabilities

If there are CVE's found, you might want to investigate them.

You can do so via the **Insight** plugin for the Tanzu CLI.

It is an optional excercise, for which I recommend [following the docs](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/getting-started-add-test-and-security.html#query-for-vulnerabilities-8)[^6]

## Enable CVE scan results in TAP GUI

If you have looked at the Supply Chain view on TAP GUI for your Workloads, you might have noticed there are no scan or CVE results there.

Either in the **Security Analysis** screen, or the Source Scanner and Image Scanner steps, the CVE tables are empty.

That is because the [TAP GUI does not have access](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/tap-gui-plugins-scc-tap-gui.html?hWord=N4IghgNiBcIBoFoDCBXAzgFwPYFsEGUsUAnAYwFMQBfIA)[^7] to the **Metadata Store** by default.

Let us enable this!

### Obtain Read Token

We can either [create a new write token](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/scst-store-create-service-account.html#rw-serv-accts)[^9], 
or we can use the [already created read token](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/scst-store-retrieve-access-tokens.html)[^8]

I prefer not to create new secrets, which we'll have to manage, unless absolutely necessary.

So we retrieve the existing token.

We do so, via this command:

```sh
export METADATA_TOKEN=$(kubectl get secret metadata-store-read-write-client \
  -n metadata-store -o jsonpath="{.data.token}"\
   | base64 -d)
echo "METADATA_TOKEN=$METADATA_TOKEN"
```

### Update TAP Profile

```yaml
tap_gui:
  app_config:
    proxy:
      /metadata-store:
        target: https://metadata-store-app.metadata-store:8443/api/v1
        changeOrigin: true
        secure: false
        headers:
          Authorization: "Bearer ACCESS-TOKEN"
          X-Custom-Source: project-star
```

!!! Warning "Replace ACCESS-TOKEN"

    If you manually update the Values file, remember to replace `ACCESS-TOKEN` with the actual token!

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

supply_chain: testing_scanning
ootb_supply_chain_testing_scanning:
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
    proxy:
      /metadata-store:
        target: https://metadata-store-app.metadata-store:8443/api/v1
        changeOrigin: true
        secure: false
        headers:
          Authorization: #@ "Bearer "+dv.metadataToken
          X-Custom-Source: project-star
    backend:
      reading:
        allow:
          - host: #@ dv.gitServer
    integrations:
      gitea:
        - host: #@ dv.gitServer
          username: #@ dv.gitUser
          password: #@ dv.gitPassword
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

ceip_policy_disclosed: true
excluded_packages:
  - eventing.tanzu.vmware.com #! not used, so removing to reduce memory/cpu footprint
  - tap-telemetry.tanzu.vmware.com #! not used, so removing to reduce memory/cpu footprint
```

### Generate updated profile

```sh
export GIT_SERVER=gitea.services.h2o-2-9349.h2o.vmware.com
export GIT_USER=gitea
export GIT_PASSWORD='VMware123!'
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
  -v buildRegistrySecret="$TAP_BUILD_REGISTRY_SECRET" \
  -v buildRepo="$BUILD_REGISTRY_REPO" \
  -v tbsRepo="$TBS_REPO" \
  -v domainName="$DOMAIN_NAME" \
  -v caCert="${CA_CERT}" \
  -v gitUser="${GIT_USER}" \
  -v gitPassword="${GIT_PASSWORD}" \
  -v gitServer="${GIT_SERVER}" \
  -v metadataToken="${METADATA_TOKEN}" \
  > "tap-values-full.yml"
```

We recommend you inspect the generated file:

```sh
cat tap-values-full.yml
```

### Install Updated Profile

We then update the TAP installation via the same command.

```sh
export TAP_INSTALL_NAMESPACE=tap-install
export TAP_VERSION=1.5.0
```

```sh
tanzu package install tap \
  -p tap.tanzu.vmware.com \
  -v $TAP_VERSION \
  --values-file tap-values-full.yml \
  -n ${TAP_INSTALL_NAMESPACE}
```

### View Security Scan Data in TAP GUI

Go to your TAP GUI, and either open a Scan step in a Supply Chain, or open the **Security Analysis** screen (shield with magnifying glass icon).

You should now see the scan result details.

## Create TLS App Cert Secret Separately

Using the SecretGen's SecretImport and SecretExport capability solves the missing `app-tls-cert` temporarily.

For a permanent solution, we need to do the following:

* copy the secret to another namespace
* update the Grype app config to use this new secret
* generate the new values file
* update our TAP install

### Copy the Certificate Secret

Let's copy the secret into a file.

```sh
kubectl get secret -n metadata-store app-tls-cert \
  -o yaml > metadata-store-app-tls-cert.yaml
```

Remove any values we can't use:

```sh
yq e -i '.metadata = {"name": "metadata-store-app-tls-cert"}' metadata-store-app-tls-cert.yaml
```

And then apply it to our target namespace:

```sh
kubectl apply -f metadata-store-app-tls-cert.yaml \
  -n ${TAP_DEVELOPER_NAMESPACE}
```

### Update TAP Profile

The Grype config to change, is the following:

```yaml
grype:
  namespace: dev
  metadataStore:
    caSecret:
      name: metadata-store-app-tls-cert
```


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

supply_chain: testing_scanning
ootb_supply_chain_testing_scanning:
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
    proxy:
      /metadata-store:
        target: https://metadata-store-app.metadata-store:8443/api/v1
        changeOrigin: true
        secure: false
        headers:
          Authorization: #@ "Bearer "+dv.metadataToken
          X-Custom-Source: project-star
    backend:
      reading:
        allow:
          - host: #@ dv.gitServer
    integrations:
      gitea:
        - host: #@ dv.gitServer
          username: #@ dv.gitUser
          password: #@ dv.gitPassword
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
  namespace: dev
  metadataStore:
    caSecret:
      name: metadata-store-app-tls-cert

ceip_policy_disclosed: true
excluded_packages:
  - eventing.tanzu.vmware.com #! not used, so removing to reduce memory/cpu footprint
  - tap-telemetry.tanzu.vmware.com #! not used, so removing to reduce memory/cpu footprint
```

### Generate updated profile

```sh
export GIT_SERVER=gitea.services.h2o-2-9349.h2o.vmware.com
export GIT_USER=gitea
export GIT_PASSWORD='VMware123!'
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
  -v buildRegistrySecret="$TAP_BUILD_REGISTRY_SECRET" \
  -v buildRepo="$BUILD_REGISTRY_REPO" \
  -v tbsRepo="$TBS_REPO" \
  -v domainName="$DOMAIN_NAME" \
  -v caCert="${CA_CERT}" \
  -v gitUser="${GIT_USER}" \
  -v gitPassword="${GIT_PASSWORD}" \
  -v gitServer="${GIT_SERVER}" \
  -v metadataToken="${METADATA_TOKEN}" \
  > "tap-values-full.yml"
```

We recommend you inspect the generated file:

```sh
cat tap-values-full.yml
```

### Install Updated Profile

We then update the TAP installation via the same command.

```sh
export TAP_INSTALL_NAMESPACE=tap-install
export TAP_VERSION=1.5.0
```

```sh
tanzu package install tap \
  -p tap.tanzu.vmware.com \
  -v $TAP_VERSION \
  --values-file tap-values-full.yml \
  -n ${TAP_INSTALL_NAMESPACE}
```

```yaml title="metadata-tls-secret-import-and-export.yaml"
---
apiVersion: secretgen.carvel.dev/v1alpha1
kind: SecretExport
metadata:
  name: metadata-store-app-tls-cert
  namespace: metadata-store-secrets
spec:
  toNamespace: dev
---
apiVersion: secretgen.carvel.dev/v1alpha1
kind: SecretImport
metadata:
  name: metadata-store-app-tls-cert
  namespace: dev
spec:
  fromNamespace: metadata-store-secrets
```

```sh
kubectl apply -f metadata-tls-secret-import-and-export.yaml
```

## References

[^1]: [Grype](https://github.com/anchore/grype)
[^2]: [Joostvdg's Blog - How relocate Grype CVE Database with Minio](https://joostvdg.github.io/tanzu/grype-airgapped/#relocate-grype-database)
[^3]: [TAP 1.5 - Testing and Scanning Supply Chain](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/scc-ootb-supply-chain-testing-scanning.html)
[^4]: [TAP 1.5 - Triage and Remidate CVEs for Supply Chain](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/scst-scan-triaging-and-remediating-cves.html)
[^5]: [TAP 1.5 - Tekton Pipeline Example for Testing and Scanning Supply Chain](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/getting-started-add-test-and-security.html#tekton-pipeline-config-example-3)
[^6]: [TAP 1.5 - Triage Security Vulnerabilities](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/getting-started-add-test-and-security.html#query-for-vulnerabilities-8)
[^7]: [TAP 1.5 - Connect TAP GUI to Metadata Store](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/tap-gui-plugins-scc-tap-gui.html?hWord=N4IghgNiBcIBoFoDCBXAzgFwPYFsEGUsUAnAYwFMQBfIA)
[^8]: [TAP 1.5 - Retrieve Metadata Store Read Token](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/scst-store-retrieve-access-tokens.html)
[^9]: [TAP 1.5 - Create Metadata Store Write Token](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/scst-store-create-service-account.html#rw-serv-accts)