# Install Kubernetes

The `master` node is our primary node. We are going to install K3s version of Kubernetes, which is a lightweight enough for our single board computers to handle.

> \[!IMPORTANT]\
> Install Helm on your workstation as a pre-requisite since we will be using it to install a lot of Helm charts.
> Install Helm using brew: `brew install helm`

## k3s master setup

> \[!NOTE]\
> If installing k3s, make sure to disable the default `servicelb` helm chart in the installation script, since we will be using `metallb` instead.
> In case you missed out on disabling the `servicelb` helm chart, you can follow the instructions mentioned [here](https://github.com/k3s-io/k3s/issues/1160).

```bash
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644 --disable servicelb --node-taint CriticalAddonsOnly=true:NoExecute --bind-address 10.0.0.98 --disable local-storage --token some_random_password
```

Some other flags to consider:

- `--node-taint CriticalAddonsOnly=true:NoExecute`: This will taint the master node to schedule critical workloads only.
- `--bind-address 192.168.0.10`: This binds the k3s master node to a specific IP address.
- `--disable-cloud-controller`: In case you do not need the cloud-controller. I have it installed in case I need it in the future.
- `--disable local-storage`: We will use `longhorn` storage provider instead.
- `--token some_random_password`:Token that will help us connect other nodes to the master node. I use the default k3s script generated token here.

Give it some time to install and configure the services. Once done, check out the Kubernetes nodes using the following command on the master node (ssh):

```bash
kubectl get nodes
```

```txt
root@master:~# kubectl get nodes
NAME     STATUS   ROLES                  AGE     VERSION
master   Ready    control-plane,master   7d1h    v1.28.7+k3s1
```

If you want to access the Kubernetes cluster from your workstation's terminal, we need to copy the contexts of the `kubeconfig` file from `/etc/rancher/k3s/k3s.yaml` on the `master node`, to our local machine at `~/.kbe.config`, and update the server IP address for the `kube-apiserver` to match the IP address of our master node:

```yaml
  server: 10.0.0.98:6443
```

Once done, use the same command as above to connect to the cluster and check the nodes on the Kubernetes cluster.

## Worker setup

We need to join some worker nodes now, in my case it's just one node, `node01`, but it might be more in your case. We are going to execute the join command on each node with Ansible.

> \[!NOTE]\
> We are doing this on our local workstation. Also note, if you are not using the `--token some_random_password` flag while installing k3s, the randomly generated token will automatically be generated for you, and will be present here: `/var/lib/rancher/k3s/server/node-token`.

```bash
ansible workers -b -m shell -a "curl -sfL https://get.k3s.io | K3S_URL=https://10.0.0.98:6443 K3S_TOKEN=some_random_password sh -"
```

In the end, it should look something like this (expect more nodes in case you have more worker nodes):

```txt
root@master:~# kubectl get nodes
NAME     STATUS   ROLES                  AGE    VERSION
node01   Ready    worker                 7d     v1.28.7+k3s1
master   Ready    control-plane,master   7d1h   v1.28.7+k3s1
```

## Setting roles/labels

We can `tag` our cluster nodes to give them labels, so that it's easier to identify and schedule workloads on our nodes.

> \[!IMPORTANT]\
> k3s by default allow pods to run on the control plane, which can be OK, but in production it would not. However, in our case, we already tagged the master node when we installed the K3s. I still want to control a bit more where workload is deployed, and it's also good to know how it's done. So, We will be using labels to tell pods/deployment where to run.

Let's add the `worker` role to the worker nodes and also a custom label `node-type:worker` to these nodes for workload deployment purposes.

```bash
# add same labels for all your worker nodes:
kubectl label nodes node01 kubernetes.io/role=worker
kubectl label nodes node01 node-type=worker
```

Lastly, add the kubeconfig file path into `/etc/environment` (so that Helm and other programs know where the kubernetes configuration is):

```bash
# on every node:
echo "KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> /etc/environment
# using ansible:
ansible homelab -b -m lineinfile -a "path='/etc/environment' line='KUBECONFIG=/etc/rancher/k3s/k3s.yaml'"
```

[Prev: Configure Ansible](./03_ansible.md) | [Next: Network Configuration](./05_network.md)
