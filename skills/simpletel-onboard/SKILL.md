---
name: simpletel-onboard
description: Onboard an application to simpletel — instrument it with upstream OpenTelemetry and send its traces to simpletel. Use when the user asks to "onboard <app> to simpletel", "instrument <app> with OpenTelemetry", or "send traces to simpletel".
---
<!-- managed by: simpletel skill install (20261010.1953.15+bc2fd7f) — do not hand-edit; regenerate with `simpletel skill install` or `simpletel skill package` -->

Onboard an application to **simpletel** — get its traces flowing into
`simpletel get traces` — using the `simpletel` CLI and **upstream
OpenTelemetry only**. There is no simpletel SDK and no code lock-in: the only
simpletel-specific facts the app needs are an endpoint URL and an
`x-simpletel-team` header, both produced by `simpletel env`. The app can be
repointed at any other OTLP backend later by changing two environment
variables and nothing in code.

Work through these steps in order. Do not skip the smoke test (step 4): a
green smoke test proves credentials and connectivity *before* any code change,
so a failure there is a login/network problem, not an instrumentation one.

## 0. Check the plan and limits

Before you act, read the account's plan and usage so you know the allowance
you are working within and can state what is left. Run:

```
simpletel usage --format json
```

The JSON is for you, not the human — it is one line you can read directly.

- If it errors, or the document reports `"billing_enabled": false`, note that
  and continue: older and self-hosted servers are fine, and the rest of
  onboarding does not depend on billing.
- If the account is near or at a limit (`records_used` approaching
  `allowance`, or `subscription_status` not active), say so to the human
  plainly and surface the `upgrade_url` from the same document. Better to name
  the ceiling before it rejects a command than after.

**When any command fails with a plan-limit rejection** (HTTP 429 whose body
carries an `upgrade_url`), present that `upgrade_url` to the human and stop
that step — do not retry the same command, and do not loop. The account needs
the plan raised, which is the human's decision, not a transient error to wait
out.

## 1. Confirm a credential

Run `simpletel whoami`. If it reports a logged-in identity, continue. If not,
stop and ask the user which path applies, then run it:

- **Hosted / shared server:** `simpletel login` (GitHub device flow).
- **Local `serve --dev`:** `simpletel login --dev` (writes the well-known
  local-dev credentials; no browser).
- **Operator with a shared admin token:** they will use
  `simpletel team create --admin-token <token>` in step 2 instead of logging
  in.

Do not assume a path — ask.

## 2. Ensure a team

Run `simpletel team list`. If the user has no team (or no current team is
set), create one: `simpletel team create --name <app>`, deriving `<app>` from
the repository or directory name and confirming it with the user first. This
prints the team's 20-hex id and sets it as the current team, so later commands
need no `--team` flag.

## 3. Mint the env block

Run `simpletel env`. It prints the four `OTEL_EXPORTER_OTLP_*` /
`OTEL_SERVICE_NAME` variables for the resolved team and endpoint, with no
network call. Apply them two ways as appropriate:

- **For an immediate run:** export them into the shell —
  `eval "$(simpletel env)"`.
- **For a durable config:** when the project has one, write them to a
  `0600` env file (`.env`), a `docker-compose.yml` `environment:` block, a
  systemd `EnvironmentFile`, or the deployment's secret store.

The header value in `OTEL_EXPORTER_OTLP_HEADERS` is the team's ingest
credential. **Treat the output as secret-bearing: never commit it to source
control and never hard-code the team id in a tracked file.** `simpletel env`
only writes to stdout — you decide where the secret lands.

If `simpletel env` cannot resolve a team, return to step 2.

## 4. Smoke test (before any code change)

Run a trivial command through `simpletel run`, then read it back:

```
simpletel run -- <a-cheap-command, e.g. /bin/sh -c "echo hi">
simpletel get traces
```

Confirm a recent `cli.run` trace appears (its `service.name` is
`simpletel-cli`). This proves the endpoint, the team header, and network
reachability with **zero changes to the app**. Only proceed once the smoke
test is green; a red one is a credentials/connectivity problem to fix first.

## 5. Detect the language and entrypoint

Read the build manifest to identify the language and how the app starts:
`requirements.txt` / `pyproject.toml` (Python), `package.json` (Node),
`pom.xml` / `build.gradle` (Java), `*.csproj` (.NET), `go.mod` (Go),
`Cargo.toml` (Rust). Note the process entrypoint / start command.

## 6. Instrument with upstream OpenTelemetry

Use the **official OpenTelemetry distribution for that language**, configured
**only** through the env block from step 3 — simpletel adds nothing to that
surface. Prefer the zero-code agent where the language has one; it adds
tracing with no source edit.

- **Python:** `pip install opentelemetry-distro opentelemetry-exporter-otlp`,
  then `opentelemetry-bootstrap -a install` (it inspects *installed*
  packages, so it must run after the app's own dependencies), then launch the
  real serving process under `opentelemetry-instrument <start-cmd>`
  (Django/Flask/FastAPI: covers WSGI/ASGI, the ORM and templates). For
  application log records correlated into traces, additionally install
  `opentelemetry-instrumentation-logging` and set `OTEL_LOGS_EXPORTER=otlp`
  and `OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED=true`.
- **Node/JS:** `npm i @opentelemetry/auto-instrumentations-node`, launch with
  `node --require @opentelemetry/auto-instrumentations-node/register <entry>`
  (Express/Fastify/Koa/plain HTTP are covered; the register entrypoint reads
  the env block itself). If the framework's default error handler swallows
  exceptions, record them in the error middleware via the upstream API —
  `const s = trace.getActiveSpan(); s?.recordException(err);
  s?.setStatus({code: SpanStatusCode.ERROR, message: err.message})` — so the
  failure is visible in the trace.
- **Ruby:** `bundle add opentelemetry-sdk opentelemetry-exporter-otlp
  opentelemetry-instrumentation-all`, then one initializer —
  `OpenTelemetry::SDK.configure { |c| c.use_all }` — as
  `config/initializers/opentelemetry.rb` (Rails) or near the top of a
  Sinatra/Rack app; start the app normally (`bin/rails server`,
  `bundle exec ruby app.rb`). Ruby ≥ 3.1 with Bundler.
- **Go:** no zero-code agent — add the smallest upstream bootstrap at the
  process entrypoint: `otlptracehttp.New(ctx)` plus
  `sdktrace.NewTracerProvider(sdktrace.WithBatcher(exporter))`, and
  `defer tp.Shutdown(ctx)` so a short-lived process flushes before exit.
  Wrap HTTP handlers with `otelhttp.NewHandler` and outbound calls with
  `otelhttp.NewTransport`; everything reads the env block.
- **Rust:** no zero-code agent — the `opentelemetry` + `opentelemetry_sdk` +
  `opentelemetry-otlp` crates give the tracer provider; build it once at
  startup and force-flush/shutdown before exit.
- **Java:** download `opentelemetry-javaagent.jar`, launch with
  `java -javaagent:opentelemetry-javaagent.jar -jar <app>`.
- **.NET:** run the OpenTelemetry .NET automatic-instrumentation install
  script and launch through the shim it documents.

Add bootstrap and wiring only; do not sprinkle manual spans through the
codebase — auto/library instrumentation provides the first useful spans, and
manual spans are a follow-up once data is flowing. **Short-lived processes**
(a CLI, a job) must flush explicitly (Go `tp.Shutdown`, Ruby
`OpenTelemetry.tracer_provider.shutdown`, Node the SDK's `shutdown`), or
their last spans never leave the process.

Never write a simpletel-specific exporter or client, and never add a simpletel
package to the app's dependency graph. If the upstream SDK can be pointed at
an OTLP endpoint by env var — and for every language above it can — that is
the integration.

Choose the protocol the endpoint actually serves. The default `http/protobuf`
works against the public HTTP endpoint; only use `simpletel env --protocol grpc`
when the target endpoint exposes OTLP/gRPC (a remote deploy is typically
HTTP-only).

## 7. Verify the app's own spans

Run the real application with the env block applied and **exercise it** (hit
an endpoint, run a request), then verify with the dedicated command:

```
simpletel verify --service <OTEL_SERVICE_NAME>
```

Definition of done is **exit code 0**: a trace newer than the last 10 minutes,
under the expected `service.name`, carrying spans from a library
auto-instrumentation. That is the difference between "the pipe works" and
"the app is instrumented" — not just the `cli.run` wrapper span from the
smoke test. If the app is slow to start, add `--deadline 30s` (or longer);
`simpletel verify` polls until it succeeds or the deadline expires.

On failure, `simpletel verify` reports the rung that did not pass and exits
non-zero. Diagnose from that rung:

| Rung | Meaning | If it fails, check |
|---|---|---|
| **pipe** | any fresh trace arrived at all | endpoint reachability (is the server up?), the team credential (`x-simpletel-team` in `OTEL_EXPORTER_OTLP_HEADERS` from `simpletel env`), and that the app's OTLP exporter points at simpletel. `simpletel run -- <cmd>` is a quick pipe test. |
| **identity** | a fresh trace under `service.name` == `<OTEL_SERVICE_NAME>` | `OTEL_SERVICE_NAME` was actually exported as the expected value (the command prints the service names it saw), and the env block was applied to the exporter process, not a parent shell that the server doesn't inherit. |
| **richness** | a matching trace has spans from a library auto-instrumentation | auto-instrumentation is not engaged — install the instrumentation packages for the app's dependencies, wrap the real serving process with the zero-code agent, and exercise real HTTP traffic (an idle server emits nothing). The failure output lists the scope names seen on the inspected trace: a name from your language's upstream family (step 6) — `opentelemetry.instrumentation.*`, `@opentelemetry/instrumentation-*`, `OpenTelemetry::Instrumentation::*`, `go.opentelemetry.io/contrib/instrumentation/...` — means the library is engaged; `simpletel-cli`, your own tracer name, or an empty list means it is not. Batch exporters flush on a delay (~5s), so wait a few seconds before concluding it is broken. |

For a **Python/Django** app whose richness rung fails, in order:

1. Run `opentelemetry-bootstrap -a install` **after** `pip install -r
   requirements.txt` — it inspects *already-installed* packages, so running
   it first installs nothing.
2. Wrap the **real serving process** with `opentelemetry-instrument` (the
   `gunicorn`/`uvicorn` command or `manage.py runserver`), not a launcher
   shell script that spawns workers outside the wrapper.
3. Make a real HTTP request to the running app — Django spans appear on an
   actual request through the WSGI middleware, not while the server sits idle.
4. Batch exporters flush on a ~5s schedule, so wait before deciding it is
   broken.

The same shape applies elsewhere: **Node** — confirm the `--require
.../register` flag wraps the real entrypoint (not a launcher script) and,
when the framework handles its own errors, that the error middleware records
them (step 6); **Ruby** — confirm `c.use_all` ran before the first request
(Rails: the initializer) and the rack middleware is in the stack; **Go** —
confirm the server runs `otelhttp.NewHandler` and that `tp.Shutdown` runs on
exit (a batch that never flushes looks exactly like no instrumentation).

## 8. Hand off to the human

`simpletel verify` exiting 0 is the end of the work, not the end of the
onboarding. Close by printing one fixed summary so the human has something
concrete, in this order:

- **Team id** — the resolved team's 20-hex id (`simpletel team list`, or the
  `x-simpletel-team` header value from `simpletel env`).
- **Service name** — the `OTEL_SERVICE_NAME` the app now exports under.
- **Verification** — that `simpletel verify` succeeded, plus the first
  verified trace id from its `verified: service=… trace=…` line.
- **Console URL** — from `simpletel console` (or the same web console in a
  browser), where the human can watch traces land.

Then add the one-line "what to do next": open the console's errors-only view
to catch problems early, and — when something breaks — work the
`simpletel-troubleshoot` skill (or the `simpletel mcp` tools) from the
debugging agent: find the failing trace, read its error span and correlated
logs, and map it back to the code. Report those four fields together — never
leave the human with just "it worked".
