---
title: TAP GUI Auth
description: Configure TAP GUI Auth via GitLab
author: joostvdg
tags: [tap, kubernetes, GitLab, TAP, LDAP]
---

* https://backstage.io/docs/auth/gitlab/provider
* https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/tap-gui-auth.html

* setup auth in GitLab
* update TAP View's values
* update TAP View install

In GitLab

* new application
* name: `tap-gui`
* redirect URI: `https://tap-gui.view.h2o-2-9349.h2o.vmware.com/api/auth/gitlab/handler/frame`

```yaml
tap_gui:
  app_config:
    auth:
      environment: development
      providers:
        gitlab:
          development:
            clientId: 272e7d7008d5ed4d43124b25d11ff288b5dcea638e065d93167430b67d2712fe
            clientSecret: 0fa2a065896b7214a07217eb29ed8ce21f402f3aed8a11eac1e6fedb2d3479d3
            audience: https://gitlab.services.h2o-2-9349.h2o.vmware.com/
```
