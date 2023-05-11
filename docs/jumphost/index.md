# Jumphost Required Tools

* needs:
  * 4 cpu
  * 8gb ram
  * 100gb disk
* download Ubuntu Image from Cloud Images
  * https://cloud-images.ubuntu.com/jammy/20230401/
* create VM in user-workload network from ubuntu image
* paste personal SSH pub key
* add it to additional network
  * set up IP
  * `ip a `
  * `sudo netplan set ethernets.ens224.dhcp4=true`
  * `sudo netplan apply`
  * `ip a `
* remove default route to alt network
  * `sudo ip r delete default via 172.16.50.1 dev ens224`
* https://matteo-magni.github.io/tanzugym/tkgm/bastion/
* https://cloud-images.ubuntu.com/jammy/20230401/
* tools:
  * go lang
  * python 3?
  * net-tools
  * git
  * yq
  * jq
  * govc
  * kubectl
  * kubectx
  * kubens
  * Tanzu CLI
    * TKG Plugins
    * TAP Plugins
  * Carvel toolsuite
  * curl (& httpie)
  * Docker
  * Kind
  * cfssl
  * minio client
  * Cosign
  * age
  * sops
* optional tools:
    * java
    * maven


## Kind Cluster

```sh
kind create cluster --image kindest/node:v1.24.12@sha256:1e12918b8bc3d4253bc08f640a231bb0d3b2c5a9b28aa3f2ca1aee93e1e8db16 --name tkgm
```

```sh
tanzu management-cluster create h2o-9349-01  --file tkgm-management-kubevip.yaml  --use-existing-bootstrap-cluster tkgm -v 6
```



## Age

* https://github.com/FiloSottile/age#installation

```sh
sudo apt install age
```

## Sops

* https://github.com/mozilla/sops/releases

```sh
http --download https://github.com/mozilla/sops/releases/download/v3.7.3/sops_3.7.3_amd64.deb
sudo apt install ./sops_3.7.3_amd64.deb
```

## Git

```sh
sudo apt install net-tools git
```

```sh
LOCALBIN="/home/ubuntu/.local/bin"
mkdir -p $LOCALBIN
export PATH=$PATH:$LOCALBIN
```

## MinIO

* MinIO: https://min.io/docs/minio/linux/reference/minio-mc.html

```sh
curl https://dl.min.io/client/mc/release/linux-amd64/mc \
  --create-dirs \
  -o $HOME/minio-binaries/mc

chmod +x $HOME/minio-binaries/mc
export PATH=$PATH:$HOME/minio-binaries/

mc --help
```

## Kubectl

```sh
curl -sSfLO "https://dl.k8s.io/release/$(curl -sSfL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
install kubectl ${LOCALBIN}/kubectl
```

## Carvel

```sh
sudo chmod -R 777 /usr/local/bin/
wget -O- https://carvel.dev/install.sh | bash
```

## VCC

```sh
export VCC_USER=""
export VCC_PASS='
```

```sh
# list all the available products
vcc get products

# list all the available subproducts belonging to the vmware_tanzu_kubernetes_grid product
vcc get subproducts -p vmware_tanzu_kubernetes_grid

# list all the versions for the subproduct tkg
vcc get versions -p vmware_tanzu_kubernetes_grid -s tkg

# list all the files available for version 2.1.0
# at this point vcc does the actual login
vcc get files -p vmware_tanzu_kubernetes_grid -s tkg -v 2.1.1

# download the tanzu cli
vcc download -p vmware_tanzu_kubernetes_grid -s tkg -v 2.1.1 -f 'tanzu-cli-bundle-linux-amd64.tar.gz' --accepteula
```

```sh
tar xvzf vcc-downloads/tanzu-cli-bundle-linux-amd64.tar.gz
install cli/core/v0.28.1/tanzu-core-linux_amd64 ${LOCALBIN}/tanzu
```

## Kind

```sh
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.18.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

## Tanzu CLI

* Download Tanzu CLI
  * login to Tanzu Network: https://network.tanzu.vmware.com/
  * download CLI from TAP Download Page: https://network.tanzu.vmware.com/products/tanzu-application-platform

```sh
scp tanzu-framework-linux-amd64-v0.28.1.1.tar ubuntu@10.220.13.174:/home/ubuntu/
```

```sh
mkdir -p $HOME/tanzu
tar -xvf tanzu-framework-linux-amd64-v0.28.1.1.tar -C $HOME/tanzu
```

```sh
cd $HOME/tanzu
export VERSION=v0.28.1.1.
# sudo install cli/core/$VERSION/tanzu-core-linux_amd64 /usr/local/bin/tanzu
tanzu plugin install --local cli all
```

```sh
tanzu plugin list
```

This should now list both TKG and TAP plugins:

```sh
Standalone Plugins
  NAME                DESCRIPTION                                                        TARGET      DISCOVERY  VERSION        STATUS
  accelerator         Manage accelerators in a Kubernetes cluster                                               v1.5.0         installed
  apps                Applications on Kubernetes                                                                v0.11.1        installed
  external-secrets    interacts with external-secrets.io resources                                              v0.1.0-beta.4  installed
  insight             post & query image, package, source, and vulnerability data                               v1.5.0         installed
  isolated-cluster    isolated-cluster operations                                                    default    v0.28.1        installed
  login               Login to the platform                                                          default    v0.28.1        installed
  pinniped-auth       Pinniped authentication operations (usually not directly invoked)              default    v0.28.1        installed
  services            Commands for working with service instances, classes and claims                           v0.6.0         installed
  management-cluster  Kubernetes management-cluster operations                           kubernetes  default    v0.28.1        installed
  package             Tanzu package management                                           kubernetes  default    v0.28.1        installed
  secret              Tanzu secret management                                            kubernetes  default    v0.28.1        installed
  telemetry           Configure cluster-wide telemetry settings                          kubernetes  default    v0.28.1        installed

Plugins from Context:  tkgm-m
  NAME                DESCRIPTION                           TARGET      VERSION  STATUS
  cluster             Kubernetes cluster operations         kubernetes  v0.28.1  installed
  feature             Operate on features and featuregates  kubernetes  v0.28.1  installed
  kubernetes-release  Kubernetes release op erations         kubernetes  v0.28.1  installed
```

## Cosign

* https://docs.sigstore.dev/cosign/installation/

```sh
wget "https://github.com/sigstore/cosign/releases/download/v2.0.0/cosign-linux-amd64"
mv cosign-linux-amd64 /usr/local/bin/cosign
chmod +x /usr/local/bin/cosign
```
