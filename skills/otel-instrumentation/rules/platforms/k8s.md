---
title: 'Kubernetes deployment'
impact: HIGH
tags:
  - kubernetes
  - deployment
  - resource-attributes
  - downward-api
---

# Kubernetes deployment

OpenTelemetry-instrumented applications running in Kubernetes need pod metadata injected as resource attributes for proper contextualization of traces, metrics, and logs.
This file covers the pod-spec changes needed for application containers and the Dynatrace Operator approach.

For the full list of required and recommended resource attributes (including `service.name`, `service.namespace`, `service.version`, `service.instance.id`, and `deployment.environment.name`), see [resource attributes](../resources.md).
In Kubernetes, every application pod has more than one instance in aggregate across replicas — always set `service.instance.id` and the other [service identity](../resources.md#service-identity) attributes so telemetry can be attributed to a specific pod.

## Pod metadata via downward API

Use the Kubernetes downward API to expose pod metadata as environment variables.
The SDK reads `OTEL_RESOURCE_ATTRIBUTES` at startup and attaches these values to every signal.

### `k8s.pod.uid` (critical)

The most important Kubernetes resource attribute.
The `k8sattributes` processor uses it to resolve most other Kubernetes metadata (namespace, deployment, node) automatically.

Always set `k8s.pod.uid` explicitly rather than relying on IP-based pod detection.
IP-based association is unreliable with service meshes (Istio, Linkerd) or non-standard network configurations where multiple pods share the same IP.

### `k8s.container.name` (critical)

Identifies the container within a pod.
Set it for every multi-container pod (e.g., pods with sidecar proxies or log collectors).
The `k8sattributes` processor cannot distinguish between containers that share the same pod UID and IP.

There is no downward API field for the container name.
Set it as a literal value that matches the container name in the pod spec.

### `k8s.pod.name`

Human-readable pod identifier.
Pod names are easier to search than UIDs but are unique only within a namespace on a cluster.

### `k8s.node.name`

Identifies the node on which the pod runs.
Required when investigating performance issues caused by resource contention or node-pressure evictions.

Not needed for AWS EKS on Fargate, where each pod runs on a dedicated virtual node.

### Complete pod spec example

```yaml
spec:
  containers:
    - name: <container-name>
      env:
        - name: K8S_POD_UID
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.uid
        - name: K8S_POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: K8S_NODE_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: spec.nodeName
        - name: OTEL_SERVICE_NAME
          value: <service-name>
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: <otlp-endpoint>
        - name: OTEL_EXPORTER_OTLP_HEADERS
          valueFrom:
            secretKeyRef:
              name: otel-auth
              key: token
        - name: OTEL_TRACES_EXPORTER
          value: otlp
        - name: OTEL_METRICS_EXPORTER
          value: otlp
        - name: OTEL_LOGS_EXPORTER
          value: otlp
        - name: OTEL_RESOURCE_ATTRIBUTES
          value: "service.namespace=<service-namespace>,service.version=<service-version or commit sha>,service.instance.id=$(K8S_POD_UID),deployment.environment.name=<dev-env>,k8s.pod.uid=$(K8S_POD_UID),k8s.pod.name=$(K8S_POD_NAME),k8s.node.name=$(K8S_NODE_NAME),k8s.container.name=<container-name>"
```

The value of `service.instance.id` is set to the pod UID via the downward API so it is unique across pods and stable for the lifetime of the process.
This is the deterministic strategy referenced in [resolving `service.instance.id`](../resolve-values.md#serviceinstanceid); when pods run multiple worker processes in one container, generate a UUID v5 seeded on the pod UID and the worker index instead so each worker has its own identifier.
Setting `service.instance.id` does not remove the need for `k8s.pod.uid` — both must be set (see [`service.instance.id`](../resources.md#serviceinstanceid)).

Set `OTEL_TRACES_EXPORTER`, `OTEL_METRICS_EXPORTER`, and `OTEL_LOGS_EXPORTER` to `otlp` explicitly.
Supported values are `otlp`, `console`, and `none`.
The OTel specification designates `otlp` as the recommended default, but older SDK versions and manual SDK setups (such as Node.js without an auto-instrumentation package) may default to `none`, silently dropping all telemetry.

The `$(K8S_POD_UID)` syntax is a Kubernetes dependent environment variable reference — Kubernetes substitutes it with the value of the `K8S_POD_UID` variable defined earlier in the same `env` block.

The placeholders (`<service-name>`, `<service-version or commit sha>`, `<dev-env>`, `<container-name>`) require project-specific values.
See [resolving configuration values](../resolve-values.md) for ordered lookup strategies to derive each value from the codebase.

## Collector-level enrichment

Attributes that cannot be set from inside the pod — workload names and UIDs (`k8s.deployment.name`, `k8s.deployment.uid`, `k8s.namespace.name`, etc.) and cluster identity (`k8s.cluster.name`, `k8s.cluster.uid`) — are handled by OpenTelemetry Collector processors.

For `k8sattributes` processor configuration (metadata extraction, pod association, passthrough mode, RBAC), see [processors](../../../otel-collector/rules/processors.md).
For `resource` processor configuration (`k8s.cluster.name`, `k8s.cluster.uid`), see [processors](../../../otel-collector/rules/processors.md).
For Collector deployment manifests (DaemonSet, Deployment, RBAC), see [raw manifests](../../../otel-collector/rules/deployment/raw-manifests.md).

## Dynatrace Operator

Use the [Dynatrace Operator](https://github.com/Dynatrace/dynatrace-operator) to automate OTel SDK injection and resource attribute injection for all workloads in a cluster.
The operator injects instrumentation via CodeModules using a CSI driver, without requiring per-workload annotations.

When the Dynatrace Operator manages the workload with `applicationMonitoring`, skip the manual downward API setup above — the operator injects the required environment variables automatically.
See [Dynatrace Operator](../../../otel-collector/rules/deployment/dynatrace-operator.md) for full setup instructions.

## Anti-patterns

- **Using `k8s.pod.ip` instead of `k8s.pod.uid`.**
  Pod IPs are reused and can be shared across pods in service mesh configurations.
  Always set `k8s.pod.uid` via the downward API.
- **Omitting `k8s.container.name` in multi-container pods.**
  The `k8sattributes` processor cannot distinguish between containers sharing the same pod UID and IP.
  There is no downward API field for the container name — set it as a literal value matching the container name in the pod spec.
- **Missing downward API environment variables.**
  Without `k8s.pod.uid` in `OTEL_RESOURCE_ATTRIBUTES`, the Collector's `k8sattributes` processor falls back to unreliable connection-IP matching.
  Always expose pod metadata via the downward API and pass it to the SDK.
- **Omitting `service.instance.id` in Kubernetes pods.**
  Multiple pod replicas share the same `service.name`, `service.namespace`, and `service.version`, so instance-level analysis is impossible without `service.instance.id`.
  Derive it from the pod UID via the downward API — see the [service identity](../resources.md#service-identity) rule.

## References

- [Kubernetes semantic conventions](https://opentelemetry.io/docs/specs/semconv/resource/k8s/)
- [Kubernetes downward API](https://kubernetes.io/docs/concepts/workloads/pods/downward-api/)
- [Resource attributes](../resources.md)
