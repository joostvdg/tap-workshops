---
title: TAP Debug Supply Chains
description: Tips & Tricks for debugging TAP Supply Chains
author: joostvdg
tags: [tap, kubernetes, cartographer, tekton, debug]
---


## Debug Workload

### Source

```sh
kubectl get gitrepo -A
```

### Workload

```sh
kubectl get workload -A
```

```sh
tanzu apps workload get smoke-app --namespace dev
```

#### Image Failed

```sh
📡 Overview
   name:        smoke-app
   type:        web
   namespace:   dev

💾 Source
   type:     git
   url:      https://github.com/sample-accelerators/tanzu-java-web-app.git
   branch:   main

📦 Supply Chain
   name:   source-to-url

   NAME               READY   HEALTHY   UPDATED   RESOURCE
   source-provider    True    True      9m49s     gitrepositories.source.toolkit.fluxcd.io/smoke-app
   image-provider     False   False     9m40s     images.kpack.io/smoke-app
   config-provider    False   Unknown   9m57s     not found
   app-config         False   Unknown   9m56s     not found
   service-bindings   False   Unknown   9m56s     not found
   api-descriptors    False   Unknown   9m56s     not found
   config-writer      False   Unknown   9m56s     not found

🚚 Delivery
   name:   delivery-basic

   NAME              READY   HEALTHY   UPDATED   RESOURCE
   source-provider   False   False     9m45s     imagerepositories.source.apps.tanzu.vmware.com/smoke-app-delivery
   deployer          False   Unknown   9m49s     not found

💬 Messages
   Workload [HealthyConditionRule]:   condition status: False, message: Unable to find builder default.
   Deliverable [HealthyConditionRule]:   Unable to resolve image with tag "harbor.services.h2o-2-9349.h2o.vmware.com/tap-apps/smoke-app-dev-bundle:ed5561a7-5bfc-45fb-be33-90b2aeb0a9a1" to a digest: HEAD https://harbor.services.h2o-2-9349.h2o.vmware.com/v2/tap-apps/smoke-app-dev-bundle/manifests/ed5561a7-5bfc-45fb-be33-90b2aeb0a9a1: unexpected status code 404 Not Found (HEAD responses have no body, use GET for details)
```

```sh
kubectl get ClusterImageTemplate kpack-template -o yaml | yq
```

```sh
kubectl describe img -n dev smoke-app
```

```sh
Status:
  Conditions:
    Last Transition Time:  2023-05-05T12:07:52Z
    Message:               Unable to find builder default.
    Reason:                BuilderNotFound
    Status:                False
    Type:                  Ready
  Observed Generation:     1
Events:                    <none>
```

This means we're missing Tanzu Buildservice components!

