# Kubernetes Setup Guide

Thisn guide refer <https://kubernetes.io/docs/setup/production-environment/> to setup production-like environment in home lab

## Preparation

before start setup kubernetes need to preparing VMs for example use 2 VMs for Master - Worker node example spec below

- **Master Node**: debain 12, 4 vCPUs, 2GB RAM, IP 192.168.0.1/24
- **Worker Node**: debian 12, 2 vCPUs, 2GB RAM, IP 192.168.0.2/24

need to fixed ip on both hosts

### OS Configuration

Kubernetes need preparation some configuration before setup, this required on all nodes

- Disabled SWAP
- Enable `ip_forward`
- [Optional] Endable `br_netfilter`

#### Disabled SWAP

If your system using SWAP your can disable by

```shell
sudo swapoff -a
```

but above command will disabled untill next restart, to disabled to remove `swap` from `/etc/fstab` is required

```shell
sudo nano /etc/fstab
# Comment out or delete swap mountin file
```

#### Enable `ip_forward`

By default will be disable, easiest way to enable it just use command

```shell
sudo sysctl -w net.ipv4.ip_forward=1
```

tell `sysctl` to enabled when system start need to edit file `/etc/sysctl.conf`

```shell
sudo nano /etc/sysctl.conf
# Adding or uncomment line
## net.ipv4.ip_forward = 1
```

Or run following command

```shell
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

#### [Optional] Endable `br_netfilter`

Like `ip_forward` but we need extra step to enable bridge modules

```shell
sudo modprobe br_netfilter
echo br_netfilter | sudo tee /etc/modules-load.d/kubernetes.conf
```

this will enable every time os restart by default, next add configuration to `sysctl`

```shell
echo 'net.bridge.bridge-nf-call-iptables=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Getting Started

Installation is flexible and allow your to choose your favourite component but required to run kubernetes cluster. 2 thing need to choose is **Container Runtime** and **CNI** in this guide will start setup step below

- Install **Container Runtime**: containerd.io
- Install Kubernetes with deployment tools
- Setup kubernetes cluster
- Install **CNI**: Calico
- Join worker node to cluster

### Container Runtime

Before install, you need to set up `apt` repository. `containerd.io` packages are distributed by Docker, we need to and docker source for `apt`

```shell
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl runc
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" |  sudo tee /etc/apt/sources.list.d/docker.list
```

Then install `containerd.io` package

```shell
sudo apt-get update
sudo apt-get install containerd.io
```

After installed need to configuring the `systemd` cgroup driver

```shell
sudo nano /etc/containerd/config.toml
```

Find and update `SystemdCgroup` to `true`

```conf
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
  ...
```

Overridin the sandbox (puase) image

```conf
[plugins."io.containerd.grpc.v1.cri"]
  ...
  sandbox_image = "registry.k8s.io/pause:3.10"
  ...
```

Restart `conatinerd` to take effect

```shell
sudo systemctl restart containerd.service
```

### Install Kubernetes with deployment tools

This install will install Kubenetes v1.31, start with add `apt` source

```shell
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update `apt` package index, install tools and pin their version

```shell
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Setup kubernetes cluster

Before intilizing cluster need to check all node can communicate with each others, then start setup cluster. For me specified `pod-network-cidr` and `service-cidr` to avoid my network setup

```shell
sudo kubeadm init --apiserver-advertise-address=0.0.0.0 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/16
```

This will take some time, after done setup `kubectl` for user

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

verify by using:

```shell
kubectl get nodes
```

if output show nodes list, everything is ok

### Install network plugin **Calico**

after initialed cluster your can see running pods in system by

```shell
kubectl get pods -A
```

You will see `coredns` will stuck in `Pending` state. you need to install pod network add-on before `coredns` network start setup

to install `calico` Download the Calico networking manifest for the Kubernetes API datastore

```shell
curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/calico.yaml -O
```

if you are using pod CIDR `192.168.0.0/16` you can skip to the next step, for me changing pod CIDR to `10.244.0.0/16` need to update `CALICO_IPV4POOL_CIDR` before install

```shell
nano calico.yaml
# Update variable CALICO_IPV4POOL_CIDR to 10.244.0.0/16
```

Apply the manifest:

```shell
kubectl apply -f calico.yaml
```

Wait and see `coredns` pod state to `Running`

```shell
kubectl get pods -A
```

### Join worker node to cluster

after initialed cluster, system will print join command to output, if your need join command using:

```shell
kubeadm token create --print-join-command
```

on worker node need to join running command to join cluster

```shell
sudo kubeadm join [MASTER_NODE_IP]:6443 --token [TOKEN] --discovery-token-ca-cert-hash [HASH]
```

then you can check new nodes has join 

```shell
kubectl get nodes -o wide
```

New node take sometime to starting some system pods after that they will `Ready` state

Checking pods status by using

```shell
kubectl get pods -A
```

This will display all pods in all namespace, all pods should be `Running` state
