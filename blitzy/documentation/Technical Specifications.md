# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

Based on the prompt, the Blitzy platform understands that the new feature requirement is to conduct a deep investigative analysis of the internal orchestration of Grafana k6 (v0.55.0, commit `ddc3b0b1d2`, Go 1.21.13), specifically answering a set of behavioral questions through runtime evidence obtained by executing temporary test scripts against the unmodified source repository. The findings must be documented in a new markdown file placed at `blitzy/documentation/<source_branch_name>.md`.

### 0.1.1 Core Feature Objective

The Blitzy platform understands the following discrete investigation requirements:

- **VU Management under SIGINT (Ramping Executor):** Determine the exact log messages emitted when a `ramping-vus` executor configured with at least 5 VUs receives a SIGINT signal. Provide specific log evidence showing whether active VUs are allowed to finish their current iteration or are terminated mid-execution during shutdown.
- **gRPC Server Streaming Interruption:** Capture the exact log entries produced at runtime when a gRPC server streaming test configured with a `30ms` graceful ramp down is interrupted. Report the specific value of `grpc_streams_msgs_received` from the final metrics summary.
- **Dropped Iterations via API:** Determine the exact value of `dropped_iterations` reported when a test exceeds its maximum duration capacity. Provide runtime evidence that this value was obtained by querying the k6 REST API (`/v1/metrics/dropped_iterations`).
- **SharedArray Data Sharing Behavior:** Investigate whether the memory footprint for files loaded via `SharedArray` remains constant with increasing VUs, or whether each VU creates its own copy. Provide test script output as proof of the behavior and identify the root cause in the source code.
- **Prometheus Output Metric Name Integrity:** Investigate metric reporting behavior when using the `experimental-prometheus-rw` output. Provide test script output proving that exported data maintains metric name integrity (specifically the `k6_` prefix convention).

### 0.1.2 Special Instructions and Constraints

- **Read-Only Repository Constraint:** The user explicitly requires: "Don't modify any repository source files." Temporary scripts and artifacts may be created to observe behavior but must be cleaned up afterward, leaving the codebase unchanged.
- **Implementation Rule (SWE-AtlasQnA-Repo):** A new markdown document named `<source_branch_name>.md` must be created in `blitzy/documentation/` that comprehensively answers all questions with thinking/rationale. No assumptions — answers must be grounded in the code. No modifications to existing repository files. No other code besides the requested document.
- **Runtime Evidence Requirement:** Every behavioral claim must be backed by actual test script output, log messages, or API responses captured during k6 execution.
- **No Assumptions:** All answers must be derived from the actual source code and observed runtime behavior of k6 v0.55.0.

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To **investigate VU management under SIGINT**, we will execute a k6 test script with `ramping-vus` executor (5+ VUs) and send `SIGINT` mid-execution, capturing verbose log output (`-v --log-output=stdout --log-format=raw`) to observe the exact sequence of signal handling, VU state transitions, and iteration completion/interruption behavior. The key source files governing this behavior are `cmd/run.go` (signal trap setup), `cmd/common.go` (`handleTestAbortSignals`), `lib/executor/ramping_vus.go` (executor logic), `lib/executor/vu_handle.go` (VU state machine with `gracefulStop`/`hardStop`), and `lib/executor/helpers.go` (`getIterationRunner` which determines full vs. interrupted iteration accounting).
- To **investigate gRPC server streaming interruption**, we will create a standalone gRPC server (using the repository's own `lib/testutils/grpcservice` proto definitions) implementing the `ListFeatures` server-streaming RPC, then run a k6 test with `gracefulRampDown: '30ms'` and interrupt via SIGINT. The gRPC metrics are defined in `js/modules/k6/grpc/metrics.go` (`grpc_streams`, `grpc_streams_msgs_sent`, `grpc_streams_msgs_received`).
- To **capture dropped iterations via API**, we will use a `constant-arrival-rate` executor intentionally configured with insufficient VUs for the target rate, enabling the k6 REST API (`--address=localhost:6565`), and query `GET /v1/metrics/dropped_iterations` mid-test. The API implementation is in `api/v1/metric_routes.go` and the dropped iterations metric is defined in `metrics/builtin.go`.
- To **investigate SharedArray memory behavior**, we will run two comparative tests: one using `SharedArray` from `k6/data` and another using `open()` + `JSON.parse()`, measuring RSS memory at runtime with `ps`. The root cause is in `js/modules/k6/data/data.go` where the `RootModule` contains a single `sharedArrays` map shared across all VU instances via the `NewModuleInstance` method.
- To **verify Prometheus metric name integrity**, we will run k6 with `-o experimental-prometheus-rw` pointed at a local HTTP receiver, capturing the remote write payloads and verifying that metric names are correctly prefixed with `k6_` as defined in `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/config.go` (`defaultMetricPrefix = "k6_"`) and `prometheus.go` (`MapSeries` function).


## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

The investigation spans the following areas of the k6 repository. Since no source files are modified, all listed files are **read-only analysis targets** used to derive answers and design temporary test scripts.

**Signal Handling and Test Lifecycle:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `cmd/run.go` | Main `k6 run` command lifecycle | Signal trap setup at lines 347–364; gracefulStop/onHardStop closures; runAbort mechanism |
| `cmd/common.go` | Shared CLI helpers | `handleTestAbortSignals` at line 97: traps SIGINT/SIGTERM, dispatches graceful then hard stop |
| `cmd/state/` | Global state management | Process-state and environment parsing utilities |

**Executor Subsystem (VU Management):**

| File | Purpose | Relevance |
|------|---------|-----------|
| `lib/executor/ramping_vus.go` | Ramping VUs executor | `Run()` method; VU handle creation; graceful ramp-down scheduling; `gracefulStop`/`hardStop` strategies |
| `lib/executor/vu_handle.go` | VU lifecycle state machine | States: `stopped`, `starting`, `running`, `toGracefulStop`, `toHardStop`; `runLoopsIfPossible` main loop |
| `lib/executor/helpers.go` | Executor helper functions | `getIterationRunner` (lines 104–141): determines full vs. interrupted iteration accounting; `getDurationContexts` |
| `lib/executor/constant_arrival_rate.go` | Constant arrival rate executor | `droppedIterationMetric` emission at line 324 when VUs are insufficient |
| `lib/executor/shared_iterations.go` | Shared iterations executor | Dropped iteration counting at end of execution |
| `lib/executor/base_config.go` | Base executor configuration | Default `gracefulStop` (30s) and `gracefulRampDown` configuration |

**Scheduler and Execution State:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `execution/scheduler.go` | Core execution scheduler | `Run()` orchestrates executors, handles interruption at line 426; `getRunStats` reports iteration counts |
| `execution/abort.go` | Cancellation plumbing | `NewTestRunContext`, `AbortTestRun`, `GetCancelReasonIfTestAborted` |
| `lib/execution.go` | Execution state container | `AddFullIterations`, `AddInterruptedIterations`, VU allocation/return |

**gRPC Module:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `js/modules/k6/grpc/grpc.go` | gRPC module entry point | `Stream` constructor export; stream initialization |
| `js/modules/k6/grpc/stream.go` | gRPC stream implementation | Server-streaming message handling; event listeners (`data`, `end`, `error`) |
| `js/modules/k6/grpc/metrics.go` | gRPC metrics registration | Defines `grpc_streams`, `grpc_streams_msgs_sent`, `grpc_streams_msgs_received` |
| `js/modules/k6/grpc/client.go` | gRPC client | `Connect` with reflection; `Load` for proto files |
| `lib/testutils/grpcservice/route_guide.proto` | Test proto definition | `FeatureExplorer` service with `ListFeatures` server-streaming RPC |
| `lib/testutils/grpcservice/route_guide_grpc.pb.go` | Generated gRPC code | Server and client interfaces for the route guide service |

**Data Sharing (SharedArray):**

| File | Purpose | Relevance |
|------|---------|-----------|
| `js/modules/k6/data/data.go` | SharedArray module | `RootModule` with single `sharedArrays` map; double-checked locking in `get()`; `NewModuleInstance` shares pointer |
| `js/modules/k6/data/share.go` | SharedArray internal types | `sharedArray` struct (holds `[]string`); `wrappedSharedArray` with on-access JSON parse; `deepFreeze` |
| `js/initcontext.go` | Init-stage file helpers | `openImpl` for `open()` file loading — each VU parses independently |

**API Endpoints:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `api/server.go` | API server assembly | Mounts `/v1/` routes, `/ping`, `/debug/pprof/`; address binding |
| `api/v1/routes.go` | Versioned route handler | Routes for `/v1/status`, `/v1/metrics`, `/v1/metrics/{id}`, `/v1/groups` |
| `api/v1/metric_routes.go` | Metric read handlers | `handleGetMetrics` and `handleGetMetric` — locks `MetricsEngine.ObservedMetrics` |
| `api/v1/metric.go` | Metric API model | `NewMetric` converts internal `*metrics.Metric` to API JSON model |
| `api/v1/metric_jsonapi.go` | JSON:API envelopes | `newMetricsJSONAPI` wraps collection for API response |
| `api/v1/control_surface.go` | Control surface struct | Bundles context, samples channel, metrics engine, scheduler |

**Prometheus Remote Write Output:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `vendor/.../remotewrite/remotewrite.go` | Prometheus output core | `Output` struct; `AddMetricSamples` processing; `MapPrompb` for each metric type |
| `vendor/.../remotewrite/prometheus.go` | Label/series mapping | `MapSeries`: applies `defaultMetricPrefix` + metric name; sorts labels lexicographically |
| `vendor/.../remotewrite/config.go` | Configuration constants | `defaultMetricPrefix = "k6_"` at line 24 |
| `vendor/.../remotewrite/trend.go` | Trend stat mapping | `TrendGauges.MapPrompb`: appends stat suffix (e.g., `_p99`, `_avg`) to `__name__` label |

**Metrics Framework:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `metrics/builtin.go` | Built-in metric definitions | `DroppedIterationsName = "dropped_iterations"` at line 10; registration at line 84 |
| `metrics/sample.go` | Sample types | `TimeSeries`, `Sample`, `SampleContainer`, `PushIfNotDone` |
| `metrics/sink.go` | Aggregation sinks | `CounterSink`, `GaugeSink`, `TrendSink`, `RateSink` |
| `metrics/registry.go` | Metric registry | `NewMetric`, deduplication, name validation |

### 0.2.2 New File Requirements

- **CREATE:** `blitzy/documentation/<source_branch_name>.md` — The comprehensive markdown document answering all investigation questions with runtime evidence, rationale, and source code references.

No other files are to be created or modified in the repository. All temporary scripts and artifacts used during investigation are created outside the repository and cleaned up after use.


## 0.3 Dependency Inventory

### 0.3.1 Key Packages Relevant to Investigation

| Registry | Package | Version | Purpose |
|----------|---------|---------|---------|
| Go modules | `go.k6.io/k6` | v0.55.0 | Core k6 load testing framework (the repository itself) |
| Go modules | `go` (toolchain) | 1.21.13 | Go language runtime specified in `go.mod` |
| Go modules | `github.com/grafana/xk6-output-prometheus-remote` | v0.5.0 | Prometheus remote write output extension |
| Go modules | `github.com/sirupsen/logrus` | (vendored) | Structured logging framework used throughout k6 |
| Go modules | `github.com/spf13/cobra` | (vendored) | CLI command framework for `k6 run`, `k6 cloud`, etc. |
| Go modules | `github.com/grafana/sobek` | (vendored) | JavaScript runtime engine (Sobek, fork of goja) |
| Go modules | `github.com/jhump/protoreflect` | v1.17.0 | gRPC reflection and proto file parsing |
| Go modules | `github.com/grpc-ecosystem/go-grpc-middleware` | v1.4.0 | gRPC middleware support |
| Go modules | `google.golang.org/grpc` | (vendored) | gRPC Go implementation |
| Go modules | `buf.build/gen/go/prometheus/prometheus/protocolbuffers/go` | v1.31.0 | Prometheus protobuf definitions for remote write |
| Go modules | `github.com/prometheus/client_golang` | v1.16.0 | Prometheus client library |
| Go modules | `github.com/mstoykov/atlas` | (vendored) | Immutable tag set implementation used by metrics |
| Go modules | `gopkg.in/guregu/null.v3` | (vendored) | Nullable types for configuration fields |
| Go modules | `github.com/mstoykov/k6-taskqueue-lib/taskqueue` | (vendored) | Task queue for async gRPC stream event dispatch |

### 0.3.2 Dependency Notes

- All dependencies are vendored in the `vendor/` directory for reproducible builds
- No dependency additions or modifications are needed since no source files are modified
- The Prometheus remote write output is integrated as `experimental-prometheus-rw` via the `grafana/xk6-output-prometheus-remote` package at v0.5.0
- The gRPC module uses `protoreflect` v1.17.0 for runtime proto reflection, enabling `{ reflect: true }` in the `client.connect()` call


## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

Since this is a read-only investigation with a single documentation artifact as output, the integration points below describe the **code paths traversed during runtime observation** rather than modification targets.

**Signal Handling Chain (SIGINT → VU Shutdown):**

- `cmd/common.go:97` — `handleTestAbortSignals()` registers signal listeners on `os.Interrupt`, `syscall.SIGINT`, `syscall.SIGTERM`
- `cmd/run.go:349` — `gracefulStop` closure calls `runAbort()` with `errext.AbortedByUser` reason
- `execution/abort.go` — `AbortTestRun()` cancels the `runCtx` context, propagating cancellation to all child contexts
- `execution/scheduler.go:426` — `Run()` detects interrupt via `GetCancelReasonIfTestAborted(runCtx)`, sets `ExecutionStatusInterrupted`
- `lib/executor/ramping_vus.go:543` — `iterateSteps()` exits via context-aware `waiter()` when executor context is cancelled
- `lib/executor/vu_handle.go:147` — `gracefulStop()` transitions VU state from `running` → `toGracefulStop`
- `lib/executor/vu_handle.go:165` — `hardStop()` transitions state to `toHardStop` and cancels the VU's context
- `lib/executor/vu_handle.go:185` — `runLoopsIfPossible()` main loop detects cancelled executor context and exits

**gRPC Stream Lifecycle:**

- `js/modules/k6/grpc/grpc.go:99` — `stream()` constructor validates client, method, and parameters
- `js/modules/k6/grpc/stream.go:40` — `stream` struct holds VU reference, method descriptor, event listeners, and task queue
- `js/modules/k6/grpc/metrics.go:17` — `grpc_streams` counter incremented per stream open
- `js/modules/k6/grpc/metrics.go:25` — `grpc_streams_msgs_received` counter incremented per message received

**Dropped Iterations Emission:**

- `lib/executor/constant_arrival_rate.go:324` — `droppedIterationMetric` obtained from `BuiltinMetrics.DroppedIterations`
- `lib/executor/constant_arrival_rate.go:339-346` — Pushes `metrics.Sample{Value: 1}` for each dropped iteration via `metrics.PushIfNotDone`
- `metrics/builtin.go:84` — `DroppedIterations` registered as `Counter` type with name `dropped_iterations`

**API Metrics Exposure:**

- `api/server.go` — `GetServer()` constructs the HTTP server, mounts `/v1/` routes
- `api/v1/routes.go` — Wires `GET /v1/metrics` and `GET /v1/metrics/{id}` to handlers
- `api/v1/metric_routes.go:9` — `handleGetMetrics` locks `MetricsEngine.MetricsLock`, snapshots `ObservedMetrics`, marshals to JSON:API
- `api/v1/metric_routes.go:27` — `handleGetMetric` retrieves single metric by ID from `ObservedMetrics` map

**SharedArray Data Path:**

- `js/modules/k6/data/data.go:43` — `RootModule.New()` creates single `sharedArrays{data: make(map[string]sharedArray)}`
- `js/modules/k6/data/data.go:53` — `NewModuleInstance()` returns `Data{shared: &rm.shared}` — all VUs share the same pointer
- `js/modules/k6/data/data.go:152` — `sharedArrays.get()` uses double-checked locking (RLock → Lock) to ensure single initialization
- `js/modules/k6/data/share.go:23` — `sharedArray.wrap()` creates a `wrappedSharedArray` with per-VU runtime references but shared underlying `[]string` data

**Prometheus Metric Name Pipeline:**

- `vendor/.../remotewrite/config.go:24` — `defaultMetricPrefix = "k6_"`
- `vendor/.../remotewrite/prometheus.go:39` — `MapSeries()` constructs `__name__` label as `k6_` + metric name + optional suffix
- `vendor/.../remotewrite/remotewrite.go:331` — Counter type appends `_total` suffix
- `vendor/.../remotewrite/remotewrite.go:341` — Rate type appends `_rate` suffix
- `vendor/.../remotewrite/trend.go:89` — Trend type appends stat suffix (e.g., `_p99`, `_avg`, `_min`, `_max`, `_med`)


## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

Since the implementation rule requires creating a single markdown document with no modifications to existing files, the execution plan consists of one file creation and supporting runtime experiments.

**Group 1 — Documentation Artifact (Single Deliverable):**

- **CREATE:** `blitzy/documentation/<source_branch_name>.md` — Comprehensive answers document containing all runtime evidence, source code analysis, and explanations for each investigation question

**Group 2 — Temporary Runtime Experiments (Created, Executed, Cleaned Up):**

The following temporary scripts are needed during the investigation phase. All are created outside the repository (in `/tmp/`) and removed after execution.

- Temporary script for ramping-vus SIGINT experiment — 5 VUs with `sleep(2)` iterations, SIGINT sent after 6 seconds
- Temporary Go server for gRPC streaming — implements `ListFeatures` server-streaming RPC using the repository's `route_guide.proto`
- Temporary script for gRPC stream interruption — `ramping-vus` with `gracefulRampDown: '30ms'`, connects via reflection
- Temporary script for dropped iterations — `constant-arrival-rate` with `rate: 10/s`, `maxVUs: 1`, `duration: 5s`
- Temporary script for SharedArray memory comparison — 50,000-item array, 5 VUs, RSS measurement via `ps`
- Temporary script for open() memory comparison — same dataset loaded via `open()` + `JSON.parse()`, 5 VUs
- Temporary Python HTTP server for Prometheus payload capture — receives remote write requests on port 9090
- Temporary script for Prometheus metric name test — custom Counter, Trend, Gauge, Rate metrics with `-o experimental-prometheus-rw`

### 0.5.2 Implementation Approach

**Environment Setup:**
- Install Go 1.21.13 (matching the `toolchain go1.21.13` in `go.mod`)
- Build k6 binary from source: `go build -o /tmp/k6 .`
- Verify: `/tmp/k6 version` → `k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)`

**Investigation Execution Sequence:**

- **Experiment 1 (SIGINT/VU Behavior):** Run k6 with `--log-output=stdout --log-format=raw -v`, send `kill -INT` after VUs stabilize. Capture full log output showing signal handling, VU state transitions, and iteration completion/interruption status.
- **Experiment 2 (gRPC Streaming):** Build standalone gRPC server from repository proto, run k6 with `reflect: true` and `gracefulRampDown: '30ms'`, interrupt with SIGINT. Capture stream message counts and final metrics summary.
- **Experiment 3 (Dropped Iterations):** Run k6 with `--address=localhost:6565`, use `curl` to query `GET /v1/metrics/dropped_iterations` during execution. Capture both API JSON response and final summary.
- **Experiment 4 (SharedArray Memory):** Run two back-to-back tests (SharedArray vs open()), measure RSS via `ps -o rss` at the same point during execution, compare values.
- **Experiment 5 (Prometheus Names):** Run k6 with `-o experimental-prometheus-rw` pointed at local receiver, capture raw payloads, extract `k6_`-prefixed metric names.

**Documentation Assembly:**
- Compile all experimental results with exact log outputs, API responses, and memory measurements
- Provide source code references for each behavioral explanation
- Structure as a comprehensive Q&A markdown document

### 0.5.3 Key Runtime Evidence Summary

The following runtime evidence was captured during the investigation:

**SIGINT Handling — Key Log Sequence:**
```
Stopping k6 in response to signal...
```
Followed by VU iteration completions, then: `"The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'"`. Final summary reports both complete and interrupted iterations (e.g., `8 complete and 5 interrupted iterations`).

**gRPC Streaming — Final Metrics:**
```
grpc_streams_msgs_received...: 80     16.289763/s
```
The value of `grpc_streams_msgs_received` is **80** (10 messages per stream × 8 streams completed).

**Dropped Iterations — API Response:**
```json
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":37,"rate":9.34}}}}
```
The exact value of `dropped_iterations` queried via API was **37** at query time (final summary reported **47** after test completion).

**SharedArray Memory:**
- SharedArray (5 VUs): **119,168 KB** RSS
- open()+JSON.parse (5 VUs): **345,140 KB** RSS
- Ratio: open() uses approximately **2.9× more memory** than SharedArray

**Prometheus Metric Names:**
Captured payloads contain correctly prefixed names: `k6_my_custom_counter_total`, `k6_my_success_rate`, `k6_iteration_duration_p99`, `k6_data_sent_total`, `k6_vus_max`.


## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

**Documentation Artifact:**
- `blitzy/documentation/<source_branch_name>.md` — The sole deliverable file

**Source Code Analysis Targets (Read-Only):**
- `cmd/run.go`, `cmd/common.go` — Signal handling and test lifecycle
- `execution/scheduler.go`, `execution/abort.go` — Scheduler orchestration and abort plumbing
- `lib/executor/ramping_vus.go` — Ramping VUs executor implementation
- `lib/executor/vu_handle.go` — VU lifecycle state machine (graceful/hard stop)
- `lib/executor/helpers.go` — Iteration runner, duration contexts
- `lib/executor/constant_arrival_rate.go` — Dropped iteration emission during arrival-rate execution
- `lib/executor/shared_iterations.go` — Dropped iteration emission for shared-iterations executor
- `lib/executor/base_config.go` — Default graceful stop/ramp-down values
- `js/modules/k6/grpc/**/*.go` — gRPC module (stream, client, metrics)
- `js/modules/k6/data/data.go`, `js/modules/k6/data/share.go` — SharedArray implementation
- `js/initcontext.go` — `open()` file loading in init context
- `api/server.go`, `api/v1/**/*.go` — REST API server and v1 metric routes
- `vendor/.../xk6-output-prometheus-remote/pkg/remotewrite/**/*.go` — Prometheus remote write output
- `metrics/builtin.go` — Built-in metric definitions (dropped_iterations, etc.)
- `metrics/sample.go`, `metrics/sink.go`, `metrics/registry.go` — Metrics framework
- `lib/testutils/grpcservice/route_guide.proto` — gRPC test service proto definition
- `go.mod` — Dependency manifest and Go toolchain version

**Runtime Experiments (Temporary, Cleaned Up):**
- Ramping-VUs + SIGINT test script
- gRPC server binary and streaming test script
- Constant-arrival-rate dropped iterations test script with API query
- SharedArray vs open() memory comparison test scripts
- Prometheus remote write capture server and test script

### 0.6.2 Explicitly Out of Scope

- Modification of any existing repository source files
- Addition of permanent code files to the repository (only the markdown document in `blitzy/documentation/`)
- Performance optimization of any k6 component
- Bug fixes or feature additions to k6 internals
- Cloud execution, distributed execution, or xk6-browser behaviors
- Output backends other than Prometheus remote write (InfluxDB, CSV, JSON are not investigated)
- Threshold evaluation or abort-on-fail behaviors
- Test archive serialization/deserialization
- k6 cloud API integration
- CI/CD pipeline modifications
- Docker or container-based test execution


## 0.7 Rules for Feature Addition

### 0.7.1 Repository Integrity Constraint

The user explicitly mandates:

> "Don't modify any repository source files. If you need to create temporary scripts or artifacts to observe behavior, that's fine, but clean them up afterward and leave the codebase unchanged."

This is the governing rule for the entire investigation. All runtime experiments must use temporary scripts created outside the source tree (or in ephemeral locations), and the repository must pass a `git status` verification showing zero modified, added, or deleted files upon completion.

### 0.7.2 Evidence-Based Answers

- Every answer must include **exact log output** or **exact API response data** captured at runtime
- No speculative or inferred behavior is acceptable; all conclusions must be substantiated by observed output from the k6 binary built from this repository
- Memory measurements must use a consistent methodology (RSS via `/proc/[pid]/status`)
- API queries must return full JSON payloads showing the metric structure and values

### 0.7.3 Experiment Design Conventions

- Each experiment must be self-contained with a reproducible test script and clear execution steps
- gRPC experiments require a companion server binary built from the repository's own proto definitions (`lib/testutils/grpcservice/route_guide.proto`)
- Signal delivery timing must allow the system to reach the target VU count and establish steady-state before interruption
- Prometheus experiments must capture the raw remote write payload to verify metric name construction at the protocol level

### 0.7.4 Output Artifact Convention

Per the project's implementation rules (`SWE-AtlasQnA-Repo`):

- Create a single markdown document named `<source_branch_name>.md` in the `blitzy/documentation` directory
- The document must comprehensively answer all posed questions with rationale
- Base all answers on the source code as ground truth
- Do not add any other code files to the source repository


## 0.8 References

### 0.8.1 Repository Files Searched and Analyzed

The following source files were retrieved and analyzed to derive the conclusions in this Agent Action Plan:

| Area | File Path | Purpose |
|------|-----------|---------|
| Signal Handling | `cmd/run.go` | Test lifecycle orchestration, graceful stop closure, signal trap registration |
| Signal Handling | `cmd/common.go` | `handleTestAbortSignals()` — SIGINT/SIGTERM channel trap, graceful vs hard stop |
| Signal Handling | `execution/abort.go` | `AbortReason`, `AbortedByUser`, abort signaling primitives |
| Scheduler | `execution/scheduler.go` | `Run()` — executor launch, `getRunStats()`, interrupt classification |
| Scheduler | `execution/controller.go` | Controller interface for scheduler coordination |
| Executor Framework | `lib/executor/ramping_vus.go` | RampingVUs executor: stages, gracefulRampDown, VU scheduling |
| Executor Framework | `lib/executor/vu_handle.go` | VU state machine: 5 states, gracefulStop(), hardStop(), runLoopsIfPossible() |
| Executor Framework | `lib/executor/helpers.go` | `getIterationRunner()` — full vs interrupted classification, `getDurationContexts()` |
| Executor Framework | `lib/executor/constant_arrival_rate.go` | `Run()` — dropped iteration emission on TryRunIteration failure |
| Executor Framework | `lib/executor/shared_iterations.go` | Dropped iteration counting at executor exit |
| Executor Framework | `lib/executor/base_config.go` | Default `GracefulStop` (30s), `GracefulRampDown` (30s) |
| gRPC Module | `js/modules/k6/grpc/stream.go` | Stream struct, message send/receive, event listeners |
| gRPC Module | `js/modules/k6/grpc/client.go` | gRPC client initialization, reflection, connection |
| gRPC Module | `js/modules/k6/grpc/metrics.go` | `grpc_streams`, `grpc_streams_msgs_sent`, `grpc_streams_msgs_received` registration |
| gRPC Module | `js/modules/k6/grpc/params.go` | Connection parameters, TLS config |
| Data Sharing | `js/modules/k6/data/data.go` | `RootModule` singleton, `sharedArrays` pointer sharing via `NewModuleInstance` |
| Data Sharing | `js/modules/k6/data/share.go` | `sharedArray` ([]string), `wrappedSharedArray.Get()` — JSON.parse + deepFreeze |
| Data Sharing | `js/initcontext.go` | `open()` function — per-VU file loading in init context |
| API Endpoints | `api/server.go` | HTTP server assembly, CORS, `newHandler()` |
| API Endpoints | `api/v1/routes.go` | `/v1/` route registration |
| API Endpoints | `api/v1/metric_routes.go` | `handleGetMetrics()`, `handleGetMetric()` — JSON:API response format |
| API Endpoints | `api/v1/control_surface.go` | Pause/resume API handlers |
| API Endpoints | `api/v1/status_routes.go` | Status API handlers |
| Prometheus Output | `vendor/.../remotewrite/remotewrite.go` | `Output.AddMetricSamples()`, `MapSeries()`, suffix logic (Counter→_total, Rate→_rate) |
| Prometheus Output | `vendor/.../remotewrite/config.go` | `defaultMetricPrefix = "k6_"`, trend stats config |
| Prometheus Output | `vendor/.../remotewrite/prometheus.go` | `MapSeries()` — `__name__` construction: prefix + name + suffix |
| Prometheus Output | `vendor/.../remotewrite/trend.go` | Trend stat suffix appending (_p99, _avg, _min, _max, _med) |
| Metrics Framework | `metrics/builtin.go` | Built-in metric definitions: `Iterations`, `DroppedIterations`, `VUs`, etc. |
| Metrics Framework | `metrics/sample.go` | `Sample`, `SampleContainer` types |
| Metrics Framework | `metrics/sink.go` | Counter/Gauge/Rate/Trend sink implementations |
| Metrics Framework | `metrics/registry.go` | `Registry` for metric registration |
| gRPC Test Service | `lib/testutils/grpcservice/route_guide.proto` | Proto definition used to build companion gRPC server |
| Dependency Manifest | `go.mod` | Go module path, Go version, direct/indirect dependencies |

### 0.8.2 Folders Explored

| Folder Path | Contents Summary |
|-------------|-----------------|
| `/` (root) | Top-level k6 repository: cmd, lib, js, api, execution, metrics, output, vendor |
| `cmd/` | CLI entry points: run.go, common.go, root.go, cloud.go, state.go |
| `execution/` | Scheduler, abort, controller — runtime orchestration |
| `lib/` | Core types: options, runner, executors, execution state |
| `lib/executor/` | All executor implementations and VU handle state machine |
| `js/` | JavaScript runtime: modules, bundle, compiler |
| `js/modules/k6/grpc/` | gRPC module: client, stream, params, metrics |
| `js/modules/k6/data/` | Data module: SharedArray implementation |
| `api/` | HTTP API server |
| `api/v1/` | Versioned REST endpoints: metrics, status, control surface |
| `output/` | Output manager, helpers, cloud, extensions |
| `metrics/` | Metric types, sinks, registry, thresholds, builtin definitions |
| `vendor/.../xk6-output-prometheus-remote/pkg/remotewrite/` | Prometheus remote write output plugin |

### 0.8.3 Attachments

No attachments were provided for this project.

### 0.8.4 External URLs

No Figma URLs or other external resource URLs were specified in the user's requirements.


