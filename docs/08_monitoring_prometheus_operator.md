# Monitoring: Prometheus Operator

We will deploy Prometheus to collect data from various services in our K3s Kubernetes cluster.

> \[NOTE]\
> In this guide, I assume the same settings and services as I have, mainly Longhorn for persistent storage. Data from Prometheus will be displayed in a single instance of Grafana.

## Components

The whole monitoring stack is built from several components, so here's a brief overview of the things we will be setting up for our monitoring setup:

### Prometheus Operator

This is a solo deployment. One instance that will help us provision Prometheus, and some of its components. It extends the Kubernetes API, so that when we create some YAML deployments it will look as if we’re telling Kubernetes to deploy something, but it’s actually telling Prometheus Operator to do it for us. [github: prometheus-operator](https://github.com/prometheus-operator/prometheus-operator)

### Prometheus

This is the collector of metrics, it uses something called service monitors that provide information Prometheus can come and scrape. Prometheus will use persistent storage, and we will specify for how long it will keep the data. You can have more than one instance of Prometheus in your cluster collecting separate data. Having multiple instances of Prometheus would ensure that if one died, it wouldn’t take the whole monitoring system with it. In our case we will have only one, which we will deploy to collect monitoring data from everything else. This is mainly because Prometheus keeps its stuff in RAM, and that comes at a premium on Raspberry Pi 4.

### Service Monitors

These are other containers/deployments. They are kind of middle steps between the data and Prometheus. We will deploy some that are a single deployment. For example, for Kubernetes API to collect metrics from a server, or a longhorn service monitor, it is also a single deployment. Then there is another kind of deployment: `daemonset`, which deploys containers to each node. `node-exporter` is using this kind of deployment, and that’s because it collects underlying OS information per node.

### Grafana

Prometheus can do some graphing of data, it has its own web UI. However, Grafana is on another level. Grafana as its name suggests, makes graphs. You can create custom dashboards, and display data collected by Prometheus. It can display data from multiple Prometheus instances, and combine them into a single dashboard, and more.

## Install Prometheus Operator

This is the helper that will extend Kubernetes API, and help us to deploy monitoring.

We are going to download the original Prometheus Operator from git, and do just one change. We are going to change the namespace in its configs.

```bash
# cd prometheus-operator/
wget https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/master/bundle.yaml
# Check current setting for namespaces in bundle.yaml
grep 'namespace: default' bundle.yaml # Should show 'namespace: default'

# We will change that to monitoring:
sed -i 's/namespace: default/namespace: monitoring/g' bundle.yaml
# NOTE: If you do not have access to `sed`, use a text/code editor to replace these values
#Check again:
grep 'namespace: ' bundle.yaml # should show 'namespace: monitoring'
```

Also create the namespace monitoring itself:

```bash
kubectl create namespace monitoring
```

Now apply `bundle.yaml` to your Kubernetes cluster:

```bash
# prometheus_operator/
kubectl apply --server-side -f bundle.yaml
```

This creates a bunch of `custom resource definitions`, which now extends our Kubernetes API and deploys one `prometheus-operator` into the namespace `monitoring`. Also created is the `service account` for this deployment, and `service`.

Next, in order to tell Prometheus which service monitors to scrape, we need to install service monitors.

[Prev: Network Configuration](./05_network.md) | [Next: Monitoring: Service Monitors](./09_monitoring_service_monitors.md)
