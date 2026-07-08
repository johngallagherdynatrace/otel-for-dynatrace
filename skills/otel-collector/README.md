# otel-collector

Expert guidance for configuring and deploying the OpenTelemetry Collector to receive, process, and export telemetry to Dynatrace or any OTLP-compatible backend.

## Structure

```
otel-collector/
├── SKILL.md              # Skill manifest and entry point
├── README.md             # This file
└── rules/
    ├── receivers.md      # OTLP, Prometheus, filelog, hostmetrics
    ├── exporters.md      # OTLP/HTTP to Dynatrace, debug, authentication
    ├── processors.md     # Memory limiter, resource detection, ordering
    ├── pipelines.md      # Service section, per-signal config, connectors
    ├── deployment.md     # Agent vs gateway patterns, deployment method selection
    ├── deployment/
    │   ├── collector-helm-chart.md    # Collector Helm chart
    │   ├── opentelemetry-operator.md  # OTel Operator and auto-instrumentation
    │   ├── dynatrace-operator.md         # Dynatrace Operator
    │   └── raw-manifests.md         # DaemonSet, Deployment, RBAC, Docker Compose
    ├── sampling.md       # Head sampling, tail sampling, load balancing
    └── red-metrics.md    # Semconv-aligned RED metrics from traces
```

## Getting started

Install the skill:

```bash
npx skills add dynatrace/otel-collector
```

The skill activates automatically when working on Collector configuration tasks.

## Rules

| Rule | Impact | Description |
|------|--------|-------------|
| [receivers](./rules/receivers.md) | CRITICAL | OTLP, Prometheus, filelog, and hostmetrics receiver configuration |
| [exporters](./rules/exporters.md) | CRITICAL | OTLP/HTTP exporter to Dynatrace with authentication, retry, and queuing |
| [processors](./rules/processors.md) | HIGH | Required and recommended processors with ordering rules |
| [pipelines](./rules/pipelines.md) | CRITICAL | Service section, per-signal pipelines, complete working config |
| [deployment](./rules/deployment.md) | HIGH | Agent vs gateway patterns, deployment method selection |
| [raw-manifests](./rules/deployment/raw-manifests.md) | HIGH | Raw Kubernetes manifests — DaemonSet, Deployment, RBAC, Docker Compose |
| [dynatrace-operator](./rules/deployment/dynatrace-operator.md) | HIGH | Dynatrace Operator — automated instrumentation and Collector management |
| [sampling](./rules/sampling.md) | HIGH | Head sampling, tail sampling, load balancing |
| [red-metrics](./rules/red-metrics.md) | HIGH | Semconv-aligned RED metrics from traces |

## Quick start

**Get your credentials:**
- **OTLP Endpoint**: In Dynatrace: Settings → OpenTelemetry and OpenTracing → copy the OTLP ingest endpoint URL (format: `https://<environment-id>.live.dynatrace.com/api/v2/otlp`).
- **Auth Token**: In Dynatrace: Settings → Access Tokens → Generate token with scopes `openTelemetryTrace.ingest`, `metrics.ingest`, and `logs.ingest`.

**Minimal working configuration:**

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
    spike_limit_mib: 128

exporters:
  otlp:
    endpoint: <OTLP_ENDPOINT>
    headers:
      Authorization: "Api-Token <AUTH_TOKEN>"
    sending_queue:
      enabled: true
      queue_size: 5000
      storage: file_storage

extensions:
  file_storage:
    directory: /var/lib/otelcol/queue

service:
  extensions: [file_storage]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter]
      exporters: [otlp]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter]
      exporters: [otlp]
    logs:
      receivers: [otlp]
      processors: [memory_limiter]
      exporters: [otlp]
```

Replace the placeholders with values from your Dynatrace account:
- `<OTLP_ENDPOINT>`: your Dynatrace OTLP HTTP endpoint (e.g., `https://<environment-id>.live.dynatrace.com/api/v2/otlp`).
- `<AUTH_TOKEN>`: a Dynatrace API token with scopes `openTelemetryTrace.ingest`, `metrics.ingest`, and `logs.ingest`.

## Key principles

- **Processor ordering matters** — `memory_limiter` first in every pipeline.
- **One pipeline per signal** — separate pipelines for traces, metrics, and logs.
- **Memory safety is non-negotiable** — always configure `memory_limiter` in production.
- **Every component must be in a pipeline** — declared but unused components cause startup failure.

## Resources

- [OpenTelemetry Collector docs](https://opentelemetry.io/docs/collector/)
- [Dynatrace OTel documentation](https://docs.dynatrace.com/docs/observe/opentelemetry)

## License

MIT
