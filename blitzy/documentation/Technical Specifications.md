# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new investigative analysis document** that comprehensively traces the concurrency model of k6's `ramping-vus` executor, answering specific technical questions about potential race conditions, VU lifecycle state machine interactions, execution segment scaling math, and graceful stop/ramp-down behavior.

- **Request Category:** Create new documentation
- **Documentation Type:** Technical investigation / architecture deep-dive (Q&A format)
- **Output File:** `blitzy/documentation/k6_ddc3b0b1d23c.md` (as per the implementation rule requiring `<source_branch_name>.md` in the `blitzy/documentation` directory)

The user has raised the following specific questions and observations, each requiring a code-grounded answer:

- **VU "stuck" state:** VUs appear to be neither fully active nor fully stopped during rapid stage transitions with a long `gracefulRampDown`. The user suspects a mismatch between the scheduled handler and the graceful handler's tracking of VU counts.
- **Ctrl+C / gracefulStop enforcement:** When the test is interrupted early, some VUs continue running longer than `gracefulStop` should permit.
- **Execution segment asymmetry:** When splitting execution across three instances using segments, one instance consistently shows more VUs than the others, and the sum exceeds the configured maximum.
- **Race condition hypothesis:** The user asks whether two handler goroutines (`maxAllowedVUsHandlerStrategy` and `scheduledVUsHandlerStrategy`) can race when simultaneously modifying VU state.
- **VU buffer leak hypothesis:** Whether the VU channel buffer in `ExecutionState.vus` can "leak" VUs that are never returned.

### 0.1.2 Special Instructions and Constraints

- **Implementation Rule (SWE-AtlasQnA-Repo):** The user's project rules mandate:
  - Create a new markdown document named `<source_branch_name>.md` (i.e., `k6_ddc3b0b1d23c.md`) that comprehensively answers the questions posed in the prompt.
  - Provide thinking and rationale behind the answers.
  - Base all answers on the code as the source of truth — do not make assumptions.
  - Do not modify any existing files in the source repository.
  - Place the generated document in the `blitzy/documentation` directory in the destination repo.
- **Temporary scripts:** The user explicitly permits temporary investigation scripts but requires cleanup afterward.
- **Style:** The document should walk through the code systematically, citing exact file paths and line numbers, and include diagrams where they clarify concurrency flows.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **answer the VU stuck-state question**, we will trace the `vuHandle` state machine (defined in `lib/executor/vu_handle.go`) and its transitions, correlating the `runLoopsIfPossible` loop with the `start()`, `gracefulStop()`, and `hardStop()` methods, documenting how rapid stage changes interact with the mutex-guarded state and the `canStartIter` channel.
- To **answer the Ctrl+C / gracefulStop question**, we will document how `getDurationContexts()` (in `lib/executor/helpers.go`) creates the `maxDurationCtx` and `regularDurationCtx`, how the `waiter()` function in `ramping_vus.go` responds to context cancellation, and how the abort mechanism in `execution/abort.go` propagates through the context tree.
- To **answer the execution segment asymmetry question**, we will document the `ExecutionSegment.Scale()` method and the `SegmentedIndex` iterator in `lib/execution_segment.go`, explaining the rounding behavior and the striping algorithm that can produce per-step differences between segments while maintaining global sum correctness.
- To **answer the race condition question**, we will document the dual-goroutine step-iteration model in `iterateSteps()` and `runRemainingGracefulSteps()`, the separation of concerns between the two handler closures, and the `vuHandle` mutex that serializes state transitions.
- To **answer the VU buffer leak question**, we will trace `GetPlannedVU()` and `ReturnVU()` in `lib/execution.go` and the `getVU`/`returnVU` closures in `runLoopsIfPossible()` to verify the symmetry of acquire/release calls.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following additional documentation elements are needed to fully answer the user's questions:

- A **state transition diagram** for `vuHandle` (the table in `vu_handle.go` lines 24-55 presented visually).
- A **sequence diagram** showing the interaction between `iterateSteps`, `runRemainingGracefulSteps`, and the two handler strategies during a rapid ramp-up/ramp-down.
- An **explanation of the execution plan** algorithm (`getRawExecutionSteps` vs. `reserveVUsForGracefulRampDowns` vs. `GetExecutionRequirements`), showing how the two step arrays (`rawSteps` and `gracefulSteps`) are constructed and consumed.
- A **walkthrough of the `roundUp` rounding** in `ExecutionSegment.Scale()` and the `ScaleInt64` method in `ExecutionSegmentSequenceWrapper` to explain why transient per-instance VU counts can appear asymmetric while the global sum remains correct.
- A **context hierarchy diagram** showing the relationship between `parentCtx`, `maxDurationCtx`, `regularDurationCtx`, and per-VU contexts to explain gracefulStop enforcement.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **minimal, design-proposal-oriented documentation structure** with no automated documentation generation pipeline. The project does not use any documentation site generator (no `mkdocs.yml`, `docusaurus.config.js`, or `sphinx/conf.py` detected).

- **Documentation root:** `docs/` — contains only a `design/` subfolder with three forward-looking design proposals (HTTP API, File API, Distributed Execution). None address the ramping executor's concurrency internals.
- **Root-level markdown files:** `README.md`, `CONTRIBUTING.md`, `Dependencies.md`, `SECURITY.md`, `SUPPORT.md`, `CODE_OF_CONDUCT.md`, `LICENSE.md` — standard project governance files. None document executor internals.
- **API documentation tools in use:** None detected. The codebase relies on Go doc comments embedded in source files.
- **Diagram tools detected:** None in the toolchain. Mermaid diagrams will be used in the new document as they render natively in GitHub markdown.
- **Relevant existing documentation:**
  - `docs/design/020-distributed-execution-and-test-suites.md` — discusses the `execution.Controller` abstraction and distributed execution coordination. Provides context for how execution segments were designed to enable future distributed execution.
  - Inline code comments in `lib/executor/ramping_vus.go` (lines 95-170) contain extensive ASCII-art diagrams explaining the execution step algorithm and segment scaling.
  - The state transition table in `lib/executor/vu_handle.go` (lines 24-55) is the primary reference for VU lifecycle behavior.

### 0.2.2 Repository Code Analysis for Documentation

The following search patterns were used to locate all code relevant to the user's questions:

- **Ramping VUs executor:** `lib/executor/ramping_vus.go` — contains `RampingVUsConfig`, `RampingVUs`, `rampingVUsRunState`, the raw/graceful step computation, and the `Run()` entry point.
- **VU handle state machine:** `lib/executor/vu_handle.go` — defines the `vuHandle` struct, its five states (`stopped`, `starting`, `running`, `toGracefulStop`, `toHardStop`), and the `runLoopsIfPossible()` event loop.
- **Execution state and VU buffer:** `lib/execution.go` — defines `ExecutionState` with the `vus` channel buffer, `GetPlannedVU()`, `ReturnVU()`, `ModCurrentlyActiveVUsCount()`, and all atomic counters.
- **Execution segment math:** `lib/execution_segment.go` — defines `ExecutionSegment`, `ExecutionSegmentSequence`, `ExecutionSegmentSequenceWrapper`, `ExecutionTuple`, `SegmentedIndex`, and the `Scale()` / `ScaleInt64()` / `GoTo()` methods.
- **Scheduler orchestration:** `execution/scheduler.go` — defines the `Scheduler` that creates `ExecutionState`, initializes VUs, and launches executors.
- **Abort/Cancel plumbing:** `execution/abort.go` — provides `NewTestRunContext()`, `AbortTestRun()`, and the `testAbortController` mutex-guarded first-reason pattern.
- **Duration context helpers:** `lib/executor/helpers.go` — `getDurationContexts()` creates the timeout contexts; `getIterationRunner()` handles per-iteration error handling and interrupted-iteration accounting.
- **Base config and gracefulStop:** `lib/executor/base_config.go` — defines `DefaultGracefulStopValue` (30 seconds) and the `GetGracefulStop()` accessor.

Key directories examined:
- `lib/executor/` — all executor implementations and tests
- `lib/` — core types, execution state, segment math
- `execution/` — scheduler, abort, controller interface

Related test files that inform the analysis:
- `lib/executor/ramping_vus_test.go` — tests for execution plan computation, graceful ramp-down, and integration runs
- `lib/executor/vu_handle_test.go` — race-condition stress tests (`TestVUHandleRace`, `TestVUHandleStartStopRace`) and state transition correctness tests (`TestVUHandleSimple`)
- `lib/execution_segment_test.go` — scaling consistency tests (`TestExecutionTupleScaleConsistency`, `TestExecutionSegmentScaleNoWobble`)

### 0.2.3 Web Search Research Conducted

No external web search is required for this task. The user's questions are entirely answerable from the codebase, which is the mandated source of truth per the implementation rules. All findings are derived from direct code inspection.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require documentation coverage in the new investigative document:

- **Module: `lib/executor/ramping_vus.go`**
  - Public APIs: `RampingVUsConfig`, `RampingVUs`, `Run()`, `Init()`, `getRawExecutionSteps()`, `reserveVUsForGracefulRampDowns()`, `GetExecutionRequirements()`
  - Internal types: `rampingVUsRunState`, `iterateSteps()`, `runRemainingGracefulSteps()`, `maxAllowedVUsHandlerStrategy()`, `scheduledVUsHandlerStrategy()`, `waiter()`
  - Current documentation: Inline comments present (extensive for step algorithms), but no external documentation exists
  - Documentation needed: Full concurrency walkthrough with diagrams; explanation of the dual-handler goroutine model

- **Module: `lib/executor/vu_handle.go`**
  - Public types: `vuHandle` (package-private but critical)
  - States: `stopped`, `starting`, `running`, `toGracefulStop`, `toHardStop`
  - Methods: `start()`, `gracefulStop()`, `hardStop()`, `runLoopsIfPossible()`, `changeState()`
  - Current documentation: State transition table in comments (lines 24-55); no external docs
  - Documentation needed: Visual state diagram; analysis of race windows; explanation of the `canStartIter` channel pattern

- **Module: `lib/execution.go`**
  - Public APIs: `ExecutionState`, `GetPlannedVU()`, `ReturnVU()`, `GetUnplannedVU()`, `ModCurrentlyActiveVUsCount()`, `GetCurrentlyActiveVUsCount()`
  - Current documentation: Extensive godoc comments with explicit warnings about synchronization
  - Documentation needed: VU buffer lifecycle analysis; explanation of why atomic counters are informational-only and not for synchronization

- **Module: `lib/execution_segment.go`**
  - Public APIs: `ExecutionSegment.Scale()`, `ExecutionSegmentSequenceWrapper.ScaleInt64()`, `SegmentedIndex.GoTo()`, `SegmentedIndex.Next()`, `SegmentedIndex.Prev()`
  - Current documentation: Extensive inline comments with examples
  - Documentation needed: Explanation of rounding behavior; why per-instance sums can transiently appear unequal at a single timestamp; proof of global sum correctness

- **Module: `lib/executor/helpers.go`**
  - Key functions: `getDurationContexts()`, `getIterationRunner()`, `trackProgress()`
  - Current documentation: Inline comments explaining the two-context model
  - Documentation needed: Context hierarchy diagram; explanation of gracefulStop enforcement via context deadlines

- **Module: `execution/abort.go`**
  - Key functions: `NewTestRunContext()`, `AbortTestRun()`, `testAbortController.abort()`
  - Current documentation: Godoc comments present
  - Documentation needed: Explanation of how Ctrl+C propagates through the context tree and interacts with executor-level contexts

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No external documentation** exists for any executor's concurrency internals — the only references are inline code comments and ASCII-art diagrams
- **No visual state diagrams** for the `vuHandle` state machine — only the textual transition table
- **No concurrency analysis document** tracing the interaction between the two handler goroutines and the VU handle state machine
- **No execution segment walkthrough** that explains the rounding behavior and striping algorithm in the context of VU count distribution
- **No context hierarchy documentation** showing how `parentCtx → maxDurationCtx → regularDurationCtx → per-VU ctx` chain operates during normal execution and interruption
- **No gracefulStop enforcement analysis** explaining the interplay between `gracefulStop`, `gracefulRampDown`, and context deadlines

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The new document `blitzy/documentation/k6_ddc3b0b1d23c.md` will follow a structured investigative format, answering each of the user's questions with code-grounded analysis:

```
blitzy/
└── documentation/
    └── k6_ddc3b0b1d23c.md
        ├── Introduction and Summary
        ├── Question 1: VU "Stuck" State Analysis
        │   ├── The vuHandle State Machine
        │   ├── State Transition Diagram
        │   ├── How Rapid Ramp-Up/Down Interacts with State
        │   └── Analysis of the Scheduled vs. Graceful Handler Mismatch
        ├── Question 2: Ctrl+C and gracefulStop Enforcement
        │   ├── Context Hierarchy
        │   ├── How the Abort Mechanism Propagates
        │   ├── Why VUs May Outlive gracefulStop
        │   └── The getDurationContexts() Two-Context Model
        ├── Question 3: Execution Segment Asymmetry
        │   ├── The Scaling Algorithm
        │   ├── The Striping Algorithm
        │   ├── Why Per-Instance VU Counts Differ at a Timestamp
        │   └── Proof of Global Sum Correctness
        ├── Question 4: Race Condition Between Handler Goroutines
        │   ├── The Dual-Handler Architecture
        │   ├── Thread Safety Analysis of vuHandle
        │   ├── The iterateSteps() Interleaving Logic
        │   └── Potential Race Windows
        ├── Question 5: VU Buffer Leak Analysis
        │   ├── VU Acquire/Release Symmetry
        │   ├── The getVU/returnVU Closure Pattern
        │   └── Edge Cases in VU Lifecycle
        └── Conclusion and Recommendations
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- "Extract the vuHandle state machine definition from `lib/executor/vu_handle.go` lines 14-55 and the method implementations at lines 115-264"
- "Extract the dual-handler strategy from `lib/executor/ramping_vus.go` lines 668-690 and the step iteration logic at lines 622-666"
- "Extract the execution segment scaling from `lib/execution_segment.go` lines 251-274 (Scale method) and 580-588 (ScaleInt64 method)"
- "Extract the context hierarchy from `lib/executor/helpers.go` lines 168-180 (getDurationContexts) and `execution/abort.go` lines 49-60 (NewTestRunContext)"
- "Extract the VU buffer management from `lib/execution.go` lines 471-488 (GetPlannedVU) and 544-549 (ReturnVU)"
- "Generate state transition diagrams from the transition table at `lib/executor/vu_handle.go` lines 24-55"
- "Derive concurrency analysis from test patterns in `lib/executor/vu_handle_test.go` (TestVUHandleRace, TestVUHandleStartStopRace)"
- "Derive scaling correctness from `lib/execution_segment_test.go` (TestExecutionTupleScaleConsistency, TestExecutionSegmentScaleNoWobble)"

**Documentation Standards:**

- Markdown formatting with proper headers (# ## ###)
- Mermaid diagram integration for state machines, sequence diagrams, and context hierarchies
- Code snippets with exact source citations: `Source: /path/to/file.go:LineNumber`
- Tables for parameter descriptions, state transitions, and segment scaling examples
- Every claim backed by a specific file path and line number range

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create:

- **State diagram** for `vuHandle` showing the five states and all transitions triggered by `start`, `loop`, `gracefulStop`, and `hardStop` inputs
- **Sequence diagram** showing the `Run()` → `runLoopsIfPossible()` → `iterateSteps()` → handler strategies → `runRemainingGracefulSteps()` interaction during a rapid ramp-up/ramp-down scenario
- **Flowchart** for the `reserveVUsForGracefulRampDowns()` algorithm showing how it traverses raw steps and decides whether to skip, delay, or break
- **Context hierarchy diagram** showing the nesting of contexts from the test run root to individual VU contexts
- **Timeline diagram** showing how `rawSteps` and `gracefulSteps` are interleaved by `iterateSteps()` and how the two handler goroutines process them

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/k6_ddc3b0b1d23c.md` | CREATE | `lib/executor/ramping_vus.go`, `lib/executor/vu_handle.go`, `lib/execution.go`, `lib/execution_segment.go`, `lib/executor/helpers.go`, `execution/abort.go`, `execution/scheduler.go`, `lib/executor/base_config.go` | Comprehensive investigative analysis document answering all five user questions about ramping executor concurrency, VU state machine behavior, execution segment scaling, gracefulStop enforcement, and VU buffer lifecycle |

No existing files will be modified, per the implementation rules.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/k6_ddc3b0b1d23c.md
Type: Technical Investigation / Architecture Deep-Dive
Source Code:
  - lib/executor/ramping_vus.go (primary — executor logic, step computation, handler strategies)
  - lib/executor/vu_handle.go (primary — VU state machine)
  - lib/execution.go (primary — VU buffer, execution state, atomic counters)
  - lib/execution_segment.go (primary — segment scaling, striping, SegmentedIndex)
  - lib/executor/helpers.go (supporting — getDurationContexts, getIterationRunner)
  - execution/abort.go (supporting — test abort propagation)
  - execution/scheduler.go (supporting — scheduler initialization, VU init)
  - lib/executor/base_config.go (supporting — DefaultGracefulStopValue)
  - lib/executor/ramping_vus_test.go (evidence — execution plan examples, graceful ramp-down tests)
  - lib/executor/vu_handle_test.go (evidence — race condition stress tests)
  - lib/execution_segment_test.go (evidence — scaling consistency proofs)
Sections:
  - Introduction and Summary of Findings
  - Question 1: VU "Stuck" State Analysis
      - vuHandle state machine with 5 states and 20 transitions
      - How rapid stage changes produce toGracefulStop ↔ running races
      - Why the scheduled handler and graceful handler see different VU counts (by design)
  - Question 2: Ctrl+C and gracefulStop Enforcement
      - Context hierarchy: testRunCtx → maxDurationCtx → regularDurationCtx → per-VU ctx
      - How AbortTestRun() cancels testRunCtx which cascades to all children
      - Why VUs may appear to outlive gracefulStop (graceful iteration completion)
  - Question 3: Execution Segment Asymmetry
      - The Scale() rounding algorithm and why round-up produces asymmetric per-instance counts
      - The ScaleInt64() striping algorithm and its cycle-based distribution
      - Proof that sum of all segments always equals the global value
  - Question 4: Race Condition Between Handler Goroutines
      - The two handlers operate on different index ranges (scheduled=rawSteps, graceful=gracefulSteps)
      - vuHandle.mutex serializes all state changes per-VU
      - iterateSteps() processes steps sequentially, never concurrently
  - Question 5: VU Buffer Leak Analysis
      - getVU/returnVU symmetry enforced by vuHandle lifecycle
      - Test evidence: TestVUHandleRace asserts getVUCount == returnVUCount
  - Conclusion and Recommendations
Diagrams:
  - Mermaid stateDiagram for vuHandle
  - Mermaid sequenceDiagram for Run() flow
  - Mermaid flowchart for context hierarchy
Key Citations:
  - lib/executor/ramping_vus.go:491-559 (Run method)
  - lib/executor/ramping_vus.go:622-666 (iterateSteps)
  - lib/executor/ramping_vus.go:654-666 (runRemainingGracefulSteps)
  - lib/executor/ramping_vus.go:668-690 (handler strategies)
  - lib/executor/vu_handle.go:14-55 (state transition table)
  - lib/executor/vu_handle.go:115-139 (start method)
  - lib/executor/vu_handle.go:147-163 (gracefulStop method)
  - lib/executor/vu_handle.go:165-181 (hardStop method)
  - lib/executor/vu_handle.go:185-264 (runLoopsIfPossible)
  - lib/execution.go:70-199 (ExecutionState definition)
  - lib/execution.go:471-488 (GetPlannedVU)
  - lib/execution.go:544-549 (ReturnVU)
  - lib/execution_segment.go:251-274 (Scale method)
  - lib/execution_segment.go:580-588 (ScaleInt64)
  - lib/execution_segment.go:768-842 (SegmentedIndex)
  - lib/executor/helpers.go:168-180 (getDurationContexts)
  - execution/abort.go:49-60 (NewTestRunContext)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration files need updating. The repository does not use a documentation site generator. The new file is a standalone markdown document placed in the `blitzy/documentation/` directory as specified by the implementation rules.

### 0.5.4 Cross-Documentation Dependencies

- The new document is self-contained and does not require updates to any existing documentation files
- No navigation links, table of contents, or index updates are needed
- No shared content or includes are involved
- The document references source code files by path and line number, but does not depend on any generated documentation artifacts

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No external documentation tools or packages are required for this task. The deliverable is a single Markdown file with embedded Mermaid diagrams, which render natively on GitHub without any build step.

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| N/A | Markdown (GitHub-flavored) | N/A | Document format — renders natively on GitHub |
| N/A | Mermaid (GitHub-native) | N/A | Diagram rendering — supported natively in GitHub markdown code blocks |

### 0.6.2 Project Dependencies Relevant to the Investigation

The following Go module dependencies are relevant to the code being documented, as extracted from `go.mod`:

| Registry | Package Name | Version | Relevance |
|----------|--------------|---------|-----------|
| Go modules | `go.k6.io/k6` | v0.55.0 | The k6 module itself (version from `lib/consts/consts.go`) |
| Go modules | Go toolchain | go1.21.13 | The Go version used to build k6 (from `go.mod` `toolchain` directive) |
| Go modules | `github.com/sirupsen/logrus` | (vendored) | Structured logging used throughout the executor package |
| Go modules | `gopkg.in/guregu/null.v3` | (vendored) | Nullable types used in `RampingVUsConfig` for `StartVUs`, stage `Target` |
| Go modules | `github.com/stretchr/testify` | (vendored) | Test assertions used in `ramping_vus_test.go` and `vu_handle_test.go` |

### 0.6.3 Documentation Reference Updates

No documentation link updates are required. The new document is a standalone creation that does not replace or modify any existing documentation.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

The new document must provide complete coverage of all five user questions:

| Question | Source Files | Coverage Target |
|----------|-------------|-----------------|
| Q1: VU "stuck" state | `vu_handle.go`, `ramping_vus.go` | 100% — all 5 states, all 20 transitions, all race windows documented |
| Q2: Ctrl+C / gracefulStop | `helpers.go`, `abort.go`, `ramping_vus.go` | 100% — full context hierarchy traced, enforcement mechanism explained |
| Q3: Execution segment asymmetry | `execution_segment.go` | 100% — `Scale()`, `ScaleInt64()`, `GoTo()` algorithms explained with examples |
| Q4: Handler goroutine race condition | `ramping_vus.go` lines 622-690 | 100% — `iterateSteps()`, both strategies, vuHandle mutex analyzed |
| Q5: VU buffer leak | `execution.go`, `vu_handle.go` | 100% — acquire/release symmetry proven with test evidence |

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**

- Every answer must cite specific source files and line numbers
- Every claim must be derivable from the codebase — no assumptions or speculation
- The state transition table must cover all 20 input×state combinations from `vu_handle.go`
- The execution segment explanation must include a concrete numerical example showing the rounding behavior
- The context hierarchy must trace from `NewTestRunContext()` down to per-VU contexts

**Accuracy validation:**

- Code snippets referenced in the document must match the actual source code
- State transitions described must match the `switch` statements in `start()`, `gracefulStop()`, `hardStop()`, and `runLoopsIfPossible()`
- Scaling arithmetic described must match the `Scale()` and `ScaleInt64()` implementations
- Test references must point to actual test functions that verify the described behaviors

**Clarity standards:**

- Each question should be answerable independently (readers can jump to the section they need)
- Mermaid diagrams should visually reinforce the textual explanations
- Technical depth should be sufficient for a Go developer familiar with goroutines and channels but not necessarily with k6 internals
- Thinking and rationale behind each answer must be explicit, per the implementation rules

### 0.7.3 Example and Diagram Requirements

| Diagram Type | Subject | Source |
|-------------|---------|--------|
| Mermaid `stateDiagram-v2` | vuHandle state machine | `vu_handle.go:14-55` |
| Mermaid `sequenceDiagram` | Run() → iterateSteps → handlers flow | `ramping_vus.go:491-559` |
| Mermaid `graph TD` | Context hierarchy | `helpers.go:168-180`, `abort.go:49-60` |
| Numerical table | Execution segment scaling example | `execution_segment.go:251-274` |
| Numerical table | Striped offset distribution example | `execution_segment.go:541-601` |

Minimum examples per question: at least one concrete worked example or code trace per question to demonstrate the behavior.

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**

- `blitzy/documentation/k6_ddc3b0b1d23c.md` — the sole deliverable; a comprehensive investigation document

**Source files to analyze and cite (read-only):**

- `lib/executor/ramping_vus.go` — primary subject: executor logic, step computation, handler strategies, waiter function
- `lib/executor/vu_handle.go` — primary subject: VU state machine, start/stop/run loop
- `lib/execution.go` — primary subject: VU buffer channel, planned/unplanned VU management, active VU counting
- `lib/execution_segment.go` — primary subject: segment scaling, striping, SegmentedIndex
- `lib/executor/helpers.go` — supporting: getDurationContexts, getIterationRunner, trackProgress
- `execution/abort.go` — supporting: test abort mechanism
- `execution/scheduler.go` — supporting: scheduler initialization, VU pre-init
- `lib/executor/base_config.go` — supporting: gracefulStop defaults
- `lib/executor/base_executor.go` — supporting: base executor scaffold
- `lib/helpers.go` — supporting: GetMaxPlannedVUs, GetEndOffset
- `lib/executors.go` — supporting: ExecutionStep definition, ExecutorConfig interface
- `lib/runner.go` — supporting: ActiveVU, InitializedVU, VUActivationParams interfaces
- `lib/executor/ramping_vus_test.go` — evidence: execution plan correctness tests
- `lib/executor/vu_handle_test.go` — evidence: race condition and state transition tests
- `lib/execution_segment_test.go` — evidence: scaling consistency and no-wobble proofs

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No existing files in the repository will be modified, per implementation rules
- **Other executors:** `constant_vus.go`, `constant_arrival_rate.go`, `ramping_arrival_rate.go`, `per_vu_iterations.go`, `shared_iterations.go`, `externally_controlled.go` — not directly relevant to the user's question about `ramping-vus`
- **Test file modifications:** No test files will be created or modified
- **Feature additions or code refactoring:** No code changes of any kind
- **Deployment or CI/CD configuration:** Not applicable
- **Documentation site generation or hosting:** No mkdocs, docusaurus, or sphinx configuration
- **Cloud execution internals:** The user's questions are about local execution behavior; cloud-specific paths (`cloudapi/`, `output/cloud/`) are out of scope
- **JavaScript runtime internals:** The user's questions are about the Go executor layer, not the Sobek JS engine (`js/` folder)
- **Metrics pipeline:** Output backends and metric sinks are not relevant to the concurrency questions
- **Browser module, WebSocket, gRPC:** Protocol-specific modules are not relevant
- **Unrelated documentation:** No modifications to `README.md`, `CONTRIBUTING.md`, or any other existing markdown files

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** N/A — the deliverable is a standalone Markdown file, no build step required
- **Documentation preview command:** Render `blitzy/documentation/k6_ddc3b0b1d23c.md` in any GitHub-flavored Markdown viewer; Mermaid diagrams render natively on GitHub
- **Diagram generation command:** N/A — Mermaid diagrams are embedded inline in the markdown and rendered by GitHub/GitLab/compatible viewers
- **Documentation deployment command:** N/A — file is committed to the repository directly
- **Default format:** GitHub-flavored Markdown with Mermaid diagram code blocks
- **Citation requirement:** Every section must reference source files with path and line numbers (e.g., `Source: lib/executor/vu_handle.go:115-139`)
- **Style guide:** Follow the existing documentation style observed in the repository: concise technical prose, inline code references with backticks, tables for structured data, and code blocks for source excerpts
- **Documentation validation:** Manual review — verify that all cited line numbers correspond to the correct code in the repository

## 0.10 Rules for Documentation

The following rules are derived from the user-specified implementation rules and the explicit instructions in the prompt:

- **Create, don't modify:** Create a new markdown document named `k6_ddc3b0b1d23c.md` — do not modify any existing files in the source repository
- **Code as truth:** Base all answers on the code as the source of truth — do not make assumptions or speculate about behavior not observable in the source
- **Provide rationale:** Every answer must include the thinking and rationale behind it, not just the conclusion
- **Placement:** Place the generated document in the `blitzy/documentation` directory
- **Source citations:** Every technical claim must cite the specific source file and line range that supports it
- **Temporary scripts permitted:** The user permits temporary investigation scripts for tracing behavior, but they must be cleaned up after use (not committed)
- **Comprehensive coverage:** All five user questions must be answered completely — no question may be deferred or left partially answered
- **Diagrams required:** Include Mermaid diagrams for the VU state machine, the concurrency flow between handler goroutines, and the context hierarchy
- **Numerical examples required:** Include concrete worked examples for execution segment scaling to demonstrate the rounding and striping behavior
- **Self-contained document:** The document must be readable independently without requiring the reader to open source files (though source citations should enable verification)

## 0.11 References

### 0.11.1 Source Files and Folders Searched

The following files and folders were systematically inspected to derive all conclusions in this Agent Action Plan:

**Primary source files (full content read):**

| File Path | Lines Read | Purpose |
|-----------|-----------|---------|
| `lib/executor/ramping_vus.go` | 1-713 | Ramping VUs executor: config, step computation, Run(), handler strategies, waiter |
| `lib/executor/vu_handle.go` | 1-265 | VU handle state machine: 5 states, start/stop/run loop |
| `lib/execution.go` | 1-550 | Execution state: VU buffer, planned/unplanned VU management, pause/resume |
| `lib/execution_segment.go` | 1-843 | Execution segments: Scale(), ScaleInt64(), SegmentedIndex, striping algorithm |
| `lib/executor/helpers.go` | 1-265 | getDurationContexts, getIterationRunner, trackProgress, getVUActivationParams |
| `lib/executor/base_config.go` | 1-148 | BaseConfig: GracefulStop default (30s), validation |
| `lib/helpers.go` | 1-95 | GetMaxPlannedVUs, GetMaxPossibleVUs, GetEndOffset |
| `lib/executors.go` | 1-100 | ExecutionStep struct, ExecutorConfig interface |
| `lib/runner.go` | 1-80 | ActiveVU, InitializedVU, VUActivationParams interfaces |
| `execution/abort.go` | 1-85 | testAbortController, NewTestRunContext, AbortTestRun |
| `execution/scheduler.go` | 1-300 | Scheduler: NewScheduler, initVUsAndExecutors, initVU, emitVUsAndVUsMax |

**Test files read for evidence:**

| File Path | Lines Read | Purpose |
|-----------|-----------|---------|
| `lib/executor/ramping_vus_test.go` | 1-700 | Execution plan examples, graceful ramp-down, segment scaling, VU ramp-down consistency |
| `lib/executor/vu_handle_test.go` | 1-416 | Race condition tests (TestVUHandleRace, TestVUHandleStartStopRace), state transition tests (TestVUHandleSimple), benchmarks |
| `lib/execution_segment_test.go` | 457-600 | Scaling consistency (TestExecutionTupleScaleConsistency), no-wobble proof (TestExecutionSegmentScaleNoWobble), striped offsets (TestGetStripedOffsets) |

**Folders explored:**

| Folder Path | Depth | Purpose |
|-------------|-------|---------|
| `` (root) | 0 | Repository structure overview |
| `lib/` | 1 | Core types and execution state |
| `lib/executor/` | 2 | Executor implementations and tests |
| `execution/` | 1 | Scheduler, abort, controller |
| `docs/` | 1 | Existing documentation assessment |
| `docs/design/` | 2 | Design proposals (020 relevant for distributed execution context) |

**Tech spec sections retrieved:**

| Section | Purpose |
|---------|---------|
| 1.1 Executive Summary | Project overview, version (v0.55.0), technology stack |
| 1.3 Scope | In-scope/out-of-scope boundaries, system capabilities |

### 0.11.2 Attachments

No attachments were provided by the user. The user's input was a text-based description of observed behaviors and questions.

### 0.11.3 External URLs

No Figma screens, external URLs, or design assets were provided or referenced.

