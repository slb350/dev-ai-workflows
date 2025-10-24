# Observability Workflow - Logs, Metrics, Traces for Side Projects

**Version:** 1.0  
**Last Updated:** 2025-10-17  
**Purpose:** A pragmatic observability blueprint suited for solo/side projects across Python, Go, Rust, and TypeScript services.

---

## Overview

This workflow keeps your observability stack lean but effective:

- Structured logging in every service.  
- Metrics export via Prometheus-compatible endpoints.  
- Distributed tracing through OTEL (OpenTelemetry) and Tempo/Jaeger.  
- Log aggregation into Loki (or simple file rotation when needed).  
- Dashboards and alerts in Grafana/Prometheus.  
- Dev/Prod parity via the Local Infrastructure workflow.

---

## Core Principles

1. **Common Schema** - All services log JSON with `service`, `environment`, `trace_id`.  
2. **Unified Instrumentation** - Use OpenTelemetry SDKs for traces/metrics/logs where feasible.  
3. **Config via Env** - Toggle sinks (stdout, Loki, OTLP) using environment variables.  
4. **Sampling & Cost Awareness** - Default to 10% trace sampling in dev, adjustable for prod.  
5. **Start Small, Expand Later** - Logs + key metrics first, add tracing when ready.  
6. **Automated Dashboards** - Provision Grafana dashboards and alerts with config files.  
7. **Testing Observability** - Smoke tests ensure metrics/traces/logs actually flow.

---

## Stack Overview

| Component | Tool | Local Target | Notes |
| --- | --- | --- | --- |
| Logs | Structured JSON to stdout (Pino, Structlog, Zerolog, Tracing) | `docker compose` piping to Loki | Fallback to file rotation for CLI apps. |
| Metrics | OpenTelemetry Metrics or Prometheus client libraries | Prometheus scrape endpoint | For SQLite-only apps, pushgateway optional. |
| Traces | OTLP exporters per language | Grafana Tempo | Jaeger is an alternative. |
| Visualization | Grafana | `http://localhost:3001` (per local infra) | Provision dashboards for each service. |
| Alerts | Prometheus Alertmanager (optional) | Email/Slack (if configured) | Start with simple thresholds (error rate, latency). |

---

## Language Instrumentation Cheatsheet

| Language | Logging | Metrics | Tracing |
| --- | --- | --- | --- |
| Python | `structlog`, `loguru`, std logging with JSONFormatter | `prometheus-client`, OTEL metric SDK | `opentelemetry-sdk`, `opentelemetry-instrumentation` |
| Go | `zerolog`, `zap` | `prometheus/client_golang`, OTEL Go metrics | `go.opentelemetry.io/otel` |
| Rust | `tracing` + `tracing-subscriber` | `metrics` crate with `prometheus` exporter | `opentelemetry` + `tracing-opentelemetry` |
| TypeScript | `pino`, `winston` | `prom-client`, OTEL Node metrics | `@opentelemetry/sdk-node` |

Each service should emit:

```json
{
  "timestamp": "2025-10-17T12:34:56.789Z",
  "level": "info",
  "service": "payments-api",
  "environment": "development",
  "message": "Created user",
  "trace_id": "a1b2c3d4",
  "span_id": "abcd1234",
  "extra": { "user_id": 42 }
}
```

Use middleware/interceptors to add trace IDs automatically.

---

## Environment Variables

```text
# .env.example additions
OTEL_SERVICE_NAME=python-service
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1

PROMETHEUS_SCRAPE_PORT=9464
LOG_LEVEL=info
LOG_FORMAT=json
```

Set per service; ensure `OTEL_SERVICE_NAME` unique.

---

## Service Instrumentation Templates

### Python (FastAPI/asyncio)

```python
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
import structlog

resource = Resource.create({
    "service.name": os.getenv("OTEL_SERVICE_NAME", "python-service"),
    "service.environment": os.getenv("APP_ENV", "development"),
})

provider = TracerProvider(resource=resource)
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
trace.set_tracer_provider(provider)

FastAPIInstrumentor.instrument_app(app)

structlog.configure(processors=[structlog.processors.JSONRenderer()])
```

Expose metrics:

```python
from prometheus_client import Counter, Histogram, start_http_server

REQUEST_COUNT = Counter("http_requests_total", "Total HTTP requests", ["method", "path", "status"])
REQUEST_LATENCY = Histogram("http_request_duration_seconds", "Request latency", ["method", "path"])

start_http_server(int(os.environ.get("PROMETHEUS_SCRAPE_PORT", 9464)))
```

### Go (Echo/Gin)

```go
logger := zerolog.New(os.Stdout).With().
    Str("service", os.Getenv("SERVICE_NAME")).
    Str("environment", os.Getenv("APP_ENV")).
    Timestamp().
    Logger()

tp := tracesdk.NewTracerProvider(
    tracesdk.WithBatcher(otlptracegrpc.NewClient()),
    tracesdk.WithResource(resource.NewWithAttributes(
        semconv.SchemaURL,
        semconv.ServiceNameKey.String("go-service"),
    )),
)
otel.SetTracerProvider(tp)

promRegistry := prometheus.NewRegistry()
promRegistry.MustRegister(promhttp_metricHandler)
```

### Rust (Axum/Tonic)

```rust
use tracing_subscriber::fmt::time::UtcTime;

let otel_tracer = opentelemetry_otlp::new_pipeline()
    .tracing()
    .with_exporter(opentelemetry_otlp::new_exporter().tonic().with_endpoint(endpoint))
    .with_trace_config(
        trace::config()
            .with_resource(Resource::new(vec![KeyValue::new("service.name", service_name)])),
    )
    .install_batch(runtime)?;

tracing_subscriber::registry()
    .with(
        tracing_subscriber::fmt::layer()
            .with_timer(UtcTime::rfc_3339())
            .json(),
    )
    .with(tracing_opentelemetry::layer().with_tracer(otel_tracer))
    .init();
```

Expose metrics via `axum-prometheus` or `metrics-exporter-prometheus`.

### TypeScript (Fastify)

```ts
import pino from 'pino';
import { PrometheusClient } from '@opentelemetry/sdk-metrics';
import { NodeSDK } from '@opentelemetry/sdk-node';

const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  transport: process.env.NODE_ENV === 'development' ? { target: 'pino-pretty' } : undefined,
});

const sdk = new NodeSDK({
  metricReader: new PrometheusClient({
    endpoint: `/metrics`,
    port: Number(process.env.PROMETHEUS_SCRAPE_PORT ?? 9464),
  }),
});
await sdk.start();
```

---

## Grafana & Dashboards

Provision Grafana dashboards automatically:

```text
config/grafana/provisioning/dashboards/services.yml
config/grafana/dashboards/python-service.json
config/grafana/dashboards/go-service.json
```

Dashboards to include:

- Request rate / latency / error rate.  
- Database latency (`pg_stat_statements`, custom metrics).  
- Resource utilization (CPU, memory).  
- Trace waterfall view (Tempo).  
- Log search panel (Loki query).

Alerts (optional) in `config/prometheus/prometheus.yml`:

```yaml
groups:
  - name: service-health
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 10m
        labels:
          severity: warning
        annotations:
          description: "High 5xx rate for {{ $labels.service }}"
```

---

## Testing Observability

1. **Logs**: Run a service endpoint, verify JSON logs appear in Loki:

   ```bash
   docker compose logs -f service | jq
   curl "http://localhost:3100/loki/api/v1/query?query={service=\"python-service\"}"
   ```

2. **Metrics**: `curl http://localhost:9464/metrics` (service) and check Prometheus UI.  
3. **Traces**: Trigger request, open Grafana Tempo explore, confirm spans present.  
4. **End-to-End Smoke** (Justfile target):

   ```just
   obs-smoke:
   	curl -fsS http://localhost:9464/metrics | head
   	curl -fsS "http://localhost:3100/loki/api/v1/labels"
   	curl -fsS "http://localhost:3001/api/health"
   ```

---

## Checklists

**Before Running Services**  

- [ ] `just up` from Local Infra workflow.  
- [ ] Grafana dashboards provisioned (check UI).  
- [ ] Environment variables set for OTEL and Prometheus ports.  
- [ ] Logging format configured (JSON in production).  
- [ ] Sampling rate verified (OTEL_TRACES_SAMPLER_ARG).

**Before Release**  

- [ ] Observability docs updated (what metrics to watch).  
- [ ] Alerts tuned (if using Prometheus Alertmanager).  
- [ ] Dashboards exported to repo (`grafana dashboards export`).  
- [ ] Logging filters reviewed (sensitive data removed).  
- [ ] Traces verified end-to-end.

---

## Anti-Patterns

- Logging plain strings without structure.  
- Ignoring trace context propagation between services.  
- Exposing metrics on default port without auth in production.  
- Forgetting to rotate or cap log storage.  
- Sampling 100% of traces in production without regard for cost.  
- Shipping instrumentation code that panics/crashes when collector offline (wrap with retries).

---

## Success Metrics

- ✅ Structured logs searchable in Loki (or local file) within seconds.  
- ✅ Metrics endpoint scraped by Prometheus without errors.  
- ✅ Traces visible for key workflows (login, user creation).  
- ✅ Grafana dashboards load instantly with relevant panels.  
- ✅ Observability configuration toggled via `.env` alone.

---

## Troubleshooting

| Issue | Fix |
| --- | --- |
| No logs in Loki | Check service stdout, ensure docker logging driver not interfering. |
| Prometheus scrape fails | Verify `/metrics` endpoint reachable, update Prometheus config. |
| Traces missing | Confirm OTEL exporter endpoint, ensure service uses context propagation. |
| Grafana dashboards missing | Check provisioning logs (`docker compose logs grafana`). |
| Metrics names mismatch | Align names/labels across languages, update dashboards. |
| High cardinality | Avoid identifiers in labels (use IDs in logs, not metrics). |

---

**Remember:** Observability is a feature. Keep it lightweight but consistent across your Python, Go, Rust, and TypeScript services, and you will spot issues long before they reach users. 🔭
