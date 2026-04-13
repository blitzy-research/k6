# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to **conduct a deep code-level investigation into potential concurrency bugs in k6's `ramping-vus` executor**, and produce a comprehensive Q&A document with findings. The user has observed multiple anomalous behaviors and requests a trace-through of the internal concurrency model. Specifically:

- **VU "stuck" state during rapid stage transitions**: When stages ramp up and down quickly with a long `gracefulRampDown`, VUs appear to enter an indeterminate state — neither fully active nor fully stopped. The user needs a code-grounded explanation of whether this is an actual state-machine gap or expected transient behavior.
- **Mismatch between scheduled handler and graceful handler VU counts**: The user has observed that the number of VUs tracked by the `scheduledVUsHandlerStrategy` closure does not align with what the `maxAllowedVUsHandlerStrategy` closure believes should exist at certain moments. The investigation must explain whether these are independently-maintained counters by design or a synchronization defect.
- **VUs exceeding `gracefulStop` after Ctrl+C**: When the test is interrupted via Ctrl+C, some VUs continue executing beyond the `gracefulStop` timeout boundary. The analysis must trace the context cancellation propagation path from the parent context through `maxDurationCtx` down to individual `vuHandle.ctx` instances.
- **Execution segment VU count imbalance**: When using three k6 instances with execution segments intended to split load evenly, one instance consistently shows more VUs than the others at the same timestamp, and the total sum of VUs across instances exceeds the configured maximum. The investigation must examine the `SegmentedIndex`, `ExecutionTuple.ScaleInt64()`, and the striping algorithm to determine whether this is a rounding or sequencing defect.
- **Race condition between handler goroutines**: The user suspects a race condition between the two handler goroutines (`handleNewMaxAllowedVUs` and `handleNewScheduledVUs`). The analysis must clarify whether these actually run concurrently and assess shared-state synchronization.
- **VU buffer leak**: The user suspects the VU channel buffer (`vus chan InitializedVU` in `ExecutionState`) may be leaking — VUs not being properly returned after use. The investigation must trace the `getVU`/`returnVU` lifecycle.

Implicit requirements detected:
- The investigation must be strictly grounded in the actual code, not assumptions
- Temporary investigation scripts are acceptable but must be cleaned up after
- The output is a comprehensive markdown analysis document, not code modifications

### 0.1.2 Special Instructions and Constraints

- **Implementation Rule `SWE-AtlasQnA-Repo`**: Create a new markdown document named `k6_ddc3b0b1d23c.md` that comprehensively answers the question(s) posed in the prompt
- Provide thinking and rationale behind the answers
- Do NOT make assumptions — base all answers on the code as the source of truth
- Do NOT modify any existing files in the source repository
- Do NOT add any other code besides the requested document
- Place the generated document in the `blitzy/documentation` directory in the destination repo
- The document must trace through actual code paths, referencing specific files, line ranges, and function names

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To **answer the VU stuck-state question**, we will trace the `vuHandle` state machine in `lib/executor/vu_handle.go`, analyzing all five states (`stopped`, `starting`, `running`, `toGracefulStop`, `toHardStop`) and their transitions, paying particular attention to the race window between `start()`, `gracefulStop()`, and the `runLoopsIfPossible()` loop.
- To **explain the handler VU count mismatch**, we will analyze `rampingVUsRunState.iterateSteps()` in `lib/executor/ramping_vus.go` and show that `handleNewMaxAllowedVUs` and `handleNewScheduledVUs` are closures with independent `cur` counters that represent conceptually different things (graceful reservation ceiling vs. actively-scheduled VU target).
- To **trace the Ctrl+C context propagation**, we will follow the chain: `ctx` (parent) → `maxDurationCtx` (via `getDurationContexts` in `lib/executor/helpers.go`) → `vuHandle.parentCtx` → `vuHandle.ctx` (via `context.WithCancel(parentCtx)`) and assess whether `hardStop()` is ever called on remaining VU handles after interrupt.
- To **analyze execution segment imbalance**, we will examine `ExecutionSegmentSequenceWrapper.ScaleInt64()` and `NewExecutionSegmentSequenceWrapper()` in `lib/execution_segment.go`, the striping algorithm's offset computation, and whether the user is providing a segment sequence (critical for correct distribution) vs. just segments alone.
- To **assess the race condition between handlers**, we will show that both handlers are called sequentially within `iterateSteps()`, not concurrently, and that the only concurrent access is between the scheduler goroutine and VU goroutines (synchronized via `vuHandle.mutex`).
- To **investigate VU buffer leaks**, we will trace the `getVU`/`returnVU` closures in `runLoopsIfPossible()`, the `ExecutionState.vus` channel, and the `GetPlannedVU()`/`ReturnVU()` methods in `lib/execution.go`.


## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

The investigation targets the core executor subsystem and its concurrency model. Every file listed below was retrieved and analyzed to form the answers in the Q&A document.

**Primary investigation targets (executor subsystem):**

| File Path | Relevance | Lines Analyzed |
|-----------|-----------|----------------|
| `lib/executor/ramping_vus.go` | Core ramping VUs executor: `Run()`, `iterateSteps()`, `scheduledVUsHandlerStrategy()`, `maxAllowedVUsHandlerStrategy()`, `runRemainingGracefulSteps()`, `waiter()`, raw and graceful step generation, `reserveVUsForGracefulRampDowns()` | Full file (713 lines) |
| `lib/executor/vu_handle.go` | VU lifecycle state machine: 5-state FSM (`stopped`, `starting`, `running`, `toGracefulStop`, `toHardStop`), `start()`, `gracefulStop()`, `hardStop()`, `runLoopsIfPossible()`, mutex-based synchronization | Full file (265 lines) |
| `lib/execution.go` | `ExecutionState`: VU buffer channel (`vus chan InitializedVU`), `GetPlannedVU()`, `ReturnVU()`, `ModCurrentlyActiveVUsCount()`, active/initialized VU counters, pause/resume | Full file (550 lines) |
| `lib/execution_segment.go` | `ExecutionSegment`, `ExecutionSegmentSequence`, `ExecutionSegmentSequenceWrapper`, `ExecutionTuple`, `SegmentedIndex`, `ScaleInt64()`, striping algorithm, `GoTo()`, `Next()`, `Prev()` | Full file (843 lines) |
| `lib/executor/helpers.go` | `getDurationContexts()`, `getIterationRunner()`, `trackProgress()`, `getVUActivationParams()`, `handleInterrupt()` | Full file (265 lines) |
| `lib/executor/base_config.go` | `BaseConfig`, `DefaultGracefulStopValue` (30s), `GetGracefulStop()` | Full file (148 lines) |
| `lib/executor/base_executor.go` | `BaseExecutor`, `nextIterationCounters()`, segmented iteration bookkeeping | Full file (86 lines) |
| `lib/executors.go` | `ExecutionStep`, `ExecutorConfig` interface, `ScenarioConfigs.GetFullExecutionRequirements()`, step consolidation algorithm | Full file (345 lines) |
| `lib/helpers.go` | `GetMaxPlannedVUs()`, `GetMaxPossibleVUs()`, `GetEndOffset()` | Full file (95 lines) |
| `lib/runner.go` | `ActiveVU`, `InitializedVU`, `VUActivationParams` interfaces | Full file (102 lines) |

**Test files analyzed for behavioral verification:**

| File Path | Relevance |
|-----------|-----------|
| `lib/executor/ramping_vus_test.go` | `TestRampingVUsRun`, `TestRampingVUsGracefulStopWaits`, `TestRampingVUsGracefulStopStops`, `TestRampingVUsGracefulRampDown`, `TestRampingVUsHandleRemainingVUs`, `TestRampingVUsRampDownNoWobble`, execution plan examples, segment sum verification (`TestSumRandomSegmentSequenceMatchesNoSegment`) |
| `lib/executor/vu_handle_test.go` | `TestVUHandleRace` (concurrent start/gracefulStop/hardStop), `TestVUHandleStartStopRace`, `TestVUHandleSimple` (start-before-gracefulStop, start-after-gracefulStop, start-after-hardStop) |
| `lib/executor/common_test.go` | `setupExecutorTest()`, `initializeVUs()`, `simpleRunner()` test helpers |

**Scheduler and coordination layer:**

| File Path | Relevance |
|-----------|-----------|
| `execution/scheduler.go` | `Scheduler.Run()`, `runExecutor()`, context propagation from `runCtx` to executors, `executorsRunCtx` cancellation on error |
| `execution/abort.go` | `testAbortController`, `NewTestRunContext()`, `AbortTestRun()`, mutex-protected abort reason tracking |
| `execution/controller.go` | `Controller` interface, `SignalAndWait()` barrier pattern |

**Configuration and module manifests:**

| File Path | Relevance |
|-----------|-----------|
| `go.mod` | Go 1.21 toolchain, module `go.k6.io/k6` |
| `lib/executor/execution_config_shortcuts.go` | `getRampingVUsScenario()` shortcut derivation |

### 0.2.2 Integration Point Discovery

- **Scheduler → Executor**: `execution/scheduler.go` calls `executor.Run(executorsRunCtx, engineOut)` per executor, passing a cancellable context derived from `runCtx`
- **Executor → VU Buffer**: `rampingVUsRunState.runLoopsIfPossible()` calls `executionState.GetPlannedVU()` to borrow VUs from the shared channel, and `executionState.ReturnVU()` to return them
- **VU Handle → VU State**: `vuHandle.start()` activates a VU via `initVU.Activate(getVUActivationParams(vh.ctx, ...))`, binding the VU's execution context to `vh.ctx`
- **Context Cancellation Chain**: `ctx` (parent/ctrl+c) → `maxDurationCtx` (executor lifetime) → `regularDurationCtx` (stages duration) → `vuHandle.ctx` (per-VU)
- **Metrics Emission**: `executionState.ModCurrentlyActiveVUsCount()` is modified via atomics in `getVU`/`returnVU` closures and reported by `Scheduler.emitVUsAndVUsMax()`

### 0.2.3 New File Requirements

Per the `SWE-AtlasQnA-Repo` implementation rule, exactly one new file is required:

| File to Create | Purpose |
|----------------|---------|
| `blitzy/documentation/k6_ddc3b0b1d23c.md` | Comprehensive Q&A document answering all user questions about concurrency in the ramping-vus executor, with code-grounded rationale and trace-throughs |

No other source files, test files, or configuration files will be created or modified.


## 0.3 Dependency Inventory

### 0.3.1 Private and Public Packages

Since this is a code investigation task (not a code modification task), no new dependencies are introduced. However, the following packages are critical to understanding the concurrency model under investigation:

| Registry | Package | Version | Purpose in Investigation |
|----------|---------|---------|--------------------------|
| Go module | `go.k6.io/k6/lib` | v0.55.0 (in-repo) | `ExecutionState`, `ExecutionSegment`, `SegmentedIndex`, `ExecutionTuple` — core execution and segmentation primitives |
| Go module | `go.k6.io/k6/lib/executor` | v0.55.0 (in-repo) | `RampingVUs`, `vuHandle`, `rampingVUsRunState` — the executor and VU state machine under investigation |
| Go module | `go.k6.io/k6/execution` | v0.55.0 (in-repo) | `Scheduler`, `testAbortController` — scheduler coordination and abort propagation |
| Go module | `go.k6.io/k6/metrics` | v0.55.0 (in-repo) | `SampleContainer` — metric emission interface passed through executor `Run()` |
| Go stdlib | `sync` | go1.21 | `sync.Mutex` used in `vuHandle` for state transitions |
| Go stdlib | `sync/atomic` | go1.21 | Atomic state reads in `vuHandle.runLoopsIfPossible()` fast path, active VU counters in `ExecutionState` |
| Go stdlib | `context` | go1.21 | `context.WithCancel`, `context.WithDeadline` — the entire cancellation propagation chain under investigation |
| Go stdlib | `math/big` | go1.21 | `big.Rat` — rational arithmetic for execution segment scaling |
| External | `github.com/sirupsen/logrus` | v1.9.3 | Structured logging in executor and VU handle |
| External | `gopkg.in/guregu/null.v3` | v3.5.0 | Nullable config fields (`null.Int`, `null.String`) in `RampingVUsConfig` |
| External | `go.k6.io/k6/lib/types` | v0.55.0 (in-repo) | `NullDuration` for stage durations and graceful stop/rampdown periods |

### 0.3.2 Dependency Updates

No dependency updates are required. This task produces only a documentation artifact. The Go toolchain specified in `go.mod` is:

```
go 1.21
toolchain go1.21.13
```

All analysis is based on the existing codebase at the `k6_ddc3b0b1d23c` branch head.


## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

The investigation does not modify any code. However, the following touchpoints form the critical concurrency interaction chain that the Q&A document must trace through:

**Context cancellation propagation chain (Ctrl+C path):**

- `execution/scheduler.go` — `Scheduler.Run()` receives `runCtx` from the CLI layer; Ctrl+C cancels `runCtx`
- `execution/scheduler.go` — `executorsRunCtx, executorsRunCancel := context.WithCancel(withExecStateCtx)` — per-executor context is a child of `runCtx`
- `lib/executor/helpers.go` — `getDurationContexts(ctx, regularDuration, gracefulStop)` creates `maxDurationCtx` (child of `executorsRunCtx`) and `regDurationCtx` (child of `maxDurationCtx`)
- `lib/executor/ramping_vus.go` — `RampingVUs.Run()` passes `maxDurationCtx` to `runLoopsIfPossible()` as the VU handle parent context
- `lib/executor/vu_handle.go` — `newStoppedVUHandle(ctx=maxDurationCtx, ...)` creates `vh.ctx, vh.cancel = context.WithCancel(parentCtx)` — VU context is a child of `maxDurationCtx`

**Dual-handler step processing chain:**

- `lib/executor/ramping_vus.go` — `RampingVUs.Init()` precalculates `rawSteps` (from `getRawExecutionSteps(et, true)`) and `gracefulSteps` (from `GetExecutionRequirements(et)`)
- `lib/executor/ramping_vus.go` — `iterateSteps()` iterates both step slices by `TimeOffset`, calling `handleNewScheduledVUs` for raw steps and `handleNewMaxAllowedVUs` for graceful steps — **these are called sequentially in a single goroutine, not concurrently**
- `lib/executor/ramping_vus.go` — After `iterateSteps()` returns, `runRemainingGracefulSteps()` is launched as a **separate goroutine** that continues processing only graceful steps via `handleNewMaxAllowedVUs`

**VU lifecycle chain:**

- `lib/executor/ramping_vus.go` — `runLoopsIfPossible()` creates `maxVUs` number of `vuHandle` instances, each with its own goroutine running `vuHandle.runLoopsIfPossible(runIteration)`
- `lib/executor/vu_handle.go` — `start()` transitions from `stopped` → `starting` (acquires VU from buffer), `gracefulStop()` transitions `running` → `toGracefulStop`, `hardStop()` transitions `running`/`toGracefulStop` → `toHardStop`
- `lib/execution.go` — `GetPlannedVU()` reads from `vus` channel with timeout/retry; `ReturnVU()` writes back to `vus` channel

**Execution segment scaling chain:**

- `lib/execution_segment.go` — `NewExecutionTuple(segment, &sequence)` creates a `ExecutionSegmentSequenceWrapper` with precomputed striped offsets
- `lib/execution_segment.go` — `ExecutionTuple.ScaleInt64(value)` uses `ExecutionSegmentSequenceWrapper.ScaleInt64(segmentIndex, value)` for proportional distribution
- `lib/executor/ramping_vus.go` — `getRawExecutionSteps()` uses `NewSegmentedIndex(et)` and `index.GoTo()`, `index.Next()`, `index.Prev()` to compute per-segment step plans

### 0.4.2 Concurrency Synchronization Points

The Q&A document must analyze these synchronization mechanisms:

| Mechanism | Location | Purpose |
|-----------|----------|---------|
| `vuHandle.mutex` (sync.Mutex) | `lib/executor/vu_handle.go` | Protects all state transitions in `start()`, `gracefulStop()`, `hardStop()`, and the slow path of `runLoopsIfPossible()` |
| `atomic.LoadInt32(&vh.state)` | `lib/executor/vu_handle.go:204` | Lock-free fast-path read in `runLoopsIfPossible()` — if state is `running`, proceed without mutex |
| `canStartIter` channel | `lib/executor/vu_handle.go` | Signal from `start()` to the VU goroutine that it can begin iterating; recreated on each `gracefulStop()`/`hardStop()` |
| `executorDone` channel | `lib/executor/vu_handle.go:197` | `vh.parentCtx.Done()` — signals total executor shutdown to VU goroutines |
| `context.WithCancel(parentCtx)` | `lib/executor/vu_handle.go:96-97` | Creates VU-scoped context; cancelled on `gracefulStop`(in slow path) and `hardStop` |
| `sync.WaitGroup` (rs.wg) | `lib/executor/ramping_vus.go:571` | Tracks active VU goroutines; `wg.Wait()` deferred in `Run()` before context cancel |
| `atomic` counters | `lib/execution.go` | `activeVUs`, `initializedVUs`, `fullIterationsCount`, `interruptedIterationsCount` — UI/metrics only, explicitly documented as NOT for synchronization |


## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

Since this is a Q&A investigation task governed by the `SWE-AtlasQnA-Repo` rule, the only file action is creating the documentation artifact. No existing files are modified.

**Group 1 — Documentation Output:**

- **CREATE: `blitzy/documentation/k6_ddc3b0b1d23c.md`** — The comprehensive Q&A markdown document. Must contain:
  - Trace-through of the `vuHandle` state machine and its five states
  - Analysis of the dual-handler architecture in `iterateSteps()` and `runRemainingGracefulSteps()`
  - Context cancellation propagation diagram from Ctrl+C through all layers
  - Execution segment scaling analysis with the striping algorithm
  - VU buffer lifecycle trace (`GetPlannedVU` → `Activate` → `ReturnVU`)
  - Verdict on each user-reported symptom (bug, by-design, or misconfiguration)

### 0.5.2 Implementation Approach

The Q&A document will be structured to address each user concern as a distinct section, with the following analytical methodology applied to each:

- **Establish the code facts**: Cite specific file paths, function names, and line ranges from the analyzed source
- **Trace the execution path**: Walk through what happens step-by-step when the described scenario occurs
- **Identify the root cause**: Determine whether the observed behavior is a bug, by-design behavior, or user misconfiguration
- **Provide the rationale**: Explain WHY the code behaves as it does, referencing design comments and the state transition table in `vu_handle.go`

**Key findings that the document will present:**

- The two handler closures (`scheduledVUsHandlerStrategy` and `maxAllowedVUsHandlerStrategy`) are NOT concurrent during the main stage processing — they are called sequentially within `iterateSteps()`. The "mismatch" the user observes is by design: the scheduled handler tracks the target number of _actively running_ VUs, while the graceful handler tracks the _maximum allowed_ VUs including those in graceful ramp-down. These are intentionally different values.

- The VU "stuck" state is likely the `toGracefulStop` state, which is a legitimate transitional state where the VU is allowed to finish its current iteration before stopping. The VU is not idle — it is completing work. The state machine explicitly handles the race between `start()` and this state by transitioning directly to `running` (line 123-125 of `vu_handle.go`).

- For Ctrl+C behavior: the deferred execution order in `Run()` is critical. `defer runState.wg.Wait()` (added last) executes FIRST (Go LIFO defer), meaning the function waits for all VU goroutines to finish before calling `cancel()`. Since `vh.ctx` is a child of `maxDurationCtx` which is a child of the parent context, Ctrl+C propagates immediately through the context chain, so VU contexts DO get cancelled. However, if `vu.RunOnce()` blocks on I/O or sleep and doesn't check its context, the VU will appear to "keep running" — this is a script-level issue, not a k6 bug.

- For execution segment imbalance: this depends critically on whether the user provides a `--execution-segment-sequence` alongside `--execution-segment`. Without the sequence, each instance uses `ExecutionSegment.Scale()` independently, which uses a rounding approach that can cause the sum to exceed the total. With a proper sequence, the striping algorithm in `NewExecutionSegmentSequenceWrapper()` guarantees non-overlapping, sum-preserving distribution — verified by `TestSumRandomSegmentSequenceMatchesNoSegment`.

- The VU buffer (`vus chan InitializedVU` in `ExecutionState`) has a 1:1 `getVU`/`returnVU` contract enforced by the `vuHandle` lifecycle. Each call to `getVU` increments `wg` and active VU count; each call to `returnVU` writes the VU back to the channel and decrements both. The `TestVUHandleRace` test explicitly verifies `getVUCount == returnVUCount` at line 110 of `vu_handle_test.go`.

### 0.5.3 Document Structure

The generated markdown document (`k6_ddc3b0b1d23c.md`) will follow this structure:

- **Introduction and Scope**: State the questions being investigated
- **Q1: VU Stuck State Analysis**: State machine trace, `toGracefulStop` explanation
- **Q2: Handler VU Count Mismatch**: Dual-counter architecture, `iterateSteps()` sequential execution
- **Q3: Ctrl+C / gracefulStop Timeout Exceeded**: Context propagation trace, defer ordering, script-level considerations
- **Q4: Execution Segment VU Imbalance**: Segment-only vs. segment+sequence analysis, striping algorithm correctness
- **Q5: Race Condition Between Handlers**: Sequential execution proof, mutex synchronization analysis
- **Q6: VU Buffer Leak Assessment**: `getVU`/`returnVU` lifecycle, channel buffer contract
- **Summary of Findings**: Verdict table (bug / by-design / misconfiguration per symptom)


## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

**Source files analyzed for the investigation (read-only):**

- `lib/executor/ramping_vus.go` — Primary executor under investigation: `RampingVUs.Run()`, `iterateSteps()`, `runRemainingGracefulSteps()`, handler strategies
- `lib/executor/vu_handle.go` — VU state machine: 5-state FSM, `start()`, `gracefulStop()`, `hardStop()`, `runLoopsIfPossible()`
- `lib/execution.go` — `ExecutionState`, VU buffer (`vus` channel), `GetPlannedVU()`, `ReturnVU()`, atomic counters
- `lib/execution_segment.go` — `ExecutionSegment`, `ExecutionSegmentSequence`, striping algorithm, `ScaleInt64()`, `SegmentedIndex`
- `lib/executor/helpers.go` — `getDurationContexts()`, `getIterationRunner()`, `getVUActivationParams()`, `handleInterrupt()`
- `lib/executor/base_config.go` — `DefaultGracefulStopValue`, `BaseConfig`, `GetGracefulStop()`
- `lib/executor/base_executor.go` — `BaseExecutor`, `GetLogger()`, `GetProgress()`
- `lib/executors.go` — `ExecutorConfig`, `ExecutionStep`, `ScenarioConfig`
- `lib/runner.go` — `VU`, `ActiveVU`, `InitVU` interfaces
- `lib/helpers.go` — `StrictJSONUnmarshal`, `GetEndOffset`
- `execution/scheduler.go` — `Scheduler.Run()`, `runExecutor()`, context chain from `globalCtx` → `runCtx` → `executorsRunCtx`
- `execution/abort.go` — `testAbortController`, `NewTestRunContext()`

**Test files analyzed for correctness evidence:**

- `lib/executor/ramping_vus_test.go` — `TestRampingVUsRun`, `TestRampingVUsGracefulStopWaits`, `TestRampingVUsGracefulStopStops`, `TestRampingVUsGracefulRampDown`, `TestRampingVUsHandleRemainingVUs`, `TestRampingVUsRampDownNoWobble`, segment sum tests
- `lib/executor/vu_handle_test.go` — `TestVUHandleRace` (concurrent -race validation), `TestVUHandleStartStopRace`, `TestVUHandleSimple`
- `lib/executor/common_test.go` — Shared helpers: `setupExecutor()`, mock VU implementations

**Configuration and dependency manifests:**

- `go.mod` — Go 1.23, dependency versions
- `lib/executor/execution_config_shortcuts.go` — DSL builder helpers for executor configs

**Documentation output:**

- `blitzy/documentation/k6_ddc3b0b1d23c.md` — The generated Q&A document (CREATE)

### 0.6.2 Explicitly Out of Scope

- **Other executor types**: `constant_vus.go`, `per_vu_iterations.go`, `shared_iterations.go`, `externally_controlled.go`, `constant_arrival_rate.go`, `ramping_arrival_rate.go` — not relevant to the ramping-VU concurrency investigation
- **Metrics subsystem** (`metrics/`): VU metrics collection is not implicated in the reported symptoms
- **JavaScript runtime** (`js/`): The user's concern is about executor-level VU lifecycle management, not script execution internals
- **HTTP/transport layer** (`lib/netext/`): Network behavior is not part of this investigation
- **Output/reporting** (`output/`): Reporting subsystem is downstream from the executor and not relevant
- **CLI layer** (`cmd/`): Command-line parsing and option handling are not under investigation
- **Cloud integration** (`cloudapi/`): Not implicated in local execution behavior
- **Code modifications**: Per the `SWE-AtlasQnA-Repo` rule, zero modifications to existing source files are permitted
- **New code additions**: No code is to be added to the source repository other than the requested documentation file
- **Performance optimization**: The scope is limited to analysis and documentation, not code fixes
- **Refactoring**: No refactoring of any kind is in scope


## 0.7 Rules for Feature Addition

### 0.7.1 SWE-AtlasQnA-Repo Rule

The governing implementation rule for this project is:

- **Create a new markdown document** named `k6_ddc3b0b1d23c.md` (derived from the source branch name) that comprehensively answers the questions posed in the prompt
- **Provide thinking / rationale** behind the answers — do not simply state conclusions but walk through the code paths that lead to each answer
- **Do not make assumptions** — base all answers on the code as the truth. Every claim must be traceable to a specific file, function, or line range
- **Do not modify any existing files** in the source repository
- **Do not add any other code** in the source repository besides the requested document
- **Place the document** in the `blitzy/documentation` directory in the destination repo

### 0.7.2 Investigation-Specific Rules

- **Evidence-based analysis only**: Every finding must cite specific source file paths and function names. No speculation about "typical behavior" of Go goroutines in general — only what this codebase actually does.
- **Address all six symptoms**: The user reported six distinct concerns (stuck VUs, handler count mismatch, Ctrl+C VU persistence, execution segment imbalance, handler race condition, VU buffer leak). Each must receive a dedicated, complete analysis.
- **Distinguish verdicts clearly**: Each symptom must be classified as one of: confirmed bug, by-design behavior, or likely user misconfiguration — with the code evidence supporting that classification.
- **Trace through code paths**: Use the actual function call chains, context inheritance hierarchies, and state transitions from the source. Do not paraphrase behavior — show the execution flow.
- **Preserve code fidelity**: When referencing code constructs (struct names, function signatures, channel operations), use the exact names from the source: `rampingVUsRunState`, `vuHandle`, `scheduledVUsHandlerStrategy`, `maxAllowedVUsHandlerStrategy`, `iterateSteps`, `runRemainingGracefulSteps`, `runLoopsIfPossible`, `GetPlannedVU`, `ReturnVU`, `ScaleInt64`, `SegmentedIndex`.
- **Temporary artifacts**: The user mentioned "temporary investigation scripts are fine but clean up after." Since the `SWE-AtlasQnA-Repo` rule forbids adding code, the documentation artifact itself fulfills this requirement by providing a complete trace-through without needing executable scripts.


## 0.8 References

### 0.8.1 Files and Folders Searched

The following source files were read in full and form the evidence base for all conclusions in this Agent Action Plan:

| File Path | Purpose in Investigation |
|-----------|------------------------|
| `lib/executor/ramping_vus.go` | Primary executor: `Run()`, step processing, handler strategies, `rampingVUsRunState` |
| `lib/executor/vu_handle.go` | VU state machine: 5-state FSM, `start()`/`gracefulStop()`/`hardStop()`, `runLoopsIfPossible()` |
| `lib/execution.go` | `ExecutionState`: VU buffer channel, `GetPlannedVU()`, `ReturnVU()`, atomic counters |
| `lib/execution_segment.go` | Segment math: `ExecutionSegment`, `ExecutionSegmentSequence`, striping, `ScaleInt64()`, `SegmentedIndex` |
| `lib/executor/helpers.go` | Context creation (`getDurationContexts`), VU activation params, interrupt handling |
| `lib/executor/base_config.go` | Default graceful stop value (30s), `BaseConfig` struct |
| `lib/executor/base_executor.go` | `BaseExecutor` struct, logger and progress accessors |
| `lib/executors.go` | `ExecutorConfig` interface, `ExecutionStep` struct, scenario config |
| `lib/runner.go` | `VU`, `ActiveVU`, `InitVU` interfaces |
| `lib/helpers.go` | JSON parsing, `GetEndOffset` |
| `lib/executor/execution_config_shortcuts.go` | DSL builder helpers for executor configurations |
| `execution/scheduler.go` | `Scheduler.Run()`, `runExecutor()`, context chain management |
| `execution/abort.go` | `testAbortController`, `NewTestRunContext()` |
| `lib/executor/ramping_vus_test.go` | Executor tests: graceful stop, ramp-down, segment sum verification |
| `lib/executor/vu_handle_test.go` | VU handle race tests, state transition verification |
| `lib/executor/common_test.go` | Test infrastructure: mock VUs, executor setup helpers |
| `go.mod` | Go 1.23, project dependency versions |

The following folders were explored via `get_source_folder_contents`:

| Folder Path | Purpose |
|-------------|---------|
| `` (root) | Top-level repository structure discovery |
| `lib` | Shared core package structure: executors, state, segments |
| `lib/executor` | All executor implementations and VU handle |
| `execution` | Runtime scheduler, abort controller |

### 0.8.2 Tech Spec Sections Consulted

| Section | Purpose |
|---------|---------|
| `1.1 Executive Summary` | Overall project context and architecture overview |
| `2.1 Feature Catalog` | Feature inventory for understanding scope boundaries |

### 0.8.3 Attachments and External Resources

- No Figma designs were provided or applicable to this investigation
- No external URLs were provided by the user
- No environment files were attached
- The user provided zero attachments for this project

### 0.8.4 Branch and Output Information

- **Source branch**: `k6_ddc3b0b1d23c`
- **Output document**: `blitzy/documentation/k6_ddc3b0b1d23c.md`
- **Repository root**: `/tmp/blitzy/k6/k6_ddc3b0b1d23c_a98e4f`


