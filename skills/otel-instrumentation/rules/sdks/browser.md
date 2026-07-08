---
title: 'Browser Instrumentation'
impact: HIGH
tags:
  - browser
  - web
  - frontend
  - rum
---

# Browser Instrumentation

Instrument web applications to monitor performance, user sessions, requests, and errors.

## Use Cases

- **Real User Monitoring (RUM)**: Capture actual user performance metrics including page load times and resource loading
- **Frontend Performance Analysis**: Identify bottlenecks in network requests, script execution, and rendering
- **Error Tracking**: Capture uncaught JavaScript errors and promise rejections
- **End-to-End Tracing**: Connect frontend interactions with backend traces
- **Custom Instrumentation**: Monitor specific user interactions or business logic workflows

## Option 1: OpenTelemetry Browser SDK (Recommended)

Standard OpenTelemetry instrumentation for browsers.
Works with any OTLP-compatible backend including Dynatrace.

**Prerequisites:**
- Your Dynatrace OTLP HTTP endpoint (format: `https://<environment-id>.live.dynatrace.com/api/v2/otlp`)
- A Dynatrace API token with scope `openTelemetryTrace.ingest`

**Install:**
```bash
npm install @opentelemetry/auto-instrumentations-web \
  @opentelemetry/sdk-trace-web \
  @opentelemetry/exporter-trace-otlp-http \
  @opentelemetry/resources \
  @opentelemetry/semantic-conventions
```

> [!NOTE]
> Dynatrace also offers a JavaScript RUM agent for full Real User Monitoring (session replay, user actions, Core Web Vitals).
> That agent is separate from OTel instrumentation — use it when you need RUM capabilities beyond distributed tracing.

## Option 2: OpenTelemetry JS SDK

OpenTelemetry's official SDK for browser instrumentation. Use when you need fine-grained control.

**Note**: Currently experimental.

### Installation

```bash
npm install @opentelemetry/api \
  @opentelemetry/sdk-trace-web \
  @opentelemetry/auto-instrumentations-web \
  @opentelemetry/exporter-trace-otlp-http
```

For bundling (Rollup example):

```bash
npm install --save-dev rollup @rollup/plugin-node-resolve rollup-plugin-commonjs
```

Add to `package.json`:

```json
{ "type": "module" }
```

### Initialization

Create `instrumentation.js`:

```javascript
import {
  WebTracerProvider,
  BatchSpanProcessor,
} from '@opentelemetry/sdk-trace-web';
import { getWebAutoInstrumentations } from '@opentelemetry/auto-instrumentations-web';
import { registerInstrumentations } from '@opentelemetry/instrumentation';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';

const provider = new WebTracerProvider();

provider.addSpanProcessor(
  new BatchSpanProcessor(
    new OTLPTraceExporter({
      url: 'https://<OTLP_ENDPOINT>/v1/traces',
      headers: { Authorization: 'Api-Token YOUR_API_TOKEN' },
    }),
  ),
);

provider.register();

registerInstrumentations({
  instrumentations: [getWebAutoInstrumentations()],
});
```

### Bundling

Create `rollup.config.js`:

```javascript
import resolve from '@rollup/plugin-node-resolve';
import commonjs from 'rollup-plugin-commonjs';

export default {
  input: 'src/instrumentation.js',
  output: { file: 'dist/index.js', format: 'iife' },
  plugins: [resolve(), commonjs()],
};
```

Build:

```bash
npm run build
```

### HTML Integration

```html
<script src="dist/index.js"></script>
```

### Testing

Open your website in a browser.
Network tab will show calls to the Dynatrace OTLP ingestion endpoint.

### Resources

- [OpenTelemetry JS Documentation](https://opentelemetry.io/docs/languages/js/getting-started/browser/)

## Browser-to-Server Correlation

To connect frontend traces with backend traces:

### 1. Configure Trace Propagation

```javascript
import { FetchInstrumentation } from '@opentelemetry/instrumentation-fetch';

new FetchInstrumentation({
  propagateTraceHeaderCorsUrls: [/api\.yoursite\.com/],
});
```

### 2. Backend CORS Configuration

```javascript
app.use(
  cors({
    allowedHeaders: [
      'Content-Type',
      'Authorization',
      'traceparent',
      'tracestate',
    ],
  }),
);
```
