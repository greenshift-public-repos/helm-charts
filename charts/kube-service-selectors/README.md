# kube-service-selectors

Exports Kubernetes service selectors as Prometheus metrics, enabling GreenShift to map pods to the services they belong to.

**This chart is installed automatically** as a dependency of `kube-cost-metrics-collector`. You do not need to install it separately under normal circumstances.

## Standalone Installation

If you need to install it independently:

```bash
helm repo add greenshift https://greenshift-public-repos.github.io/helm-charts
helm repo update

helm install kube-service-selectors greenshift/kube-service-selectors \
  --namespace greenshift \
  --create-namespace
```

## Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `readinessProbe.initialDelaySeconds` | Seconds before the first readiness probe fires | `30` |
| `readinessProbe.timeoutSeconds` | Seconds to wait for a probe response before counting it as a failure | `15` |
| `readinessProbe.periodSeconds` | How often (in seconds) to run the probe | `30` |
| `readinessProbe.successThreshold` | Consecutive successes required to mark the pod ready | `1` |
| `readinessProbe.failureThreshold` | Consecutive failures before the pod is marked not ready | `3` |

The defaults are tuned for large clusters where the service needs time to enumerate all namespaces on startup. For smaller clusters you can tighten them:

```bash
helm install kube-service-selectors greenshift/kube-service-selectors \
  --set readinessProbe.initialDelaySeconds=5 \
  --set readinessProbe.timeoutSeconds=5 \
  --set readinessProbe.periodSeconds=10 \
  --namespace greenshift \
  --create-namespace
```

For all available options:

```bash
helm show values greenshift/kube-service-selectors
```

## Uninstall

```bash
helm uninstall kube-service-selectors --namespace greenshift
```
