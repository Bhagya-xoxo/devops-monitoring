# kube-prometheus-stack-monitoring

A reusable Helm chart wrapper for deploying production-grade Kubernetes monitoring
using [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) v82.14.1.

## Features

- Prometheus with EC2 service discovery and custom scrape configs
- Alertmanager with Teams webhook and email routing
- Grafana with persistent storage and auto-provisioned dashboards
- Custom PrometheusRule alerts for containers, EC2 hosts, and pod health
- Compatible with both Devtron CD and on-premise Helm deployments
- CRDs managed separately for safe upgrades

## Prerequisites

1. Kubernetes 1.25+
2. Helm 3.x
3. CRDs installed separately (see below)
4. StorageClass available for Grafana/Prometheus persistence
5. (For EC2 scraping) AWS credentials with EC2 describe permissions

## Installing CRDs

CRDs must be installed **before** deploying this chart:

```bash
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/prometheus-community/helm-charts/main/charts/kube-prometheus-stack/crds/crd-alertmanagerconfigs.yaml
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/prometheus-community/helm-charts/main/charts/kube-prometheus-stack/crds/crd-alertmanagers.yaml
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/prometheus-community/helm-charts/main/charts/kube-prometheus-stack/crds/crd-podmonitors.yaml
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/prometheus-community/helm-charts/main/charts/kube-prometheus-stack/crds/crd-probes.yaml
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/prometheus-community/helm-charts/main/charts/kube-prometheus-stack/crds/crd-prometheusagents.yaml
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/prometheus-community/helm-charts/main/charts/kube-prometheus-stack/crds/crd-prometheuses.yaml
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/prometheus-community/helm-charts/main/charts/kube-prometheus-stack/crds/crd-prometheusrules.yaml
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/prometheus-community/helm-charts/main/charts/kube-prometheus-stack/crds/crd-scrapeconfigs.yaml
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/prometheus-community/helm-charts/main/charts/kube-prometheus-stack/crds/crd-servicemonitors.yaml
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/prometheus-community/helm-charts/main/charts/kube-prometheus-stack/crds/crd-thanosrulers.yaml
```

## Installation

### On-premise (Helm CLI)

```bash
# Update chart dependencies
helm dependency update ./charts/kube-prometheus-stack-monitoring

# Install
helm install monitoring ./charts/kube-prometheus-stack-monitoring \
  -f ./charts/kube-prometheus-stack-monitoring/ci/values-onprem.yaml \
  -n monitoring --create-namespace

# Upgrade
helm upgrade monitoring ./charts/kube-prometheus-stack-monitoring \
  -f ./charts/kube-prometheus-stack-monitoring/ci/values-onprem.yaml \
  -n monitoring
```

### With Devtron

1. Add this repo as a Helm chart source in Devtron
2. Create a new Helm app pointing to `kube-prometheus-stack-monitoring`
3. Override values using `ci/values-devtron.yaml` as a template
4. Provide secrets (SMTP password, webhook URLs) via Devtron's secret management

## Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `fullnameOverride` | Release name prefix for all resources | `""` |
| `crds.enabled` | Install CRDs as part of chart | `false` |
| `defaultRules.create` | Create built-in Prometheus rules | `false` |
| `grafana.enabled` | Deploy Grafana | `true` |
| `grafana.adminPassword` | Grafana admin password | `"<SET>"` |
| `grafana.persistence.storageClassName` | Storage class for Grafana | `gp3` |
| `alertmanager.enabled` | Deploy Alertmanager | `true` |
| `prometheus.prometheusSpec.retention` | Data retention period | `10d` |
| `prometheus.prometheusSpec.image.tag` | Prometheus image version | `v3.3.1` |

## Custom Alert Rules

The chart ships with the following custom PrometheusRule groups:

| Rule Group | Alert | Threshold | Severity |
|------------|-------|-----------|----------|
| HighContainerCPU | ContainerCpuUsage | >80% for 15m | critical |
| HighContainerCPU | HighContainerCpuUsage | >90% for 15m | critical |
| EC2Host | EC2HostCPUUsage | >85% for 5m | critical |
| EC2Host | EC2HostDown | down for 5m | critical |
| EC2Host | EC2VMHostMemoryUsage | >90% for 5m | critical |
| EC2Host | EC2HostOutOfDiskSpace | >85% disk for 5m | critical |
| CPUThrottlingHigh | CPUThrottlingHigh | >88% throttle for 15m | critical |
| ContainerMemory | ContainerMemoryUsageWarning | >85% for 10m | warning |
| ContainerMemory | HighContainerMemoryUsage | >90% for 10m | critical |
| CrashLoopBackOff | CrashLoopBackOffDetected | any for 1m | critical |

## Alert Routing

| Matcher | Receiver | Channel |
|---------|----------|---------|
| `alertname = CrashLoopBackOffDetected` | crashloopback-multichannel | Teams webhook |
| `severity = critical` | critical-multichannel | Teams webhook |
| (default) | teams-notifications | Teams webhook |

## Kubelet Metric Relabeling

The chart drops high-cardinality and unused kubelet metrics to reduce storage:
- `container_memory_(mapped_file|swap)`
- `container_(network_tcp_usage_total|network_udp_usage_total|tasks_state|cpu_load_average_10s)`
- `container_blkio_device_usage_total`
- `container_file_descriptors`
