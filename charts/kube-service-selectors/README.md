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

For all available options:

```bash
helm show values greenshift/kube-service-selectors
```

## Uninstall

```bash
helm uninstall kube-service-selectors --namespace greenshift
```
