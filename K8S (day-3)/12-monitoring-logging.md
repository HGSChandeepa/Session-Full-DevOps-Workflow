# Monitoring & Logging

## Overview

Effective monitoring and logging are crucial for maintaining healthy Kubernetes clusters. This guide covers metrics collection, logging strategies, and observability best practices.

## Observability Pillars

```
┌──────────────────────────────────────────┐
│          OBSERVABILITY                    │
├──────────────────────────────────────────┤
│                                           │
│  ┌────────────┐  ┌────────────┐         │
│  │  METRICS   │  │   LOGS     │         │
│  │            │  │            │         │
│  │ What/When  │  │    Why     │         │
│  └────────────┘  └────────────┘         │
│         │              │                  │
│         └──────┬───────┘                 │
│                ▼                          │
│         ┌────────────┐                   │
│         │  TRACING   │                   │
│         │            │                   │
│         │    How     │                   │
│         └────────────┘                   │
│                                           │
└──────────────────────────────────────────┘
```

---

## Metrics with Prometheus & Grafana

### Metrics Server

Basic resource metrics (CPU, memory).

**Install:**

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**Verify:**

```bash
kubectl top nodes
kubectl top pods
kubectl top pods -n kube-system
```

### Prometheus Stack

**Install with Helm:**

```bash
# Add Prometheus helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

**What's Included:**

- Prometheus Operator
- Prometheus
- Alertmanager
- Grafana
- Node Exporter
- kube-state-metrics
- Pre-configured dashboards

**Access Grafana:**

```bash
# Get Grafana password
kubectl get secret -n monitoring prometheus-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode

# Port forward
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Open: http://localhost:3000
# Username: admin
# Password: <from above>
```

### ServiceMonitor

Tells Prometheus how to scrape your application.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-metrics
  labels:
    app: myapp
spec:
  ports:
    - name: metrics
      port: 8080
      targetPort: 8080
  selector:
    app: myapp
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-metrics
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

### Application Metrics Example

**Instrument your app (Python example):**

```python
from prometheus_client import Counter, Histogram, generate_latest
from flask import Flask, Response

app = Flask(__name__)

# Define metrics
request_count = Counter('app_requests_total', 'Total requests')
request_duration = Histogram('app_request_duration_seconds', 'Request duration')

@app.route('/metrics')
def metrics():
    return Response(generate_latest(), mimetype='text/plain')

@app.route('/')
@request_duration.time()
def home():
    request_count.inc()
    return "Hello, World!"
```

### PrometheusRule (Alerts)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-alerts
  namespace: monitoring
spec:
  groups:
    - name: app.rules
      interval: 30s
      rules:
        # High error rate
        - alert: HighErrorRate
          expr: |
            rate(http_requests_total{status=~"5.."}[5m]) > 0.05
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High error rate detected"
            description: "Error rate is {{ $value }} for {{ $labels.pod }}"

        # Pod not ready
        - alert: PodNotReady
          expr: |
            kube_pod_status_ready{condition="false"} == 1
          for: 10m
          labels:
            severity: critical
          annotations:
            summary: "Pod {{ $labels.pod }} not ready"

        # High memory usage
        - alert: HighMemoryUsage
          expr: |
            container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High memory usage in {{ $labels.pod }}"
            description: "Memory usage is {{ $value | humanizePercentage }}"
```

### Grafana Dashboards

**Common Dashboards (pre-installed):**

- Kubernetes / Compute Resources / Cluster
- Kubernetes / Compute Resources / Namespace
- Kubernetes / Compute Resources / Pod
- Node Exporter / Full
- Persistent Volumes

**Import Custom Dashboard:**

```bash
# Via UI: + → Import → Enter Dashboard ID
# Popular IDs:
# - 7249: Kubernetes Cluster Monitoring
# - 6417: Kubernetes Cluster Monitoring
# - 8588: Kubernetes Deployment Stats
```

---

## Logging

### Logging Architecture

```
┌────────────────────────────────────────────┐
│              Application Pods              │
│  ┌──────┐  ┌──────┐  ┌──────┐            │
│  │ App  │  │ App  │  │ App  │            │
│  │ Logs │  │ Logs │  │ Logs │            │
│  └──┬───┘  └──┬───┘  └──┬───┘            │
└─────┼─────────┼─────────┼─────────────────┘
      │         │         │
      ▼         ▼         ▼
┌────────────────────────────────────────────┐
│         DaemonSet (Log Collector)          │
│    Fluentd / Fluent Bit / Filebeat        │
└────────────────┬───────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────┐
│         Log Aggregation                    │
│    Elasticsearch / Loki / CloudWatch      │
└────────────────┬───────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────┐
│         Visualization                      │
│       Kibana / Grafana                    │
└────────────────────────────────────────────┘
```

### EFK Stack (Elasticsearch, Fluentd, Kibana)

**Install Elasticsearch:**

```bash
helm repo add elastic https://helm.elastic.co
helm install elasticsearch elastic/elasticsearch \
  --namespace logging \
  --create-namespace \
  --set replicas=1 \
  --set minimumMasterNodes=1
```

**Install Kibana:**

```bash
helm install kibana elastic/kibana \
  --namespace logging \
  --set elasticsearchHosts="http://elasticsearch-master:9200"
```

**Install Fluentd:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluentd
rules:
  - apiGroups: [""]
    resources: ["pods", "namespaces"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluentd
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: fluentd
subjects:
  - kind: ServiceAccount
    name: fluentd
    namespace: logging
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccountName: fluentd
      containers:
        - name: fluentd
          image: fluent/fluentd-kubernetes-daemonset:v1-debian-elasticsearch
          env:
            - name: FLUENT_ELASTICSEARCH_HOST
              value: "elasticsearch-master"
            - name: FLUENT_ELASTICSEARCH_PORT
              value: "9200"
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
```

### Loki Stack (Prometheus-like for logs)

**Install with Helm:**

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --namespace logging \
  --create-namespace \
  --set grafana.enabled=true \
  --set prometheus.enabled=true \
  --set promtail.enabled=true
```

**Access Grafana:**

```bash
kubectl port-forward -n logging svc/loki-grafana 3000:80
```

**Query logs in Grafana:**

```
{namespace="production", app="myapp"}
{namespace="production"} |= "error"
{namespace="production"} |= "error" | json
```

### Application Logging Best Practices

**Structured Logging (JSON):**

```python
import logging
import json

logger = logging.getLogger(__name__)

def log_structured(level, message, **kwargs):
    log_entry = {
        "timestamp": datetime.utcnow().isoformat(),
        "level": level,
        "message": message,
        **kwargs
    }
    logger.log(getattr(logging, level), json.dumps(log_entry))

# Usage
log_structured("INFO", "User logged in", user_id=123, ip="192.168.1.1")
```

**Log to stdout/stderr:**

```yaml
# Application logs to stdout
# Kubernetes captures and stores
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: app
      image: myapp
      # No need for volume mounts for logs
```

---

## Distributed Tracing

### Jaeger

**Install:**

```bash
kubectl create namespace observability
kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/main/deploy/crds/jaegertracing.io_jaegers_crd.yaml
kubectl create -n observability -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/main/deploy/service_account.yaml
kubectl create -n observability -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/main/deploy/role.yaml
kubectl create -n observability -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/main/deploy/role_binding.yaml
kubectl create -n observability -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/main/deploy/operator.yaml
```

**Deploy Jaeger instance:**

```yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger
  namespace: observability
spec:
  strategy: allInOne
  allInOne:
    image: jaegertracing/all-in-one:latest
    options:
      log-level: debug
  storage:
    type: memory
  ingress:
    enabled: false
  ui:
    options:
      dependencies:
        menuEnabled: true
```

**Instrument application (Python example):**

```python
from jaeger_client import Config
from opentracing import tracer

config = Config(
    config={
        'sampler': {'type': 'const', 'param': 1},
        'logging': True,
    },
    service_name='myapp',
)
tracer = config.initialize_tracer()

# Use in code
with tracer.start_span('process_request') as span:
    span.set_tag('user_id', user_id)
    # ... process request
```

---

## Monitoring Best Practices

### 1. Define SLOs/SLIs

**Service Level Indicators (SLIs):**

```yaml
# Availability SLI
up{job="myapp"} == 1

# Latency SLI
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Error rate SLI
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])
```

**Service Level Objectives (SLOs):**

- Availability: 99.9% uptime
- Latency: 95th percentile < 200ms
- Error rate: < 0.1%

### 2. Golden Signals

**Latency:**

```promql
histogram_quantile(0.95,
  rate(http_request_duration_seconds_bucket[5m])
)
```

**Traffic:**

```promql
rate(http_requests_total[5m])
```

**Errors:**

```promql
rate(http_requests_total{status=~"5.."}[5m])
```

**Saturation:**

```promql
container_memory_usage_bytes / container_spec_memory_limit_bytes
```

### 3. RED Method

- **Rate**: Requests per second
- **Errors**: Failed requests per second
- **Duration**: Request latency

### 4. USE Method

- **Utilization**: % time resource is busy
- **Saturation**: Amount of queued work
- **Errors**: Count of error events

### 5. Alert Fatigue Prevention

**Good Alert:**

```yaml
# Actionable, specific, includes context
- alert: HighErrorRate
  expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
  for: 5m # Not flapping
  labels:
    severity: warning
    team: backend
  annotations:
    summary: "High error rate in {{ $labels.service }}"
    description: "Error rate is {{ $value | humanizePercentage }}"
    runbook: "https://wiki.company.com/runbooks/high-error-rate"
```

**Bad Alert:**

```yaml
# Too sensitive, no context
- alert: ServiceDown
  expr: up == 0
  for: 1m
  annotations:
    summary: "Something is down"
```

---

## Dashboards

### Essential Dashboards

**1. Cluster Overview:**

```
- CPU usage per node
- Memory usage per node
- Pod count per namespace
- Network I/O
- Storage usage
```

**2. Namespace View:**

```
- CPU/Memory per pod
- Request rate
- Error rate
- Pod restarts
```

**3. Application View:**

```
- Request latency (p50, p95, p99)
- Request rate
- Error rate
- Active connections
- Queue depth
```

**4. Infrastructure:**

```
- Node health
- etcd metrics
- API server latency
- Scheduler latency
```

---

## Log Queries

### Kibana (Elasticsearch)

```
# Search for errors in production namespace
kubernetes.namespace_name:"production" AND log:"error"

# Specific pod
kubernetes.pod_name:"myapp-xyz" AND log:"timeout"

# Time range and aggregation
kubernetes.namespace_name:"production"
AND log:"error"
AND @timestamp:[now-1h TO now]
```

### Loki (LogQL)

```
# Basic query
{namespace="production", app="myapp"}

# Filter by content
{namespace="production"} |= "error"

# JSON parsing
{namespace="production"}
| json
| level="ERROR"

# Metrics from logs
rate({namespace="production"} |= "error" [5m])

# Quantile
quantile_over_time(0.95,
  {namespace="production"}
  | json
  | unwrap duration [5m]
)
```

---

## Monitoring Tools Comparison

| Feature            | Prometheus      | Loki            | Elasticsearch    | Jaeger              |
| ------------------ | --------------- | --------------- | ---------------- | ------------------- |
| **Type**           | Metrics         | Logs            | Logs             | Traces              |
| **Query Language** | PromQL          | LogQL           | Lucene           | -                   |
| **Storage**        | Time-series     | Chunks          | Inverted index   | Various             |
| **Resource Usage** | Moderate        | Low             | High             | Moderate            |
| **Best For**       | Metrics, alerts | Log aggregation | Full-text search | Distributed tracing |

---

## Production Setup Example

**Complete monitoring stack:**

```bash
# 1. Install metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 2. Install Prometheus stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi

# 3. Install Loki
helm install loki grafana/loki-stack \
  --namespace logging \
  --create-namespace \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=50Gi

# 4. Install Jaeger
kubectl apply -f jaeger-operator.yaml
kubectl apply -f jaeger-instance.yaml

# 5. Configure alerts
kubectl apply -f prometheus-rules.yaml

# 6. Configure dashboards
# Import via Grafana UI
```

---

## Key Takeaways

1. **Use Prometheus** for metrics collection
2. **Use Loki or EFK** for log aggregation
3. **Use Jaeger** for distributed tracing
4. **Define SLOs** and monitor against them
5. **Follow Golden Signals** (Latency, Traffic, Errors, Saturation)
6. **Create actionable alerts** with runbooks
7. **Structure logs** in JSON format
8. **Monitor the 4 golden signals**
9. **Use dashboards** for visualization
10. **Regular review** and optimization

---

## Next Steps

👉 **[13-cicd-kubernetes.md](./13-cicd-kubernetes.md)** - Set up CI/CD pipelines for Kubernetes

## Additional Resources

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Loki Documentation](https://grafana.com/docs/loki/)
- [Jaeger Documentation](https://www.jaegertracing.io/docs/)
- [Google SRE Book](https://sre.google/books/)
