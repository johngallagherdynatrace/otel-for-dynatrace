---
title: "Dynatrace derived attributes and Davis AI"
impact: HIGH
tags:
  - dynatrace
  - davis-ai
  - entity-detection
  - topology
---

# Dynatrace derived attributes and Davis AI

Dynatrace ingests OpenTelemetry telemetry and derives additional attributes for entity detection, topology mapping, and Davis AI analysis.
Understanding which standard OTel attributes Dynatrace uses for these features determines what attributes you must set.

## Entity detection

Dynatrace creates monitored entities (services, processes, hosts) from incoming telemetry.
The attributes below are required for correct entity detection.
Missing or inconsistent values result in duplicate entities or unresolvable topology edges.

### Service entity

| OTel attribute | Required | Purpose |
|---|---|---|
| `service.name` | **Required** | Primary service identifier. Determines entity name. |
| `service.namespace` | Recommended | Scopes service name within a namespace. Prevents name collisions across teams or products. |
| `service.version` | Recommended | Enables version-aware analysis and deployment tracking. |
| `service.instance.id` | Recommended | Uniquely identifies each service instance. Required for per-instance analysis in multi-instance deployments. |

Set `service.name` on the resource, not on individual spans.
Putting `service.name` on spans creates unresolvable entity references.

```python
# CORRECT — service identity on the resource
resource = Resource({
    "service.name": "checkout-service",
    "service.namespace": "commerce",
    "service.version": "3.2.1",
    "service.instance.id": "pod-abc123",
})

# BAD — service.name on a span, not the resource
span.set_attribute("service.name", "checkout-service")
```

### Kubernetes entity correlation

When running on Kubernetes, set these attributes via the downward API so Dynatrace can correlate telemetry to Kubernetes entities (pods, deployments, namespaces, nodes):

| OTel attribute | Source |
|---|---|
| `k8s.pod.uid` | `metadata.uid` (downward API) |
| `k8s.pod.name` | `metadata.name` (downward API) |
| `k8s.namespace.name` | `metadata.namespace` (downward API) |
| `k8s.node.name` | `spec.nodeName` (downward API) |
| `k8s.deployment.name` | Set manually in the pod spec |

Use `k8s.pod.uid` (not `k8s.pod.ip`) for pod identity.
Pod IP addresses are unstable and break correlation when service meshes are in use.

The OpenTelemetry Collector's `k8sattributes` processor can inject Kubernetes metadata automatically.
See [otel-collector](../../otel-collector/) for configuration details.

## Davis AI topology mapping

Davis AI uses call relationships derived from distributed trace data to build a service topology map.
Topology edges are created from parent–child span relationships propagated via the `traceparent` header (W3C Trace Context).

For topology edges to appear correctly:

1. **Propagate `traceparent` on every outbound HTTP call.**
   Use an instrumentation library that injects the header automatically (e.g., `@opentelemetry/instrumentation-http`, `opentelemetry-instrumentation-requests`).
2. **Set `peer.service` on client spans for uninstrumented downstream services.**
   Without `peer.service`, Davis AI cannot label the downstream node in the topology map.

```python
# Set peer.service on a client span when the downstream is not instrumented
with tracer.start_as_current_span("call-payment-gateway") as span:
    span.set_attribute("peer.service", "payment-gateway")
    # ... make the outbound call
```

3. **Set `server.address` and `server.port` on client spans.**
   Davis AI uses these to group calls to the same host.

## Attributes that unlock Davis AI features

| Davis AI feature | Required attributes |
|---|---|
| Anomaly detection | `service.name`, `deployment.environment.name` |
| Root cause analysis | W3C `traceparent` propagation, `service.name` on resource |
| Topology map | `peer.service` on client spans for uninstrumented services |
| Deployment events | `service.version` |
| Release impact analysis | `service.version`, `deployment.environment.name` |

## References

- [Dynatrace OTel ingestion documentation](https://docs.dynatrace.com/docs/observe/opentelemetry)
- [Dynatrace entity model](https://docs.dynatrace.com/docs/discover-dynatrace/references/dynatrace-concepts/davis-ai/entity-detection)
- [OpenTelemetry resource attributes specification](https://opentelemetry.io/docs/specs/semconv/resource/)
