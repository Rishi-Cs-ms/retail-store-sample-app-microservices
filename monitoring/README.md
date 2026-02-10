# Retail Store Monitoring Stack

## Overview
This directory contains the monitoring infrastructure for the Retail Store microservices application using Prometheus and Grafana.

## Components

### 1. Prometheus
- **Purpose**: Metrics collection and storage
- **Retention**: 7 days
- **Storage**: 10Gi persistent volume
- **Scrape Interval**: 30 seconds

### 2. Grafana
- **Purpose**: Metrics visualization and dashboards
- **Default Credentials**: admin / admin123 (change in production!)
- **Access**: LoadBalancer service

### 3. ServiceMonitor
- Automatically discovers and scrapes metrics from retail store services
- Targets services with label: `app.kubernetes.io/part-of: retail-store`
- Scrapes `/actuator/prometheus` endpoint

### 4. Alerts
Configured alerts for:
- High error rate (>5%)
- High response time (>2s p95)
- Pod down
- High CPU usage (>80%)
- High memory usage (>80%)
- Service unavailable

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Monitoring Stack                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐      ┌──────────────┐                    │
│  │  Prometheus  │◄─────│ServiceMonitor│                    │
│  │              │      └──────────────┘                    │
│  │  - Scrapes   │              │                            │
│  │  - Stores    │              ▼                            │
│  │  - Alerts    │      ┌──────────────┐                    │
│  └──────┬───────┘      │ Retail Store │                    │
│         │              │   Services   │                    │
│         │              │              │                    │
│         │              │ - UI         │                    │
│         │              │ - Cart       │                    │
│         │              │ - Catalog    │                    │
│         │              │ - Checkout   │                    │
│         │              │ - Orders     │                    │
│         │              └──────────────┘                    │
│         │                                                   │
│         ▼                                                   │
│  ┌──────────────┐                                          │
│  │   Grafana    │                                          │
│  │              │                                          │
│  │  - Dashboards│                                          │
│  │  - Alerts    │                                          │
│  │  - Queries   │                                          │
│  └──────────────┘                                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Deployment

### Prerequisites
- EKS cluster running
- Terraform applied with monitoring enabled
- ArgoCD installed

### Deploy via Terraform
```bash
cd terraform
terraform apply
```

This will deploy:
- kube-prometheus-stack Helm chart
- Prometheus operator
- Grafana
- AlertManager
- Node exporter
- Kube-state-metrics

### Deploy Monitoring Resources
```bash
kubectl apply -f monitoring/
```

Or via ArgoCD:
```bash
kubectl apply -f argocd/applications/monitoring.yaml
```

## Access

### Get Grafana URL
```bash
kubectl get svc -n monitoring kube-prometheus-stack-grafana
```

### Get Prometheus URL
```bash
kubectl get svc -n monitoring kube-prometheus-stack-prometheus
```

### Port Forward (Alternative)
```bash
# Grafana
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80

# Prometheus
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

## Dashboards

### Pre-configured Dashboards
1. **Retail Store Overview**
   - Request rate by service
   - Response time (p95)
   - Error rate
   - CPU usage
   - Memory usage
   - Active pods

2. **Kubernetes Cluster**
   - Node metrics
   - Pod metrics
   - Namespace metrics

3. **Application Metrics**
   - JVM metrics (for Java services)
   - HTTP metrics
   - Database connections

## Metrics Exposed

### Application Metrics
- `http_server_requests_seconds_count` - Request count
- `http_server_requests_seconds_sum` - Request duration
- `http_server_requests_seconds_bucket` - Request duration histogram
- `jvm_memory_used_bytes` - JVM memory usage
- `jvm_threads_live` - Active threads

### Infrastructure Metrics
- `container_cpu_usage_seconds_total` - CPU usage
- `container_memory_working_set_bytes` - Memory usage
- `kube_pod_status_phase` - Pod status
- `kube_deployment_status_replicas` - Deployment replicas

## Alerts

### Alert Severity Levels
- **Critical**: Immediate action required (service down, pod crash)
- **Warning**: Attention needed (high resource usage, slow response)
- **Info**: Informational (deployment events)

### Alert Channels
Configure in Grafana:
- Email
- Slack
- PagerDuty
- Webhook

## Troubleshooting

### Prometheus not scraping targets
```bash
# Check ServiceMonitor
kubectl get servicemonitor -n monitoring

# Check Prometheus targets
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# Visit http://localhost:9090/targets
```

### Grafana dashboard not showing data
1. Check data source configuration
2. Verify Prometheus is collecting metrics
3. Check service labels match ServiceMonitor selector

### High memory usage
```bash
# Check Prometheus storage
kubectl exec -n monitoring prometheus-kube-prometheus-stack-prometheus-0 -- df -h

# Reduce retention period in terraform/addons.tf
```

## Best Practices

1. **Retention Policy**: Adjust based on storage capacity
2. **Alert Tuning**: Fine-tune thresholds to reduce noise
3. **Dashboard Organization**: Create role-specific dashboards
4. **Resource Limits**: Set appropriate limits for Prometheus/Grafana
5. **Backup**: Regular backup of Grafana dashboards and Prometheus data

## Metrics to Monitor

### Golden Signals
1. **Latency**: Response time of requests
2. **Traffic**: Request rate
3. **Errors**: Error rate
4. **Saturation**: Resource utilization

### RED Method (for services)
- **Rate**: Requests per second
- **Errors**: Failed requests
- **Duration**: Response time

### USE Method (for resources)
- **Utilization**: % time resource is busy
- **Saturation**: Queue depth
- **Errors**: Error count

## Cost Optimization

- Use persistent volumes for long-term storage
- Configure appropriate retention periods
- Use recording rules for expensive queries
- Implement metric relabeling to reduce cardinality

## Security

- Change default Grafana password
- Enable authentication for Prometheus
- Use RBAC for access control
- Enable TLS for external access
- Implement network policies

## Next Steps

1. Add custom application metrics
2. Create service-specific dashboards
3. Configure alert notifications
4. Implement distributed tracing (Jaeger/X-Ray)
5. Add log aggregation (ELK/Loki)
