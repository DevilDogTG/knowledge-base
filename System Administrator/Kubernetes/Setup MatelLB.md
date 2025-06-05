# Setup MatelLB for load balancer

[MetalLB](https://metallb.io/) is a load-balancer implementation for bare metal Kubernetes clusters, using standard routing protocols.

## Installation by manifest

To install MetalLB, apply the manifest:

```sh
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
```

Check for latest verion on [MatelLB](https://metallb.io/installation/)

## Allocate IP Pool for load balancing

to allowcate IP need to add some configure to Kubernetes cluster

```yaml
# matllb.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  namespace: metallb-system
  name: local-pool
spec:
  addresses:
    - 192.168.99.100 - 192.168.99.110
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: advert
  namespace: metallb-system
```

Apply configure to kube

```sh
kubectl apply -f <your config name>.yaml
```
