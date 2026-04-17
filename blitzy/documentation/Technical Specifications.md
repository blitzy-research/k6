# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to **produce a comprehensive investigative markdown document** that traces, analyzes, and answers specific concurrency-related questions about k6's `ramping-vus` executor. The document must be created as `k6_ddc3b0b1d23c.md` and placed in the `blitzy/documentation/` directory. No existing source files may be modified.

The user's investigation centers on the following concrete questions:

- **VU State Inconsistency**: When stages ramp up and down rapidly with a long `gracefulRampDown`, VUs appear to enter a limbo state—neither fully active nor fully stopped. The user has observed that the VU count tracked by the "scheduled handler" (`scheduledVUsHandlerStrategy`) diverges from what the "graceful handler" (`maxAllowedVUsHandlerStrategy`) believes should exist.
- **Ctrl+C / gracefulStop Overrun**: Upon sending SIGINT (Ctrl+C), some VUs continue executing for longer than the configured `gracefulStop` period should permit.
- **Execution Segment VU Overcounting**: When running with three execution segment instances (e.g., `0:1/3`, `1/3:2/3`, `2/3:1`), one instance consistently reports more VUs than the others at the same timestamp, and summing all instances' VU counts exceeds the configured maximum.
- **Race Condition Hypothesis**: Whether a race condition exists between the two handler goroutines (`handleNewMaxAllowedVUs` and `handleNewScheduledVUs`) that both manipulate VU handle state.
- **VU Buffer Leak Hypothesis**: Whether the shared VU channel buffer in `ExecutionState.vus` can leak VUs under concurrent access.

Implicit requirements surfaced from analysis:

- The answer must trace through actual source code paths, citing specific files and line numbers
- The answer must distinguish between design-by-contract behavior (e.g., documented "IMPORTANT: for UI/information purposes only" warnings on counters) and genuine bugs
- The analysis must cover both the mathematical/scheduling layer (`getRawExecutionSteps`, `reserveVUsForGracefulRampDowns`, `GetExecutionRequirements`) and the runtime/goroutine layer (`vuHandle` state machine, `runLoopsIfPossible`, `iterateSteps`)
- Temporary investigation scripts may be created but must be cleaned up; per the project rules, however, no code may be added to the source repository at all—only the markdown document

### 0.1.2 Special Instructions and Constraints

- **Implementation Rule "SWE-AtlasQnA-Repo"**: Create a new markdown document named `k6_ddc3b0b1d23c.md` that comprehensively answers the questions posed. Build and run the source code to analyze behavior as needed. Base answers on the code as truth. Do not modify any existing files. Do not add any other code besides the requested document. Place the document in the `blitzy/documentation` directory in the destination repo.
- **No Source Modifications**: The investigation must be purely analytical—reading, building, and running tests against the existing codebase without modifying it
- **Clean-Up Obligation**: The user mentioned "Temporary investigation scripts are fine but clean up after"; given the SWE-AtlasQnA-Repo rule, no scripts should be added at all
- **Evidence-Based Analysis**: All claims about race conditions, VU leaks, or behavior must be grounded in specific code references from the repository

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To **answer the VU state inconsistency question**, we will trace the interaction between `iterateSteps()` (in `lib/executor/ramping_vus.go:622-645`), `scheduledVUsHandlerStrategy()` (lines 679-690), and `maxAllowedVUsHandlerStrategy()` (lines 668-677), analyzing how the two step sequences (`rawSteps` and `gracefulSteps`) are interleaved by time offset and how the `vuHandle` state machine transitions mediate concurrent `start()`, `gracefulStop()`, and `hardStop()` calls.
- To **answer the Ctrl+C / gracefulStop overrun question**, we will trace from the signal handler in `cmd/run.go:347-364` through `execution.AbortTestRun()` in `execution/abort.go:64-72`, into how context cancellation propagates to the `maxDurationCtx` used by `getDurationContexts()` in `lib/executor/helpers.go:168-180`, and finally to how the `vuHandle.runLoopsIfPossible()` loop (in `lib/executor/vu_handle.go:185-264`) observes context cancellation via the `executorDone` channel.
- To **answer the execution segment overcounting question**, we will analyze `ExecutionSegmentSequenceWrapper.ScaleInt64()` in `lib/execution_segment.go:579-588`, the `SegmentedIndex` iterator (`GoTo`, `Next`, `Prev` methods at lines 782-842), and the `getRawExecutionSteps()` method with its segmented scaling logic, showing how VU counts are partitioned and whether they are guaranteed to sum to the unsegmented total.
- To **answer the race condition hypothesis**, we will examine the mutex-based synchronization in `vuHandle` (the `mutex` field at line 71) and the atomic state reads in `runLoopsIfPossible` (line 204), documenting the intentional lock-free fast path and the mutex-guarded slow path.
- To **answer the VU buffer leak hypothesis**, we will analyze the `ExecutionState.vus` channel (in `lib/execution.go:106`), `GetPlannedVU()` (lines 471-488), and `ReturnVU()` (lines 544-549) for correctness of the borrow/return protocol.

## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

The investigation scope is read-only analysis (no modifications) of the following files and folders. Each file has been individually retrieved and studied.

**Primary Investigation Files — Ramping VUs Executor:**

| File Path | Purpose | Relevance |
|---|---|---|
| `lib/executor/ramping_vus.go` (712 lines) | Core ramping-vus executor: `RampingVUsConfig`, stage definition, `getRawExecutionSteps()`, `reserveVUsForGracefulRampDowns()`, `GetExecutionRequirements()`, `Run()`, `rampingVUsRunState`, handler strategies, `iterateSteps()`, `runRemainingGracefulSteps()`, `waiter()` | Central to all five user questions |
| `lib/executor/vu_handle.go` (264 lines) | VU lifecycle state machine (`stopped`, `starting`, `running`, `toGracefulStop`, `toHardStop`), `start()`, `gracefulStop()`, `hardStop()`, `runLoopsIfPossible()` | Directly relevant to VU "stuck" state and race condition questions |
| `lib/executor/helpers.go` (264 lines) | Shared utilities: `getDurationContexts()`, `getIterationRunner()`, `trackProgress()`, `handleInterrupt()`, `getVUActivationParams()` | Relevant to gracefulStop timeout and Ctrl+C propagation |
| `lib/executor/base_config.go` (147 lines) | `BaseConfig` with `GracefulStop` field, default 30s value | Configures the graceful stop period |

**Secondary Investigation Files — Execution State and Segments:**

| File Path | Purpose | Relevance |
|---|---|---|
| `lib/execution.go` (549 lines) | `ExecutionState`: VU channel buffer (`vus`), `GetPlannedVU()`, `ReturnVU()`, `GetUnplannedVU()`, active VU counters, pause/resume logic | Relevant to VU buffer leak hypothesis and VU counting discrepancies |
| `lib/execution_segment.go` (842 lines) | `ExecutionSegment`, `ExecutionSegmentSequence`, `ExecutionSegmentSequenceWrapper`, `ExecutionTuple`, `SegmentedIndex`, `ScaleInt64()`, `Next()`, `Prev()`, `GoTo()` | Central to execution segment overcounting question |
| `lib/executors.go` | `ExecutionStep` struct, `ExecutorConfig` interface, `ScenarioConfigs.GetFullExecutionRequirements()` | Defines the step structure used by both handler strategies |
| `lib/helpers.go` (95 lines) | `GetMaxPlannedVUs()`, `GetMaxPossibleVUs()`, `GetEndOffset()` | Computes VU limits from execution plans |

**Tertiary Investigation Files — Scheduler and Abort Logic:**

| File Path | Purpose | Relevance |
|---|---|---|
| `execution/scheduler.go` (591 lines) | `Scheduler.Run()`, `runExecutor()`, `Init()`, context creation, VU initialization, executor orchestration | Relevant to how Ctrl+C propagates through the execution tree |
| `execution/abort.go` (85 lines) | `testAbortController`, `NewTestRunContext()`, `AbortTestRun()`, context cancellation with reason tracking | Relevant to Ctrl+C / gracefulStop overrun analysis |
| `cmd/run.go` | Signal trapping (`gracefulStop`, `onHardStop`), `handleTestAbortSignals()` | Entry point for SIGINT handling |

**Test Files Analyzed:**

| File Path | Purpose | Relevance |
|---|---|---|
| `lib/executor/ramping_vus_test.go` (1201 lines) | `TestRampingVUsRun`, `TestRampingVUsGracefulStopWaits`, `TestRampingVUsGracefulStopStops`, `TestRampingVUsGracefulRampDown`, `TestRampingVUsHandleRemainingVUs`, `TestRampingVUsRampDownNoWobble`, `TestRampingVUsConfigExecutionPlanExample`, `TestRampingVUsExecutionTupleTests`, `TestSumRandomSegmentSequenceMatchesNoSegment` | Validates existing behavior, segment summation correctness, and absence of data races under `-race` |
| `lib/executor/vu_handle_test.go` (415 lines) | `TestVUHandleRace`, `TestVUHandleStartStopRace`, `TestVUHandleSimple` | Validates VU handle state machine correctness and absence of data races |

**Integration Point Discovery:**

- **API endpoints**: No API endpoints connect directly to the ramping-vus executor; the REST API in `api/` exposes test status and pause/resume but does not interact with per-executor VU scheduling
- **Metrics emission**: The `rampingVUsRunState.runLoopsIfPossible()` method triggers VU-level iteration counting via `getIterationRunner()` which calls `executionState.AddFullIterations()` and `AddInterruptedIterations()`
- **Context hierarchy**: The context chain flows from `cmd/run.go` → `execution/scheduler.go:Run()` → per-executor `runCtx` → `getDurationContexts()` → `maxDurationCtx` / `regDurationCtx` → per-VU `vuHandle.ctx`

### 0.2.2 Web Search Research Conducted

No web search was required for this investigation. All questions can be answered definitively from the source code, which serves as the single source of truth per the project rules. The k6 executor architecture, VU handle state machine, and execution segment math are entirely self-contained within the repository.

### 0.2.3 New File Requirements

**Single New File to Create:**

| File Path | Purpose |
|---|---|
| `blitzy/documentation/k6_ddc3b0b1d23c.md` | Comprehensive markdown document answering all concurrency-related questions about the ramping-vus executor |

No new source files, test files, or configuration files are required. The project rules explicitly prohibit adding any code to the source repository beyond the requested documentation.

## 0.3 Dependency Inventory

### 0.3.1 Private and Public Packages

The following packages are relevant to the ramping-vus executor concurrency investigation, drawn from `go.mod` and the source imports:

| Registry | Package | Version | Purpose |
|---|---|---|---|
| Go standard library | `sync` | Go 1.21 | `sync.Mutex` used in `vuHandle` for state machine synchronization |
| Go standard library | `sync/atomic` | Go 1.21 | Atomic state reads/writes in `vuHandle.changeState()`, `ExecutionState` counters |
| Go standard library | `context` | Go 1.21 | Context cancellation chain from scheduler through VU handles |
| Go standard library | `time` | Go 1.21 | `time.Timer` in `waiter()`, `time.After` in `GetPlannedVU()` retries |
| Go standard library | `math/big` | Go 1.21 | Rational number arithmetic in `ExecutionSegment` and `SegmentedIndex` |
| Go module | `github.com/sirupsen/logrus` | v1.9.3 | Structured logging throughout executor and VU handle code |
| Go module | `gopkg.in/guregu/null.v3` | v3.5.0 | Nullable types for config fields (`StartVUs`, stage `Target`) |
| Go module | `go.k6.io/k6/lib/types` | internal | `NullDuration` used for `GracefulRampDown`, `GracefulStop` |
| Go module | `go.k6.io/k6/metrics` | internal | `SampleContainer` interface for metrics emission |
| Go module | `go.k6.io/k6/ui/pb` | internal | Progress bar rendering during ramp stages |
| Go module (test) | `github.com/stretchr/testify` | v1.9.0 | `assert` and `require` packages used in all test files |

### 0.3.2 Dependency Updates

No dependency updates are required for this investigation. The task produces only a markdown document and does not modify any source code, configuration, or build files. All dependencies remain at their current versions as specified in `go.mod` and `go.sum`.

## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

The investigation document must trace through the following integration touchpoints to answer the user's questions. No modifications are made; these are read-only analysis targets.

**VU Handle State Machine Integration (Question: VU "stuck" state)**

- `lib/executor/vu_handle.go` — The `vuHandle` struct exposes three mutation methods (`start()`, `gracefulStop()`, `hardStop()`) and one long-running loop (`runLoopsIfPossible()`). The state machine has five states (`stopped`, `starting`, `running`, `toGracefulStop`, `toHardStop`) with documented transitions in the comment table at lines 24-55.
- `lib/executor/ramping_vus.go:679-690` — `scheduledVUsHandlerStrategy()` calls `vuHandle.start()` and `vuHandle.gracefulStop()` based on `rawSteps` PlannedVUs values. It maintains a local `cur` counter for currently scheduled VUs.
- `lib/executor/ramping_vus.go:668-677` — `maxAllowedVUsHandlerStrategy()` calls `vuHandle.hardStop()` based on `gracefulSteps` PlannedVUs values. It maintains a separate local `cur` counter for the graceful VU ceiling.
- `lib/executor/ramping_vus.go:622-645` — `iterateSteps()` interleaves `rawSteps` and `gracefulSteps` by time offset, dispatching to the appropriate handler. When `g.TimeOffset < r.TimeOffset`, the graceful handler fires first; otherwise the scheduled handler fires.

**Context Cancellation Chain (Question: Ctrl+C overrun)**

- `cmd/run.go:347-364` — The `gracefulStop` function calls `runAbort()` which invokes `execution.AbortTestRun()`, cancelling the `runCtx`
- `execution/abort.go:24-36` — `testAbortController.abort()` acquires a mutex, stores the abort reason, and calls `cancel()` on the run context
- `execution/scheduler.go:497` — `executorsRunCtx` is derived from `runCtx`; its cancellation propagates to all executor `Run()` calls
- `lib/executor/helpers.go:168-180` — `getDurationContexts()` creates `maxDurationCtx` (with deadline = start + regularDuration + gracefulStop) as a child of `parentCtx` (the executor's `runCtx`). When `runCtx` is cancelled by Ctrl+C, `maxDurationCtx` is also immediately cancelled regardless of its remaining deadline.
- `lib/executor/vu_handle.go:197` — `executorDone = vh.parentCtx.Done()` — the VU handle monitors the parent context (which is `maxDurationCtx`). When this is cancelled, the loop exits at line 212.

**Execution Segment Scaling Integration (Question: VU overcounting)**

- `lib/execution_segment.go:579-588` — `ScaleInt64()` computes how many VUs a given segment "owns" out of a global total. The algorithm uses the segment's striped offsets and the LCD of the sequence to ensure additive correctness: `sum(ScaleInt64(i, value) for all segments i) == value`.
- `lib/execution_segment.go:782-842` — `SegmentedIndex.Next()`, `Prev()`, and `GoTo()` implement the segmented iterator that determines exactly when each VU should be added or removed for a given segment.
- `lib/executor/ramping_vus.go:171-234` — `getRawExecutionSteps()` uses `SegmentedIndex` to produce segment-specific execution step sequences. Each segment gets a different number of VUs at potentially different time offsets.
- `lib/executor/ramping_vus_test.go:1112-1201` — `TestSumRandomSegmentSequenceMatchesNoSegment` is a randomized property test verifying that the sum of per-segment VU counts matches the unsegmented total at every time offset.

**VU Buffer Channel Integration (Question: VU buffer leak)**

- `lib/execution.go:106` — `vus chan InitializedVU` is a buffered channel of size `maxPossibleVUs`
- `lib/execution.go:471-488` — `GetPlannedVU()` receives from the channel with a timeout-and-retry loop
- `lib/execution.go:544-549` — `ReturnVU()` sends the VU back into the channel
- `lib/executor/ramping_vus.go:592-617` — `runLoopsIfPossible()` sets up `getVU` (which calls `GetPlannedVU`) and `returnVU` (which calls `ReturnVU`), passing them to each `vuHandle`. The VU is acquired in `vuHandle.start()` at line 128 and returned via the `returnVU` callback registered in `getVUActivationParams()` as the `DeactivateCallback`.

### 0.4.2 Cross-Goroutine Communication Map

The ramping-vus executor spawns and coordinates the following goroutines during `Run()`:

```mermaid
graph TD
    A["Run() goroutine<br/>(main executor thread)"] -->|spawns maxVUs| B["vuHandle.runLoopsIfPossible()<br/>(one per VU slot)"]
    A -->|calls iterateSteps()| C["iterateSteps loop<br/>(interleaves raw + graceful steps)"]
    C -->|dispatches to| D["scheduledVUsHandlerStrategy()<br/>(calls start/gracefulStop)"]
    C -->|dispatches to| E["maxAllowedVUsHandlerStrategy()<br/>(calls hardStop)"]
    A -->|go runRemainingGracefulSteps()| F["runRemainingGracefulSteps()<br/>(handles remaining graceful steps)"]
    A -->|go trackProgress()| G["trackProgress()<br/>(monitors context completion)"]
    D -->|mutex-guarded| B
    E -->|mutex-guarded| B
    H["Ctrl+C / SIGINT"] -->|cancels runCtx| A
    H -->|propagates to maxDurationCtx| B
```

Key synchronization primitives:
- **`vuHandle.mutex`**: Guards all state transitions in `start()`, `gracefulStop()`, `hardStop()`, and the slow path of `runLoopsIfPossible()`
- **`vuHandle.canStartIter` channel**: Signals the run loop that it may begin iterating; closed by `start()` and recreated by `gracefulStop()`/`hardStop()`
- **Atomic state read**: The fast path in `runLoopsIfPossible()` at line 204 reads `vh.state` atomically without the mutex for performance
- **`waiter()` function**: Uses `time.Timer` and context cancellation to sleep until the next step's time offset

## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

Since this task produces only a single documentation file per the SWE-AtlasQnA-Repo rules, the execution plan is structured around the analysis activities that produce the document content, and the single file creation.

**Group 1 — Analysis Activities (read-only, no file changes):**

- READ: `lib/executor/ramping_vus.go` — Trace the `Run()` method control flow, the `iterateSteps()` interleaving logic, and both handler strategy closures to determine whether the two local `cur` variables can diverge from actual VU handle state
- READ: `lib/executor/vu_handle.go` — Analyze the state machine transition table for completeness and race-safety; verify that the `toGracefulStop` state correctly models a VU that is "winding down" without being fully stopped; confirm that the atomic fast-path read at line 204 is safe given that all state writes go through `changeState()` which uses `atomic.StoreInt32`
- READ: `lib/execution_segment.go` — Verify the additive property of `ScaleInt64()` by tracing through the `GetStripedOffsets()` algorithm and confirming that offsets partition the LCD cycle exactly
- READ: `lib/execution.go` — Analyze the `vus` channel protocol for correctness: every `GetPlannedVU()` call must be balanced by exactly one `ReturnVU()` call, and the channel must not grow unbounded
- READ: `execution/abort.go` + `cmd/run.go` — Trace context cancellation from SIGINT through to VU handle loop termination
- RUN: Execute `go test -v -race ./lib/executor/ -run TestVUHandle` to confirm no data races are detected by the Go race detector
- RUN: Execute `go test -v -race ./lib/executor/ -run TestSumRandomSegmentSequenceMatchesNoSegment` to confirm segment math additive property holds
- RUN: Execute `go test -v -race ./lib/executor/ -run TestRampingVUs` to confirm existing test suite passes under the race detector

**Group 2 — Document Creation (single new file):**

- CREATE: `blitzy/documentation/k6_ddc3b0b1d23c.md` — Comprehensive investigation document with the following structure:
  - Introduction and question summary
  - Architecture overview of the ramping-vus executor
  - Detailed analysis for each of the five user questions
  - Code-referenced evidence for each conclusion
  - Summary of findings and recommendations

### 0.5.2 Implementation Approach

The investigation document will be assembled by systematically answering each question with evidence from the code:

**Q1: VU State Inconsistency Between Handlers**

The document will explain that the `scheduledVUsHandlerStrategy` and `maxAllowedVUsHandlerStrategy` maintain independent local `cur` counters (lines 669, 680 of `ramping_vus.go`) by design. The scheduled handler tracks the "active target" VU count derived from `rawSteps`, while the graceful handler tracks the "maximum allowed" VU count derived from `gracefulSteps`. These are intentionally different—`gracefulSteps` always has VU counts >= `rawSteps` at any given time offset, because `reserveVUsForGracefulRampDowns()` prevents VU counts from decreasing immediately during ramp-downs. The apparent "mismatch" the user observes is the intended graceful ramp-down reservation: VUs in `toGracefulStop` state are still counted by the graceful handler but not by the scheduled handler. This is not a bug.

The document will also note that the `toGracefulStop` state (line 20 of `vu_handle.go`) is a legitimate intermediate state where a VU has been asked to stop gracefully but is still finishing its current iteration. From the perspective of the scheduled handler, this VU is "stopped"; from the perspective of the graceful handler, it is still "reserved". This duality is by design—see the ASCII art diagram in `ramping_vus.go` lines 278-289 showing dots (`.`) for VUs in graceful wind-down.

**Q2: Ctrl+C / gracefulStop Overrun**

The document will trace the cancellation chain and explain that when SIGINT arrives, `cmd/run.go:352` calls `runAbort()` which cancels `runCtx`. This propagates through `executorsRunCtx` (scheduler.go:497) to `maxDurationCtx` (the parent context of all VU handles). The `vuHandle.runLoopsIfPossible()` loop monitors `executorDone = vh.parentCtx.Done()` (line 197), and when this fires, the loop immediately returns (line 213). The VU's iteration context (`vh.ctx`) is a child of `parentCtx`, so it is also cancelled, causing `vu.RunOnce()` to return with a context error.

However, the document will explain that there is an important subtlety: the `parentCtx` for VU handles is `maxDurationCtx`, not `runCtx` directly. The `maxDurationCtx` has a deadline of `startTime + regularDuration + gracefulStop`. When `runCtx` is cancelled (by Ctrl+C), `maxDurationCtx` is also cancelled immediately because it is a child context—this is standard Go `context.WithDeadline` behavior. So VUs should stop promptly after Ctrl+C. If the user observes VUs running longer, this is likely because the VU's `RunOnce()` call is blocking on network I/O that doesn't respect context cancellation immediately (a known Go HTTP client behavior), or because the `defer runState.wg.Wait()` at line 540 is waiting for goroutines that are still processing.

**Q3: Execution Segment Overcounting**

The document will analyze `ScaleInt64()` and the `SegmentedIndex` iterator, demonstrating mathematically that the striped offset algorithm guarantees additive correctness. The test `TestSumRandomSegmentSequenceMatchesNoSegment` (line 1112) provides a randomized proof of this property. The document will show that if the user is summing VU counts by observing each instance's `activeVUs` counter at "the same timestamp," timing skew between instances (due to clock differences, scheduling jitter, or different network latencies) can cause apparent overcounting even though the mathematical model is correct. The document will also note that the `activeVUsCount` field in `rampingVUsRunState` (line 569) is a local atomic counter used only for progress display—it is incremented in `getVU` and decremented in `returnVU`, and if a VU is in `toGracefulStop` state, it is still counted as "active" in this counter.

**Q4: Race Condition Between Handler Goroutines**

The document will explain that the two handler strategies are NOT run in separate goroutines concurrently. Looking at `iterateSteps()` (lines 622-645), it runs in the main `Run()` goroutine and processes steps sequentially—either the graceful handler or the scheduled handler fires, never both simultaneously. The `runRemainingGracefulSteps()` (line 554) does run in a separate goroutine, but only after `iterateSteps()` has finished processing all raw steps. At that point, the scheduled handler is no longer being called. So there is no concurrent mutation of VU handles from both handlers.

Within each VU handle, `start()`, `gracefulStop()`, and `hardStop()` are each mutex-guarded, and `runLoopsIfPossible()` uses a carefully designed lock-free fast path (atomic read at line 204) combined with a mutex-guarded slow path. The `TestVUHandleRace` test exercises concurrent `start()`/`gracefulStop()`/`hardStop()` calls under the race detector, confirming thread safety.

**Q5: VU Buffer Leak**

The document will trace the VU lifecycle: a VU is borrowed from `ExecutionState.vus` via `GetPlannedVU()` in the `getVU` closure (line 594), and returned via `ReturnVU()` in the `returnVU` closure (line 605). The `returnVU` callback is registered as the `DeactivateCallback` in `getVUActivationParams()` (line 252 of helpers.go). The `vuHandle` ensures a 1:1 correspondence between `getVU` and `returnVU` calls through its state machine: every path that calls `getVU` (in `start()` at line 128) will eventually lead to `returnVU` being called when the VU is deactivated. The `TestVUHandleRace` test at line 110 explicitly asserts `getVUCount == returnVUCount`.

### 0.5.3 Key Code Evidence Summary

The following code patterns will be cited as evidence in the investigation document:

- The `vuHandle` state machine transition table (lines 24-55 of `vu_handle.go`) is the authoritative reference for all possible state transitions
- The `iterateSteps()` method (lines 622-645) processes steps sequentially, not concurrently
- The `reserveVUsForGracefulRampDowns()` algorithm (lines 307-414) deliberately keeps VU counts elevated during ramp-downs
- The `ScaleInt64()` method (lines 579-588) with its `(value / lcd) * len(offsets)` formula guarantees correct partitioning
- The `getDurationContexts()` hierarchy (lines 168-180) ensures context cancellation propagates correctly from parent to children

## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

**Documentation Output:**
- `blitzy/documentation/k6_ddc3b0b1d23c.md` — The sole deliverable

**Source Files to Analyze (read-only, using wildcards where applicable):**
- `lib/executor/ramping_vus.go` — Primary investigation target
- `lib/executor/vu_handle.go` — VU state machine analysis
- `lib/executor/helpers.go` — Context creation and iteration runner
- `lib/executor/base_config.go` — GracefulStop configuration
- `lib/executor/ramping_vus_test.go` — Test evidence for correctness properties
- `lib/executor/vu_handle_test.go` — Test evidence for race-freedom
- `lib/executor/common_test.go` — Shared test fixtures
- `lib/execution.go` — VU buffer and execution state
- `lib/execution_segment.go` — Segment scaling and iterator logic
- `lib/execution_segment_test.go` — Segment math verification tests
- `lib/executors.go` — ExecutionStep definition and scenario aggregation
- `lib/helpers.go` — GetMaxPlannedVUs, GetEndOffset
- `lib/runner.go` — VU lifecycle interfaces
- `execution/scheduler.go` — Scheduler orchestration and context creation
- `execution/abort.go` — Test abort / cancellation mechanism
- `cmd/run.go` — Signal handling entry point

**Test Execution (read-only, for validation):**
- `go test -race ./lib/executor/ -run TestVUHandle*` — Race detector validation
- `go test -race ./lib/executor/ -run TestRampingVUs*` — Executor behavior validation
- `go test -race ./lib/executor/ -run TestSumRandomSegmentSequenceMatchesNoSegment` — Segment math property test

### 0.6.2 Explicitly Out of Scope

- **Arrival-rate executors** (`constant_arrival_rate.go`, `ramping_arrival_rate.go`) — Different execution model, not relevant to ramping-vus investigation
- **Constant VUs executor** (`constant_vus.go`) — No ramping behavior
- **Shared/Per-VU iterations executors** (`shared_iterations.go`, `per_vu_iterations.go`) — Different VU management model
- **Externally controlled executor** (`externally_controlled.go`) — Uses `manualVUHandle` wrapper, separate from the ramping VU control path
- **Cloud/output subsystems** (`output/`, `cloudapi/`) — Not related to VU scheduling
- **JavaScript runtime** (`js/`) — The JS engine's behavior during iteration execution is downstream of the concurrency model
- **Browser module** (`vendor/github.com/grafana/xk6-browser/`) — Separate subsystem
- **Performance optimizations** — The investigation focuses on correctness, not performance
- **Refactoring proposals** — The document only diagnoses and explains; it does not propose code changes
- **Any modification to existing source files** — Explicitly prohibited by project rules

## 0.7 Rules for Feature Addition

### 0.7.1 Project-Specific Rules

The following rules are explicitly enforced by the user-specified implementation rule **"SWE-AtlasQnA-Repo"**:

- **Document-Only Output**: Create a new markdown document named `k6_ddc3b0b1d23c.md` (matching the source branch name) that comprehensively answers the question(s) posed in the prompt
- **Build-and-Run Analysis**: Build and run the source code to analyze the repository behavior as needed; the Go test suite with `-race` flag is the primary diagnostic tool
- **Code-as-Truth**: Do not make assumptions; base all answers on the code as the truth. Every claim in the document must be traceable to a specific file, line number, and code construct
- **No Existing File Modifications**: Do not modify any existing files in the source repository
- **No Additional Code**: Do not add any other code in the source repository besides the requested markdown document
- **Output Location**: Place the generated document in the `blitzy/documentation` directory in the destination repo

### 0.7.2 Investigation-Specific Conventions

- **Evidence Standard**: Each answer in the document must cite specific file paths and line numbers from the k6 repository
- **State Machine Accuracy**: The VU handle state machine transition table in `vu_handle.go:24-55` must be treated as the authoritative reference; any behavior not in the table is either a bug or an undocumented assumption
- **Test Validation**: Conclusions about race-freedom must be supported by successful `-race` test runs, not just static code analysis
- **Segment Math Proofs**: Claims about execution segment correctness must reference the `TestSumRandomSegmentSequenceMatchesNoSegment` randomized property test as evidence
- **Clean-Up Obligation**: Although the user mentioned "temporary investigation scripts are fine but clean up after," the SWE-AtlasQnA-Repo rule prohibits adding any code at all, so no temporary scripts should be created

## 0.8 References

### 0.8.1 Files and Folders Searched

The following files and folders were systematically retrieved and analyzed to derive the conclusions in this Agent Action Plan:

**Root-Level Exploration:**
- `/` (repository root) — Full folder listing and project summary
- `go.mod` — Go module definition, version constraints (Go 1.21, toolchain go1.21.13)

**Executor Subsystem (Deep Search, 3+ levels):**
- `lib/` — Folder listing and summary
- `lib/executor/` — Folder listing and summary (all 24 files enumerated)
- `lib/executor/ramping_vus.go` — Full file content (712 lines)
- `lib/executor/vu_handle.go` — Full file content (264 lines)
- `lib/executor/helpers.go` — Full file content (264 lines)
- `lib/executor/base_config.go` — Full file content (147 lines)
- `lib/executor/ramping_vus_test.go` — Lines 1-850 and 1112-1201 (key test cases)
- `lib/executor/vu_handle_test.go` — Full file content (415 lines)

**Execution State and Segment Math:**
- `lib/execution.go` — Full file content (549 lines)
- `lib/execution_segment.go` — Lines 1-100 and 570-842 (ScaleInt64, SegmentedIndex, ExecutionTuple)
- `lib/executors.go` — Lines 26-60 (ExecutionStep definition)
- `lib/helpers.go` — Full file content (95 lines)

**Scheduler and Abort:**
- `execution/` — Folder listing and summary
- `execution/scheduler.go` — Lines 1-100 and 300-591
- `execution/abort.go` — Full file content (85 lines)

**Signal Handling:**
- `cmd/run.go` — Lines 330-395 (signal trapping, Ctrl+C handling)

**Git History:**
- `git log --oneline -20` — Recent commit history
- `git log --all --oneline --grep="race"` — Race-condition-related commits in executor code
- `git branch --show-current` — Branch name: `k6_ddc3b0b1d23c`

**Test Executions:**
- `go test -v -race ./lib/executor/ -run TestRampingVUsConfigExecutionPlanExample` — PASS
- `go test -v -race ./lib/executor/ -run TestSumRandomSegmentSequenceMatchesNoSegment` — PASS (10/10 random seeds)
- `go test -v -race ./lib/executor/ -run TestVUHandle` — PASS (3 tests, including race detector)
- `go test -v -race ./lib/executor/ -run TestRampingVUsRampDownNoWobble` — PASS
- `go build ./...` — Successful build

### 0.8.2 Attachments and External Metadata

- **No Figma URLs** provided
- **No external attachments** provided
- **Environment**: Docker container `andrewparkscaleai/coding-agent:grafana__k6__ddc3b0b1d23c128e34e2792fc9075f9126e32375` from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_grafana_k6_1.0`
- **Source Branch**: `k6_ddc3b0b1d23c`
- **Output Document**: `blitzy/documentation/k6_ddc3b0b1d23c.md`

### 0.8.3 Key Historical Commits

The following commits from `git log --grep="race" -- lib/executor/` provide relevant historical context about prior race condition fixes in the executor subsystem:

| Commit | Message | Relevance |
|---|---|---|
| `e0eab8fb6` | "rewrite vuHandle as a state machine and fix some more races (#1506)" | The foundational commit that introduced the current state machine design in `vu_handle.go` |
| `93649df09` | "Fix race condition in ramping-vus tests" | Test-level race fix |
| `bdcdcb19d` | "Fix remaining gracefulSteps bug" | Fixed a bug in `runRemainingGracefulSteps` — directly related to user's question |
| `0486c11b8` | "merge steps in VLV executors for more stability (#1496)" | Step merging for execution stability |
| `b515245a0` | "Fix unlikely data race when calling BaseExecutor methods" | Prior race fix in the executor base |
| `5117c32b8` | "Fix for TestVUHandleSimple/start_before_gracefulStop_finishes" | VU handle timing fix |
| `60b2ffe9f` | "Change defer order in VLV Run" | Ordering fix in the ramping VUs Run method |

