# GreenShift Helm Charts

Helm charts for connecting Kubernetes clusters to [GreenShift](https://greenshift.app) for cost management and FinOps.

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

## Uninstall

```bash
helm uninstall <release-name> --namespace greenshift
```
