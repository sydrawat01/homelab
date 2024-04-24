# GitOps: ArgoCD

From the ArgoCD website:

_"Application definitions, configurations, and environments should be declarative and version controlled. Application deployment and lifecycle management should be automated, auditable, and easy to understand."_

ArgoCD is a GitOps tool that looks into git repositories and looks for `.yaml` files or `helm charts`, and deploys them to the Kubernetes cluster. We can then easily deploy our apps to one or many clusters. The application config files (`yaml` files) which are present in a git repository allow for code history and auditing.

ArgoCD has a nice UI, it tracks if your app actually deployed, not just fires the deployment and forgets about it. It even gives you easy access to pod logs from UI.

Argo CD runs in your Kubernetes cluster, that is very much aligned with my philosophy, to keep everything contained in Kubernetes. This also in effect helps with security, you do not have to give kubectl config file to anybody.

## Setup

Let's install ArgoCD:

### Create a namespace

```bash
kubectl create namespace argocd
```

### Apply the ArgoCD manifest file

Manifest file location: [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### External IP to ArgoCD service

Since we are using MetalLB, let's give the ArgoCD service an external IP from our address pool. We can do this by patching the existing `argocd-server` service by patching it:

```bash
kubectl patch service argocd-server -n argocd --patch '{ "spec": { "type": "LoadBalancer", "loadBalancerIP": "10.0.0.208" } }'
```

If you are not comfortable with patching manifest files like above, feel free to edit the service and apply the changes by adding the `loadBalancerIP` and `type: LoadBalancer` to the `argocd-server` service.

We can now access the ArgoCD UI at `http://10.0.0.208`, which will automatically switch to port `443` instead of `80`.

### Access the dashboard

- The username is `admin`.
- The password is randomly generated and stored as a secret on the cluster. In order to access it, use the below command:

  ```bash
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
  ```

You should be able to access the dashboard now.

The first thing you should do is update the password through the UI and avoid using the random generated password as well as delete the secret on the cluster.

### Install the CLI Tool

We need to install the ArgoCD CLI tool if we need additional functionality for our GitOps workflow:

```bash
brew install argocd
```

Once installed, login using the following command:

```bash
# argocd.sydrawat.me OR IP address of service where argocd is exposed
argocd login argocd.sydrawat.me
# will prompt for password
```

### Create a custom project

You can use the `default` project to deploy your applications and cluster manifest files, or you can create separate projects with special permissions to specific namespaces. Read more about this in the official docs [here](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/).

The following command creates a new project `myproject` which can deploy applications to namespace `mynamespace` of cluster `https://kubernetes.default.svc`. The permitted Git source repository is set to `https://github.com/argoproj/argocd-example-apps.git` repository.

```bash
argocd proj create myproject -d https://kubernetes.default.svc,mynamespace -s https://github.com/argoproj/argocd-example-apps.git
```

> \[!NOTE]\
> In order to successfully deploy applications to ArgoCD the GitOps way, make sure you have proper permissions for the projects you create.
> Apply proper RBAC rules to your projects, as well as the `Cluster Resource Allow List`, which are Cluster-scoped K8s API Groups and Kinds which are permitted to be deployed on your homelab cluster.

In my setup I have the following projects:

- `default`: This project comes pre-bundled with ArgoCD project. We will not deploy any applications in this project.
- `storage`: This is where we will deploy the `longhorn` Helm chart along with it's custom configuration.
- `network`: This is where we will deploy the `MetalLB` Helm chart along with it's custom configurations.
- `service-monitor`: This is where we will deploy all manifest files for the service monitors which include the `kube-state-metrics`, `kubelet` monitor, `longhorn` monitor, `node-exporter` and `traefik` monitor.
- `monitoring`: This project deploys the `Prometheus` and Grafana manifest files to run these applications on the cluster.

### Creating applications the GitOps way

We can deploy our ArgoCD applications using manifest files. The files present in [`config/argocd`](../config/argocd/) are the manifest files which we can apply to the Kubernetes cluster using the command below:

```bash
kubectl apply -n argocd -f config/argocd/
```

Once everything is deployed in the `argocd` namespace, we will have to sync our applications.

To check the ArgoCD application status using the `kubectl` command:

```bash
kubectl get application -n argocd
```

The above command should give you an output similar to mine:

```text
NAME                 SYNC STATUS   HEALTH STATUS
traefik-monitor      Synced        Healthy
metallb-config       Synced        Healthy
kubelet-monitor      Synced        Healthy
longhorn-monitor     Synced        Healthy
kube-state-metrics   Synced        Healthy
grafana              Synced        Healthy
longhorn-config      Synced        Healthy
prometheus           Synced        Healthy
node-exporter        Synced        Healthy
metallb              OutOfSync     Healthy
longhorn             Synced        Healthy
```

Here's a screenshot of all my applications on the ArgoCD UI:

![argocd-dashboard](../assets/argocd/argo-dashboard.png)

[Next: Grafana Dashboards](./12_grafana_dashboards.md) | [Next: Cloudflare(WIP)](./14_cloudflare.md)
