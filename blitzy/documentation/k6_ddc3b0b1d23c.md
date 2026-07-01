# k6 VU Orchestration — Runtime-Verified Investigation

This document answers five questions about how the Grafana **k6** Go engine orchestrates Virtual Users (VUs). Every behavioral claim is backed by output **captured at runtime** from a locally built binary — not by reading source alone — and every mechanism is grounded to an exact `file:line` in this repository.

## Preamble

- **Build under test:** `k6 v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)`. The repository version constant is `const Version = "0.55.0"` [`lib/consts/consts.go:L12`].
- **How it was built:** from source with `go build`, which is exactly what the `build` target runs [`Makefile:L7-8`]. The binary was written to `/tmp/k6` to keep the repository tree clean.
- **Where logs go:** k6 writes its logs to **`stderr`** using a logrus text formatter — the default log output is `LogOutput: "stderr"` [`cmd/state/state.go:L153`], for which the logger output is set via `c.globalState.Logger.SetOutput(c.globalState.Stderr)` [`cmd/root.go:L220-222`], and the text formatter is applied via `c.globalState.Logger.SetFormatter(&logrus.TextFormatter{...})` [`cmd/root.go:L255-258`]. The shutdown log lines examined in O1/O2 and the REST request-log lines in O3 are emitted at **debug** level, so every run that needs them uses the `-v`/`--verbose` flag. The flag is registered at [`cmd/root.go:L184`] (`flags.BoolVarP(&gs.Flags.Verbose, "verbose", "v", ...)`), and it raises the log level to debug at [`cmd/root.go:L209-210`] (`if c.globalState.Flags.Verbose { c.globalState.Logger.SetLevel(logrus.DebugLevel) }`).
- **Read-only guarantee:** all temporary scenario scripts, the large data file, the example gRPC server, and the remote-write receiver lived **outside** the repository tree, under `/tmp/k6obs`, and were removed afterward. No repository source, configuration, test, example, vendored, or documentation file was modified or deleted. `git status --porcelain` is empty apart from this document, and `examples/grpc_server/go.mod`/`go.sum` were never touched (the example server was built from a `/tmp` copy).

## Methodology

For each objective (O1–O5) a purpose-built temporary scenario was run against the locally built `/tmp/k6` binary, forcing the exact condition the question asks about, and the resulting output was captured **verbatim** (`stderr` logs, end-of-test summary, HTTP responses, and process RSS). Each section below follows the same shape:

**Question restated → Scenario/command run → Verbatim observed evidence → `file:line` grounding → Rationale → explicit named-item answer.**

Deterministic facts (log strings, the `105`/`ExternalAbort` exit code, `15 complete and 5 interrupted iterations`, the final `dropped_iterations` total, the exported `__name__` formats) are stable across runs. Magnitude- and timing-dependent values (the O2 gRPC message count, the O3 mid-run drop snapshots, and the O4 memory figures) are reported as **observed in a representative run** with an explanation of how they arise; they are not universal constants.

---

## O1 — Exact log messages when a ramping executor (≥5 VUs) receives a SIGINT; are active VUs allowed to finish, or terminated?

**Question restated.** What are the exact log messages when a `ramping-vus` executor running at least 5 VUs receives a `SIGINT`? And from that log evidence, are currently-active VUs allowed to **finish their current iteration**, or are they **terminated mid-execution**?

**Scenario run.** A `ramping-vus` scenario with `startVUs: 5`, a single stage `{ duration: '60s', target: 5 }`, `gracefulStop: '5s'`, `gracefulRampDown: '5s'`, and an iteration body of `sleep(2)`. It was run verbose and interrupted with a **real** signal (not simulated) at roughly 7 seconds:

```
/tmp/k6 run -v o1_ramping.js &          # background; capture stderr
kill -INT <pid>                          # a real SIGINT at ~7s
```

### Claim A — the exact shutdown log messages (and the exit code)

Verbatim from `stderr` (verbose):

```
time="2026-07-01T21:54:30Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-01T21:54:30Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

The process exited with **code `105` (`ExternalAbort`)**.

**`file:line` grounding.** The signal trap is installed by `handleTestAbortSignals` [`cmd/common.go:L97`], which allocates a **size-2 buffered** channel `sigC := make(chan os.Signal, 2)` [`cmd/common.go:L99`] and registers `gs.SignalNotify(sigC, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)` [`cmd/common.go:L101`]. The **first** signal invokes the graceful-stop handler in `cmd/run.go`, which logs the debug line `logger.WithField("sig", sig).Debug("Stopping k6 in response to signal...")` [`cmd/run.go:L350`] and then aborts the run with `fmt.Errorf("test run was aborted because k6 received a '%s' signal", sig)` carrying `exitcodes.ExternalAbort` [`cmd/run.go:L354`] — which surfaces as the `level=error` line and exit code `105`. A **second** signal would call `gs.OSExit(int(exitcodes.ExternalAbort))` [`cmd/common.go:L118`], and its handler logs `Error("Aborting k6 in response to signal")` [`cmd/run.go:L360`]; only one signal was sent here, so that line does not appear. The debug line requires `-v` [`cmd/root.go:L184`, `cmd/root.go:L209-210`].

### Claim B (decisive) — active VUs are TERMINATED mid-execution, not allowed to finish

Verbatim end-of-test progress/counter line:

```
running (0m07.0s), 0/5 VUs, 15 complete and 5 interrupted iterations
```

and the summary counter:

```
     iterations...........: 15  2.15695/s
```

Each iteration is `sleep(2s)` — comfortably inside the 5-second graceful window — yet at the moment of the interrupt the **5 in-flight iterations are reported `interrupted`, not completed**. If active VUs were allowed to finish their current iteration, those five 2-second iterations (with ~5s of grace available) would have completed and the line would read `20 complete and 0 interrupted`. It does not. Therefore, **on a manual `SIGINT`, currently-active VUs are terminated mid-execution.**

**`file:line` grounding (authoritative rationale).** This is exactly what the engine documents. The doc comment on `GetGracefulStop` [`lib/executor/base_config.go:L91-96`] explains that the graceful window applies only at the end of the **normal** executor duration, and states verbatim [`lib/executor/base_config.go:L95-96`]:

> "Of course, that doesn't count when the user manually interrupts the test, then iterations are immediately stopped."

So the word "graceful" in `gracefulStop`/`gracefulRampDown` is about *normal stage/duration end*, not about a manual interrupt. On a `SIGINT`, iterations are stopped immediately.

### Claim C — corroborating negative evidence

The normal graceful-end log line was **absent** from the interrupted run. Grepping the captured `stderr` for it returned a count of **0**:

```
$ grep -c 'Regular duration is done, waiting for iterations to gracefully finish' o1_out.log
0
```

**`file:line` grounding.** That message is emitted only when the regular duration ends *and the parent context is still alive*: the guard `if parentCtx.Err() == nil && gracefulStop > 0` [`lib/executor/helpers.go:L193`] gates the log `"Regular duration is done, waiting for iterations to gracefully finish"` [`lib/executor/helpers.go:L196`]. On a manual abort the parent context is cancelled, so the guard is false and that line is skipped; instead the interrupted branch runs — `case <-parentCtx.Done():` [`lib/executor/helpers.go:L208`] sets `progressBar.Modify(pb.WithStatus(pb.Interrupted), constProg)` [`lib/executor/helpers.go:L209`], which is why the progress line ends in `interrupted iterations`. At the VU level the two shutdown paths are `gracefulStop()` [`lib/executor/vu_handle.go:L147`], which logs `Debug("Graceful stop")` [`lib/executor/vu_handle.go:L161`], and `hardStop()` [`lib/executor/vu_handle.go:L165`], which logs `Debug("Hard stop")` [`lib/executor/vu_handle.go:L177`] and **cancels the VU context** via `vh.cancel()` [`lib/executor/vu_handle.go:L178`] — the mechanism by which an in-flight iteration is torn down. The ramping executor's options are `StartVUs` [`lib/executor/ramping_vus.go:L42`] and `GracefulRampDown` [`lib/executor/ramping_vus.go:L44`] (default 30s [`lib/executor/ramping_vus.go:L52`]).

### Explicit answer (O1)

- **Exact log messages:** `level=debug msg="Stopping k6 in response to signal..." sig=interrupt` followed by `level=error msg="test run was aborted because k6 received a 'interrupt' signal"`; the process exits with code **`105` (`ExternalAbort`)**.
- **Finish vs. terminate:** on a manual `SIGINT`, currently-active VUs are **terminated mid-execution — they are NOT allowed to finish their current iteration**. The decisive log evidence is `15 complete and 5 interrupted iterations` despite a 2-second iteration under a 5-second graceful window, grounded in `lib/executor/base_config.go:L95-96` ("…then iterations are immediately stopped").

---

## O2 — Exact log entries when a gRPC server-streaming test with a 30ms graceful ramp-down is interrupted; value of `grpc_streams_msgs_received`

**Question restated.** What are the exact runtime log entries when a **gRPC server-streaming** test configured with a **30ms graceful ramp-down** is interrupted? And what is the value of the "number of grpc messages received" (the `grpc_streams_msgs_received` metric) reported in the final metrics summary?

**Scenario run.** The bundled example gRPC server was built from a `/tmp` copy of `examples/grpc_server/` (so the repository stayed pristine) and started listening on `localhost:10000`. The k6 script used a `ramping-vus` scenario with `startVUs: 5`, stage `{ duration: '60s', target: 5 }`, `gracefulRampDown: '30ms'`, `gracefulStop: '30ms'`, opening a server-streaming RPC with `new Stream(client, 'main.FeatureExplorer/ListFeatures', null)` (the `route_guide.proto` copied **beside** the temp script and loaded by relative name), with `sleep(30)` per iteration. It was run verbose and interrupted at ~4.5s:

```
/tmp/k6obs/grpc_server_bin -port 10000 &      # example gRPC server (from /tmp copy)
/tmp/k6 run -v o2_grpc_stream.js &            # background; capture stderr
kill -INT <pid>                                # a real SIGINT at ~4.5s
```

### Claim A — the interrupt shutdown lines (same handler as O1) and exit code

Verbatim from `stderr` (verbose):

```
time="2026-07-01T21:56:09Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-01T21:56:09Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

The process exited with **code `105` (`ExternalAbort`)**.

**`file:line` grounding.** These are produced by the same signal path as O1: the debug line at [`cmd/run.go:L350`], the error/abort with `exitcodes.ExternalAbort` at [`cmd/run.go:L354`], via the trap in `handleTestAbortSignals` [`cmd/common.go:L97`], `sigC` buffered size 2 [`cmd/common.go:L99`]. The `gracefulRampDown: '30ms'` option maps to `GracefulRampDown` on the ramping executor [`lib/executor/ramping_vus.go:L44`].

### Claim B — `grpc_streams_msgs_received` from the final summary (representative run)

Verbatim from the final end-of-test summary:

```
     grpc_streams.................: 5      1.117335/s
     grpc_streams_msgs_received...: 220    49.16273/s
     grpc_streams_msgs_sent.......: 5      1.117335/s
```

with the accompanying counter line:

```
running (0m04.5s), 0/5 VUs, 0 complete and 5 interrupted iterations
```

**The observed value of `grpc_streams_msgs_received` is `220`.**

**How the value arises (timing-dependent — reported per the magnitude-realism rule).** The example server streams one in-range feature every **100 ms**: `ListFeatures` [`lib/testutils/grpcservice/service.go:L57`] executes `time.Sleep(100 * time.Millisecond)` before each `stream.Send` [`lib/testutils/grpcservice/service.go:L62`]. Therefore, per stream, received ≈ `streaming_elapsed / 100ms`, and across 5 concurrent VU streams the total ≈ `(streaming_elapsed / 100ms) × 5`. This run was interrupted ~4.4s into streaming → ≈44 messages × 5 streams = **220**. The value is therefore configuration- and timing-dependent; it is reported here as observed in this representative run, not as a universal constant (an earlier representative run yielded 195 = 39 × 5).

**`file:line` grounding.** The `grpc_streams_msgs_received` counter is registered as a `metrics.Counter` at [`js/modules/k6/grpc/metrics.go:L25`] (its companions `grpc_streams` [`js/modules/k6/grpc/metrics.go:L17`] and `grpc_streams_msgs_sent` [`js/modules/k6/grpc/metrics.go:L21`]). It is incremented by **1 per received message** in `queueMessage` [`js/modules/k6/grpc/stream.go:L149`], which pushes a sample with `Metric: s.instanceMetrics.StreamsMessagesReceived` [`js/modules/k6/grpc/stream.go:L153`] and `Value: 1` [`js/modules/k6/grpc/stream.go:L158`]; the sent counter is pushed at [`js/modules/k6/grpc/stream.go:L257`]. The server registers `FeatureExplorer` and listens on `localhost:10000`: `net.Listen("tcp", fmt.Sprintf("localhost:%d", *port))` [`examples/grpc_server/main.go:L59`] with `port` defaulting to `10000` [`examples/grpc_server/main.go:L51`] and `RegisterFeatureExplorerServer` [`examples/grpc_server/main.go:L80`]. The RPC is defined as `service FeatureExplorer` [`lib/testutils/grpcservice/route_guide.proto:L23`] / `rpc ListFeatures(Rectangle) returns (stream Feature)` [`lib/testutils/grpcservice/route_guide.proto:L38`]; the client stream pattern is `new Stream(client, 'main.FeatureExplorer/ListFeatures', null)` [`examples/grpc_server_streaming.js:L19`] with the proto path default at [`examples/grpc_server_streaming.js:L10`].

> Read-only note: `examples/grpc_server/` is a separate Go module (`go 1.19` [`examples/grpc_server/go.mod:L3`], `replace go.k6.io/k6 => ../../` [`examples/grpc_server/go.mod:L5`], `google.golang.org/grpc v1.64.1` [`examples/grpc_server/go.mod:L9`]). It was built from a `/tmp` copy with a rewritten `replace` path, so its `go.mod`/`go.sum` were never modified in place.

### Explicit answer (O2)

- **Exact log entries:** `level=debug msg="Stopping k6 in response to signal..." sig=interrupt` followed by `level=error msg="test run was aborted because k6 received a 'interrupt' signal"`, with exit code **`105` (`ExternalAbort`)** — identical to O1's shutdown path.
- **`grpc_streams_msgs_received`:** the final summary reported **`220`** in this representative run (≈44 messages × 5 active streams), a timing-dependent value driven by the server's 100 ms per-message send interval.

---

## O3 — `dropped_iterations` when a test exceeds its capacity, reported by querying the REST API

**Question restated.** What is the exact value of `dropped_iterations` when a test exceeds its maximum-duration/VU capacity — and report that value **by querying the REST API**, with runtime evidence proving the value came from an API query (not just the terminal summary)?

**Scenario run.** A `constant-arrival-rate` scenario with `rate: 100`, `timeUnit: '1s'`, `duration: '10s'`, `preAllocatedVUs: 2`, `maxVUs: 2`, and an iteration body of `sleep(1)`. Capacity is ≈2 iterations/second against a demand of 100/second, so the vast majority of scheduled iterations are dropped. It was run verbose, and the REST API (enabled by default on `localhost:6565`) was polled concurrently with `curl` while the run was in progress:

```
/tmp/k6 run -v o3_arrival.js &                                   # background
curl -s http://localhost:6565/v1/metrics/dropped_iterations       # polled at ~3s, ~5s, ~7s
```

### Claim A — the value obtained BY QUERYING THE API

The command and the verbatim JSON:API response body (captured ~3s into the run, HTTP status **`200`**):

```
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":284,"rate":95.13492963789282}}}}
```

The queried value is read from `data.attributes.sample.count` → **`284`** at ~3s. Subsequent polls returned `count=490` (~5s) and `count=686` (~7s); the counter accumulates monotonically, so these mid-run values are time-dependent snapshots (representative).

**`file:line` grounding.** The REST API default address is `Address: "localhost:6565"` [`cmd/state/state.go:L150`]. The routes are registered in `api/v1/routes.go`: `/v1/metrics` → `handleGetMetrics` [`api/v1/routes.go:L23`], and `/v1/metrics/` → parses the id with `id := r.URL.Path[len("/v1/metrics/"):]` [`api/v1/routes.go:L37`] and dispatches to `handleGetMetric(cs, rw, r, id)` [`api/v1/routes.go:L38`]. The handlers `handleGetMetrics` [`api/v1/metric_routes.go:L9`] and `handleGetMetric` [`api/v1/metric_routes.go:L27`] marshal the JSON:API envelope shown above.

### Claim B (proof the value came from the API) — k6's own server-side request log

Verbatim from `stderr` (verbose) — k6's REST API request-logging middleware logged each `curl` round-trip:

```
time="2026-07-01T21:56:46Z" level=debug msg="GET /v1/metrics/dropped_iterations" status=200
```

This server-side `status=200` line (one per poll) is the runtime evidence that the value was obtained via an **API round-trip**, not merely read from the terminal summary. For context, startup also logged that the server was listening:

```
time="2026-07-01T21:56:43Z" level=debug msg="Starting the REST API server on localhost:6565"
```

### Claim C — root-cause warning for the drops

Verbatim from `stderr`:

```
time="2026-07-01T21:56:43Z" level=warning msg="Insufficient VUs, reached 2 active VUs and cannot initialize more" executor=constant-arrival-rate scenario=overload
```

**`file:line` grounding.** The metric name is `DroppedIterationsName = "dropped_iterations"` [`metrics/builtin.go:L10`], registered as a `Counter` [`metrics/builtin.go:L84`]. In the executor, `droppedIterationMetric := car.executionState.Test.BuiltinMetrics.DroppedIterations` [`lib/executor/constant_arrival_rate.go:L324`]; when the scheduler ticks (`case <-timer.C:` [`lib/executor/constant_arrival_rate.go:L331`]) and `TryRunIteration()` finds no free VU, a `dropped_iterations` sample with `Value: 1` [`lib/executor/constant_arrival_rate.go:L345`] is pushed, and once unplanned VUs are exhausted it warns `Warningf("Insufficient VUs, reached %d active VUs and cannot initialize more", maxVUs)` [`lib/executor/constant_arrival_rate.go:L352`].

### Claim D — the final end-of-test summary value (deterministic for this config)

Verbatim from the final summary:

```
     dropped_iterations...: 981 97.114972/s
```

For this exact configuration (`rate: 100`, `duration: '10s'`, `maxVUs: 2`, `sleep(1)`) the final total is a stable **`981`** — the run schedules ~1000 iterations over 10s and completes only ~19 of them, dropping the remaining `981`. The mid-run API polls (284 / 490 / 686) are representative snapshots along the way toward that total.

**Related-behavior note (so this is not presented as unique to `constant-arrival-rate`).** The `dropped_iterations` counter is also emitted by the other capacity-bounded executors: `ramping-arrival-rate` [`lib/executor/ramping_arrival_rate.go:L472`], `shared-iterations` [`lib/executor/shared_iterations.go:L222`], and `per-vu-iterations` [`lib/executor/per_vu_iterations.go:L202`].

### Explicit answer (O3)

Queried via `GET http://localhost:6565/v1/metrics/dropped_iterations`, the **`dropped_iterations`** value is read from `attributes.sample.count` (e.g. **`284`** at ~3 s mid-run). The proof it came from the API — not the terminal summary — is k6's own server-side log line `level=debug msg="GET /v1/metrics/dropped_iterations" status=200`. The final end-of-test total for this configuration is a deterministic **`981`**.

---

## O4 — Data-sharing memory footprint: constant with `SharedArray` vs. a per-VU copy, and the root cause

**Question restated.** Does the memory footprint for file-backed data **stay constant as the number of VUs increases**, or does **each VU create its own copy** of the data? Prove the behavior with test-script output across multiple VU counts, and identify the **root cause**.

**Scenario run.** A large JSON file was generated under `/tmp`: `big.json` = **37,383,417 bytes (~35.7 MB), 100,000 records**. Two scripts were compared at **1 / 50 / 100 VUs** (`k6 run --vus N --duration 6s`), with peak process RSS sampled every 0.1 s from `/proc/<pid>/status` `VmRSS` (`/usr/bin/time` is unavailable in this environment); iterations `sleep` to keep VUs alive during sampling:

- **SharedArray:** `const data = new SharedArray('records', () => JSON.parse(open('./big.json')));`
- **Per-VU copy:** top-level `const data = JSON.parse(open('./big.json'));` (executes once per VU init).

### Claim A — `SharedArray` footprint is ≈constant across VU counts; the per-VU copy grows ≈linearly

Verbatim peak-RSS measurements (representative magnitudes), sampled from `/proc/<pid>/status` `VmRSS` and quoted exactly as observed:

```text
VUs    SharedArray peak RSS        Per-VU open()/JSON.parse peak RSS
----   -------------------------   ---------------------------------
  1    413376 kB   (~404 MB)       351852 kB    (~344 MB)
 50    433536 kB   (~423 MB)       9744596 kB   (~9.3 GB)
100    415032 kB   (~405 MB)       18522776 kB  (~17.7 GB)
```

With `SharedArray`, the footprint stays **≈constant** as VUs increase (~404 → 423 → 405 MB). With a plain per-VU `open()` + `JSON.parse`, it grows **≈linearly** at roughly ~180 MB per VU — i.e. **each VU creates its own copy** of the data. At 100 VUs that is ~405 MB versus ~17.7 GB, a ≈**44.6×** difference. These magnitudes are **representative** (they scale with the ~35.7 MB file, per-object Sobek overhead, and peak-RSS/GC timing); the **constant-vs-linear behavior** and its **root cause** are what is invariant.

### Claim B — root cause (verified in source)

The behavior has a precise, three-part root cause in the `k6/data` module:

1. **One shared root map.** The RootModule's `New()` [`js/modules/k6/data/data.go:L43`] builds a **single** `sharedArrays` map — `data: make(map[string]sharedArray)` [`js/modules/k6/data/data.go:L46`].
2. **Every VU references that map by pointer.** `NewModuleInstance` [`js/modules/k6/data/data.go:L53`] returns `&Data{ vu: vu, shared: &rm.shared }` [`js/modules/k6/data/data.go:L56`], so all VU instances point at the same backing store rather than owning a copy.
3. **The loader runs exactly once; VUs then read through a proxy that copies only the requested element.** In `(*sharedArrays).get` [`js/modules/k6/data/data.go:L152`] a double-checked lock (`RLock` [`js/modules/k6/data/data.go:L153`] → `Lock` [`js/modules/k6/data/data.go:L157`] → re-check → `getShareArrayFromCall(rt, call)` [`js/modules/k6/data/data.go:L161`] → store `s.data[name] = array` [`js/modules/k6/data/data.go:L162`]) ensures the constructor runs a single time. VUs then read through a **read-only** `DynamicArray` proxy created by `wrap(...)` [`js/modules/k6/data/share.go:L23`] via `rt.NewDynamicArray(wrappedSharedArray{...})` [`js/modules/k6/data/share.go:L27`]; the proxy is immutable — `Set` [`js/modules/k6/data/share.go:L36`] and `SetLen` [`js/modules/k6/data/share.go:L40`] both `panic(s.rt.NewTypeError("SharedArray is immutable"))` — and `Get(index)` [`js/modules/k6/data/share.go:L44`] JSON-parses [`js/modules/k6/data/share.go:L48`] and `deepFreeze`s [`js/modules/k6/data/share.go:L53`] then returns [`js/modules/k6/data/share.go:L58`] a copy of **only the requested element**.

Consequently, all VUs share one backing store and each copies only the element it touches. **Without** `SharedArray`, each VU is a **separate JavaScript VM** (the Sobek runtime, `github.com/grafana/sobek` [`go.mod:L78`]), so the top-level `open()` + `JSON.parse` executes once **per VU** and each VU holds its own full parsed copy — hence the linear growth measured above.

### Explicit answer (O4)

With `SharedArray` the memory footprint stays **≈constant** as VUs increase (~404–423 MB at 1/50/100 VUs); without it, **each VU creates its own full copy**, so memory grows **≈linearly** (~344 MB → ~9.3 GB → ~17.7 GB). **Root cause:** one pointer-shared root `sharedArrays` map [`js/modules/k6/data/data.go:L46,L56`] + a one-time double-checked-lock loader [`js/modules/k6/data/data.go:L152-162`] + a read-only `DynamicArray` proxy that copies only the requested element [`js/modules/k6/data/share.go:L44-58`]; absent `SharedArray`, each VU's separate Sobek JS VM re-parses and holds its own copy.

---

## O5 — Prometheus (remote-write) output preserves metric-name integrity

**Question restated.** When using the k6 **Prometheus (remote-write) output**, provide test-script output proving the exported data **maintains the integrity of the metric names** (how names are prefixed/suffixed versus mangled or truncated).

**Scenario run.** A temporary remote-write receiver was stood up at `http://localhost:9090/api/v1/write` (a small program under `/tmp` that snappy-decompresses and protobuf-decodes the `WriteRequest` and reads the `__name__` labels). k6 was run with `--out experimental-prometheus-rw` on a workload defining a custom `Counter('my_custom_counter')`, `Trend('waiting_time')`, `Gauge('my_gauge')`, and `Rate('my_rate')`, alongside built-in metrics:

```
python3 rw_receiver.py &                                # receiver on localhost:9090
/tmp/k6 run --out experimental-prometheus-rw o5_prom.js
```

### Claim A — output wiring confirmed

Verbatim from stdout (startup banner):

```
output: Prometheus remote write (http://localhost:9090/api/v1/write)
```

**`file:line` grounding.** `cmd/outputs.go` imports the remote-write package [`cmd/outputs.go:L20`] and wires the built-in output via `return remotewrite.New(params)` [`cmd/outputs.go:L67`]. The receiver URL matches the package default `defaultServerURL = "http://localhost:9090/api/v1/write"` [`.../remotewrite/config.go:L21`].

### Claim B — exported `__name__` labels preserve name integrity

The receiver captured these `__name__` labels (verbatim, sorted), with annotations showing the applied prefix/suffix:

```
k6_data_received_total        (built-in Counter → _total)
k6_data_sent_total            (built-in Counter → _total)
k6_iteration_duration_p99     (built-in Trend, stat p(99) → _p99)
k6_iterations_total           (built-in Counter → _total)
k6_my_custom_counter_total    (custom Counter 'my_custom_counter' → _total)
k6_my_gauge                   (custom Gauge 'my_gauge' → NO suffix)
k6_my_rate_rate               (custom Rate 'my_rate' → _rate)
k6_vus                        (built-in Gauge → NO suffix)
k6_vus_max                    (built-in Gauge → NO suffix)
k6_waiting_time_p99           (custom Trend 'waiting_time', stat p(99) → _p99)
```

**Metric-name integrity is maintained.** Each original metric name is preserved **verbatim** as the core of the exported name; only a deterministic `k6_` prefix and a type/stat suffix are added. Names are **not truncated** and **not character-mangled**: the custom names `my_custom_counter`, `waiting_time`, `my_gauge`, and `my_rate` all pass through intact.

**`file:line` grounding.** The label key is `const namelbl = "__name__"` [`.../remotewrite/prometheus.go:L11`]. `MapSeries` [`.../remotewrite/prometheus.go:L39`] builds the value as `v := defaultMetricPrefix + series.Metric.Name` [`.../remotewrite/prometheus.go:L40`], then appends the suffix only if present — `if suffix != "" { v += "_" + suffix }` [`.../remotewrite/prometheus.go:L41-42`] — and sets it as the `__name__` label [`.../remotewrite/prometheus.go:L44-47`]. The prefix is `defaultMetricPrefix = "k6_"` [`.../remotewrite/config.go:L24`], and the default trend stats are `defaultTrendStats = []string{"p(99)"}` [`.../remotewrite/config.go:L28`] (hence `_p99`). The per-type suffix is assigned in `MapPrompb` [`.../remotewrite/remotewrite.go:L316`] through the `mapMonoSeries` closure [`.../remotewrite/remotewrite.go:L319`] / `MapSeries(s, suffix)` [`.../remotewrite/remotewrite.go:L321`]: Counter → `"total"` [`.../remotewrite/remotewrite.go:L331`], Gauge → `""` (no suffix) [`.../remotewrite/remotewrite.go:L336`], Rate → `"rate"` [`.../remotewrite/remotewrite.go:L341`], and Trend → `trend.MapPrompb(...)` [`.../remotewrite/remotewrite.go:L356`] producing the stat suffix (e.g. `p(99)` → `p99`). For the Trend case that call resolves to `(*extendedTrendSink).MapPrompb` [`.../remotewrite/trend.go:L36`], which iterates the configured stats and, per stat, appends the suffix to the `__name__` label via `ts.Labels[tg.ixname].Value += "_" + suffix` [`.../remotewrite/trend.go:L89`] inside `(*trendAsGauges).Append` [`.../remotewrite/trend.go:L78`].

> The full vendored paths are `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/{prometheus.go,config.go,remotewrite.go,trend.go}`.

### Explicit answer (O5)

The exported time series **maintain metric-name integrity**: each `__name__` is `k6_` + the **verbatim** original metric name + a deterministic type/stat suffix (Counter → `_total`, Gauge → none, Rate → `_rate`, Trend → `_p99`). No truncation or character-mangling occurs, and custom names (`my_custom_counter`, `waiting_time`, `my_gauge`, `my_rate`) pass through intact.

---

## Coverage Confirmation

Every distinct sub-question and every named item is answered explicitly and by name:

- **O1(a) — exact SIGINT log messages:** provided verbatim (two lines) — `level=debug msg="Stopping k6 in response to signal..." sig=interrupt` and `level=error msg="test run was aborted because k6 received a 'interrupt' signal"` — with exit code **`105` (`ExternalAbort`)**.
- **O1(b) — finish vs. terminate:** answered — active VUs are **terminated mid-execution** (not allowed to finish), proven by `15 complete and 5 interrupted iterations` and grounded in `lib/executor/base_config.go:L95-96`.
- **O2(a) — exact gRPC-streaming interrupt log entries:** provided verbatim (same two shutdown lines) with exit code **`105`**.
- **O2(b) — `grpc_streams_msgs_received`:** provided from the final summary — **`220`** (representative), with the ×5-streams / 100 ms-interval derivation.
- **O3 — `dropped_iterations` via the REST API:** value read from the `curl` JSON:API body `attributes.sample.count` (e.g. **`284`** mid-run); API-origin proof is k6's own `level=debug msg="GET /v1/metrics/dropped_iterations" status=200` log line; final end-of-test total **`981`**.
- **O4 — constant vs. per-VU copy:** answered — **constant** with `SharedArray`, **per-VU copy** otherwise (linear growth) — with the peak-RSS table and the **root cause** named and grounded (`js/modules/k6/data/data.go:L46,L56,L152-162`; `js/modules/k6/data/share.go:L44-58`).
- **O5 — Prometheus name integrity:** exported `__name__` labels shown verbatim; prefix (`k6_`) + type/stat suffix behavior explained; **no mangling or truncation**; custom names intact.
- **Attribution:** all findings are from `k6 v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)`.
- **Read-only confirmation:** no repository source file was modified or deleted; all temporary scripts, binaries, and data files lived under `/tmp/k6obs` and were removed; `git status --porcelain` is empty apart from this document (`blitzy/documentation/k6_ddc3b0b1d23c.md`), and `examples/grpc_server/go.mod`/`go.sum` were never touched.

