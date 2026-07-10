---
name: review-instrumentation
description: >
  Reviews OpenTelemetry instrumentation changes for correctness and semantic convention compliance.
  Use when reviewing a pull request or current working-tree changes that add or modify OTel spans,
  metrics, logs, or resource attributes. Triggers on requests to review, audit, or check
  instrumentation, telemetry, tracing, metrics, or OTel changes — with or without a PR reference.
---

# Review instrumentation

## Input

`$ARGUMENTS` is an optional PR number, PR URL, or branch name.

- If provided: run `gh pr diff $ARGUMENTS` to get the diff.
- If empty: build a combined diff of everything that has changed on the current branch — committed and uncommitted — relative to `origin/main`:
  1. Run `git diff origin/main` to get all changes (committed branch commits plus any uncommitted working-tree changes, staged or not, relative to the upstream main).
  2. If that is empty (e.g., on main with no working-tree changes), run `git diff --cached` for staged-only changes.

If there is no diff after both attempts, tell the user there is nothing to review and stop.

## Rules to load

The otel-dt skills live in the installed plugin. Locate them by running:

```bash
find ~/.claude/plugins -path "*/otel-dt/skills/otel-instrumentation/rules" -type d 2>/dev/null | head -1
find ~/.claude/plugins -path "*/otel-dt/skills/otel-semantic-conventions/rules" -type d 2>/dev/null | head -1
```

Read rule files selectively based on what signals appear in the diff:

| Condition | Rule files to read |
|---|---|
| Any span or trace code | `spans.md`, `telemetry.md` |
| Any metric code | `metrics.md` |
| Any log code | `logs.md` |
| Resource attributes or service identity | `resources.md` |
| Attribute names or values | otel-semantic-conventions `attributes.md` |
| Sensitive data (PII, tokens, passwords) | `sensitive-data.md` |
| Always | `validation.md` |

If the plugin is not installed and you cannot find the rule files, proceed with your built-in OpenTelemetry knowledge and note in the report that the local skill files were not found.

## Review criteria

Check the diff against the loaded rules. Focus on:

1. **Span correctness** — naming follows `<verb> <noun>` pattern, kind is appropriate (`SERVER`, `CLIENT`, `INTERNAL`, etc.), status is set correctly (`ERROR` on exceptions, `UNSET` otherwise, never `OK` unless explicitly warranted), `span.end()` is always called.
2. **Error handling** — exceptions are recorded with `recordException`, status set to `ERROR` with a message, span still ends in a finally/defer block.
3. **Semantic conventions** — attribute names follow the registry (`http.request.method` not `http.method`, `db.system.name` not `db.type`, etc.). Flag deprecated names.
4. **Resource attributes** — `service.name`, `service.version`, and `deployment.environment.name` are present and resolved from environment variables, not hardcoded.
5. **Metrics** — correct instrument type for the measurement (Histogram for latency/size, Counter for totals, Gauge for current state), units are UCUM, names follow `<namespace>.<noun>` convention.
6. **Logs** — severity level is set, trace context (`trace_id`, `span_id`) is correlated, structured fields use semantic convention attribute names.
7. **Sensitive data** — no PII, credentials, or tokens in attribute values or log bodies.
8. **Context propagation** — inbound context is extracted before spans are created; outbound context is injected into requests.

## Output format

Print a markdown report with this structure:

```
## Instrumentation review

### Summary
<one sentence: what was changed and overall quality>

### Findings

#### Critical
<findings that will cause broken traces, wrong data, or data loss — numbered list, or "None">

#### Warnings
<semantic convention violations, missing best-practice attributes, deprecated names — numbered list, or "None">

#### Suggestions
<improvements that would raise quality but are not blocking — numbered list, or "None">

### Verdict
PASS | PASS WITH WARNINGS | FAIL
```

Each finding must include:
- The file and line number (or range)
- What is wrong
- What it should be instead, with a corrected code snippet if the fix is non-obvious

If there are no findings in a category, write "None" — do not omit the section.
Verdict is `FAIL` if there are any Critical findings, `PASS WITH WARNINGS` if there are Warnings but no Critical, `PASS` otherwise.
