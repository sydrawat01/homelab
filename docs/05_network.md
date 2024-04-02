# Network setup

K3s comes pre-bundled with `traefik` and `servicelb`. However we did not install `servicelb` that came pre-configured with k3s, so we will install another `LoadBalancer` tool called `MetalLB`.

A LoadBalancer, in essence is able to give services(PODs) an external IP through which we can access our applications. Normally this is an external component, and a cloud provider automatically gives that to us, but since we are our own cloud provider here and are trying to keep everything in a single cluster, `MetalLB` is the answer to our problems.

Read more about `MetalLB` here: [https://metallb.universe.tf/](https://metallb.universe.tf/)

## Deployment

```bash
# First add metallb repository to our helm
helm repo add metallb https://metallb.github.io/metallb
# Check if it was found
helm search repo metallb
# Install metallb
helm upgrade --install metallb metallb/metallb --create-namespace \
  --namespace metallb-system --wait
```

This should install MetalLB on the cluster. Next, we need to configure `MetalLB`.

## Configuration

MetalLB needs to know what IP range to use for external IPs. This is done via a `Custom Resource`. You can find the CR in the [official documentation](https://metallb.universe.tf/configuration/). So lets create and apply a `Custom Resource` with the following content:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.0.0.200-10.0.0.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
    - default-pool
```

This file is also present [here](../config/metallb-system/00_metallb_config.yaml) for your reference.

Let's apply this configuration to our Kubernetes cluster:

```bash
cd config
kubectl apply -f metallb-system/
```

To check if everything is deployed and is OK, we must have as many `speaker-xxxx` PODs in the `metallb-system` namespace as the number of nodes in our cluster, since they run one per node. In my case, I have two. It may be more for you if you have more number of nodes.

With this configuration applied, we can now see our services being assigned an external IP address from the IP address range we specified in the configuration for `MetalLB` above.

[Prev: Install k3s](./04_k3s_install.md) | [Next: Storage configuration](./06_storage.md)
