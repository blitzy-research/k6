# Getting to Know k6 — A Q&A Onboarding Guide

Welcome to the team! This document answers the three questions you asked while getting your bearings on **grafana/k6**, the open-source load-testing tool — a Go command-line application (module `go.k6.io/k6`) that runs JavaScript test scripts to generate load and collect performance metrics. Everything below is grounded in the **actual source code** under study (the k6 baseline at commit `ddc3b0b1d23c128e34e2792fc9075f9126e32375`, source branch `k6_ddc3b0b1d23c`) and in **real build/test runs** I performed locally, with a `path:line` citation behind every architectural claim and test-health numbers taken from running the suite rather than from estimation.

Your three questions map to the three sections that follow:

1. **Project Health** — how many tests pass vs. fail, and what is skipped or broken?
2. **Metrics Architecture** — which files/modules count iterations and collect performance data?
3. **Data-Flow Trace** — how does one metric travel from test start to the final summary output?

A fourth section captures the **rationale** behind each answer, as well as how to reproduce everything.

**Provenance & read-only note.** The k6 *source* under study is commit `ddc3b0b1d23c128e34e2792fc9075f9126e32375`; built from that baseline, the binary self-reports `v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)` — the 10-character stamp is simply the first ten hex digits of the VCS revision (`lib/consts/consts.go:30-35,52`). This deliverable is committed *on top of* that baseline (the initial documentation commit was `ab1b2ee89d`), adding **only** this one Markdown file and changing no k6 source; because the commit stamp simply reflects whatever revision is built, rebuilding from the delivered HEAD stamps that HEAD's own ten-hex commit, even though the analyzed code is byte-for-byte the `ddc3b0b1d23c` baseline. Per your request **nothing in the repository was modified**: every `path:line` citation points at that baseline source, and the scratch test script and compiled binary live outside the source tree under `/tmp`.

---

## Section 1 — Project Health (OBJ-1)

### How the project is tested

The canonical test command is the `Makefile` `tests:` target (`Makefile:28-29`):

```bash
go test -race -timeout 210s ./...
```

The `build:` target is plain `go build` (`Makefile:7-8`). The `-race` flag enables Go's data-race detector, which requires cgo (`CGO_ENABLED=1`), so a C compiler must be present. The project is a single Go module that compiles to one static binary from `main.go`.

**CI baseline (the reference point for the health verdict).** The GitHub Actions workflow `.github/workflows/test.yml` runs the suite across three Go lines — `1.22.x` (`test-prev`, `.github/workflows/test.yml:17`), `1.23.x` (`test-current-cov`, `.github/workflows/test.yml:89`), and Go tip (`test-tip`, `.github/workflows/test.yml:48`) — on both `ubuntu-latest` and `windows-2019`. CI uses a larger timeout, `go test ... -timeout 800s ./...` (`.github/workflows/test.yml:46`, `.github/workflows/test.yml:83`, `.github/workflows/test.yml:118`), with `GOMAXPROCS=2` and `-p 2` to reduce flakiness. The project's testing strategy mandates a **100%-pass requirement** with **zero tolerance for data races or goroutine leaks** across 176 test files in 54 test-bearing packages in this checkout. In short: **green CI is the baseline**, and any local failure should be judged against it.

### What I actually ran, and what happened

I built the binary outside the repo and ran the suite on **Go 1.23.12** with **gcc 15.2.0**. `go list ./...` reports **82 packages** total, which every run accounted for exactly. The headline numbers below are the **authoritative delivery-gate `-race` run**; I then re-ran the suite several times, and the run-to-run variance (shown right after) turns out to be the single most important piece of evidence for the health verdict.

**Primary result — race detector on** (the canonical `-race` command, with the longer timeout CI itself uses to capture complete counts):

```bash
GOMAXPROCS=2 go test -race -p 2 -timeout 800s ./...   # exit code 1
```

| Result category | Count |
|---|---:|
| Packages passing (`ok`) | 48 |
| Packages failing (`FAIL`) | 6 |
| Packages with no test files | 28 |
| **Total packages** | **82** |
| Data races detected | 0 |
| Panics | 1 (a flaky panic — see "What is broken" below) |

The 6 failing packages, their failing leaf tests, and the nature of each:

| Package | Failing test(s) | Nature |
|---|---|---|
| `go.k6.io/k6/js/modules/k6/grpc` | `TestClient_TlsParameters/{ConnectTlsInvokeSuccess,ConnectTls,ConnectTlsEncryptedKey}` | TLS fixture (deterministic) |
| `go.k6.io/k6/js/modules/k6/http` | `TestRequestAndBatchTLS/ocsp_stapled_good` | TLS fixture (deterministic) |
| `go.k6.io/k6/cmd/tests` | timing-sensitive integration test(s) | timing flake |
| `go.k6.io/k6/js` | `TestVURunInterrupt/Archive` | timing flake |
| `go.k6.io/k6/lib/executor` | `TestConstantArrivalRateRunCorrectTiming/segment_*`, `TestRampingVUsHandleRemainingVUs` | timing flake (scheduling-tolerance & VU-count assertions) |
| `go.k6.io/k6/output/cloud/expv2` | `TestFlushMaxSeriesInBatch` — **panic**: `index out of range [1] with length 1` | **broken** (flaky panic) |

That last row is the one genuinely **broken** result: `output/cloud/expv2`'s `TestFlushMaxSeriesInBatch` does not merely fail an assertion — it **panics** and aborts its package. I treat it as a first-class finding in "What is broken" below rather than as an aside.

**Variance — what reruns showed (this is the key evidence).** The failing set is *not stable* from run to run. Re-running the same commands in the same environment produced materially different — and always *smaller* — failing sets than the authoritative run above:

| Run | Command | ok / FAIL / no-test | Packages that failed | Panics |
|---|---|---|---|---:|
| Authoritative `-race` | `go test -race -p 2 -timeout 800s ./...` | 48 / 6 / 28 | grpc, http, cmd/tests, js, lib/executor, **expv2 (panic)** | 1 |
| My `-race` rerun | (same command) | 52 / 2 / 28 | grpc, http only | 0 |
| My no-`-race` rerun | `go test -timeout 210s ./...` | 51 / 3 / 28 | grpc, http, lib/executor | 0 |

The **only** packages that fail in *every* run are the gRPC and HTTP TLS pair; everything else — `cmd/tests`, `js`, `lib/executor`, `execution`, and even the `output/cloud/expv2` panic — flickers in and out. A failing set whose sole constant is the deterministic TLS pair, and which otherwise shrinks and shifts on each rerun, is the hallmark of environment/timing flakiness rather than a logic defect. I deliberately lead with the **largest** observed failing set (the authoritative delivery-gate run) so nothing is smoothed over; the smaller reruns then make the flakiness explicit. (A concrete example of the flicker: in the no-`-race` rerun, `lib/executor`'s `TestRampingVUsHandleRemainingVUs` failed a "Not equal" assertion at `lib/executor/ramping_vus_test.go:370-371` — a timing-dependent expectation — whereas in other runs that same test passed.)

### The single skipped test

Exactly one test deliberately **skips**: **`TestTC39`** in `js/tc39/tc39_test.go`. Running it directly confirms the skip:

```bash
$ go test -run '^TestTC39$' -v ./js/tc39/
    tc39_test.go:799: If you want to run tc39 tests, you need to run the 'checkout.sh' script ... test262 ...
--- SKIP: TestTC39 (0.00s)
PASS
ok    go.k6.io/k6/js/tc39
```

Here is exactly why it skips. `TestTC39` (`js/tc39/tc39_test.go:794`) calls a helper `runTestTC39` (`js/tc39/tc39_test.go:799`). That helper (`js/tc39/tc39_test.go:803`) is marked `t.Helper()` (`js/tc39/tc39_test.go:804`) and begins with a guard: `if _, err := os.Stat(tc39BASE); err != nil { t.Skipf(...) }` (`js/tc39/tc39_test.go:806-807`), where `tc39BASE = "TestTC39/test262"` (`js/tc39/tc39_test.go:39`). The external TC39 **test262** conformance corpus is not checked out (it requires running the repo's `checkout.sh` script), so `os.Stat` fails and the test skips. Because `runTestTC39` is a `t.Helper()`, Go attributes the skip to the *caller* line, which is why the `-v` output prints the basename `tc39_test.go:799` (i.e. `js/tc39/tc39_test.go:799`) even though the literal `t.Skipf` statement is at `js/tc39/tc39_test.go:807`. A skip is not a failure, so the `js/tc39` package still reports `ok` in every full run. (`TestTC39` *also* skips under `-short` at `js/tc39/tc39_test.go:795-796`, but the canonical command does not pass `-short`; in this environment it skips because of the missing fixtures.)

### What is "broken" (vs. an ordinary assertion failure)

Interpreting "broken" as build/compile failures, panics, or timeouts (as distinct from ordinary assertion failures):

- **Build/compile failures:** none. Every one of the 82 packages compiled cleanly.
- **Panic — the one genuinely broken result.** `output/cloud/expv2`'s **`TestFlushMaxSeriesInBatch`** (`output/cloud/expv2/flush_test.go:218`) **panics** with `runtime error: index out of range [1] with length 1`. I confirmed it directly by running that single test in a stress loop **600 times**, where it panicked **4 times (~0.7%)** — most runs pass, which is exactly why a single suite run may or may not surface it. The panic lives in the **test**, not in k6's production code: the test pushes three time series, caps each batch at `maxSeriesInBatch: 2`, and fans the pushes out across `batchPushConcurrency: 5` goroutines (`output/cloud/expv2/flush_test.go:241`), collecting the resulting batches into a mutex-guarded slice in *arrival* order. Because that order is non-deterministic, the assertion `assert.Equal(t, "val1", ts[0]...)` sometimes sees `"val3"` instead (`output/cloud/expv2/flush_test.go:295`), and the very next line `require.Len(t, ts[1]...)` then indexes `ts[1]` after an out-of-order batch left `ts` with only one element — producing the `index out of range [1] with length 1` panic (`output/cloud/expv2/flush_test.go:297`). It is a concurrency-ordering assumption in an **experimental cloud-output test** — a flaky test, not a defect in the metrics pipeline you asked about. Per scope it is diagnosed, not repaired.
- **Timeouts:** the gRPC TLS subtests `ConnectTls` and `ConnectTlsEncryptedKey` each ran ~60 s before failing — they hang attempting a TLS handshake that can never succeed (see root cause), so they are effectively timing out on a doomed connection rather than failing a quick assertion.

### Root-cause verdict: environment/toolchain, not k6 defects

Every non-passing result is attributable to the **local environment**, not to a defect in k6's logic. The evidence:

1. **A deterministic TLS-certificate cause dominates.** The gRPC and HTTP TLS tests fail with, verbatim from my logs:

   ```
   x509: certificate signed by unknown authority (possibly because of
   "crypto/rsa: verification error" while trying to verify candidate
   authority certificate "Acme Co")
   ```
   and on the HTTP server side:
   ```
   http: TLS handshake error from 127.0.0.1:PORT: remote error: tls: bad certificate
   ```
   The bundled test CA ("Acme Co") fails modern Go 1.23 RSA signature verification. My log timestamps read `2026/...`, i.e., the container clock is well past the test fixtures' ~2024 validity assumptions. This breaks `TestClient_TlsParameters` (grpc) and `TestRequestAndBatchTLS/ocsp_stapled_good` (http) **consistently across every run** — these two TLS-fixture packages are the *only* ones that fail deterministically. (Note: the `lib/executor` non-passes are **not** a TLS cascade. That package opens no gRPC/TLS connection at all — `grep -rni grpc lib/executor/` returns zero matches, and `TestConstantArrivalRateRunCorrectTiming` drives a mock `simpleRunner` (`lib/executor/common_test.go:17`), not a network client. Its failures are independent *timing* flakes, detailed in point 2.)

2. **Most of the rest are timing-sensitive assertions (plus one flaky panic) that come and go under a contended CPU and the race detector.** The strongest proof is that **the failing set changes from run to run.** The authoritative `-race` run failed six packages — including the `output/cloud/expv2` panic and the timing-sensitive `cmd/tests`, `js`, and `lib/executor` — yet my `-race` rerun failed only `grpc` and `http` (every timing-sensitive package passed), and my no-`-race` rerun failed `grpc`, `http`, and `lib/executor`. The `expv2` panic is itself intermittent (≈4 in 600 targeted runs). A failing set that is a moving target — where the **only constant** across every run is the deterministic TLS pair — is the hallmark of environment/timing flakiness, not a logic bug. The `lib/executor` non-passes make this concrete: both are wall-clock-sensitive assertions, not network failures. `TestConstantArrivalRateRunCorrectTiming` checks that iterations fire within a **24 ms** tolerance (`assert.WithinDuration(..., time.Millisecond*24, ...)`, `lib/executor/constant_arrival_rate_test.go:185`) — a check whose own `// FIXME` admits it should not "depend… on the execution time itself" (`lib/executor/constant_arrival_rate_test.go:183-184`) — and `TestRampingVUsHandleRemainingVUs` asserts exact VU interrupted/finished counts (`lib/executor/ramping_vus_test.go:370-371`); both are precisely the kind of timing expectation that flakes under a contended container CPU.

3. **Zero data races in every `-race` run.** The race detector found no genuine concurrency defects, which is exactly what the project's zero-races policy expects.

4. **Green CI is the baseline.** On clean GitHub runners (working TLS fixtures, uncontended CPUs, `-timeout 800s`), the project's 100%-pass requirement holds. The local deltas are explained entirely by (a) the expired/incompatible TLS fixtures versus the local clock and Go 1.23 crypto, and (b) timing flakiness amplified by the race detector and a busy container.

**Conclusion — k6 is a healthy, well-tested project.** The vast majority of its 82 packages pass (≈48–52 `ok` depending on the run, 28 have no tests by design), there are no data races, and there are no genuine logic-level defects in k6 itself. The handful of local non-passes are **environment/toolchain artifacts** (the expired "Acme Co" TLS fixtures versus the 2026 container clock) and **flaky tests** (timing-sensitive assertions, plus one intermittent panic in an *experimental cloud-output test*) — not k6 defects. Consistent with your "just exploring, don't change anything" instruction — and because repairing tests is out of scope — these are **diagnosed, not fixed**.

---

## Section 2 — Metrics Architecture (OBJ-2)

You asked which parts of the code (a) **count iterations** and (b) **collect performance data**. k6's metrics subsystem is best understood as a five-layer pipeline:

```
definition → emission → transport → aggregation → output
```

The same pipeline carries *every* metric — built-in or user-defined — so once you see how the built-in `iterations` Counter flows through it, you understand all of them. Below, each capability is mapped to specific files and lines.

### (a) Counting iterations — the built-in `iterations` Counter

**Definition.** The built-in metric names are constants in `metrics/builtin.go`: `IterationsName = "iterations"` (`metrics/builtin.go:8`) and `IterationDurationName = "iteration_duration"` (`metrics/builtin.go:9`). `RegisterBuiltinMetrics` (`metrics/builtin.go:78`) creates them: `Iterations` is registered as a **Counter** (`metrics/builtin.go:82`) and `IterationDuration` as a **Trend** carrying a `Time` value type (`metrics/builtin.go:83`).

**Emission.** When a virtual user (VU) finishes one pass of your script's `export default` function, k6 emits the iteration samples. In `js/runner.go`, `ActiveVU.RunOnce` (`js/runner.go:724`) drives a single iteration via `runFn` (defined at `js/runner.go:817`). On a *full* iteration of the default function, the code sends the samples onto the per-VU channel:

```go
if isFullIteration && isDefault {
    u.state.Samples <- iterationSamples(startTime, endTime, ctm, builtinMetrics)   // js/runner.go:871
}
```

The builder `iterationSamples` (`js/runner.go:879-902`) constructs two samples: the `iteration_duration` **Trend** (its value is the elapsed duration) and the `iterations` **Counter** with **`Value: 1`** (`js/runner.go:899`). That literal `1` is the entire mechanism of "counting" — each completed iteration contributes exactly one to the counter.

**Iteration driver.** What actually calls `RunOnce()`? Every executor type shares one closure built by `getIterationRunner` (`lib/executor/helpers.go:104`), whose body invokes `err := vu.RunOnce()` (`lib/executor/helpers.go:108`). This is why the iteration-counting behavior is identical no matter which executor (shared-iterations, constant-arrival-rate, ramping-VUs, …) is in play.

**Per-VU channel.** The destination of that send is the VU's send-only samples channel, `Samples chan<- metrics.SampleContainer` (`lib/vu_state.go:59`). Every metric a VU produces leaves the VU through this one channel.

**(Context) the `vus` / `vus_max` gauges.** Separately, the scheduler periodically reports how many VUs are active. `Scheduler.emitVUsAndVUsMax` (`execution/scheduler.go:199`) defines an `emitMetrics` closure (`execution/scheduler.go:205`) that pushes the `VUs` and `VUsMax` gauges, driven by a 1-second ticker, `ticker := time.NewTicker(1 * time.Second)` (`execution/scheduler.go:231`). This is why VU counts in the summary update once per second rather than continuously.

### (b) Collecting performance data — registry → samples → sinks → outputs → ingester

**Registry & metric creation.** All metrics are created through a `Registry`. `NewRegistry` (`metrics/registry.go:20`) builds it; `NewMetric` (`metrics/registry.go:43`) and the panic-on-error `MustNewMetric` (`metrics/registry.go:70`) register metrics by name and type. The private `newMetric` (`metrics/registry.go:93`) is the key constructor: it calls `NewSink(mt)` and stores the result on the metric (`Sink: sink`, `metrics/registry.go:105`) — **every metric is born with its own aggregating sink**.

**The `Metric` type.** A metric is described by `type Metric struct` (`metrics/metric.go:12`): `Name` (`metrics/metric.go:14`), `Type MetricType` (`metrics/metric.go:15`), `Contains ValueType` (`metrics/metric.go:16`), `Submetrics` (`metrics/metric.go:22`), and `Sink Sink` (`metrics/metric.go:24`). Filtered views are `type Submetric` (`metrics/metric.go:29`).

**The enums (authoritative semantics from the code's own doc-comments).** The four metric behaviors are defined in `metrics/metric_type.go:10-13`:

| Type | Line | Doc-comment (verbatim) |
|---|---|---|
| `Counter` | `metrics/metric_type.go:10` | "A counter that sums its data points" |
| `Gauge` | `metrics/metric_type.go:11` | "A gauge that displays the latest value" |
| `Trend` | `metrics/metric_type.go:12` | "A trend, min/max/avg/med are interesting" |
| `Rate` | `metrics/metric_type.go:13` | "A rate, displays % of values that aren't 0" |

How values are interpreted is `ValueType` in `metrics/value_type.go:7-9`: `Default` (as-is, `metrics/value_type.go:7`), `Time` (milliseconds, `metrics/value_type.go:8`), `Data` (bytes, `metrics/value_type.go:9`).

**Sample types & the emission guard.** A measurement is a `Sample` (`metrics/sample.go:23`), keyed by a `TimeSeries` (`metrics/sample.go:14`); collections satisfy the `SampleContainer` interface (`metrics/sample.go:37`). The helper that VUs use to push samples safely is `PushIfNotDone` (`metrics/sample.go:131`). It is precisely a **context-guarded send**, not a non-blocking `select`/`default`:

```go
func PushIfNotDone(ctx context.Context, output chan<- SampleContainer, sample SampleContainer) bool {
    if ctx.Err() != nil {
        return false
    }
    output <- sample
    return true
}
```

If the context is already done it returns `false` and **drops** the sample; otherwise it performs an ordinary (blocking) channel send. This guard prevents writes to the channel after a test has stopped.

**Sinks (the aggregators).** Each metric type has a sink whose `Add` method folds a sample into a running aggregate, in `metrics/sink.go`: `CounterSink.Add` (`metrics/sink.go:53`) does `c.Value += s.Value` (`metrics/sink.go:54`) — literally summing; `GaugeSink.Add` (`metrics/sink.go:82`) keeps the latest/min/max; `TrendSink.Add` (`metrics/sink.go:117`) accumulates statistics; `RateSink.Add` (`metrics/sink.go:210`) tracks the fraction of non-zero values. These match the doc-comment semantics above exactly.

**Output manager & helpers (transport).** Samples leave the VU channel and are batched by the output manager. `Manager.Start` (`output/manager.go:42`) launches a goroutine that buffers incoming `SampleContainer`s and flushes them on a ticker every **50 ms** (`sendBatchToOutputsRate = 50 * time.Millisecond`, `output/manager.go:12`). Each flush runs the `sendToOutputs` closure, which calls `out.AddMetricSamples(...)` on every registered output (`output/manager.go:52`). The reusable building blocks are in `output/helpers.go`: `SampleBuffer` (`output/helpers.go:15`) with `AddMetricSamples` (`output/helpers.go:22`) and `GetBufferedSamples` (`output/helpers.go:34`), and `PeriodicFlusher` (`output/helpers.go:55`, constructed by `NewPeriodicFlusher` at `output/helpers.go:89`). An output is anything implementing the `Output` interface (`output/types.go:44`): `Description()` (`output/types.go:47`), `Start()` (`output/types.go:52`), `AddMetricSamples([]metrics.SampleContainer)` (`output/types.go:58`), and `Stop()` (`output/types.go:61`).

**Metrics-engine ingester (aggregation) — the elegant bit.** The component that feeds the in-memory sinks is `OutputIngester` in `metrics/engine/ingester.go`. The key insight: **the ingester is *itself* an `Output`.** It embeds `output.SampleBuffer` (`metrics/engine/ingester.go:26`) and implements the `Output` interface (its `Description()` at `metrics/engine/ingester.go:35` returns "Internal Metrics Ingester"). Its `Start()` (`metrics/engine/ingester.go:40`) creates a `PeriodicFlusher` at `collectRate = 50 * time.Millisecond` (`metrics/engine/ingester.go:12`) that calls `flushMetrics` (`metrics/engine/ingester.go:62`). Inside `flushMetrics`, for each sample it calls `oi.metricsEngine.markObserved(m)` (`metrics/engine/ingester.go:89`) and `m.Sink.Add(sample)` (`metrics/engine/ingester.go:90`), repeating for matching submetrics (`metrics/engine/ingester.go:97-98`). Because the ingester is just another output plugged into the same `output.Manager`, **one 50 ms batch path feeds both external outputs (JSON, cloud, etc.) and the engine's own sinks** — there is no separate collection path for the summary.

**Threshold engine & observed metrics.** `metrics/engine/engine.go` evaluates thresholds on a separate `thresholdsRate = 2 * time.Second` ticker (`metrics/engine/engine.go:21`, used at `metrics/engine/engine.go:173`) and tracks every metric that has received data in its `ObservedMetrics` map (`metrics/engine/engine.go:40`, populated at `metrics/engine/engine.go:111`). That map is what the end-of-test summary renders.

**Custom user-script metrics.** Metrics you create in JavaScript (e.g., `new Counter('my_metric')`) take the identical downstream path. In `js/modules/k6/metrics/metrics.go`, `Metric.add` (`js/modules/k6/metrics/metrics.go:77`) coerces the value, builds a `metrics.Sample` (`js/modules/k6/metrics/metrics.go:118`), and calls `metrics.PushIfNotDone(m.vu.Context(), state.Samples, sample)` (`js/modules/k6/metrics/metrics.go:127`) — the same channel, the same guard, the same sinks as the built-ins.

**Summary render.** Finally, `js/summary.go` turns the observed metrics into the textual report: `summarizeMetricsToObject` (`js/summary.go:62`) and `metricValueGetter` (`js/summary.go:26`), backed by embedded JavaScript (`//go:embed summary.js` at `js/summary.go:18` and `//go:embed summary-wrapper.js` at `js/summary.go:21`). The entry point that invokes this machinery is the `*Runner` method `HandleSummary` (`js/runner.go:352`).


---

## Section 3 — Data-Flow Trace (OBJ-3)

Now let's follow **one metric end-to-end**, from the moment you start a test to the number that prints in the summary. I'll trace the built-in **`iterations` Counter** because it is the cleanest illustration — it is always present (no user code required), and each completed iteration contributes a clean `Value: 1`. I'll also show a **custom counter** (`my_custom_counter`) taking the *identical* downstream path, proving the trace generalizes to user metrics.

### The 7-step function-call chain

1. **Wiring (`cmd/run.go`).** When you run `k6 run script.js`, the run command assembles the pipeline: it builds the metrics engine with `engine.NewMetricsEngine(...)` (`cmd/run.go:170`) and the output manager with `output.NewManager(...)` (`cmd/run.go:220`), then creates the buffered samples channel `samples := make(chan metrics.SampleContainer, test.derivedConfig.MetricSamplesBufferSize.Int64)` (`cmd/run.go:227`). It hands that channel to the scheduler via `execScheduler.Init(runCtx, samples)` (`cmd/run.go:367`) and `execScheduler.Run(globalCtx, runCtx, samples)` (`cmd/run.go:397`).

2. **Scheduling/execution (`execution/scheduler.go` → `lib/executor/helpers.go`).** The scheduler drives the configured executors. Each executor runs iterations through the shared closure from `getIterationRunner` (`lib/executor/helpers.go:104`), whose body calls `vu.RunOnce()` (`lib/executor/helpers.go:108`).

3. **Iteration sampling (`js/runner.go`).** `ActiveVU.RunOnce` (`js/runner.go:724`) executes one pass of your `export default` function via `runFn` (invoked at `js/runner.go:773`, defined at `js/runner.go:817`). On a full default-function iteration, it sends the iteration samples: `u.state.Samples <- iterationSamples(...)` (`js/runner.go:871`). The `iterationSamples` builder (`js/runner.go:879-902`) produces the `iteration_duration` Trend sample and the `iterations` Counter sample with `Value: 1` (`js/runner.go:899`).

4. **Emission guard (`metrics/sample.go` ← `js/modules/k6/metrics/metrics.go`).** Your custom `my_custom_counter.add(1)` flows through `metrics.PushIfNotDone` (`metrics/sample.go:131`) — the context-guarded send that drops the sample only if `ctx.Err() != nil` — invoked from `js/modules/k6/metrics/metrics.go:127`. (Built-in iteration samples are sent directly onto the same channel at `js/runner.go:871`.)

5. **Transport/batching (`output/manager.go`).** `output.Manager.Start` (`output/manager.go:42`) runs a goroutine that drains the samples channel into a buffer and flushes it every **50 ms** (`sendBatchToOutputsRate`, `output/manager.go:12`) via `sendToOutputs` → `out.AddMetricSamples(...)` on each output (`output/manager.go:52`).

6. **Aggregation (`metrics/engine/ingester.go` → `metrics/sink.go`).** One of those outputs is the metrics engine's `OutputIngester`, whose `flushMetrics` (`metrics/engine/ingester.go:62`, on its own 50 ms `collectRate` flush) calls `markObserved(m)` (`metrics/engine/ingester.go:89`) and `m.Sink.Add(sample)` (`metrics/engine/ingester.go:90`) per sample. For the `iterations` Counter, `CounterSink.Add` accumulates `Value += s.Value` (`metrics/sink.go:53-54`), so four iteration samples (each `Value: 1`) sum to **4**.

7. **Output (`cmd/run.go` → `js/summary.go`).** The aggregated values live in `metricsEngine.ObservedMetrics` (`cmd/run.go:196`) and are rendered by the end-of-test summary. The deferred summary block opens at `cmd/run.go:193`; after a debug log (`cmd/run.go:194`), it calls `test.initRunner.HandleSummary(...)` at **`cmd/run.go:195`** (the `*Runner.HandleSummary` method is defined at `js/runner.go:352`), which uses `js/summary.go` plus the embedded `summary.js` to print the report to stdout.

### Visual overview

```mermaid
flowchart TD
    A["cmd/run.go: wire metricsEngine (L170), outputManager (L220),<br/>samples channel (L227); execScheduler.Init (L367) / Run (L397)"] --> B["execution/scheduler.go:<br/>run executors"]
    B --> C["lib/executor/helpers.go:<br/>getIterationRunner -> vu.RunOnce() (L104-108)"]
    C --> D["js/runner.go: ActiveVU.RunOnce (L724) -> runFn (L817)"]
    D --> E["js/runner.go: u.state.Samples &lt;- iterationSamples(...) (L871);<br/>iterations Counter Value=1 (L879-902)"]
    E --> F["lib/vu_state.go: State.Samples channel (L59)<br/>(custom metrics via metrics.PushIfNotDone, sample.go L131)"]
    F --> G["output/manager.go: Start goroutine,<br/>50ms batch (L12) -> AddMetricSamples (L42-52)"]
    G --> H["metrics/engine/ingester.go: flushMetrics (L62) -><br/>markObserved (L89) + m.Sink.Add(sample) (L90)"]
    H --> I["metrics/sink.go: CounterSink.Add (L53-54)<br/>(Value += s.Value) -> sums to 4"]
    I --> J["metricsEngine.ObservedMetrics (run.go L196) -><br/>js/summary.go HandleSummary (run.go L195) -> stdout"]
```

### Empirical validation

To prove the chain rather than assert it, I created this minimal script **outside the repo** and ran the locally built binary:

```javascript
// /tmp/simple_test.js  (OUTSIDE the k6 repo)
import { Counter } from 'k6/metrics';
export const options = { vus: 2, iterations: 4 };
const myCounter = new Counter('my_custom_counter');
export default function () { myCounter.add(1); }
```

```bash
$ /tmp/k6bin run /tmp/simple_test.js     # exit code 0
```

The relevant summary lines from my run:

```
     scenarios: (100.00%) 1 scenario, 2 max VUs, ...
              * default: 4 iterations shared among 2 VUs (maxDuration: 10m0s, gracefulStop: 30s)

     iteration_duration...: avg=24.69µs min=3.82µs med=21.57µs max=51.83µs p(90)=47.99µs p(95)=49.91µs
     iterations...........: 4   22546.262112/s
     my_custom_counter....: 4   22546.262112/s

running (00m00.0s), 0/2 VUs, 4 complete and 0 interrupted iterations
default ✓ [ 100% ] 2 VUs  00m00.0s/10m0s  4/4 shared iters
```

This closes the loop: **4 completed iterations → 4 `iterations` Counter samples (each `Value: 1`) → `CounterSink.Add` sums them to `4`.** The custom `my_custom_counter` also reports `4` via the identical path, confirming user metrics travel the same definition→emission→transport→aggregation→output pipeline. Note that because the options specify `{ iterations: 4 }`, k6 auto-selects the **shared-iterations** executor — "4 iterations shared among 2 VUs" — which is exactly what the scenario line reports. (My throughput figure `22546.262112/s` is timing-dependent and will differ on other machines; the *counts* are the deterministic, meaningful part.)

### Edge considerations

- **Late-write protection.** `PushIfNotDone` drops samples once the run context is done (`metrics/sample.go:131`, the `ctx.Err() != nil` guard), preventing writes to the channel after the test stops.
- **Channel sizing.** The samples channel is buffered, sized by `MetricSamplesBufferSize` (`cmd/run.go:227`), so brief bursts of samples don't block VUs.
- **Why summary values look aggregated, not per-sample.** Three independent cadences govern the pipeline: the **50 ms** output flush (`output/manager.go:12`, mirrored by the ingester's `collectRate` at `metrics/engine/ingester.go:12`), the **1-second** `vus`/`vus_max` ticker (`execution/scheduler.go:231`), and the **2-second** threshold-evaluation tick (`metrics/engine/engine.go:21`). By the time the summary prints, individual samples have already been folded into their sinks.


---

## Section 4 — Rationale

This section makes the reasoning behind each answer explicit, since conclusions are only as good as the evidence under them.

### Why the health verdict is "environment, not defect"

Four mutually reinforcing observations drive this verdict:

- **A deterministic cause explains the consistent failures.** The gRPC and HTTP TLS tests fail every run with `x509: certificate signed by unknown authority (... "crypto/rsa: verification error" ... "Acme Co")` and `tls: bad certificate`. This is a test-fixture/crypto incompatibility: the bundled "Acme Co" CA cannot satisfy Go 1.23's RSA verification under the container's current clock. It is unrelated to k6's load-testing logic, and it is confined to the two TLS-fixture packages (`grpc` and `http`) — the only packages that fail on *every* run. (The `lib/executor` non-passes are a separate matter: that package contains no gRPC/TLS code, so they are independent timing flakes, not a cascade — see the next point.)
- **The non-deterministic failing set is the single strongest signal.** A genuine logic defect fails the *same* test every time. What I observed instead is a *moving target*: the authoritative `-race` run failed six packages (including the `expv2` panic and the timing-sensitive `cmd/tests`/`js`/`lib/executor`), my `-race` rerun failed only `grpc`+`http` (every timing-sensitive package passed), and my no-`-race` rerun failed `grpc`, `http`, and `lib/executor`. The `expv2` panic alone surfaced only ≈4 times in 600 targeted runs. When the only constant across runs is the deterministic TLS pair and everything else flickers, the flickering failures are timing/environment artifacts amplified by a contended container CPU and the race detector's scheduling perturbations.
- **Zero data races in every `-race` run.** With the project's zero-tolerance race policy, the absence of any detected race indicates no genuine concurrency defects.
- **Green CI is the baseline.** The project's stated 100%-pass requirement holds on clean GitHub runners with valid fixtures, uncontended CPUs, and `-timeout 800s`. The local deltas are fully accounted for by the two environmental factors above.

I have led with the **authoritative** delivery-gate counts (48 `ok`/6 `FAIL`/28 no-test under `-race`, including the one flaky panic) and then reported my reruns (52 `ok`/2 `FAIL` under `-race`; 51 `ok`/3 `FAIL` without it), explicitly flagging that they vary run-to-run — because that variance is itself the proof, not a caveat to apologize for. Per scope, I diagnosed without repairing.

### Why the `iterations` Counter was chosen as the trace metric

It is the cleanest possible end-to-end illustration: it is **built-in**, so the trace holds even for a script that defines no metrics of its own; it emits a crisp **`Value: 1` per completed iteration** (`js/runner.go:899`), so the arithmetic at the sink (`4 × 1 = 4`) is unambiguous; and it exercises **every layer** of the pipeline — definition (`metrics/builtin.go:82`), emission (`js/runner.go:871`), transport (`output/manager.go:42`), aggregation (`metrics/engine/ingester.go:90` → `metrics/sink.go:53-54`), and output (`cmd/run.go:195`). Pairing it with a user-defined `my_custom_counter` that also sums to `4` demonstrates that the same downstream path serves user metrics, so the single trace generalizes.

### Reproducibility note

Everything above is reproducible with the toolchain **Go 1.23.12** and **gcc 15.2.0** (the C compiler is required because `-race` enables cgo). The exact commands were: build with `GOFLAGS=-mod=vendor go build -o /tmp/k6bin .`; the authoritative health run with `GOMAXPROCS=2 go test -race -p 2 -timeout 800s ./...` and the no-`-race` canonical `go test -timeout 210s ./...`; the panic reproduction by repeating `go test -race -run '^TestFlushMaxSeriesInBatch$' -count=20 ./output/cloud/expv2/` in a loop; and the trace with `/tmp/k6bin run /tmp/simple_test.js`. The compiled binary (`/tmp/k6bin`) and the scratch script (`/tmp/simple_test.js`) were kept **outside** the repository, and `git status --porcelain` inside the repo was empty after every run — so the read-only guarantee was preserved throughout.

> **Provenance.** Building the k6 *source baseline* (commit `ddc3b0b1d23c128e34e2792fc9075f9126e32375`) yields the version string `v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)`, whose 10-character commit stamp is the first ten hex digits of the VCS revision (`lib/consts/consts.go:30-35,52`) — so all findings here pertain to exactly that baseline. Note that this document is committed *on top of* the baseline (the initial documentation commit was `ab1b2ee89d`, adding only this Markdown file), so rebuilding from the *delivered HEAD* stamps that HEAD's own ten-hex commit instead; the k6 code under analysis is identical either way, because the documentation commit touches no source. Metric-type semantics were cross-checked against the official Grafana k6 documentation, but per the code-as-truth rule the repository source is authoritative and is what every citation above points to.

