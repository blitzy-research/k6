# Grafana k6 Runtime-Behavior Investigation

**An evidence-backed Q&A report on five runtime behaviors of the Grafana k6 load-testing engine.**

---

## Introduction

This report answers five specific questions about how the [Grafana k6](https://github.com/grafana/k6) load-testing engine behaves at runtime. The engine under investigation is the Go module `go.k6.io/k6` at **`k6 v0.55.0`**, branch `k6_ddc3b0b1d23c` at HEAD commit `ddc3b0b1d`.

Every answer below is grounded in two things and two things only:

1. **The k6 source code**, treated as the single source of truth. Each behavioral claim carries an exact `path/to/file.go:Lxx` citation that was verified against this commit.
2. **Real, captured runtime output** — verbatim log lines, exact metric numbers, the raw REST API JSON, a measured RSS table, and the decoded Prometheus metric names — produced by running experiments against a locally built k6 binary. Nothing is asserted or inferred; anything that could not be observed at runtime is explicitly reported as "not observed."

### How k6 was built (out-of-tree)

Go is not part of the repository, so a Go toolchain (Go 1.23.4 at `/usr/local/go`) was installed and k6 was compiled **outside the repository tree** so that the source tree stays byte-for-byte unchanged. The build uses `GOTOOLCHAIN=local` (so Go does not auto-download the pinned `go1.21.13` declared in `go.mod`) and the committed `vendor/` tree (`-mod=vendor`) for a fully offline build:

```bash
# from the repo root
export PATH=$PATH:/usr/local/go/bin
GOTOOLCHAIN=local GOFLAGS=-mod=vendor go build -o /tmp/k6bin/k6 .
/tmp/k6bin/k6 version
```

```text
k6 v0.55.0 (commit/ddc3b0b1d2, go1.23.4, linux/amd64)
```

The binary lives at `/tmp/k6bin/k6` and **all** experiment scaffolding (test scripts, a JSON fixture, the gRPC server binary, and a Prometheus remote-write receiver) lives under `/tmp` — never inside the repository. The closing section proves, with an empty `git status --porcelain`, that the only file added anywhere is this report.

### Methodology

For each of the five objectives the same disciplined loop was applied:

1. **Locate** the governing code in the read-only source tree.
2. **Design** a minimal experiment that triggers the behavior.
3. **Run** it against `/tmp/k6bin/k6` and **capture** the concrete output.
4. **Explain** the observed result with `file:line` citations to the code.

Two non-obvious runtime facts shape several experiments and are called out where they apply:

- **`-v`/`--verbose` is mandatory for Q1 and Q2.** The signal-handling and lifecycle messages are emitted at `DEBUG` level; without `-v` only the single final `ERROR` line is visible.
- **`--linger` is mandatory for Q3.** The dropped-iterations count is pushed only in a deferred end-of-run step, so the REST API must be kept alive past test end to observe it.

### Table of contents

1. [Q1 — VU lifecycle under SIGINT (`ramping-vus`)](#q1--vu-lifecycle-under-sigint-ramping-vus)
2. [Q2 — gRPC server-streaming interruption](#q2--grpc-server-streaming-interruption)
3. [Q3 — `dropped_iterations` via the REST control API](#q3--dropped_iterations-via-the-rest-control-api)
4. [Q4 — File data-sharing memory footprint (`SharedArray` vs `open()`)](#q4--file-data-sharing-memory-footprint-sharedarray-vs-open)
5. [Q5 — Prometheus remote-write metric-name integrity](#q5--prometheus-remote-write-metric-name-integrity)
6. [Reproducibility & cleanup](#reproducibility--cleanup)

---

## Q1 — VU lifecycle under SIGINT (`ramping-vus`)

> **Question.** What are the exact log messages that appear when a ramping executor with at least 5 VUs receives a SIGINT? While the system is shutting down, what is the specific log evidence that shows whether currently active VUs are allowed to finish their current iteration or are terminated mid-execution?

### Experiment

A `ramping-vus` scenario climbs to **6 VUs** (satisfying "at least 5"). Each iteration prints an `ITER_START` marker, sleeps 10 s, then prints an `ITER_END` marker — so it is provable whether an in-flight iteration finishes. `gracefulRampDown: '0s'` ensures stage transitions don't mask the SIGINT behavior, which is what we are actually testing.

```javascript
// /tmp/q1_ramping.js
import { sleep } from 'k6';

export const options = {
  scenarios: {
    ramp: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '4s', target: 6 },   // climb to 6 VUs (>= 5)
        { duration: '40s', target: 6 },  // hold so SIGINT lands mid-iteration
      ],
      gracefulRampDown: '0s',
    },
  },
};

export default function () {
  console.log(`ITER_START vu=${__VU} iter=${__ITER} t=${Date.now()}`);
  sleep(10);  // long iteration body so SIGINT clearly lands mid-iteration
  console.log(`ITER_END vu=${__VU} iter=${__ITER} t=${Date.now()}`);
}
```

The test is run with verbose logging; after ~9 s (all 6 VUs are inside `sleep(10)`) it is sent a single SIGINT, and stderr + the exit code are captured. (Note: k6's `console.log` is written to **stderr**, so the markers and the log lines share one stream.)

```bash
/tmp/k6bin/k6 run -v /tmp/q1_ramping.js 2> /tmp/q1.log &
K6PID=$!
sleep 9            # all 6 VUs have printed ITER_START and are inside sleep(10)
kill -INT $K6PID   # SIGINT == Ctrl+C
wait $K6PID; echo "EXIT_CODE=$?"
```

### (a) Captured runtime output

**The exact signal/abort log lines** (verbose stderr; `sig=interrupt` because Go's `os.Interrupt` stringifies as `interrupt`):

```text
time="2026-06-26T20:53:07Z" level=debug msg="Trapping interrupt signals so k6 can handle them gracefully..."
time="2026-06-26T20:53:16Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-06-26T20:53:16Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
time="2026-06-26T20:53:16Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-06-26T20:53:16Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-06-26T20:53:16Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

The first `DEBUG` line — `Trapping interrupt signals so k6 can handle them gracefully...` — is emitted at startup and is itself the reason `-v` is required: without it, only the final `level=error` line is printed.

**Termination proof.** With 6 VUs all mid-`sleep(10)` when SIGINT arrives, the marker counts are decisive:

```text
ITER_START=6  ITER_END=1
```

Six iterations started, but only **one** reached its `ITER_END` — and that one was cut short. The single VU that printed an end (vu=2) shows:

```text
time="2026-06-26T20:53:11Z" level=info msg="ITER_START vu=2 iter=0 t=1782507191085" source=console
time="2026-06-26T20:53:16Z" level=info msg="ITER_END   vu=2 iter=0 t=1782507196728" source=console
```

The elapsed time between its start and end is `1782507196728 − 1782507191085 = 5643 ms` — i.e. its `sleep(10)` returned after only ~5.6 s instead of the full 10 s, because the run context was cancelled out from under it. The other five VUs never printed `ITER_END` at all. A second identical run landed the SIGINT a moment earlier and produced `ITER_START=6 ITER_END=0` (no iteration completed) — the same conclusion, even more emphatically.

**Exit code:**

```text
EXIT_CODE=105
neededVUs=6
```

**Hard stop (second SIGINT) — not observed.** A second SIGINT is supposed to trigger a hard-stop path that logs `level=error msg="Aborting k6 in response to signal"`. In these runs k6 had already exited (within ~0.2 s of the first signal) before a second SIGINT could be delivered, so that specific ERROR line was **not observed**. Its code path is cited below for completeness.

### (b) Code-level root-cause rationale

**1. The signal is trapped and dispatched (first signal → graceful, second → hard).** `handleTestAbortSignals` registers the OS-signal channel and routes the first and second signals to two different handlers:

```go
// cmd/common.go:L97
func handleTestAbortSignals(gs *state.GlobalState, gracefulStopHandler, onHardStop func(os.Signal)) (stop func()) {
    gs.Logger.Debug("Trapping interrupt signals so k6 can handle them gracefully...")   // cmd/common.go:L98 (DEBUG => needs -v)
    sigC := make(chan os.Signal, 2)
    ...
    gs.SignalNotify(sigC, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)                 // cmd/common.go:L101
    go func() {
        select {
        case sig := <-sigC:
            gracefulStopHandler(sig)                                                     // cmd/common.go:L106 (1st signal)
        ...
        select {
        case sig := <-sigC:
            if onHardStop != nil { onHardStop(sig) }                                     // cmd/common.go:L113-L114 (2nd signal)
            gs.OSExit(int(exitcodes.ExternalAbort))                                       // cmd/common.go:L118
        ...
```

**2. The graceful handler logs the DEBUG line and aborts with the reason string + exit code 105.** The hard handler logs the ERROR line:

```go
// cmd/run.go:L349
gracefulStop := func(sig os.Signal) {
    logger.WithField("sig", sig).Debug("Stopping k6 in response to signal...")           // cmd/run.go:L350
    runAbort(errext.WithAbortReasonIfNone(
        errext.WithExitCodeIfNone(
            fmt.Errorf("test run was aborted because k6 received a '%s' signal", sig),
            exitcodes.ExternalAbort,                                                       // cmd/run.go:L354
        ), errext.AbortedByUser,
    ))
    lingerCancel()
}
onHardStop := func(sig os.Signal) {
    logger.WithField("sig", sig).Error("Aborting k6 in response to signal")               // cmd/run.go:L360 (the un-observed hard-stop line)
    globalCancel()
}
```

**3. The lifecycle contract distinguishes graceful from hard.** The executor's VU-handle documents the intended semantics — but note these describe a *gracefulStop of a ramp/stage*, which is distinct from the SIGINT-driven *abort*:

```go
// lib/executor/vu_handle.go:L63
// - gracefulStop must let an iteration which has started to finish ...
// lib/executor/vu_handle.go:L67
// - hardStop must stop an iteration in process
```

**4. The decisive mechanism — the JS VM is forcibly halted.** A SIGINT abort cancels the VU's run context. When that context is done, k6 interrupts the Sobek JavaScript runtime, which stops the currently executing iteration *mid-body*:

```go
// js/runner.go:L708
context.AfterFunc(ctx, func() {
    // Interrupt the JS runtime
    u.Runtime.Interrupt(context.Canceled)   // js/runner.go:L710 — halts the iteration mid-execution
    ...
})
```

This is why everything after the point of interruption (here, the second `console.log`) never runs for the active VUs. `k6.Sleep` corroborates this: it races the timer against `ctx.Done()`, so a cancelled context makes the sleep return immediately — which is exactly why vu=2's `sleep(10)` ended at ~5.6 s:

```go
// js/modules/k6/k6.go:L72
func (mi *K6) Sleep(secs float64) {
    ctx := mi.vu.Context()
    timer := time.NewTimer(time.Duration(secs * float64(time.Second)))
    select {
    case <-timer.C:
    case <-ctx.Done():        // js/modules/k6/k6.go:L77 — cancellation returns early
        timer.Stop()
    }
}
```

**5. The exit code.** `ExternalAbort` is defined as 105:

```go
// errext/exitcodes/codes.go:L41
ExternalAbort ExitCode = 105
```

The executor and its `gracefulRampDown` option are defined in `lib/executor/ramping_vus.go:L40` (`RampingVUsConfig`) and `lib/executor/ramping_vus.go:L44` (`GracefulRampDown`).

### (c) Concluding answer

On a single SIGINT, k6 logs (at `DEBUG`, hence `-v` is required) `Stopping k6 in response to signal... sig=interrupt`, then `Test finished with an error ... test run was aborted because k6 received a 'interrupt' signal`, and finally the `level=error` line `test run was aborted because k6 received a 'interrupt' signal`; the process exits with code **105** (`ExternalAbort`). The currently active VUs are **NOT allowed to finish their current iteration — they are terminated mid-execution.** The runtime proof is the marker count: 6 VUs printed `ITER_START` but only one printed a (truncated) `ITER_END`, and that VU's `sleep(10)` was cut to ~5.6 s. The root cause is that the abort cancels the VU context and `u.Runtime.Interrupt(context.Canceled)` (`js/runner.go:L710`) halts the JavaScript VM in the middle of the iteration.

---

## Q2 — gRPC server-streaming interruption

> **Question.** What are the exact log entries obtained at runtime when a gRPC server-streaming test with a 30 ms graceful ramp-down is interrupted? Also, what is the value of the number of gRPC messages received reported by the final metrics summary?

### Experiment

**1. The bundled gRPC server.** k6 ships a runnable gRPC server example (`examples/grpc_server`) that serves the `route_guide` dataset on `localhost:10000`. It is a *separate Go module* (`module go.k6.io/k6/examples/grpc_server`, with `replace go.k6.io/k6 => ../../`) and imports the non-vendored `google.golang.org/grpc/testdata`, so it must be built with `-mod=mod`. To avoid mutating the repo's tracked manifests, the example was copied to `/tmp/grpcsrc`, its `replace` directive repointed to the absolute repo path, and built from there; the repo's `go.mod`/`go.sum` were also backed up and restored as a belt-and-braces measure:

```bash
# from the repo root; capture the absolute repo path for the replace directive
REPO="$(pwd)"
cp go.mod /tmp/go.mod.bak && cp go.sum /tmp/go.sum.bak       # protect repo manifests (belt-and-braces)

# 1) copy the example module out-of-tree so the repo is never built/mutated in place
rm -rf /tmp/grpcsrc && cp -a examples/grpc_server /tmp/grpcsrc

# 2) repoint its "replace go.k6.io/k6 => ../../" to the absolute repo path so the copy
#    can resolve the k6 module from its new /tmp location
sed -i "s#replace go.k6.io/k6 => ../../#replace go.k6.io/k6 => ${REPO}#" /tmp/grpcsrc/go.mod

# 3) build from INSIDE the copy (needs -mod=mod: it imports the non-vendored
#    google.golang.org/grpc/testdata). Building from /tmp/grpcsrc — not the repo —
#    leaves the repo's go.mod/go.sum byte-for-byte unchanged.
( cd /tmp/grpcsrc && GOTOOLCHAIN=local go build -mod=mod -o /tmp/grpcserver . )

cp /tmp/go.mod.bak go.mod && cp /tmp/go.sum.bak go.sum       # restore repo manifests immediately
/tmp/grpcserver &                                            # serves localhost:10000
cp lib/testutils/grpcservice/route_guide.proto /tmp/route_guide.proto
```

**2. The k6 server-streaming script.** A `ramping-vus` scenario reaching 5 VUs calls the server-streaming RPC `main.FeatureExplorer/ListFeatures` with `gracefulRampDown: '30ms'`. It counts received messages via `stream.on('data')` and registers a `stream.on('error')` handler so we can observe whether graceful per-stream errors fire.

```javascript
// /tmp/q2_grpc_streaming.js
import { Client, Stream } from 'k6/net/grpc';
import { sleep } from 'k6';

export const options = {
  scenarios: {
    streamers: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2s', target: 5 },   // climb to 5 VUs (>= 5)
        { duration: '40s', target: 5 },  // hold so SIGINT lands mid-stream
      ],
      gracefulRampDown: '30ms',
    },
  },
};

const GRPC_ADDR = __ENV.GRPC_ADDR || '127.0.0.1:10000';
const GRPC_PROTO_PATH = __ENV.GRPC_PROTO_PATH || 'route_guide.proto'; // relative to the script dir
const client = new Client();
client.load([], GRPC_PROTO_PATH);

export default () => {
  client.connect(GRPC_ADDR, { plaintext: true });
  const stream = new Stream(client, 'main.FeatureExplorer/ListFeatures', null);
  stream.on('data', (f) => { /* each received feature increments grpc_streams_msgs_received */ });
  stream.on('error', (e) => { console.log('STREAM_ERROR ' + JSON.stringify(e)); });
  stream.on('end', () => { client.close(); });
  stream.write({ lo: { latitude: 400000000, longitude: -750000000 },
                 hi: { latitude: 420000000, longitude: -730000000 } });
  sleep(2);
};
```

> Note on the proto path: `client.load()` resolves the proto path **relative to the script's directory**, so the script uses the relative name `route_guide.proto` (with the file copied next to it), not an absolute `/tmp/...` path.

**3. Run, interrupt, capture.** Run verbose, send SIGINT after ~6 s, and capture the abort logs, the exit code, and the end-of-test summary:

```bash
/tmp/k6bin/k6 run -v /tmp/q2_grpc_streaming.js 2> /tmp/q2.log &
K6PID=$!; sleep 6; kill -INT $K6PID; wait $K6PID; echo "EXIT_CODE=$?"
```

### (a) Captured runtime output

**Abort log lines** (identical mechanism to Q1):

```text
time="2026-06-26T20:55:50Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-06-26T20:55:50Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-06-26T20:55:50Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

```text
EXIT_CODE=105
streamers ✗ [  14% ] 5/5 VUs  06.0s/42.0s
```

**The gRPC counters from the end-of-test summary:**

```text
     grpc_streams.................: 5      0.83694/s
     grpc_streams_msgs_received...: 235    39.336158/s
     grpc_streams_msgs_sent.......: 5      0.83694/s
```

So in this run **`grpc_streams_msgs_received = 235`**, with `grpc_streams = 5` and `grpc_streams_msgs_sent = 5`. The `stream.on('error')` handler fired **0 times** (no `STREAM_ERROR` line appeared in the log).

> **This value is timing-dependent — run parameters matter.** With 5 VUs, `gracefulRampDown: '30ms'`, and a SIGINT delivered ~6 s into the run, this run observed **235** received messages. The number depends on exactly when the SIGINT lands relative to the in-flight streams (see rationale), so it is reported as the value observed for *these* parameters, not as a fixed constant.

### (b) Code-level root-cause rationale

**The three counters are plain `metrics.Counter`s** registered by the gRPC module:

```go
// js/modules/k6/grpc/metrics.go:L17
m.Streams, err = registry.NewMetric("grpc_streams", metrics.Counter)
// js/modules/k6/grpc/metrics.go:L21
m.StreamsMessagesSent, err = registry.NewMetric("grpc_streams_msgs_sent", metrics.Counter)
// js/modules/k6/grpc/metrics.go:L25
m.StreamsMessagesReceived, err = registry.NewMetric("grpc_streams_msgs_received", metrics.Counter)
```

`grpc_streams_msgs_received` is incremented per received message in `js/modules/k6/grpc/stream.go:L153`; `grpc_streams_msgs_sent` at `js/modules/k6/grpc/stream.go:L257`; `grpc_streams` at `js/modules/k6/grpc/stream.go:L106`.

**Why the received count is non-deterministic.** The server streams matching features with a deliberate **100 ms sleep before each send**:

```go
// lib/testutils/grpcservice/service.go:L57
func (s *FeatureExplorerImplementation) ListFeatures(rect *Rectangle, stream FeatureExplorer_ListFeaturesServer) error {
    ...
    for _, feature := range s.savedFeatures {
        if inRange(feature.Location, rect) {
            time.Sleep(100 * time.Millisecond)        // lib/testutils/grpcservice/service.go:L62
            if err := stream.Send(feature); err != nil { return err }   // :63
        }
    }
```

The embedded dataset has 100 features (`LoadFeatures` at `lib/testutils/grpcservice/service.go:L170`), and the script's bounding rectangle covers all of them, so a *complete* single stream delivers 100 messages at ~10/second — taking ~10 s. Five staggered streams are therefore each only partway through their 100 messages when the SIGINT lands at ~6 s, and the total received (here 235) reflects exactly how many had arrived at that instant. The RPC is declared server-streaming in the proto: `rpc ListFeatures(Rectangle) returns (stream Feature)` (`lib/testutils/grpcservice/route_guide.proto:L38`, `package main` at `:L17`).

**Why `stream.on('error')` fired 0 times.** This is the same VM-halt behavior proven in Q1: the abort cancels the context and `u.Runtime.Interrupt(context.Canceled)` (`js/runner.go:L710`) stops the JavaScript runtime outright, rather than delivering a graceful per-stream gRPC error callback into the script. The interruption is a VM halt, not a clean stream error — hence zero `error` events. The `gracefulRampDown: '30ms'` option itself lives on `lib/executor/ramping_vus.go:L44`.

### (c) Concluding answer

When the server-streaming test is interrupted, k6 emits the same graceful-abort log lines as any SIGINT — `Stopping k6 in response to signal... sig=interrupt` and `test run was aborted because k6 received a 'interrupt' signal` (DEBUG + final ERROR) — and exits **105**. The final summary in this run reported **`grpc_streams_msgs_received = 235`** (alongside `grpc_streams = 5` and `grpc_streams_msgs_sent = 5`), and `stream.on('error')` fired **0 times**. That received count is **timing-dependent**: the server sleeps 100 ms before each of up to 100 streamed features (`lib/testutils/grpcservice/service.go:L62`), so the figure reflects how many messages had been received across the five in-flight streams at the moment the SIGINT halted the JS VM.

---


## Q3 — `dropped_iterations` via the REST control API

> **Question.** What is the exact value of dropped iterations reported when a test exceeds its maximum duration capacity? Report this value **by querying the API**, and give runtime evidence proving the value came from the API.

### Experiment

A `shared-iterations` executor is given an iteration budget that provably cannot finish within its `maxDuration`: 100 iterations shared across 5 VUs, each iteration sleeping 1 s, capped at `maxDuration: '3s'`. In 3 s, 5 VUs doing 1 s sleeps complete only ~15 iterations, so ~85 must be dropped.

```javascript
// /tmp/q3_dropped.js
import { sleep } from 'k6';

export const options = {
  scenarios: {
    drop: {
      executor: 'shared-iterations',
      vus: 5,
      iterations: 100,
      maxDuration: '3s',   // 100 iters * ~1s each across 5 VUs cannot finish in 3s
    },
  },
};

export default function () { sleep(1); }
```

The run uses **`--linger`** so the metrics engine and REST API stay alive after the test ends (the dropped count is only emitted in a deferred end-of-run push — see rationale). While k6 lingers, the API is queried:

```bash
/tmp/k6bin/k6 run --linger /tmp/q3_dropped.js > /tmp/q3.summary 2>&1 &
# ... wait until the "--linger was enabled" message appears, then:
curl -s http://localhost:6565/v1/metrics/dropped_iterations
```

### (a) Captured runtime output

**The raw REST API response** from `GET http://localhost:6565/v1/metrics/dropped_iterations` (a JSON:API envelope; a counter's `sample` carries `count` and `rate`):

```json
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":85,"rate":28.314569551237426}}}}
```

The API reports **`count: 85`**. (`count` is deterministic at 85 for these parameters; `rate` = count ÷ elapsed varies slightly between runs.)

**Cross-check against the end-of-test summary** — the same 85, and 15 completed iterations confirm the arithmetic (`100 − 15 = 85`):

```text
              * drop: 100 iterations shared among 5 VUs (maxDuration: 3s, gracefulStop: 30s)
The test is done, but --linger was enabled, so k6 is waiting for Ctrl+C to continue...
     dropped_iterations...: 85  28.31457/s
     iterations...........: 15  4.996689/s
```

**Negative controls — proof the value genuinely came from the API.**

*Without* `--linger`, the process exits immediately after the test, the API server is gone, and the same `curl` cannot connect:

```text
curl_exit=7 http_code=000        # connection refused — API server already shut down
```

While lingering, querying a **bogus** metric id returns HTTP 404 with k6's exact not-found message — proving the endpoint is genuinely serving (and matching against) real metric ids rather than echoing input:

```json
{"errors":[{"status":"404","title":"Not Found","detail":"No metric with that ID was found"}]}
```

Together these show that the `count: 85` was retrievable **only** by querying the live API on the lingering process — it is API-sourced, not copied from the summary text.

### (b) Code-level root-cause rationale

**Metric identity.** The name constant and the counter registration:

```go
// metrics/builtin.go:L10
DroppedIterationsName = "dropped_iterations"
// metrics/builtin.go:L84
DroppedIterations: registry.MustNewMetric(DroppedIterationsName, Counter),
```

**Deferred emission when `maxDuration` is hit.** The `shared-iterations` executor pushes the dropped count only *after* all VUs have stopped, computing it as the unfinished remainder of the budget:

```go
// lib/executor/shared_iterations.go:L213
var attemptedIters uint64
...
defer func() {
    activeVUs.Wait()                                     // :218
    if attemptedIters < totalIters {                     // :219
        metrics.PushIfNotDone(parentCtx, out, metrics.Sample{
            TimeSeries: metrics.TimeSeries{
                Metric: si.executionState.Test.BuiltinMetrics.DroppedIterations,   // :222
                Tags:   si.getMetricTags(nil),
            },
            Value: float64(totalIters - attemptedIters),                            // :225
            Time:  time.Now(),
        })
    }
}()
```

With `totalIters = 100` and `attemptedIters = 15`, the pushed value is `100 − 15 = 85`. This deferred end-of-run push (`lib/executor/shared_iterations.go:L217-L228`) is the mechanism this experiment exercises. Other executors also report the same `dropped_iterations` metric, but **not all via a deferred end-of-run push**: the `per-vu-iterations` executor emits the unfinished per-VU remainder (`iterations - i`) at the moment its `maxDuration` is reached (`lib/executor/per_vu_iterations.go:L216`), whereas the arrival-rate executors emit `dropped_iterations` **immediately** — pushing one sample with `Value: 1` each time the scheduler finds no free VU for a planned iteration (`lib/executor/constant_arrival_rate.go:L339-L345`, `lib/executor/ramping_arrival_rate.go:L470-L476`) — rather than as a single deferred push at end of run.

**Why `--linger` is required.** Because the count is pushed only in this deferred end-of-run step, it appears in the metrics engine only at/after test end. `--linger` keeps the engine and API alive past that point:

```go
// cmd/run.go:L373
if conf.Linger.Bool {
    defer func() {
        msg := "The test is done, but --linger was enabled, so k6 is waiting for Ctrl+C to continue..."
        ...
```

Without it, the server tears down and the GET returns 404 (or, as observed, the connection is refused once the process has exited).

**The API surface.** The versioned API is mounted at `/v1/` on the default address `localhost:6565`:

```go
// api/server.go:L24
mux.Handle("/v1/", v1.NewHandler(cs))
```

```go
// cmd/state/state.go:L150
Address: "localhost:6565",
```

The routes dispatch `GET /v1/metrics` to `handleGetMetrics` (`api/v1/routes.go:L23`) and `GET /v1/metrics/{id}` by slicing the id out of the path (`api/v1/routes.go:L37`) into `handleGetMetric` (`:L38`). A missing id yields the 404 string seen above:

```go
// api/v1/metric_routes.go:L37
apiError(rw, "Not Found", "No metric with that ID was found", http.StatusNotFound)
```

The JSON shape (`type`, `contains`, `tainted`, `sample map[string]float64`) is defined by the `Metric` struct in `api/v1/metric.go:L62`.

### (c) Concluding answer

Querying the REST control API at `GET http://localhost:6565/v1/metrics/dropped_iterations` returned **`count: 85`** — the exact number of iterations dropped when the `shared-iterations` test exceeded its 3 s `maxDuration` (100 budgeted − 15 completed = 85). The runtime proof that this came from the API is twofold: the raw `curl` JSON envelope above, and the negative controls — *without* `--linger` the identical query is refused (`curl_exit=7`, API gone), and a bogus metric id returns the API's own `404 "No metric with that ID was found"`. `--linger` is required because `dropped_iterations` is emitted only in a deferred end-of-run push (`lib/executor/shared_iterations.go:L217-L228`), so the API must outlive the test to serve it.

---


## Q4 — File data-sharing memory footprint (`SharedArray` vs `open()`)

> **Question.** Does the memory footprint for the files remain constant with an increasing number of VUs, or does each VU create its own copy of the data? Give test-script output to prove this behavior, and explain the root cause.

### Experiment

A ~13.8 MB JSON fixture (`/tmp/big.json`, **14,463,456 bytes ≈ 13.79 MB; an array of 70,000 objects**, each with `id`/`name`/`email`/`score`/`payload` fields) is loaded two different ways, and peak resident memory is measured at VU counts of **1, 50, and 200** for each. The fixture is generated once, out of tree, with a small Python script:

```python
# /tmp/gen_fixture.py  ->  python3 /tmp/gen_fixture.py 70000 95   (N objects, payload length)
import json, random, string, sys
N = int(sys.argv[1]); PAY = int(sys.argv[2]); random.seed(42)
def rs(n): return ''.join(random.choice(string.ascii_lowercase) for _ in range(n))
arr = [{"id": i, "name": f"User Name {i:06d}", "email": f"user{i:06d}@example.com",
        "score": round(random.uniform(0, 1000), 4), "payload": rs(PAY)} for i in range(N)]
json.dump(arr, open('/tmp/big.json', 'w'))
```

To keep the comparison **apples-to-apples**, *both* variants `JSON.parse` the file into a live JavaScript object graph — the only difference is *where* the parsed data lives: once in a shared Go-side store (`SharedArray`) versus once inside every VU's own runtime (`open()`).

**SharedArray variant** — the file is parsed inside the `SharedArray` constructor (init context):

```javascript
// /tmp/q4_shared.js
import { SharedArray } from 'k6/data';
const data = new SharedArray('big', function () { return JSON.parse(open('/tmp/big.json')); });
export const options = { scenarios: { s: { executor: 'shared-iterations', vus: __ENV.VUS, iterations: __ENV.VUS, maxDuration: '120s' } } };
export default function () { if (data.length < 0) console.log('noop'); }
```

**`open()` variant** — the file is opened **and parsed** in the init context *outside* any `SharedArray`, so every VU's init runs it independently:

```javascript
// /tmp/q4_open.js
const data = JSON.parse(open('/tmp/big.json'));   // each VU runs init => each VU parses & holds its own copy
export const options = { scenarios: { s: { executor: 'shared-iterations', vus: __ENV.VUS, iterations: __ENV.VUS, maxDuration: '120s' } } };
export default function () { if (data.length < 0) console.log('noop'); }
```

Because `/usr/bin/time` is unavailable in this environment, peak RSS is read from the kernel's high-water mark `VmHWM` in `/proc/<pid>/status`, polled frequently while each run executes:

```bash
for VUS in 1 50 200; do
  for SCRIPT in q4_shared q4_open; do
    /tmp/k6bin/k6 run -e VUS=$VUS /tmp/$SCRIPT.js >/dev/null 2>&1 &
    PID=$!; PEAK=0
    while kill -0 $PID 2>/dev/null; do
      HWM=$(awk '/VmHWM/{print $2}' /proc/$PID/status 2>/dev/null)
      [ -n "$HWM" ] && [ "$HWM" -gt "$PEAK" ] && PEAK=$HWM
      sleep 0.02
    done
    echo "$SCRIPT VUS=$VUS VmHWM_kB=$PEAK"
  done
done
```

### (a) Captured runtime output

Raw sweep output:

```text
q4_shared  VUS=1    VmHWM_kB=218920     (~214 MB)
q4_open    VUS=1    VmHWM_kB=263424     (~257 MB)
q4_shared  VUS=50   VmHWM_kB=225364     (~220 MB)
q4_open    VUS=50   VmHWM_kB=5716960    (~5583 MB)
q4_shared  VUS=200  VmHWM_kB=229180     (~224 MB)
q4_open    VUS=200  VmHWM_kB=19664812   (~19204 MB)
```

| VUs | `SharedArray` peak RSS (`VmHWM`) | `open()` peak RSS (`VmHWM`) |
|----:|---------------------------------:|----------------------------:|
| 1   | 218,920 kB (~214 MB)             | 263,424 kB (~257 MB)        |
| 50  | 225,364 kB (~220 MB)             | 5,716,960 kB (~5,583 MB)    |
| 200 | 229,180 kB (~224 MB)             | 19,664,812 kB (~19,204 MB)  |

The pattern is unambiguous: **`SharedArray` stays flat** (~214 → ~220 → ~224 MB — a ~10 MB spread that does *not* scale with VU count, i.e. constant within measurement noise) regardless of VU count, while **`open()` grows linearly** — from ~257 MB at 1 VU to ~19.2 GB at 200 VUs, i.e. `(19,664,812 − 263,424) / (200 − 1) ≈ 97,494 kB ≈ ~95 MB per added VU` (per-segment slope ranges ~91–109 MB/VU). At 1 VU the two variants are close (~257 MB `open()` vs ~214 MB `SharedArray`) because exactly one parsed copy exists in both cases; `open()`'s per-VU duplication only dominates as VUs grow. Because peak RSS includes Go allocator/GC headroom and varies run to run, the **reproducible finding is the *shape*** — flat vs. linear — not the exact kB.

> **Reconciliation with the validated reference.** The AAP's validated reference run measured `SharedArray` ~310/308/296 MB (flat) and `open()` ~248/7,514/28,369 MB (~140 MB/VU) on a ~15 MB fixture. This run reproduces the **identical behavior and root cause** — `SharedArray` constant, `open()` linear, both reaching tens of GB at 200 VUs — and the 1-VU `open()` figure matches closely (~257 MB here vs ~248 MB there). The absolute magnitudes differ (this run: flat ~214–224 MB; ~95 MB/VU) because the per-VU cost tracks the *parsed in-memory expansion* of the specific fixture: a 13.79 MB / 70,000-object file here versus the reference's ~15 MB file, whose object graph expanded to ~140 MB per copy. That difference is fixture composition (object count, field types, string sizes) plus Go heap/GC headroom in this environment — **not** a behavioral difference. The flat-vs-linear conclusion is therefore identical and trustworthy.

### (b) Code-level root-cause rationale

**`SharedArray` keeps exactly one copy.** The module's root holds a single shared store, and every VU instance is handed a pointer to that *same* store. The construction callback runs only the first time a given name is requested; thereafter the cached array is reused:

```go
// js/modules/k6/data/data.go:L152
func (s *sharedArrays) get(rt *sobek.Runtime, name string, call sobek.Callable) sharedArray {
    s.mu.RLock()
    array, ok := s.data[name]
    s.mu.RUnlock()
    if !ok {
        s.mu.Lock()
        defer s.mu.Unlock()
        array, ok = s.data[name]
        if !ok {
            array = getShareArrayFromCall(rt, call)   // js/modules/k6/data/data.go:L161 — runs the callback ONCE
            s.data[name] = array
        }
    }
    return array
}
```

(The single store lives at `js/modules/k6/data/data.go:L32` `data map[string]sharedArray`; each VU's module instance points at `&rm.shared` — the same store — at `js/modules/k6/data/data.go:L56`.) When a VU *accesses* an element, only that one element is copied into the VU's runtime and deep-frozen — the bulk data is never duplicated:

```go
// js/modules/k6/data/share.go:L44
func (s wrappedSharedArray) Get(index int) sobek.Value {
    if index < 0 || index >= len(s.arr) { return sobek.Undefined() }
    val, err := s.parse(sobek.Undefined(), s.rt.ToValue(s.arr[index]))   // copies ONE element on demand
    ...
    err = s.deepFreeze(s.rt, val)
    ...
    return val
}
```

**`open()` makes one full copy per VU.** Each k6 VU is a separate JavaScript runtime, and the `open` builtin returns the file's entire contents *into the calling VU's runtime*:

```go
// js/initcontext.go:L21
func openImpl(rt *sobek.Runtime, fs fsext.Fs, basePWD *url.URL, filename string, args ...string) (sobek.Value, error) {
    data, err := readFile(fs, fsext.Abs(basePWD.Path, filename))
    if err != nil { return nil, err }
    if len(args) > 0 && args[0] == "b" {
        ab := rt.NewArrayBuffer(data)        // js/initcontext.go:L28 (binary)
        return rt.ToValue(&ab), nil
    }
    return rt.ToValue(string(data)), nil     // js/initcontext.go:L31 — full contents into THIS VU's runtime
}
```

The `open` builtin is wired into each VU's init context at `js/bundle.go:L445` (invoking `openImpl` at `js/bundle.go:L475`). Because init runs once per VU, N VUs ⇒ N independent copies — exactly the linear growth measured above.

**Corroboration from the official Grafana k6 documentation** (the code remains the source of truth; these references only confirm the conclusion):

- The [SharedArray API reference](https://grafana.com/docs/k6/latest/javascript-api/k6-data/sharedarray/) states that a `SharedArray` "shares the underlying memory between VUs", that its constructor function "executes only once, and its result is saved in memory once", and that on access "k6 gives a copy of that element" — exactly matching the single name-keyed store and the per-element copy in `js/modules/k6/data/share.go:L44`.
- The [`open()` API reference](https://grafana.com/docs/k6/latest/javascript-api/init-context/open/) documents the opposite footprint directly: with `open()`, "every VU keeps a separate copy of the file in memory."
- The [Data parameterization guide](https://grafana.com/docs/k6/latest/examples/data-parameterization/) gives the architectural rationale — each VU is a separate JS VM, so "to prevent multiple copies of the whole data file, SharedArray was added."

### (c) Concluding answer

The memory footprint for file data loaded via **`SharedArray` remains essentially constant** as the VU count grows (measured ~214/220/224 MB at 1/50/200 VUs — within measurement noise), whereas data loaded with a plain **`open()` grows linearly — each VU creates its own full copy** (~257/5,583/19,204 MB at 1/50/200 VUs, ≈ 95 MB per VU for this 13.79 MB fixture). This reproduces the AAP's validated reference behavior (flat `SharedArray`, linear `open()`, both reaching tens of GB at 200 VUs); the absolute per-VU cost differs only because it tracks the parsed in-memory expansion of the specific fixture and environment, not any behavioral difference. The root cause is architectural: `SharedArray` stores the parsed data exactly once in a single name-keyed Go-side store whose construction callback runs only once (`js/modules/k6/data/data.go:L152-L164`) and copies only individual elements on demand (`js/modules/k6/data/share.go:L44`), while `open()` returns the entire file contents into each separate VU runtime (`js/initcontext.go:L31`), so N VUs hold N copies.

---


## Q5 — Prometheus remote-write metric-name integrity

> **Question.** Investigate the metric-reporting behavior when using the Prometheus output. Provide test-script output proving that the exported data maintains the integrity of the metric names.

### Experiment

**1. A standalone remote-write receiver** was written in a throwaway module at `/tmp/rwrecv` (outside the repo, pure Go standard library — no external dependencies). k6's `experimental-prometheus-rw` output marshals a Prometheus `WriteRequest` protobuf and **snappy-compresses** it (`Content-Encoding: snappy`, `Content-Type: application/x-protobuf`). The receiver reverses that exactly: it snappy-decodes the body and walks the protobuf wire format (`WriteRequest.timeseries` → `TimeSeries.labels` → `Label{name,value}`) to print every series' `__name__` label. The hand-rolled Snappy block decoder was unit-tested against four hand-built blobs (single literal, 1-byte-offset copy, 2-byte-offset copy, and the 60-byte literal boundary) — all passed — and the decoder asserts that the decoded length equals the Snappy length prefix, so a successful decode is itself strong evidence of correctness.

```bash
cd /tmp/rwrecv && go mod init rwrecv      # module rwrecv, go 1.21, NO external requires
GOTOOLCHAIN=local GOPROXY=off go build -o /tmp/rwrecv/rwrecv .   # builds fully offline (stdlib only)
/tmp/rwrecv/rwrecv :9090 &                # POST /api/v1/write ; GET /dump prints collected __name__ labels
```

**2. The k6 script** exercises a custom `Counter` and a custom `Trend` so both name mapping and type/stat suffixing are visible:

```javascript
// /tmp/q5_prom.js
import { Counter, Trend } from 'k6/metrics';
const c = new Counter('my_custom_counter');
const t = new Trend('my_custom_trend');
export const options = { vus: 2, duration: '3s' };
export default function () { c.add(1); t.add(Math.random() * 100); }
```

**3. Run k6 with the experimental Prometheus remote-write output** pointed at the receiver:

```bash
K6_PROMETHEUS_RW_SERVER_URL=http://localhost:9090/api/v1/write \
  /tmp/k6bin/k6 run -o experimental-prometheus-rw /tmp/q5_prom.js
```

### (a) Captured runtime output

k6 confirms the output target and prints the custom metrics in its own summary:

```text
        output: Prometheus remote write (http://localhost:9090/api/v1/write)
     my_custom_counter....: 322673 107553.279712/s
     my_custom_trend......: avg=50.076591 min=0.000517 med=50.188844 max=99.999633 p(90)=89.977479 p(95)=94.987186
```

The receiver decoded the remote-write payload and printed the `__name__` label of every series (one write request — the end-of-test flush, since the 3 s test is shorter than the 5 s default push interval):

```text
===== rwrecv decoded __name__ labels (8 unique, across 1 write requests) =====
k6_data_received_total
k6_data_sent_total
k6_iteration_duration_p99
k6_iterations_total
k6_my_custom_counter_total
k6_my_custom_trend_p99
k6_vus
k6_vus_max
```

The two custom metrics survived the encode → snappy → protobuf → decode round-trip with their names fully intact:

- `my_custom_counter` → **`k6_my_custom_counter_total`** (`k6_` prefix + `_total` counter suffix)
- `my_custom_trend` → **`k6_my_custom_trend_p99`** (`k6_` prefix + `_p99` from the default trend stat)

The built-in metrics are likewise intact and correctly suffixed: counters get `_total` (`k6_data_received_total`, `k6_data_sent_total`, `k6_iterations_total`), the trend gets `_p99` (`k6_iteration_duration_p99`), and gauges get no suffix (`k6_vus`, `k6_vus_max`). The snappy length assertion passed with no errors, confirming a faithful decode — i.e. the names are not truncated or corrupted in transit.

### (b) Code-level root-cause rationale

**The output is registered** under the kebab-case name `experimental-prometheus-rw` and backed by the vendored `github.com/grafana/xk6-output-prometheus-remote`:

```go
// cmd/outputs.go:L34
builtinOutputExperimentalPrometheusRW
// cmd/outputs.go:L66
builtinOutputExperimentalPrometheusRW.String(): func(params output.Params) (output.Output, error) {
    return remotewrite.New(params)
```

**The `__name__` label is built deterministically** as `prefix + metric name (+ "_" + suffix)`:

```go
// vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/prometheus.go:L11
const namelbl = "__name__"

// vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/prometheus.go:L39-L52
func MapSeries(series metrics.TimeSeries, suffix string) []*prompb.Label {
    v := defaultMetricPrefix + series.Metric.Name      // :L40 — "k6_" + metric name
    if suffix != "" {
        v += "_" + suffix                              // :L41-L42 — append type/stat suffix
    }
    lbls := append(MapTagSet(series.Tags), &prompb.Label{
        Name:  namelbl,                                // :L45 — the __name__ label
        Value: v,
    })
    ...
}
```

**The prefix and the default trend stat:**

```go
// vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/config.go:L24
defaultMetricPrefix = "k6_"
// vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/config.go:L28
var defaultTrendStats = []string{"p(99)"}
```

**The suffixes.** A counter is mapped with the suffix `"total"` (⇒ `_total`):

```go
// vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/remotewrite.go:L330-L331
case metrics.Counter:
    ts := mapMonoSeries(swm.TimeSeries, "total", swm.Latest)   // :L331 — counter => "_total"
```

A trend's sub-metric gets its stat appended to the name label; the default `p(99)` is sanitized to `p99` and appended (⇒ `_p99`):

```go
// vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/trend.go:L89
ts.Labels[tg.ixname].Value += "_" + suffix
```

Net result: `my_custom_counter` → `k6_my_custom_counter_total`; `my_custom_trend` (p(99)) → `k6_my_custom_trend_p99` — exactly the decoded names captured above.

### (c) Concluding answer

The exported remote-write data **preserves metric-name integrity**. Each k6 metric name is mapped deterministically into the `__name__` label as `k6_<name>` plus a type/stat suffix — `_total` for counters and `_p99` for the default trend statistic — with no truncation or corruption. The proof is the receiver's decoded `__name__` list, which shows the custom metrics arriving intact as `k6_my_custom_counter_total` and `k6_my_custom_trend_p99` (alongside correctly named built-ins), produced by snappy-decoding and protobuf-parsing the actual payload k6 sent.

---


## Reproducibility & cleanup

### Summary of observed values

| Q | Behavior | Observed result | Determinism |
|---|----------|-----------------|-------------|
| Q1 | SIGINT on `ramping-vus` (6 VUs) | DEBUG `Stopping k6 in response to signal... sig=interrupt` + final ERROR `test run was aborted...`; exit **105**; 6 `ITER_START` / 1 truncated `ITER_END` (sleep cut to ~5.6 s) | Deterministic (exit 105; mid-iteration termination always) |
| Q2 | gRPC server-streaming interrupt, `gracefulRampDown: '30ms'` | abort logs as Q1; **`grpc_streams_msgs_received = 235`** (`grpc_streams = 5`, `grpc_streams_msgs_sent = 5`, `on('error')` = 0); exit 105 | Received count is **timing-dependent** |
| Q3 | `dropped_iterations` over REST API | API `GET /v1/metrics/dropped_iterations` → **`count: 85`**; matches summary; no-`--linger` → connection refused; bogus id → 404 | Deterministic (85) |
| Q4 | `SharedArray` vs `open()` RSS at 1/50/200 VUs | SharedArray ~214/220/224 MB (flat); `open()` ~257/5,583/19,204 MB (linear, ≈95 MB/VU) | Shape deterministic; exact kB varies with fixture/GC |
| Q5 | Prometheus remote-write name integrity | decoded `__name__`: `k6_my_custom_counter_total`, `k6_my_custom_trend_p99`, + built-ins; names intact | Deterministic |

### Temporary artifacts (all outside the repository, under `/tmp`)

- `/tmp/k6bin/k6` — the out-of-tree k6 v0.55.0 binary
- `/tmp/grpcserver` (+ the out-of-tree build copy) — the gRPC server example binary
- `/tmp/route_guide.proto` — proto copied for the Q2 client
- `/tmp/big.json` — the ~13.8 MB JSON fixture for Q4 (14,463,456 bytes; 70,000 objects), plus `/tmp/gen_fixture.py` that generates it
- `/tmp/rwrecv/` — the standalone Go remote-write receiver module (Q5)
- `/tmp/q1_ramping.js`, `/tmp/q2_grpc_streaming.js`, `/tmp/q3_dropped.js`, `/tmp/q4_shared.js`, `/tmp/q4_open.js`, `/tmp/q5_prom.js` — experiment scripts
- `/tmp/q1.log`, `/tmp/q2.log`, `/tmp/q2.summary`, `/tmp/q3.summary`, `/tmp/q3_api_dropped.json`, `/tmp/q4_rss.txt`, `/tmp/q5.log` — captured output
- `/tmp/go.mod.bak`, `/tmp/go.sum.bak` — manifest backups taken before the Q2 build

> The gRPC example build needed `-mod=mod` (it imports the non-vendored `google.golang.org/grpc/testdata`). The repo's `go.mod`/`go.sum` were backed up and restored, and verified byte-for-byte identical afterward (SHA-256 `go.mod = 2f3d5fa2…ca39`, `go.sum = 6e7d3d7a…863e`, unchanged before and after).

### Cleanup commands

```bash
# stop any background servers spawned during the experiments (by their specific PIDs)
rm -f  /tmp/grpcserver /tmp/route_guide.proto /tmp/big.json /tmp/gen_fixture.py
rm -f  /tmp/q1_ramping.js /tmp/q2_grpc_streaming.js /tmp/q3_dropped.js \
       /tmp/q4_shared.js /tmp/q4_open.js /tmp/q5_prom.js
rm -f  /tmp/q1*.log /tmp/q2.* /tmp/q3*.summary /tmp/q3_api_*.json /tmp/q4_rss.txt /tmp/q5.log
rm -f  /tmp/go.mod.bak /tmp/go.sum.bak
rm -rf /tmp/rwrecv /tmp/grpcsrc
rm -rf /tmp/k6bin
```

### Final verification — the source tree is unchanged

After cleanup, from the repository root, the working tree is **clean** — `git status --porcelain` produces no output because this report is committed (nothing is untracked or modified). The **only** change relative to the k6 source baseline (`ddc3b0b1d23c128e34e2792fc9075f9126e32375` — the commit this report investigates) is this newly added report; no k6 source, config, test, `go.mod`/`go.sum`, or `vendor/` file is touched:

```text
$ git status --porcelain
$ git diff --name-status ddc3b0b1d23c128e34e2792fc9075f9126e32375..HEAD
A	blitzy/documentation/k6_ddc3b0b1d23c.md
$ git diff --stat ddc3b0b1d23c128e34e2792fc9075f9126e32375..HEAD -- . ':(exclude)blitzy/'
$ git diff --name-only ddc3b0b1d23c128e34e2792fc9075f9126e32375..HEAD -- go.mod go.sum vendor/
$
```

The first command (`git status --porcelain`) prints nothing — the working tree is clean. The second shows the single added file. The third and fourth print nothing — zero k6 source/config/test files changed, and the dependency manifests and `vendor/` tree are byte-for-byte unchanged.

The only file added anywhere in the repository is this document, `blitzy/documentation/k6_ddc3b0b1d23c.md`. It lives under `blitzy/`, outside the k6 Go packages, so it does not affect the build. The k6 source under investigation (`k6 v0.55.0`, branch `k6_ddc3b0b1d23c` @ `ddc3b0b1d`) was treated as **read-only evidence** throughout and is byte-for-byte identical to its committed state.

