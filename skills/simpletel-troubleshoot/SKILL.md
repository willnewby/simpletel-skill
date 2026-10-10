---
name: simpletel-troubleshoot
description: Diagnose an application that sends telemetry to simpletel — find the failing or slow request in its traces, read the failing span's error status, events and correlated logs, map them back to code, and confirm the fix. Use when the user asks to "debug <app> with simpletel", "find why <service> is failing", "read the traces for this error", or "troubleshoot this slowdown".
---
<!-- managed by: simpletel skill install (20261010.1953.15+bc2fd7f) — do not hand-edit; regenerate with `simpletel skill install` or `simpletel skill package` -->

Diagnose a problem in an application that already sends telemetry to
**simpletel** — a failing request, a slow endpoint, a regression — using the
read tools, and confirm the fix. Everything here is read-only; the repair
happens in the application's code.

Two equivalent surfaces, same documents:

- **CLI:** `simpletel get traces` (list) and `simpletel get trace <trace_id>`
  (detail), resolving the connection exactly like `run`/`verify`
  (`--endpoint`/`--team`/`--admin-token`, env, config).
- **MCP:** `simpletel mcp` exposes `get_traces`, `get_trace`, `verify`, and
  `usage` as agent tools. If your client already has them connected, prefer
  them — they return the same JSON documents.

If the application is not sending telemetry yet, run the
`simpletel-onboard` skill first; this one assumes the pipe works.

## 1. Confirm the connection and the team

Run `simpletel whoami`, and `simpletel team list` if the team is unclear.
A wrong team shows up as an empty list, never as another team's data
(reads are team-scoped). An error (`401`, unreachable endpoint) is a
credential/connection problem to fix before interpreting anything as
"no problems".

## 2. Find the signal

Start wide, then narrow. The default lookback is one hour:

```
simpletel get traces --errors-only                # every failing trace
simpletel get traces --service checkout-api       # one service
simpletel get traces --min-duration 500ms         # slow traces
simpletel get traces --since 15m --errors-only    # just now
simpletel get traces --all --limit 200            # widen (retention-bounded)
```

The table is newest-first: trace id, service, root operation, duration,
span count, error count. `--format json` prints the same rows verbatim
(MCP: `get_traces {errors_only: true, service, min_duration, since, limit}`).

If the list is empty: no data in the window — widen `--since`/`--all`, check
the service-name spelling, and confirm the app is still exporting
(`simpletel verify --service <name>` is the fastest pipe check). Traces age
out with the account's retention window, so "gone" usually means "older than
retention", not "never happened".

## 3. Open the trace

```
simpletel get trace <trace_id>            # span tree (default)
simpletel get trace <trace_id> --format flat
simpletel get trace <trace_id> --format json
```

Read the tree top-down: the root span is the requested operation; child
spans are the work it did. What to look for, in order:

| Evidence | Where it shows | What it tells you |
|---|---|---|
| Error status | a span marked error | which operation failed, and its status message |
| Exception event | events on the failing span (`exception.type`, `exception.message`, stacktrace) | the actual error and where it was raised |
| Correlated logs | log records beneath the span that emitted them (same trace and span id) | the application's own log lines at that moment — present when log correlation is wired |
| Durations | per-span duration | where the wall-clock went (the longest child is the usual culprit) |
| Attributes | span attributes | http status/route, db statement, peer address — the inputs to the failing call |

MCP: `get_trace {trace_id}` returns the same document.

## 4. Map the evidence to code

- `span.name` plus route/method attributes — the handler entrypoint.
- `service.name` (+ `service.version` when the app stamps it) — which
  process/build produced this.
- The span's instrumentation `scope.name` — which library produced the span:
  `opentelemetry.instrumentation.*` (Python),
  `@opentelemetry/instrumentation-*` (Node),
  `OpenTelemetry::Instrumentation::*` (Ruby),
  `go.opentelemetry.io/contrib/instrumentation/...` (Go), etc. A DB or HTTP
  client child span failing *before* your handler surfaced an error means the
  fault is downstream — read its `db.statement` / `server.address` /
  `http.response.status_code` attributes.
- A trace whose spans lack library scopes was produced without
  auto-instrumentation engaged: fix the pipe first (onboard skill step 6).

## 5. Confirm the fix

Reproduce the request after the change, then:

```
simpletel get traces --service <name> --since 5m --errors-only
simpletel get trace <fresh trace id>
```

The failing request should now produce a clean trace (no error spans, no
exception events), and the original trace remains for the record. When you
want the full onboarding gate again, `simpletel verify --service <name>`
still applies.

## 6. Report

Give the human the trace id — it is the shareable handle — plus the failing
operation, the evidence (exception message / status message / log line), and
the window you searched. Anyone with the team's credential opens the same
trace.

## Guardrails

- Read-only tools: never attempt to "fix" by editing data; change the code.
- Never print `simpletel env` output or credential values into shared logs or
  reports.
- Absence of data is a data question, not an absence of bugs: say which
  service and window you searched before concluding anything.
