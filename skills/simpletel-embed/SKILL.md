---
name: simpletel-embed
description: Embed simpletel inside an application — spawn the simpletel-embed sidecar via a language adapter (Go, Django, Rust), mount the trace viewer in the app, and verify the round trip. Use when the user asks to "embed simpletel in <app>", "add an in-app trace viewer", "run simpletel as a sidecar", or "set up the embedded harness".
---
<!-- managed by: simpletel skill install (20261009.1952.16+ae9a795) — do not hand-edit; regenerate with `simpletel skill install` or `simpletel skill package` -->

Embed [simpletel](https://simpletel.dev) **inside** an
application — single-tenant OTLP ingest, storage, and a trace viewer served
from the app's own origin via a per-language adapter that supervises one
shared sidecar binary (`simpletel-embed`). Use this when the user asks to
"embed simpletel in <app>", "add an in-app trace viewer", "run simpletel as a
sidecar", or wants telemetry inside a single-server app without operating a
separate backend.

There is exactly one integration pattern for every language: **an adapter
package spawns the same `simpletel-embed` binary, polls `/healthz`, mounts the
viewer behind the app's origin via reverse proxy, and signs an HMAC user
assertion per request.** Ingest is tokenless over loopback. Work through the
steps in order; do not skip the verification step.

## 1. Locate or build the sidecar binary

The adapter needs the `simpletel-embed` binary. Check `simpletel-embed` on
PATH or a `bin/simpletel-embed` in the simpletel repo checkout. If neither
exists, build it:

```
go build -o bin/simpletel-embed ./cmd/simpletel-embed
```

The adapter locates the binary by: explicit config option → `SIMPLETEL_EMBED_BINARY`
env var → `simpletel-embed` on PATH. For deterministic tests always set
`SIMPLETEL_EMBED_BINARY` to the specific path you built.

## 2. Pick the adapter for the host language

| Language | Adapter | Install |
|---|---|---|
| Go | `github.com/willnewby/simpletel/adapters/go` | import from the simpletel module |
| Python / Django | `django-simpletel` | `pip install -e adapters/python-django` |
| Rust | `simpletel_embed` crate | path dependency on `adapters/rust` |

All three expose the same options in native form: data path, shared secret,
binary path override, viewer mount path (default `/simpletel/`), buffer size,
and the shutdown flush deadline (default 10s).

## 3. Generate a shared secret and wire the adapter

The secret signs viewer assertions; the adapter holds it and passes it to the
sidecar at spawn. Generate once (e.g. `openssl rand -hex 32`) and treat it as
app config — never commit it.

**Go** — plain struct, spawn on startup, mount on the app mux:

```go
rt, err := simpleteladapter.Start(ctx, simpleteladapter.Config{
    DataPath: "./data/simpletel", Secret: secret, ServiceName: appName,
})
defer rt.Stop()
mux.Handle("/simpletel/", rt.Viewer(userID, userName)) // proxy + assertion
```

**Django** — settings dict, adapter runtime, mountable URLconf:

```python
SIMPLETEL = {"enabled": True, "mount": "simpletel/", "secret": secret,
             "data": str(BASE_DIR / "data/simpletel"),
             "shutdown_flush_timeout": 10, "binary": os.environ.get("SIMPLETEL_EMBED_BINARY")}
# apps.py ready(): runtime.ensure_running(settings.SIMPLETEL)
# urls.py: path("simpletel/", include("django_simpletel.urls"))
```

**Rust** — builder, spawn + health poll, then nest the viewer service:

```rust
let st = SimpleTel::builder().data_path("./data/simpletel")
    .secret(secret).spawn().await?;
let app = Router::new().nest_service("/simpletel", st.viewer(user.clone()));
```

The adapter spawns the sidecar with the same flags in every language
(`--data --http 127.0.0.1:4318 --secret --shutdown-flush-timeout …`), waits
for `/healthz`, and proxies the viewer with prefix stripping. Never edit the
sidecar's flags per language — parity is the contract.

## 4. Point the app's OpenTelemetry SDK at the sidecar

Standard upstream OTLP, no team header, loopback:

```
OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:4318
OTEL_SERVICE_NAME=<app-name>
```

The runtime ignores `x-simpletel-team` and attributes everything to its one
seeded tenant. The runtime owns its SQLite file (`--data`) and never touches
the host app's database.

## 5. Verify the round trip

From the simpletel repo (full matrix): `just verify` — or a single language:
`just verify-go` / `just verify-django` / `just verify-rust`. Each runs the
installed adapter's tests against the specific-path binary.

From inside the host app, prove all four contract points:

1. `curl -f http://127.0.0.1:4318/healthz` → `ok` (sidecar up).
2. Export a test trace from the app → HTTP 200, tokenless.
3. Viewer without an assertion → "Sign in to view traces" empty state, never
   data; with the adapter's signed assertion (a logged-in app user) → the
   trace renders at `/simpletel/traces`.
4. Stop the app (Ctrl-C) → adapter SIGTERMs the sidecar → bounded flush
   (default 10s) → sidecar exits; `pgrep -x simpletel-embed` finds nothing.

## Troubleshooting

- **Empty state with a logged-in user** → assertion invalid or expired
  (`exp` is unix seconds, ~5 min skew allowed) or the secret differs between
  adapter and sidecar. Fail-closed by design.
- **Ingest 400** → wrong Content-Type (send `application/json` or
  `application/x-protobuf`); sidecar accepts both.
- **Sidecar won't start** → port taken (pass a different `--http` via the
  adapter's `IngestAddr`/`ingest_addr` option) or data path not writable.
- **Leaked processes after crashes** → kill leftover `simpletel-embed`
  manually; adapters clean up on graceful exits only.
