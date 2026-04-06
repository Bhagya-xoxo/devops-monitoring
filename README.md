# devops-monitoring

Reusable Kubernetes monitoring Helm charts based on kube-prometheus-stack v82.14.1.

## Overview

This repository contains a reusable Helm chart wrapper for deploying comprehensive
Kubernetes monitoring with Prometheus, Alertmanager, and Grafana. Designed to work
with both Devtron CD and on-premise Helm deployments.

## Structure

```
charts/
  kube-prometheus-stack-monitoring/
    Chart.yaml                    # Chart metadata and dependencies
    values.yaml                   # Default (sanitized) values
    README.md                     # Chart documentation
    ci/
      values-devtron.yaml         # Devtron-specific overrides  
      values-onprem.yaml          # On-prem Helm overrides
```

## Key Features

- Custom alerting rules for container CPU, memory, and crash detection
- EC2 host monitoring rules (CPU, memory, disk)
- Alertmanager with Teams webhook and email notification routing
- Grafana with persistent storage and auto-provisioned dashboards
- CRDs managed separately (crds.enabled: false)

## Usage

### With Devtron
Configure your Devtron Helm app to use this chart with environment-specific values.

### On-prem (Helm CLI)

Install CRDs separately first, then deploy the chart:

```bash
helm dependency update ./charts/kube-prometheus-stack-monitoring
helm install monitoring ./charts/kube-prometheus-stack-monitoring \
  -f ./charts/kube-prometheus-stack-monitoring/ci/values-onprem.yaml \
  -n monitoring --create-namespace
```

## Note on CRDs

CRDs (PrometheusRule, ServiceMonitor, etc.) are intentionally excluded from this
chart (crds.enabled: false). They must be installed separately before deploying
this chart to avoid Helm upgrade conflicts.
