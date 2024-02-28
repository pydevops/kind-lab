
- [Why do we even need it?](#why-do-we-even-need-it)
- [Pre-requisites](#pre-requisites)
  - [Hardware](#hardware)
  - [Software](#software)
- [Install OrbStack](#install-orbstack)
- [Set the VM's memory limit](#set-the-vms-memory-limit)
- [Create a linux VM](#create-a-linux-vm)
  - [Troubleshooting](#troubleshooting)
- [Post set up](#post-set-up)
  - [SSH](#ssh)
  - [Set up bashrc](#set-up-bashrc)
  - [Validate Metal LB](#validate-metal-lb)
    - [arm64](#arm64)
    - [amd64](#amd64)
  - [Test MetalLB](#test-metallb)
  - [Cleanup test](#cleanup-test)
    - [arm64](#arm64-1)
    - [amd64](#amd64-1)
  - [Install Istio](#install-istio)
  - [Install bookinfo demo](#install-bookinfo-demo)
  - [Clean up VM](#clean-up-vm)
- [Customize](#customize)


TL;DR: How to set up a Kubernetes cluster managed by [KIND](https://kind.sigs.k8s.io/) with cloud-init, which runs on a linux VM managed by [OrbStack](https://orbstack.dev/) on MacOS. Istio can be installed. 

# Why do we even need it? 

* An ephemeral cluster to troubleshoot a kubernetes or Istio issue.
* An ephemeral cluster to test out new features of kubernetes or Istio release. 
* A lab cluster to learn Kubernetes and Istio, which can be discarded anytime.
* A useful way to demo a proof of concept based on any features leveraging Kubernetes and Istio. 

# Pre-requisites

## Hardware
My set up:
* 13-inch 2020 Apple MacBook Pro with M1 Chip: 8GB RAM, 512GB SSD.
* 15-inch 2015 Apple MacBook Pro with Intel Chip: 16GB RAM, 256GB SSD.

## Software 

* [OrbStack](https://orbstack.dev/) is lighting fast which makes all the difference here. 
* [KIND](https://kind.sigs.k8s.io/) is perfect for ephemeral cluster.
* [MetalLB](https://metallb.universe.tf/) can be configured with [L2 mode](https://metallb.universe.tf/configuration/_advanced_l2_configuration/). 
* [Istio](https://istio.io/) is the most popular service mesh.


# Install OrbStack

* https://docs.orbstack.dev/install

```
brew install orbstack
```

# Set the VM's memory limit
* https://docs.orbstack.dev/settings#memory-limit

The default limit can be bumped to 4G to run istio workload such as bookinfo. 

```
orb config set memory_mib 4096
orb restart mantic
orb config show
```

# Create a linux VM
The following script will create a linux VM with Ubuntu 23.10 (codenamed mantic)
```
$ ./create-vm.sh

$ orb list
NAME    STATE    DISTRO  VERSION  ARCH
----    -----    ------  -------  ----
mantic  running  ubuntu  mantic   arm64

```
OrbStack is used to create a Ubuntu(code named mantic) with the following software installed. The `create-vm.sh` uses cloud-init to install and configure the following softwares: 
* docker engine
* kubectl and kind CLI 
* A kubernetes cluster called *basic* from kind CLI. 
* Metal LB as load balancer, configured with layer2. Please note [a range of IP addresses for load balancer](cloud-init-kind.yaml#L40) can be configured as needed. 

## Troubleshooting
* Check vm boot logs by running `orb logs mantic`. 

# Post set up

## SSH

* https://docs.orbstack.dev/machines/ssh
By default Orbstack add `orb` to your SSH config for easy access. In our case, the following ssh command will drop us into a SSH session with the VM created. 
```
ssh orb
```

## Set up bashrc


Install bash-completion
```
$ sudo apt install bash-completion
```

Set up KUBECONFIGß
```
$ cp /etc/kind/kubeconfig-basic . && export KUBECONFIG=$HOME/kubeconfig-basic
```


Append the following to $HOME/.bashrc
```
cat <<EOF >> $HOME/.bashrc
source <(kubectl completion bash)
alias k=kubectl
complete -F __start_kubectl k
alias kns="kubectl config set-context --current --namespace"
export KUBECONFIG=$HOME/kubeconfig-basic
EOF
```

Then refresh bash
```
$ . $HOME/.bashrc
```

run kubectl to verify the cluster
```
$ k get nodes
NAME                  STATUS   ROLES           AGE   VERSION
basic-control-plane   Ready    control-plane   11m   v1.29.2
basic-worker          Ready    <none>          10m   v1.29.2
basic-worker2         Ready    <none>          10m   v1.29.2
```

run kind to verify the cluster
```
$ sudo kind get clusters
basic
```

## Validate Metal LB

### arm64
```
kubectl apply -n default -f https://raw.githubusercontent.com/pydevops/kind-lab/main/metallb/usage.yaml
```
### amd64

```
kubectl apply -n default -f https://kind.sigs.k8s.io/examples/loadbalancer/usage.yaml
```

## Test MetalLB

```
LB_IP=$(kubectl -n default get svc/foo-service -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')
for _ in {1..10}; do
  curl ${LB_IP}:5678
done
```

## Cleanup test

### arm64
```
kubectl delete -n default -f https://raw.githubusercontent.com/pydevops/kind-lab/main/metallb/usage.yaml
```
### amd64

```
kubectl delete -n default -f https://kind.sigs.k8s.io/examples/loadbalancer/usage.yaml
```

## Install Istio
* https://istio.io/latest/docs/setup/getting-started/ with demo profile.

## Install bookinfo demo

* https://istio.io/latest/docs/examples/bookinfo/


## Clean up VM
When it is time to stop and delete VM 
```
orbctl stop mantic
orbctl delete mantic
```

# Customize
Feel free to customize anything used in the repo. Enjoy with your local cluster. 