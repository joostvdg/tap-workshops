---
title: Workload from Self-hosted Git
description: Create, onboard, and update a TAP Workload from a self-hosted Git Repository
author: joostvdg
tags: [tap, kubernetes, spring, java]
---

In this workshop, we explore the creation, updating, and onboarding of a TAP Workload from a self-hosted Git server.

## Requirements

Before you can proceed with this workshop, ensure you have met the following requirements:

* Kubernetes cluster with TAP installed
* With one of the following TAP Profiles
  * Full or Iterate (recommended), or a combination of View, Build & Run (and then manually copy the **Deliverable**)
* Self-hosted Git server with TLS
   * Ideally with the CA certificate on hand
* SSH Key usable for the self-hosted Git server

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

## Steps

* go to TAP GUI -> Create | use VSCode Extention
    * https://tap-gui.view.h2o-2-9349.h2o.vmware.com/create
    * Tanzu Java Web UI
* create project in Gitea
    * make the project public (should be the default)
* Push project to Gitea
* create secret for Gitea with Credentials & CA Cert

* create workload
* verify application runs
* update TAP GUI config
    * to trust Gitea
* register application in TAP GUI
* view app resources in App Live View

The workshop consists of the following steps:

* Generate Application: to create the application source code to use for our Workload
* Import the Application's source code into Gitea
* Onboard the Application into TAP
* View the Application's Supply Chain in TAP GUI
* Register the Appliction in TAP GUI
* View Application's live resources in App Live View (part of TAP)

## Generate Project

Go to the TAP GUI, either your own or the [shared environment TAP GUI](https://tap-gui.view.h2o-2-9349.h2o.vmware.com).

Then go the to Create page, or the `+` icon (with a circle) in the left hand menu.

Select `Tanzu Java Web App`, by clicking on the `CHOOSE` button next it, or [go directly to the wizard screen](https://tap-gui.view.h2o-2-9349.h2o.vmware.com/create/templates/tanzu-java-web-app).

* **Name**: give it a unique name, at least for our Gitea instance
* **Prefix**: you can leave this to `dev.local`
* **Use Spring Boot 3.0**: as the name says, if you use this, you have to use `Java 17`, which you set in the **Workload** definition
    * either in the Workload manifest, or add `--build-env BP_JVM_VERSION=17` to the `tanzu apps workload create` command
* **Java Version**: either leave it on Java 11 or set it to Java 17 (recommended)

Click `NEXT`.

Review the settings, for example:

```sh
Project Name: tap-demo-04
Repository Prefix: dev.local
Update Boot 3: Yes
Java Version: 17
Include Build Tool Wrapper: Yes
```

And then click `GENERATE ACCELERATOR`.

Wait a few seconds, and the click `DOWNLOAD ZIP FILE`.

```sh
export APP_NAME=
export LAB=
```

Then SCP the Zip to your Lab:

```sh
scp ~/Downloads/${APP_NAME}.zip ubuntu@${LAB}.h2o-2-9349.h2o.vmware.com:/home/ubuntu
```

Test the Zip:

```sh
unzip -t ${APP_NAME}.zip
```

And then unpack it:

```sh
unzip ${APP_NAME}.zip
```

We should now have the application's source ready to import into Gitea.

```sh
tree $APP_NAME
```

Which should yield:

```sh
tap-demo-04
├── LICENSE
├── README.md
├── Tiltfile
├── accelerator-info.yaml
├── accelerator-log.md
├── catalog
│   └── catalog-info.yaml
├── config
│   └── workload.yaml
├── mvnw
├── mvnw.cmd
├── pom.xml
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── example
    │   │           └── springboot
    │   │               ├── Application.java
    │   │               └── HelloController.java
    │   └── resources
    │       └── application.yml
    └── test
        └── java
            └── com
                └── example
                    └── springboot
                        └── HelloControllerTest.java
```

!!! Tip "Use IDE Plugins"

    TAP also has an IDE plugin for VSCode and Jetbrain's IDE's.
    When this plugin is connected to a TAP GUI, you can create a new Accelerator based Application directly from your IDE.

    TODO: add link

## Import into Gitea

Before we can push our newly created Project to Gitea, we need a repository in Gitea.

### Create Gitea Repository

You can login into Gitea and create a new Repository that way, or use the REST API:

```sh
export APP_NAME=
export LAB=
```

!!! Warning "Replace App Name bin the command"
    Make sure you replace `REPLACE_WITH_APP_NAME` with your App Name.

```sh
curl -k -X POST "https://gitea.services.h2o-2-9349.h2o.vmware.com/api/v1/user/repos" \
    -u ${LAB}:'12345678' \
    -H "content-type: application/json" \
    --data '{"auto_init": false,"default_branch": "main","name": "REPLACE_WITH_APP_NAME","private": true}'
```

### Push to Gitea

Enter the directory of the application:

```sh
cd $APP_NAME
```

And then run the following Git commands, to push the branch to Gitea.

```sh
git init
git add .
git commit -m "first commit"
git remote add origin "git@gitssh.h2o-2-9349.h2o.vmware.com:${LAB}/${APP_NAME}.git"
git branch -m main
git push -u origin main
```

The SSH should be configured for you, so it should push directly.

## Onboard Application Into TAP

Your application is now in Gitea, great!

As you might have noticed, we made the repository private.
While not required in this case, it is expected in customer environments that Repositories require credendentials.

So before we can onboard our application via a **Workload** definition, we need to create a credential so [FluxCD]https://fluxcd.io/flux/components/source/gitrepositories/)[^1] can checkout the code.

We have two things to look at:

1. do we want a Per Repository credential (e.g., per Workload) or per TAP Profile (not recommended)
1. do we want to use a HTTPS credential (username, password, cert) or SSH Key credential

### Create Credential

```sh
export TAP_DEVELOPER_NAMESPACE=dev
export GITEA_SSH_URL=gitssh.h2o-2-9349.h2o.vmware.com
export LAB=
export LAB_GITEA_PASSWORD=
```

We will use a Credential per Repository, so we will include it in our Workload definition later.

First, we must decide on using a HTTPS credential or SSH.
Both will work, with the HTTPS type we need to include the CA, with the SSK Key we need to include the Known Hosts.

=== "HTTPS Credential"
    ```sh
    kubectl create secret generic https-credentials \
      --namespace $TAP_DEVELOPER_NAMESPACE \
      --from-file caFile=ca.crt \
      --from-literal username="${LAB}" \
      --from-literal password="${LAB_GITEA_PASSWORD}"
    ```
=== "SSH Credential"
    ```sh
    ssh-keyscan $GITEA_SSH_URL > gitea-known-hosts.txt
    ```

    ```yaml title="ssh-secret.ytt.yaml"
    #@ load("@ytt:data", "data")
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: #@ data.values.secretName
      annotations:
        tekton.dev/git-0: #@ data.values.server
    type: kubernetes.io/ssh-auth
    stringData:
      ssh-privatekey: #@ data.values.sshPushKey
      identity: #@ data.values.sshPullKey
      identity.pub: #@ data.values.sshPullId
      known_hosts: #@ data.values.knownHosts
    ```

    ```sh
    export GIT_SSH_SECRET_KEY="ssh-credentials"
    export GIT_SSH_PUSH_KEY=$(cat ~/.ssh/id_ed25519)
    export GIT_SSH_PULL_KEY=$(cat ~/.ssh/id_ed25519)
    export GIT_SSH_PULL_ID=$(cat ~/.ssh/id_ed25519.pub)
    export GIT_SSH_KNOWN_HOSTS=$(cat gitea-known-hosts.txt)
    ```

    ```sh
    ytt -f ssh-secret.ytt.yaml \
      -v secretName="$GIT_SSH_SECRET_KEY" \
      -v server="$GITEA_SSH_URL" \
      -v sshPushKey="$GIT_SSH_PUSH_KEY" \
      -v sshPullKey="$GIT_SSH_PULL_KEY" \
      -v sshPullId="$GIT_SSH_PULL_ID" \
      -v knownHosts="$GIT_SSH_KNOWN_HOSTS" \
      > "ssh-secret.yaml"
    ```

    ```sh
    kubectl apply -f ssh-secret.yaml \
      --namespace ${TAP_DEVELOPER_NAMESPACE}
    ```

You should now have a usable credential, and can create the Workload.

### Create Workload Via Tanzu CLI

```sh
export TAP_DEVELOPER_NAMESPACE=dev
export JAVA_VERSION=17
export APP_NAME=
```

!!! Warning
    Make sure you set the `JAVA_VERSION` to the version you selected when you created the application!

As the repository URL and the Credential we used depends on the type we created earlier, there are different commands to set them.

=== "HTTPS Credential"
    ```sh
    export APP_REPO="https://gitea.services.h2o-2-9349.h2o.vmware.com/${LAB}/${APP_NAME}.git"
    export APP_REPO_SECRET=https-credentials
    ```
=== "SSH Credential"
    Also note, that FluxCD doesn't support the SSH URL that Gitea generates, so we use a `/` after the SSH URL (instead of the traditional `:`).
    ```sh
    export APP_REPO="ssh://git@${GITEA_SSH_URL}/${LAB}/${APP_NAME}.git"
    export APP_REPO_SECRET="${GIT_SSH_SECRET_KEY}"
    ```

We can now create the Workload:

!!! Note
    Note that the parameter is called `gitops_ssh_secret`.

    The name implies its for SSH Keys only, but that is not true.
    The secret will be used by FluxCD[^1] to checkout the repository.

```sh
tanzu apps workload create ${APP_NAME} \
  --namespace ${TAP_DEVELOPER_NAMESPACE} \
  --git-repo ${APP_REPO} \
  --git-branch main \
  --type web \
  --label app.kubernetes.io/part-of=${APP_NAME} \
  --annotation autoscaling.knative.dev/minScale=1 \
  --param gitops_ssh_secret=${APP_REPO_SECRET} \
  --build-env BP_JVM_VERSION=${JAVA_VERSION} \
  --yes
```

### Create Workload Via Manifest

The application we generated contains a Workload definition!

Alas, we need to add the `gitops_ssh_secret` parameter before we can apply it.

Open up the Workload definition:

```sh
vim config/workload.yaml
```

!!! Warning "Replace URL Placeholder"

    Replace the URL placeholder values, with your HTTPS or SSH URLs.

Add a parameter for the Secret: `gitops_ssh_secret`:

```sh
  - name: gitops_ssh_secret
    value: ssh-credentials
```

??? Example "Full Example"
    This a full example of a `workload.yaml` after the changes.

    ```yaml title="config/workload.yaml"
    apiVersion: carto.run/v1alpha1
    kind: Workload
    metadata:
      name: YOUR_APP
      labels:
        apps.tanzu.vmware.com/workload-type: web
        apps.tanzu.vmware.com/has-tests: "true"
        apps.tanzu.vmware.com/auto-configure-actuators: "true"
        app.kubernetes.io/part-of: YOUR_APP
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
          url: ssh://git@gitssh.h2o-2-9349.h2o.vmware.com/YOUR_LAB/YOUR_APP.git
          ref:
            branch: main
    ```

```sh
kubectl apply -f config/workload.yaml \
    -n ${TAP_DEVELOPER_NAMESPACE}
```

### Verify Workload

Verify the Workload exists and is valid.

```sh
kubectl get workload -n ${TAP_DEVELOPER_NAMESPACE}
```

Verify if FluxCD can checkout your application's source from Gitea.

```sh
kubectl get GitRepository -n ${TAP_DEVELOPER_NAMESPACE}
```

Verify if your Workload is targeting a valid Supply Chain.

```sh
tanzu apps workload get ${APP_NAME} -n ${TAP_DEVELOPER_NAMESPACE}
```

Tail the logs:

```sh
tanzu apps workload tail ${APP_NAME} -n ${TAP_DEVELOPER_NAMESPACE} --timestamp --since 1h
```

And last but not least, once the Supply Chain completes, retrieve its HTTPProxy:

```sh
kubectl get httpproxy \
  -n ${TAP_DEVELOPER_NAMESPACE} \
  -l contour.networking.knative.dev/parent=${APP_NAME}
```

It will have four proxy entries, select the one with the external DNS name and save it:

```sh
export URL=
```

Now we can `curl` the application:

```sh
curl -lk "https://${URL}"
```

Which should respond with:

```sh
Greetings from Spring Boot + Tanzu!
```

## Update Workload

Let's test the update of the Workload.

If all went well, the curl command returned `Greetings from Spring Boot + Tanzu!`.

Let's change this.

The place this line is written, is in the `HelloController`.

The file resides here: `src/main/java/com/example/springboot/HelloController.java`.

If we change it here, we must also change it in the test: `src/test/java/com/example/springboot/HelloControllerTest.java`.

Make sure all three occurances are the same.
Either manually edit the files with **vim**, or use the **sed** command below:

```sh
export ORIGINAL="Greetings from Spring Boot + Tanzu!"
export NEW="Greetings from Barcelona!"
```

Feel free to replace `NEW` with your own text.

```sh
sed -i -e \
  "s/$ORIGINAL/$NEW/g" \
  src/main/java/com/example/springboot/HelloController.java

sed -i -e \
  "s/$ORIGINAL/$NEW/g" \
  src/test/java/com/example/springboot/HelloControllerTest.java
```

Then we add, commit, and push the changes:

```sh
git add src/
git commit -m "change test"
```

Verify the change happend:

```sh
curl -lk "https://${URL}"
```

Which should now return (or your message):

```sh
Greetings from Barcelona!
```

## View App Supply Chain

Open the TAP GUI and visit the Supply Chain screen.

You should be able to see your application listed there.

If we want to see more of the application's resources, such as its (runtime) Deployment, we need to register the application.

## Register Application in TAP GUI

Before we can register our application in the TAP GUI, we need to make TAP GUI trust our Gitea server.

We do this by updating the TAP Profile intallation values.

We will do the following steps:

* Update TAP GUI Config / TAP install
* Register Application in TAP GUI
* View Workload in App Live View (part of TAP GUI)

### Update Profile

We do this by updating the TAP GUI configuration in our profile.
Either change the `ytt` template, or edit the `tap-values-full.yml` file directly.


=== "Directly"
    Add the `backend` and `integrations` segment to the `tap_gui.app_config` property, so that it looks like below.
    ```yaml
    tap_gui:
      app_config:
        backend:
          reading:
            allow:
              - host: "gitea.services.h2o-2-9349.h2o.vmware.com"
        integrations:
          gitea:
            - host: gitea.services.h2o-2-9349.h2o.vmware.com
              username: gitea
              password: 'VMware123!'
    ```
=== "Via YTT Template"
    ```yaml title="full-profile.ytt.yaml" hl_lines="45"
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

    ceip_policy_disclosed: true
    excluded_packages:
      - scanning.apps.tanzu.vmware.com #! disabled for now, enabled when we upgrade OOTB Basic to Test & Scanning
      - grype.scanning.apps.tanzu.vmware.com #! disabled for now, enabled when we upgrade OOTB Basic to Test & Scanning
      - policy.apps.tanzu.vmware.com #! disabled for now, enabled when we upgrade OOTB Basic to Test & Scanning
      - eventing.tanzu.vmware.com #! not used, so removing to reduce memory/cpu footprint
      - tap-telemetry.tanzu.vmware.com.0.5.0-build #! not used, so removing to reduce memory/cpu footprint
    ```

    ```sh
    export TAP_BUILD_REGISTRY_SECRET=registry-credentials
    export BUILD_REGISTRY_REPO=tap-apps
    export TBS_REPO=buildservice/tbs-full-deps
    export CA_CERT=$(cat ca.crt)
    export BUILD_REGISTRY=
    export DOMAIN_NAME=
    ```

    ```sh
    export GIT_SERVER=gitea.services.h2o-2-9349.h2o.vmware.com
    export GIT_USER=gitea
    export GIT_PASSWORD='VMware123!'
    ```

    And then we run YTT to generate our Profile configuration file.

    ```sh
    ytt -f full-profile.ytt.yaml \
      -v buildRegistry="$BUILD_REGISTRY" \
      -v buildRegistrySecret="$BUILD_REGISTRY_SECRET" \
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

### Register App in TAP GUI

To get a ***live*** view[^2] of our application, we have to register it in the Software Catalog[^3].

We do so by going to the `Home` screen, via the house icon in the left hand menu.

We can then add the application, by clicking the `REGISTER ENTITY` button on the right.

Here we add a link to the `catalog-info.yaml` of the application.

Our generated application has such a file generated, in the `catalog` folder.

The URL to use, is the _raw_ URL, which should look like this: `https://gitea.services.h2o-2-9349.h2o.vmware.com/lab02/tap-demo-04/raw/branch/main/catalog/catalog-info.yaml`

If all went well, you should now see your application on the TAP GUI home screen.

## References

[^1]: [FluxCD Git Repository authentication](https://fluxcd.io/flux/components/source/gitrepositories/)
[^2]: [Tanzu Application Platform - App Live View](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.3/tap/GUID-app-live-view-about-app-live-view.html)
[^3]: [Backstage - Software Catalog](https://backstage.io/docs/features/software-catalog/)
