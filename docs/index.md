---
hide:
- navigation
- toc
---

Welcome.

For the Tanzu Solution Engineer workshops, [visit the E2E Demo Portal](https://portal.end2end.link/).

## TODO

* ask people for public keys -> so we can upload them to the jump hosts
* generate google sheet with pre-defined groups
* 

## DockerHub Proxy

* In Harbor -> `harbor.services.h2o-2-9349.h2o.vmware.com/dockerhub/`

So instead of `alpine` -> `harbor.services.h2o-2-9349.h2o.vmware.com/dockerhub/library/alpine`

```sh
docker pull harbor.services.h2o-2-9349.h2o.vmware.com/dockerhub/library/alpine
```

## Labs

1. [Full & OOTB Basic](/tap-workshops/install/basic)
1. [Hello World++ Workload](/tap-workshops/apps/hello-world.md)
1. [Custom Supply Chain](/tap-workshops/supply-chain/custom.md)
1. [Upgrade OOTB Basic to OOTB Test & Scanning](/tap-workshops/supply-chain/basic-to-test-scan.md)
1. [External Services](/tap-workshops/apps/external-service.md)

## Workshops

* Installations
    * [Full & OOTB Basic](/tap-workshops/install/basic)
    * [GitOps](/tap-workshops/install/gitops.md)
* Workloads
    * [Hello World](/tap-workshops/apps/hello-world.md)
    * [External Services](/tap-workshops/apps/external-service.md)
* Supply Chains
    * [Custom Supply Chain](/tap-workshops/supply-chain/custom.md)
    * [Upgrade OOTB Basic to OOTB Test & Scanning](/tap-workshops/supply-chain/basic-to-test-scan.md)
* TAP GUI
    * View Supply Chain
    * Register App
    * View App Live View
    * View App API

### Ideas

* Create Application From Accelerator
    * TAP GUI
    * VSCode Extension
* Workload in local Git server
    * Register Workload in TAP GUI -> App Live View
* Workload that uses external services (Bitnami Services + Crossplane)
    * Register API
    * API Gateway View
* Install with GitOps
* Install Multicluster
* Install Scanning & Testing Supply Chain
* Create Custom Supply Chain
* 

## Outcome Steps

### Basic

I can install Cluster Essentials
I setup wildcard certs to be configured in TAP
I can configure tap-values.yaml for a Full profile installation of TAP (with OOTB Basic)
I manually installed TAP
I'm able to manually setup a single developer namespace
I'm able to do basic troubleshooting of a TAP Installation
I can configure a public Git provider for TAP GUI in TAP Values
I can configure DNS for TAP Endpoints
I understand Kapp Controller and Secret Gen Controller in the context of TAP
I know how to configure access to private container registries in TAP Values
I have installed TAP on TKGx
I have configured custom CA's for TAP

To add:
I know how to configure an integration to a private Git provider in TAP GUI
I know how to configure access to private SCM repositories for TAP GUI.

### Hello-world

I'm able to use an accelerator to generate a Web TAP Workload
I applied a TAP Web Workload and register it within TAP GUI
I'm able to do basic troubleshooting of a TAP Workload
I've configured App Live View in TAP GUI for a TAP Web Workload
I've configured Tanzu Build Service and it works for a Spring Boot Application to produce an Image
I am able to view and verify a TAP Workload displays in properly in Supply Chain view of TAP GUI
I know how to configure an integration to a private Git provider in TAP GUI
I know how to configure access to private repositories (SCM and Container) for TAP Workloads
I have configured custom CA's for TAP

To add:
I am able to view and verify results for the Security Analysis within TAP GUI

### External Services

I am able to register API Documentation for a Workload in TAP GUI
I am able to stand up shared services and claim resources on a TAP Run Cluster
I can confgure API Auto Registration within a Supply Chain for a Workload
I can configure API Scoring and Validation within TAP GUI
I can configure API Portal with TAP
I can install Shared Services on a TAP Cluster (Services Toolkit)
I can claim a shared service on a TAP Cluster (Service Bindings)
I can use a postgres resource claim with sample Spring Boot Application Pet Clinic Accelerator
Expose Accerators Endpoint for use in Tanzu Accelerator Plugins

### Upgrade Basic To Test & Scanning Supply Chain

I performed an upgrade of TAP
I've able to configure and install the OOTB Testing and Scanning Supply Chain and verify a Workload
I am able to configure a Scan Policy for a Source and Image Scan
I am able to create a Tekton Task to execute Unit Tests within a Supply Chain for a Developer Namespace
I've customized the OOTB Supply Chains
I can create custom Tekton Tasks in Supply Chains

Maybe?
I have installed Grype in an air-gapped environment

### GitOps Install

I am able to manage a TAP installation with GitOPs

### Custom Supply Chain

I've written and implemented my own Custom Supply Chain on a Cluster

### Build & View & Run

I understand and have implemented the TAP Multi Cluster Reference Architecture
Configure TAP GUI to read resources from multiple clusters


### Create Accelerator

I can create a custom Accelerator and register it in TAP GUI

### Maybe

I understand TAP GUI Catalog System
I can do a Live Debug for a Workload
I can do a Live Update for a Workload
I can install the Dev Tools for IDEs (VSCode and IntelliJ)
I am able to configure Image Signing
I am able to use Namespace Provisioner to manage Tenant Onboarding
I have implemented SRE best practices for a TAP installation (Monitoring/Alerting/etc.)

### TODO

I'm able to install Learning Center and use the sample workshop.
I understand the different Workload Types available in TAP
I can update TAP values and reconcile changes on the cluster.
I have relocated TAP Images to my own container registry and understand the imgpkg utility.
I can use the VMware compatibility matrix
I'm able to gather requirements for a TAP Installation (Use the TAP Questionaire)
I've able to configure and install the OOTB Testing Supply Chain and verify a Workload
I've able to configure and install the OOTB Testing and Scanning Supply Chain and verify a Workload
I'm able to configure different Scanners (Trivy, Snyk) in the OOTB Supply Chains
I am able to configure a Scan Policy for a Source and Image Scan
I am able to create a Tekton Task to execute Unit Tests within a Supply Chain for a Developer Namespace
I know how to setup Authorization and Authentication of RBAC for a TAP Cluster
I know how to configure an Auth Provider for TAP GUI access
I know how to configure an external Postgres database for TAP GUI
I'm able to configure an external Postgres Database for Metadata Store
I'm able to configure TAP GUI to read from Metadata Store.
I've configured and generated Tech Docs for TAP GUI
I am able to view and verify results for the Security Analysis within TAP GUI
I've installed Tilt CLI
I can force reconcilation of TAP changes
I understand GitOps
I can configure Tanzu Telemetry
I can configure App SSO and consume it within a sample app generated from an Accelerator from TAP GUI.
I can configure TAP Values to use External Secrets
Multi Language Support (.NET, Java, Node, etc.) of TAP
I can configure TAP GUI TLS
I can configure Metadata Store TLS
I can configure App Live View TLS
Configure Accelerators to read from a Gitops Repo
I have installed TAP on a Public Cloud Provider (EKS, AKS, GKE)
I understand the OOTB Convention Services provided with TAP
I understand how to customize Cloud Native Runtimes (Knative) domains
I can customize the look and feel of TAP GUI
I've implemented role based access for TAP GUI
I have integrated a Supply Chain with Jenkins
I can configure caching of artifacts for workload in a Supply Chain
I have implemented Pull Requests in a Supply Chain to manage delivery of a workload in a config repo
I have implemented automation to reconcile a Deliverable from a Build Cluster to a Run Cluster to apply workloads
I have configured custom accelerators to be reconciled with TAP GUI with GitOps
I've written my own Convention Service for a Supply Chain
I have configured custom namespace provisioned resources
I have installed TBS in an airgapped environment
I have installed Grype in an air-gapped environment
I can do Capacity Planning and Sizing for TAP
I've configured the TAS to TAP workload adaptor
I have migrated a TAS Workload to TAP
I have implemented an active/active architecture with TAP
I have built a custom Multi-cluster architecture for TAP (Customize the profiles)
I have rotated Certificates used by TAP
I have restored TAP from a Backup
I have implemented SRE best practices for a TAP installation (Monitoring/Alerting/etc.)
I have implemented Cert monitoring
I have written my own custom buildpack for TBS
I have migrated datastores used by TAP
Upgrade TBS outside of TAP upgrade Cycle
I have integrated Tanzu Service Mesh with TAP
I have implemented GitHub Actions for TBS
I can practice a failover of TAP
I have installed TAP on OpenShift

### Developer Goals

* I'm able to deploy an app generated from an OOTB Accelerator
    * Outcome: I'm able to understand the basics of TAP as a developer
* I can use Inner Loop with an app generated from an OOTB Accelerator on TAP 
    * Outcome: I'm able to iterate my development/debugging of an app as a Developer running on an Iterate Cluster
* I can use Outer Loop with an app generated from an OOTB Accelerator on TAP
    * Outcome:  I'm able to understand the path to production for an app running on TAP"
* I can deploy an application that connects to Services provided on TAP. I can write custom Accelerators.


## Links

### Examples

* [Tanzu Labs US - Custom Cartographer Supply Chains](https://github.com/x95castle1/custom-cartographer-supply-chain-examples)
* [TAP Application Accelerator samples](https://github.com/vmware-tanzu/application-accelerator-samples/)
* [VRabbi TAP GitOps](https://github.com/vrabbi-tap/tap-gitops/tree/master/clusters/tap-full-cls-01/cluster-config/config/operators)

### Example Applications

* [Where For Dinner - Main TAP demo application](https://github.com/vmware-tanzu/application-accelerator-samples/blob/main/where-for-dinner/doc/TAPDeployment.md)
* [Spring Cloud Stream](https://github.com/maliksalman/spring-cloud-stream-sample)
* [TAP - Open Telemetry For Applications demo](https://github.com/alexandreroman/tap-otel-demo)
* [Spring Cloud Demo](https://github.com/benwilcock/spring-cloud-demo-tap)

### Community Links

* [VRabbi - Whats New In 1.5](https://vrabbi.cloud/post/tap-1-5-whats-new/)
* [VRabbi - TAP 1.5 GitOps](https://vrabbi.cloud/post/tap-1-5-gitops-installation/)
* [VRabbi - Service Toolkit Dynamic Provisioning](https://vrabbi.cloud/post/tap-1-5-dynamic-service-provisioning-with-crossplane/)

### Other Links

* [Testcontainers with Tekton](https://www.testcontainers.org/supported_docker_environment/continuous_integration/tekton/)
* [Tanzu Developer - Getting Started With Testcontainers](https://tanzu.vmware.com/developer/guides/spring-testcontainers-gs/)
* [CNCF Security TAG - Software Supply Chain Best Practices](https://github.com/cncf/tag-security/blob/main/supply-chain-security/supply-chain-security-paper/CNCF_SSCP_v1.pdf)
* [Miro Board with US based TAP Engagements](https://miro.com/app/board/uXjVPO1yfag=/?moveToWidget=3458764535658257109&cot=14)
* [Cartographer Lifecyle docs](https://cartographer.sh/docs/v0.7.0/tutorials/lifecycle/) (useful for leveraging Tekton TaskRun)
* [What Is Supply Chain Choreography, and Why Should You Care?](https://tanzu.vmware.com/content/blog/what-is-supply-chain-choreography-and-why-should-you-care)
