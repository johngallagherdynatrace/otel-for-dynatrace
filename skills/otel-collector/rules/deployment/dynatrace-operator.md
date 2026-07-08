---
title: "Dynatrace Operator"
impact: HIGH
tags:
  - deployment
  - kubernetes
  - operator
  - dynatrace
  - auto-instrumentation
---

# Dynatrace Operator

The [Dynatrace Operator](https://github.com/Dynatrace/dynatrace-operator) manages Dynatrace monitoring on Kubernetes.
It provides automatic code instrumentation injection via the CodeModules CSI driver — no changes to application images or Dockerfiles required.

Use it when Dynatrace is the observability backend and you want automatic OTel SDK injection for Node.js, Java, Python, .NET, and Go workloads running on Kubernetes.

## When to use the Dynatrace Operator

Use the Dynatrace Operator when:

- Dynatrace is the observability backend.
- You want automatic OTel SDK injection without modifying application images.
- You need Kubernetes metadata enrichment (pod, namespace, node attributes) without manual configuration.

Use the [OpenTelemetry Operator](./opentelemetry-operator.md) instead when:

- You are using a backend other than Dynatrace.
- You need fine-grained control over Collector configuration via Kubernetes CRDs.

## Installation

### 1. Create the Dynatrace namespace and credentials secret

```bash
kubectl create namespace dynatrace

kubectl create secret generic dynatrace-credentials \
  --namespace dynatrace \
  --from-literal=apiToken=<DT_API_TOKEN>
```

The API token must have these scopes:
- `openTelemetryTrace.ingest`
- `metrics.ingest`
- `logs.ingest`
- `installer.download` (required for CodeModules injection)

### 2. Install the operator with Helm

```bash
helm repo add dynatrace https://raw.githubusercontent.com/Dynatrace/dynatrace-operator/main/config/helm/repos/stable
helm repo update

helm install dynatrace-operator dynatrace/dynatrace-operator \
  --namespace dynatrace \
  --atomic
```

### 3. Create a DynaKube resource

```yaml
apiVersion: dynatrace.com/v1beta3
kind: DynaKube
metadata:
  name: dynakube
  namespace: dynatrace
spec:
  apiUrl: https://<environment-id>.live.dynatrace.com/api
  tokens: dynatrace-credentials
  applicationMonitoring:
    useCSIDriver: true
```

Replace `<environment-id>` with your Dynatrace environment ID.

## Enabling injection for a namespace

Label the target namespace so the operator injects instrumentation into pods:

```bash
kubectl label namespace <your-app-namespace> \
  instrumentation.opentelemetry.io/inject-nodejs=true
```

Use the label that matches your runtime:

| Runtime | Label |
|---------|-------|
| Node.js | `instrumentation.opentelemetry.io/inject-nodejs=true` |
| Java | `instrumentation.opentelemetry.io/inject-java=true` |
| Python | `instrumentation.opentelemetry.io/inject-python=true` |
| .NET | `instrumentation.opentelemetry.io/inject-dotnet=true` |
| Go | `instrumentation.opentelemetry.io/inject-go=true` |

The operator injects the OTel SDK into all new pods in the labelled namespace.
Existing pods require a restart.

## Validating the setup

### Check operator pods

```bash
kubectl get pods -n dynatrace
```

All operator pods should show `Running`.

### Check DynaKube status

```bash
kubectl describe dynakube dynakube -n dynatrace
```

Look for `Phase: Running` in the status section.

### Check injection in an application pod

```bash
kubectl describe pod <pod-name> -n <your-app-namespace>
```

Look for init containers named `install-codemodules` and environment variables beginning with `OTEL_` in the container spec.

### Confirm data in Dynatrace

Navigate to Dynatrace → Distributed traces and filter by `service.name` matching your workload.
Allow up to 2 minutes for the first traces to appear after pod restart.

## Comparison with raw Kubernetes manifests and the OpenTelemetry Operator

| Capability | Dynatrace Operator | OpenTelemetry Operator | Raw manifests |
|---|---|---|---|
| Auto-instrumentation injection | Yes (CodeModules CSI) | Yes (init container) | No |
| Collector management | No | Yes (CRD) | Manual |
| Kubernetes metadata enrichment | Yes | Yes (k8sattributes) | Manual |
| Dashboard and alert sync | No | No | No |
| Prometheus CRD support | No | Yes | No |
| Dynatrace backend required | Yes | No | No |

## References

- [Dynatrace Operator GitHub](https://github.com/Dynatrace/dynatrace-operator)
- [Dynatrace Operator documentation](https://docs.dynatrace.com/docs/setup-and-configuration/setup-on-k8s)
- [Dynatrace OTel documentation](https://docs.dynatrace.com/docs/observe/opentelemetry)
