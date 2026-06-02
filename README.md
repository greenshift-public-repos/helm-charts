# GreenShift Helm Charts

Helm charts for connecting Kubernetes clusters to [GreenShift](https://greenshift.app) for cost management and FinOps.

## Architecture

The `kube-cost-metrics-collector` chart deploys a Prometheus stack in agent mode inside the clients' cluster. It scrapes resource metrics from cluster components and forwards them to GreenShift over HTTPS. Nothing is written back to the clients' cluster.

![Architecture overview](diagrams/k8s_connection_flow.svg)

### Data collection

Four components feed metrics into Prometheus before they are remote-written to GreenShift:

| Component | What it provides |
|-----------|-----------------|
| **kube-state-metrics** | Kubernetes object state (pods, deployments, resource requests/limits, …) |
| **node-exporter** | Node-level hardware and OS metrics |
| **cAdvisor** (embedded in kubelet) | Per-container CPU, memory, and filesystem usage |
| **kube-service-selectors** | Service-to-pod label mappings for cost attribution |

![Data collection components](diagrams/connector.png)

## Connection setup

When a client adds a Kubernetes data source in the GreenShift UI, it generates a pre-filled `helm install` command containing the `dataSourceId` and the remote-write endpoint. After running that command, Prometheus begins scraping every ~60 seconds and metrics are available in the UI within ~1 hour.

![Connection setup flow](diagrams/connection_flow.png)

## Prerequisites

- [Helm 3+](https://helm.sh/docs/intro/install/)

## Add the Repository

```bash
helm repo add greenshift https://greenshift-public-repos.github.io/helm-charts
helm repo update
```

## Available Charts

| Chart | Description |
|-------|-------------|
| [kube-cost-metrics-collector](charts/kube-cost-metrics-collector) | Collects Kubernetes resource metrics and forwards them to GreenShift |
| [kube-service-selectors](charts/kube-service-selectors) | Exports Kubernetes service selectors as metrics (installed automatically as a dependency) |

## Permissions

All access is read-only. The chart creates the following ClusterRoles:

![RBAC permissions per component](diagrams/permissions.png)

- **node-exporter** requires no Kubernetes API access — it reads `/proc` and `/sys` from the host filesystem only.
- **kube-service-selectors** gets `list`/`watch` on `services`.
- **kube-state-metrics** gets `list`/`watch` across core workload, storage, and policy resources.
- **Prometheus** gets `get`/`list`/`watch` on nodes, node metrics/proxy endpoints, services, endpoints, pods, and namespaces (needed for service discovery and kubelet scraping).

No resources are ever created, updated, or deleted in the clients' cluster by any of these components.

## Metric filtering

GreenShift only queries a small subset of the metrics Prometheus scrapes. To reduce storage and prevent Thanos-receive OOMs on high-cardinality clusters, the chart drops unused metrics at the `remote_write` layer before they reach Thanos — after scraping, so nothing in the client cluster is affected.

### What is dropped and why

| Dropped | Series saved | Reason |
|---------|-------------|--------|
| `apiserver_.*` | ~33K | API server latency — not a cost metric |
| `kube_replicaset_.*` | ~13K | ReplicaSet level — not queried |
| `etcd_.*` | ~9K | etcd internals — not used |
| `kube_pod_status_reason/phase/ready/scheduled` | ~11K | Pod status detail — not queried |
| `kube_pod_tolerations` | ~2K | Not queried |
| `container_.*` (except `container_cpu_usage_seconds_total` and `container_memory_usage_bytes`) | ~98 per container | cAdvisor exports ~100 container metrics; GreenShift only reads two |

Combined these drops reduce Thanos storage by ~48% compared to an unfiltered install.

Dropping at `write_relabel_configs` rather than disabling kube-state-metrics collectors is intentional — collectors like `pods` expose both needed metrics (`kube_pod_info`, `kube_pod_labels`) and unused ones (`kube_pod_status_phase`, etc.) in the same scrape. The only way to keep the former and discard the latter is at the remote_write layer.

### node-exporter

`prometheus-node-exporter` is disabled by default. GreenShift does not query any `node_*` metrics — node capacity is derived from `kube_node_status_capacity` (kube-state-metrics) instead.

### kube-state-metrics collectors

Only the collectors GreenShift actually needs are enabled: `pods`, `nodes`, `resourcequotas`, `services`. Disabling the rest (deployments, daemonsets, statefulsets, etc.) prevents those metrics from being scraped at all.

## Overriding values

Scalar values can be set with `--set` as normal. **Do not use `--set` for `prometheus.server.remote_write`** — `remote_write` is an array and Helm replaces the entire array entry at the given index rather than merging into it. This silently wipes `write_relabel_configs` and the auth wiring the chart template depends on, causing Prometheus to fail to authenticate and stop sending data.

Override the remote_write block via a values file instead:

```bash
cat > /tmp/my-values.yaml << 'EOF'
prometheus:
  server:
    remote_write:
      - url: https://<your-greenshift-host>/storage/api/v2/write
        name: greenshift
        write_relabel_configs:
          # ... full block here
EOF

helm install kcmc greenshift/kube-cost-metrics-collector \
  -f /tmp/my-values.yaml \
  --set prometheus.server.dataSourceId=<id> \
  --set prometheus.server.username=<user> \
  --set prometheus.server.password=<pass> \
  --namespace greenshift \
  --create-namespace
```

## Uninstall

```bash
helm uninstall <release-name> --namespace greenshift
```
