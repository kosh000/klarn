---
inclusion: manual
---

# Stage 9: Observability

## Goal
See everything happening in your cluster — metrics, logs, and traces. Know when things break before users notice.

## Three Pillars of Observability

| Pillar | What it answers | Tool |
|--------|----------------|------|
| **Metrics** | "How is the system performing?" (CPU, memory, request rate, error rate, latency) | Prometheus + Grafana |
| **Logs** | "What happened?" (application output, error messages, audit trail) | Fluent Bit + CloudWatch/Loki |
| **Traces** | "Where did this request go?" (request path across microservices) | OpenTelemetry + X-Ray/Jaeger |

## Metrics: Prometheus + Grafana

### Architecture
```
Your App → exposes /metrics endpoint (Prometheus format)
     ↓
Prometheus → scrapes /metrics every 15-30s, stores time-series data
     ↓
Grafana → queries Prometheus, displays dashboards
     ↓
Alertmanager → fires alerts when thresholds are breached
```

### Install with kube-prometheus-stack (Helm)
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set prometheus.prometheusSpec.retention=7d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```

This installs:
- Prometheus (metrics collection and storage)
- Grafana (dashboards)
- Alertmanager (alerting)
- Node Exporter (host metrics from each node)
- kube-state-metrics (Kubernetes object metrics)
- Pre-built dashboards for K8s monitoring

### Access Grafana
```bash
# Port forward
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80

# Default credentials: admin / prom-operator
# Open http://localhost:3000
```

### Prometheus Metrics Format
```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",path="/api/users",status="200"} 1234
http_requests_total{method="POST",path="/api/users",status="201"} 56
http_requests_total{method="GET",path="/api/users",status="500"} 3

# HELP http_request_duration_seconds Request duration histogram
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.1"} 900
http_request_duration_seconds_bucket{le="0.5"} 1100
http_request_duration_seconds_bucket{le="1.0"} 1180
http_request_duration_seconds_bucket{le="+Inf"} 1200
```

### Instrument Your App
```javascript
// Node.js example with prom-client
const client = require('prom-client');

const httpRequests = new client.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'path', 'status']
});

const httpDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Request duration',
  labelNames: ['method', 'path'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5]
});

// Expose metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});
```

### ServiceMonitor (tell Prometheus to scrape your app)
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: monitoring
  labels:
    release: monitoring   # Must match Prometheus selector
spec:
  namespaceSelector:
    matchNames: ["default"]
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

### PromQL (Query Language)
```promql
# Request rate (requests per second over 5 minutes)
rate(http_requests_total[5m])

# Error rate
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])

# 95th percentile latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# CPU usage by pod
rate(container_cpu_usage_seconds_total{namespace="default"}[5m])

# Memory usage
container_memory_working_set_bytes{namespace="default"}

# Pod restart count
kube_pod_container_status_restarts_total
```

### Alerting Rules
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-alerts
  namespace: monitoring
  labels:
    release: monitoring
spec:
  groups:
    - name: app.rules
      rules:
        - alert: HighErrorRate
          expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "High error rate on {{ $labels.service }}"
            description: "Error rate is {{ $value | humanizePercentage }}"

        - alert: PodCrashLooping
          expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} is crash looping"

        - alert: HighMemoryUsage
          expr: container_memory_working_set_bytes / container_spec_memory_limit_bytes > 0.9
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} memory usage > 90%"
```

## Logging: Fluent Bit

### Architecture
```
Pod stdout/stderr → written to /var/log/pods/ on node
     ↓
Fluent Bit (DaemonSet on each node) → reads log files, parses, enriches
     ↓
CloudWatch Logs / Loki / Elasticsearch → stores and indexes logs
     ↓
CloudWatch Insights / Grafana → search and visualize
```

### Install Fluent Bit for CloudWatch
```bash
# Using AWS for Fluent Bit
helm repo add eks https://aws.github.io/eks-charts
helm install aws-for-fluent-bit eks/aws-for-fluent-bit \
  --namespace logging --create-namespace \
  --set cloudWatch.region=us-east-1 \
  --set cloudWatch.logGroupName=/eks/eks-learning \
  --set cloudWatch.autoCreateGroup=true
```

### Structured Logging (Best Practice)
```json
{"timestamp":"2026-01-15T10:30:00Z","level":"error","message":"Failed to connect to database","service":"user-api","trace_id":"abc123","error":"connection refused","retry_count":3}
```

Always log in JSON format. This makes logs searchable and parseable.

### Fluent Bit with Loki (for Grafana)
```yaml
# Fluent Bit output config for Loki
[OUTPUT]
    Name        loki
    Match       *
    Host        loki.monitoring.svc.cluster.local
    Port        3100
    Labels      job=fluent-bit, namespace=$kubernetes['namespace_name'], pod=$kubernetes['pod_name']
    Auto_Kubernetes_Labels on
```

### CloudWatch Insights Queries
```
# Find errors in a specific pod
fields @timestamp, @message
| filter kubernetes.pod_name = "my-app-xyz"
| filter @message like /error/i
| sort @timestamp desc
| limit 50

# Count errors by service
fields @timestamp, @message
| filter @message like /error/i
| stats count() by kubernetes.labels.app
| sort count desc
```

## Tracing: OpenTelemetry

### What is Distributed Tracing?
When a request hits your system and passes through multiple services:
```
User → API Gateway → Auth Service → User Service → Database
```
A trace shows the entire journey with timing for each hop.

### OpenTelemetry (OTel)
Vendor-neutral standard for instrumentation. Collects traces, metrics, and logs.

```yaml
# Deploy OpenTelemetry Collector
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
spec:
  template:
    spec:
      containers:
        - name: collector
          image: otel/opentelemetry-collector-contrib:latest
          ports:
            - containerPort: 4317  # gRPC receiver
            - containerPort: 4318  # HTTP receiver
```

### Instrument Your App
```javascript
// Node.js with OpenTelemetry
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');
const { HttpInstrumentation } = require('@opentelemetry/instrumentation-http');
const { ExpressInstrumentation } = require('@opentelemetry/instrumentation-express');

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: 'http://otel-collector:4317'
  }),
  instrumentations: [
    new HttpInstrumentation(),
    new ExpressInstrumentation(),
  ]
});
sdk.start();
```

### AWS X-Ray Integration
```yaml
# OTel Collector config to export to X-Ray
exporters:
  awsxray:
    region: us-east-1

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [awsxray]
```

## Key Metrics to Monitor (Golden Signals)

| Signal | What to measure | Alert threshold |
|--------|----------------|-----------------|
| **Latency** | Request duration (p50, p95, p99) | p99 > 1s |
| **Traffic** | Requests per second | Sudden drop > 50% |
| **Errors** | Error rate (5xx / total) | > 1% for 5 min |
| **Saturation** | CPU, memory, disk usage | > 80% |

## Labs

### Lab 9.1: Prometheus + Grafana Setup
1. Install kube-prometheus-stack via Helm
2. Access Grafana, explore pre-built K8s dashboards
3. Find: node CPU usage, pod memory, container restarts
4. Write a PromQL query for pod CPU usage in your namespace

### Lab 9.2: Instrument an App
1. Add Prometheus metrics to your app (/metrics endpoint)
2. Create a ServiceMonitor to scrape it
3. Verify metrics appear in Prometheus
4. Build a Grafana dashboard with request rate, error rate, latency

### Lab 9.3: Alerting
1. Create a PrometheusRule for high error rate
2. Configure Alertmanager to send to Slack/email
3. Trigger the alert (make your app return 500s)
4. Verify alert fires and notification arrives

### Lab 9.4: Logging
1. Install Fluent Bit
2. Deploy an app that logs in JSON format
3. View logs in CloudWatch Insights
4. Write a query to find all errors in the last hour
5. Correlate a log entry with a specific trace ID

### Lab 9.5: Tracing
1. Deploy OpenTelemetry Collector
2. Instrument two services that call each other
3. View traces in X-Ray or Jaeger
4. Find a slow request and identify which service is the bottleneck

## Self-Test Questions

1. What are the "three pillars of observability"? What question does each one answer?
2. What's the difference between monitoring and observability?
3. Prometheus uses a "pull" model. What does that mean? How is it different from a "push" model (like CloudWatch)?
4. You deploy an app but forget to add a ServiceMonitor. Does Prometheus scrape it? Why not?
5. What's the difference between a counter and a histogram in Prometheus? Give an example of each.
6. Write a PromQL query that gives you the error rate (5xx responses / total responses) over the last 5 minutes.
7. Your liveness probe checks `/healthz` and your Prometheus scrapes `/metrics`. Are these the same thing? Should they be?
8. You set up an alert for "CPU > 80% for 5 minutes." It fires at 3 AM but the app is fine. What might be wrong with this alert?
9. What are the "four golden signals"? Name them.
10. Your app logs `Error: connection refused`. This appears in Fluent Bit output but not in CloudWatch. What could be wrong?
11. What's the difference between structured logging (JSON) and unstructured logging (plain text)? Why does it matter for searching?
12. What's a trace? What's a span? How do they relate to each other?
13. You have 3 microservices: A → B → C. A request takes 2 seconds total. How do you figure out which service is slow without tracing?
14. What's OpenTelemetry? Why is it better than using vendor-specific SDKs (like X-Ray SDK directly)?
15. You install kube-prometheus-stack and get pre-built dashboards. Name 3 things you can see immediately without any custom instrumentation.

## Checklist Before Moving On

- [ ] Can install and configure Prometheus + Grafana on EKS
- [ ] Can write PromQL queries for common scenarios
- [ ] Can instrument an application with Prometheus metrics
- [ ] Can create ServiceMonitors for custom apps
- [ ] Can set up alerting rules and Alertmanager
- [ ] Can deploy Fluent Bit for log collection
- [ ] Understand structured logging best practices
- [ ] Can query logs in CloudWatch Insights
- [ ] Understand distributed tracing concepts
- [ ] Can set up OpenTelemetry for trace collection
- [ ] Know the four golden signals and what to alert on
