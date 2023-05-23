---
hide:
- navigation
- toc
---

# TAP Workshop

Welcome.

## Lab Exercises

1. [TAP Install - Full Profile & OOTB Basic Supply Chain](/tap-workshops/install/basic)
1. [Hello World++ Workload](/tap-workshops/apps/hello-world/)
1. [Create Custom Supply Chain](/tap-workshops/supply-chain/custom/)
1. [Upgrade OOTB Basic Supply Chain to OOTB Testing & Scanning](/tap-workshops/supply-chain/basic-to-test-scan/)
1. [Leverage External Services for your Workload](/tap-workshops/apps/external-service/)

There are more guided workshop materials out there, for example the Tanzu Solution Engineer workshops, [visit the E2E Demo Portal](https://portal.end2end.link/).

## DockerHub Proxy

We have a DockerHub Proxy repository setup in the local Harbor instance.

This Proxy repository is logged in, and thus helps you prevent DockerHub's Rate Limits.

The project in Harbor is `dockerhub`, or `harbor.services.h2o-2-9349.h2o.vmware.com/dockerhub/` in full.

So instead of `alpine`, you would use `harbor.services.h2o-2-9349.h2o.vmware.com/dockerhub/library/alpine`

```sh
docker pull harbor.services.h2o-2-9349.h2o.vmware.com/dockerhub/library/alpine
```

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
