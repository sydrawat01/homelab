# Monitoring: Service Monitors

Let's install a few service monitors that will provide data to our `Prometheus` data aggregator.

## Longhorn Service Monitor

The `Longhorn` data driver provides data to `Prometheus` natively. The manifest file is present in [`config/monitoring/longhorn/longhorn-service-monitor.yaml`](../config/monitoring/longhorn/longhorn-servicemonitor.yaml) file.

### [longhorn-servicemonitor.yaml](../config/monitoring/longhorn/longhorn-servicemonitor.yaml)

```yaml
# ../config/monitoring/longhorn/longhorn-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: longhorn-prometheus-servicemonitor
  namespace: monitoring
  labels:
    name: longhorn-prometheus-servicemonitor
spec:
  selector:
    matchLabels:
      app: longhorn-manager
  namespaceSelector:
    matchNames:
      - longhorn-system
  endpoints:
    - port: manager
```

In this manifest file, we are not talking to the Kubernetes API, rather it is the `prometheus-operator` and we instruct it to create a custom resource called `ServiceMonitor`. We are targeting the `longhorn-manager`so that it reports the metrics to the prometheus instance.

The `endpoints.port` is `manager` since the `longhorn-manager` `daemonset` deployed by the `Longhorn` Helm chart configures the `containerPort` for the `longhorn-manager` `daemonset` pods to be `TCP 9500`,and also a name: `manager`. We can replace the `endpoints.port` with `9500` as well here, the service monitor will work the same.

Apply the longhorn service monitor:

```bash
kubectl apply -f config/monitoring/longhorn/
```

## Node-exporter

This is the daemon set we will deploy to collect metrics from individual cluster nodes, underlying HW, etc.
The manifest files for `node-exporter` are available in [`config/monitoring/node-exporter`](../config/monitoring/node-exporter/).

### [00_clusterrolebinding.yaml](../config/monitoring/node-exporter/00_clusterrolebinding.yaml)

```yaml
# ../config/monitoring/node-exporter/00_clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-exporter
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: node-exporter
subjects:
  - kind: ServiceAccount
    name: node-exporter
    namespace: monitoring
```

### [01_clusterrole.yaml](../config/monitoring/node-exporter/01_clusterrole.yaml)

```yaml
# ../config/monitoring/node-exporter/01_clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-exporter
rules:
  - apiGroups:
      - authentication.k8s.io
    resources:
      - tokenreviews
    verbs:
      - create
  - apiGroups:
      - authorization.k8s.io
    resources:
      - subjectaccessreviews
    verbs:
      - create
```

### [02_serviceaccount.yaml](../config/monitoring/node-exporter/02_serviceaccount.yaml)

```yaml
# ../config/monitoring/node-exporter/02_serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: node-exporter
  namespace: monitoring
```

### [03_service.yaml](../config/monitoring/node-exporter/03_service.yaml)

```yaml
# ../config/monitoring/node-exporter/03_service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: node-exporter
  name: node-exporter
  namespace: monitoring
spec:
  clusterIP: None
  ports:
    - name: https
      port: 9100
      targetPort: https
  selector:
    app.kubernetes.io/name: node-exporter
```

### [04_daemonset.yaml](../config/monitoring/node-exporter/04_daemonset.yaml)

```yaml
# ../config/monitoring/node-exporter/04_daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: node-exporter
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 1.1.0
    name: node-exporter
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app.kubernetes.io/component: exporter
      app.kubernetes.io/name: node-exporter
      app.kubernetes.io/part-of: kube-prometheus
  template:
    metadata:
      labels:
        app.kubernetes.io/component: exporter
        app.kubernetes.io/name: node-exporter
        app.kubernetes.io/part-of: kube-prometheus
        app.kubernetes.io/version: 1.1.0
    spec:
      containers:
        - args:
            - --web.listen-address=127.0.0.1:9100
            - --path.sysfs=/host/sys
            - --path.rootfs=/host/root
            - --no-collector.wifi
            # - --no-collector.hwmon
            - --collector.filesystem.ignored-mount-points=^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/pods/.+)($|/)
            - --collector.netclass.ignored-devices=^(veth.*)$
            - --collector.netdev.device-exclude=^(veth.*)$
          image: quay.io/prometheus/node-exporter:v1.7.0
          name: node-exporter
          resources:
            limits:
              cpu: 250m
              memory: 180Mi
            requests:
              cpu: 102m
              memory: 180Mi
          volumeMounts:
            - mountPath: /host/sys
              mountPropagation: HostToContainer
              name: sys
              readOnly: true
            - mountPath: /host/root
              mountPropagation: HostToContainer
              name: root
              readOnly: true
        - args:
            - --logtostderr
            - --secure-listen-address=[$(IP)]:9100
            - --tls-cipher-suites=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
            - --upstream=http://127.0.0.1:9100/
          env:
            - name: IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          image: quay.io/brancz/kube-rbac-proxy:v0.12.0
          name: kube-rbac-proxy
          ports:
            - containerPort: 9100
              hostPort: 9100
              name: https
          resources:
            limits:
              cpu: 20m
              memory: 40Mi
            requests:
              cpu: 10m
              memory: 20Mi
          securityContext:
            runAsGroup: 65532
            runAsNonRoot: true
            runAsUser: 65532
      hostNetwork: true
      hostPID: true
      nodeSelector:
        kubernetes.io/os: linux
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
      serviceAccountName: node-exporter
      tolerations:
        - operator: Exists
      volumes:
        - hostPath:
            path: /sys
          name: sys
        - hostPath:
            path: /
          name: root
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 10%
    type: RollingUpdate
```

### [05_service-monitor.yaml](../config/monitoring/node-exporter/05_service-monitor.yaml)

```yaml
# ../config/monitoring/node-exporter/05_service-monitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app.kubernetes.io/name: node-exporter
    name: node-exporter
  name: node-exporter
  namespace: monitoring
spec:
  endpoints:
    - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
      interval: 15s
      port: https
      relabelings:
        - action: replace
          regex: (.*)
          replacement: $1
          sourceLabels:
            - __meta_kubernetes_pod_node_name
          targetLabel: instance
      scheme: https
      tlsConfig:
        insecureSkipVerify: true
  jobLabel: app.kubernetes.io/name
  selector:
    matchLabels:
      app.kubernetes.io/name: node-exporter
```

Apply the node-exporter service monitor and other required resources:

```bash
kubectl apply -f config/monitoring/node-exporter/
```

This will create all permissions, and deploy the pod with the application Node Exporter, that will read metrics from Linux.

After doing so, you should see `node-exporter-xxxx` pods in the `monitoring` namespace.

## Kube State Metrics

This is a simple service that listens to the Kubernetes API, and generates metrics about the state of the objects.
Link to official GitHub: [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics).

The manifest files for `node-exporter` are available in [`config/monitoring/kube-state-metrics`](../config/monitoring/kube-state-metrics/).

### [00_kube-state-metrics-clusterRole.yaml](../config/monitoring/kube-state-metrics/00_kube-state-metrics-clusterRole.yaml)

```yaml
# ../config/monitoring/kube-state-metrics/00_kube-state-metrics-clusterRole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.4.2
  name: kube-state-metrics
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - secrets
      - nodes
      - pods
      - services
      - resourcequotas
      - replicationcontrollers
      - limitranges
      - persistentvolumeclaims
      - persistentvolumes
      - namespaces
      - endpoints
    verbs:
      - list
      - watch
  - apiGroups:
      - apps
    resources:
      - statefulsets
      - daemonsets
      - deployments
      - replicasets
    verbs:
      - list
      - watch
  - apiGroups:
      - batch
    resources:
      - cronjobs
      - jobs
    verbs:
      - list
      - watch
  - apiGroups:
      - autoscaling
    resources:
      - horizontalpodautoscalers
    verbs:
      - list
      - watch
  - apiGroups:
      - authentication.k8s.io
    resources:
      - tokenreviews
    verbs:
      - create
  - apiGroups:
      - authorization.k8s.io
    resources:
      - subjectaccessreviews
    verbs:
      - create
  - apiGroups:
      - policy
    resources:
      - poddisruptionbudgets
    verbs:
      - list
      - watch
  - apiGroups:
      - certificates.k8s.io
    resources:
      - certificatesigningrequests
    verbs:
      - list
      - watch
  - apiGroups:
      - storage.k8s.io
    resources:
      - storageclasses
      - volumeattachments
    verbs:
      - list
      - watch
  - apiGroups:
      - admissionregistration.k8s.io
    resources:
      - mutatingwebhookconfigurations
      - validatingwebhookconfigurations
    verbs:
      - list
      - watch
  - apiGroups:
      - networking.k8s.io
    resources:
      - networkpolicies
      - ingresses
    verbs:
      - list
      - watch
  - apiGroups:
      - coordination.k8s.io
    resources:
      - leases
    verbs:
      - list
      - watch
```

### [01_kube-state-metrics-clusterRoleBinding.yaml](../config/monitoring/kube-state-metrics/01_kube-state-metrics-clusterRoleBinding.yaml)

```yaml
# ../config/monitoring/kube-state-metrics/01_kube-state-metrics-clusterRoleBinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.4.2
  name: kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
  - kind: ServiceAccount
    name: kube-state-metrics
    namespace: monitoring
```

### [02_kube-state-metrics-serviceAccount.yaml](../config/monitoring/kube-state-metrics/02_kube-state-metrics-serviceAccount.yaml)

```yaml
# ../config/monitoring/kube-state-metrics/02_kube-state-metrics-serviceAccount.yaml
apiVersion: v1
automountServiceAccountToken: false
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.4.2
  name: kube-state-metrics
  namespace: monitoring
```

### [03_kube-state-metrics-service.yaml](../config/monitoring/kube-state-metrics/03_kube-state-metrics-service.yaml)

```yaml
# ../config/monitoring/kube-state-metrics/03_kube-state-metrics-service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.4.2
  name: kube-state-metrics
  namespace: monitoring
spec:
  clusterIP: None
  ports:
    - name: http-metrics
      port: 8080
      targetPort: http-metrics
    - name: telemetry
      port: 8081
      targetPort: telemetry
  selector:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/part-of: kube-prometheus
```

### [04_kube-state-metrics-deployment.yaml](../config/monitoring/kube-state-metrics/04_kube-state-metrics-deployment.yaml)

```yaml
# ../config/monitoring/kube-state-metrics/04_kube-state-metrics-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.4.2
  name: kube-state-metrics
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: exporter
      app.kubernetes.io/name: kube-state-metrics
      app.kubernetes.io/part-of: kube-prometheus
  template:
    metadata:
      labels:
        app.kubernetes.io/component: exporter
        app.kubernetes.io/name: kube-state-metrics
        app.kubernetes.io/part-of: kube-prometheus
        app.kubernetes.io/version: 2.4.2
    spec:
      automountServiceAccountToken: true
      containers:
        - image: k8s.gcr.io/kube-state-metrics/kube-state-metrics:v2.4.2
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            timeoutSeconds: 5
          name: kube-state-metrics
          ports:
            - containerPort: 8080
              name: http-metrics
            - containerPort: 8081
              name: telemetry
          resources:
            limits:
              cpu: 20m
              memory: 40Mi
            requests:
              cpu: 10m
              memory: 20Mi
          readinessProbe:
            httpGet:
              path: /
              port: 8081
            initialDelaySeconds: 5
            timeoutSeconds: 5
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: true
            runAsUser: 65534
      nodeSelector:
        kubernetes.io/os: linux
      serviceAccountName: kube-state-metrics
```

### [05_kube-state-metrics-serviceMonitor.yaml](../config/monitoring/kube-state-metrics/05_kube-state-metrics-serviceMonitor.yaml)

```yaml
# ../config/monitoring/kube-state-metrics/05_kube-state-metrics-serviceMonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 1.9.7
    name: kube-state-metrics
    prometheus-enabled: "true"
  name: kube-state-metrics
  namespace: monitoring
spec:
  namespaceSelector:
    matchNames:
      - monitoring
  endpoints:
    - port: http-metrics
      interval: 30s
      honorLabels: true
  jobLabel: app.kubernetes.io/name
  selector:
    matchLabels:
      app.kubernetes.io/component: exporter
      app.kubernetes.io/name: kube-state-metrics
      app.kubernetes.io/part-of: kube-prometheus
```

Let's apply all these files using `kubectl`:

```bash
kubectl apply -f config/monitoring/kube-state-metrics/
```

Check the pods in the `monitoring` namespace, which should have `kube-state-metrics-xxx` pods up and running.

## Kubelet

`Kubelet` is an essential part of Kubernetes control plane, and is also something that exposes Prometheus metrics by default in the port `10255`. So, it makes sense to create a Service Monitor for it as well.
The `kube-state-metrics` collects lots of data, and some of them overlap with `Kubelet` provided metrics, but not all; some information can be collected only from `Kubelet`.

We only need to create one file: [`kubelet-servicemonitor.yaml`](../config/monitoring/kubelet/kubelet-servicemonitor.yaml):

### [kubelet-servicemonitor.yaml](../config/monitoring/kubelet/kubelet-servicemonitor.yaml)

```yaml
# ../config/monitoring/kubelet/kubelet-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app.kubernetes.io/name: kubelet
    name: kubelet
  name: kubelet
  namespace: monitoring
spec:
  endpoints:
    - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
      honorLabels: true
      interval: 30s
      metricRelabelings:
        - action: drop
          regex: kubelet_(pod_worker_latency_microseconds|pod_start_latency_microseconds|cgroup_manager_latency_microseconds|pod_worker_start_latency_microseconds|pleg_relist_latency_microseconds|pleg_relist_interval_microseconds|runtime_operations|runtime_operations_latency_microseconds|runtime_operations_errors|eviction_stats_age_microseconds|device_plugin_registration_count|device_plugin_alloc_latency_microseconds|network_plugin_operations_latency_microseconds)
          sourceLabels:
            - __name__
        - action: drop
          regex: scheduler_(e2e_scheduling_latency_microseconds|scheduling_algorithm_predicate_evaluation|scheduling_algorithm_priority_evaluation|scheduling_algorithm_preemption_evaluation|scheduling_algorithm_latency_microseconds|binding_latency_microseconds|scheduling_latency_seconds)
          sourceLabels:
            - __name__
        - action: drop
          regex: apiserver_(request_count|request_latencies|request_latencies_summary|dropped_requests|storage_data_key_generation_latencies_microseconds|storage_transformation_failures_total|storage_transformation_latencies_microseconds|proxy_tunnel_sync_latency_secs)
          sourceLabels:
            - __name__
        - action: drop
          regex: kubelet_docker_(operations|operations_latency_microseconds|operations_errors|operations_timeout)
          sourceLabels:
            - __name__
        - action: drop
          regex: reflector_(items_per_list|items_per_watch|list_duration_seconds|lists_total|short_watches_total|watch_duration_seconds|watches_total)
          sourceLabels:
            - __name__
        - action: drop
          regex: etcd_(helper_cache_hit_count|helper_cache_miss_count|helper_cache_entry_count|request_cache_get_latencies_summary|request_cache_add_latencies_summary|request_latencies_summary)
          sourceLabels:
            - __name__
        - action: drop
          regex: transformation_(transformation_latencies_microseconds|failures_total)
          sourceLabels:
            - __name__
        - action: drop
          regex: (admission_quota_controller_adds|crd_autoregistration_controller_work_duration|APIServiceOpenAPIAggregationControllerQueue1_adds|AvailableConditionController_retries|crd_openapi_controller_unfinished_work_seconds|APIServiceRegistrationController_retries|admission_quota_controller_longest_running_processor_microseconds|crdEstablishing_longest_running_processor_microseconds|crdEstablishing_unfinished_work_seconds|crd_openapi_controller_adds|crd_autoregistration_controller_retries|crd_finalizer_queue_latency|AvailableConditionController_work_duration|non_structural_schema_condition_controller_depth|crd_autoregistration_controller_unfinished_work_seconds|AvailableConditionController_adds|DiscoveryController_longest_running_processor_microseconds|autoregister_queue_latency|crd_autoregistration_controller_adds|non_structural_schema_condition_controller_work_duration|APIServiceRegistrationController_adds|crd_finalizer_work_duration|crd_naming_condition_controller_unfinished_work_seconds|crd_openapi_controller_longest_running_processor_microseconds|DiscoveryController_adds|crd_autoregistration_controller_longest_running_processor_microseconds|autoregister_unfinished_work_seconds|crd_naming_condition_controller_queue_latency|crd_naming_condition_controller_retries|non_structural_schema_condition_controller_queue_latency|crd_naming_condition_controller_depth|AvailableConditionController_longest_running_processor_microseconds|crdEstablishing_depth|crd_finalizer_longest_running_processor_microseconds|crd_naming_condition_controller_adds|APIServiceOpenAPIAggregationControllerQueue1_longest_running_processor_microseconds|DiscoveryController_queue_latency|DiscoveryController_unfinished_work_seconds|crd_openapi_controller_depth|APIServiceOpenAPIAggregationControllerQueue1_queue_latency|APIServiceOpenAPIAggregationControllerQueue1_unfinished_work_seconds|DiscoveryController_work_duration|autoregister_adds|crd_autoregistration_controller_queue_latency|crd_finalizer_retries|AvailableConditionController_unfinished_work_seconds|autoregister_longest_running_processor_microseconds|non_structural_schema_condition_controller_unfinished_work_seconds|APIServiceOpenAPIAggregationControllerQueue1_depth|AvailableConditionController_depth|DiscoveryController_retries|admission_quota_controller_depth|crdEstablishing_adds|APIServiceOpenAPIAggregationControllerQueue1_retries|crdEstablishing_queue_latency|non_structural_schema_condition_controller_longest_running_processor_microseconds|autoregister_work_duration|crd_openapi_controller_retries|APIServiceRegistrationController_work_duration|crdEstablishing_work_duration|crd_finalizer_adds|crd_finalizer_depth|crd_openapi_controller_queue_latency|APIServiceOpenAPIAggregationControllerQueue1_work_duration|APIServiceRegistrationController_queue_latency|crd_autoregistration_controller_depth|AvailableConditionController_queue_latency|admission_quota_controller_queue_latency|crd_naming_condition_controller_work_duration|crd_openapi_controller_work_duration|DiscoveryController_depth|crd_naming_condition_controller_longest_running_processor_microseconds|APIServiceRegistrationController_depth|APIServiceRegistrationController_longest_running_processor_microseconds|crd_finalizer_unfinished_work_seconds|crdEstablishing_retries|admission_quota_controller_unfinished_work_seconds|non_structural_schema_condition_controller_adds|APIServiceRegistrationController_unfinished_work_seconds|admission_quota_controller_work_duration|autoregister_depth|autoregister_retries|kubeproxy_sync_proxy_rules_latency_microseconds|rest_client_request_latency_seconds|non_structural_schema_condition_controller_retries)
          sourceLabels:
            - __name__
      port: https-metrics
      relabelings:
        - sourceLabels:
            - __metrics_path__
          targetLabel: metrics_path
      scheme: https
      tlsConfig:
        insecureSkipVerify: true
    - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
      honorLabels: true
      honorTimestamps: false
      interval: 30s
      metricRelabelings:
        - action: drop
          regex: container_(network_tcp_usage_total|network_udp_usage_total|tasks_state|cpu_load_average_10s)
          sourceLabels:
            - __name__
      path: /metrics/cadvisor
      port: https-metrics
      relabelings:
        - sourceLabels:
            - __metrics_path__
          targetLabel: metrics_path
      scheme: https
      tlsConfig:
        insecureSkipVerify: true
    - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
      honorLabels: true
      interval: 30s
      path: /metrics/probes
      port: https-metrics
      relabelings:
        - sourceLabels:
            - __metrics_path__
          targetLabel: metrics_path
      scheme: https
      tlsConfig:
        insecureSkipVerify: true
  jobLabel: k8s-app
  namespaceSelector:
    matchNames:
      - kube-system
  selector:
    matchLabels:
      k8s-app: kubelet
```

Apply the `Kubelet` service monitor:

```bash
kubectl apply -f config/monitoring/kubelet/
```

## Traefik

I do not use `Traefik` much in my setup, but it is there, and it also exposes Prometheus-ready data, so why not.

We only need to create one file: [`kubelet-servicemonitor.yaml`](../config/monitoring/kubelet/kubelet-servicemonitor.yaml):

### [traefik-servicemonitor.yaml](../config/monitoring/traefik/traefik-servicemonitor.yaml)

```yaml
# ../config/monitoring/traefik/traefik-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: traefik
    release: prometheus
    name: traefik
  name: traefik
  namespace: monitoring
spec:
  endpoints:
    - port: metrics
  namespaceSelector:
    matchNames:
      - kube-system
  selector:
    matchLabels:
      app: traefik
```

Apply the `Traefik` service monitor:

```bash
kubectl apply -f config/monitoring/traefik/
```

## Phew..

That was a lot of configuration setup. We should now have the following Service Monitors up and ready to be scraped by Prometheus.

```bash
kubectl get ServiceMonitor -n monitoring
```

```text
NAME                                 AGE
longhorn-prometheus-servicemonitor   2d18h
node-exporter                        2d18h
kube-state-metrics                   2d18h
kubelet                              2d18h
traefik                              2d18h
```

[Prev: Monitoring: Prometheus Operator](./08_monitoring_prometheus_operator.md) | [Next: Monitoring: Prometheus](./10_monitoring_prometheus.md)
