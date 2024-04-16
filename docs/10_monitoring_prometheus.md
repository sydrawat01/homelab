# Monitoring: Prometheus

We are going to deploy a single instance of Prometheus. Normally, you would/should deploy multiple instances spread throughout the cluster.
For example, one instance dedicated to monitor just Kubernetes API, the next dedicated to monitor nodes, and so on. As with many things in the Kubernetes world, there is no specific way things should look, so to save resources, we will deploy just one.

All manifest files for the Prometheus instance is present at [`config/monitoring/prometheus/`](../config/monitoring/prometheus/).

## Manifest files

Let's have a closer looks at the manifest files for the prometheus instance.

### [prometheus.yaml](../config/monitoring/prometheus/prometheus.yaml)

```yaml
# ../config/monitoring/prometheus/prometheus.yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus-persistent
  namespace: monitoring
spec:
  replicas: 1
  retention: 7d
  resources:
    requests:
      memory: 400Mi
  nodeSelector:
    node-type: worker
  securityContext:
    fsGroup: 2000
    runAsNonRoot: true
    runAsUser: 1000
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchExpressions:
      - key: name
        operator: In
        values:
          - longhorn-prometheus-servicemonitor
          - kube-state-metrics
          - node-exporter
          - kubelet
          - traefik
  storage:
    volumeClaimTemplate:
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: longhorn
        resources:
          requests:
            storage: 20Gi
```

Things too look for here:

- `retention`: How long to keep the prometheus data.

- `nodeSelector`

  ```yaml
  nodeSelector:
    node-type: worker
  ```

  > \[!NOTE]\
  > If you followed my setup, I set some tags for worker nodes and control planes. Above, I just say to prefer nodes with the tag: worker. I use this just because I didn't want to tax the control nodes more. If it does not matter to you where the Prometheus is running, remove these two lines.

- `serviceMonitorSelector`

  ```yaml
    serviceMonitorSelector:
      matchExpressions:
      - key: name
        operator: In
        values:
        - longhorn-prometheus-servicemonitor
        - kube-state-metrics
        - node-exporter
        - kubelet
        - traefik
  ```

  Above are the Service Monitors we created. This part tells Prometheus to go and take data from them. If you add more or less, edit this part.

- `storage`: We will just tell it what provisioner to use, and how many `GB` to provision. Our Prometheus Operator will take care of mounting and assigning the storage for persistent data. Make sure that longhorn is the default storage provisioner.

### [prometheus-service-ext.yaml](../config/monitoring/prometheus/prometheus-service-ext.yaml)

```yaml
# ../config/monitoring/prometheus/prometheus-service-ext.yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus-external
  namespace: monitoring
spec:
  selector:
    prometheus: prometheus-persistent
  type: LoadBalancer
  ports:
    - name: web
      protocol: TCP
      port: 9090
      targetPort: web
  loadBalancerIP: 10.0.0.205
```

### [prometheus-service-local.yaml](../config/monitoring/prometheus/prometheus-service-local.yaml)

```yaml
# ../config/monitoring/prometheus/prometheus-service-local.yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
spec:
  ports:
    - name: web
      port: 9090
      targetPort: web
  selector:
    prometheus: prometheus-persistent
  sessionAffinity: ClientIP
```

### [prometheus-serviceaccount.yaml](../config/monitoring/prometheus/prometheus-serviceaccount.yaml)

```yaml
# ../config/monitoring/prometheus/prometheus-serviceacount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
```

### [prometheus-rbac-role-binding.yaml](../config/monitoring/prometheus/prometheus-rbac-role-binding.yaml)

```yaml
# ../config/monitoring/prometheus/prometheus-rbac-role-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: monitoring
```

### [prometheus-rbac-clusterrole.yaml](../config/monitoring/prometheus/prometheus-rbac-clusterrole.yaml)

```yaml
# ../config/monitoring/prometheus/prometheus-rbac-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
  namespace: monitoring
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - services
      - endpoints
      - pods
      - nodes/metrics
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources:
      - configmaps
    verbs: ["get"]
  - nonResourceURLs: ["/metrics"]
    verbs: ["get"]
```

Apply the prometheus instance to the cluster:

```bash
kubectl apply -f config/monitoring/prometheus/
```

## UI

The prometheus UI should now be accessible via [`http://10.0.0.205:9090`](http://10.0.0.205:9090).

[Prev: Monitoring: Service Monitors](./09_monitoring_service_monitors.md) | [Next: Monitoring: Grafana](./11_monitoring_grafana.md)
