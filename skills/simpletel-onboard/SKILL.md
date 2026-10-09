---
name: simpletel-onboard
description: Onboard an application to simpletel — instrument it with upstream OpenTelemetry and send its traces to simpletel. Use when the user asks to "onboard <app> to simpletel", "instrument <app> with OpenTelemetry", or "send traces to simpletel".
---
<!-- managed by: simpletel skill install (20261009.1952.16+ae9a795) — do not hand-edit; regenerate with `simpletel skill install` or `simpletel skill package` -->

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
  `opentelemetry-bootstrap -a install`, then launch under
  `opentelemetry-instrument <start-cmd>`.
- **Node/JS:** `npm i @opentelemetry/api @opentelemetry/auto-instrumentations-node`,
  launch with
  `node --require @opentelemetry/auto-instrumentations-node/register <entry>`.
- **Java:** download `opentelemetry-javaagent.jar`, launch with
  `java -javaagent:opentelemetry-javaagent.jar -jar <app>`.
- **.NET:** the OpenTelemetry .NET automatic-instrumentation install script
  and its launch shim.
- **Go / Rust (no zero-code agent):** add the smallest upstream SDK bootstrap
  from that language's getting-started guide — for Go, the
  `otel` + `otlptracehttp` + `sdk/trace` setup that reads the
  `OTEL_EXPORTER_OTLP_*` env vars — placed at the process entrypoint and shut
  down (flushed) on exit. Add bootstrap and wiring only; do not sprinkle
  manual spans through the codebase. Auto/library instrumentation provides the
  first useful spans; manual spans are a follow-up once data is flowing.

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
| **richness** | a matching trace has spans from a library auto-instrumentation | auto-instrumentation is not engaged — install the instrumentation packages for the app's dependencies, wrap the real serving process with the zero-code agent, and exercise real HTTP traffic (an idle server emits nothing). Batch exporters flush on a delay (~5s), so wait a few seconds before concluding it is broken. |

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
to catch problems early. Report those four fields together — never leave the
human with just "it worked".
