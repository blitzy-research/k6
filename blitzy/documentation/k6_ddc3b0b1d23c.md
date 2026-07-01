# k6 Runtime Behavior Investigation — commit `ddc3b0b1d23c` (v0.55.0)

This document answers five questions about the internal runtime behavior of the
[Grafana k6](https://github.com/grafana/k6) load-testing engine
(`go.k6.io/k6`, source branch `k6_ddc3b0b1d23c`, commit
`ddc3b0b1d23c128e34e2792fc9075f9126e32375`, **k6 v0.55.0**). Every answer is
**evidence-first**: k6 was built from source and each behavior (R1–R5) was
provoked by a purpose-built temporary test script, run while capturing verbatim
output. Each answer shows the exact test script, the exact run command, the
verbatim observed output (logs, metric values, HTTP responses/headers, memory
measurements, exported metric names), the precise `file:line` source citations
that explain the behavior, a reasoned answer to every sub-question, and a
coverage checklist. The investigation is strictly **read-only**: no repository
source file was modified — all test scripts and helper binaries lived under
`/tmp`, outside the repository tree.

---

## Setup / Reproduction

### Build provenance

The k6 binary under test was compiled from source with the Go toolchain of the
build environment, **Go 1.23.10**, under `GOTOOLCHAIN=local`. Note that `go.mod`
declares `toolchain go1.21.13`; with `GOTOOLCHAIN=local` the `go` command uses
the locally installed 1.23.10 instead of fetching the pinned version. The k6
source is byte-identical regardless of which of these toolchains compiles it, so
the observed runtime behavior (R1–R5) is unaffected — only the
`runtime.Version()` token embedded in the version string differs (see
`lib/consts/consts.go`, where `FullVersion()` derives the Go token from
`runtime.Version()` and the commit token from the build's `vcs.revision`).
Because the version string's commit token is the git `HEAD` at build time, the
binary is built from a clean checkout at the source commit
`ddc3b0b1d23c128e34e2792fc9075f9126e32375` so that `commit/ddc3b0b1d2` is stamped
in. Reproduction commands:

```
# Go 1.23.10 is the build toolchain (GOTOOLCHAIN=local; go.mod pins
# toolchain go1.21.13, but the local toolchain is used — k6 behavior is identical).
go version                       # -> go version go1.23.10 linux/amd64

# Build from a clean checkout at the source commit so the version string embeds
# commit/ddc3b0b1d2 (vendor mode, fully offline; `make build` wraps `go build`).
git checkout ddc3b0b1d23c128e34e2792fc9075f9126e32375
export GOFLAGS=-mod=vendor GOPROXY=off
go build -o /tmp/k6bin .
```

Verified binary identity (the version-string prefix is the output binary's
basename, `k6bin`; the commit token is the source commit; the Go token is the
build toolchain reported by `runtime.Version()`):

```
$ /tmp/k6bin version
k6bin v0.55.0 (commit/ddc3b0b1d2, go1.23.10, linux/amd64)
```

### Read-only guarantee

All test scripts and helper binaries were created **outside** the repository
tree (under `/tmp`). `git status --porcelain` was empty before and after every
run — no k6 source, test, config, build, or `vendor/` file was touched. The
runtime investigation was performed against **source commit
`ddc3b0b1d23c128e34e2792fc9075f9126e32375`** (k6 v0.55.0). The **only** change
this task makes to the repository is adding **this document**; committing it
advances the destination branch's `HEAD` by exactly that single documentation
commit, so the diff from the source commit to that commit is precisely
`A blitzy/documentation/k6_ddc3b0b1d23c.md` and nothing else.

### Environment caveat (applies to R3 & R4)

The observation host had ~3.8 TiB RAM and no swap, and `/usr/bin/time -v` was
unavailable, so **peak memory was sampled from `/proc/<pid>/status` `VmHWM`**.
Consequently:

- **R3**: the number of iterations that complete within the capacity window (and
  therefore the exact drop count) is **timing-dependent**; the invariant is
  `dropped = requested_total − completed`.
- **R4**: the dataset was sized to `N=25000` rows — a bounded size held identical
  across the SharedArray and plain-array scripts so the two are directly
  comparable. The reported RSS figures are approximate (`VmHWM` sampling) and
  drift a few MB run-to-run; the **qualitative** behavior (constant vs. linear
  growth) is what is being demonstrated and is independent of `N`.

---

## R1 — VU lifecycle on SIGINT

### 1. Question (verbatim)

> What are the exact log messages that appear when a ramping executor with at
> least 5 VUs receives a sigint. While the system is shutting down, what is the
> specific log evidence that shows whether currently active VUs are allowed to
> finish their current iteration or are terminated mid execution? You need to
> give me the exact log outputs to show runtime evidence.

### 2. Test script (`/tmp/obs/r1_ramp_sigint.js`)

```js
import { sleep } from 'k6';

export const options = {
  scenarios: {
    ramp: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '4s', target: 8 },
        { duration: '60s', target: 8 },
      ],
      gracefulRampDown: '30s',
      gracefulStop: '30s',
    },
  },
};

export default function () {
  sleep(30);
}
```

### 3. Exact run command

SIGINT was delivered at ≈ t=8 s, while all 8 VUs were mid-`sleep(30)`:

```
/tmp/k6bin run -v /tmp/obs/r1_ramp_sigint.js
# (SIGINT sent to the k6 process ~8s in, i.e. `kill -INT <pid>`)
```

### 4. Verbatim observed output

`-v` is **required** — the signal-response lines are `level=debug`:

```
level=debug msg="k6 version: v0.55.0 (commit/ddc3b0b1d2, go1.23.10, linux/amd64)"
     execution: local
     scenarios: (100.00%) 1 scenario, 8 max VUs, 1m34s max duration (incl. graceful stop):
              * ramp: Up to 8 looping VUs for 1m4s over 2 stages (gracefulRampDown: 30s, gracefulStop: 30s)
level=debug msg="Starting the REST API server on localhost:6565"
level=debug msg="Trapping interrupt signals so k6 can handle them gracefully..."
running (0m07.0s), 8/8 VUs, 0 complete and 0 interrupted iterations
level=debug msg="Stopping k6 in response to signal..." sig=interrupt
level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
running (0m08.0s), 0/8 VUs, 0 complete and 8 interrupted iterations
level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

Measured (reported exactly):

- **Exit code = 105.**
- **Elapsed from SIGINT to process exit ≈ 0.032 s** (the 30-second sleeps did
  **not** run to completion).
- Pre-signal footer: `running (0m07.0s), 8/8 VUs, 0 complete and 0 interrupted iterations`
  (all 8 VUs active).
- Decisive final footer: `running (0m08.0s), 0/8 VUs, 0 complete and 8 interrupted iterations`.

### 5. Source citations

- `cmd/common.go:97-101` — `handleTestAbortSignals` logs
  `"Trapping interrupt signals so k6 can handle them gracefully..."` (line 98)
  and registers `os.Interrupt, syscall.SIGINT, syscall.SIGTERM` via
  `gs.SignalNotify(...)` (line 101).
- `cmd/run.go:349-360` — the `gracefulStop` handler logs
  `"Stopping k6 in response to signal..."` with the `sig` field (line 350) and
  builds the abort error
  `fmt.Errorf("test run was aborted because k6 received a '%s' signal", sig)`
  (line 354); `onHardStop` logs `"Aborting k6 in response to signal"` (line 360).
  *(Verified on-disk: the `fmt.Errorf` literal is at line 354 and the `onHardStop`
  error log is at line 360.)*
- `lib/executor/base_config.go:91-96` — doc comment on `GetGracefulStop`: the
  graceful window is how long k6 waits at the **end of the normal executor
  duration** before killing still-running iterations; "*that doesn't count when
  the user manually interrupts the test, then iterations are immediately
  stopped*". The graceful window therefore does **not** apply to a SIGINT abort.
- `lib/executor/ramping_vus.go:19,44` — `const rampingVUsType = "ramping-vus"`
  (line 19) and the `GracefulRampDown` field (line 44).
- `lib/executor/vu_handle.go:161,177` — the `"Graceful stop"` (line 161) and
  `"Hard stop"` (line 177) debug logs (the graceful-window mechanism used at
  end-of-stage/ramp-down, not for SIGINT).
- `lib/execution.go:280-304` — full vs. partial (interrupted) iteration
  accounting: `GetFullIterationCount` (line 284) and `GetPartialIterationCount`
  (line 300).
- `execution/scheduler.go:155-159` — the footer format string
  `"%s, "+vusFmt+"/"+vusFmt+" VUs, %d complete and %d interrupted iterations"`
  (line 156).

### 6. Reasoned answer

- **Exact log messages on SIGINT:** the six lines shown above — the debug
  `Trapping interrupt signals…`, `Stopping k6 in response to signal… sig=interrupt`,
  the `execution-scheduler-run` "interrupted" line, `Test finished with an error`,
  `Everything has finished, exiting k6 with an error!`, and the final
  `level=error` `test run was aborted because k6 received a 'interrupt' signal`.
- **Finish current iteration vs. terminated mid-execution?** **Terminated
  mid-execution.** The decisive evidence is the footer transition from
  `8/8 VUs, 0 complete and 0 interrupted` to
  `0/8 VUs, 0 complete and 8 interrupted iterations` — **0 complete, 8
  interrupted**. If the active VUs had been allowed to finish their in-flight
  iteration, they would be counted as *complete*. Instead all 8 are *interrupted*.
  Additionally, k6 exited **~0.03 s** after the signal (exit code **105**), far
  shorter than the in-flight `sleep(30)`, proving the iterations were aborted,
  not completed. This matches the source semantics at
  `lib/executor/base_config.go:91-96` (a manual interrupt ⇒ iterations are
  immediately stopped).
- **Why `-v`?** The `Stopping k6 in response to signal…` line and the related
  shutdown lines are `level=debug` (emitted at `cmd/run.go:350`), so verbose mode
  is required to observe them.

### 7. Coverage checklist (R1)

- [x] Exact SIGINT log messages provided (the six verbatim lines).
- [x] Evidence of mid-execution termination (`0 complete and 8 interrupted`).
- [x] Confirmation VUs did **not** finish (exit code 105; ~0.03 s vs. `sleep(30)`).
- [x] Verbose-mode (`-v`) requirement noted (signal lines are `level=debug`).

---

## R2 — gRPC server-streaming interrupt

### 1. Question (verbatim)

> I wonder what are the exact log entries obtained at runtime when a grpc server
> streaming test with a 30ms graceful ramp down is interrupted. Also tell me what
> is the value of number of grpc messages received reported by final metrics
> summary.

### 2. Test script (`/tmp/obs/r2_grpc_streaming.js`)

The `route_guide.proto` from `lib/testutils/grpcservice/` was copied next to the
script; `client.load()` uses a **script-relative** path (k6's `client.load()`
resolves proto paths relative to the script directory, so a script-relative
filename is required).

```js
import { Client, Stream } from 'k6/net/grpc';
import { sleep } from 'k6';

const client = new Client();
client.load([], 'route_guide.proto');

export const options = {
  scenarios: {
    server_streaming: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1s', target: 5 },
        { duration: '2s', target: 5 },
        { duration: '1s', target: 0 },
      ],
      gracefulRampDown: '30ms',
    },
  },
};

export default function () {
  client.connect('127.0.0.1:10000', { plaintext: true });
  const stream = new Stream(client, 'main.FeatureExplorer/ListFeatures', null);
  stream.on('data', function (feature) {});
  stream.on('error', function (e) {
    console.log('STREAM_ERROR ' + JSON.stringify(e));
  });
  stream.on('end', function () {
    client.close();
  });
  stream.write({
    lo: { latitude: 400000000, longitude: -750000000 },
    hi: { latitude: 420000000, longitude: -730000000 },
  });
  sleep(30);
}
```

### 3. Exact run command

The bundled gRPC server (`examples/grpc_server`) was copied **out of the repo**
to `/tmp/grpcserver` (its `replace go.k6.io/k6 => <repo path>` adjusted) and built
with `-mod=mod` (per the Makefile's `grpc-server-run` target) to
`/tmp/grpcserver_bin`, then run listening on `localhost:10000`. The repo's own
`examples/grpc_server/go.mod` was never touched.

```
# gRPC server already running on localhost:10000 (bundled examples/grpc_server, built out-of-repo with -mod=mod)
/tmp/k6bin run -v /tmp/obs/r2_grpc_streaming.js
```

### 4. Verbatim observed output

**Per-VU interruption sequence** — the final `1s → 0` ramp-down gives each VU
only 30 ms to finish, forcing cancellation (sample for `vuNum=4`, repeated per VU):

```
time="2026-07-01T09:00:57Z" level=debug msg="Graceful stop" executor=ramping-vus scenario=server_streaming vuNum=4
time="2026-07-01T09:00:57Z" level=debug msg="Hard stop" executor=ramping-vus scenario=server_streaming vuNum=4
time="2026-07-01T09:00:57Z" level=debug msg="stream is cancelled/finished" error="canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-01T09:00:57Z" level=debug msg="stream /main.FeatureExplorer/ListFeatures is closing" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-01T09:00:57Z" level=info msg="STREAM_ERROR {\"code\":2,\"details\":[],\"message\":\"canceled by client (k6)\"}" source=console
```

A few VUs also logged variant console errors such as
`"message":"context canceled at file:///tmp/obs/r2_grpc_streaming.js:25:21(0)"`
and, at the very end, `{"code":4,...,"message":"context deadline exceeded"}` —
these are observed variation, but the canonical cancellation cause is
`canceled by client (k6)`.

**Final metrics summary + footer:**

```
running (3.0s), 5/5 VUs, 0 complete and 0 interrupted iterations
     grpc_streams.................: 5      1.240301/s
     grpc_streams_msgs_received...: 150    37.20904/s
     grpc_streams_msgs_sent.......: 5      1.240301/s
running (4.0s), 0/5 VUs, 0 complete and 5 interrupted iterations
```

Scenario line observed:
`scenarios: (100.00%) 1 scenario, 5 max VUs, 4s max duration (incl. graceful stop):`

### 5. Source citations

- `js/modules/k6/grpc/metrics.go:25` — `grpc_streams_msgs_received` is defined via
  `registry.NewMetric("grpc_streams_msgs_received", metrics.Counter)`.
- `js/modules/k6/grpc/stream.go:149-159` — `queueMessage` pushes the counter for
  each received message with `Value: 1`; `:201` logs
  `"stream is cancelled/finished"`; `:371` logs `"stream %s is closing"`.
- `lib/netext/grpcext/stream.go:28-29` — **defines** the literal:
  `// ErrCanceled canceled by client (k6)` (line 28) /
  `var ErrCanceled = errors.New("canceled by client (k6)")` (line 29). This is the
  source of `error="canceled by client (k6)"`; the k6/grpc module logs it at
  `js/modules/k6/grpc/stream.go:201`.
- `lib/executor/vu_handle.go:161,177` — the `"Graceful stop"` (line 161) →
  `"Hard stop"` (line 177) pair (the 30 ms `gracefulRampDown` window expiring).
- `lib/testutils/grpcservice/service.go:57-63` — `ListFeatures` sleeps
  `100 * time.Millisecond` (line 62) then `stream.Send(feature)` (line 63) — one
  feature per 100 ms.
- `examples/grpc_server/main.go:51,57-82` — the bundled server: the default
  listen port `flag.Int("port", 10000, "The server port")` (line 51), the startup
  log `"gRPC server starting on localhost:%d"` (line 57),
  `net.Listen("tcp", fmt.Sprintf("localhost:%d", *port))` (line 59), then
  `RegisterFeatureExplorerServer(...)` (line 80) — the `main.FeatureExplorer`
  service the test streams from — followed by `grpcServer.Serve(lis)` (line 82).
  Built/run with `-mod=mod`.
- `examples/grpc_server_streaming.js:1,9,14,17,19,21-30,38-41` — the reference
  client: imports `Client, Stream` from `k6/net/grpc` (line 1), the default
  `GRPC_ADDR = __ENV.GRPC_ADDR || '127.0.0.1:10000'` (line 9), `client.load(...)`
  (line 14), `client.connect(GRPC_ADDR, { plaintext: true })` (line 17), and — the
  key line — `new Stream(client, 'main.FeatureExplorer/ListFeatures', null)`
  (line 19), with the `stream.on('data', ...)` (lines 21-30) and
  `stream.on('error', ...)` (lines 38-41) handlers.

### 6. Reasoned answer

- **Exact runtime log entries on interruption:** the per-VU sequence
  `Graceful stop` → `Hard stop` → `stream is cancelled/finished`
  (`error="canceled by client (k6)"`) → `stream …/ListFeatures is closing` → the
  console `STREAM_ERROR {"code":2,…,"message":"canceled by client (k6)"}`. The
  `Graceful stop` → `Hard stop` gap is the **30 ms** `gracefulRampDown` window
  closing (`lib/executor/vu_handle.go:161,177`), after which the open stream is
  cancelled (`js/modules/k6/grpc/stream.go:201`, cause defined at
  `lib/netext/grpcext/stream.go:29`).
- **Value of gRPC messages received in the final summary:**
  **`grpc_streams_msgs_received = 150`** (rate `37.20904/s`). This is 5 streams ×
  ~30 features each received before the 30 ms ramp-down cancelled them (the server
  sends one feature per 100 ms — `service.go:62-63`). The counter is still
  reported in the summary even though the footer shows
  **`0 complete and 5 interrupted iterations`** (all iterations were interrupted).

### 7. Coverage checklist (R2)

- [x] Exact interruption log entries provided.
- [x] 30 ms graceful-ramp-down mechanism identified (`Graceful stop` → `Hard stop`).
- [x] `grpc_streams_msgs_received = 150` read from the final summary.
- [x] Noted the counter is reported despite all-interrupted iterations.

---

## R3 — dropped iterations via the REST API

### 1. Question (verbatim)

> I wonder what is the exact value of dropped iterations that is reported when a
> test exceeds its maximum duration capacity. I want you to report this value by
> querying the api. Give me runtime evidence to prove that the values were
> reported by querying the api.

### 2. Test script (`/tmp/obs/r3_dropped.js`)

```js
import { sleep } from 'k6';

export const options = {
  scenarios: {
    over_capacity: {
      executor: 'shared-iterations',
      vus: 5,
      iterations: 1000,
      maxDuration: '3s',
    },
  },
};

export default function () {
  sleep(1);
}
```

### 3. Exact run command

`--linger` keeps the REST API on `localhost:6565` alive after the test so it can
be queried:

```
/tmp/k6bin run --linger -v /tmp/obs/r3_dropped.js &      # run in background
curl -sS -D - http://localhost:6565/v1/metrics           # query the REST API
```

### 4. Verbatim observed output

**REST API response** (headers + JSON:API body — this is the PROOF the value
came from the API, not the summary):

```
$ curl -sS -D - http://localhost:6565/v1/metrics
HTTP/1.1 200 OK
Date: Wed, 01 Jul 2026 09:15:14 GMT
Content-Length: 1067
Content-Type: text/plain; charset=utf-8

{"data":[ ... ,{"type":"metrics","id":"iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":15,"rate":4.9950137408532305}}},{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":985,"rate":328.00590231602877}}}]}
```

The single-metric endpoint was also confirmed:
`curl -sS -D - http://localhost:6565/v1/metrics/dropped_iterations` →
`HTTP/1.1 200 OK`, `Content-Length: 170`, `sample.count = 985`.

**Server-side proof the API was hit** (from the k6 `-v` log):

```
level=debug msg="GET /v1/metrics" status=200
level=debug msg="GET /v1/metrics/dropped_iterations" status=200
level=debug msg="The test is done, but --linger was enabled, so k6 is waiting for Ctrl+C to continue..."
```

For contrast, the end-of-test summary printed the same value
(`dropped_iterations...: 985 328.005902/s`) with footer
`running (03.0s), 0/5 VUs, 15 complete and 0 interrupted iterations` and progress
`over_capacity ✗ … 0015/1000 shared iters` — but the **reported value comes from
the API query above**, as the user required.

### 5. Source citations

- `metrics/builtin.go:10,84` — `DroppedIterationsName = "dropped_iterations"`
  (line 10) and its `Counter` registration
  `registry.MustNewMetric(DroppedIterationsName, Counter)` (line 84).
- `lib/executor/shared_iterations.go:218-226` — on completion, if
  `attemptedIters < totalIters` (line 219), it pushes the `DroppedIterations`
  metric (line 222) with `Value: float64(totalIters - attemptedIters)` (line 225).
- `api/v1/routes.go:23` — the `GET /v1/metrics` route
  (`mux.HandleFunc("/v1/metrics", ...)`).
- `api/v1/metric_routes.go:9-24` — `handleGetMetrics` (line 9) marshals observed
  metrics as JSON:API: `newMetricsJSONAPI(cs.MetricsEngine.ObservedMetrics, t)`
  (line 16) → `json.Marshal(metrics)` (line 19).

### 6. Reasoned answer

- **Exact `dropped_iterations` value:** **`985`** (rate `328.00590231602877`). The
  scenario requested 1000 iterations but only **15** could complete within the
  `maxDuration: '3s'` capacity (5 VUs × `sleep(1)`), so
  `dropped_iterations = 1000 − 15 = 985` (`lib/executor/shared_iterations.go:218-226`).
  **This value is timing-dependent** — a run that completes a slightly different
  number of iterations yields `1000 − completed`; the invariant is
  `dropped = requested_total − completed`. (In this environment the completed
  count was a stable **15** across repeated runs, giving `985` every time.)
- **Reported by querying the API (proof):** yes — the value was read from
  `GET http://localhost:6565/v1/metrics`, which returned **`HTTP/1.1 200 OK`** and
  the JSON:API object
  `{"type":"metrics","id":"dropped_iterations",…"sample":{"count":985,…}}`
  (`api/v1/metric_routes.go:9-24`, `api/v1/routes.go:23`). The k6 server log
  independently confirms `GET /v1/metrics status=200`. `--linger` kept the API
  alive so the recorded value could be fetched after the test ended.

### 7. Coverage checklist (R3)

- [x] Exact `dropped_iterations = 985` reported.
- [x] Derived from an over-capacity run (`1000 − 15`).
- [x] Obtained via REST API (`GET /v1/metrics`, `HTTP/1.1 200 OK`, JSON:API body).
- [x] Runtime proof of the API query (server-side `status=200` log + `curl -D -` headers).
- [x] Timing-dependence noted (`dropped = requested_total − completed`).

---

## R4 — SharedArray memory behavior and root cause

### 1. Question (verbatim)

> Does the memory footprint for the files remain constant with increasing number
> of VUs, or is each VU creating its own copy of the data? Give me test script
> output to prove this behavior. What is the root cause of this behavior.

### 2. Test scripts

Two scripts build an **identical 25,000-record dataset** — one via `SharedArray`,
one as a plain module-level array.

`/tmp/obs/r4_shared.js`:

```js
import { SharedArray } from 'k6/data';
import { sleep } from 'k6';

const N = 25000;
const data = new SharedArray('users', function () {
  console.log('R4_MARKER: SharedArray constructor building dataset ONCE');
  const arr = [];
  for (let i = 0; i < N; i++) {
    arr.push({ id: i, name: 'user_' + i, email: 'user_' + i + '@example.com', bio: 'x'.repeat(80) });
  }
  return arr;
});

export const options = {
  scenarios: {
    s: { executor: 'constant-vus', vus: __ENV.VUS ? parseInt(__ENV.VUS) : 1, duration: '3s' },
  },
};

export default function () {
  if (data[0].id === -1) console.log('never');
  sleep(1);
}
```

`/tmp/obs/r4_plain.js`:

```js
import { sleep } from 'k6';

const N = 25000;
const data = (function () {
  const arr = [];
  for (let i = 0; i < N; i++) {
    arr.push({ id: i, name: 'user_' + i, email: 'user_' + i + '@example.com', bio: 'x'.repeat(80) });
  }
  return arr;
})();

export const options = {
  scenarios: {
    s: { executor: 'constant-vus', vus: __ENV.VUS ? parseInt(__ENV.VUS) : 1, duration: '3s' },
  },
};

export default function () {
  if (data[0].id === -1) console.log('never');
  sleep(1);
}
```

Marker script `/tmp/obs/r4_marker.js` (proves the builder runs once regardless of
VU/iteration count):

```js
import { SharedArray } from 'k6/data';
import { sleep } from 'k6';

const data = new SharedArray('users', function () {
  console.log('R4_MARKER: SharedArray constructor invoked — building dataset');
  const arr = [];
  for (let i = 0; i < 1000; i++) arr.push({ id: i, v: 'row_' + i });
  return arr;
});

export const options = {
  scenarios: {
    s: { executor: 'shared-iterations', vus: 10, iterations: 50 },
  },
};

export default function () {
  const _ = data[Math.floor(Math.random() * data.length)].id;
  sleep(0.01);
}
```

### 3. Exact run commands

Peak RSS was sampled from `/proc/<pid>/status` `VmHWM` via a small wrapper, since
`/usr/bin/time -v` was unavailable:

```
# measure.sh: runs k6 in background and tracks max VmHWM until exit
#   VUS="$2" /tmp/k6bin run --quiet "$SCRIPT" & then poll /proc/$PID/status VmHWM
for V in 1 50 100; do ./measure.sh r4_shared.js $V r4_shared_$V.log; done
for V in 1 50 100; do ./measure.sh r4_plain.js  $V r4_plain_$V.log;  done
/tmp/k6bin run r4_marker.js            # marker run (10 VUs, 50 iterations)
```

### 4. Verbatim observed output

**Peak RSS (`VmHWM`) matrix** (scaled measurements — see the environment caveat):

| VUs | SharedArray peak RSS | Plain per-VU array peak RSS |
|-----|----------------------|-----------------------------|
| 1   | **83 MB** (84760 KB) | **80 MB** (82144 KB)        |
| 50  | **88 MB** (90104 KB) | **1212 MB** (1240840 KB)    |
| 100 | **89 MB** (91148 KB) | **2307 MB** (2362104 KB)    |

k6 run confirmations, e.g. for the SharedArray script each run logged the marker
exactly once and reported the expected iteration counts:

```
# r4_shared.js VUS=50
level=info msg="R4_MARKER: SharedArray constructor building dataset ONCE" source=console
     iterations...........: 150 49.966521/s
     vus..................: 50  min=50      max=50
```

**Single-build marker proof** (`r4_marker.js`, 10 VUs / 50 iterations):

```
level=info msg="R4_MARKER: SharedArray constructor invoked — building dataset" source=console
     scenarios: (100.00%) 1 scenario, 10 max VUs, 10m30s max duration (incl. graceful stop):
              * s: 50 iterations shared among 10 VUs (maxDuration: 10m0s, gracefulStop: 30s)
     iterations...........: 50  967.538989/s
running (00m00.1s), 00/10 VUs, 50 complete and 0 interrupted iterations
```

`grep -c "R4_MARKER" r4_marker.log` → **1** (the builder ran **exactly once**
across all 10 VUs and 50 iterations).

### 5. Source citations

- `js/modules/k6/data/data.go:21-23,31-34` — the root module holds a single
  `shared sharedArrays` field (line 22) whose type is
  `sharedArrays struct { data map[string]sharedArray; mu sync.RWMutex }`
  (lines 31-34).
- `js/modules/k6/data/data.go:53-58` — every VU's module instance is constructed
  with `shared: &rm.shared` (line 56) — the **same** map for all VUs.
- `js/modules/k6/data/data.go:73-97` — the `SharedArray` constructor calls
  `d.shared.get(rt, name, fn)` (line 95).
- `js/modules/k6/data/data.go:152-167` — `get` **caches by name**: it reads any
  existing entry (line 154) and, only when the name is absent, builds it once
  under a write lock (`if !ok { … s.data[name] = array }`, lines 160-162) — so the
  dataset is built at most once per name.
- `js/modules/k6/data/data.go:113-115` — the source comment states the
  implementation "relies on maintaining *a single instance of arrays for the
  whole test setup and VUs*" (line 115).
- `js/modules/k6/data/share.go:10-12` — the backing store
  `sharedArray struct { arr []string }` (the single shared slice).
- `js/modules/k6/data/share.go:14-21` — the lightweight read-only
  `wrappedSharedArray` type, which embeds `sharedArray`.
- `js/modules/k6/data/share.go:23-34` — `wrap` builds a `wrappedSharedArray` over
  the **same** `sharedArray` value (`sharedArray: s`, line 28) via
  `rt.NewDynamicArray(...)` — no copy of the backing slice is made.
- `js/modules/k6/data/share.go:36-42` — `Set` (line 37) and `SetLen` (line 41)
  both `panic(s.rt.NewTypeError("SharedArray is immutable"))` (specifically a JS
  `TypeError`, per the source's own "this is specifically a type error" comment),
  enforcing read-only access.
- `js/modules/k6/data/share.go:44-48` — `Get` reads element `s.arr[index]`
  (line 48) from the shared slice.

### 6. Reasoned answer

- **Constant or per-VU copy?** With `SharedArray`, the footprint is **essentially
  constant** — **83 → 88 → 89 MB** from 1 → 50 → 100 VUs (only **+6 MB** total).
  With a plain module-level array, memory grows **linearly** —
  **80 → 1212 → 2307 MB**, i.e. **~22.5 MB per additional VU**
  (`(2307 − 80) / 99`). So the plain array is **copied per VU**, while
  `SharedArray` is **not**.
- **Test-script output proof:** the RSS matrix above, plus the marker logged
  **exactly once** across 10 VUs / 50 iterations (`grep -c` = 1) — the dataset is
  built a single time.
- **Root cause:** the `SharedArray` dataset is stored **once** in the root
  module's shared, name-keyed map and cached by name
  (`js/modules/k6/data/data.go:53-58,95,152-167`); every VU gets only a read-only
  `wrappedSharedArray` referencing the **same backing slice**
  (`js/modules/k6/data/share.go:14-34`). A plain array, by contrast, is re-created
  inside **every VU's own (sobek) JS runtime**, because k6 uses a
  **shared-nothing VU model** — hence the linear growth.

### 7. Coverage checklist (R4)

- [x] Constant vs. per-VU answered (constant for SharedArray, per-VU for plain).
- [x] Test-script output proof provided (RSS matrix + single marker).
- [x] Root cause explained (shared name-keyed store + read-only wrapper vs. shared-nothing per-VU runtime).
- [x] Environment-scaling caveat noted (`N=25000`, `VmHWM` sampling).

---

## R5 — Prometheus output metric-name integrity

### 1. Question (verbatim)

> I also want you to investigate the metric reporting behavior when using the
> prometheus output. Specifically, provide test script output to prove that the
> exported data maintain the integrity of the metric names.

### 2. Test script (`/tmp/obs/r5_prometheus.js`)

```js
import { Counter, Trend, Gauge, Rate } from 'k6/metrics';
import { sleep } from 'k6';

const myCounter = new Counter('my_custom_counter');
const myTrend = new Trend('my_custom_trend');
const myGauge = new Gauge('my_custom_gauge');
const myRate = new Rate('my_custom_rate');

export const options = {
  scenarios: {
    s: { executor: 'constant-vus', vus: 5, duration: '8s' },
  },
};

export default function () {
  myCounter.add(1);
  myTrend.add(Math.random() * 100);
  myGauge.add(Math.random() * 50);
  myRate.add(Math.random() > 0.5);
  sleep(1);
}
```

### 3. Exact run command

A **mock remote-write receiver** was built in an isolated `/tmp/promrecv` Go
module (NOT added to the repo) with wire-compatible versions —
`buf.build/gen/go/prometheus/prometheus/protocolbuffers/go v1.31.0-20230627135113-9a12bc2590d2.1`,
`github.com/klauspost/compress v1.17.11`, `google.golang.org/protobuf v1.35.1`.
It listens on `:9090` at `POST /api/v1/write`, `snappy.Decode`s then
`proto.Unmarshal`s each body into a `prompb.WriteRequest`, and prints request
headers plus every unique `__name__` label.

```
# mock receiver already listening on :9090 (out-of-repo /tmp/promrecv_bin)
K6_PROMETHEUS_RW_SERVER_URL=http://localhost:9090/api/v1/write \
K6_PROMETHEUS_RW_PUSH_INTERVAL=2s \
K6_PROMETHEUS_RW_TREND_STATS=p(99) \
/tmp/k6bin run --out experimental-prometheus-rw /tmp/obs/r5_prometheus.js
```

k6 confirmed the output:
`output: Prometheus remote write (http://localhost:9090/api/v1/write)` and
completed `iterations: 40`.

### 4. Verbatim observed output

**Receiver log** (5 POSTs with the exact encoding headers, then the final unique
`__name__` set):

```
mock remote-write receiver listening on :9090 (POST /api/v1/write)
POST #1 path=/api/v1/write Content-Encoding="snappy" Content-Type="application/x-protobuf" X-Prometheus-Remote-Write-Version="0.1.0" User-Agent="k6-prometheus-rw-output" bodyBytes=304
  NEW __name__=k6_my_custom_gauge
  NEW __name__=k6_my_custom_rate_rate
  NEW __name__=k6_data_sent_total
  NEW __name__=k6_data_received_total
  NEW __name__=k6_vus_max
  NEW __name__=k6_my_custom_counter_total
  NEW __name__=k6_my_custom_trend_p99
  NEW __name__=k6_iteration_duration_p99
  NEW __name__=k6_iterations_total
  NEW __name__=k6_vus
POST #2 path=/api/v1/write Content-Encoding="snappy" Content-Type="application/x-protobuf" X-Prometheus-Remote-Write-Version="0.1.0" User-Agent="k6-prometheus-rw-output" bodyBytes=317
POST #3 path=/api/v1/write Content-Encoding="snappy" Content-Type="application/x-protobuf" X-Prometheus-Remote-Write-Version="0.1.0" User-Agent="k6-prometheus-rw-output" bodyBytes=309
POST #4 path=/api/v1/write Content-Encoding="snappy" Content-Type="application/x-protobuf" X-Prometheus-Remote-Write-Version="0.1.0" User-Agent="k6-prometheus-rw-output" bodyBytes=312
POST #5 path=/api/v1/write Content-Encoding="snappy" Content-Type="application/x-protobuf" X-Prometheus-Remote-Write-Version="0.1.0" User-Agent="k6-prometheus-rw-output" bodyBytes=203
==== FINAL UNIQUE __name__ SET (10) ====
NAME k6_data_received_total
NAME k6_data_sent_total
NAME k6_iteration_duration_p99
NAME k6_iterations_total
NAME k6_my_custom_counter_total
NAME k6_my_custom_gauge
NAME k6_my_custom_rate_rate
NAME k6_my_custom_trend_p99
NAME k6_vus
NAME k6_vus_max
```

The **ten exported series names** (sorted) are exactly:
`k6_data_received_total`, `k6_data_sent_total`, `k6_iteration_duration_p99`,
`k6_iterations_total`, `k6_my_custom_counter_total`, `k6_my_custom_gauge`,
`k6_my_custom_rate_rate`, `k6_my_custom_trend_p99`, `k6_vus`, `k6_vus_max`.

### 5. Source citations

- `cmd/outputs.go:66-68` — registers the `experimental-prometheus-rw` output
  (`builtinOutputExperimentalPrometheusRW.String(): func(params output.Params) (output.Output, error) { return remotewrite.New(params) }`).
- `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/prometheus.go:11,39-50`
  — `const namelbl = "__name__"` (line 11); `MapSeries` (line 39) builds
  `v := defaultMetricPrefix + series.Metric.Name` (line 40) and, if a type suffix
  exists, appends `"_" + suffix`, emitting the label `{Name: "__name__", Value: v}`
  (line 45).
- `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/config.go:21,24,28`
  — `defaultServerURL = "http://localhost:9090/api/v1/write"` (line 21);
  `defaultMetricPrefix = "k6_"` (line 24); `defaultTrendStats = []string{"p(99)"}`
  (line 28).
- `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/remotewrite.go:319-345`
  — the `mapMonoSeries` helper (lines 319-326) applies the type suffix for the
  single-sample types: Counter → `mapMonoSeries(..., "total", ...)` (`_total`,
  line 331), Gauge → `mapMonoSeries(..., "", ...)` (none, line 336), Rate →
  `mapMonoSeries(..., "rate", ...)` (`_rate`, line 341).
- `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/remotewrite.go:347-356`
  — the Trend branch does **not** append the suffix itself; it **delegates** via
  `newts = trend.MapPrompb(swm.TimeSeries, swm.Latest)` (line 356).
- `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/trend.go:36,48,53-54,78,89`
  — the delegated Trend mapping is where the per-stat suffix is actually appended:
  `extendedTrendSink.MapPrompb` (line 36) builds the base labels with no suffix
  (`MapSeries(series, "")`, line 48), then iterates `sink.trendStats` calling
  `tg.Append(stat, ...)` for each stat (lines 53-54); `Append` (line 78) appends
  the stat name as the suffix — `ts.Labels[tg.ixname].Value += "_" + suffix`
  (line 89) — e.g. `p99` → `_p99`.
- `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/remotewrite.go:177-189`
  — `setTrendStatsResolver` normalizes the configured stat `p(99)` into the series
  suffix `p99`: it trims the parentheses (`statKey = stat[2 : len(statKey)-1]`,
  line 185) and re-prepends `"p"` (line 187), storing the resolver under key `p99`
  (line 189) — this is why the exported Trend suffix is `_p99`, not `_p(99)`.
- `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remote/client.go:78-133`
  — `Store` (line 78) POSTs with headers `User-Agent: k6-prometheus-rw-output`
  (line 96), `Content-Encoding: snappy` (line 99),
  `Content-Type: application/x-protobuf` (line 100),
  `X-Prometheus-Remote-Write-Version: 0.1.0` (line 101); the body is
  `proto.Marshal(&prompb.WriteRequest{...})` (line 123) then `snappy.Encode(...)`
  (line 133).

### 6. Reasoned answer

- **Test-script output proving metric-name integrity:** the receiver decoded
  **5 POSTs** (`Content-Encoding="snappy"`, `Content-Type="application/x-protobuf"`)
  and recovered exactly the **10 names** above. Every exported name equals
  **`"k6_"` + the original metric name + a Prometheus-conventional type suffix**:
  - Counter → `_total`: `my_custom_counter` → `k6_my_custom_counter_total`
  - Gauge → none: `my_custom_gauge` → `k6_my_custom_gauge`
  - Rate → `_rate`: `my_custom_rate` → `k6_my_custom_rate_rate`
  - Trend → per-stat, default `p(99)` → `_p99`: `my_custom_trend` →
    `k6_my_custom_trend_p99` (and the built-in `iteration_duration` →
    `k6_iteration_duration_p99`)

  The original names appear **intact — no truncation, mangling, or collision** —
  confirming metric-name integrity. This matches
  `prometheus.go:39-50` + `config.go:24` (the `k6_` prefix) +
  `remotewrite.go:329-356` (the per-type suffixes) + `trend.go:53-54,89` (the
  Trend per-stat `_p99` suffix, appended after `remotewrite.go:356` delegates to
  `trend.MapPrompb`).

### 7. Coverage checklist (R5)

- [x] Test-script output provided (5 decoded POSTs + 10 `__name__` labels).
- [x] Encoding headers shown (snappy + protobuf).
- [x] Name-integrity rule stated (`k6_` prefix + type suffix, no mangling).
- [x] Each custom metric's exported name mapped.

---

## Coverage pass

Every distinct sub-question of R1–R5 is answered, as summarized below.

| Req | Sub-question | Answered with |
|-----|--------------|---------------|
| R1 | Exact SIGINT log messages | the six verbatim log lines |
| R1 | Finish vs. mid-execution termination | `0 complete and 8 interrupted`; exit 105; ~0.03 s |
| R2 | Exact interruption log entries | `Graceful stop`→`Hard stop`→`stream is cancelled/finished`→`is closing`→`STREAM_ERROR` |
| R2 | gRPC messages received value | `grpc_streams_msgs_received = 150` |
| R3 | Exact `dropped_iterations` | `985` (1000 − 15) |
| R3 | Proof it came from the API | `GET /v1/metrics` → `HTTP/1.1 200 OK` + JSON:API body; server `status=200` log |
| R4 | Constant vs. per-VU copy | SharedArray 83→89 MB (constant) vs. plain 80→2307 MB (linear) |
| R4 | Root cause | shared name-keyed store + read-only wrapper vs. per-VU JS runtime (shared-nothing) |
| R5 | Name-integrity proof | 10 decoded `__name__` labels; `k6_` + name + type-suffix, no mangling |

### Read-only guarantee (verified)

- No repository source, test, config, build, or `vendor/` file was modified,
  added, or deleted — the **only** artifact added is this document.
- The runtime investigation was performed against **source commit
  `ddc3b0b1d23c128e34e2792fc9075f9126e32375`**; the final repository diff from
  that commit contains only this Markdown deliverable
  (`git diff --name-status ddc3b0b1d23c128e34e2792fc9075f9126e32375..HEAD` →
  `A blitzy/documentation/k6_ddc3b0b1d23c.md`). Committing the deliverable
  necessarily advances the destination branch's `HEAD` by that single
  documentation commit.
- All observation scripts and helper binaries lived under `/tmp` (outside the
  repository) and are not committed.
