# Deep Code Investigation: Concurrency in k6's `ramping-vus` Executor

## Introduction and Scope

This document presents a deep code-level investigation into potential concurrency bugs in k6's `ramping-vus` executor. Every finding is grounded strictly in the source code — no assumptions, no speculation about "typical" Go concurrency behavior. Each claim traces to a specific file, function, and line range in the repository at the `k6_ddc3b0b1d23c` branch head (k6 v0.55.0, Go 1.21 toolchain per `go.mod` lines 1–5).

### Concerns Investigated

Six distinct anomalous behaviors have been reported:

1. **VU "stuck" state during rapid stage transitions** — VUs appear to enter an indeterminate state, neither fully active nor fully stopped.
2. **Handler VU count mismatch** — `scheduledVUsHandlerStrategy` and `maxAllowedVUsHandlerStrategy` report divergent VU counts at certain moments.
3. **VUs exceeding `gracefulStop` after Ctrl+C** — Some VUs continue executing beyond the `gracefulStop` timeout after interruption.
4. **Execution segment VU count imbalance** — When splitting load across three k6 instances, one consistently shows more VUs and the sum exceeds the configured maximum.
5. **Race condition between handler goroutines** — Suspected data race between `handleNewMaxAllowedVUs` and `handleNewScheduledVUs`.
6. **VU buffer leak** — Suspicion that VUs are not properly returned to the `ExecutionState.vus` channel after use.

### Primary Source Files Analyzed

| File Path | Role |
|-----------|------|
| `lib/executor/ramping_vus.go` (712 lines) | Primary executor: `Run()`, `iterateSteps()`, handler strategies, `rampingVUsRunState` |
| `lib/executor/vu_handle.go` (264 lines) | VU lifecycle state machine: 5-state FSM, `start()`, `gracefulStop()`, `hardStop()`, `runLoopsIfPossible()` |
| `lib/execution.go` (549 lines) | `ExecutionState`: VU buffer channel (`vus`), `GetPlannedVU()`, `ReturnVU()`, atomic counters |
| `lib/execution_segment.go` (842 lines) | `ExecutionSegment`, `ExecutionSegmentSequenceWrapper`, `ExecutionTuple`, `SegmentedIndex`, `ScaleInt64()`, striping algorithm |
| `lib/executor/helpers.go` (264 lines) | `getDurationContexts()`, `getIterationRunner()`, `getVUActivationParams()` |
| `execution/scheduler.go` (570+ lines) | `Scheduler.Run()`, `runExecutor()`, context propagation from `globalCtx` → `runCtx` → `executorsRunCtx` |
| `lib/executor/base_config.go` (147 lines) | `BaseConfig`, `DefaultGracefulStopValue` (30s), `GetGracefulStop()` |
| `lib/runner.go` (101 lines) | `ActiveVU`, `InitializedVU`, `VUActivationParams` interfaces |
| `lib/helpers.go` (94 lines) | `GetMaxPlannedVUs()`, `GetMaxPossibleVUs()`, `GetEndOffset()` |

### Test Files Analyzed for Behavioral Verification

| File Path | Key Tests |
|-----------|-----------|
| `lib/executor/vu_handle_test.go` | `TestVUHandleRace` (concurrent start/gracefulStop/hardStop, getVU==returnVU assertion at line 110), `TestVUHandleStartStopRace` |
| `lib/executor/ramping_vus_test.go` | `TestRampingVUsRun`, `TestRampingVUsGracefulStopWaits`, `TestRampingVUsGracefulRampDown`, segment sum verification tests |

---

## Q1: VU "Stuck" State During Rapid Stage Transitions

### The 5-State FSM

The `vuHandle` type in `lib/executor/vu_handle.go` (lines 70–88) implements a finite state machine with five states, defined at lines 16–22:

```go
const (
    stopped        stateType = iota // 0
    starting                        // 1
    running                         // 2
    toGracefulStop                  // 3
    toHardStop                      // 4
)
```

### State Transition Table

The complete state transition table is documented in `vu_handle.go` lines 24–54. Here is the full table from the source comments (corrected from source typo `toHardSTop` at line 48 to `toHardStop`):

```
+-------+-----------------+-------------------+---------------------------------------------------+
| input | current         | next state        | notes                                             |
+-------+-----------------+-------------------+---------------------------------------------------+
| start | stopped         | starting          | normal                                            |
| start | starting        | starting          | nothing                                           |
| start | running         | running           | nothing                                           |
| start | toGracefulStop  | running           | we raced with the loop stopping, just continue    |
| start | toHardStop      | starting          | same as stopped really                            |
| loop  | stopped         | stopped           | we actually are blocked on canStartIter           |
| loop  | starting        | running           | get new VU and context                            |
| loop  | running         | running           | usually fast path                                 |
| loop  | toGracefulStop  | stopped           | cancel the context and make new one               |
| loop  | toHardStop      | stopped           | cancel the context and make new one               |
| grace | stopped         | stopped           | nothing                                           |
| grace | starting        | stopped           | cancel the context to return the VU               |
| grace | running         | toGracefulStop    | normal one, the actual work is in the loop        |
| grace | toGracefulStop  | toGracefulStop    | nothing                                           |
| grace | toHardStop      | toHardStop        | nothing                                           |
| hard  | stopped         | stopped           | nothing                                           |
| hard  | starting        | stopped           | short circuit as in the grace case, not necessary |
| hard  | running         | toHardStop        | normal, cancel context and reinitialize it        |
| hard  | toGracefulStop  | toHardStop        | normal, cancel context and reinitialize it        |
| hard  | toHardStop      | toHardStop        | nothing                                           |
+-------+-----------------+-------------------+---------------------------------------------------+
```

### Analysis of the "Stuck" State

The VU is **not stuck** — it is in the `toGracefulStop` state (value 3), which is a legitimate transitional state. Here is the detailed trace-through:

**Step 1: `gracefulStop()` transitions `running` → `toGracefulStop`**

When the scheduled handler decides to ramp down a VU, it calls `vuHandle.gracefulStop()` at `vu_handle.go` lines 147–163:

```go
func (vh *vuHandle) gracefulStop() {
    vh.mutex.Lock()
    defer vh.mutex.Unlock()
    switch vh.state {
    case toGracefulStop, toHardStop, stopped:
        return // nothing to do
    case starting:
        vh.cancel()
        vh.ctx, vh.cancel = context.WithCancel(vh.parentCtx)
        vh.changeState(stopped)
    case running:
        vh.changeState(toGracefulStop)  // line 158
    }

    vh.logger.Debug("Graceful stop")
    vh.canStartIter = make(chan struct{})  // line 162: blocks new iterations
}
```

At line 158, the state transitions to `toGracefulStop`. At line 162, a new `canStartIter` channel is created — this replaces the previously-closed channel, effectively blocking the VU from starting new iterations. Critically, **the VU's context is NOT cancelled here** — the current iteration is allowed to finish.

**Step 2: The VU goroutine processes the state change in `runLoopsIfPossible()`**

The VU goroutine runs in `runLoopsIfPossible()` (`vu_handle.go` lines 185–264). Its main loop has a fast path and a slow path:

- **Fast path** (lines 204–207): `atomic.LoadInt32(&vh.state)` is checked. If state is `running`, the iteration proceeds without acquiring the mutex. This is a lock-free hot path for normal execution.

- **Slow path** (lines 210 onward): When the fast path fails (state is NOT `running`), the mutex is acquired. At lines 223–230, when state is `toGracefulStop`:

```go
case toGracefulStop:
    if cancel != nil {
        cancel()  // cancel the VU's context to return the VU
        vh.ctx, vh.cancel = context.WithCancel(vh.parentCtx)
    }
    fallthrough // to set the state
case toHardStop:
    vh.changeState(stopped)  // line 233
```

The context is cancelled (line 227), a new context is created (line 228), and the state falls through to `stopped` (line 233). Then at lines 238–240, the goroutine captures `canStartIter` and the new `ctx`, releases the mutex, and enters a blocking `select` at lines 244–262:

```go
select {
case <-canStartIter:   // line 245: signal to start again
    vh.mutex.Lock()
    select {
    case <-vh.canStartIter:
        vu, ctx, cancel = vh.activeVU, vh.ctx, vh.cancel
        vh.changeState(running)   // line 251
    default:
        // raced, loop again
    }
    vh.mutex.Unlock()
case <-ctx.Done():      // line 256: hardStop was called
case <-executorDone:    // line 259: executor is completely done
    return
}
```

**Step 3: The `start()` → `toGracefulStop` race optimization**

If `start()` is called while the VU is still in `toGracefulStop` (i.e., before the VU goroutine has reached the slow path), the transition at `vu_handle.go` lines 122–125 fires:

```go
case toGracefulStop:
    vh.logger.Debug("Start")
    close(vh.canStartIter)      // line 124: unblocks the VU goroutine
    vh.changeState(running)     // line 125: back to running
```

This closes the `canStartIter` channel, which unblocks the VU goroutine if it's waiting on `<-canStartIter` at line 245. The state transitions directly to `running` **without returning the VU to the buffer and re-acquiring it**. This is an intentional optimization, noted in the source comment at `vu_handle.go` lines 68–69:

> *"it's not required but preferable, if where possible to not reactivate VUs and to reuse context as this speed ups the execution"*

**Step 4: The window where the VU appears "stuck"**

The "stuck" window exists between:
1. `gracefulStop()` setting state to `toGracefulStop` (line 158), and
2. The VU goroutine reaching the slow path in `runLoopsIfPossible()` (line 210+).

During this window, the VU is **finishing its current iteration**. It is not idle — it is completing work. The state reads as `toGracefulStop` (value 3), which could appear as "neither running nor stopped" to external observers.

### Verdict: BY DESIGN

The `toGracefulStop` state is intentionally designed to let VUs finish their current iteration. The "graceful" in `gracefulStop` means allowing the in-progress iteration to complete naturally. The VU is not stuck; it is in a well-defined transitional state. The state machine explicitly handles the race between `start()` and `toGracefulStop` by transitioning directly back to `running` (lines 122–125) without VU reacquisition overhead.

---

## Q2: Handler VU Count Mismatch (scheduledVUsHandlerStrategy vs maxAllowedVUsHandlerStrategy)

### The Two Handler Strategies

The `ramping-vus` executor uses two independent handler strategies, both defined in `lib/executor/ramping_vus.go`:

**`scheduledVUsHandlerStrategy()`** — lines 679–690:

```go
func (rs *rampingVUsRunState) scheduledVUsHandlerStrategy() func(lib.ExecutionStep) {
    var cur uint64 // closure-local counter (line 680)
    return func(raw lib.ExecutionStep) {
        pv := raw.PlannedVUs
        for ; cur < pv; cur++ {
            _ = rs.vuHandles[cur].start()   // ramp UP: start VUs (line 684)
        }
        for ; pv < cur; cur-- {
            rs.vuHandles[cur-1].gracefulStop()  // ramp DOWN: graceful stop (line 687)
        }
    }
}
```

This handler tracks the **target number of actively-running VUs** — the "raw" execution plan. It calls `start()` to ramp up and `gracefulStop()` to ramp down.

**`maxAllowedVUsHandlerStrategy()`** — lines 668–677:

```go
func (rs *rampingVUsRunState) maxAllowedVUsHandlerStrategy() func(lib.ExecutionStep) {
    var cur uint64 // closure-local counter (line 669)
    return func(graceful lib.ExecutionStep) {
        pv := graceful.PlannedVUs
        for ; pv < cur; cur-- {
            rs.vuHandles[cur-1].hardStop()  // only shrinks: hard stop (line 673)
        }
        cur = pv
    }
}
```

This handler tracks the **maximum allowed VUs including those in graceful ramp-down** — the "graceful" execution plan. It only calls `hardStop()` to shrink the ceiling. It **never** calls `start()`.

### How Both Handlers Are Invoked

In `Run()` at lines 545–558, both handlers are created and then passed to `iterateSteps()`:

```go
var (
    handleNewMaxAllowedVUs = runState.maxAllowedVUsHandlerStrategy()
    handleNewScheduledVUs  = runState.scheduledVUsHandlerStrategy()
)
handledGracefulSteps := runState.iterateSteps(
    ctx,
    handleNewMaxAllowedVUs,
    handleNewScheduledVUs,
)
```

The `iterateSteps()` method (lines 622–645) processes both step arrays (`rawSteps` and `gracefulSteps`) in time-offset order, calling the handlers **sequentially in a single goroutine**:

```go
for i != len(rs.executor.rawSteps) {
    r, g := rs.executor.rawSteps[i], rs.executor.gracefulSteps[j]
    if g.TimeOffset < r.TimeOffset {
        if wait(g.TimeOffset) { break }
        handleNewMaxAllowedVUs(g)   // line 634
        j++
    } else {
        if wait(r.TimeOffset) { break }
        handleNewScheduledVUs(r)    // line 640
        i++
    }
}
```

### Why the Counters Differ — By Design

The two step arrays represent conceptually different things:

- **`rawSteps`** (from `getRawExecutionSteps()`, lines 171–234): The ideal number of actively-running VUs at each moment. These are the "active target" values.
- **`gracefulSteps`** (from `GetExecutionRequirements()`, lines 434–449): The maximum allowed VUs **including reservations** for VUs that are in their graceful ramp-down period. These are computed by `reserveVUsForGracefulRampDowns()` (lines 307–414).

**Example**: If we ramp from 10 to 5 VUs with a 30s `gracefulRampDown`:
- The raw step says PlannedVUs = 5 (the active target).
- The graceful step may say PlannedVUs = 10 (keeping the old 5 VUs reserved for their graceful ramp-down period).

The "mismatch" is the graceful ramp-down reservation system working exactly as designed.

### How the Steps Are Precalculated

In `Init()` at lines 479–487:

```go
func (vlv *RampingVUs) Init(_ context.Context) error {
    vlv.rawSteps = vlv.config.getRawExecutionSteps(
        vlv.executionState.ExecutionTuple, true,
    )
    vlv.gracefulSteps = vlv.config.GetExecutionRequirements(
        vlv.executionState.ExecutionTuple,
    )
    return nil
}
```

Both step arrays are pre-calculated during initialization, before execution begins.

### The Invariant

At any moment during `iterateSteps()`, the following invariant holds:

```
scheduledVUsHandlerStrategy.cur ≤ maxAllowedVUsHandlerStrategy.cur
```

Because the graceful plan always includes the active target **plus** any additional VU reservations for graceful ramp-down periods.

### Verdict: BY DESIGN

The two counters represent intentionally different concepts. `scheduledVUsHandlerStrategy.cur` is the active VU target. `maxAllowedVUsHandlerStrategy.cur` is the graceful reservation ceiling. They are deliberately independent, with different closure-local `cur` counters, because they serve different purposes in the executor's lifecycle management.

---

## Q3: VUs Exceeding `gracefulStop` After Ctrl+C

### Context Propagation Chain

The full context cancellation chain from Ctrl+C to individual VU handles:

**Layer 1: Scheduler** — `execution/scheduler.go`

At line 419, `Scheduler.Run()` receives both `globalCtx` and `runCtx`:

```go
func (e *Scheduler) Run(globalCtx, runCtx context.Context, samplesOut chan<- metrics.SampleContainer) (runErr error) {
```

Ctrl+C cancels `runCtx`. At line 497, the executor-specific context is created:

```go
executorsRunCtx, executorsRunCancel := context.WithCancel(withExecStateCtx)
```

where `withExecStateCtx` is derived from `runCtx` at line 464. Each executor is launched at line 500:

```go
go e.runExecutor(executorsRunCtx, runResults, samplesOut, exec)
```

And `runExecutor()` at line 369 passes `executorsRunCtx` directly to the executor:

```go
err := executor.Run(runCtx, engineOut)
```

(Note: the parameter name `runCtx` at line 332 is the argument `executorsRunCtx` passed at line 500.)

**Layer 2: Executor** — `lib/executor/ramping_vus.go`

At line 491, `RampingVUs.Run()` receives this context as `ctx`:

```go
func (vlv *RampingVUs) Run(ctx context.Context, _ chan<- metrics.SampleContainer) error {
```

At lines 501–502, `getDurationContexts()` creates sub-contexts:

```go
startTime, maxDurationCtx, regularDurationCtx, cancel := getDurationContexts(
    ctx, regularDuration, maxDuration-regularDuration,
)
```

**Layer 3: Duration Contexts** — `lib/executor/helpers.go`

At lines 168–180, `getDurationContexts()` creates the context hierarchy:

```go
func getDurationContexts(parentCtx context.Context, regularDuration, gracefulStop time.Duration) (
    startTime time.Time, maxDurationCtx, regDurationCtx context.Context, maxDurationCancel func(),
) {
    startTime = time.Now()
    maxEndTime := startTime.Add(regularDuration + gracefulStop)
    maxDurationCtx, maxDurationCancel = context.WithDeadline(parentCtx, maxEndTime)  // line 174
    if gracefulStop == 0 {
        return startTime, maxDurationCtx, maxDurationCtx, maxDurationCancel
    }
    regDurationCtx, _ = context.WithDeadline(maxDurationCtx, startTime.Add(regularDuration))  // line 178
    return startTime, maxDurationCtx, regDurationCtx, maxDurationCancel
}
```

At line 174: `maxDurationCtx` is a **child of `ctx`** (which is `executorsRunCtx`).

**Layer 4: VU Handles** — `lib/executor/vu_handle.go`

At line 543 of `ramping_vus.go`, `maxDurationCtx` is passed as the parent for all VU handles:

```go
runState.runLoopsIfPossible(maxDurationCtx, cancel)
```

And in `runLoopsIfPossible()` at lines 611–616, each VU handle is created with `ctx` = `maxDurationCtx`:

```go
rs.vuHandles[i] = newStoppedVUHandle(
    ctx, getVU, returnVU, rs.executor.nextIterationCounters,
    &rs.executor.config.BaseConfig, rs.executor.logger.WithField("vuNum", i))
```

In `newStoppedVUHandle()` at line 96:

```go
ctx, cancel := context.WithCancel(parentCtx)
```

The VU's context is a **child of `maxDurationCtx`**.

**Complete chain:**

```
Ctrl+C → runCtx cancelled
  → executorsRunCtx cancelled (child of runCtx, via line 497)
    → maxDurationCtx cancelled (child of executorsRunCtx, via line 174)
      → vuHandle.ctx cancelled (child of maxDurationCtx, via line 96)
```

### Critical Defer Ordering in `Run()`

The defer stack in `Run()` is set up as follows:

```go
// Line 504-507 (pushed FIRST onto defer stack):
defer func() {
    cancel()              // cancels maxDurationCtx
    <-waitOnProgressChannel
}()

// ... setup code ...

// Line 540 (pushed LAST onto defer stack):
defer runState.wg.Wait()
```

Go executes defers in LIFO (last-in, first-out) order, so:

1. **FIRST**: `runState.wg.Wait()` — waits for ALL VU goroutines to finish.
2. **THEN**: `cancel()` — cancels `maxDurationCtx` (redundant if parent was already cancelled).

### The Critical Insight

When Ctrl+C is pressed:
- `ctx` (the parent passed to `Run()`) is cancelled **immediately**.
- `maxDurationCtx` is a **child of `ctx`**, so it is **also cancelled immediately** via Go's context tree propagation.
- The `cancel()` in the deferred function (line 505) is now redundant but harmless.
- `runState.wg.Wait()` blocks until VU goroutines complete.
- VU goroutines see their context cancelled and should stop.

### Why VUs May Appear to Exceed `gracefulStop`

The issue lies in how `getIterationRunner()` checks context cancellation. In `lib/executor/helpers.go` lines 107–141:

```go
return func(ctx context.Context, vu lib.ActiveVU) bool {
    err := vu.RunOnce()          // line 108: blocks until iteration finishes

    select {
    case <-ctx.Done():           // line 114: checked AFTER RunOnce() returns
        executionState.AddInterruptedIterations(1)
        return false
    default:
        // ... handle errors, count iteration ...
    }
}
```

The context check at line 114 happens **after** `RunOnce()` returns, not during it. This means:

1. **If `RunOnce()` blocks** (on HTTP I/O, `sleep()`, or other blocking operations) **and does NOT internally check its context**, the VU will appear to keep running even after Ctrl+C.
2. The VU's iteration must **voluntarily honor its context** for cancellation to be timely.
3. The `hardStop()` mechanism (line 165–181 of `vu_handle.go`) is the mitigation — it directly cancels the VU's individual context via `vh.cancel()` at line 178.

### Verdict: MOSTLY BY DESIGN, With a Script-Level Caveat

Context cancellation from Ctrl+C propagates correctly through the entire chain: `runCtx` → `executorsRunCtx` → `maxDurationCtx` → `vuHandle.ctx`. The k6 executor framework is correct. However, if the user's script iteration (`RunOnce()`) blocks on I/O or sleep without checking `ctx.Done()`, the VU will appear to run beyond `gracefulStop`. This is a script authoring responsibility, not a k6 framework bug. The `RunOnce()` documentation in `lib/runner.go` line 16 states:

> *"The only way to interrupt the execution is to cancel the context given to InitializedVU.Activate()"*

---

## Q4: Execution Segment VU Count Imbalance Across Instances

### Two Different Scaling Mechanisms

k6 has two fundamentally different approaches to scaling VU counts across execution segments, and the choice between them depends on whether a sequence is provided.

**Mechanism 1: `ExecutionSegment.Scale()` — WITHOUT a sequence**

Located in `lib/execution_segment.go` lines 253–274:

```go
func (es *ExecutionSegment) Scale(value int64) int64 {
    if es == nil {
        return value
    }
    // round(value * to - round(value * from))
    toValue := big.NewRat(value, 1)
    toValue.Mul(toValue, es.to)

    fromValue := big.NewRat(value, 1)
    fromValue.Mul(fromValue, es.from)

    toValue.Sub(toValue, new(big.Rat).SetFrac(roundUp(fromValue), oneBigInt))
    return roundUp(toValue).Int64()
}
```

This uses rounding-based arithmetic: `round(value × to − round(value × from))`. The source comment at lines 257–263 explains the formula. This is **NOT sum-preserving** — each instance scales independently. With 3 instances and 10 VUs, the individual results can sum to more than 10 (e.g., 4 + 3 + 4 = 11).

**Mechanism 2: `ExecutionSegmentSequenceWrapper.ScaleInt64()` — WITH a sequence**

Located in `lib/execution_segment.go` lines 580–588:

```go
func (essw *ExecutionSegmentSequenceWrapper) ScaleInt64(segmentIndex int, value int64) int64 {
    start := essw.offsets[segmentIndex][0]
    offsets := essw.offsets[segmentIndex][1:]
    result := (value / essw.lcd) * int64(len(offsets))
    for gi, i := 0, start; i < value%essw.lcd; gi, i = gi+1, i+offsets[gi] {
        result++
    }
    return result
}
```

This uses the **striping algorithm** from `NewExecutionSegmentSequenceWrapper()` (lines 490–571). It IS sum-preserving and guaranteed non-overlapping.

### The Striping Algorithm

`NewExecutionSegmentSequenceWrapper()` (lines 490–571) pre-computes the striped offsets:

1. **Compute LCD** of all segment denominators (line 497): `lcd := ess.LCD()`
2. **Normalize numerators** to LCD base (lines 508–513): Each segment's length numerator is scaled to the common denominator.
3. **Sort segments** from biggest to smallest (lines 515–517): Biggest segments take elements first to maximize spacing.
4. **Stripe elements** (lines 558–568): For each element index 0..LCD-1, assign it to the first eligible segment:

```go
for i := int64(0); i < lcd; i++ {
    for sortedIndex, chosenCount := range chosenCounts {
        num := chosenCount * lcd
        denom := sortedNormalizedIndexes[sortedIndex].normNumerator
        if i > num/denom || (i == num/denom && num%denom == 0) {
            chosenCounts[sortedIndex]++
            saveIndex(i, sortedNormalizedIndexes[sortedIndex].originalIndex, denom)
            break
        }
    }
}
```

The algorithm is explained in the lengthy comment at lines 519–548: it distributes elements proportionally using the LCD as the cycle length, giving each segment its fair share while maximizing the distance between consecutive elements assigned to the same segment.

### Why Imbalance Occurs — Segment Without Sequence

When the user runs three k6 instances:
- Instance 1: `k6 --execution-segment=0:1/3`
- Instance 2: `k6 --execution-segment=1/3:2/3`
- Instance 3: `k6 --execution-segment=2/3:1`

**Without** `--execution-segment-sequence=0,1/3,2/3,1`:

Each instance creates its `ExecutionTuple` via `NewExecutionTuple()` at lines 723–731:

```go
func NewExecutionTuple(segment *ExecutionSegment, sequence *ExecutionSegmentSequence) (*ExecutionTuple, error) {
    filledSeq := GetFilledExecutionSegmentSequence(sequence, segment)
    wrapper := NewExecutionSegmentSequenceWrapper(filledSeq)
    index, err := wrapper.FindSegmentPosition(segment)
    // ...
}
```

When `sequence` is `nil`, `GetFilledExecutionSegmentSequence(nil, segment)` at lines 445–472 creates a **minimal 2-element sequence** by filling around the segment with placeholder segments:

```go
if sequence == nil || len(*sequence) == 0 {
    if fallback == nil || fallback.length.Cmp(oneRat) == 0 {
        return ExecutionSegmentSequence{newExecutionSegment(zeroRat, oneRat)}
    }
    result = ExecutionSegmentSequence{fallback}
}
// Fill gaps at beginning and end
if result[0].from.Cmp(zeroRat) != 0 {
    es := newExecutionSegment(zeroRat, result[0].from)
    result = append(ExecutionSegmentSequence{es}, result...)
}
if result[len(result)-1].to.Cmp(oneRat) != 0 {
    es := newExecutionSegment(result[len(result)-1].to, oneRat)
    result = append(result, es)
}
```

Each instance gets its own independent 2-element sequence (e.g., `[0:1/3, 1/3:1]` for instance 1). Since the three instances don't share the same sequence, their striping algorithms operate independently, and the sums can exceed the total.

**With** `--execution-segment-sequence=0,1/3,2/3,1`:

All instances share the same 3-element sequence. The striping algorithm in `NewExecutionSegmentSequenceWrapper()` computes non-overlapping offsets, and `ScaleInt64()` returns values that sum exactly to the unscaled total.

### Test Evidence

The `TestSumRandomSegmentSequenceMatchesNoSegment` test in `lib/executor/ramping_vus_test.go` verifies that with a proper sequence, the sum of all segments' scaled VU counts equals the unsegmented VU count at every execution step. This is the correctness guarantee of the striping algorithm.

### Verdict: USER MISCONFIGURATION (Likely)

If the user provides `--execution-segment` without `--execution-segment-sequence`, each instance scales independently using `GetFilledExecutionSegmentSequence(nil, segment)`, which creates per-instance independent sequences. The rounding in the striping of those independent sequences is **not sum-preserving** across instances. The fix is to always provide **both** `--execution-segment` AND `--execution-segment-sequence` for distributed execution. With the proper shared sequence, the striping algorithm guarantees exact, non-overlapping, sum-preserving distribution.

---

## Q5: Race Condition Between Handler Goroutines

### Sequential Execution Proof

The `iterateSteps()` method in `lib/executor/ramping_vus.go` lines 622–645 provides the definitive answer:

```go
func (rs *rampingVUsRunState) iterateSteps(
    ctx context.Context,
    handleNewMaxAllowedVUs, handleNewScheduledVUs func(lib.ExecutionStep),
) (handledGracefulSteps int) {
    wait := waiter(ctx, rs.started)
    i, j := 0, 0
    for i != len(rs.executor.rawSteps) {
        r, g := rs.executor.rawSteps[i], rs.executor.gracefulSteps[j]
        if g.TimeOffset < r.TimeOffset {
            if wait(g.TimeOffset) {
                break
            }
            handleNewMaxAllowedVUs(g)   // line 634: called directly
            j++
        } else {
            if wait(r.TimeOffset) {
                break
            }
            handleNewScheduledVUs(r)    // line 640: called directly
            i++
        }
    }
    return j
}
```

Both handlers are called from the **same goroutine, sequentially**. There is no `go` keyword — no concurrent invocation during the main stage processing loop. The for loop at line 628 processes steps one at a time, calling either `handleNewMaxAllowedVUs` (line 634) or `handleNewScheduledVUs` (line 640), never both simultaneously.

### The Only True Concurrency Point

After `iterateSteps()` returns, `runRemainingGracefulSteps()` IS launched as a **separate goroutine** at lines 554–558 of `ramping_vus.go`:

```go
go runState.runRemainingGracefulSteps(
    ctx,
    handleNewMaxAllowedVUs,
    handledGracefulSteps,
)
```

This goroutine processes the remaining graceful steps (lines 654–666) and calls `handleNewMaxAllowedVUs()`, which calls `vuHandle.hardStop()`. This runs **concurrently with VU goroutines** that are finishing their iterations.

### Why This Is Safe

The concurrency between `runRemainingGracefulSteps()` and VU goroutines is properly synchronized via the `vuHandle.mutex`:

1. **`hardStop()`** acquires `vh.mutex` at line 166:
```go
func (vh *vuHandle) hardStop() {
    vh.mutex.Lock()
    defer vh.mutex.Unlock()
    // ...
}
```

2. **`runLoopsIfPossible()`** acquires `vh.mutex` in the slow path at line 210:
```go
vh.mutex.Lock()
```

3. **The fast path** at line 204 uses `atomic.LoadInt32((*int32)(&vh.state))` — a lock-free atomic read that is safe for concurrent access. State transitions use `atomic.StoreInt32` via `changeState()` at line 144.

4. **Every state-modifying method** (`start()` at line 116, `gracefulStop()` at line 148, `hardStop()` at line 165) acquires `vh.mutex` before modifying state.

### Test Evidence

`TestVUHandleRace` in `vu_handle_test.go` (lines 25–111) is specifically designed to test this:

- Lines 72–94: Three concurrent goroutines are launched:
  - One calling `start()` 10,000 times (line 74)
  - One calling `gracefulStop()` 1,000 times (line 83)
  - One calling `hardStop()` 100 times (line 91)
- Line 24 comment: *"this test is mostly interesting when -race is enabled"*
- The test passes with `-race` flag, confirming no data races exist.

### Verdict: BY DESIGN (NOT a Race Condition)

During the main stage processing (`iterateSteps()`), the two handlers execute **sequentially in a single goroutine**. The only concurrent access point is between `runRemainingGracefulSteps()` (a separate goroutine) and VU goroutines, which is properly synchronized via `vuHandle.mutex` and `sync/atomic` operations. The `TestVUHandleRace` test confirms correctness under heavy concurrent access.

---

## Q6: VU Buffer Leak Investigation

### The VU Buffer Channel

In `lib/execution.go`, the VU buffer is defined at line 106:

```go
vus chan InitializedVU
```

It is created at line 217 with capacity equal to `maxPossibleVUs`:

```go
vus: make(chan InitializedVU, maxPossibleVUs),
```

### `GetPlannedVU()` — Reading from the Buffer

At `lib/execution.go` lines 471–488:

```go
func (es *ExecutionState) GetPlannedVU(logger *logrus.Entry, modifyActiveVUCount bool) (InitializedVU, error) {
    for i := 1; i <= MaxRetriesGetPlannedVU; i++ {
        select {
        case vu := <-es.vus:                  // line 474: read from channel
            if modifyActiveVUCount {
                es.ModCurrentlyActiveVUsCount(+1)
            }
            return vu, nil
        case <-time.After(MaxTimeToWaitForPlannedVU):
            logger.Warnf("Could not get a VU from the buffer for %s", ...)
        }
    }
    return nil, fmt.Errorf("could not get a VU from the buffer in %s", ...)
}
```

It retries up to `MaxRetriesGetPlannedVU` = 5 times (line 29), with each attempt waiting `MaxTimeToWaitForPlannedVU` = 400ms (line 25). Total timeout: 2 seconds.

### `ReturnVU()` — Writing Back to the Buffer

At `lib/execution.go` lines 544–549:

```go
func (es *ExecutionState) ReturnVU(vu InitializedVU, wasActive bool) {
    es.vus <- vu                    // line 545: write back to channel
    if wasActive {
        es.ModCurrentlyActiveVUsCount(-1)
    }
}
```

### The `getVU`/`returnVU` Closures in the Executor

In `rampingVUsRunState.runLoopsIfPossible()` at `ramping_vus.go` lines 592–616:

**`getVU` closure** (lines 593–604):

```go
getVU := func() (lib.InitializedVU, error) {
    pvu, err := rs.executor.executionState.GetPlannedVU(rs.executor.logger, false)  // line 594
    if err != nil {
        rs.executor.logger.WithError(err).Error("Cannot get a VU from the buffer")
        cancel()
        return pvu, err
    }
    rs.wg.Add(1)                                                   // line 600: increment WaitGroup
    atomic.AddInt64(rs.activeVUsCount, 1)                          // line 601: increment display counter
    rs.executor.executionState.ModCurrentlyActiveVUsCount(+1)      // line 602: increment global counter
    return pvu, err
}
```

Note: `GetPlannedVU` is called with `modifyActiveVUCount=false` (line 594). The executor manages its own active VU count bookkeeping in the closure.

**`returnVU` closure** (lines 605–610):

```go
returnVU := func(initVU lib.InitializedVU) {
    rs.executor.executionState.ReturnVU(initVU, false)             // line 606: return VU to channel
    atomic.AddInt64(rs.activeVUsCount, -1)                         // line 607: decrement display counter
    rs.wg.Done()                                                   // line 608: decrement WaitGroup
    rs.executor.executionState.ModCurrentlyActiveVUsCount(-1)      // line 609: decrement global counter
}
```

Note: `ReturnVU` is called with `wasActive=false` (line 606) for the same reason — the executor handles its own counter bookkeeping in the closures.

### How `returnVU` Is Connected to VU Deactivation

The connection is through the VU activation parameters:

1. **`vuHandle.start()`** at lines 133–134 of `vu_handle.go`:

```go
vh.activeVU = vh.initVU.Activate(getVUActivationParams(
    vh.ctx, *vh.config, vh.returnVU, vh.nextIterationCounters))
```

2. **`getVUActivationParams()`** at `helpers.go` lines 251–264:

```go
func getVUActivationParams(
    ctx context.Context, conf BaseConfig, deactivateCallback func(lib.InitializedVU),
    nextIterationCounters func() (uint64, uint64),
) *lib.VUActivationParams {
    return &lib.VUActivationParams{
        RunContext:               ctx,
        Scenario:                 conf.Name,
        Exec:                     conf.GetExec(),
        Env:                      conf.GetEnv(),
        Tags:                     conf.GetTags(),
        DeactivateCallback:       deactivateCallback,   // line 261: THIS IS returnVU
        GetNextIterationCounters: nextIterationCounters,
    }
}
```

The `deactivateCallback` parameter is `vh.returnVU` (which is the `returnVU` closure from `runLoopsIfPossible`). When the VU's context is cancelled and the VU deactivates, the VU runtime invokes `DeactivateCallback`, which calls `returnVU` with the `InitializedVU`. This writes the VU back to the channel buffer.

### The 1:1 Contract

The `getVU`/`returnVU` contract is strictly 1:1, enforced by the `vuHandle` state machine:

- `getVU()` is called **only** in `vuHandle.start()` at line 128, and only when transitioning from `stopped` or `toHardStop` to `starting`:

```go
case stopped, toHardStop:
    vh.logger.Debug("Start")
    vh.initVU, err = vh.getVU()   // line 128: acquire VU from buffer
```

- The VU is stored in `vh.initVU` (line 128) and activated into `vh.activeVU` (line 133).
- When the VU deactivates (context cancelled), the `DeactivateCallback` = `returnVU` is invoked by the VU runtime.
- Each `getVU()` call results in exactly one VU activation, which results in exactly one deactivation callback, which calls `returnVU()` exactly once.

This is also reinforced by the design contract documented at `vu_handle.go` line 62:

> *"for each call to getVU there must be 1 (and only 1) call to returnVU"*

### Test Evidence

`TestVUHandleRace` in `vu_handle_test.go` explicitly verifies the 1:1 contract at line 110:

```go
require.Equal(t, atomic.LoadInt64(&getVUCount), atomic.LoadInt64(&returnVUCount))
```

After 10,000 `start()` calls, 1,000 `gracefulStop()` calls, and 100 `hardStop()` calls running concurrently, the test asserts that `getVUCount` equals `returnVUCount`. This confirms that no VU buffer leak exists under heavy concurrent access.

### Verdict: NO LEAK (BY DESIGN)

The `getVU`/`returnVU` contract is strictly 1:1, enforced by the `vuHandle` state machine and the VU activation/deactivation lifecycle. Every `getVU()` results in exactly one `returnVU()` via the `DeactivateCallback`. The `TestVUHandleRace` test verifies this property under concurrent access at line 110 of `vu_handle_test.go`.

---

## Summary of Findings

| # | Symptom | Verdict | Root Cause |
|---|---------|---------|------------|
| Q1 | VU "stuck" state during rapid stage transitions | **By Design** | The `toGracefulStop` state (value 3) allows the current iteration to complete. `start()` can resume from `toGracefulStop` → `running` without VU reacquisition (lines 122–125 of `vu_handle.go`). |
| Q2 | Handler VU count mismatch | **By Design** | Two independent closure-local `cur` counters tracking different concepts: `scheduledVUsHandlerStrategy.cur` = active VU target; `maxAllowedVUsHandlerStrategy.cur` = graceful reservation ceiling. |
| Q3 | VUs exceeding `gracefulStop` after Ctrl+C | **Mostly By Design** | Context cancellation propagates correctly through the chain. Apparent persistence is due to `RunOnce()` blocking without checking `ctx.Done()` — a script-level responsibility (see `helpers.go` line 108). |
| Q4 | Execution segment VU count imbalance | **User Misconfiguration (Likely)** | Using `--execution-segment` without `--execution-segment-sequence` causes each instance to scale independently via per-instance filled sequences. With a shared sequence, the striping algorithm (`ScaleInt64()` at line 580) guarantees exact partition. |
| Q5 | Race condition between handler goroutines | **By Design (No Race)** | Both handlers execute sequentially in `iterateSteps()` (lines 634, 640). The only concurrency is with VU goroutines after `runRemainingGracefulSteps()` (line 554), synchronized via `vuHandle.mutex`. |
| Q6 | VU buffer leak | **No Leak (By Design)** | 1:1 `getVU`/`returnVU` contract enforced by `vuHandle` state machine and `DeactivateCallback`. Verified by `TestVUHandleRace` assertion at line 110 of `vu_handle_test.go`. |

### Key Synchronization Mechanisms

| Mechanism | Location | Purpose |
|-----------|----------|---------|
| `vuHandle.mutex` (`sync.Mutex`) | `vu_handle.go` line 71 | Protects all state transitions in `start()`, `gracefulStop()`, `hardStop()`, and the slow path of `runLoopsIfPossible()` |
| `atomic.LoadInt32(&vh.state)` | `vu_handle.go` line 204 | Lock-free fast-path read in `runLoopsIfPossible()` — if state is `running`, proceed without mutex |
| `canStartIter` channel | `vu_handle.go` line 80 | Signal from `start()` to the VU goroutine that it can begin iterating; recreated on each `gracefulStop()`/`hardStop()` |
| `vh.parentCtx.Done()` | `vu_handle.go` line 197 | Signals total executor shutdown to VU goroutines |
| `context.WithCancel(parentCtx)` | `vu_handle.go` line 96 | Creates VU-scoped context; cancelled on `gracefulStop` (in slow path) and `hardStop` |
| `sync.WaitGroup` (`rs.wg`) | `ramping_vus.go` line 571 | Tracks active VU goroutines; `wg.Wait()` deferred in `Run()` at line 540 before context cancel |
| Atomic counters in `ExecutionState` | `execution.go` lines 125–153 | `activeVUs`, `initializedVUs`, `fullIterationsCount` — UI/metrics only, explicitly documented as NOT for synchronization (line 65) |
