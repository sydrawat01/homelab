# Traefik Ingress

Traefik comes pre-installed with the K3s distribution. We do not need to install Traefik through Helm charts externally. We will use the pre-installed version of Traefik to create an Kubernetes Ingress resource.

## Working with Traefik

### Expose Longhorn UI as ClusterIP

Let's expose the `Longhorn` frontend service using the Traefik Ingress. This service should already have an `External IP` assigned to it by the `MetalLB` LoadBalancer.

```bash
kubectl get svc -n longhorn-system | grep 'frontend'
```

```output
longhorn-frontend             LoadBalancer   10.43.228.143   10.0.0.201    80:30364/TCP   9d
```

This looks good, but in order to access the service, we need to change the service type from `LoadBalancer` to `ClusterIP`. Let's describe the `longhorn-frontend` service and get some details about it.

```bash
kubectl describe svc longhorn-frontend -n longhorn-system
```

```output
Name:                     longhorn-frontend
Namespace:                longhorn-system
Labels:                   app=longhorn-ui
                          app.kubernetes.io/instance=longhorn
                          app.kubernetes.io/managed-by=Helm
                          app.kubernetes.io/name=longhorn
                          app.kubernetes.io/version=v1.6.0
                          helm.sh/chart=longhorn-1.6.0
Annotations:              meta.helm.sh/release-name: longhorn
                          meta.helm.sh/release-namespace: longhorn-system
                          metallb.universe.tf/ip-allocated-from-pool: default-pool
Selector:                 app=longhorn-ui
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.43.228.143
IPs:                      10.43.228.143
IP:                       10.0.0.201
LoadBalancer Ingress:     10.0.0.201
Port:                     http  80/TCP
TargetPort:               http/TCP
NodePort:                 http  30364/TCP
Endpoints:                10.42.0.68:8000,10.42.1.48:8000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

From this data, all we need is the `Selector: app=longhorn-ui` and the `Endpoints: 10.42.0.68:8000,10.42.1.48:8000`. Based on these two things, we can create a new service to expose the Longhorn UI.

Let's create the service:

```yaml
# config/longhorn/00_longhorn-int-svc.yaml
---
kind: Service
apiVersion: v1
metadata:
  name: longhorn-int-svc
  namespace: longhorn-system
spec:
  type: ClusterIP
  selector:
    app: longhorn-ui
  ports:
  - name: http
    port: 8000
    protocol: TCP
    targetPort: 8000
```

This is the minimal configuration we need to expose the service as a `ClusterIP`. Let's create the service:

```bash
kubectl apply -f config/longhorn/00_longhorn-int-svc.yaml
```

With this, we have a `ClusterIP` service which is targeting the pods with the label `app=longhorn-ui` and expose it on the port `8000`. The port is named `http` for easier access and identification.

### Create Traefik Ingress resource

Now in order to expose the `longhorn-int-svc` using Traefik, we need to create an `Ingress` object definition. This object will tell Traefik how, what and where to expose our services.

```yaml
# config/longhorn/01_longhorn-ingress-traefik.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-ing-traefik
  namespace: longhorn-system
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: "longhorn.homelab.local"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: longhorn-int-svc
            port:
              number: 8000
```

```bash
kubectl apply -f config/longhorn/01_longhorn-ingress-traefik.yaml
```

> \[!NOTE]\
> If you are not using `Cloudflare DNS` (more on this [here](./14_cloudflare.md)), make sure you edit your `/etc/hosts` file to include you `Traefik` loadbalancer external IP along with `longhorn.homelab.local` to access the Longhorn dashboard at the URL `http://longhorn.homelab.local`.
>
> An example entry would be: `10.0.0.200 longhorn.homelab.local`.

There seems to be a new way of working with Traefik, which includes the use of routers, middlewares and services. I haven't had the time to explore this, but I'm planning on testing it out on my homelab, which I will most definitely include here for reference.

[Prev: Storage configuration](./06_storage.md) | [Next: Monitoring: Prometheus Operator](./08_monitoring_prometheus_operator.md)
