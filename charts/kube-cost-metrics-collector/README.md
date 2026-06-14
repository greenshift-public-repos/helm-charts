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
  --namespace greenshift \
  --create-namespace \
  --set prometheus.server.dataSourceId=<data-source-id> \
  --set prometheus.server.username=<username> \
  --set prometheus.server.password=<password> \
  --set "prometheus.server.remote_write[0].url=https://<your-greenshift-host>/storage/api/v2/write" \
  --set "prometheus.server.remote_write[0].name=greenshift"
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
| `kube-service-selectors.readinessProbe.initialDelaySeconds` | Seconds before the first readiness check on kube-service-selectors (default `30`) |
| `kube-service-selectors.readinessProbe.timeoutSeconds` | Timeout per readiness probe attempt on kube-service-selectors (default `15`) |
| `kube-service-selectors.readinessProbe.periodSeconds` | How often the readiness probe runs on kube-service-selectors (default `30`) |
| `kube-service-selectors.readinessProbe.successThreshold` | Consecutive successes to mark the pod ready (default `1`) |
| `kube-service-selectors.readinessProbe.failureThreshold` | Consecutive failures before the pod is marked not ready (default `3`) |

On large clusters the defaults are recommended. For smaller clusters you can tighten them:

```bash
helm install kube-cost-metrics-collector greenshift/kube-cost-metrics-collector \
  --set prometheus.server.dataSourceId=<data-source-id> \
  --set prometheus.server.username=<username> \
  --set prometheus.server.password=<password> \
  --set prometheus.server.remote_write[0].url=https://<your-greenshift-host>/storage/api/v2/write \
  --set prometheus.server.remote_write[0].name=greenshift \
  --set kube-service-selectors.readinessProbe.initialDelaySeconds=5 \
  --set kube-service-selectors.readinessProbe.timeoutSeconds=5 \
  --set kube-service-selectors.readinessProbe.periodSeconds=10 \
  --set kube-service-selectors.readinessProbe.successThreshold=1 \
  --set kube-service-selectors.readinessProbe.failureThreshold=3 \
  --namespace greenshift \
  --create-namespace
```

For all available options:

```bash
helm show values greenshift/kube-cost-metrics-collector
```

## Metric collection defaults

By default this chart collects only the metrics that GreenShift actively uses:

| Source | Default behaviour | Metrics kept |
|--------|------------------|--------------|
| kube-state-metrics | Runs with 4 collectors: `pods`, `nodes`, `resourcequotas`, `services` | `kube_pod_*`, `kube_node_*`, `kube_resourcequota`, `kube_service_*` |
| cAdvisor (kubelet) | Scraped; unused `container_*` metrics dropped at remote_write | `container_cpu_usage_seconds_total`, `container_memory_usage_bytes` |
| node-exporter | Disabled | — |

All unused collectors, labels, annotations, and cAdvisor metrics are dropped before reaching Thanos storage.

### Re-enabling optional collectors

Use `helm upgrade` — no uninstall required. Prometheus reloads its config automatically when the ConfigMap is updated.

**Re-enable node-exporter:**
```bash
helm upgrade kube-cost-metrics-collector greenshift/kube-cost-metrics-collector \
  --namespace greenshift \
  --reuse-values \
  --set prometheus.prometheus-node-exporter.enabled=true
```

**Add more kube-state-metrics collectors** (e.g. deployments, daemonsets):
```bash
helm upgrade kube-cost-metrics-collector greenshift/kube-cost-metrics-collector \
  --namespace greenshift \
  --reuse-values \
  --set "prometheus.kube-state-metrics.collectors={pods,nodes,resourcequotas,deployments,daemonsets}"
```

**Expose labels for additional resource types:**
```bash
helm upgrade kube-cost-metrics-collector greenshift/kube-cost-metrics-collector \
  --namespace greenshift \
  --reuse-values \
  --set "prometheus.kube-state-metrics.metricLabelsAllowlist={nodes=[*],pods=[*],services=[*]}"
```

**Expose annotations for additional resource types** (disabled by default):
```bash
helm upgrade kube-cost-metrics-collector greenshift/kube-cost-metrics-collector \
  --namespace greenshift \
  --reuse-values \
  --set "prometheus.kube-state-metrics.metricAnnotationsAllowList={nodes=[*],pods=[*],services=[*]}"
```

**Disable all metric filtering** (sends everything to Thanos):
```bash
helm upgrade kube-cost-metrics-collector greenshift/kube-cost-metrics-collector \
  --namespace greenshift \
  --reuse-values \
  --set "prometheus.server.writeRelabelConfigs=null"
```

## Upgrade

```bash
helm upgrade kube-cost-metrics-collector greenshift/kube-cost-metrics-collector \
  --namespace greenshift
```

### Upgrading from 0.1.7 to 0.1.8

`prometheus-node-exporter` is now **disabled by default**. Helm will delete the node-exporter DaemonSet as part of the upgrade — this is intentional. To keep it running, pass `--set prometheus.prometheus-node-exporter.enabled=true`.

Metric filtering rules (`write_relabel_configs`) are now injected by the chart template and no longer live in `values.yaml`. If you previously supplied a custom `remote_write` block via a values file, remove `write_relabel_configs` from it — the template now adds them automatically.

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
