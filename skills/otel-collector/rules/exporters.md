---
title: "Exporters"
impact: CRITICAL
tags:
  - exporters
  - otlp
  - dynatrace
  - authentication
---

# Exporters

Exporters send processed telemetry to backends.
Every pipeline must end with at least one exporter.

## Choosing a protocol

Use OTLP/gRPC for Collector-to-backend communication.
gRPC provides better throughput and supports bidirectional streaming, which matters at the Collector's aggregation volume.
Fall back to OTLP/HTTP only when network proxies do not support HTTP/2.

| Protocol | Exporter key | Default port | When to use |
|----------|-------------|-------------|-------------|
| gRPC | `otlp` | 4317 | Default for all Collector-to-backend exports |
| HTTP | `otlphttp` | 4318 | Network proxies that block HTTP/2 |

## OTLP exporter to Dynatrace

Use the `otlphttp` exporter (HTTP/protobuf) to send traces, metrics, and logs to Dynatrace.
Dynatrace primarily supports OTLP/HTTP on port 4318.

### Where to get configuration values

1. **OTLP Endpoint**: In Dynatrace: Settings → OpenTelemetry and OpenTracing → Copy the OTLP endpoint URL.
2. **Auth Token**: In Dynatrace: Settings → Access Tokens → Generate token with scopes `openTelemetryTrace.ingest`, `metrics.ingest`, `logs.ingest`.

### Minimal configuration

```yaml
exporters:
  otlp:
    endpoint: <OTLP_ENDPOINT>
    headers:
      Authorization: "Api-Token <AUTH_TOKEN>"
```

Replace `<OTLP_ENDPOINT>` with your Dynatrace environment OTLP endpoint (e.g., `https://<environment-id>.live.dynatrace.com/api/v2/otlp`).
Replace `<AUTH_TOKEN>` with your Dynatrace API token; see the [Authentication](#authentication) section for how to optimally set up the authentication token.

### Production configuration

Configure retry, timeout, compression, and sending queue for reliable delivery.

```yaml
exporters:
  otlp:
    endpoint: <OTLP_ENDPOINT>
    headers:
      Authorization: "Api-Token <AUTH_TOKEN>"
    compression: gzip
    timeout: 30s
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 30s
      max_elapsed_time: 300s
    sending_queue:
      enabled: true
      num_consumers: 10
      queue_size: 5000
      storage: file_storage
```

### Compression

Enable `gzip` compression to reduce network bandwidth.
The Dynatrace OTLP endpoint supports gzip-compressed OTLP/HTTP.

```yaml
# GOOD — reduces bandwidth by 60-80 percent
exporters:
  otlp:
    endpoint: <OTLP_ENDPOINT>
    headers:
      Authorization: "Api-Token <AUTH_TOKEN>"
    compression: gzip

# BAD — uncompressed traffic wastes bandwidth
exporters:
  otlp:
    endpoint: <OTLP_ENDPOINT>
    headers:
      Authorization: "Api-Token <AUTH_TOKEN>"
    compression: none
```

### Retry on failure

Enable retries to handle transient network errors and backend unavailability.

| Setting | Default | Recommendation |
|---------|---------|----------------|
| `initial_interval` | `5s` | Keep default unless backend has strict rate limiting |
| `max_interval` | `30s` | Keep default for exponential backoff ceiling |
| `max_elapsed_time` | `300s` | Increase for backends with extended maintenance windows |
| `randomization_factor` | `0.5` | Keep default to spread retry storms |

### Sending queue

The sending queue buffers telemetry when the backend is temporarily unavailable.
Without it, data is dropped during transient failures.

```yaml
exporters:
  otlp:
    endpoint: <OTLP_ENDPOINT>
    headers:
      Authorization: "Api-Token <AUTH_TOKEN>"
    sending_queue:
      enabled: true
      num_consumers: 10
      queue_size: 5000
      storage: file_storage
```

| Setting | Default | Recommendation |
|---------|---------|----------------|
| `num_consumers` | `10` | Increase for high-throughput pipelines |
| `queue_size` | `1000` | Set to 5000 for production workloads |
| `storage` | (in-memory) | Set to `file_storage` for persistence across restarts |

Use `file_storage` with the [file storage extension](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/extension/storage/filestorage) to persist the queue to disk.
In-memory queues lose buffered data when the Collector restarts.

### Authentication

Do not hardcode auth tokens in configuration files.
Reference an environment variable instead.

Create a dedicated API token with ingest permissions only.
The Collector needs to send telemetry, not query or manage the environment.
In Dynatrace, create the token at Settings → Access Tokens → Generate token and select the scopes `openTelemetryTrace.ingest`, `metrics.ingest`, and `logs.ingest`.
See [Dynatrace OTel documentation](https://docs.dynatrace.com/docs/observe/opentelemetry) for details on available token scopes.

```yaml
# GOOD — token from environment variable
exporters:
  otlp:
    endpoint: <OTLP_ENDPOINT>
    headers:
      Authorization: "Api-Token ${env:DT_API_TOKEN}"

# BAD — hardcoded token in config
exporters:
  otlp:
    endpoint: <OTLP_ENDPOINT>
    headers:
      Authorization: "Api-Token dt0c01.xxx..."
```

Set the environment variable in your deployment manifest:

```yaml
env:
  - name: DT_API_TOKEN
    valueFrom:
      secretKeyRef:
        name: dynatrace-credentials
        key: api-token
```

## OTLP/HTTP exporter

Use the OTLP/HTTP exporter when gRPC is not available (e.g., network proxies that do not support HTTP/2).

```yaml
exporters:
  otlphttp:
    endpoint: https://<OTLP_ENDPOINT>
    headers:
      Authorization: "Api-Token ${env:DT_API_TOKEN}"
    compression: gzip
```

The OTLP/HTTP exporter uses port 4318 by default.
Check your Dynatrace environment OTLP endpoint for the correct URL.

## Debug exporter

Use the debug exporter during development to print telemetry to the Collector's stdout.
Do not enable the debug exporter in production — it generates excessive log output.

```yaml
# GOOD — development only
exporters:
  debug:
    verbosity: detailed

# BAD — debug exporter in production pipeline
exporters:
  debug:
    verbosity: detailed
  otlp:
    endpoint: <OTLP_ENDPOINT>
    headers:
      Authorization: "Api-Token ${env:DT_API_TOKEN}"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter]
      exporters: [otlp, debug]  # debug wastes CPU and I/O in production
```

### Verbosity levels

| Level | Output | Use for |
|-------|--------|---------|
| `basic` | One line per export (count only) | Verifying that data flows through the pipeline |
| `normal` | One line per telemetry item | Spot-checking individual items |
| `detailed` | Full telemetry item with all attributes | Debugging attribute values and structure |

## Multiple exporters

Send telemetry to multiple backends by listing multiple exporters in a pipeline.
Each exporter receives a copy of the data independently.

```yaml
exporters:
  otlp/dynatrace:
    endpoint: <OTLP_ENDPOINT>
    headers:
      Authorization: "Api-Token ${env:DT_API_TOKEN}"
  otlp/secondary:
    endpoint: secondary-backend:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter]
      exporters: [otlp/dynatrace, otlp/secondary]
```

Use named instances (`otlp/dynatrace`, `otlp/secondary`) to configure multiple exporters of the same type.

## Anti-patterns

- **Exporting without TLS.**
  OTLP/gRPC uses TLS by default.
  Setting `tls.insecure: true` sends telemetry (including potentially sensitive data) in plaintext.
  Only disable TLS for local development or within a trusted network with mTLS (e.g., a service mesh).
- **Hardcoding auth tokens in configuration files.**
  Tokens in config files end up in version control, container images, and logs.
  Use environment variables with `${env:VARIABLE_NAME}` syntax.
- **Missing `sending_queue` in production.**
  Without a queue, transient backend failures cause immediate data loss.
  Always enable the sending queue for production deployments.
- **Using debug exporter in production.**
  The debug exporter serializes every telemetry item to stdout, consuming CPU and disk I/O.
  Remove it from production pipelines.

## References

- [OTLP/gRPC exporter](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter/otlpexporter)
- [OTLP/HTTP exporter](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter/otlphttpexporter)
- [Debug exporter](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter/debugexporter)
- [File storage extension](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/extension/storage/filestorage)
