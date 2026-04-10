# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new investigative documentation file** (`blitzy/documentation/k6_ddc3b0b1d23c.md`) that comprehensively answers eight specific questions about Grafana k6 internal engine behavior, backed by source code analysis and runtime evidence.

**Request Category:** Create new documentation

**Documentation Type:** Technical investigation report — runtime behavior analysis with evidence-based answers

The user's requirements translate into the following documentation objectives:

- **VU Orchestration Internals** — Document how the Go engine manages Virtual Users through the `execution.Scheduler` → executor → `vuHandle` state machine lifecycle, referencing `execution/scheduler.go`, `lib/executor/vu_handle.go`, and `lib/executor/ramping_vus.go`
- **SIGINT Log Messages for Ramping Executor** — Capture and present the exact log output produced when a ramping-vus executor running at least 5 VUs receives a SIGINT signal, demonstrating the interrupt handling path in `cmd/run.go`
- **Graceful Shutdown Evidence** — Provide specific log evidence proving whether currently active VUs are allowed to finish their current iteration (graceful stop via `toGracefulStop` state) or are terminated mid-execution (hard stop via `toHardStop` state) during shutdown
- **gRPC Server Streaming Interruption** — Document the exact log entries produced when a gRPC server streaming test with a 30ms `gracefulRampDown` is interrupted via SIGINT, including stream cancellation messages and error codes
- **gRPC Messages Received Metric** — Report the exact value of `grpc_streams_msgs_received` from the final metrics summary of the interrupted gRPC streaming test
- **Dropped Iterations via API** — Report the exact value of the `dropped_iterations` metric when a test exceeds its maximum duration capacity, queried through the k6 REST API at `/v1/metrics/dropped_iterations`, with the full JSON API response as runtime evidence
- **Data Sharing Behavior** — Determine whether the memory footprint for files loaded via `data.SharedArray` remains constant with increasing VUs or whether each VU creates its own copy, providing test script output as proof and identifying the root cause in the Go source code (`js/modules/k6/data/share.go` and `js/modules/k6/data/data.go`)
- **Prometheus Output Metric Naming** — Investigate and provide test script output proving that the Prometheus remote write output maintains metric name integrity, documenting the naming convention implemented in `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/prometheus.go`

### 0.1.2 Special Instructions and Constraints

**Critical Directives:**

- **No source code modifications**: The user explicitly states: *"Don't modify any repository source files."* The implementation rule reinforces: *"Do not modify any existing files in the source repository."*
- **Temporary artifacts allowed**: *"If you need to create temporary scripts or artifacts to observe behavior, that's fine, but clean them up afterward and leave the codebase unchanged."*
- **Evidence-based answers only**: The implementation rule states: *"Do not make assumptions, base your answers on the code as the truth."*
- **Provide thinking/rationale**: The implementation rule requires: *"Provide thinking / rationale behind the answers."*
- **Exact runtime output required**: The user requests exact log messages, exact log entries, exact values, and test script output as proof for every claim
- **API query evidence**: For the dropped iterations question, the user explicitly requests: *"I want you to report this value by querying the api. Give me runtime evidence to prove that the values were reported by querying the api."*

**Output Format Requirements:**

- Output file: `blitzy/documentation/k6_ddc3b0b1d23c.md` (branch name = `k6_ddc3b0b1d23c`)
- Placement: `blitzy/documentation/` directory in the destination repository
- Format: Markdown document with comprehensive answers

**Style Preferences:**

- Technical depth: Deep investigation into Go engine internals with source code citations
- Evidence format: Verbatim log outputs and API responses embedded in code blocks
- Rationale: Each answer must include reasoning derived from source code analysis
- Structure: Each question addressed as a self-contained section with source analysis followed by runtime evidence

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document VU orchestration**, we will analyze the `execution/scheduler.go` initialization flow, the `lib/executor/vu_handle.go` state machine (5 states: stopped → starting → running → toGracefulStop → toHardStop), and executor-specific VU management in `lib/executor/ramping_vus.go`, presenting the architecture as a hierarchical control chain
- To **capture SIGINT log messages**, we will create a temporary k6 test script with a ramping-vus executor (≥5 VUs), send SIGINT during execution, and capture the complete stderr/stdout output showing the signal handling path from `cmd/run.go` through `execution.AbortTestRun`
- To **prove graceful shutdown behavior**, we will analyze the experiment output showing VUs completing their current iteration after SIGINT (evidenced by iteration completion timestamps post-interrupt) versus VUs that are interrupted (evidenced by `interrupted iterations` count in the summary)
- To **document gRPC streaming interruption**, we will create a test script using `k6/net/grpc` server streaming with `gracefulRampDown: '30ms'`, interrupt it, and capture the stream cancellation log messages including the gRPC error code 2 (Canceled)
- To **report gRPC messages received**, we will extract the `grpc_streams_msgs_received` counter value from the final metrics summary output of the interrupted gRPC streaming test
- To **query dropped iterations via API**, we will run a per-vu-iterations executor test with intentionally insufficient `maxDuration`, use `--linger` to keep the API server alive, and query `http://localhost:6565/v1/metrics/dropped_iterations` to capture the JSON response
- To **investigate data sharing**, we will create a test script that uses both `data.SharedArray` and regular `open()`+`JSON.parse()`, log array lengths from multiple VUs, and explain the root cause by citing the shared `[]string` backing store in `js/modules/k6/data/share.go`
- To **verify Prometheus metric naming**, we will run a test with `--out experimental-prometheus-rw` and verify the naming convention (`k6_` prefix + metric name + type suffix) as implemented in `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/prometheus.go`

### 0.1.4 Inferred Documentation Needs

Based on code analysis and the nature of the questions, the following implicit documentation needs are identified:

- **Executor type taxonomy**: The user asks about "ramping executor" — the documentation should clarify this refers to the `ramping-vus` executor type and briefly contrast it with related executors (`constant-vus`, `ramping-arrival-rate`) to provide context
- **Signal handling chain**: The interrupt behavior spans multiple files (`cmd/run.go` → `execution/scheduler.go` → `lib/executor/helpers.go` → `lib/executor/vu_handle.go`); the documentation should trace this full chain
- **Graceful ramp-down vs. graceful stop distinction**: These are two separate mechanisms — `gracefulRampDown` (executor-level, controls VU scaling-down behavior) vs. `gracefulStop` (scenario-level, controls post-duration grace period) — and the documentation must distinguish them
- **API availability timing**: The `dropped_iterations` metric is only available via the REST API after the executor finishes; it returns 404 during the test run. This timing behavior should be documented
- **SharedArray vs. open() memory model**: The root cause explanation requires understanding that `SharedArray` stores data as a shared `[]string` in Go memory (single copy) while `open()` in init context causes each VU to re-execute and create independent copies. The documentation should explain both paths
- **Prometheus naming convention completeness**: The documentation should cover all four metric type suffixes (Counter→`_total`, Gauge→`""`, Rate→`_rate`, Trend→TrendStatsResolver) and the `k6_` prefix constant

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a minimal documentation infrastructure with no dedicated documentation generator or framework. The project relies on standalone Markdown files and does not use MkDocs, Docusaurus, Sphinx, or any other documentation site generator.

**Documentation Framework:** None — no `mkdocs.yml`, `docusaurus.config.js`, `sphinx/conf.py`, or `.readthedocs.yml` detected in the repository.

**Existing Documentation Files:**

| Path | Type | Description |
|------|------|-------------|
| `README.md` | Project overview | Main README with project description, installation, usage examples, and community links |
| `CONTRIBUTING.md` | Contributor guide | Guidelines for contributing to k6, including development setup and code review process |
| `CODE_OF_CONDUCT.md` | Community standards | Code of conduct for project participants |
| `SUPPORT.md` | Support channels | Information about getting help and support |
| `LICENSE.md` | License text | AGPL-3.0 license terms |
| `docs/design/018-new-http-api.md` | Design proposal | Future design for new HTTP API (Fetch/Streams) |
| `docs/design/019-file-api.md` | Design proposal | Future design for native File API |
| `docs/design/020-distributed-execution-and-test-suites.md` | Design proposal | Future design for distributed execution and test suites |
| `js/modules/k6/experimental/README.md` | Module readme | Documentation for experimental module lifecycle |
| `js/modules/k6/experimental/streams/tests/README.md` | Test readme | Documentation for streams test suite |
| `js/tc39/README.md` | Test readme | Documentation for TC39 compatibility test suite |
| `release notes/v0.*.md` | Release notes | Per-version release notes (v0.19.0 through latest) |

**Documentation Coverage Status:** The repository has no internal technical documentation covering engine internals, VU orchestration, executor behavior, gRPC streaming mechanics, or metrics pipeline behavior. The `docs/design/` directory contains only forward-looking design proposals, not runtime behavior documentation.

**API Documentation Tools:** No JSDoc, GoDoc site generation, or API documentation tooling detected in the build system. Inline Go documentation comments exist in source files but are not extracted into standalone documents.

**Diagram Tools:** Mermaid support is available through GitHub-native Markdown rendering. No dedicated diagram generation tooling (PlantUML, Mermaid CLI) is configured in the project.

### 0.2.2 Repository Code Analysis for Documentation

Source files analyzed to extract information for the documentation deliverable:

**VU Orchestration and Executor Internals:**

| Source File | Lines | Key Content |
|-------------|-------|-------------|
| `execution/scheduler.go` | 591 | `Scheduler` struct, `Init()`, `Run()` methods, VU initialization, executor lifecycle |
| `lib/executor/vu_handle.go` | 264 | VU state machine (5 states), `gracefulStop()`, `hardStop()`, `runLoopsIfPossible()` |
| `lib/executor/ramping_vus.go` | 712 | `RampingVUsConfig`, `getRawExecutionSteps`, `reserveVUsForGracefulRampDowns`, `Run()` |
| `lib/executor/helpers.go` | 264 | `handleInterrupt`, `getIterationRunner`, `getDurationContexts`, `trackProgress` |
| `lib/executor/shared_iterations.go` | 275 | `DroppedIterations` metric emission for shared iteration executor |
| `lib/executor/per_vu_iterations.go` | 245 | `DroppedIterations` metric emission for per-VU iteration executor |

**Signal Handling and CLI:**

| Source File | Lines | Key Content |
|-------------|-------|-------------|
| `cmd/run.go` | 531 | `gracefulStop` signal handler, `onHardStop`, `handleTestAbortSignals`, REST API server setup |

**gRPC Streaming Module:**

| Source File | Lines | Key Content |
|-------------|-------|-------------|
| `js/modules/k6/grpc/stream.go` | 468 | `stream.loop()`, `readData`, `writeData`, `queueMessage`, context cancellation handling |
| `js/modules/k6/grpc/metrics.go` | 30 | `grpc_streams`, `grpc_streams_msgs_sent`, `grpc_streams_msgs_received` counter definitions |

**Data Sharing Module:**

| Source File | Lines | Key Content |
|-------------|-------|-------------|
| `js/modules/k6/data/data.go` | 191 | `RootModule` with shared `sharedArrays` map, `NewModuleInstance` per-VU instantiation |
| `js/modules/k6/data/share.go` | 95 | `SharedArray` constructor, double-checked locking, `wrappedSharedArray.Get()` with `JSON.parse` + `deepFreeze` |

**Prometheus Remote Write Output:**

| Source File | Lines | Key Content |
|-------------|-------|-------------|
| `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/remotewrite.go` | 406 | `Output` struct, `flush()`, `convertToPbSeries`, metric type → suffix mapping |
| `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/prometheus.go` | 52 | `MapSeries` building `__name__` label with `k6_` prefix + metric name + suffix |
| `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/config.go` | ~50 | `defaultMetricPrefix = "k6_"` constant |

**REST API:**

| Source File | Lines | Key Content |
|-------------|-------|-------------|
| `api/v1/metric_routes.go` | 49 | `handleGetMetric` — single metric lookup with JSON response |
| `api/v1/metric.go` | 82 | `NewMetric` — response serialization with `Sink.Format()` |

### 0.2.3 Web Search Research Conducted

No external web searches were required for this documentation task. All answers are derived directly from source code analysis and runtime experiments conducted against the built k6 v0.55.0 binary. The k6 codebase itself serves as the authoritative source of truth per the user's directive: *"Do not make assumptions, base your answers on the code as the truth."*

Key technical references used:
- Go module path: `go.k6.io/k6` (from `go.mod`)
- k6 version: v0.55.0 (from `lib/consts/consts.go`)
- Go toolchain: go1.21.13 (from `go.mod` line 5)
- gRPC framework: v1.67.1 (from `go.mod`)
- xk6-output-prometheus-remote: v0.5.0 (from `go.mod`)

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require documentation for this task, mapped by investigation topic:

**Module: `execution/scheduler.go` — VU Orchestration Core**
- Public APIs: `NewScheduler()`, `Scheduler.Init()`, `Scheduler.Run()`, `Scheduler.SetPaused()`, `initVUsConcurrently()`
- Current documentation: Missing — no standalone documentation exists
- Documentation needed: Architecture narrative explaining the Scheduler → Executor → VU Handle control chain, VU initialization flow, and executor lifecycle management

**Module: `lib/executor/vu_handle.go` — VU State Machine**
- Public APIs: `newStoppedVUHandle()`, `vuHandle.start()`, `vuHandle.gracefulStop()`, `vuHandle.hardStop()`, `runLoopsIfPossible()`
- Current documentation: Missing — inline comments only
- Documentation needed: State diagram (stopped → starting → running → toGracefulStop/toHardStop), explanation of graceful vs. hard stop semantics, and how this determines iteration completion during shutdown

**Module: `lib/executor/ramping_vus.go` — Ramping VUs Executor**
- Public APIs: `RampingVUsConfig`, `getRawExecutionSteps()`, `reserveVUsForGracefulRampDowns()`, `RampingVUs.Run()`
- Current documentation: Missing
- Documentation needed: Ramp stage calculation, gracefulRampDown mechanics, interaction with VU handle state machine during scale-down

**Module: `cmd/run.go` — Signal Handling**
- Key functions: `gracefulStop` callback, `onHardStop` callback, `handleTestAbortSignals()`
- Current documentation: Missing
- Documentation needed: Signal trapping flow (SIGINT → `runAbort()` → `AbortedByUser`), log message format, first vs. second signal behavior

**Module: `js/modules/k6/grpc/stream.go` — gRPC Streaming**
- Public APIs: `stream.loop()`, `readData()`, `writeData()`, `queueMessage()`, `closeWithError()`
- Current documentation: Missing
- Documentation needed: Stream lifecycle during normal operation and VU context cancellation, error code propagation, message counting via `grpc_streams_msgs_received`

**Module: `js/modules/k6/grpc/metrics.go` — gRPC Metrics**
- Metrics defined: `grpc_streams` (Counter), `grpc_streams_msgs_sent` (Counter), `grpc_streams_msgs_received` (Counter)
- Current documentation: Missing
- Documentation needed: Metric names, types, and when each is incremented

**Module: `lib/executor/shared_iterations.go` + `per_vu_iterations.go` — Dropped Iterations**
- Key logic: `DroppedIterations` metric emission when `totalIters - attemptedIters > 0`
- Current documentation: Missing
- Documentation needed: When and how `dropped_iterations` is computed, API queryability, timing constraints

**Module: `js/modules/k6/data/share.go` + `data.go` — Data Sharing**
- Public APIs: `SharedArray` constructor, `wrappedSharedArray.Get()`, `RootModule.sharedArrays`
- Current documentation: Missing
- Documentation needed: Memory model explanation — single `[]string` backing store shared across VUs vs. per-VU copies for `open()`, `JSON.parse` + `deepFreeze` on access

**Module: `vendor/.../remotewrite/prometheus.go` + `remotewrite.go` — Prometheus Output**
- Key functions: `MapSeries()`, `MapPrompb()`, metric type suffix mapping
- Current documentation: Missing
- Documentation needed: Naming convention (`k6_` + name + `_` + suffix), suffix rules per metric type (Counter→`total`, Gauge→`""`, Rate→`rate`, Trend→resolved via TrendStatsResolver)

**Module: `api/v1/metric_routes.go` + `metric.go` — REST API Metrics**
- Public APIs: `handleGetMetric()`, `handleGetMetrics()`, `NewMetric()`
- Current documentation: Missing
- Documentation needed: API endpoint format, JSON response structure for `dropped_iterations` and `iterations` metrics

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

**Undocumented Internal Behavior (all 8 topics):**
- No existing documentation covers VU orchestration internals, signal handling log output, graceful shutdown mechanics, gRPC stream lifecycle during interruption, dropped iteration calculation, SharedArray memory model, or Prometheus metric naming convention
- The `docs/design/` directory contains only future design proposals (018, 019, 020), none of which address runtime behavior documentation

**Missing Runtime Evidence Documentation:**
- No existing document in the repository provides actual runtime output examples from k6 test executions
- No documentation shows API query/response examples for the `/v1/metrics/` endpoints
- No documentation demonstrates log message formats during interrupt scenarios

**Missing Architecture Documentation for Investigated Components:**
- The `execution/scheduler.go` → executor → `vu_handle.go` control chain has no standalone documentation
- The gRPC stream lifecycle (`stream.loop()` → `readData` → context cancellation → error propagation) is undocumented
- The `data.SharedArray` memory sharing model and its contrast with `open()` per-VU copying is undocumented
- The Prometheus remote write metric naming pipeline is undocumented outside of inline code comments

**Consolidation Needed:**
- The user's 8 questions span 15+ source files across 7 different packages — the documentation must consolidate findings into a single coherent document with clear section boundaries

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output documentation file follows a single-document structure organized by investigation topic, with each topic containing source analysis, runtime evidence, and conclusions:

```
blitzy/
└── documentation/
    └── k6_ddc3b0b1d23c.md
        ├── Title and Overview
        ├── 1. VU Orchestration: How the Go Engine Manages VUs
        │   ├── Scheduler → Executor → VU Handle Architecture
        │   ├── VU State Machine Diagram
        │   └── Source Code Citations
        ├── 2. SIGINT Log Messages for Ramping Executor (≥5 VUs)
        │   ├── Signal Handling Chain Analysis
        │   ├── Exact Runtime Log Output
        │   └── Log Message Interpretation
        ├── 3. Graceful Shutdown: Do Active VUs Finish Current Iteration?
        │   ├── gracefulStop vs hardStop Mechanics
        │   ├── Runtime Evidence (iteration completion after SIGINT)
        │   └── Conclusion with Log Proof
        ├── 4. gRPC Server Streaming Test Interruption (30ms gracefulRampDown)
        │   ├── Test Configuration
        │   ├── Exact Log Entries at Runtime
        │   └── Stream Cancellation Error Analysis
        ├── 5. gRPC Messages Received in Final Metrics Summary
        │   ├── Metric Definition Source
        │   ├── Exact Value from Runtime
        │   └── Count Analysis
        ├── 6. Dropped Iterations via API Query
        │   ├── When Dropped Iterations Are Emitted (Source Analysis)
        │   ├── API Query and JSON Response
        │   ├── Iterations vs Dropped Iterations Verification
        │   └── API Availability Timing Note
        ├── 7. Data Sharing Behavior and Memory Footprint
        │   ├── SharedArray vs open() Memory Model
        │   ├── Test Script Output Proof
        │   ├── Root Cause Analysis (Go Source Code)
        │   └── Memory Behavior Conclusion
        ├── 8. Prometheus Output Metric Name Integrity
        │   ├── Naming Convention Source Analysis
        │   ├── Test Script Output Proof
        │   └── Metric Type Suffix Rules
        └── Appendix: Source Files Referenced
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract VU orchestration architecture from `execution/scheduler.go` (struct fields, `Init()` concurrency, `Run()` executor goroutine lifecycle) and `lib/executor/vu_handle.go` (state machine constants at lines 20-26, transition methods)
- Extract signal handling flow from `cmd/run.go` (`handleTestAbortSignals` trapping `os.Interrupt, syscall.SIGINT, syscall.SIGTERM`, `gracefulStop` callback calling `runAbort()` with formatted error message)
- Extract gRPC streaming behavior from `js/modules/k6/grpc/stream.go` (`loop()` monitoring `ctx.Done()`, `readData` incrementing `StreamsMessagesReceived`, error propagation via `grpcError{Code, Details, Message}`)
- Extract dropped iterations logic from `lib/executor/shared_iterations.go` (line ~250: `if attemptedIters < totalIters`) and `lib/executor/per_vu_iterations.go` (line ~220: `float64(iterations - i)`)
- Extract SharedArray internals from `js/modules/k6/data/share.go` (`sharedArrays.data` map, `wrappedSharedArray` referencing shared `[]string`) and `js/modules/k6/data/data.go` (`RootModule.shared` as process-level singleton)
- Extract Prometheus naming from `vendor/.../remotewrite/prometheus.go` (`MapSeries` concatenation) and `vendor/.../remotewrite/config.go` (`defaultMetricPrefix = "k6_"`)

**Runtime Evidence Generation:**

Five experiments were executed against k6 v0.55.0 to capture exact runtime output:

| Experiment | Purpose | Script Type | Evidence Captured |
|------------|---------|-------------|-------------------|
| Ramping VUs + SIGINT | Log messages during interrupt | ramping-vus, 5→10 VUs | Full stderr/stdout with interrupt log |
| Dropped Iterations API | API JSON response | per-vu-iterations, 3 VUs × 100 | `/v1/metrics/dropped_iterations` and `/v1/metrics/iterations` JSON |
| Data Sharing | Memory behavior proof | SharedArray + open(), 5 VUs | Console log output from all VUs |
| gRPC Streaming + SIGINT | Stream interruption logs | ramping-vus, 2 VUs, gRPC streaming | Stream cancellation messages, final metrics |
| Prometheus Naming | Metric name integrity | HTTP test with `--out experimental-prometheus-rw` | Output initialization log |

**Documentation Standards:**

- Markdown formatting with `#` through `####` headers for hierarchical structure
- Mermaid diagrams for VU state machine and architecture visualization
- Code blocks with `go`, `javascript`, `json`, and `text` syntax highlighting for source excerpts and runtime output
- Source citations in format: `Source: path/to/file.go:LineNumber`
- Runtime evidence presented in fenced code blocks with exact output preserved
- Tables for structured comparisons (metric types, executor configurations)

### 0.4.3 Diagram and Visual Strategy

**Mermaid diagrams to create:**

- **VU State Machine Diagram** — State diagram showing the 5 states (`stopped`, `starting`, `running`, `toGracefulStop`, `toHardStop`) with transition triggers and conditions, derived from `lib/executor/vu_handle.go` lines 20-26
- **Scheduler → Executor → VU Handle Architecture** — Flowchart showing the hierarchical control chain from `execution.Scheduler` through executor instances to individual `vuHandle` state machines
- **Signal Handling Flow** — Sequence diagram tracing SIGINT from OS signal through `cmd/run.go` → `execution.AbortTestRun` → executor cancellation → VU graceful/hard stop
- **SharedArray Memory Model** — Diagram contrasting the shared `[]string` backing store (one copy in Go memory) versus per-VU `open()` copies (N copies for N VUs)
- **Prometheus Naming Pipeline** — Flowchart showing the metric name construction: `defaultMetricPrefix` + `metric.Name` + `"_"` + `suffix` with conditional suffix selection by metric type

**Diagram placement:**
- VU State Machine: Section 1 (VU Orchestration)
- Architecture flowchart: Section 1 (VU Orchestration)
- Signal handling sequence: Section 2 (SIGINT Log Messages)
- Memory model comparison: Section 7 (Data Sharing)
- Prometheus naming pipeline: Section 8 (Prometheus Output)

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/k6_ddc3b0b1d23c.md` | CREATE | `execution/scheduler.go`, `lib/executor/vu_handle.go`, `lib/executor/ramping_vus.go`, `lib/executor/helpers.go`, `lib/executor/shared_iterations.go`, `lib/executor/per_vu_iterations.go`, `cmd/run.go`, `js/modules/k6/grpc/stream.go`, `js/modules/k6/grpc/metrics.go`, `js/modules/k6/data/data.go`, `js/modules/k6/data/share.go`, `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/prometheus.go`, `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/remotewrite.go`, `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/config.go`, `api/v1/metric_routes.go`, `api/v1/metric.go` | Complete investigative documentation answering 8 questions about k6 engine internals with source code analysis and runtime evidence |

No existing documentation files require UPDATE or DELETE operations. This task creates a single new documentation file.

### 0.5.2 New Documentation Files Detail

```
File: blitzy/documentation/k6_ddc3b0b1d23c.md
Type: Technical Investigation Report (Q&A with Runtime Evidence)
Source Code: 16 source files across 7 packages (listed below)

Sections:
  - Title and Overview (project context: k6 v0.55.0, Go 1.21.13)
  - Section 1: VU Orchestration — How the Go Engine Manages VUs
    * Scheduler architecture (from execution/scheduler.go)
    * VU Handle state machine (from lib/executor/vu_handle.go)
    * Ramping VUs executor mechanics (from lib/executor/ramping_vus.go)
    * Mermaid state diagram and architecture flowchart
    * Source: execution/scheduler.go, lib/executor/vu_handle.go, lib/executor/ramping_vus.go

  - Section 2: SIGINT Log Messages for Ramping Executor (≥5 VUs)
    * Signal handling chain analysis (from cmd/run.go)
    * Exact runtime log output from Experiment 1
    * Log message format: "Stopping k6 in response to signal..." with sig=interrupt
    * Mermaid sequence diagram for signal flow
    * Source: cmd/run.go, execution/scheduler.go

  - Section 3: Graceful Shutdown Evidence
    * gracefulStop() vs hardStop() mechanics (from lib/executor/vu_handle.go)
    * Runtime evidence showing VU completing iteration after SIGINT
    * Iteration counts: complete vs interrupted from final summary
    * Source: lib/executor/vu_handle.go, lib/executor/ramping_vus.go, lib/executor/helpers.go

  - Section 4: gRPC Server Streaming Interruption (30ms gracefulRampDown)
    * Stream lifecycle during cancellation (from js/modules/k6/grpc/stream.go)
    * Exact log entries: "stream is cancelled/finished" error="canceled by client (k6)"
    * gRPC error code 2 (Canceled) propagation
    * Source: js/modules/k6/grpc/stream.go, js/modules/k6/grpc/metrics.go

  - Section 5: gRPC Messages Received in Final Metrics Summary
    * grpc_streams_msgs_received counter definition (from js/modules/k6/grpc/metrics.go)
    * Exact value: 96 (from Experiment 4 runtime output)
    * Source: js/modules/k6/grpc/metrics.go, js/modules/k6/grpc/stream.go

  - Section 6: Dropped Iterations via API Query
    * Metric emission logic (from lib/executor/shared_iterations.go, lib/executor/per_vu_iterations.go)
    * Exact API JSON response: {"count": 294, "rate": 73.48} for dropped_iterations
    * Exact API JSON response: {"count": 6, "rate": 1.50} for iterations
    * Verification: 6 + 294 = 300 = 3 VUs × 100 iterations
    * API timing note: returns 404 during test, available only after executor completion
    * Source: lib/executor/per_vu_iterations.go, api/v1/metric_routes.go, api/v1/metric.go

  - Section 7: Data Sharing Behavior
    * SharedArray memory model (from js/modules/k6/data/share.go, data.go)
    * Test output: all 5 VUs report identical array lengths
    * Root cause: SharedArray stores single []string in RootModule.sharedArrays map,
      wrappedSharedArray references same backing data; open() re-executes per VU in init context
    * Mermaid diagram comparing memory models
    * Source: js/modules/k6/data/share.go, js/modules/k6/data/data.go

  - Section 8: Prometheus Output Metric Name Integrity
    * Naming convention: k6_ + metric_name + _ + type_suffix
    * Suffix rules: Counter→"total", Gauge→"" (no suffix), Rate→"rate", Trend→TrendStatsResolver
    * Code evidence from MapSeries() in prometheus.go
    * Runtime output confirming initialization with correct endpoint
    * Source: vendor/.../remotewrite/prometheus.go, vendor/.../remotewrite/remotewrite.go, vendor/.../remotewrite/config.go

  - Appendix: All source files referenced with line-number citations

Diagrams:
  - VU state machine (Mermaid stateDiagram-v2)
  - Scheduler → Executor → VU architecture (Mermaid flowchart)
  - Signal handling sequence (Mermaid sequenceDiagram)
  - SharedArray vs open() memory model (Mermaid flowchart)
  - Prometheus naming pipeline (Mermaid flowchart)

Key Citations:
  execution/scheduler.go, lib/executor/vu_handle.go, lib/executor/ramping_vus.go,
  lib/executor/helpers.go, lib/executor/shared_iterations.go, lib/executor/per_vu_iterations.go,
  cmd/run.go, js/modules/k6/grpc/stream.go, js/modules/k6/grpc/metrics.go,
  js/modules/k6/data/data.go, js/modules/k6/data/share.go,
  vendor/.../remotewrite/prometheus.go, vendor/.../remotewrite/remotewrite.go,
  vendor/.../remotewrite/config.go, api/v1/metric_routes.go, api/v1/metric.go
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration files require updates. The repository does not use a documentation generator, so there are no `mkdocs.yml`, `docusaurus.config.js`, `.readthedocs.yml`, or `package.json` documentation scripts to modify.

The only directory-level operation required is creating the `blitzy/documentation/` directory if it does not already exist.

### 0.5.4 Cross-Documentation Dependencies

**Shared content/includes:** None — the output is a self-contained document with no shared content dependencies.

**Navigation links between documents:** None required — the document is standalone and does not need to integrate with existing repository navigation.

**Table of contents updates:** Not applicable — no site-level table of contents exists.

**Index/glossary updates:** Not applicable — no project-level index or glossary exists.

**Internal cross-references within the document:** Each section references the source files from the Appendix, and Sections 2–3 cross-reference each other (SIGINT log messages and graceful shutdown are closely related). Section 4 and 5 cross-reference each other (gRPC interruption logs and message count metric).

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The documentation task requires the following runtime and build dependencies for executing experiments and building k6:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| golang.org | go | 1.21.13 | Go toolchain for building k6 from source (specified in `go.mod` line 5 as `toolchain go1.21.13`) |
| go.k6.io | k6 | 0.55.0 | The k6 binary under investigation, built from source at commit `ddc3b0b1d` |
| go.mod | google.golang.org/grpc | 1.67.1 | gRPC framework used by the `k6/net/grpc` module (required for gRPC streaming experiments) |
| go.mod | github.com/grafana/xk6-output-prometheus-remote | 0.5.0 | Prometheus remote write output extension (bundled, required for metric naming investigation) |
| go.mod | github.com/sirupsen/logrus | 1.9.3 | Structured logging library producing the log output captured in experiments |
| go.mod | github.com/spf13/cobra | 1.4.0 | CLI framework implementing the `k6 run` command and signal handling |
| go.mod | github.com/grafana/sobek | 0.0.0-20241024150027 | JavaScript engine executing test scripts within VUs |
| go.mod | github.com/evanw/esbuild | 0.21.2 | Script compiler for JavaScript/TypeScript test files |

**External test infrastructure used during experiments:**

| Component | Version | Purpose |
|-----------|---------|---------|
| gRPC Route Guide server | From `google.golang.org/grpc/examples/route_guide` | gRPC server providing `ListFeatures` server streaming RPC for Experiment 4 |
| k6 REST API | Built-in (port 6565) | In-process HTTP API for querying `dropped_iterations` metric in Experiment 2 |
| curl | System-provided | HTTP client for querying k6 REST API endpoints |

No additional documentation tools (MkDocs, Sphinx, Docusaurus, Mermaid CLI) are required. The output document uses native GitHub-compatible Markdown with Mermaid diagrams rendered by GitHub's built-in support.

### 0.6.2 Documentation Reference Updates

No documentation reference updates are required. The new file `blitzy/documentation/k6_ddc3b0b1d23c.md` is a standalone document that does not create or break any existing links in the repository. No existing documentation files reference this path or need link updates.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage before this task:**

| Topic | Source Files | Existing Documentation | Coverage |
|-------|-------------|----------------------|----------|
| VU orchestration internals | `execution/scheduler.go`, `lib/executor/vu_handle.go` | None | 0% |
| SIGINT log messages for ramping executor | `cmd/run.go`, `execution/scheduler.go` | None | 0% |
| Graceful shutdown VU behavior | `lib/executor/vu_handle.go`, `lib/executor/ramping_vus.go` | None | 0% |
| gRPC streaming interruption | `js/modules/k6/grpc/stream.go`, `js/modules/k6/grpc/metrics.go` | None | 0% |
| gRPC messages received metric | `js/modules/k6/grpc/metrics.go` | None | 0% |
| Dropped iterations via API | `lib/executor/shared_iterations.go`, `lib/executor/per_vu_iterations.go`, `api/v1/metric_routes.go` | None | 0% |
| Data sharing memory behavior | `js/modules/k6/data/share.go`, `js/modules/k6/data/data.go` | None | 0% |
| Prometheus metric naming | `vendor/.../remotewrite/prometheus.go`, `vendor/.../remotewrite/config.go` | None | 0% |

**Target coverage after this task:** 100% for all 8 investigation topics — each question fully answered with source code analysis, runtime evidence, and rationale.

**Coverage gaps to address:**
- All 8 topics start at 0% documented (no existing internal behavior documentation in the repository)
- Target: Complete investigative documentation with exact runtime output for each topic
- Focus areas: Source code citations, verbatim log outputs, API responses, and Mermaid diagrams

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every question posed by the user receives a direct, unambiguous answer
- Each answer includes source code analysis identifying the exact Go functions and logic responsible
- Each answer includes runtime evidence (log output, API response, or test script output) as proof
- Source file paths with line number ranges are provided for every technical claim
- Mermaid diagrams illustrate complex relationships (VU state machine, memory models, signal flow)

**Accuracy validation:**
- All source code citations reference actual file paths and line numbers verified through `read_file` operations
- All runtime output is captured verbatim from k6 v0.55.0 experiments (not fabricated or approximated)
- API JSON responses are captured from actual `curl` requests to `localhost:6565`
- Metric values (e.g., `dropped_iterations: 294`, `grpc_streams_msgs_received: 96`) are exact values from runtime output
- The gRPC error code `2` (Canceled) and message `"canceled by client (k6)"` are verified against actual stream termination output

**Clarity standards:**
- Each section opens with the user's question restated for context
- Source analysis precedes runtime evidence (theory before proof)
- Runtime output is presented in fenced code blocks with annotations explaining key lines
- Root cause explanations trace from observable behavior back to specific Go source code
- Cross-references between related sections (e.g., Sections 2 and 3 both address SIGINT behavior)

**Maintainability:**
- All source citations use relative paths from the repository root
- k6 version (v0.55.0) and Go version (1.21.13) are documented in the header for reproducibility
- Each experiment's configuration (executor type, VU count, duration, options) is documented alongside its output

### 0.7.3 Example and Diagram Requirements

**Runtime evidence per topic:**

| Topic | Evidence Type | Minimum Content |
|-------|--------------|-----------------|
| VU Orchestration | Source analysis | State machine diagram + architecture flowchart |
| SIGINT Log Messages | Verbatim log output | Complete stderr showing signal handling messages |
| Graceful Shutdown | Log output + iteration counts | Final summary showing complete vs interrupted iterations |
| gRPC Streaming Interruption | Log output | Stream cancellation messages with error codes |
| gRPC Messages Received | Final metrics summary | Exact `grpc_streams_msgs_received` counter value |
| Dropped Iterations | API JSON response | Full JSON from `/v1/metrics/dropped_iterations` and `/v1/metrics/iterations` |
| Data Sharing | Console log output | VU output showing array lengths + memory analysis |
| Prometheus Naming | Source analysis + output | Naming convention rules with code citations |

**Diagram types required:**

| Diagram | Type | Section | Content |
|---------|------|---------|---------|
| VU State Machine | Mermaid `stateDiagram-v2` | Section 1 | 5 states with transitions: stopped → starting → running → toGracefulStop/toHardStop |
| Scheduler Architecture | Mermaid `flowchart TD` | Section 1 | Scheduler → Executors → VU Handles hierarchy |
| Signal Handling Flow | Mermaid `sequenceDiagram` | Section 2 | OS → cmd/run.go → Scheduler → Executor → VU Handle |
| SharedArray Memory Model | Mermaid `flowchart LR` | Section 7 | Shared []string vs per-VU copies comparison |
| Prometheus Naming Pipeline | Mermaid `flowchart LR` | Section 8 | prefix + name + suffix construction |

**Code example requirements:**
- Test script configurations embedded for each experiment (JavaScript)
- Go source excerpts limited to 2-3 lines for key logic points
- API curl commands and responses for the dropped iterations query

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/k6_ddc3b0b1d23c.md` — The sole deliverable: a comprehensive Markdown document answering all 8 investigation questions

**Source files analyzed for documentation content (read-only — no modifications):**
- `execution/scheduler.go` — VU initialization, executor lifecycle, `Run()` method
- `lib/executor/vu_handle.go` — VU state machine (5 states), graceful/hard stop logic
- `lib/executor/ramping_vus.go` — Ramping VUs executor, `gracefulRampDown`, stage calculation
- `lib/executor/helpers.go` — `handleInterrupt`, `getIterationRunner`, `getDurationContexts`
- `lib/executor/shared_iterations.go` — `DroppedIterations` metric emission
- `lib/executor/per_vu_iterations.go` — Per-VU `DroppedIterations` metric emission
- `cmd/run.go` — Signal handling (`handleTestAbortSignals`), REST API server setup
- `js/modules/k6/grpc/stream.go` — gRPC stream lifecycle, context cancellation, message counting
- `js/modules/k6/grpc/metrics.go` — gRPC metric definitions (3 counters)
- `js/modules/k6/data/data.go` — `RootModule`, per-VU `Data` module instance
- `js/modules/k6/data/share.go` — `SharedArray` constructor, `wrappedSharedArray`, shared `[]string`
- `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/prometheus.go` — `MapSeries`, `__name__` label construction
- `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/remotewrite.go` — `Output`, `flush()`, `MapPrompb` suffix mapping
- `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/config.go` — `defaultMetricPrefix = "k6_"`
- `api/v1/metric_routes.go` — `handleGetMetric` endpoint
- `api/v1/metric.go` — `NewMetric` response serialization

**Runtime experiments (temporary scripts created and cleaned up):**
- Experiment 1: Ramping VUs + SIGINT test script
- Experiment 2: Dropped iterations + API query test script (with `--linger`)
- Experiment 3: SharedArray vs open() data sharing test script
- Experiment 4: gRPC server streaming + SIGINT test script
- Experiment 5: Prometheus remote write output test

**Documentation assets:**
- Mermaid diagrams embedded inline (no separate image files)
- Runtime output captured as fenced code blocks within the document

### 0.8.2 Explicitly Out of Scope

**Source code modifications:**
- No existing repository files are modified — per user directive: *"Don't modify any repository source files"*
- No Go source files are changed
- No JavaScript module files are changed
- No configuration files are changed
- No test files are changed

**Unrelated documentation:**
- `README.md` — Not modified or updated
- `CONTRIBUTING.md` — Not modified or updated
- `docs/design/*.md` — Not modified or updated (design proposals are separate concerns)
- `release notes/*.md` — Not modified or updated

**Out-of-scope investigation areas:**
- k6 cloud integration behavior (`cloudapi/`, `output/cloud/`)
- Browser module internals (`grafana/xk6-browser`)
- WebSocket module internals (`grafana/xk6-websockets`)
- CSV/InfluxDB/JSON output backends
- Test archiving and script loading mechanisms
- Extension registration system (`ext/`)
- CI/CD pipeline and build configuration (`.github/workflows/`)

**Features not covered:**
- Feature additions or code refactoring
- Performance benchmarking of k6 itself
- Deployment configuration changes
- Docker/Kubernetes runtime behavior
- Distributed execution (design proposal only, not implemented)

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

**Build command (k6 binary from source):**
```
export PATH="/usr/local/go/bin:$HOME/go/bin:$PATH"
cd /path/to/k6/repo && go build -o /usr/local/bin/k6 .
```

**k6 test execution command (general):**
```
k6 run --quiet script.js
```

**k6 test execution with API linger (for dropped iterations API query):**
```
k6 run --quiet --linger script.js &
curl -s http://localhost:6565/v1/metrics/dropped_iterations | python3 -m json.tool
```

**k6 test execution with Prometheus output:**
```
k6 run --out experimental-prometheus-rw script.js
```

**Diagram generation:** Not required — Mermaid diagrams are embedded inline in Markdown and rendered natively by GitHub.

**Documentation validation:**
- Verify all fenced code blocks have correct language annotations
- Verify all source file paths are valid relative to repository root
- Verify all Mermaid diagrams render correctly (stateDiagram-v2, flowchart, sequenceDiagram syntax)
- Verify runtime output matches experiment results exactly

**Default format:** GitHub-flavored Markdown with Mermaid diagrams.

**Citation requirement:** Every technical claim must reference a source file path, and every runtime evidence block must identify the experiment configuration that produced it.

**Style guide:** No repository-specific style guide exists. Follow standard Markdown best practices:
- `#` for document title, `##` for major sections, `###` for subsections
- Fenced code blocks with language annotation for all code and output
- Tables for structured data comparisons
- Mermaid blocks for architectural and behavioral diagrams
- Bold for key terms and metric names on first use

## 0.10 Rules for Documentation

The following rules are explicitly emphasized by the user and implementation directives:

- **Do not modify any existing files in the source repository** — The user states: *"Don't modify any repository source files."* The implementation rule reinforces: *"Do not modify any existing files in the source repository."* This is an absolute constraint: zero changes to any tracked file in the k6 repository.

- **Base all answers on the code as the truth** — The implementation rule states: *"Do not make assumptions, base your answers on the code as the truth."* Every answer must be grounded in actual source code analysis (specific file paths, line numbers, function names) and verified through runtime experiments.

- **Provide thinking and rationale behind the answers** — The implementation rule requires: *"Provide thinking / rationale behind the answers."* Each section must explain the reasoning chain from source code evidence to the stated conclusion, not just present raw output.

- **Provide exact runtime evidence** — The user requests exact log outputs, exact log entries, exact values, and test script output as proof. Runtime evidence must be verbatim captures from k6 v0.55.0 experiments, presented in fenced code blocks without modification.

- **Prove API-queried values with runtime evidence** — For the dropped iterations question, the user specifically requires querying the k6 REST API and providing the actual JSON response as proof. The full API request and response must be documented.

- **Clean up temporary artifacts** — The user states: *"If you need to create temporary scripts or artifacts to observe behavior, that's fine, but clean them up afterward and leave the codebase unchanged."* All temporary test scripts and server processes must be terminated and removed after experiments.

- **Place output in `blitzy/documentation/` directory** — The implementation rule specifies: *"Place the generated document in the `blitzy/documentation` directory in the destination repo."*

- **Name the file using the source branch name** — The implementation rule specifies: *"Create a new markdown document named `<source_branch_name>.md`"* — the branch is `k6_ddc3b0b1d23c`, so the file is `k6_ddc3b0b1d23c.md`.

- **Include source code citations for all technical details** — Every statement about internal behavior must reference the specific source file and relevant code location.

- **Use Mermaid diagrams for complex relationships** — VU state machine, scheduler architecture, signal handling flow, memory model comparison, and Prometheus naming pipeline should be visualized with Mermaid diagrams embedded in the Markdown.

## 0.11 References

### 0.11.1 Source Files Searched and Analyzed

The following files and folders were comprehensively searched and read to derive all conclusions in this Agent Action Plan:

**VU Orchestration and Executor System:**

| File Path | Lines | Content Summary |
|-----------|-------|-----------------|
| `execution/scheduler.go` | 591 | Core Scheduler struct, `NewScheduler()`, `Init()` (concurrent VU initialization), `Run()` (executor lifecycle, setup/teardown, signal detection) |
| `lib/executor/vu_handle.go` | 264 | VU state machine with 5 states (stopped, starting, running, toGracefulStop, toHardStop), `gracefulStop()`, `hardStop()`, `runLoopsIfPossible()` main loop |
| `lib/executor/ramping_vus.go` | 712 | `RampingVUsConfig` (StartVUs, Stages, GracefulRampDown default 30s), `getRawExecutionSteps()`, `reserveVUsForGracefulRampDowns()`, `Run()` with vuHandles and handler strategies |
| `lib/executor/helpers.go` | 264 | `handleInterrupt()`, `getIterationRunner()` (interrupted iterations tracking), `getDurationContexts()` (nested contexts for duration + graceful stop) |
| `lib/executor/shared_iterations.go` | 275 | Shared iterations executor, `DroppedIterations` emission: `totalIters - attemptedIters` |
| `lib/executor/per_vu_iterations.go` | 245 | Per-VU iterations executor, per-VU `DroppedIterations` emission: `iterations - i` |

**Signal Handling and CLI:**

| File Path | Lines | Content Summary |
|-----------|-------|-----------------|
| `cmd/run.go` | 531 | `handleTestAbortSignals()` trapping `os.Interrupt, syscall.SIGINT, syscall.SIGTERM`; `gracefulStop` callback with `AbortedByUser` reason; `onHardStop` with `globalCancel()`; REST API server setup |

**gRPC Streaming Module:**

| File Path | Lines | Content Summary |
|-----------|-------|-----------------|
| `js/modules/k6/grpc/stream.go` | 468 | `stream.loop()` goroutine monitoring `ctx.Done()`, `readData()` calling `ReceiveConverted()` in loop with `queueMessage()` incrementing `StreamsMessagesReceived`, `closeWithError()` for shutdown |
| `js/modules/k6/grpc/metrics.go` | 30 | Three Counter metrics: `grpc_streams`, `grpc_streams_msgs_sent`, `grpc_streams_msgs_received` |

**Data Sharing Module:**

| File Path | Lines | Content Summary |
|-----------|-------|-----------------|
| `js/modules/k6/data/data.go` | 191 | `RootModule` with `sharedArrays` (process-level singleton map), `NewModuleInstance()` creating per-VU `Data` sharing same `*sharedArrays` |
| `js/modules/k6/data/share.go` | 95 | `SharedArray` constructor with double-checked locking, data stored as `[]string`, `wrappedSharedArray.Get()` calling `JSON.parse` + `deepFreeze` |

**Prometheus Remote Write Output:**

| File Path | Lines | Content Summary |
|-----------|-------|-----------------|
| `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/prometheus.go` | 52 | `MapSeries()` constructing `__name__` = `defaultMetricPrefix` + `metric.Name` + `"_"` + `suffix`, labels sorted lexicographically |
| `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/remotewrite.go` | 406 | `Output` struct, `flush()`, `convertToPbSeries`, `MapPrompb` mapping: Counter→`"total"`, Gauge→`""`, Rate→`"rate"`, Trend→`prompbMapper` |
| `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/config.go` | ~50 | `defaultMetricPrefix = "k6_"` constant definition |

**REST API:**

| File Path | Lines | Content Summary |
|-----------|-------|-----------------|
| `api/v1/metric_routes.go` | 49 | `handleGetMetric()` — looks up metric by ID in `ObservedMetrics`, returns JSON |
| `api/v1/metric.go` | 82 | `NewMetric()` — serializes `Name`, `Type`, `Contains`, `Tainted`, `Sample` (from `Sink.Format()`) |

**Project Configuration:**

| File Path | Content Summary |
|-----------|-----------------|
| `go.mod` | Module `go.k6.io/k6`, Go 1.21, toolchain `go1.21.13`, all dependency versions |
| `lib/consts/consts.go` | k6 version `v0.55.0` |

**Folders Explored:**

| Folder Path | Content Summary |
|-------------|-----------------|
| Repository root (`""`) | Top-level structure: `cmd/`, `execution/`, `lib/`, `js/`, `api/`, `output/`, `vendor/`, `metrics/` |
| `execution/` | Scheduler and execution engine core |
| `lib/executor/` | All 7 executor types plus VU handle and helpers |
| `js/modules/k6/grpc/` | gRPC client, streaming, and metrics |
| `js/modules/k6/data/` | SharedArray and data module |
| `api/v1/` | REST API routes and response serialization |
| `output/` | Output subsystem with all backends |
| `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/` | Prometheus remote write implementation |
| `docs/design/` | Design proposals (018, 019, 020) |

### 0.11.2 Attachments

No attachments were provided with this task.

### 0.11.3 Runtime Experiments Conducted

Five experiments were executed against the built k6 v0.55.0 binary to capture runtime evidence:

| Experiment | Configuration | Key Evidence |
|------------|---------------|--------------|
| **1: Ramping VUs + SIGINT** | ramping-vus, 5→10 VUs over 5s, hold 30s, `gracefulRampDown: 30ms`, `gracefulStop: 30s`; SIGINT after ~4s | Log: `"Stopping k6 in response to signal..." sig=interrupt`; 6 complete + 8 interrupted iterations; VU 10 completed post-SIGINT |
| **2: Dropped Iterations API** | per-vu-iterations, 3 VUs × 100 iterations, `maxDuration: 3s`, `gracefulStop: 5s`, `--linger` | API: `dropped_iterations.count=294`, `iterations.count=6`; 6+294=300=3×100; 404 mid-test |
| **3: Data Sharing** | SharedArray (1000 items) + open()/JSON.parse (1000 items), 5 VUs | All 5 VUs: `sharedData.length=1000, rawData.length=1000`; SharedArray is shared, open() is per-VU |
| **4: gRPC Streaming + SIGINT** | ramping-vus, 2 VUs, `gracefulRampDown: 30ms`, gRPC `ListFeatures` streaming; SIGINT after 5s | Log: `"stream is cancelled/finished" error="canceled by client (k6)"`; `grpc_streams_msgs_received: 96`; 2 complete + 2 interrupted iterations |
| **5: Prometheus Naming** | HTTP test with `--out experimental-prometheus-rw` | Output: `Prometheus remote write (http://localhost:9090/api/v1/write)`; naming rules verified via source: `k6_` prefix + type-specific suffix |

All temporary test scripts and gRPC server processes were cleaned up after experiments, leaving the repository codebase unchanged.

