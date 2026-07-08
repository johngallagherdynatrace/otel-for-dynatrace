---
title: "Versioning, Stability, and Migration"
impact: MEDIUM
tags:
  - versioning
  - migration
  - stability
  - semconv-upgrades
---

# Versioning

Use the rename table below when writing or reviewing attribute names.
Always use the current (new) name — not the deprecated one.
If you encounter a deprecated name in existing code, replace it with the current name.

## Stability levels

| Level | Meaning |
|---|---|
| **Stable** | Will not change in breaking ways. Safe to depend on. |
| **Experimental** | May change or be removed. Use with caution. |
| **Deprecated** | Replaced by a newer attribute. Migrate away. |

Always use stable attributes.
Only use experimental attributes when no stable alternative exists, and add a code comment noting the attribute is experimental so it can be updated when it stabilizes.

## Key renames to know

| Old Name | New Name |
|---|---|
| `http.method` | `http.request.method` |
| `http.status_code` | `http.response.status_code` |
| `http.url` | `url.full` |
| `http.target` | `url.path` + `url.query` |
| `http.scheme` | `url.scheme` |
| `http.flavor` | `network.protocol.version` |
| `http.user_agent` | `user_agent.original` |
| `http.client_ip` | `client.address` |
| `net.peer.name` | `server.address` |
| `net.peer.port` | `server.port` |
| `net.host.name` | `server.address` |
| `net.host.port` | `server.port` |
| `db.system` | `db.system.name` |
| `db.name` | `db.namespace` |
| `db.statement` | `db.query.text` |
| `db.operation` | `db.operation.name` |
| `deployment.environment` | `deployment.environment.name` |
| `http.server.duration` | `http.server.request.duration` (unit: ms → s) |
| `http.client.duration` | `http.client.request.duration` (unit: ms → s) |

For the full list of attribute changes, see the [common attributes reference](./attributes.md#most-common-span-attributes).

## Dynatrace semantic convention handling

Dynatrace accepts telemetry using both old and new OpenTelemetry semantic convention naming at ingestion time.
This means:

- Services instrumented with older semconv naming (e.g., `http.method`) and services using the current stable naming (e.g., `http.request.method`) are both ingested and queryable.
- You do not need to upgrade all services simultaneously — Dynatrace normalises incoming telemetry and maps both naming schemes to a common internal representation.

Upgrade your instrumentation libraries at your own pace.
See [Dynatrace OTel ingestion documentation](https://docs.dynatrace.com/docs/observe/opentelemetry) for the current list of supported semantic convention versions.

## References

- [Semantic Conventions Repository](https://github.com/open-telemetry/semantic-conventions) — source YAML models, version tags, and changelogs
- [Dynatrace OTel documentation](https://docs.dynatrace.com/docs/observe/opentelemetry)
