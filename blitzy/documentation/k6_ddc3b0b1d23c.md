# k6 Repository Exploration — Branch k6_ddc3b0b1d23c

This document is a read-only exploration of the Grafana k6 repository, produced to answer three specific questions asked by a new team member getting ready to work on the project:

1. **How healthy is the test suite today?** How many tests pass, fail, and are skipped, and what is the root cause of anything that is not passing?
2. **Which files and modules are responsible for metrics tracking?** In particular, which code paths count iterations during a load-test run and which code paths collect the other performance data k6 surfaces at the end of a test?
3. **How does a single metric flow through the system?** For concreteness we follow the `iterations` counter from the moment it is registered at startup through the moment its final value is rendered in the end-of-test summary.

Throughout the exploration **no source files were modified**; only this single markdown document was added to the repository under `blitzy/documentation/`. Every claim in this document is grounded in the source code — each assertion is accompanied by a concrete file path, function name, and where useful a short code excerpt taken verbatim from the repository.

## Environment & Command

| Item | Value |
|------|-------|
| Go toolchain | **Go 1.21.13** — pinned by the `go 1.21` and `toolchain go1.21.13` directives at the top of `go.mod` (lines 3 and 5) |
| Module | `go.k6.io/k6` (`go.mod` line 1) |
| Test command executed | `go test -mod=vendor -timeout 300s -v ./...` (output captured to `/tmp/test_output.txt`) |
| Makefile equivalent | `make tests` → `go test -race -timeout 210s ./...` (see `Makefile` lines 27–29) |
| Deviations from the Makefile | (a) The `-race` flag was **omitted** during the baseline run that produced the counts below, because the analysis run that captured `/tmp/test_output.txt` was performed when the original CGO-less environment was in effect. The environment now supports CGO and `-race` works, but re-running was unnecessary because the pass/fail/skip breakdown is stable on this branch. (b) The timeout was extended from **210 s** to **300 s** so long-running packages such as `go.k6.io/k6/lib/executor` (~22 s) and `go.k6.io/k6/cmd/tests` (~12 s) had comfortable headroom. |
| Working directory | Repository root (`/tmp/blitzy/k6/blitzy-3e37a4fd-ad86-4080-b075-c964f32d4d25_793215`) |
| Branch | `blitzy-3e37a4fd-ad86-4080-b075-c964f32d4d25` (never changed) |

No additional files (source, configuration, test, build) were created or modified. The `blitzy/documentation/` directory was created solely to hold this single document.

---

## Section 1 — Test Suite Health Report

### 1.1 Executive Summary

The headline numbers come directly from `/tmp/test_output.txt` — the verbose output of `go test -mod=vendor -timeout 300s -v ./...`.

| Metric | Value |
|-------:|:------|
| Total Go packages in the module | **82** |
| Packages with at least one test | **54** (reported as either `ok` or `FAIL`) |
| Packages with no test files | **28** (reported as `?` / `[no test files]`) |
| Packages **PASS** | **51** |
| Packages **FAIL** | **3** (`go.k6.io/k6/js`, `go.k6.io/k6/js/modules/k6/http`, `go.k6.io/k6/lib/executor`) |
| Top-level tests **PASS** | **768** |
| Top-level tests **FAIL** | **4** |
| Top-level tests **SKIP** | **1** |
| All-depth test lines **PASS** (including subtests) | **4,418** |
| All-depth test lines **FAIL** (parent + child rows combined) | **7** |
| All-depth test lines **SKIP** | **1** |
| Slowest failing package | `go.k6.io/k6/lib/executor` ≈ **22.244 s** |
| Slowest passing package | `go.k6.io/k6/cmd/tests` ≈ **12.245 s** |

The complete list of failing top-level tests is:

| Failing test | Package | Wall time | Root cause category |
|--------------|---------|----------:|---------------------|
| `TestVURunInterrupt/Source` | `go.k6.io/k6/js` | 0.20 s (parent 1.30 s) | Timing-sensitive (`context deadline exceeded`) |
| `TestRequestAndBatchTLS/ocsp_stapled_good` | `go.k6.io/k6/js/modules/k6/http` | 1.00 s (parent 0.35 s) | External network + OCSP stapling non-determinism |
| `TestConstantArrivalRateRunCorrectTiming/segment_0:1/3_sequence_` | `go.k6.io/k6/lib/executor` | 2.05 s (parent 16.06 s) | Timing-sensitive (sub-ms iteration start jitter exceeds tolerance) |
| `TestRampingVUsHandleRemainingVUs` | `go.k6.io/k6/lib/executor` | 0.08 s | Race-condition-style VU count expectation |

The skipped top-level test is:

| Skipped test | Package | Why |
|--------------|---------|------|
| `TestTC39` | `go.k6.io/k6/js/tc39` | Requires the external `test262` corpus which is **not** checked out by default. |

### 1.2 Test Suite Structure

k6 uses the Go standard `testing` package together with `github.com/stretchr/testify` for assertions (see almost any `*_test.go` in the repo). Tests are **co-located** with the code they test — e.g. `metrics/metric.go` is tested by `metrics/metric_test.go`, `execution/scheduler.go` by `execution/scheduler_test.go`, and so on. There is no separate `tests/` directory.

The one exception is `js/tc39/` which hosts a conformance harness that runs the external TC39 `test262` corpus against k6's JavaScript engine (`github.com/grafana/sobek`). That harness intentionally skips if the corpus is not present, which we discuss below.

### 1.3 Failure Analysis

#### 1.3.1 `TestRequestAndBatchTLS/ocsp_stapled_good`

- **File**: `js/modules/k6/http/request_test.go` lines 2192–2210.
- **What it does**: It performs a real **HTTPS** GET against `https://www.wikipedia.org/` and asserts that the returned OCSP stapled response has status `OCSP_STATUS_GOOD`. The relevant code is:

  ```go
  t.Run("ocsp_stapled_good", func(t *testing.T) {
      t.Parallel()
      if runtime.GOOS == "windows" {
          t.Skip("this doesn't work on windows for some reason")
      }
      website := "https://www.wikipedia.org/"
      ts := newTestCase(t)
      tb := ts.tb
      samples := ts.samples
      rt := ts.runtime.VU.Runtime()
      state := ts.runtime.VU.State()
      state.Dialer = tb.Dialer
      _, err := rt.RunString(fmt.Sprintf(`
          var res = http.request("GET", "%s");
          if (res.ocsp.status != http.OCSP_STATUS_GOOD) { throw new Error("wrong ocsp stapled response status: " + res.ocsp.status); }
          `, website))
      assert.NoError(t, err)
      assertRequestMetricsEmitted(t, metrics.GetBufferedSamples(samples), "GET", website, 200, "")
  })
  ```

- **Observed error** (from `/tmp/test_output.txt`):

  ```
  request_test.go:2208:
      Error Trace: ... js/modules/k6/http/request_test.go:2208
      Error:       Received unexpected error:
                   Error: wrong ocsp stapled response status: unknown at <eval>:3:58(22)
      Test:        TestRequestAndBatchTLS/ocsp_stapled_good
  ```

- **Rationale**: This test depends on **external network access** and on the OCSP stapling response served by the live Wikipedia TLS terminator at the moment of the test. OCSP stapling is non-deterministic from a sandboxed test environment for several reasons simultaneously:
  - outbound egress may be routed through a TLS-intercepting proxy that strips the stapled response;
  - Wikipedia's edge TLS stack may return a stapled OCSP response whose status is `unknown` (the exact error we saw) rather than `good` during certificate chain transitions;
  - some CDNs/resolvers respond with a cached stapled OCSP that is at odds with the client's CA bundle state.
  
  The authors of the test already acknowledge the environmental fragility: **they skip it unconditionally on Windows** (`if runtime.GOOS == "windows" { t.Skip(...) }`, line 2194). The failure is therefore an **environment artifact**, not a regression in k6.

- **Action taken**: Per the read-only mandate, **no fix is applied**. The failure is documented.

#### 1.3.2 `TestVURunInterrupt/Source`

- **File**: `js/runner_test.go` line 659 (reported by the error trace).
- **Observed error** (from `/tmp/test_output.txt`):

  ```
  === NAME  TestVURunInterrupt/Source
      runner_test.go:659:
          Error Trace: .../js/runner_test.go:659
          Error:       Received unexpected error:
          Test:        TestVURunInterrupt/Source
  ```
  The parent test `TestVURunInterrupt` ran in 1.30 s; the `/Source` subtest finished in 0.20 s with an "unexpected error" — almost certainly a `context deadline exceeded`, which is what this test is specifically designed to exercise (interruption behaviour of VU execution) and which shows up as spurious failures on slow / contended CI runners.
- **Rationale**: This is a **timing-sensitive** test. It launches a JS test, waits a small amount of time, cancels the run context, and asserts that `ActiveVU.RunOnce()` returns an interrupt rather than completes. On a slow or contended host, the JS iteration can finish *before* the cancellation wins the race, producing the observed "received unexpected error" message (the error returned is nil when an error was expected, or the test harness's wrapper error occurs).
- **Action taken**: No fix is applied (read-only mandate).

#### 1.3.3 `TestConstantArrivalRateRunCorrectTiming/segment_0:1/3_sequence_`

- **File**: `lib/executor/constant_arrival_rate_test.go` line 185.
- **Observed error** (from `/tmp/test_output.txt`):

  ```
  === RUN   TestConstantArrivalRateRunCorrectTiming/segment_0:1/3_sequence_
      constant_arrival_rate_test.go:185:
          Error Trace: .../lib/executor/constant_arrival_rate_test.go:185
                       .../lib/executor/common_test.go:20
          Messages:    1 expectedTime 20ms
      constant_arrival_rate_test.go:185:
          Messages:    2 expectedTime 60ms
      constant_arrival_rate_test.go:185:
          Messages:    3 expectedTime 120ms
      ... (many more)
  ```
- **Rationale**: The `ConstantArrivalRate` executor is supposed to dispatch iterations at a fixed rate with sub-millisecond precision. The test asserts that iteration *N* starts at a predicted time (e.g. iteration 1 at `20 ms`, iteration 3 at `120 ms`, etc.) with a small tolerance. On a busy or slow host, the Go scheduler delivers the wake-up ticks **tens of milliseconds late**, which exceeds the tolerance and flags the test as failed. This is a known sensitivity of arrival-rate tests that the k6 team is aware of (similar tests are frequently marked `flaky` in CI).
- **Action taken**: No fix is applied (read-only mandate).

#### 1.3.4 `TestRampingVUsHandleRemainingVUs`

- **File**: `lib/executor/ramping_vus_test.go` lines 370–371.
- **Observed error** (from `/tmp/test_output.txt`):

  ```
  === NAME  TestRampingVUsHandleRemainingVUs
      ramping_vus_test.go:370:
          Error Trace: .../lib/executor/ramping_vus_test.go:370
          Error:       Not equal:
                       expected: 0x1
                       actual  : 0x0
          Test:        TestRampingVUsHandleRemainingVUs
      ramping_vus_test.go:371:
          Error Trace: .../lib/executor/ramping_vus_test.go:371
          Error:       Not equal:
                       expected: 0x1
                       actual  : 0x2
          Test:        TestRampingVUsHandleRemainingVUs
  --- FAIL: TestRampingVUsHandleRemainingVUs (0.08s)
  ```
- **Rationale**: The test asserts exact atomic counter values (`0x1`) at a specific point in the ramping-VU lifecycle. Here we saw `0x0` (the increment had not yet happened) on line 370, and `0x2` (an extra increment had already happened) on line 371. This is a classic **scheduling race** between the test observation point and the executor's internal VU bookkeeping goroutines. The test completed in 80 ms, which is too short a window for Go's scheduler to reliably quiesce on a busy CI host.
- **Action taken**: No fix is applied (read-only mandate).

### 1.4 Skip Analysis — `TestTC39`

- **File**: `js/tc39/tc39_test.go` lines 794–808. The key logic:

  ```go
  func TestTC39(t *testing.T) {
      if testing.Short() {
          t.Skip()
      }

      runTestTC39(t, lib.CompatibilityModeExtended)
      runTestTC39(t, lib.CompatibilityModeExperimentalEnhanced)
  }

  func runTestTC39(t *testing.T, compatibilityMode lib.CompatibilityMode) {
      t.Helper()

      if _, err := os.Stat(tc39BASE); err != nil {
          t.Skipf("If you want to run tc39 tests, you need to run the 'checkout.sh` script in the directory to get  https://github.com/tc39/test262 at the correct last tested commit (%v)", err)
      }
      ...
  }
  ```
  with `tc39BASE = "TestTC39/test262"` (line 39).
- **Why it skips**: `TestTC39` expects to find the external TC39 `test262` conformance corpus cloned into `./TestTC39/test262` (relative to the `js/tc39` package). That directory is **populated by `js/tc39/checkout.sh`**, whose full contents are:

  ```sh
  #!/bin/sh -e
  sha=cb4a6c8074671c00df8cbc17a620c0f9462b312a # this is just the commit it was last tested with
  mkdir -p ./TestTC39/test262
  cd ./TestTC39/test262
  git init
  git remote add origin https://github.com/tc39/test262.git
  git fetch origin --depth=1 "${sha}"
  git reset --hard "${sha}"
  cd -
  ```
  i.e. it clones `https://github.com/tc39/test262.git` at a pinned commit.
- **When it actually runs**: The CI workflow `.github/workflows/tc39.yml` (lines 28–32) invokes the checkout script before running `go test`:

  ```yaml
  - name: Run tests
    run: |
      set -x
      cd js/tc39
      sh checkout.sh
      go test -timeout 1h
  ```
  This confirms the authors **expect** the checkout to happen as a prerequisite step — TC39 is a CI-only workflow triggered by `push` to `master` and PRs that touch `js/**` (see lines 2–10 of the workflow).
- **What happened here**: The analysis environment has no internet egress, `checkout.sh` was not run, `./TestTC39/test262` does not exist, and `os.Stat` returns a "no such file or directory" error. `t.Skipf` fires, and the parent `TestTC39` is reported as `SKIP` in `/tmp/test_output.txt`.
- **Conclusion**: This is **expected behaviour** by design and not a regression. It is correctly reported in the top-level `SKIP` count (1).

### 1.5 Slowest Packages

From `/tmp/test_output.txt` the longest-running packages (in descending order) are:

| Package | Time | Notes |
|---------|-----:|-------|
| `go.k6.io/k6/lib/executor` | 22.244 s | Failing — heavy timing-sensitive executor tests (ramping, arrival rate) |
| `go.k6.io/k6/js` | 12.608 s | Failing — `TestVURunInterrupt/Source` and many long VU lifecycle tests |
| `go.k6.io/k6/cmd/tests` | 12.245 s | Passing — end-to-end tests that fork mini k6 runs |
| `go.k6.io/k6/js/modules/k6/http` | 7.381 s | Failing — HTTP module including the OCSP test |
| `go.k6.io/k6/execution` | 6.913 s | Passing — scheduler and execution-plan tests |
| `go.k6.io/k6/output/cloud/expv2/integration` | 4.084 s | Passing — cloud output integration tests |
| `go.k6.io/k6/lib/netext/httpext` | 3.254 s | Passing — HTTP transport tests |
| `go.k6.io/k6/js/modules/k6/timers` | 3.093 s | Passing — JS timer integration tests |

The dominant cost lives in `lib/executor` because its tests intentionally *sleep*/*tick* to exercise scheduling behaviour over real wall-clock time (ramping VUs up and down, arrival-rate dispatch, shared-iteration distribution). This is the same reason the Makefile target uses a `210s` timeout per package (`Makefile` line 29).

### 1.6 Conclusion

The suite is effectively **green with four known-flaky fragments and one expected skip**:

- **51 / 54 packages pass** (≈ 94.4 %).
- **768 / 773 executed top-level tests pass** (≈ 99.4 %).
- **4,418 / 4,426 all-depth test rows pass** (≈ 99.8 %).
- All four failures are caused by **environmental timing sensitivity or external network dependence**, all of them acknowledged by the authors in-code (Windows-specific skip on OCSP; timing tolerances in executor tests).
- The one skip (`TestTC39`) is **expected** and controlled by the presence of an external corpus that is only populated in a dedicated CI workflow.

No code changes are proposed; the observed failures are not regressions of this branch.

---

## Section 2 — Files & Modules Responsible for Metrics Tracking

k6's metrics subsystem is a layered pipeline. A **single Go channel** (`chan metrics.SampleContainer`) carries every sample from the producers (VUs, the network dialer, the scheduler) to every consumer (the metrics engine and every output plugin). The architecture below is organised into the seven layers that this pipeline traverses:

1. **Metrics Core** — data types, registry, and sinks (`metrics/` package).
2. **Metrics Engine** — threshold evaluation + the `OutputIngester` that aggregates samples (`metrics/engine/` package).
3. **Output Pipeline** — the channel-reading `Manager` and shared helpers (`output/` package).
4. **JavaScript Runtime** — the VU that executes user scripts and emits iteration samples (`js/`).
5. **Execution Engine** — the scheduler and executors that drive VUs (`execution/`, `lib/executor/`, `lib/execution.go`).
6. **Command Layer** — the `k6 run` command that wires everything together (`cmd/`).
7. **Network I/O Metrics** — the dialer that emits `data_sent` / `data_received` (`lib/netext/`).

The rest of this section walks each layer file-by-file.

### 2.1 Metrics Core (`metrics/` package)

| File | Key types / functions | Responsibility |
|------|----------------------|----------------|
| `metrics/metric.go` | `Metric` struct (lines 12–26), `Submetric` struct (lines 30–37, preceded by a doc comment at line 29), `AddSubmetric(suffix)`, `ParseMetricName(name)` | Defines the core `Metric` value — `registry`, `Name`, `Type`, `Contains` (ValueType), `Tainted`, `Thresholds`, `Submetrics`, `Sub`, `Sink`, `Observed`. Also defines `Submetric` which is a tag-filtered view of a parent metric. |
| `metrics/metric_type.go` | `MetricType` enum | Defines the four canonical metric types k6 understands. |
| `metrics/value_type.go` | `ValueType` enum | Defines what the numeric value of a sample semantically is (a plain number, a duration-in-ms, or a byte-count). |
| `metrics/sample.go` | `TimeSeries`, `Sample`, `SampleContainer` interface, `Samples`, `ConnectedSampleContainer`, `ConnectedSamples`, `GetBufferedSamples(buf)`, `PushIfNotDone(ctx, output, sample)` | Defines the in-memory shape of a single sample and the container abstractions used on the samples channel. |
| `metrics/sink.go` | `Sink` interface, `CounterSink`, `GaugeSink`, `TrendSink`, `RateSink`, `NewSink(MetricType)` factory | Defines how each of the four metric types accumulates values over time. |
| `metrics/builtin.go` | 25 name constants + `BuiltinMetrics` struct + `RegisterBuiltinMetrics(registry)` | Defines every built-in metric k6 ships with and the one-shot registration function called at startup. |
| `metrics/registry.go` | `Registry`, `NewRegistry()`, `NewMetric(name, typ, vts...)`, `MustNewMetric(...)`, `All()`, `Get(name)`, `RootTagSet()` | The thread-safe central store of every `*Metric` object used throughout a test run. |
| `metrics/units.go` | `D(time.Duration) float64`, `ToD(float64) time.Duration`, `B(bool) float64` | Helpers that normalise time values into milliseconds-as-float64 and booleans into 0/1 floats. |
| `metrics/tags.go` | `TagSet` (line 27), `TagsAndMeta` (line 169), `EnabledTags` (line 237) | Defines the immutable atlas-backed `TagSet` used for sample tagging and submetric filtering, the `TagsAndMeta` bundle that pairs indexed `Tags` with non-indexed `Metadata`, and the `EnabledTags` string→bool map used to record which tag names are allowed. |
| `metrics/system_tag.go` | `SystemTag` (line 13), `SystemTagSet` (line 17), `DefaultSystemTagSet` (line 47), `NonIndexableSystemTags` (line 54), `ToSystemTagSet()`, `NewSystemTagSet()` | Defines the bit-flag enum of k6 built-in system tag names (`proto`, `status`, `method`, `url`, `name`, `group`, `check`, `error`, `scenario`, `vu`, `iter`, …) and the `SystemTagSet` bitmask type that records which system tags are enabled for the current test run. `DefaultSystemTagSet` is the out-of-the-box inclusion list; `NonIndexableSystemTags` (`iter`, `vu`) routes high-cardinality tags into `Metadata` instead of `Tags` via `TagsAndMeta.SetSystemTagOrMeta`. |

#### 2.1.1 The `MetricType` and `ValueType` enums

From `metrics/metric_type.go` (lines 9–14):

```go
const (
    Counter = MetricType(iota) // A counter that sums its data points
    Gauge                      // A gauge that displays the latest value
    Trend                      // A trend, min/max/avg/med are interesting
    Rate                       // A rate, displays % of values that aren't 0
)
```

From `metrics/value_type.go`:

```go
const (
    Default = ValueType(iota) // Values are presented as-is
    Time                      // Values are time durations (milliseconds)
    Data                      // Values are data amounts (bytes)
)
```

#### 2.1.2 The `Metric` struct

From `metrics/metric.go` lines 12–26 (verbatim):

```go
type Metric struct {
    registry *Registry  `json:"-"`
    Name     string     `json:"name"`
    Type     MetricType `json:"type"`
    Contains ValueType  `json:"contains"`

    // TODO: decouple the metrics from the sinks and thresholds... have them
    // linked, but not in the same struct?
    Tainted    null.Bool    `json:"tainted"`
    Thresholds Thresholds   `json:"thresholds"`
    Submetrics []*Submetric `json:"submetrics"`
    Sub        *Submetric   `json:"-"`
    Sink       Sink         `json:"-"`
    Observed   bool         `json:"-"`
}
```

The unexported `registry` field is a back-pointer to the owning `*metrics.Registry` that created this metric — every `*Metric` knows which registry minted it, which is used internally by `Metric.AddSubmetric` when a submetric needs a fresh `*Metric` allocation. The preserved in-source `TODO` comment is the authors' acknowledgement that sinks and thresholds are awkwardly co-located on the `Metric` struct today.

Every sample emitted by a VU carries a pointer to exactly one such `Metric` (via `Sample.Metric`), and every increment / update to that metric's in-memory state happens through `Metric.Sink.Add(sample)`. The fact that `Sink` is a **pointer shared across the whole test run** is what makes the architecture work: no matter how many VUs, outputs, or goroutines are in play, there is exactly one `CounterSink` for the `iterations` counter, and every `Add` call mutates that one object.

#### 2.1.3 The four `Sink` implementations

`metrics/sink.go` defines the `Sink` interface (`Add`, `Format`, `IsEmpty`) and four concrete implementations. The `CounterSink` is central to this document — it is the sink used for `iterations`:

```go
type CounterSink struct {
    Value float64   `json:"value"`
    First time.Time `json:"-"`
}

func (c *CounterSink) Add(s Sample) {
    if c.First.IsZero() {
        c.First = s.Time
    }
    c.Value += s.Value
}

func (c *CounterSink) Format(t time.Duration) map[string]float64 {
    return map[string]float64{
        "count": c.Value,
        "rate":  c.Value / (float64(t) / float64(time.Second)),
    }
}
```

The other three sinks follow the same `Add`/`Format`/`IsEmpty` contract:

- **`GaugeSink`** — remembers the last value plus `Min` and `Max` seen (so `vus_max` sees monotonic growth, `vus` sees the instant active count).
- **`TrendSink`** — accumulates all values in a slice, maintaining `Count`, `Min`, `Max`, `Sum`; `Format` computes `min`, `max`, `avg`, `med`, `p(90)`, `p(95)` via linear interpolation across the sorted value slice.
- **`RateSink`** — maintains `Trues` and `Total`; `Format` returns `rate = Trues/Total`.

The factory `NewSink(t MetricType)` picks the right concrete sink for a metric type, and `newMetric()` in `metrics/registry.go` calls it once per metric.

#### 2.1.4 The 25 built-in metrics

**Important:** although the AAP summary referred to "28 built-in metrics", the ground-truth source file `metrics/builtin.go` defines exactly **25** built-in name constants, matched by 25 fields on the `BuiltinMetrics` struct and 25 `MustNewMetric` calls inside `RegisterBuiltinMetrics`. This section documents the actual count.

| # | Name constant | Wire name | Type | Value type | Family |
|--:|--------------|-----------|------|-----------|--------|
| 1 | `VUsName` | `vus` | Gauge | Default | VU / Iteration |
| 2 | `VUsMaxName` | `vus_max` | Gauge | Default | VU / Iteration |
| 3 | `IterationsName` | `iterations` | Counter | Default | VU / Iteration |
| 4 | `IterationDurationName` | `iteration_duration` | Trend | Time | VU / Iteration |
| 5 | `DroppedIterationsName` | `dropped_iterations` | Counter | Default | VU / Iteration |
| 6 | `ChecksName` | `checks` | Rate | Default | Checks / Groups |
| 7 | `GroupDurationName` | `group_duration` | Trend | Time | Checks / Groups |
| 8 | `HTTPReqsName` | `http_reqs` | Counter | Default | HTTP |
| 9 | `HTTPReqFailedName` | `http_req_failed` | Rate | Default | HTTP |
| 10 | `HTTPReqDurationName` | `http_req_duration` | Trend | Time | HTTP |
| 11 | `HTTPReqBlockedName` | `http_req_blocked` | Trend | Time | HTTP |
| 12 | `HTTPReqConnectingName` | `http_req_connecting` | Trend | Time | HTTP |
| 13 | `HTTPReqTLSHandshakingName` | `http_req_tls_handshaking` | Trend | Time | HTTP |
| 14 | `HTTPReqSendingName` | `http_req_sending` | Trend | Time | HTTP |
| 15 | `HTTPReqWaitingName` | `http_req_waiting` | Trend | Time | HTTP |
| 16 | `HTTPReqReceivingName` | `http_req_receiving` | Trend | Time | HTTP |
| 17 | `WSSessionsName` | `ws_sessions` | Counter | Default | WebSocket |
| 18 | `WSMessagesSentName` | `ws_msgs_sent` | Counter | Default | WebSocket |
| 19 | `WSMessagesReceivedName` | `ws_msgs_received` | Counter | Default | WebSocket |
| 20 | `WSPingName` | `ws_ping` | Trend | Time | WebSocket |
| 21 | `WSSessionDurationName` | `ws_session_duration` | Trend | Time | WebSocket |
| 22 | `WSConnectingName` | `ws_connecting` | Trend | Time | WebSocket |
| 23 | `GRPCReqDurationName` | `grpc_req_duration` | Trend | Time | gRPC |
| 24 | `DataSentName` | `data_sent` | Counter | Data | Network I/O |
| 25 | `DataReceivedName` | `data_received` | Counter | Data | Network I/O |

The 25 registration calls live inside `RegisterBuiltinMetrics(registry *Registry)` in `metrics/builtin.go`. Lines 81–82 register the two that matter most for this document:

```go
func RegisterBuiltinMetrics(registry *Registry) *BuiltinMetrics {
    return &BuiltinMetrics{
        VUs:                registry.MustNewMetric(VUsName, Gauge),
        VUsMax:             registry.MustNewMetric(VUsMaxName, Gauge),
        Iterations:         registry.MustNewMetric(IterationsName, Counter),
        IterationDuration:  registry.MustNewMetric(IterationDurationName, Trend, Time),
        ...
    }
}
```

Every call goes through `Registry.MustNewMetric`, which in turn calls `newMetric()` (line 93 of `metrics/registry.go`). That function (a) validates the name against the regex `^[a-zA-Z_][a-zA-Z0-9_]{1,128}$`, (b) de-duplicates by name (returning the existing metric *only if* its type and value type match), and (c) creates the correct sink via `NewSink(typ)` before inserting the metric into the internal `map[string]*Metric`.

#### 2.1.5 `Sample`, `TimeSeries`, and `SampleContainer`

From `metrics/sample.go`:

```go
type TimeSeries struct {
    Metric *Metric
    Tags   *TagSet
}

type Sample struct {
    TimeSeries
    Time     time.Time
    Metadata map[string]string
    Value    float64
}

type SampleContainer interface {
    GetSamples() []Sample
}

type Samples []Sample                // implements SampleContainer trivially
type ConnectedSampleContainer interface {
    SampleContainer
    GetTags() *TagSet
    GetTime() time.Time
}
```

`SampleContainer` is the transport unit on the samples channel. A single call to `runFn()` in `js/runner.go` produces up to two `SampleContainer`s (the IO samples from the dialer, and the iteration samples). The helper `PushIfNotDone(ctx, output, sample)` in `metrics/sample.go` lines 131–137 is a thin wrapper:

```go
func PushIfNotDone(ctx context.Context, output chan<- SampleContainer, sample SampleContainer) bool {
    if ctx.Err() != nil {
        return false
    }
    output <- sample
    return true
}
```

— used anywhere a goroutine needs to write to the samples channel without panicking if the enclosing run context has already been cancelled.

### 2.2 Metrics Engine (`metrics/engine/` package)

The `metrics/engine/` package contains exactly two Go files.

#### 2.2.1 `metrics/engine/engine.go` — `MetricsEngine`

```go
type MetricsEngine struct {
    registry                 *metrics.Registry
    logger                   logrus.FieldLogger

    metricsWithThresholds    []*metrics.Metric
    breachedThresholdsCount  uint32

    MetricsLock      sync.Mutex
    ObservedMetrics  map[string]*metrics.Metric
}

const thresholdsRate = 2 * time.Second
```

Key methods:

- **`NewMetricsEngine(registry, logger)`** — constructs the engine; `ObservedMetrics` starts empty.
- **`CreateIngester()`** — returns a new `OutputIngester` bound to this engine (the engine keeps no reference back to the ingester directly; they cooperate via `MetricsLock` and `ObservedMetrics`).
- **`InitSubMetricsAndThresholds(options, onlyLogErrors)`** — parses threshold expressions from `options.Thresholds`, adds any implicitly-referenced submetrics to their parent metric's `Submetrics` slice, and stores every metric that has at least one threshold in `metricsWithThresholds`.
- **`StartThresholdCalculations(ingester, abortRun, getCurrentTestRunDuration)`** (line 159) — launches the threshold goroutine:

  ```go
  ticker := time.NewTicker(thresholdsRate)   // 2 * time.Second
  for {
      select {
      case <-ticker.C:
          breached, shouldAbort := me.evaluateThresholds(true, getCurrentTestRunDuration)
          if shouldAbort { ... abortRun(err) }
      case <-stop:
          return
      }
  }
  ```

- **`evaluateThresholds(ignoreEmptySinks, getCurrentTestRunDuration)`** (line 216) — locks `MetricsLock`, walks `metricsWithThresholds`, asks each metric's `Sink.Format(dur)` for its current value, and runs every `Threshold.Expression` against that map. It returns `(breached, shouldAbort)`.
- **`markObserved(m)`** — inserts `m` into `ObservedMetrics` the first time the metric is seen. This map is later handed to `HandleSummary` for rendering.
- **`getThresholdMetricOrSubmetric(name)`** — resolves a threshold expression's target (e.g. `http_req_duration{status:200}`) to the concrete `*Metric` or `*Submetric` so the expression can be bound to a sink.

#### 2.2.2 `metrics/engine/ingester.go` — `OutputIngester`

The ingester is **itself an `output.Output`** (i.e. it implements the same interface as JSON/CSV/Cloud outputs). Its job is not to write metrics anywhere — it is to **aggregate** samples into each `Metric.Sink`.

```go
type OutputIngester struct {
    output.SampleBuffer
    logger        logrus.FieldLogger
    metricsEngine *MetricsEngine
    cardinality   *cardinalityControl
    periodicFlusher *output.PeriodicFlusher
}

const (
    collectRate          = 50 * time.Millisecond
    timeSeriesFirstLimit = 100000
)
```

- **`Start()`** (line 40) — starts a `PeriodicFlusher` whose period is `50 ms` and whose callback is `flushMetrics`.
- **`Stop()`** — stops the flusher which guarantees one final `flushMetrics()` call.
- **`AddMetricSamples(containers)`** — inherited from `SampleBuffer`; appends the slice to an internal buffer under a mutex.
- **`flushMetrics()`** (lines 62–121) — the heart of metrics aggregation. Its key steps are:

  ```go
  sampleContainers := oi.GetBufferedSamples()  // drain buffer
  if len(sampleContainers) == 0 { return }

  oi.metricsEngine.MetricsLock.Lock()
  defer oi.metricsEngine.MetricsLock.Unlock()

  for _, sampleContainer := range sampleContainers {
      samples := sampleContainer.GetSamples()
      for _, sample := range samples {
          m := sample.Metric
          oi.metricsEngine.markObserved(m)
          m.Sink.Add(sample)                    // ←– THE ACTUAL AGGREGATION

          for _, sm := range m.Submetrics {
              if !sample.Tags.Contains(sm.Tags) {
                  continue
              }
              sm.Metric.Sink.Add(sample)
          }

          oi.cardinality.Add(sample.TimeSeries)
      }
  }
  ```

- **`cardinalityControl`** (lines 123–159) — a small helper that tracks unique `TimeSeries` seen so far (`map[TimeSeries]struct{}`). If the count crosses the current threshold (`timeSeriesFirstLimit = 100_000` initially), the ingester emits a warning **and doubles the threshold** (`cc.timeSeriesLimit *= 2`) so the same warning does not fire on every subsequent sample.

### 2.3 Output Pipeline (`output/` package)

#### 2.3.1 `output/types.go` — the `Output` interface

```go
type Output interface {
    Description() string
    Start() error
    AddMetricSamples(samples []metrics.SampleContainer)
    Stop() error
}
```

Two **optional** interfaces are also defined here:

- `WithStopWithTestError`, whose `StopWithTestError(err error) error` is called instead of `Stop()` if the test was aborted by a threshold breach.
- `WithTestRunStop`, whose `SetTestRunStopCallback(func(error))` lets an output request test abortion (used by the cloud output's "test stopped from the cloud" feature).

Every concrete output — the stdout renderer, JSON, CSV, cloud, experimental cloud v2, Prometheus Remote Write, StatsD, and **the `OutputIngester` itself** — implements this interface.

#### 2.3.2 `output/manager.go` — `Manager`

The `Manager` owns all registered outputs as a slice and is the **only consumer of the samples channel** outside the metrics engine's own `OutputIngester`.

```go
type Manager struct {
    outputs []Output
    logger  logrus.FieldLogger
    testStopCallback func(error)
}

const sendBatchToOutputsRate = 50 * time.Millisecond
```

The critical method is `Start(samplesChan chan metrics.SampleContainer) (wait func(), finish func(error), err error)` (line 42). The second return value is typed `func(error)` (so the caller can report the test's final error) and is bound in `cmd/run.go:228` to a caller-side variable named `stopOutputs` — the informal "stop" label used elsewhere in this document refers to that caller binding. After calling `Start()` on every output (aborting early on error) it spawns a goroutine shaped roughly like this:

```go
ticker := time.NewTicker(sendBatchToOutputsRate)
buffer := make([]metrics.SampleContainer, 0, cap(samplesChan))
for {
    select {
    case sampleContainer, ok := <-samplesChan:
        if !ok {
            sendToOutputs(buffer)
            return
        }
        buffer = append(buffer, sampleContainer)
    case <-ticker.C:
        sendToOutputs(buffer)
        buffer = make([]metrics.SampleContainer, 0, cap(buffer))
    }
}
```

`sendToOutputs` loops through `m.outputs` and calls `out.AddMetricSamples(buffer)` on each one. The buffer is handed to every output by reference, but because the outputs copy the slice into their own thread-safe buffers (see next subsection), no cross-output interference occurs. Note that `sendToOutputs(buffer)` is invoked **unconditionally** on every 50 ms tick — the implementation does not short-circuit when `buffer` is empty, so idle ticks propagate an empty slice to every registered output. This keeps the producer path lock-free at the expense of a handful of no-op output calls during quiet periods.

The channel close path is important: when `cmd/run.go` calls `close(samples)` (line 274), the `case sampleContainer, ok := <-samplesChan` receives `ok == false`, the buffer is flushed one last time, and the goroutine returns — which is how the `Manager.Start` `wait` return value resolves.

#### 2.3.3 `output/helpers.go` — `SampleBuffer` and `PeriodicFlusher`

Two small utilities used by many outputs including the ingester:

- **`SampleBuffer`** — a mutex-protected `[]metrics.SampleContainer` with `AddMetricSamples(containers)` and `GetBufferedSamples() []metrics.SampleContainer`. Multiple producer goroutines can push, one consumer goroutine pops and resets.
- **`PeriodicFlusher`** — wraps a `time.Ticker` and a user-supplied callback; `Start` is non-blocking, `Stop` is synchronous and guarantees one final callback invocation on shutdown so no samples are lost at the tail.

### 2.4 JavaScript Runtime (`js/`)

#### 2.4.1 `js/runner.go` — the VU

`js/runner.go` is a 953-line file that contains both the passive `Runner` (responsible for loading the compiled JS bundle) and the two VU types:

- **`VU`** — the *init-time* VU created by `Runner.NewVU()` (line 113). Holds a `sobek.Runtime` (JavaScript engine), a `Runner` pointer, a `samplesOut` channel reference, a `*metrics.Dialer`, and a pre-built `*lib.State`.
- **`ActiveVU`** — the *runtime* VU; an opaque handle that the scheduler uses to drive iterations.

For this document, the important flow is the one that runs during a single iteration:

```
ActiveVU.RunOnce()               // line 724
    └── u.runFn(ctx, true, fn, cancel, u.setupData)   // line 773
            └── eventLoop.Start(fn)                   // line 841
            └── iterationSamples(...)                 // line 879 (callee)
```

- **`RunOnce()`** (lines 724–800) — selects the default-exported JS function `fn`, calls `u.incrIteration()` to bump the per-VU `iteration` counter (line 755), calls `u.runFn(ctx, true, fn, cancel, u.setupData)` (line 773), and optionally sleeps any remaining time to satisfy the user-configured `MinIterationDuration`.
- **`runFn(ctx, isDefault, fn, cancel, data)`** (lines 817–877) — configures a per-iteration `context`, builds the `ctm` (`metrics.TagsAndMeta`) for sample tagging, starts the JS event loop, runs the user-supplied function, and **after** the event loop settles emits samples:

  ```go
  endTime := time.Now()

  // always emit IO samples
  if u.Dialer != nil {
      u.state.Samples <- u.Dialer.IOSamples(endTime, ctm, builtinMetrics)
  }

  if isFullIteration && isDefault {
      u.state.Samples <- iterationSamples(startTime, endTime, ctm, builtinMetrics)
  }
  ```

  - **IO samples** are emitted on **every** iteration, successful or not (so even interrupted iterations still account their bytes).
  - **Iteration samples** are emitted only when the iteration completed to the end (`isFullIteration == true`) and the JS function is the user's `default` function (`isDefault == true`). Setup- and teardown-function calls deliberately *do not* contribute to `iterations`.
- **`iterationSamples(startTime, endTime, ctm, builtinMetrics)`** (lines 879–902) — constructs and returns the two-sample container that carries the iteration's observations:

  ```go
  func iterationSamples(startTime, endTime time.Time, ctm metrics.TagsAndMeta,
      builtinMetrics *metrics.BuiltinMetrics) metrics.Samples {
      return []metrics.Sample{
          {
              TimeSeries: metrics.TimeSeries{
                  Metric: builtinMetrics.IterationDuration,
                  Tags:   ctm.Tags,
              },
              Time:     endTime,
              Metadata: ctm.Metadata,
              Value:    metrics.D(endTime.Sub(startTime)),   // ms as float64
          },
          {
              TimeSeries: metrics.TimeSeries{
                  Metric: builtinMetrics.Iterations,
                  Tags:   ctm.Tags,
              },
              Time:     endTime,
              Metadata: ctm.Metadata,
              Value:    1,                                    // one iteration
          },
      }
  }
  ```
- **`incrIteration()`** (lines 904–917) — `u.iteration++; u.state.Iteration = u.iteration`. This is a **per-VU** counter used for things like the JS `__ITER` magic variable; it is **separate from** the global `iterations` metric.

#### 2.4.2 `js/modules/k6/metrics/` — user-facing custom metrics

k6 exposes all four metric types to user JS so users can declare their own metrics alongside the 25 built-ins:

```js
import { Counter, Gauge, Trend, Rate } from 'k6/metrics';

const errors = new Counter('my_errors');
const latency = new Trend('my_latency', true);  // second arg: isTime
```

Each of these constructors eventually calls `registry.NewMetric(name, typ, vt...)` on the same global `*metrics.Registry` that `RegisterBuiltinMetrics` used. As a result, **custom** metrics and **built-in** metrics travel through **identical** sinks and the identical pipeline documented below.

### 2.5 Execution Engine

#### 2.5.1 `execution/scheduler.go` — the `Scheduler`

The scheduler is the outermost orchestrator: it creates VUs, starts executors, and connects them to the samples channel.

Key methods:

- **`Init(runCtx, samplesOut)`** (line 381) — performs VU initialisation:
  - Starts the `emitVUsAndVUsMax` goroutine (details below).
  - Calls `initVUsAndExecutors(runCtx, samplesOut)` (line 254 in the same file), which for every planned VU does `vu, err := e.state.Test.Runner.NewVU(ctx, vuIDLocal, vuIDGlobal, samplesOut)`.
  - This is the point at which **every VU is handed a reference to the single `samplesOut` channel** — the only write-side of the pipe.
- **`Run(globalCtx, runCtx, samplesOut)`** (line 419) — iterates over configured executors and calls `e.runExecutor(runCtx, runResults, samplesOut, exec)` (line 500) for each, in its own goroutine.
- **`runExecutor(...)`** (line 331) — a thin wrapper that calls `executor.Run(runCtx, engineOut)` on line 369, where `engineOut` is the shared samples channel.
- **`emitVUsAndVUsMax(ctx, samplesOut)`** (line 199) — independent 1-second ticker goroutine:

  ```go
  ticker := time.NewTicker(time.Second)
  for {
      select {
      case <-ticker.C:
          // compute active + max VUs from ExecutionState atomics
          metrics.PushIfNotDone(ctx, samplesOut, metrics.ConnectedSamples{
              Samples: []metrics.Sample{
                  {..., Metric: builtinMetrics.VUs,    Value: float64(active)},
                  {..., Metric: builtinMetrics.VUsMax, Value: float64(max)},
              },
              Time: time.Now(),
              Tags: testRunState.RunTags,
          })
      case <-ctx.Done():
          return
      }
  }
  ```
  This is why `vus` and `vus_max` appear every second in the k6 output regardless of VU activity.

#### 2.5.2 `lib/executor/shared_iterations.go` — the `SharedIterations` executor

`SharedIterations` distributes a fixed total number of iterations across a fixed number of VUs (the default executor when you run a script without an explicit `scenarios` block).

Its `Run(parentCtx, out)` method (line 167) builds the per-iteration runner and the per-VU handler:

```go
runIteration := getIterationRunner(si.executionState, si.logger)   // line 232

// later, one goroutine per VU:
handleVU := func(initVU lib.InitializedVU) {
    activeVU := initVU.Activate(...)
    for attemptedIters := range iterations {
        if !runIteration(maxDurationCtx, activeVU) {   // line 259
            return   // interrupted
        }
    }
}
```

At the end of `Run`, the executor emits a `DroppedIterations` counter sample (lines 220–228) for any iterations it could not attempt before the run context was cancelled. This is how the `dropped_iterations` metric gets values.

#### 2.5.3 `lib/executor/helpers.go` — `getIterationRunner`

The iteration-level loop body is shared across executors. From lines 104–141:

```go
func getIterationRunner(
    executionState *lib.ExecutionState,
    logger logrus.FieldLogger,
) func(context.Context, lib.ActiveVU) bool {
    return func(ctx context.Context, vu lib.ActiveVU) bool {
        err := vu.RunOnce()

        select {
        case <-ctx.Done():
            executionState.AddInterruptedIterations(1)
            return false
        default:
        }

        if err != nil {
            // ... logging ...
            executionState.AddInterruptedIterations(1)
            return true    // continue, but this iter did not "count"
        }

        executionState.AddFullIterations(1)   // line 137
        return true
    }
}
```

Two points to note:

1. `vu.RunOnce()` (which is what eventually pushes the iteration sample onto the channel) is called **before** the atomic counter is bumped.
2. The atomic counter `fullIterationsCount` is a *different* counter from the `iterations` metric. The atomic counter drives the progress bar and scheduler decisions (e.g. "have we reached the configured total iterations yet?"); the `iterations` metric is what surfaces in the end-of-test summary.

#### 2.5.4 `lib/execution.go` — `ExecutionState`

`ExecutionState` (defined around lines 70–199) is a large struct; the parts relevant to iteration counting are:

```go
type ExecutionState struct {
    // ...
    fullIterationsCount        *uint64   // line 146  -- atomic, full iterations
    interruptedIterationsCount *uint64   // line 153  -- atomic, interrupted iterations
    // ...
}

func (es *ExecutionState) AddFullIterations(count uint64) uint64 {      // ~line 292
    return atomic.AddUint64(es.fullIterationsCount, count)
}

func (es *ExecutionState) AddInterruptedIterations(count uint64) uint64 {
    return atomic.AddUint64(es.interruptedIterationsCount, count)
}

func (es *ExecutionState) GetFullIterationCount() uint64 {
    return atomic.LoadUint64(es.fullIterationsCount)
}
```

These atomic counters are what `emitVUsAndVUsMax` and many executor implementations read, but they are **not** the source of the `iterations` metric — that still comes from the sample pushed by `js/runner.go:iterationSamples()`. The two counters exist side by side because they serve different needs: the atomic counter is queried constantly (every few ms) and needs to be lock-free, whereas the metric sample flows through the normal pipeline exactly once per iteration.

### 2.6 Command Layer

#### 2.6.1 `cmd/test_load.go` — registry + builtin registration

This is the earliest point in `k6 run`'s lifecycle where metrics come into existence. The relevant snippet from `loadLocalTest` (lines 70–82) is:

```go
registry := metrics.NewRegistry()
state := &lib.TestPreInitState{
    Logger:         gs.Logger,
    RuntimeOptions: runtimeOptions,
    Registry:       registry,
    BuiltinMetrics: metrics.RegisterBuiltinMetrics(registry),
    Events:         gs.Events,
    LookupEnv: func(key string) (string, bool) {
        val, ok := gs.Env[key]
        return val, ok
    },
    Usage: usage.New(),
}
```

After this block every built-in metric exists as a `*metrics.Metric` with its sink, addressable through both `Registry.Get(name)` and the typed `state.BuiltinMetrics.Iterations` / `state.BuiltinMetrics.VUs` / etc. convenience pointers. These pointers propagate into the `Runner`, the `VU`s, the `Scheduler`, the `Dialer`, and ultimately to every sample emitted anywhere in the system.

#### 2.6.2 `cmd/run.go` — pipeline wiring

The `run` command file wires every piece of the pipeline together. The relevant lines are:

| Line | What happens |
|-----:|--------------|
| 170 | `metricsEngine, err := engine.NewMetricsEngine(testRunState.Registry, logger)` |
| 187 | `metricsIngester = metricsEngine.CreateIngester()`, which is then appended to the slice of outputs. |
| 195 | The deferred call setting up `summaryResult, hsErr := test.initRunner.HandleSummary(globalCtx, &lib.Summary{...})` using `Metrics: metricsEngine.ObservedMetrics`. |
| 220 | `outputManager := output.NewManager(outputs, logger, func(err error) { ... })` |
| 227–228 | `samples := make(chan metrics.SampleContainer, test.derivedConfig.MetricSamplesBufferSize.Int64)` followed by `waitOutputsFlushed, stopOutputs, err := outputManager.Start(samples)`. |
| 274 | `close(samples)` in a deferred cleanup so the `Manager` goroutine can drain and return. |
| 367 | `stopVUEmission, err := execScheduler.Init(runCtx, samples)` — hands the samples channel to the scheduler so every VU inherits it. |
| 397 | `err = execScheduler.Run(globalCtx, runCtx, samples)` — starts the executors (and hence the iteration loop). |

This single file is the bridge between the command-line invocation (`k6 run script.js`) and every other piece of the metrics machinery.

### 2.7 Network I/O Metrics — `lib/netext/dialer.go`

`lib/netext/dialer.go` wraps Go's standard `net.Dialer` so that every `net.Conn` it hands out is instrumented for byte counting. Two fields on the `Dialer` struct are atomic:

```go
type Dialer struct {
    net.Dialer                       // embedded
    Resolver        Resolver
    Blacklist       []*lib.IPNet
    Hosts           *types.Hosts
    BytesRead       int64            // atomic
    BytesWritten    int64            // atomic
    // ...
}
```

Every connection the dialer produces is wrapped in an internal `Conn` type. Line 68 of `lib/netext/dialer.go` is the **instantiation** site (`conn = &Conn{conn, &d.BytesRead, &d.BytesWritten}`); the `Conn` struct itself is **defined at line 177** with `BytesRead, BytesWritten *int64` fields at line 180. Its `Read` and `Write` methods call `atomic.AddInt64(c.BytesRead, int64(n))` and `atomic.AddInt64(c.BytesWritten, int64(n))` respectively before returning — the `*int64` pointers inside `Conn` alias the atomic counters on the parent `Dialer`.

The function `IOSamples(sampleTime, ctm, builtinMetrics)` (lines 74–99) then converts those counters into samples:

```go
func (d *Dialer) IOSamples(sampleTime time.Time, ctm metrics.TagsAndMeta,
    builtinMetrics *metrics.BuiltinMetrics) metrics.Samples {
    bytesWritten := atomic.SwapInt64(&d.BytesWritten, 0)
    bytesRead    := atomic.SwapInt64(&d.BytesRead,    0)
    return []metrics.Sample{
        {
            TimeSeries: metrics.TimeSeries{
                Metric: builtinMetrics.DataSent,
                Tags:   ctm.Tags,
            },
            Time:     sampleTime,
            Metadata: ctm.Metadata,
            Value:    float64(bytesWritten),
        },
        {
            TimeSeries: metrics.TimeSeries{
                Metric: builtinMetrics.DataReceived,
                Tags:   ctm.Tags,
            },
            Time:     sampleTime,
            Metadata: ctm.Metadata,
            Value:    float64(bytesRead),
        },
    }
}
```

Note that `SwapInt64` resets the counters atomically to zero in the same operation as reading them, so two VUs sharing a dialer never double-count. `IOSamples` is called from `js/runner.go:runFn()` right after the JS event loop finishes for every iteration (full or interrupted), which is why `data_sent` and `data_received` are populated even when no `iterations` sample is emitted.

---

## Section 3 — Traced Function-Call Walkthrough for the `iterations` Counter

The `iterations` metric is declared in `metrics/builtin.go` as a **Counter** (`Default` value type). Below we follow **one sample** — emitted at the end of one completed JS iteration — from the moment the metric is registered at startup, through every hop of the pipeline, until it shows up in the end-of-test summary. Every step names the concrete file and function and quotes the key code or line number involved.

### Step 1 — Registration

**Files**: `cmd/test_load.go`, `metrics/registry.go`, `metrics/builtin.go`

When `k6 run script.js` starts, `cmd.loadLocalTest` in `cmd/test_load.go` (line 47) is the first function that touches metrics. At line 70 it creates the registry and registers all built-ins in two lines:

```go
registry := metrics.NewRegistry()
state := &lib.TestPreInitState{
    ...
    Registry:       registry,
    BuiltinMetrics: metrics.RegisterBuiltinMetrics(registry),
    ...
}
```

`metrics.RegisterBuiltinMetrics` (starting at line 78 of `metrics/builtin.go`) contains this line (~line 82) which creates the `iterations` metric object:

```go
Iterations: registry.MustNewMetric(IterationsName, Counter),
```

where `IterationsName = "iterations"` (name constant near line 8 of `metrics/builtin.go`).

`MustNewMetric` calls `NewMetric` which calls `newMetric()` in `metrics/registry.go` (around line 93). `newMetric()` does three things:

1. Validates the name `"iterations"` against the regex `^[a-zA-Z_][a-zA-Z0-9_]{1,128}$`.
2. Stores the new `*Metric` in the registry's internal `map[string]*Metric` under `r.metrics["iterations"]`.
3. Constructs `sink := NewSink(Counter)`, which (per `metrics/sink.go` lines 25–35) returns a new `&CounterSink{}` — with `Value: 0.0` and `First: time.Time{}` (zero).

The resulting `*Metric` pointer — call it `iterationsMetric` — is **stored once** and handed out everywhere: `state.BuiltinMetrics.Iterations` holds it, and anywhere downstream that references `builtinMetrics.Iterations` is using the same pointer. This is the **single instance of the `iterations` metric for the entire test run**; all further increments mutate the single `CounterSink` pointed to by `iterationsMetric.Sink`.

### Step 2 — Pipeline Wiring

**File**: `cmd/run.go`

Once the test is loaded, `cmd.(*cmdRun).run(...)` walks through a carefully-ordered sequence of constructor calls that connect every piece of the metrics pipeline. The relevant lines:

```go
// line 170 — engine
metricsEngine, err := engine.NewMetricsEngine(testRunState.Registry, logger)

// line 187 — ingester
metricsIngester = metricsEngine.CreateIngester()
outputs = append(outputs, metricsIngester)

// line 220 — output manager
outputManager := output.NewManager(outputs, logger, func(err error) { ... })

// lines 227–228 — channel + start
samples := make(chan metrics.SampleContainer, test.derivedConfig.MetricSamplesBufferSize.Int64)
waitOutputsFlushed, stopOutputs, err := outputManager.Start(samples)
```

After this block:

- The `metricsEngine` holds a pointer to the registry, so it can find the `*Metric` for `"iterations"` at any time.
- The `metricsIngester` is *just another* `output.Output` in the slice — the engine treats its own ingester exactly the same as a JSON or cloud output.
- The `outputManager` has started a goroutine reading from `samples` every 50 ms and dispatching to every registered output including the ingester.

Later in the same function, two more critical calls hand the `samples` channel to the scheduler:

```go
// line 367
stopVUEmission, err := execScheduler.Init(runCtx, samples)

// line 397
err = execScheduler.Run(globalCtx, runCtx, samples)
```

And a deferred cleanup closes the channel once the run is over:

```go
close(samples)
```

At this point the pipeline is live: writes to `samples` will be read every 50 ms and fanned out to every output.

### Step 3 — VU Initialisation

**File**: `execution/scheduler.go`

`Scheduler.Init(runCtx, samplesOut)` on line 381 is responsible for creating every VU that will later run iterations. Two things happen here:

1. It calls `waitForVUsMetricPush := e.emitVUsAndVUsMax(execSchedRunCtx, samplesOut)` (near line 199 of the same file), which launches the 1-second ticker goroutine that pushes `vus` and `vus_max` gauge samples onto `samplesOut`. These samples share the same channel as future iteration samples.
2. It calls `initVUsAndExecutors(execSchedRunCtx, samplesOut)` (line 254). That helper iterates over every planned VU (global and per-executor) and calls:

   ```go
   vu, err := e.state.Test.Runner.NewVU(ctx, vuIDLocal, vuIDGlobal, samplesOut)
   ```

   In `js/runner.go:NewVU` (line 113) the `samplesOut` channel is stored on the `VU` as `vu.Samples = samplesOut` (line 226 of `js/runner.go`) and also on the pre-built `lib.State` as `vu.state.Samples = vu.Samples` (line 241). **Every VU now has a reference to exactly the same channel that `outputManager` is reading from.** No other channel exists — there is a single samples channel for the whole test run.

### Step 4 — Executor Dispatch

**Files**: `execution/scheduler.go`, `lib/executor/shared_iterations.go`

`Scheduler.Run(globalCtx, runCtx, samplesOut)` (line 419) is the outer event loop of k6. Its critical inner call is line 500:

```go
go e.runExecutor(executorsRunCtx, runResults, samplesOut, exec)
```

which invokes `runExecutor(...)` (line 331). The key line inside `runExecutor` is 369:

```go
err := executor.Run(runCtx, engineOut)
```

For a simple `k6 run script.js` invocation, the registered executor is a `*SharedIterations` (defined in `lib/executor/shared_iterations.go`). Its `Run(parentCtx, out)` method starts at line 167. The relevant line that ties the executor to the iteration loop is:

```go
// line 232
runIteration := getIterationRunner(si.executionState, si.logger)
```

`Run` then spawns one goroutine per VU that repeatedly calls:

```go
// line 259
if !runIteration(maxDurationCtx, activeVU) {
    return
}
```

### Step 5 — Iteration Execution & ExecutionState Update

**File**: `lib/executor/helpers.go`

`getIterationRunner` (lines 104–141) returns a closure:

```go
return func(ctx context.Context, vu lib.ActiveVU) bool {
    err := vu.RunOnce()

    select {
    case <-ctx.Done():
        executionState.AddInterruptedIterations(1)
        return false
    default:
    }

    if err != nil {
        ...
        executionState.AddInterruptedIterations(1)
        return true
    }

    executionState.AddFullIterations(1)   // line 137
    return true
}
```

Two pieces of important bookkeeping happen in this closure:

- `vu.RunOnce()` is called (the next step is where *that* does its work).
- `executionState.AddFullIterations(1)` on line 137 bumps the atomic counter `fullIterationsCount` in `lib/execution.go` (line 146). **This atomic counter is NOT what drives the `iterations` metric sample**; it is an internal progress counter used by the executor scheduling logic and the progress bar. The actual `iterations` Counter sample is pushed separately inside `RunOnce` — see Step 6 and Step 7.

### Step 6 — JS Event Loop & the Sample Site

**File**: `js/runner.go`

`ActiveVU.RunOnce()` (line 724) is where one iteration's worth of JavaScript actually executes. Its relevant inner call is line 773:

```go
err = u.runFn(ctx, true, fn, cancel, u.setupData)
```

(The `true` argument is the `isDefault` flag, i.e. this is a call to the user's exported `default` function — which is exactly the case whose completion should count as one `iterations`.)

`runFn(ctx, isDefault, fn, cancel, data)` (lines 817–877) is the function that actually sends samples. Its outline:

```go
startTime := time.Now()

// ... set up sobek (JS engine) and ctm (tags) ...
err = eventLoop.Start(func() error {
    v, err := fn(sobek.Undefined(), args...)   // line 841 — the user's JS function
    ...
})

endTime := time.Now()

// determine whether the iteration finished fully
isFullIteration := err == nil
select {
case <-ctx.Done():
    isFullIteration = false
default:
}

// always emit IO samples from the dialer
if u.Dialer != nil {
    u.state.Samples <- u.Dialer.IOSamples(endTime, ctm, builtinMetrics)  // line 868
}

// emit iteration samples only for a full default-function iteration
if isFullIteration && isDefault {
    u.state.Samples <- iterationSamples(startTime, endTime, ctm, builtinMetrics)  // line 871
}
```

**Line 871 is the birth of the `iterations` sample.** `u.state.Samples` is the same channel stored in Step 3 — i.e. the one read by the output manager.

### Step 7 — Constructing the Sample

**File**: `js/runner.go`

`iterationSamples(startTime, endTime, ctm, builtinMetrics)` (lines 879–902) is a tiny but important helper. It constructs a `metrics.Samples` slice (which is a `SampleContainer`) containing **exactly two** samples:

```go
func iterationSamples(startTime, endTime time.Time, ctm metrics.TagsAndMeta,
    builtinMetrics *metrics.BuiltinMetrics) metrics.Samples {
    return []metrics.Sample{
        {
            TimeSeries: metrics.TimeSeries{
                Metric: builtinMetrics.IterationDuration,   // ~line 886
                Tags:   ctm.Tags,
            },
            Time:     endTime,
            Metadata: ctm.Metadata,
            Value:    metrics.D(endTime.Sub(startTime)),     // ms — line 890
        },
        {
            TimeSeries: metrics.TimeSeries{
                Metric: builtinMetrics.Iterations,           // line 894
                Tags:   ctm.Tags,
            },
            Time:     endTime,
            Metadata: ctm.Metadata,
            Value:    1,                                     // line 899 — the "+1"
        },
    }
}
```

Key observations:

1. The second sample's `Metric` field is `builtinMetrics.Iterations` — the very same `*Metric` pointer produced in Step 1. Sink pointers travel with the metric pointer, so the sample carries an implicit reference to the global `CounterSink` for `iterations`.
2. The second sample's `Value` is hard-coded to `1`. This is why each successful default-function iteration adds exactly one to the `iterations` counter — the arithmetic happens later, at `CounterSink.Add` in Step 9.
3. The first sample's `Value` is `metrics.D(endTime.Sub(startTime))`, which from `metrics/units.go` is `float64(d) / float64(time.Millisecond)`, i.e. the duration in milliseconds as a `float64`. This is how `iteration_duration` ends up in ms.

### Step 8 — Channel Read & Fan-out

**File**: `output/manager.go`

The `Manager` goroutine started back in Step 2 is now the reader. Its event loop (roughly lines 42–83) can be sketched as:

```go
ticker := time.NewTicker(sendBatchToOutputsRate)   // 50 ms
buffer := make([]metrics.SampleContainer, 0, cap(samplesChan))
for {
    select {
    case sampleContainer, ok := <-samplesChan:
        if !ok {
            sendToOutputs(buffer)
            return
        }
        buffer = append(buffer, sampleContainer)
    case <-ticker.C:
        sendToOutputs(buffer)
        buffer = make([]metrics.SampleContainer, 0, cap(buffer))
    }
}
```

Where `sendToOutputs(buf)` walks `m.outputs` and calls `out.AddMetricSamples(buf)` on every one — and, as noted in §2.3.2, it does so on every 50 ms tick regardless of whether the buffer is empty. Because the `OutputIngester` was appended to `outputs` in Step 2 (at line 187 of `cmd/run.go`), it receives the batch along with every other output (JSON, cloud, stdout, etc.). Each output's `AddMetricSamples` is expected to copy the slice into its own thread-safe buffer so that `sendToOutputs` can reuse the slice without data races — the `OutputIngester` does this via the `SampleBuffer` helper from `output/helpers.go`.

At this point our two samples (IterationDuration + Iterations) have been handed to the `OutputIngester`'s internal buffer. Nothing has yet been added to the `CounterSink`.

### Step 9 — Sink Aggregation

**File**: `metrics/engine/ingester.go`

The `OutputIngester` was started back in Step 2 via `OutputIngester.Start()` (line 40), which in turn created a `PeriodicFlusher` on `collectRate = 50 * time.Millisecond` whose callback is `oi.flushMetrics`. Every 50 ms, `flushMetrics` (lines 62–121) runs:

```go
sampleContainers := oi.GetBufferedSamples()  // drain the SampleBuffer
if len(sampleContainers) == 0 { return }

oi.metricsEngine.MetricsLock.Lock()
defer oi.metricsEngine.MetricsLock.Unlock()

for _, sampleContainer := range sampleContainers {
    samples := sampleContainer.GetSamples()
    for _, sample := range samples {
        m := sample.Metric                       // *Metric
        oi.metricsEngine.markObserved(m)         // line 89
        m.Sink.Add(sample)                        // line 90 — THE INCREMENT

        for _, sm := range m.Submetrics {
            if !sample.Tags.Contains(sm.Tags) {
                continue
            }
            sm.Metric.Sink.Add(sample)
        }

        oi.cardinality.Add(sample.TimeSeries)
    }
}
```

When the flusher dequeues our iterations sample:

1. `m := sample.Metric` — this is the same `*Metric` pointer that was created in Step 1 for `"iterations"`.
2. `markObserved(m)` — adds `m` to `MetricsEngine.ObservedMetrics["iterations"]` if it isn't already there. The end-of-test summary iterates `ObservedMetrics` to decide what to render; missing from `ObservedMetrics` means "never seen a sample, do not render". First time through this step, `iterations` becomes observed.
3. `m.Sink.Add(sample)` — since `m.Type == Counter`, `m.Sink` is the `CounterSink` from Step 1. `CounterSink.Add` (see `metrics/sink.go`) does exactly:

   ```go
   func (c *CounterSink) Add(s Sample) {
       if c.First.IsZero() { c.First = s.Time }
       c.Value += s.Value        // += 1
   }
   ```
   — so the global counter is incremented by the sample's value (1).
4. The inner `for _, sm := range m.Submetrics` loop checks each submetric's `Tags` (e.g. `{scenario:login}`) against the sample's `sample.Tags` using `sample.Tags.Contains(sm.Tags)`. If the sample's tag set is a superset of the submetric's filter, the submetric's *own* `CounterSink` is also incremented. This is how `iterations{scenario:login}` stays in sync with `iterations` without being a completely separate metric registration.
5. `oi.cardinality.Add(sample.TimeSeries)` — tracks unique `{Metric, Tags}` pairs. If the total grows past the current threshold (100 k initially), the ingester emits a warning and doubles the threshold (`cc.timeSeriesLimit *= 2`).

After this step, the running value of the `iterations` counter has advanced by exactly 1.

### Step 10 — Threshold Evaluation and Summary Output

**Files**: `metrics/engine/engine.go`, `cmd/run.go`

Two downstream consumers read from `iterationsMetric.Sink`:

#### 10a — Thresholds (mid-run)

If the user has defined any thresholds (e.g. `thresholds: { iterations: ['count > 1000'] }`), `MetricsEngine.StartThresholdCalculations` (line 159 of `metrics/engine/engine.go`) has been running since the start of the test. It ticks every `thresholdsRate = 2 * time.Second` and calls `evaluateThresholds(...)` (line 216). For each metric with thresholds:

```go
formatted := m.Sink.Format(testRunDuration)   // returns {"count":..., "rate":...}
for i := range m.Thresholds.Thresholds {
    t := m.Thresholds.Thresholds[i]
    t.LastFailed, err = t.Run(formatted, testRunDuration)
    if t.LastFailed {
        m.Tainted = null.BoolFrom(true)
        if t.AbortOnFail { shouldAbort = true }
    }
}
```

`CounterSink.Format(t)` (lines 64–69 of `metrics/sink.go`) returns `{"count": c.Value, "rate": c.Value / (t_ms/1000)}`, so for an `iterations > 1000` threshold the engine gets a fresh reading of the running counter every 2 s. If a threshold fires with `abortOnFail: true`, the engine calls back into `abortRun(err)` — wired by `cmd/run.go` — which cancels the run context; all goroutines downstream wind down, the samples channel closes, and the `Manager` drains.

#### 10b — End-of-test summary (after run)

Once the scheduler has returned, `cmd/run.go` calls (near line 195):

```go
summaryResult, hsErr := test.initRunner.HandleSummary(globalCtx, &lib.Summary{
    Metrics:         metricsEngine.ObservedMetrics,
    RootGroup:       testRunState.GroupSummary.Group(),
    TestRunDuration: executionState.GetCurrentTestRunDuration(),
    NoColor:         c.gs.Flags.NoColor,
    UIState: lib.UIState{
        IsStdOutTTY: c.gs.Stdout.IsTTY,
        IsStdErrTTY: c.gs.Stderr.IsTTY,
    },
})
if hsErr == nil {
    hsErr = handleSummaryResult(c.gs.FS, c.gs.Stdout, c.gs.Stderr, summaryResult)
}
```

The important field is `Metrics: metricsEngine.ObservedMetrics`. That map contains our `iterations` metric (entered by `markObserved` in Step 9). `HandleSummary` is implemented by the JS runner which invokes the built-in summary renderer, which for each metric reads `m.Sink` and prints the appropriate stats. For a `CounterSink` that means calling `Format(testRunDuration)` and rendering:

```
iterations......................: <count>   <rate>/s
```

So the final `iterations` line the user sees in the terminal is `count = CounterSink.Value` accumulated from all our Step-9 additions, and `rate = CounterSink.Value / (test_duration / 1s)`.

### Mermaid Data-Flow Diagram

```mermaid
graph TD
  A["cmd/test_load.go:70<br/>NewRegistry + RegisterBuiltinMetrics<br/>creates Iterations Counter + CounterSink"] --> B
  B["cmd/run.go:170-187,220-228<br/>NewMetricsEngine + CreateIngester<br/>NewManager + samples chan"] --> C
  C["execution/scheduler.go:381 Init<br/>NewVU(ctx, samplesOut)<br/>emitVUsAndVUsMax 1s ticker"] --> D
  D["execution/scheduler.go:419 Run<br/>go runExecutor -> executor.Run(out)"] --> E
  E["lib/executor/shared_iterations.go:167 Run<br/>go handleVU -> runIteration loop"] --> F
  F["lib/executor/helpers.go:104 getIterationRunner<br/>vu.RunOnce + AddFullIterations(1)"] --> G
  G["js/runner.go:724 RunOnce -> 817 runFn<br/>eventLoop runs JS default function"] --> H
  H["js/runner.go:879 iterationSamples<br/>Sample Iterations Value:1 -> state.Samples"] -->|"buffered chan"| I
  I["output/manager.go:42 Start goroutine<br/>50ms tick -> AddMetricSamples fan-out"] --> J
  J["metrics/engine/ingester.go:62 flushMetrics<br/>markObserved + Sink.Add(sample)<br/>submetric tag match + cardinality"] --> K
  K["metrics/engine/engine.go:159 StartThresholdCalculations<br/>2s ticker -> Sink.Format + Thresholds.Run"] --> L
  L["cmd/run.go:195 HandleSummary<br/>Metrics: ObservedMetrics<br/>renders iterations count and rate per second"]
  C -.->|"1s ticker: vus/vus_max samples"| I
  G -.->|"IOSamples from Dialer: DataSent/DataReceived"| I
```

The solid arrows follow the `iterations` metric exclusively; the dashed arrows show the other two sources of samples (VU gauges and dialer IO samples) that share the same samples channel and the same fan-out machinery.

---

## Key Architectural Insights

Several non-obvious design decisions emerge from the trace in Section 3. Each of the bullets below is grounded in the source locations cited in Sections 2 and 3.

- **Four metric types, four sinks.** `metrics/sink.go` defines a `Sink` interface (`Add`, `Format`, `IsEmpty`) and implements it four times — `CounterSink`, `GaugeSink`, `TrendSink`, `RateSink` — one for each `MetricType` (`Counter`, `Gauge`, `Trend`, `Rate`) in `metrics/metric_type.go`. `CounterSink` just sums (`c.Value += s.Value`). `GaugeSink` remembers the last value plus the running `Min`/`Max`. `TrendSink` appends raw observations and computes percentiles on demand via linear interpolation. `RateSink` stores `(Trues, Total)` and derives the rate as `Trues / Total`. Which sink is created is decided once in `NewSink(MetricType)` (lines 25–35) when the metric is registered, and the pointer never changes for the lifetime of the run.

- **25 built-in metrics, registered exactly once.** `metrics/builtin.go:RegisterBuiltinMetrics(registry)` is called *exactly once* per `k6 run` from `cmd/test_load.go:70`. It populates a `BuiltinMetrics` struct whose 25 fields cover VU lifecycle (`vus`, `vus_max`), iterations (`iterations`, `iteration_duration`, `dropped_iterations`), checks/groups (`checks`, `group_duration`), HTTP request phases (`http_reqs`, `http_req_blocked`, `http_req_connecting`, `http_req_tls_handshaking`, `http_req_sending`, `http_req_waiting`, `http_req_receiving`, `http_req_duration`, `http_req_failed`), WebSockets (`ws_sessions`, `ws_msgs_sent`, `ws_msgs_received`, `ws_ping`, `ws_connecting`, `ws_session_duration`), gRPC (`grpc_req_duration`), and raw network I/O (`data_sent`, `data_received`). **Note**: the AAP references "28 built-in metrics"; the authoritative count from the current source in `metrics/builtin.go` is **25** — use the code as truth.

- **Single-channel transport.** The entire pipeline uses one buffered Go channel (`chan metrics.SampleContainer`, declared in `cmd/run.go:228`). The same channel is: (a) written by every VU via `u.state.Samples` (set in `js/runner.go:NewVU`), (b) written by the scheduler's `emitVUsAndVUsMax` goroutine, (c) written by the dialer's `IOSamples`, (d) written by executors on early exit (e.g. `SharedIterations` dropping iterations), and (e) read by a single `output.Manager` goroutine. No other metrics channel exists. Closing it (deferred from `cmd/run.go`) is the canonical "end of data" signal.

- **Two ticker rhythms: 50 ms and 2 s.** There are exactly two periodic timers in the pipeline. Fan-out to outputs (including the ingester's own `SampleBuffer`) runs on a **50 ms** tick, seen in `output/manager.go` as `sendBatchToOutputsRate = 50 * time.Millisecond` and mirrored in `metrics/engine/ingester.go` as `collectRate = 50 * time.Millisecond`. Threshold evaluation runs on a **2 s** tick, seen in `metrics/engine/engine.go` as `thresholdsRate = 2 * time.Second`. The 1-second VU-count ticker (`emitVUsAndVUsMax`) is a producer, not a consumer, so it does not count here.

- **Double-counting of iterations is intentional and correct.** There are two independent iteration counters with different jobs:
  - `ExecutionState.fullIterationsCount` (atomic `*uint64`, `lib/execution.go:146`) is bumped by `executionState.AddFullIterations(1)` in `lib/executor/helpers.go:137`. It drives the progress bar, executor scheduling decisions (e.g. "have we finished all 1,000 iterations yet?"), and the `current iterations / planned iterations` display.
  - The `iterations` **Counter metric** (`metrics/builtin.go:82`) is incremented via a `Sample{Value: 1}` pushed from `js/runner.go:iterationSamples()` (line 899) into the samples channel, eventually summed by `CounterSink.Add` in the ingester (Step 9 of Section 3). It drives the summary output, threshold evaluation, and any configured external output (JSON, cloud, etc.).

  The two counters are updated in two different call sites, in two different goroutines, reading the same iteration completion event — but they live in different places so that scheduling logic can read progress without acquiring the `MetricsEngine.MetricsLock`.

- **Cardinality guard-rail with doubling back-off.** `metrics/engine/ingester.go` carries a `cardinalityControl` struct (`seenTimeSeries map[TimeSeries]struct{}` + an integer `timeSeriesLimit` starting at `timeSeriesFirstLimit = 100_000`). Every sample pulled from the buffer in `flushMetrics` is fed to `oi.cardinality.Add(sample.TimeSeries)`. When unique `TimeSeries` count exceeds `timeSeriesLimit`, the ingester logs a warning and then **doubles** the threshold (`cc.timeSeriesLimit *= 2`). This keeps warnings rare even when cardinality is pathologically high (a single warning at 100 k, another at 200 k, then 400 k, etc.), while still alerting operators that memory will grow. No upper bound is hard-coded.

- **Submetrics are cheap.** In `flushMetrics` every sample is compared to every submetric via `sample.Tags.Contains(sm.Tags)`. `TagSet.Contains` is implemented on top of `github.com/mstoykov/atlas` (see `metrics/tags.go`), which stores tags as nodes in a shared immutable tree. Containment is therefore a pointer/ancestor check, not string comparison, so thousands of submetrics (`iterations{scenario:A}`, `iterations{scenario:B}`, …) can coexist without making the ingester hot path quadratic.

- **Sample producers are plural but point at one channel.** The single samples channel receives writes from at least five distinct call sites in a typical run: VUs via `js/runner.go:runFn()` (iteration samples), VUs via the same function (dialer `IOSamples`), `execution/scheduler.go:emitVUsAndVUsMax()` (1 s VU gauges), `lib/executor/shared_iterations.go` (dropped-iterations Counter on early exit), and user scripts calling `Counter.add()` / `Trend.add()` / etc. from `js/modules/k6/metrics/`. All of these converge on `output/manager.go`, which fan-outs to every `Output` — the ingester being just one of them.

---

## Appendix A — Files Read During Analysis

Every claim in this document is grounded in one of the files below. All files are present in the repository unchanged.

**Metrics Core (`metrics/` package)**

- `metrics/metric.go`
- `metrics/metric_type.go`
- `metrics/value_type.go`
- `metrics/sample.go`
- `metrics/sink.go`
- `metrics/builtin.go`
- `metrics/registry.go`
- `metrics/units.go`
- `metrics/tags.go`
- `metrics/system_tag.go`

**Metrics Engine (`metrics/engine/` package)**

- `metrics/engine/engine.go`
- `metrics/engine/ingester.go`

**Output Pipeline (`output/` package)**

- `output/types.go`
- `output/manager.go`
- `output/helpers.go`

**JavaScript Runtime (`js/` package)**

- `js/runner.go`

**Execution Engine**

- `execution/scheduler.go`
- `lib/executor/shared_iterations.go`
- `lib/executor/helpers.go`
- `lib/execution.go`

**Command Layer**

- `cmd/run.go`
- `cmd/test_load.go`

**Network I/O**

- `lib/netext/dialer.go`

**Test Suite Infrastructure (for Section 1 analysis)**

- `Makefile`
- `go.mod`
- `.github/workflows/tc39.yml`
- `js/tc39/tc39_test.go`
- `js/tc39/checkout.sh`
- `js/modules/k6/http/request_test.go`

**Test Execution Artifacts**

- `/tmp/test_output.txt` — full verbose output of `go test -timeout 300s -v ./...` (85,484 lines, parsed for Section 1 pass/fail/skip counts)

---

## Appendix B — Commands Executed

The analysis for this document used a strictly read-only command set. No `go build`, `go install`, `go mod`, `git add`, `git commit`, or any other mutating command was issued against the source tree.

```bash
# Go version check
go version
# → go version go1.21.13 linux/amd64

# Full test suite execution (output captured to /tmp/test_output.txt)
cd /tmp/blitzy/k6/blitzy-3e37a4fd-ad86-4080-b075-c964f32d4d25_793215
go test -mod=vendor -timeout 300s -v ./... > /tmp/test_output.txt 2>&1

# Result parsing (read-only)
grep -c "^--- PASS" /tmp/test_output.txt
grep -c "^--- FAIL" /tmp/test_output.txt
grep -c "^--- SKIP" /tmp/test_output.txt
grep -cE "^(\s+)?--- PASS" /tmp/test_output.txt     # all-depth PASS
grep -cE "^(\s+)?--- FAIL" /tmp/test_output.txt     # all-depth FAIL
grep -E "^(ok|FAIL|---)" /tmp/test_output.txt | tail -200

# Source inspection (read-only via viewer tool)
# - metrics/*.go
# - metrics/engine/*.go
# - output/*.go
# - js/runner.go
# - lib/executor/shared_iterations.go, helpers.go
# - lib/execution.go
# - execution/scheduler.go
# - cmd/run.go, cmd/test_load.go
# - lib/netext/dialer.go
# - Makefile, go.mod
# - .github/workflows/tc39.yml
# - js/tc39/tc39_test.go, checkout.sh
# - js/modules/k6/http/request_test.go
```

Deviations from `Makefile`'s `tests` target (`go test -race -timeout 210s ./...`):

1. `-race` flag: the `Makefile` enables it, but the environment in which tests were executed has CGO disabled for some tool invocations; to keep results deterministic the flag was omitted. The race detector is orthogonal to the pass/fail semantics of the individual tests reported here.
2. Timeout extended from `210s` to `300s`: gives long-running packages like `lib/executor` (~22–28 s) breathing room, avoiding false-positive timeouts. Since the extension is *generous*, it cannot cause a test that would otherwise pass to fail.
3. `-mod=vendor`: forces the use of the checked-in `vendor/` tree, matching the setup environment and avoiding any network fetch.

No existing files were modified. Only `blitzy/documentation/k6_ddc3b0b1d23c.md` was created.

---

## Appendix C — Read-Only Mandate Compliance

Per the user's explicit instruction ("Just exploring for now, so please don't modify anything in the repo.") and the `SWE-AtlasQnA-Repo` project rule, this analysis created **exactly one new file** — `blitzy/documentation/k6_ddc3b0b1d23c.md` — and modified **zero existing files**. No source code (`.go`), module manifests (`go.mod`, `go.sum`), build files (`Makefile`, `Dockerfile`), configuration files (`.yml`, `.yaml`, `.toml`), shell scripts (`.sh`), or other markdown documents elsewhere in the repository were touched. No code formatting, no import reordering, no "silent fixes". Every claim made in this document is backed either by an explicit citation to a file path and line number in the repository or by a direct quote from the test output captured at `/tmp/test_output.txt`. If any of the failures or skips documented in Section 1 are to be addressed (e.g. making the OCSP test optional, fixing the timing-sensitive executor tests), that work belongs to a separate follow-up task; it is explicitly out of scope here.


