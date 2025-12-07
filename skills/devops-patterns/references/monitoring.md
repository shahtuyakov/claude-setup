# Monitoring & Observability

## The Three Pillars

| Pillar | Purpose | Tools |
|--------|---------|-------|
| Metrics | Numerical measurements over time | Prometheus, Datadog, CloudWatch |
| Logs | Event records with context | Loki, ELK Stack, CloudWatch Logs |
| Traces | Request flow across services | Jaeger, Zipkin, AWS X-Ray |

---

## Prometheus

### Installation (Kubernetes)

```bash
# Using Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-app'
    static_configs:
      - targets: ['app:3000']
    metrics_path: /metrics

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
```

### Node.js Metrics

```typescript
// metrics.ts
import { Registry, Counter, Histogram, Gauge, collectDefaultMetrics } from 'prom-client';

const register = new Registry();

// Collect default Node.js metrics
collectDefaultMetrics({ register });

// Custom metrics
export const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'path', 'status'],
  registers: [register],
});

export const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'path', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5],
  registers: [register],
});

export const activeConnections = new Gauge({
  name: 'active_connections',
  help: 'Number of active connections',
  registers: [register],
});

// Express middleware
import { Request, Response, NextFunction } from 'express';

export function metricsMiddleware(req: Request, res: Response, next: NextFunction) {
  const start = Date.now();

  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    const labels = {
      method: req.method,
      path: req.route?.path || req.path,
      status: res.statusCode.toString(),
    };

    httpRequestsTotal.inc(labels);
    httpRequestDuration.observe(labels, duration);
  });

  next();
}

// Metrics endpoint
export async function metricsHandler(req: Request, res: Response) {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
}
```

```typescript
// app.ts
import express from 'express';
import { metricsMiddleware, metricsHandler } from './metrics';

const app = express();

app.use(metricsMiddleware);
app.get('/metrics', metricsHandler);
```

### Alert Rules

```yaml
# alerts.yml
groups:
  - name: application
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: High error rate detected
          description: Error rate is {{ $value | humanizePercentage }}

      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: High latency detected
          description: p95 latency is {{ $value | humanizeDuration }}

      - alert: PodRestarting
        expr: |
          increase(kube_pod_container_status_restarts_total[1h]) > 3
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: Pod restarting frequently
          description: Pod {{ $labels.pod }} has restarted {{ $value }} times

      - alert: HighMemoryUsage
        expr: |
          container_memory_usage_bytes
          / container_spec_memory_limit_bytes > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: High memory usage
          description: Memory usage is {{ $value | humanizePercentage }}
```

---

## Grafana

### Dashboard Configuration

```json
{
  "dashboard": {
    "title": "Application Dashboard",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (path)",
            "legendFormat": "{{ path }}"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m])) * 100"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "steps": [
                { "value": 0, "color": "green" },
                { "value": 1, "color": "yellow" },
                { "value": 5, "color": "red" }
              ]
            }
          }
        }
      },
      {
        "title": "Latency (p95)",
        "type": "timeseries",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, path))",
            "legendFormat": "{{ path }}"
          }
        ]
      }
    ]
  }
}
```

### Grafana as Code (Kubernetes)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  labels:
    grafana_dashboard: "true"
data:
  app-dashboard.json: |
    {
      "title": "Application",
      "panels": [...]
    }
```

---

## Logging

### Structured Logging (Node.js)

```typescript
// logger.ts
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level: (label) => ({ level: label }),
  },
  base: {
    service: process.env.SERVICE_NAME || 'myapp',
    version: process.env.APP_VERSION || 'unknown',
    environment: process.env.NODE_ENV || 'development',
  },
  timestamp: pino.stdTimeFunctions.isoTime,
});

// Request logging middleware
import { Request, Response, NextFunction } from 'express';
import { v4 as uuidv4 } from 'uuid';

export function requestLogger(req: Request, res: Response, next: NextFunction) {
  const requestId = req.headers['x-request-id'] as string || uuidv4();
  const start = Date.now();

  // Add request ID to response headers
  res.setHeader('x-request-id', requestId);

  // Create child logger with request context
  req.log = logger.child({
    requestId,
    method: req.method,
    path: req.path,
    ip: req.ip,
  });

  req.log.info('Request started');

  res.on('finish', () => {
    const duration = Date.now() - start;
    req.log.info({
      status: res.statusCode,
      duration,
    }, 'Request completed');
  });

  next();
}
```

```typescript
// Usage
import { logger } from './logger';

logger.info({ userId: '123' }, 'User logged in');
logger.error({ err, orderId: '456' }, 'Failed to process order');
logger.warn({ threshold: 80, current: 85 }, 'Memory usage high');
```

### Loki + Promtail (Kubernetes)

```yaml
# promtail-config.yaml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    pipeline_stages:
      - json:
          expressions:
            level: level
            message: msg
            service: service
      - labels:
          level:
          service:
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
```

### LogQL Queries (Loki)

```logql
# Find errors
{app="myapp"} |= "error"

# Parse JSON logs
{app="myapp"} | json | level="error"

# Rate of errors
sum(rate({app="myapp"} |= "error" [5m])) by (pod)

# Latency from logs
{app="myapp"} | json | duration > 1000

# Search with context
{app="myapp"} |= "failed" | line_format "{{.message}}"
```

---

## Distributed Tracing

### OpenTelemetry (Node.js)

```typescript
// tracing.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'myapp',
    [SemanticResourceAttributes.SERVICE_VERSION]: '1.0.0',
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV,
  }),
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://jaeger:4318/v1/traces',
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-fs': { enabled: false },
    }),
  ],
});

sdk.start();

// Graceful shutdown
process.on('SIGTERM', () => {
  sdk.shutdown()
    .then(() => console.log('Tracing terminated'))
    .catch((error) => console.log('Error terminating tracing', error))
    .finally(() => process.exit(0));
});
```

```typescript
// Custom spans
import { trace, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('myapp');

async function processOrder(orderId: string) {
  return tracer.startActiveSpan('process-order', async (span) => {
    try {
      span.setAttribute('order.id', orderId);

      // Child span
      await tracer.startActiveSpan('validate-order', async (validateSpan) => {
        await validateOrder(orderId);
        validateSpan.end();
      });

      await tracer.startActiveSpan('charge-payment', async (paymentSpan) => {
        await chargePayment(orderId);
        paymentSpan.end();
      });

      span.setStatus({ code: SpanStatusCode.OK });
    } catch (error) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: error.message });
      span.recordException(error);
      throw error;
    } finally {
      span.end();
    }
  });
}
```

### Jaeger Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
spec:
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
        - name: jaeger
          image: jaegertracing/all-in-one:latest
          ports:
            - containerPort: 16686  # UI
            - containerPort: 4318   # OTLP HTTP
            - containerPort: 4317   # OTLP gRPC
          env:
            - name: COLLECTOR_OTLP_ENABLED
              value: "true"
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger
spec:
  selector:
    app: jaeger
  ports:
    - name: ui
      port: 16686
    - name: otlp-http
      port: 4318
    - name: otlp-grpc
      port: 4317
```

---

## Health Checks

### Kubernetes Probes

```typescript
// health.ts
import { Router } from 'express';
import { Pool } from 'pg';

const router = Router();
const db: Pool = /* your pool */;

// Liveness - is the app running?
router.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// Readiness - is the app ready to receive traffic?
router.get('/ready', async (req, res) => {
  try {
    // Check database
    await db.query('SELECT 1');

    // Check Redis (if used)
    // await redis.ping();

    res.json({ status: 'ready' });
  } catch (error) {
    res.status(503).json({
      status: 'not ready',
      error: error.message,
    });
  }
});

// Detailed health for debugging
router.get('/health/detailed', async (req, res) => {
  const checks = {
    database: await checkDatabase(),
    redis: await checkRedis(),
    memory: process.memoryUsage(),
    uptime: process.uptime(),
  };

  const allHealthy = Object.values(checks).every(
    (c) => typeof c !== 'object' || c.status === 'healthy'
  );

  res.status(allHealthy ? 200 : 503).json(checks);
});

async function checkDatabase() {
  try {
    const start = Date.now();
    await db.query('SELECT 1');
    return { status: 'healthy', latency: Date.now() - start };
  } catch (error) {
    return { status: 'unhealthy', error: error.message };
  }
}

export default router;
```

---

## Alertmanager

### Configuration

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/...'

route:
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'slack-notifications'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
    - match:
        severity: warning
      receiver: 'slack-notifications'

receivers:
  - name: 'slack-notifications'
    slack_configs:
      - channel: '#alerts'
        send_resolved: true
        title: '{{ .Status | toUpper }}: {{ .CommonAnnotations.summary }}'
        text: '{{ .CommonAnnotations.description }}'

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: '<pagerduty-key>'
        severity: critical

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname']
```

---

## SLA/SLO Monitoring

### SLI Definitions

```yaml
# slo.yml
slos:
  - name: availability
    description: Service availability
    sli:
      events:
        error_query: sum(rate(http_requests_total{status=~"5.."}[5m]))
        total_query: sum(rate(http_requests_total[5m]))
    objectives:
      - target: 0.999  # 99.9%
        window: 30d

  - name: latency
    description: Request latency
    sli:
      events:
        good_query: |
          sum(rate(http_request_duration_seconds_bucket{le="0.5"}[5m]))
        total_query: |
          sum(rate(http_request_duration_seconds_count[5m]))
    objectives:
      - target: 0.95   # 95% under 500ms
        window: 30d
```

### Error Budget Dashboard Query

```promql
# Error budget remaining
1 - (
  sum(increase(http_requests_total{status=~"5.."}[30d]))
  / sum(increase(http_requests_total[30d]))
) / (1 - 0.999)

# Burn rate (how fast we're consuming budget)
(
  sum(rate(http_requests_total{status=~"5.."}[1h]))
  / sum(rate(http_requests_total[1h]))
) / 0.001
```

---

## Quick Reference

### Essential Metrics

| Metric | Type | Use |
|--------|------|-----|
| Request rate | Counter | Traffic volume |
| Error rate | Counter | Service health |
| Latency p50/p95/p99 | Histogram | User experience |
| CPU/Memory usage | Gauge | Resource planning |
| Active connections | Gauge | Concurrency |
| Queue depth | Gauge | Backpressure |

### Essential Alerts

| Alert | Condition | Severity |
|-------|-----------|----------|
| High error rate | >5% errors for 5m | Critical |
| High latency | p95 >1s for 5m | Warning |
| Pod crash loop | >3 restarts in 1h | Warning |
| High memory | >90% for 5m | Warning |
| Disk full | >85% for 5m | Critical |

### Log Levels

| Level | Use |
|-------|-----|
| error | Failures requiring attention |
| warn | Potential issues, degraded state |
| info | Normal operations, key events |
| debug | Detailed debugging (disable in prod) |
