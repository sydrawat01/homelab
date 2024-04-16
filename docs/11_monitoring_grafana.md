# Monitoring: Grafana

Let’s deploy `Grafana` to read data from our `Prometheus` instance.

The manifest file for Grafana are present at [`config/monitoring/grafana/`](../config/monitoring/grafana/).

## Manifest files

Let's have a look at the manifest files required to get Grafana up and running:

### [grafana-pvc.yaml](../config/monitoring/grafana/grafana-pvc.yaml)

```yaml
# ../config/monitoring/grafana/grafana-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-grafana-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 10Gi
```

### [grafana-deployment.yaml](../config/monitoring/grafana/grafana-deployment.yaml)

```yaml
# ../config/monitoring/grafana/grafana-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: grafana
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - env: []
          image: grafana/grafana:latest
          name: grafana
          ports:
            - containerPort: 3000
              name: http
          readinessProbe:
            httpGet:
              path: /api/health
              port: http
          resources:
            limits:
              cpu: 200m
              memory: 200Mi
            requests:
              cpu: 100m
              memory: 100Mi
          volumeMounts:
            - mountPath: /var/lib/grafana
              name: grafana-storage
              readOnly: false
      nodeSelector:
        node-type: worker
      securityContext:
        fsGroup: 65534
        runAsNonRoot: true
        runAsUser: 65534
      serviceAccountName: grafana
      volumes:
        - name: grafana-storage
          persistentVolumeClaim:
            claimName: longhorn-grafana-pvc
```

### [grafana-serviceAccount.yaml](../config/monitoring/grafana/grafana-serviceAccount.yaml)

```yaml
# ../config/monitoring/grafana/grafana-serviceAccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: grafana
  namespace: monitoring
```

### [grafana-service.yaml](../config/monitoring/grafana/grafana-service.yaml)

```yaml
# ../config/monitoring/grafana/grafana-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  selector:
    app: grafana
  type: LoadBalancer
  ports:
    - name: http
      port: 3000
      targetPort: http
  loadBalancerIP: 10.0.0.206
```

Apply the `Grafana` resources:

```bash
kubectl apply -f config/monitoring/grafana
```

## UI

The grafana UI should now be accessible via [`http://10.0.0.206:3000`](http://10.0.0.206:3000). The default username and password to login to the dashboard are `admin`:`admin`.

[Prev: Monitoring: Prometheus](./10_monitoring_prometheus.md) | [Next: Grafana Dashboards](./12_grafana_dashboards.md)
