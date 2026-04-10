# kube-cost-metrics-collector

Collects Kubernetes resource and cost metrics from your cluster and forwards them to GreenShift.

## Prerequisites

- Kubernetes 1.17+
- Helm 3+
- A GreenShift account with a Kubernetes data source connected

## Installation

The recommended way is to connect a Kubernetes data source in the GreenShift UI — it generates the full install command with your credentials pre-filled.

For a self-hosted GreenShift instance, run:

```bash
helm repo add greenshift https://greenshift-public-repos.github.io/helm-charts
helm repo update

helm install kube-cost-metrics-collector greenshift/kube-cost-metrics-collector \
  --set prometheus.server.dataSourceId=<data-source-id> \
  --set prometheus.server.username=<username> \
  --set prometheus.server.password=<password> \
  --set prometheus.server.remote_write[0].url=https://<your-greenshift-host>/storage/api/v2/write \
  --set prometheus.server.remote_write[0].name=greenshift \
  --namespace greenshift \
  --create-namespace
```

One release per cluster. All required values are provided by GreenShift when you connect a data source.

## Parameters

| Parameter | Description |
|-----------|-------------|
| `prometheus.server.dataSourceId` | Data source ID from GreenShift |
| `prometheus.server.username` | Username set when connecting the data source |
| `prometheus.server.password` | Password set when connecting the data source |
| `prometheus.server.remote_write[0].url` | GreenShift metrics endpoint |
| `prometheus.kubeStateMetrics.enabled` | Set to `false` if kube-state-metrics is already running in your cluster |
| `prometheus.prometheus-node-exporter.enabled` | Set to `false` if node-exporter is already running in your cluster |
| `prometheus.prometheus-pushgateway.enabled` | Set to `false` if pushgateway is already running in your cluster |

For all available options:

```bash
helm show values greenshift/kube-cost-metrics-collector
```

## Upgrade

```bash
helm upgrade kube-cost-metrics-collector greenshift/kube-cost-metrics-collector \
  --namespace greenshift
```

### Upgrading from 0.1.0 to 0.1.1+

Run these commands before upgrading:

```bash
kubectl delete --namespace greenshift daemonset kube-cost-metrics-collector-prometheus-node-exporter
kubectl delete --namespace greenshift deployments kube-cost-metrics-collector-prometheus-pushgateway
kubectl scale deployments --namespace greenshift kube-cost-metrics-collector-prometheus-server --replicas=0
```

## Uninstall

```bash
helm uninstall kube-cost-metrics-collector --namespace greenshift
```
