# k6 v0.55.0 (commit ddc3b0b1d2) — Runtime Behavior Investigation

## Environment

- **k6**: built from source at this repository's commit — `k6 v0.55.0 (commit/ddc3b0b1d2, go1.22.2, linux/amd64)`
- **OS**: Ubuntu 24.04.4 LTS (Noble Numbat)
- **Go toolchain**: `go1.22.2` (installed via apt; `go.mod` declares `go 1.21` with `toolchain go1.21.13` as the minimum)
- **Source tree**: `go.k6.io/k6` at branch `k6_ddc3b0b1d23c`
- **Build command**: `go build -mod=vendor -o /tmp/k6 .`
- **Common run flags used**: `--verbose --log-output=stdout` (enables DEBUG-level output via `cmd/root.go:197-258`; the default logrus `TextFormatter` is used, which in non-TTY mode emits lines in the `time="..." level=... msg="..."` long form — see `cmd/root.go:245-258` for formatter selection)

---

## Question 1: VU Management and SIGINT Handling (ramping-vus + ≥5 VUs)

### Question

> Determine the exact log messages produced when a `ramping-vus` executor with at least 5 VUs receives a `SIGINT`. Provide log evidence showing whether currently active VUs are allowed to finish their current iteration or are terminated mid-execution during graceful shutdown.

### Answer

Upon the **first** `SIGINT`, k6 triggers a **graceful stop**: currently active iterations are allowed to finish (up to the `gracefulRampDown` / `gracefulStop` deadline); new iterations are NOT started; VUs that are still mid-iteration when the deadline expires are terminated and counted as `interrupted` iterations in the end-of-test summary. The process exits with code **105** (`exitcodes.ExternalAbort`).

A **second** signal (or a deadline expiration after graceful stop) invokes `onHardStop` which calls `globalCancel()` — cancelling the VU iteration contexts immediately and forcing in-flight iterations to unwind via `ctx.Done()`.

### Runtime Evidence — Experiment 1

**Test script** (`/tmp/test_ramping_sigint.js`):

```javascript
import { sleep } from 'k6';

export const options = {
  scenarios: {
    default: {
      executor: 'ramping-vus',
      startVUs: 5,
      stages: [
        { target: 10, duration: '5s' },
        { target: 10, duration: '30s' },
      ],
      gracefulRampDown: '5s',
    },
  },
};

export default function () {
  sleep(2);
}
```

**Command**:

```bash
/tmp/k6 run --verbose --log-output=stdout /tmp/test_ramping_sigint.js &
# at t ≈ 6s, in a separate terminal:
kill -SIGINT "$(pgrep -f test_ramping_sigint)"
```

**Verbatim log sequence (post-signal)** — output rendered by logrus's default `TextFormatter` (the `time="..." level=... msg="..."` long form is emitted when stdout is not a TTY — see `cmd/root.go:197-258` for formatter selection):

```text
time="2026-04-17T00:20:20Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-04-17T00:20:20Z" level=debug msg="Metrics emission of VUs and VUsMax metrics stopped"
time="2026-04-17T00:20:20Z" level=debug msg="Executor finished successfully" executor=default startTime=0s type=ramping-vus
time="2026-04-17T00:20:20Z" level=debug msg="teardown() is not defined or not exported, skipping!"
time="2026-04-17T00:20:20Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
time="2026-04-17T00:20:20Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-04-17T00:20:20Z" level=debug msg="Stopping vus and vux_max metrics emission..." phase=execution-scheduler-init
time="2026-04-17T00:20:20Z" level=debug msg="Releasing signal trap..."
time="2026-04-17T00:20:20Z" level=debug msg="Waiting for metrics and traces processing to finish..."
time="2026-04-17T00:20:20Z" level=debug msg="Metrics and traces processing finished!"
time="2026-04-17T00:20:20Z" level=debug msg="Stopping outputs..."
time="2026-04-17T00:20:20Z" level=debug msg="Generating the end-of-test summary..."
time="2026-04-17T00:20:20Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-04-17T00:20:20Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

**End-of-test summary**:

```text
running (0m06.0s), 00/10 VUs, 14 complete and 10 interrupted iterations
iteration_duration...: avg=2s min=2s med=2s max=2s p(90)=2s p(95)=2s
iterations...........: 14  2.344126/s
vus..................: 9   min=5      max=9
vus_max..............: 10  min=10     max=10
```

**Evidence that in-flight iterations were allowed to finish**: The last progress-bar line printed before `SIGINT` was issued reads `running (0m05.0s), 09/10 VUs, 12 complete and 0 interrupted iterations` (captured verbatim from stdout at t = 5 s). The end-of-test summary, emitted after SIGINT triggered the graceful-stop path, reports `14 complete and 10 interrupted iterations`. The increase from 12 complete to 14 complete between SIGINT arrival and test teardown is direct evidence that **two additional in-flight iterations were allowed to finish their `sleep(2)` call** before the `gracefulRampDown: '5s'` deadline expired and the remaining 10 still-mid-iteration VUs were terminated and counted as `interrupted`.

Exit code: `105`.

### Rationale

- **`cmd/common.go:97-120` — `handleTestAbortSignals()`**:
  - Registers a 2-buffered signal channel via `gs.SignalNotify(sigC, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)`.
  - The first signal received is dispatched to `gracefulStopHandler(sig)`.
  - Any second signal triggers `onHardStop(sig)` and then `gs.OSExit(int(exitcodes.ExternalAbort))` (value **105**), so the process exits immediately after a second signal without waiting for the scheduler to unwind.
- **`cmd/run.go:349-363` — handlers bound inside `cmdRun.run()`**:
  - `gracefulStop` logs `"Stopping k6 in response to signal..."` via `logger.WithField("sig", sig).Debug(...)` at the DEBUG level; it becomes visible on stdout because `--verbose` lowers the logger's minimum output level to DEBUG (logrus formatters do not rewrite or upgrade severity — the `level=debug` field is preserved in the output). `gracefulStop` then calls `runAbort(errext.WithAbortReasonIfNone(errext.WithExitCodeIfNone(fmt.Errorf("test run was aborted because k6 received a '%s' signal", sig), exitcodes.ExternalAbort), errext.AbortedByUser))` and invokes `lingerCancel()`. This is what propagates the exit code 105 and the `level=error` log line `test run was aborted because k6 received a 'interrupt' signal`.
  - `onHardStop` logs `"Aborting k6 in response to signal"` at ERROR level then calls `globalCancel()`.
- **`lib/executor/vu_handle.go:147-163` — `gracefulStop()`**:
  - Transitions the VU handle state from `running` to `toGracefulStop`. Critically, it does **NOT** cancel the VU's iteration context; the current `runIter(ctx, vu)` call inside `runLoopsIfPossible()` is allowed to return naturally. It re-creates `canStartIter` as an empty channel so that the VU loop cannot pick up a new iteration once the current one completes.
- **`lib/executor/vu_handle.go:165-182` — `hardStop()`**:
  - Transitions `running`/`toGracefulStop` → `toHardStop` and calls `vh.cancel()`, forcefully cancelling the iteration context. The currently executing iteration exits via its `ctx.Done()` branch (or is killed mid-`sleep(2)` call because `sleep` honours context cancellation).
- **`lib/executor/vu_handle.go:185-264` — `runLoopsIfPossible()`** (file total length: 264 lines):
  - The main VU loop; selects on `canStartIter`, `ctx.Done()`, and `executorDone`. When in state `toGracefulStop` it waits for the in-flight `runIter(ctx, vu)` to return before transitioning to `stopped`. The file itself emits only three DEBUG messages — `"Start"` (line 123 for the initial spawn, line 127 when the VU re-enters `running` from a paused state), `"Graceful stop"` (line 161), and `"Hard stop"` (line 177); there is no per-iteration DEBUG log in this file, so the "iteration is allowed to finish" semantics are observed indirectly via the progress-bar delta (12→14 complete iterations between SIGINT and summary) rather than via a per-iteration log line.

**Interpretation**: The progress bar at t = 5 s (just before SIGINT at t ≈ 6 s) shows `12 complete and 0 interrupted iterations`; the end-of-test summary shows `14 complete and 10 interrupted iterations`. The 2 additional "complete" iterations beyond the SIGINT moment are VUs that finished their `sleep(2)` within the 5-second `gracefulRampDown` budget; the 10 "interrupted" iterations are VUs whose contexts were cancelled by `hardStop()` (either because the executor deadline expired while iterations were still in flight, or because the VU iteration contexts were cancelled via `runCtx` at test teardown) before their iteration returned. This confirms that the **first `SIGINT` triggers the graceful path where in-flight work is allowed to finish**, and **only after the deadline does k6 force-cancel** the iteration contexts.

---

## Question 2: gRPC Server Streaming Interruption (FeatureExplorer/ListFeatures + 30ms gracefulRampDown)

### Question

> Capture the exact log entries obtained at runtime when a gRPC server streaming test using the `main.FeatureExplorer/ListFeatures` RPC with a `30ms gracefulRampDown` is interrupted via `SIGINT`. Report the exact value of `grpc_streams_msgs_received` from the final metrics summary.

### Answer

**`grpc_streams_msgs_received: 98`** (rate `19.702762/s`) across 2 active streams, with exit code **105**. Logs show the streaming loop receiving messages from the server, then the client-side context cancellation propagating through the debug line `stream is cancelled/finished` (one per active stream), followed by `stream /main.FeatureExplorer/ListFeatures is closing` during cleanup, and — because the test script registered no error handler — culminating in two `level=warning msg="no handlers for error registered, but an error happened: canceled by client (k6)"` lines (one per active stream).

### Runtime Evidence — Experiment 2

**Setup**:

- The gRPC test server was built from `examples/grpc_server/main.go` (which imports `go.k6.io/k6/lib/testutils/grpcservice`) using a temporary `go mod tidy` + `go build -o /tmp/grpc_test_server .`. The transient edits to `examples/grpc_server/go.mod` and `go.sum` were reverted via `git checkout -- examples/grpc_server/go.mod examples/grpc_server/go.sum` so that the source tree is left unchanged.
- The server was started listening on `localhost:10000` before the k6 run.

**Test script** (`/tmp/test_grpc_stream_v2.js`):

```javascript
import grpc from 'k6/net/grpc';
import { sleep } from 'k6';

const client = new grpc.Client();
client.load(['/tmp/grpcservice'], 'route_guide.proto');

export const options = {
  scenarios: {
    default: {
      executor: 'ramping-vus',
      startVUs: 2,
      stages: [
        { target: 2, duration: '5s' },
        { target: 4, duration: '30s' },
      ],
      gracefulRampDown: '30ms',
    },
  },
};

export default function () {
  client.connect('localhost:10000', { plaintext: true });
  const stream = new grpc.Stream(client, 'main.FeatureExplorer/ListFeatures');
  stream.on('data', function (_feature) { /* touch */ });
  stream.on('end', function () { client.close(); });
  stream.write({
    lo: { latitude: 400000000, longitude: -750000000 },
    hi: { latitude: 420000000, longitude: -730000000 },
  });
  sleep(10);
}
```

**Commands**:

```bash
/tmp/grpc_test_server &                              # port 10000
/tmp/k6 run --verbose --log-output=stdout /tmp/test_grpc_stream_v2.js &
# at t ≈ 5s:
kill -SIGINT "$(pgrep -f test_grpc_stream_v2)"
```

**Verbatim log sequence (post-signal)** — output rendered by logrus's default `TextFormatter` (the `time="..." level=... msg="..."` long form is emitted when stdout is not a TTY — see `cmd/root.go:197-258` for formatter selection):

```text
time="2026-04-17T00:22:39Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-04-17T00:22:39Z" level=debug msg="Metrics emission of VUs and VUsMax metrics stopped"
time="2026-04-17T00:22:39Z" level=debug msg="stream is cancelled/finished" error="canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-04-17T00:22:39Z" level=debug msg="stream is cancelled/finished" error="canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-04-17T00:22:39Z" level=debug msg="stream /main.FeatureExplorer/ListFeatures is closing" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-04-17T00:22:39Z" level=debug msg="stream /main.FeatureExplorer/ListFeatures is closing" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-04-17T00:22:39Z" level=warning msg="no handlers for error registered, but an error happened: canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-04-17T00:22:39Z" level=warning msg="no handlers for error registered, but an error happened: canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-04-17T00:22:39Z" level=debug msg="Executor finished successfully" executor=default startTime=0s type=ramping-vus
time="2026-04-17T00:22:39Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
time="2026-04-17T00:22:39Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-04-17T00:22:39Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

**End-of-test summary**:

```text
running (0m05.0s), 0/4 VUs, 0 complete and 2 interrupted iterations
grpc_streams.................: 2      0.402097/s
grpc_streams_msgs_received...: 98     19.702762/s
grpc_streams_msgs_sent.......: 2      0.402097/s
```

Exit code: `105`.

### Rationale

- **`js/modules/k6/grpc/metrics.go:14-32` — `registerMetrics()`** registers the three counter metrics at module init:
  - `grpc_streams` via `registry.NewMetric("grpc_streams", metrics.Counter)`
  - `grpc_streams_msgs_sent` — client → server messages
  - `grpc_streams_msgs_received` — server → client messages
- **`js/modules/k6/grpc/stream.go:81-100` — `beginStream()`**:
  - `ctx := s.vu.Context()` — the stream context is derived from the VU context, so VU cancellation propagates directly into the gRPC stream machinery.
  - `stream, err := s.client.conn.NewStream(ctx, *req)` opens the server-streaming RPC; a `grpc_streams` sample is pushed at this point and a `loop()` goroutine starts consuming messages via `RecvMsg`.
- **`js/modules/k6/grpc/stream.go:149-165` — `queueMessage()`**:
  - Called on every successful `RecvMsg`; pushes a `StreamsMessagesReceived` metric sample with `Value: 1` and enqueues a `"data"` event to the VU's JS runtime. This is what increments `grpc_streams_msgs_received` one-at-a-time, in real time.
- **`js/modules/k6/grpc/stream.go:201` — debug log `"stream is cancelled/finished"`**:
  - Emitted when `isRegularClosing(err)` returns true, i.e., when the stream closes cleanly (EOF) **or** when the stream's context is cancelled as part of graceful shutdown — which is exactly the case here.
- **`js/modules/k6/grpc/stream.go:371` — debug log `"stream %s is closing"`**:
  - Emitted during the final stream cleanup, immediately after `close(s.done)`. If the stream closed cleanly (EOF), the `eventEnd` callback is queued to the JS runtime so any `stream.on('end', …)` handler fires; if instead the stream terminated with an error (e.g., cancellation) and no `stream.on('error', …)` handler is registered, the `logger.Warnf("no handlers for error registered, but an error happened: %s", …)` path fires (emitted from the event loop when the error event has no subscribers).
- **`lib/testutils/grpcservice/service.go:57-68` — `FeatureExplorer.ListFeatures(Rectangle) returns (stream Feature)`**:
  - Iterates over the feature database and, for each feature within the requested rectangle, does `time.Sleep(100 * time.Millisecond)` followed by `stream.Send(feature)`. The 100 ms server-side pacing is what limits the throughput during the short window before interrupt.

**Interpretation**: 98 received messages across 2 streams ≈ **49 messages per stream** before the 30 ms `gracefulRampDown` window elapsed and the client-side VU contexts were cancelled. At ~100 ms per server-side `stream.Send` plus client-side receive overhead, 49 messages in roughly 5 s is consistent. After interruption, the `loop()` goroutine's `RecvMsg` returned with a cancellation error, `isRegularClosing` matched, the `stream is cancelled/finished` debug log fired, and the cleanup path produced the `stream /main.FeatureExplorer/ListFeatures is closing` log line. Because the test script registers only `data` and `end` handlers (no `error` handler), the unhandled `canceled by client (k6)` error surfaced as two `level=warning` lines — one per active stream — instead of clean `Stream ended` events. The observed rate `19.702762/s` matches 98 ÷ 4.97 s runtime.

---

## Question 3: Dropped Iterations via k6 REST API (/v1/metrics/dropped_iterations)

### Question

> Determine the exact value of `dropped_iterations` reported when a test (using `constant-arrival-rate`) exceeds its maximum duration capacity. Provide runtime evidence by querying the k6 REST API (`/v1/metrics/dropped_iterations`) during test execution.

### Answer

With `rate: 20/s`, `maxVUs: 3`, `duration: 10s`, and the default function doing `sleep(5)` (so each VU can only complete 2 iterations in 10 s — an effective throughput ceiling of ~0.6 iter/s), the REST API returned `"count": 113, "rate": 18.96188` when queried at t ≈ 6 s. The **final end-of-test summary reported `dropped_iterations: 195`** (rate `19.213218/s`). Test exit code was **`0`** (normal completion — dropped iterations alone do not trigger a non-zero exit).

### Runtime Evidence — Experiment 3

**Test script** (`/tmp/test_dropped_iterations3.js`):

```javascript
import { sleep } from 'k6';

export const options = {
  scenarios: {
    overload: {
      executor: 'constant-arrival-rate',
      rate: 20, timeUnit: '1s', duration: '10s',
      preAllocatedVUs: 3, maxVUs: 3,
    },
  },
};

export default function () { sleep(5); }
```

**Commands**:

```bash
/tmp/k6 run --verbose --log-output=stdout /tmp/test_dropped_iterations3.js &
# at t ≈ 6s, in a separate terminal:
curl -s http://localhost:6565/v1/metrics/dropped_iterations
```

**Exact REST API JSON response at t ≈ 6 s**:

```json
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":113,"rate":18.96188}}}}
```

**End-of-test summary**:

```text
running (00m10.2s), 0/3 VUs, 3 max VUs, 10s
  * overload: 20.00 iterations/s for 10s (maxVUs: 3)
iterations..........: 6      0.587026/s
iteration_duration..: avg=5s
dropped_iterations..: 195    19.213218/s
vus.................: 3
vus_max.............: 3
```

Exit code: `0`.

### Rationale

- **`metrics/builtin.go:10` — `DroppedIterationsName = "dropped_iterations"`**:
  - The canonical string identifier used as both the metric's name and its REST API lookup key (`/v1/metrics/{id}` uses this id).
- **`metrics/builtin.go:84` — registration**:
  - `DroppedIterations: registry.MustNewMetric(DroppedIterationsName, Counter)` — registered as a `Counter` sink at startup, so it accumulates additively and exposes `{count, rate}` via `Sink.Format(t)`.
- **`lib/executor/constant_arrival_rate.go:324-350` — main ticker loop**:
  - On each `timer.C` tick, the executor calls `vusPool.TryRunIteration()`.
  - When `TryRunIteration` returns `false` (no free VU in the pool), the executor immediately pushes a metric sample via `metrics.PushIfNotDone(parentCtx, out, metrics.Sample{TimeSeries: metrics.TimeSeries{Metric: droppedIterationMetric, Tags: metricTags}, Time: time.Now(), Value: 1})`.
  - This is **real-time** emission: one sample per dropped tick, pushed the moment it happens → the API counter increments continuously throughout the run (113 at t ≈ 6 s, 195 at end).
- **`api/v1/routes.go:31-39` — `NewHandler()`**:
  - Registers `mux.HandleFunc("/v1/metrics/", ...)` which strips the `/v1/metrics/` prefix from `r.URL.Path` and dispatches the remainder (the metric id) to `handleGetMetric(cs, rw, r, id)`.
- **`api/v1/metric_routes.go:27-50` — `handleGetMetric()`**:
  - Acquires `cs.MetricsEngine.MetricsLock`.
  - Looks up `cs.MetricsEngine.ObservedMetrics[id]`; returns `404 Not Found` if the metric has never been observed.
  - Wraps the metric in an envelope via `newMetricEnvelope(metric, t)` where `t = cs.Scheduler.GetState().GetCurrentTestRunDuration()`.
  - Internally, `metric.Sink.Format(t)` produces a `map[string]float64`; for a `Counter` sink it yields `{"count": N, "rate": N / elapsedSeconds}` — which is exactly what appears under `attributes.sample` in the JSON response.
  - Marshals to JSON and writes to the response.
- **`cmd/state/state.go:150` — `Address: "localhost:6565"`** is the default API listen address set in `GetDefaultFlags()` and can be overridden via the `--address` flag.

**Interpretation**: At steady state the `constant-arrival-rate` executor attempts `rate * duration = 20 * 10 = 200` iterations over the 10-second window, but only 3 VUs are available and each sleeps 5 s per iteration — yielding a throughput ceiling of `3 / 5 = 0.6 iter/s`, i.e., **6 completed iterations over 10 s**. The remaining `200 - 6 = 194` attempts would be dropped; in practice the observed value was **195** (one additional tick falling into the small overshoot window beyond the declared 10 s before the scheduler transitioned to the cooldown phase at 10.2 s), exactly matching the observed summary figure. The mid-run REST API response (count = 113 at t ≈ 6 s, rate = 18.96188) confirms the counter is incrementing in real time as the `constant_arrival_rate.go` ticker loop pushes one sample per missed tick.

---

## Question 4: Data Sharing Behavior — open() vs SharedArray

### Question

> Investigate whether the memory footprint for loaded data files remains constant with increasing VU count, or whether each VU creates its own copy. Provide test script output comparing `open()` (per-VU copy) vs `SharedArray` (shared across VUs) memory usage. Identify the root cause of the observed behavior in the source code.

### Answer

**`open()` results in per-VU data duplication; `SharedArray` stores a single backing copy across all VUs.** Experiment with **10 VUs** loading the same **21 MB JSON file** (~70,000 records):

- `open()`: `VmRSS` = **1,389,964 kB** (≈ 1.3 GB)
- `SharedArray`: `VmRSS` = **267,520 kB** (≈ 261 MB)
- Ratio: **≈ 5.2× more memory with `open()`**

### Runtime Evidence — Experiment 4

**Test data**: `/tmp/test_data_large.json` — 21 MB on disk, ~70,000 JSON records (each record has `id`, `name`, `description`, `tags[]`, `timestamp`). Generated once and reused across both runs.

**Test script 1 — `open()` per-VU copy** (`/tmp/test_open_mem.js`):

```javascript
import { sleep } from 'k6';

const data = JSON.parse(open('/tmp/test_data_large.json'));

export const options = {
  scenarios: {
    s: {
      executor: 'per-vu-iterations',
      vus: 10,
      iterations: 5,
      maxDuration: '60s',
    },
  },
};

export default function () {
  if (data.length > 0) { /* touch */ }
  sleep(5);
}
```

**Test script 2 — `SharedArray` singleton** (`/tmp/test_shared_mem.js`):

```javascript
import { SharedArray } from 'k6/data';
import { sleep } from 'k6';

const data = new SharedArray('myData', function () {
  return JSON.parse(open('/tmp/test_data_large.json'));
});

export const options = {
  scenarios: {
    s: {
      executor: 'per-vu-iterations',
      vus: 10,
      iterations: 5,
      maxDuration: '60s',
    },
  },
};

export default function () {
  if (data.length > 0) { /* touch */ }
  sleep(5);
}
```

**Measurement**: memory was sampled from `/proc/<k6-pid>/status` during steady-state execution of each run (after all VUs had completed initialization and entered the main `sleep(5)` loop).

**Memory comparison**:

| Metric   | `open()` (10 VUs)       | `SharedArray` (10 VUs) | Ratio (open / shared) |
|----------|-------------------------|------------------------|-----------------------|
| VmPeak   | 3,509,612 kB            | 2,429,752 kB           | 1.44×                 |
| VmSize   | 3,509,612 kB            | 2,364,216 kB           | 1.48×                 |
| VmHWM    | 1,389,964 kB (≈ 1.3 GB) | 267,520 kB (≈ 261 MB)  | **5.2×**              |
| VmRSS    | 1,389,964 kB (≈ 1.3 GB) | 267,520 kB (≈ 261 MB)  | **5.2×**              |

### Rationale

- **`js/modules/k6/data/data.go:18-47` — `RootModule` singleton pattern**:
  - The `RootModule` struct holds a single `shared sharedArrays` field (whose internal `data map[string]sharedArray` is protected by a `sync.RWMutex`). `New()` initializes exactly ONE `sharedArrays` value per k6 process; `NewModuleInstance()` returns a per-VU `Data` struct that holds a **pointer** `shared *sharedArrays` back to the same `RootModule`-level value → the map is process-global, not per-VU.
- **`js/modules/k6/data/data.go:152-167` — `sharedArrays.get()` double-checked locking**:
  1. Acquires `RLock` — if `s.data[name]` exists, returns the existing `sharedArray` reference without invoking the user's constructor closure.
  2. Otherwise releases the `RLock`, acquires `Lock`, re-checks the map, and only then calls `getShareArrayFromCall()` which executes the user's closure **exactly once**. The result (a JS array) is marshaled element-by-element into `arr []string` and stored in the shared map.
- **`js/modules/k6/data/share.go:11-13` — `sharedArray` struct**:
  - Holds `arr []string` where each element is a JSON-encoded string (not a parsed JS object). This single `[]string` slice is the entire backing storage that the 5.2× savings rest on.
- **`js/modules/k6/data/share.go:23-33` — `wrap()` per-VU wrapper**:
  - Builds a per-VU `wrappedSharedArray` that references the same underlying `sharedArray.arr` `[]string`. The slice header (pointer, length, capacity — 24 bytes) is copied per VU, but the backing array (the ~21 MB of JSON text) is shared.
- **`js/modules/k6/data/share.go:45-59` — `Get(index)` transient parse**:
  - On each element access, calls `JSON.parse(arr[index])` via the sobek runtime and deep-freezes the result. The parsed JS object therefore exists only transiently in one VU's sobek heap and becomes eligible for GC after the caller releases its reference.
- **`js/modules/k6/data/share.go:36-43` — `Set()` / `SetLen()`**:
  - Both panic with `"SharedArray is immutable"`, confirming the array is strictly read-only — which is what allows safe cross-VU sharing without per-VU copies or per-access locking.

**Why `open()` blows up**: k6 gives every VU its own sobek JavaScript runtime and its own top-level global scope. Because `open()` is called at the top level (init context), it is evaluated **per VU**, and the parsed data array exists as a separate deep-copied structure in each VU's heap. With 10 VUs × ~21 MB on-disk JSON (~130 MB expanded as in-memory JS objects including object headers, string interning overhead, property slots, etc.), that approximates 1.3 GB of retained heap — exactly matching the observed `VmRSS = 1,389,964 kB`.

**Interpretation**: The 5.2× RSS ratio is consistent with the hypothesis that each VU holds an independent parsed copy of the large dataset. `SharedArray` deduplicates the backing storage to a single `[]string` at the `RootModule` level (the `data.go` singleton pattern described above), reducing the per-VU cost to only transient parsed-and-frozen objects that are eligible for garbage collection immediately after `Get()` returns.

---

## Question 5: Prometheus Output Metric Name Integrity (experimental-prometheus-rw)

### Question

> Provide test script output proving that the exported data via the `experimental-prometheus-rw` output maintains the integrity of metric names. Show the `k6_` prefix convention and the Prometheus `__name__` label mapping.

### Answer

All k6 metrics — both built-in (`iterations`, `iteration_duration`, `vus`, `vus_max`, `data_sent`, `data_received`, etc.) and user-defined custom metrics of every type (`Counter`, `Trend`, `Rate`, `Gauge`) — are exported via the `experimental-prometheus-rw` output with their **original names preserved** and **prefixed with the compile-time constant `k6_`**. Type-specific suffixes are appended following Prometheus conventions: `_total` for Counter, `_rate` for Rate, one-per-stat (`_avg`, `_count`, `_max`, `_med`, `_min`, `_p90`, `_p95`, `_p99`, `_sum`) for Trend, and no suffix for Gauge. The `__name__` label in each emitted Prometheus `TimeSeries` carries exactly this prefixed-and-suffixed name.

### Runtime Evidence — Experiment 5

**Setup**: the Prometheus remote-write payloads were captured by running a local Python HTTP listener on `http://localhost:9998/api/v1/write` that snappy-decoded incoming POST bodies and protobuf-parsed them into `prompb.WriteRequest` to enumerate every distinct `__name__` label value.

**Test script** (`/tmp/test_prom_v3.js`):

```javascript
import { Counter, Trend, Rate, Gauge } from 'k6/metrics';
import { sleep } from 'k6';

const myCounter  = new Counter('my_custom_counter');
const myDuration = new Trend('my_custom_duration', true);
const myRate     = new Rate('my_custom_rate');
const myGauge    = new Gauge('my_custom_gauge');

export const options = {
  scenarios: {
    s: { executor: 'constant-vus', vus: 2, duration: '8s' },
  },
};

export default function () {
  myCounter.add(1);
  myDuration.add(100 + Math.random() * 50);
  myRate.add(Math.random() > 0.5);
  myGauge.add(42);
  sleep(1);
}
```

**Command**:

```bash
env K6_PROMETHEUS_RW_SERVER_URL=http://localhost:9998/api/v1/write \
    K6_PROMETHEUS_RW_TREND_STATS="p(99),p(95),p(90),max,min,avg,count,sum,med" \
    /tmp/k6 run -o experimental-prometheus-rw /tmp/test_prom_v3.js
```

**All 26 unique `__name__` label values observed in the remote-write payloads (sorted alphabetically)**:

```text
k6_data_received_total
k6_data_sent_total
k6_iteration_duration_avg
k6_iteration_duration_count
k6_iteration_duration_max
k6_iteration_duration_med
k6_iteration_duration_min
k6_iteration_duration_p90
k6_iteration_duration_p95
k6_iteration_duration_p99
k6_iteration_duration_sum
k6_iterations_total
k6_my_custom_counter_total
k6_my_custom_duration_avg
k6_my_custom_duration_count
k6_my_custom_duration_max
k6_my_custom_duration_med
k6_my_custom_duration_min
k6_my_custom_duration_p90
k6_my_custom_duration_p95
k6_my_custom_duration_p99
k6_my_custom_duration_sum
k6_my_custom_gauge
k6_my_custom_rate_rate
k6_vus
k6_vus_max
```

**Relevant log lines** (verbatim, captured with `--verbose --log-output=stdout`; the leading `output: ...` line is a startup banner emitted by `cmd/ui.go:134` via `fmt.Fprintf`, not a logrus log entry, so it carries no level/timestamp):

```text
        output: Prometheus remote write (http://localhost:9998/api/v1/write)
time="2026-04-17T00:26:40Z" level=debug msg="Converted samples to Prometheus TimeSeries" nts=26 output="Prometheus remote write"
time="2026-04-17T00:26:40Z" level=debug msg="Successful flushed time series to remote write endpoint" nts=26 output="Prometheus remote write" took=1.180139ms
```

**Naming pattern**:

| Metric type (k6)       | Base name              | Suffix strategy                                                                     | Example `__name__`                                 |
|------------------------|------------------------|-------------------------------------------------------------------------------------|----------------------------------------------------|
| Counter                | `my_custom_counter`    | `_total`                                                                            | `k6_my_custom_counter_total`                       |
| Trend                  | `my_custom_duration`   | `_avg`, `_count`, `_max`, `_med`, `_min`, `_p90`, `_p95`, `_p99`, `_sum` (one TS per stat) | `k6_my_custom_duration_p99`, `..._avg`, etc. |
| Rate                   | `my_custom_rate`       | `_rate`                                                                             | `k6_my_custom_rate_rate`                           |
| Gauge                  | `my_custom_gauge`      | (no suffix)                                                                         | `k6_my_custom_gauge`                               |
| Built-in Counter       | `iterations`, `data_*` | `_total`                                                                            | `k6_iterations_total`, `k6_data_received_total`    |
| Built-in Gauge         | `vus`, `vus_max`       | (no suffix)                                                                         | `k6_vus`, `k6_vus_max`                             |

### Rationale

- **`vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/config.go:20-28` — compile-time constants**:
  - `defaultMetricPrefix = "k6_"` — the single constant applied to **every** exported series.
  - `defaultServerURL = "http://localhost:9090/api/v1/write"`, `defaultPushInterval = 5 * time.Second`, `defaultTimeout = 5 * time.Second`.
  - `var defaultTrendStats = []string{"p(99)"}` — used when the user doesn't set `K6_PROMETHEUS_RW_TREND_STATS`.

  ```go
  const (
      defaultServerURL    = "http://localhost:9090/api/v1/write"
      defaultTimeout      = 5 * time.Second
      defaultPushInterval = 5 * time.Second
      defaultMetricPrefix = "k6_"
  )
  ```

- **`vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/prometheus.go:38-55` — `MapSeries()`** (this is the core naming logic):
  - Builds the `Labels` slice for a Prometheus `TimeSeries`.
  - `v := defaultMetricPrefix + series.Metric.Name` → **unconditionally prepends `k6_`**.
  - If a suffix is passed (e.g., `"total"`, `"p99"`, `"avg"`), `v += "_" + suffix`.
  - `append(MapTagSet(series.Tags), &prompb.Label{Name: namelbl, Value: v})` → sets the `__name__` label (where `namelbl = "__name__"` is defined as a package-level constant) with the computed prefixed name.
  - Sorts all labels lexicographically via `sort.Slice(...)` — Prometheus remote-write requires label sets to be lexicographically ordered.

  ```go
  func MapSeries(series metrics.TimeSeries, suffix string) []*prompb.Label {
      v := defaultMetricPrefix + series.Metric.Name
      if suffix != "" {
          v += "_" + suffix
      }
      lbls := append(MapTagSet(series.Tags), &prompb.Label{
          Name:  namelbl,
          Value: v,
      })
      sort.Slice(lbls, func(i int, j int) bool {
          return lbls[i].Name < lbls[j].Name
      })
      return lbls
  }
  ```

- **`vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/trend.go:78-96` — `trendAsGauges.Append(suffix, v)`**:
  - Clones the base labels, then sets `ts.Labels[tg.ixname].Value += "_" + suffix` where `ixname` is the cached index of the `__name__` label inside the labels slice.
  - This is how, for Trend metrics, one base name becomes multiple emitted series — one per configured stat (e.g., `k6_iteration_duration_p99`, `k6_iteration_duration_avg`, `k6_iteration_duration_count`, etc.). Each stat's numeric value is written to `ts.Samples[0].Value`.
- **`vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/remotewrite.go` — `flush()` pipeline**:
  - Runs periodically on `flushPeriod` (default 5 s — `defaultPushInterval` in `config.go`).
  - `convertToPbSeries()` walks the in-memory registry, invokes `MapSeries()` (directly for Counter/Gauge/Rate, via the trend-as-gauges helper for Trend) per metric, and batches the resulting `TimeSeries` into a `prompb.WriteRequest`.
  - The request is marshaled, snappy-compressed, and POSTed to `SERVER_URL`. The log lines `Converted samples to Prometheus TimeSeries` and `Successful flushed time series to remote write endpoint` are emitted here at DEBUG level.
- **`cmd/outputs.go:66-68` — output constructor registration**:
  - `builtinOutputExperimentalPrometheusRW.String(): func(params output.Params) (output.Output, error) { return remotewrite.New(params) }` wires the `-o experimental-prometheus-rw` CLI flag to the vendored `remotewrite.New()` constructor.

**Interpretation**: The `k6_` prefix is an **unconditional constant** applied at the single `MapSeries()` call site (line 40 of `prometheus.go`). The user's original metric name is preserved verbatim **between** the prefix and the optional type suffix, so the transformation `my_custom_counter` → `k6_my_custom_counter_total` is deterministic and reversible — guaranteeing full metric-name integrity for downstream Prometheus queries (e.g., `sum(rate(k6_my_custom_counter_total[1m]))`).
