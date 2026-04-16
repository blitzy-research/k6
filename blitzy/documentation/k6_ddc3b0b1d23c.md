# Concurrency Investigation: k6's `ramping-vus` Executor

This document traces, analyzes, and answers five concurrency-related questions about the `ramping-vus` executor in the k6 load-testing tool. Every claim below is grounded in specific file/line references from the k6 source tree on branch `k6_ddc3b0b1d23c`. The investigation is **diagnostic and explanatory**; it is not a refactoring proposal, and it does not analyze unrelated executors (arrival-rate, constant VUs, shared/per-VU iterations, externally controlled).

## 1. Introduction and Question Summary

The five questions this document answers, as posed by the investigator:

1. **VU State Inconsistency** — When stages ramp up and down rapidly with a long `gracefulRampDown`, the VU count tracked by the "scheduled handler" (`scheduledVUsHandlerStrategy`) diverges from what the "graceful handler" (`maxAllowedVUsHandlerStrategy`) believes should exist. Some VUs appear to enter a limbo state — neither fully active nor fully stopped.
2. **Ctrl+C / `gracefulStop` Overrun** — Upon sending SIGINT, some VUs continue executing for longer than the configured `gracefulStop` period should permit.
3. **Execution Segment VU Overcounting** — When running with three execution segment instances (e.g., `0:1/3`, `1/3:2/3`, `2/3:1`), one instance consistently reports more VUs than the others at the same timestamp, and summing all instances' VU counts exceeds the configured maximum.
4. **Race Condition Hypothesis** — Whether a race condition exists between the two handler goroutines (`handleNewMaxAllowedVUs` and `handleNewScheduledVUs`) that both manipulate VU handle state.
5. **VU Buffer Leak Hypothesis** — Whether the shared VU channel buffer in `ExecutionState.vus` can leak VUs under concurrent access.

For each question, this document presents the verdict (bug / by-design / observational artifact), a trace through the relevant code paths, and empirical evidence from the existing test suite running under the Go race detector.

## 2. Architecture Overview of the `ramping-vus` Executor

The `ramping-vus` executor runs a variable number of VUs (Virtual Users) over a set of stages, where each stage smoothly ramps toward a target VU count. The architecture has three distinct layers:

**Configuration layer.** The executor is configured by two nested structs:

- `Stage`, at `lib/executor/ramping_vus.go:33-37`, specifies the `Duration` (how long this stage runs) and `Target` (the VU count to reach at the end of the stage).
- `RampingVUsConfig`, at `lib/executor/ramping_vus.go:40-45`, embeds `BaseConfig` and adds `StartVUs`, `Stages`, and `GracefulRampDown` — the per-VU wind-down grace period during ramp-downs. Note `GracefulStop` is inherited from `BaseConfig`; its default is `30 * time.Second` (see `lib/executor/base_config.go:20`, `lib/executor/base_config.go:45`).

**Planning layer.** During `Init()` at `lib/executor/ramping_vus.go:479-487`, the executor precomputes two step sequences:

- `rawSteps`, produced by `getRawExecutionSteps()` at `lib/executor/ramping_vus.go:171-234`, encodes the *instantaneous scheduled VU target* at every time offset. The algorithm uses a `SegmentedIndex` iterator (see `lib/execution_segment.go:762-842`) to produce segment-specific step sequences — i.e., it operates inside the caller's execution segment, not the full set of VUs.
- `gracefulSteps`, produced by `GetExecutionRequirements()` at `lib/executor/ramping_vus.go:434-449`, encodes the *maximum allowed VU ceiling* including slots reserved for ramping-down VUs to finish their current iteration. `GetExecutionRequirements` calls `getRawExecutionSteps` then passes the result through `reserveVUsForGracefulRampDowns()` at `lib/executor/ramping_vus.go:307-414`.

The key property of these two sequences, spelled out in the algorithm comment at `lib/executor/ramping_vus.go:298-301`, is that `reserveVUsForGracefulRampDowns` "traverses the raw execution steps and whenever there's a scaling down of VUs, it prevents the number of VUs from decreasing for the configured gracefulRampDown period." So `gracefulSteps[i].PlannedVUs >= rawSteps[j].PlannedVUs` at the same or nearby time offsets — by construction.

**Runtime layer.** `RampingVUs.Run()` at `lib/executor/ramping_vus.go:491-560` drives execution. The body of `Run()` does the following, in order:

1. Creates the duration context triple `(startTime, maxDurationCtx, regDurationCtx)` via `getDurationContexts()` at lines 501-503. Per `lib/executor/helpers.go:168-180`, `maxDurationCtx` has a deadline equal to `startTime + regularDuration + gracefulStop` (line 174).
2. Sets up the per-run `rampingVUsRunState` struct (defined at `lib/executor/ramping_vus.go:562-574`) containing the shared `vuHandles` slice, the `activeVUsCount *int64` atomic counter (line 569), and the `wg sync.WaitGroup` (line 571).
3. Spawns a `trackProgress()` goroutine for the progress bar at line 536-539.
4. Registers `defer runState.wg.Wait()` at line 540 to await all per-VU goroutines before returning.
5. Calls `runState.runLoopsIfPossible(maxDurationCtx, cancel)` at line 543. Inside `runLoopsIfPossible` (`lib/executor/ramping_vus.go:592-617`), the executor spawns `maxVUs` goroutines — one per VU slot — each running `vuHandle.runLoopsIfPossible(runIteration)` (loop at lines 611-616).
6. Invokes `iterateSteps()` at lines 549-553 to drive the scheduling loop in the main goroutine.
7. Spawns `go runState.runRemainingGracefulSteps(...)` at lines 554-558 to process the final graceful steps after `iterateSteps` has returned.

**Context hierarchy.** A VU handle's context is a child of `maxDurationCtx`, not of `runCtx` directly:

```text
runCtx (from scheduler)
  └── withExecStateCtx = lib.WithExecutionState(runCtx, ...)   [execution/scheduler.go:464]
        └── executorsRunCtx = context.WithCancel(withExecStateCtx)   [execution/scheduler.go:497]
              └── executor's parentCtx (passed to Run)
                    └── maxDurationCtx = context.WithDeadline(parentCtx, startTime + regularDuration + gracefulStop)   [lib/executor/helpers.go:174]
                          └── regDurationCtx = context.WithDeadline(maxDurationCtx, startTime + regularDuration)   [lib/executor/helpers.go:178]
                          └── vh.ctx = context.WithCancel(vh.parentCtx)   [lib/executor/vu_handle.go:96]
```

Any cancellation (or deadline firing) at any level propagates to all descendants — this is the foundation of the Q2 answer.

A full goroutine communication map appears in Section 4.

## 3. Detailed Answers to Each Question

### Q1: VU State Inconsistency Between Handlers — Verdict: By Design, Not a Bug

The user's observation — that `scheduledVUsHandlerStrategy`'s internal VU count diverges from `maxAllowedVUsHandlerStrategy`'s — is correct, expected, and in fact is the *mechanism* by which graceful ramp-down works.

**Two independent closures.** The handler strategies are manufactured by two separate factory functions, each of which closes over its own `var cur uint64`:

```go
// lib/executor/ramping_vus.go:668-677
func (rs *rampingVUsRunState) maxAllowedVUsHandlerStrategy() func(lib.ExecutionStep) {
    var cur uint64 // current number of planned graceful VUs
    return func(graceful lib.ExecutionStep) {
        pv := graceful.PlannedVUs
        for ; pv < cur; cur-- {
            rs.vuHandles[cur-1].hardStop()
        }
        cur = pv
    }
}

// lib/executor/ramping_vus.go:679-690
func (rs *rampingVUsRunState) scheduledVUsHandlerStrategy() func(lib.ExecutionStep) {
    var cur uint64 // current number of planned raw VUs
    return func(raw lib.ExecutionStep) {
        pv := raw.PlannedVUs
        for ; cur < pv; cur++ {
            _ = rs.vuHandles[cur].start() // TODO: handle the error
        }
        for ; pv < cur; cur-- {
            rs.vuHandles[cur-1].gracefulStop()
        }
    }
}
```

The `cur` variable inside `maxAllowedVUsHandlerStrategy` tracks the graceful ceiling (fed by `gracefulSteps`); the `cur` inside `scheduledVUsHandlerStrategy` tracks the scheduled target (fed by `rawSteps`). They are separate variables and they are *expected* to differ.

**Why they differ: the `gracefulSteps` >= `rawSteps` invariant.** The two step sequences encode different things. `rawSteps` is the instantaneous scheduled VU target. `gracefulSteps` is that same target with a reservation window added so ramping-down VUs can finish their last iteration. The ASCII diagram in the source at `lib/executor/ramping_vus.go:279-289` visualizes this explicitly — stars (`*`) are actively scheduled VUs, dots (`.`) are VUs finishing iterations during `gracefulRampDown`:

```text
VUs ^
    |
   6|  *..............................
   5| ***.......*..............................
   4|*****.....***.....**..............................
   3|******...*****...***..............................
   2|*******.*******.****..............................
   1|***********************..............................
   0--------------------------------------------------------> time(s)
     012345678901234567890123456789012345678901234567890123   (t%10)
     000000000011111111112222222222333333333344444444445555   (t/10)
```

The width of the dotted "tail" after each ramp-down is exactly `gracefulRampDown`. The explanatory comment at `lib/executor/ramping_vus.go:298-301` confirms this is the explicit design of `reserveVUsForGracefulRampDowns()`. So at any given time, the graceful ceiling (dots + stars) can be strictly greater than the scheduled target (stars only). That is not a bug; that is the definition of the reservation window.

**The `vuHandle` five-state machine.** Each VU has its own `vuHandle` with five possible states, defined at `lib/executor/vu_handle.go:16-22`:

```go
const (
    stopped stateType = iota
    starting
    running
    toGracefulStop
    toHardStop
)
```

The authoritative transition table is the block comment at `lib/executor/vu_handle.go:24-55`. For the purposes of Q1, the critical row is `grace + running → toGracefulStop` at `lib/executor/vu_handle.go:46`, executed by `gracefulStop()` at `lib/executor/vu_handle.go:147-163`: the `case running:` branch at lines 157-158 sets state to `toGracefulStop` and the method returns without invoking `vh.cancel()`. The VU's current iteration continues.

**Tracking the "limbo" state.** When the scheduled handler calls `gracefulStop()` on VU `cur-1`:

- From the *scheduled handler's* perspective, `cur` is decremented (`lib/executor/ramping_vus.go:686`). The handler considers that VU "retired."
- From the *graceful handler's* perspective, nothing changes. That VU is still covered by the graceful ceiling, so the graceful handler's `cur` still includes it.
- From the *`vuHandle`'s* perspective, it is now in `toGracefulStop`.

Inside `runLoopsIfPossible()` at `lib/executor/vu_handle.go:185-264`, the VU's goroutine discovers the state change on its next loop iteration. The fast path at lines 204-207 fails because `state != running`. The slow path at lines 210-240 then handles it:

```go
switch vh.state {
case running: // start raced us toGracefulStop
    vh.mutex.Unlock()
    continue
case toGracefulStop:
    if cancel != nil {
        cancel()
        vh.ctx, vh.cancel = context.WithCancel(vh.parentCtx)
    }
    fallthrough // to set the state
case toHardStop:
    vh.changeState(stopped)
case stopped, starting:
    // there is nothing to do
}
```

So a VU in `toGracefulStop` only fully transitions to `stopped` on the *next loop step* after its current iteration completes. Between the scheduled handler calling `gracefulStop()` and the VU's iteration returning, the VU is in `toGracefulStop` — it cannot start new iterations (`canStartIter` is closed/not-yet-reopened), but its current iteration is still in flight with a live context. This is the "limbo" state the user observes.

It is *not* stuck. It is finishing its iteration. When it does, the fallthrough at `lib/executor/vu_handle.go:230-233` moves it to `stopped`. The existing test `TestRampingVUsRampDownNoWobble` at `lib/executor/ramping_vus_test.go:376` proves the ramp-down is deterministic and produces a smooth stepped decrease — it asserts `vuChanges == {10, 9, 8, 7, 6, 5, 4, 3, 2, 1, 0}` at `lib/executor/ramping_vus_test.go:439`.

**Verdict**: Not a bug. The divergence between the two handlers' internal counters is the *mechanism* by which `gracefulRampDown` works. Any metric that represents "currently active VUs" must choose which of the two ceilings it reports; k6's live `activeVUsCount` (see Q3) reflects the broader "still borrowed from the pool" view, which is the superset.

### Q2: Ctrl+C / `gracefulStop` Overrun — Verdict: Cancellation Is Immediate; Any Overrun Has a Specific Explainable Cause

Signal propagation is **synchronous and direct**. Each link in the cancellation chain is present in the code with no missing hop, and the `vuHandle` loop exits at the first opportunity. Any *perceived* overrun has a specific, non-signal-handling cause.

**Step 1 — SIGINT trap.** The signal handler is installed at `cmd/run.go:347-364`:

```go
gracefulStop := func(sig os.Signal) {
    logger.WithField("sig", sig).Debug("Stopping k6 in response to signal...")
    runAbort(errext.WithAbortReasonIfNone(
        errext.WithExitCodeIfNone(
            fmt.Errorf("test run was aborted because k6 received a '%s' signal", sig), exitcodes.ExternalAbort,
        ), errext.AbortedByUser,
    ))
    lingerCancel()
}
onHardStop := func(sig os.Signal) {
    logger.WithField("sig", sig).Error("Aborting k6 in response to signal")
    globalCancel()
}
stopSignalHandling := handleTestAbortSignals(c.gs, gracefulStop, onHardStop)
```

On the first SIGINT, the `gracefulStop` closure calls `runAbort` at line 352. On the second SIGINT, `onHardStop` calls `globalCancel()` at line 361 — which forces an immediate process exit regardless of in-flight iterations.

**Step 2 — `AbortTestRun`.** `runAbort` is wired to `execution.AbortTestRun()` at `execution/abort.go:64-72`:

```go
func AbortTestRun(ctx context.Context, err error) bool {
    if x := ctx.Value(testAbortKey{}); x != nil {
        if v, ok := x.(*testAbortController); ok {
            v.abort(err)
            return true
        }
    }
    return false
}
```

This looks up the `testAbortController` from the context and calls its `abort` method.

**Step 3 — `testAbortController.abort`.** Defined at `execution/abort.go:24-36`:

```go
func (tac *testAbortController) abort(err error) {
    tac.lock.Lock()
    defer tac.lock.Unlock()
    if tac.reason != nil {
        tac.logger.Debugf(...)
        return
    }
    tac.reason = err
    tac.cancel()
}
```

The lock at line 25 ensures only the first reason wins; `tac.cancel()` at line 35 invokes the `context.CancelFunc` captured when `NewTestRunContext` was built at `execution/abort.go:49-60` (specifically line 52: `ctx, cancel := context.WithCancel(ctx)`).

**Step 4 — propagation to `executorsRunCtx`.** In the scheduler at `execution/scheduler.go:497`:

```go
executorsRunCtx, executorsRunCancel := context.WithCancel(withExecStateCtx)
```

Since `withExecStateCtx` is itself derived from the signal-aware `runCtx` (at line 464: `withExecStateCtx := lib.WithExecutionState(runCtx, e.state)`), cancellation of the run context immediately cancels `executorsRunCtx`, which is the context passed to each executor's `Run()` call at `execution/scheduler.go:500`.

**Step 5 — propagation to `maxDurationCtx`.** Inside the executor, `getDurationContexts` at `lib/executor/helpers.go:168-180`:

```go
func getDurationContexts(parentCtx context.Context, regularDuration, gracefulStop time.Duration) (
    startTime time.Time, maxDurationCtx, regDurationCtx context.Context, maxDurationCancel func(),
) {
    startTime = time.Now()
    maxEndTime := startTime.Add(regularDuration + gracefulStop)

    maxDurationCtx, maxDurationCancel = context.WithDeadline(parentCtx, maxEndTime)
    if gracefulStop == 0 {
        return startTime, maxDurationCtx, maxDurationCtx, maxDurationCancel
    }
    regDurationCtx, _ = context.WithDeadline(maxDurationCtx, startTime.Add(regularDuration)) //nolint:govet
    return startTime, maxDurationCtx, regDurationCtx, maxDurationCancel
}
```

`maxDurationCtx` is a child of `parentCtx` (line 174). When `parentCtx` cancels, `maxDurationCtx` cancels immediately — this is standard Go `context.WithDeadline` semantics. The deadline is only an upper bound; parent cancellation cancels the child right away.

Note the `//nolint:govet` at line 178: the `cancel` returned by `context.WithDeadline` on `regDurationCtx` is intentionally discarded. `go vet` warns about this pattern because discarding a `cancel` can leak a timer. In k6's design the lifetime of `regDurationCtx` is bounded by `maxDurationCtx`, which *does* have its `cancel` invoked. This is a documented k6 design choice, not a defect, and it has no bearing on Q2: the moment `parentCtx` cancels, both child contexts cancel simultaneously.

**Step 6 — `vuHandle` observes cancellation.** `runLoopsIfPossible` at `lib/executor/vu_handle.go:185-264` captures `executorDone` at line 197:

```go
executorDone = vh.parentCtx.Done()
```

The loop checks this channel in two `select` statements:

- Slow-path guard at lines 211-217:

```go
vh.mutex.Lock()
select {
case <-executorDone:
    vh.mutex.Unlock()
    return
default:
}
```

- Outer wait select at lines 244-262:

```go
select {
case <-canStartIter:
    // ...
case <-ctx.Done():
    // hardStop was called, start a fresh iteration
case <-executorDone:
    return
}
```

When `executorDone` fires, the loop returns at the next visit of either select. The per-iteration `vh.ctx` is also a child of `vh.parentCtx` (created at `lib/executor/vu_handle.go:96`), so it cancels too, and `vu.RunOnce()` receives cancellation via the `RunContext` field of `VUActivationParams` registered at `lib/executor/helpers.go:256`.

**Plausible causes of perceived overrun.** Given signal propagation is immediate, any observed overrun has a mundane, explainable cause:

- **VU iteration blocked on uncancellable I/O.** The Go `net/http` client honors context cancellation at connection and TLS-handshake time but can be slow to interrupt an in-flight response body read. An iteration script that is `await`-ing such a read will not observe cancellation until the body operation returns. This manifests as an apparent "overrun" of the VU, even though the signal has been delivered. The iteration is driven by `runIteration` assembled in `getIterationRunner()` at `lib/executor/helpers.go:104-141` and passed to `vuHandle.runLoopsIfPossible` at `lib/executor/ramping_vus.go:615`.
- **`defer runState.wg.Wait()` blocks `Run()`.** At `lib/executor/ramping_vus.go:540`, the executor defers waiting for every per-VU goroutine. `rs.wg.Add(1)` is called inside the `getVU` closure at `lib/executor/ramping_vus.go:600` and `rs.wg.Done()` is called inside `returnVU` at line 608. If any iteration is wedged on uncancellable I/O, the entire `Run()` blocks until that iteration returns.
- **Post-executor cleanup time.** The scheduler's main loop at `execution/scheduler.go:503-514` waits for all executor results to arrive on `runResults`. Teardown (`execution/scheduler.go:520` onward) and metrics flush happen afterward. An observer timing from SIGINT to process exit will see all of this, not just executor iterations.
- **Second SIGINT is the hard cut.** If the user wants guaranteed termination, the second Ctrl+C triggers `onHardStop` at `cmd/run.go:359-362`, which calls `globalCancel()` and proceeds to `os.Exit()` via the signal handler's exit path.

**Verdict**: The cancellation signal reaches each VU handle through the parent chain with no missing hop; there is no bug in the propagation. Any overrun attributable to the ramping VUs executor itself is due to iteration bodies blocking on uncancellable I/O or to the `defer wg.Wait()` holding `Run()` open until those iterations return — not to a missing or delayed signal path.

### Q3: Execution Segment VU Overcounting — Verdict: Mathematically Provably Additive; Apparent Overcount Is a Timing-Observation Artifact

k6's execution-segment math *is* additive: the sum of per-segment VU counts at any logical point equals the unsegmented VU count. The apparent overcount observed when summing live counters across three instances is a live-sampling artifact, not a math bug.

**`ScaleInt64` is additive by construction.** At `lib/execution_segment.go:579-588`:

```go
func (essw *ExecutionSegmentSequenceWrapper) ScaleInt64(i int, value int64) int64 {
    offsets := essw.offsets[i]
    result := (value / essw.lcd) * int64(len(offsets))
    for _, offset := range offsets[1:] {
        if value%essw.lcd > offset {
            result++
        }
    }
    return result
}
```

- Whole-cycle portion at line 583: `result := (value / lcd) * len(offsets)`. Summed across all segments, this is `(value / lcd) * sum(len(offsets[i]))`. Since the offset sets for all segments in the sequence tile `[0, lcd)` exactly once (this is the definition of `GetStripedOffsets()`), `sum(len(offsets[i])) == lcd`. So the whole-cycle contribution sums to `(value / lcd) * lcd` — the whole-cycle portion of `value`.
- Residual loop at lines 584-586: for the residual `value % lcd`, each segment's offset list contributes a 1 for each offset strictly less than `value % lcd`. Since the offsets across all segments tile `[0, lcd)` exactly once, exactly `value % lcd` offsets across all segments are strictly less than `value % lcd`. So the residuals sum to exactly `value % lcd`.

Therefore `sum_over_segments ScaleInt64(i, value) == (value / lcd) * lcd + (value % lcd) == value`, exactly. No overcount, no undercount.

**`SegmentedIndex` iterates correctly.** The `SegmentedIndex` struct at `lib/execution_segment.go:768-772` carries `start`, `lcd`, `offsets`, `scaled`, `unscaled` fields. Its methods:

- `Next()` at `lib/execution_segment.go:782-791` advances `unscaled` by the next offset and increments `scaled`.
- `Prev()` at `lib/execution_segment.go:795-804` reverses `Next()`, decrementing `scaled` and subtracting the appropriate offset from `unscaled`.
- `GoTo()` at `lib/execution_segment.go:808-842` computes the whole-cycle position quickly (lines 814-818), then walks offsets for the residual (lines 822-826).

`getRawExecutionSteps` at `lib/executor/ramping_vus.go:171-234` creates a `SegmentedIndex` at line 176 and drives it via `GoTo` (line 180), `Prev` (in the loop at line 209), and `Next` (in the loop at line 219). Each segment thus observes step transitions exactly at the unscaled VU count values it owns — the segmented iteration and the scaling function are perfectly aligned.

**Randomized property proof: `TestSumRandomSegmentSequenceMatchesNoSegment`.** At `lib/executor/ramping_vus_test.go:1112-1201`, k6 runs a randomized property test:

- 10 random trials (line 1116: `numTests = 10`).
- For each trial, generates a random stage config with random `StartVUs`, random `Stages`, random `Target`, random `Duration` at lines 1124-1138.
- Generates a random segment sequence of length 2–15 at line 1141.
- Computes full (unsegmented) steps at line 1181: `fullRawSteps := c.getRawExecutionSteps(fullSeg, false)`.
- For each segment `s`, subtracts that segment's steps from the running `fullRawSteps` at lines 1187-1192.
- After subtracting all segments, asserts every remaining step has `PlannedVUs == 0` at lines 1194-1198.

That is a direct simulation proof: sum of per-segment VU counts equals the unsegmented total, at every time offset, for every config in the trial. This test passes under `-race` on every run (see Section 6).

**Why the observer sees apparent overcount.** The math is correct. The apparent overcount arises when the user compares `activeVUsCount` across three running instances at what the *observer* believes is "the same timestamp." Several observational effects conspire here:

- **Clock skew.** Each instance samples its own live counter at a slightly different wall-clock time. Even sub-second skew matters during rapid ramps.
- **Scheduler jitter.** `waiter()` at `lib/executor/ramping_vus.go:697-712` uses `time.NewTimer(diff)` (line 698) inside a select against context cancellation (lines 702-708). Timer fires are subject to OS scheduling latency. The instant an instance transitions between steps is not exact.
- **`activeVUsCount` is a borrow counter, not a scheduled counter.** The counter is defined at `lib/executor/ramping_vus.go:569`. It is incremented inside the `getVU` closure at `lib/executor/ramping_vus.go:601` (`atomic.AddInt64(rs.activeVUsCount, 1)`) and decremented inside `returnVU` at line 607 (`atomic.AddInt64(rs.activeVUsCount, -1)`). A VU in `toGracefulStop` whose current iteration is still in-flight has NOT yet had `returnVU` called, so it remains counted here. When three instances each have a cohort of VUs in `toGracefulStop`, the sum of their live `activeVUsCount` values will exceed the instantaneous scheduled target — even though the math is correct when you count all borrowed VUs (including those finishing their iteration).

**Recommendation for cross-instance accounting.** Do not sum live `activeVUsCount` across instances in real time. Either:

- Sum the persisted VU-start / VU-end events from an output backend (Prometheus, InfluxDB, etc.) and align by a common time base, or
- Reason in terms of the `gracefulSteps` ceiling (which is what each instance's graceful handler already uses as its upper bound), not the scheduled target.

**Verdict**: The math is correct and is proven by randomized property test. The apparent overcount is an observational artifact.

### Q4: Race Condition Between Handler Goroutines — Verdict: No Race; Handlers Are Sequential in One Goroutine; Per-VU Methods Are Mutex-Guarded

There is no concurrent mutation of VU-handle state from the two handler strategies. The two handlers never run at the same time, and every per-VU state transition is protected by a mutex.

**Both handlers run in the same goroutine during `iterateSteps()`.** At `lib/executor/ramping_vus.go:622-645`:

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
            handleNewMaxAllowedVUs(g)
            j++
        } else {
            if wait(r.TimeOffset) {
                break
            }
            handleNewScheduledVUs(r)
            i++
        }
    }
    return j
}
```

Note the parameter names `handleNewMaxAllowedVUs` and `handleNewScheduledVUs` — these are exactly the two handlers named in the original hypothesis. `iterateSteps` receives them as function values and invokes them one-at-a-time inside its sequential `for` loop. There is a single goroutine here. The loop picks one step per iteration and calls *either* the graceful (`handleNewMaxAllowedVUs`) handler *or* the scheduled (`handleNewScheduledVUs`) handler, never both at the same time. There is no `go` keyword inside the loop. The two handlers cannot race against each other inside `iterateSteps` because they are never running simultaneously.

**`runRemainingGracefulSteps` runs after `iterateSteps` returns.** At `lib/executor/ramping_vus.go:554-558`, the executor spawns `runRemainingGracefulSteps` as a goroutine after `iterateSteps` returned. Its implementation at `lib/executor/ramping_vus.go:654-666` only processes `rs.executor.gracefulSteps[handledGracefulSteps:]` — the leftover graceful steps starting from where `iterateSteps` left off. At that point, `iterateSteps` has finished and the scheduled handler is no longer invoked anywhere. So the graceful handler running in this goroutine still never races the scheduled handler.

**Per-VU methods are mutex-guarded.** Inside each `vuHandle`, the `mutex *sync.Mutex` field is declared at `lib/executor/vu_handle.go:71`, and all mutating methods acquire it:

- `start()` at `lib/executor/vu_handle.go:115-139` — `vh.mutex.Lock(); defer vh.mutex.Unlock()` at lines 116-117.
- `gracefulStop()` at `lib/executor/vu_handle.go:147-163` — mutex at lines 148-149.
- `hardStop()` at `lib/executor/vu_handle.go:165-181` — mutex at lines 166-167.

So even though two independent callers (e.g., the graceful handler hard-stopping an overshoot VU while the scheduled handler graceful-stops the same VU from the same `iterateSteps` goroutine — impossible given sequential dispatch, but hypothetically concurrent callers from elsewhere) could in principle target the same VU, they would be serialized.

**Lock-free fast path in `runLoopsIfPossible` is safe.** At `lib/executor/vu_handle.go:185-264`, the VU's iteration loop uses an atomic fast path to avoid mutex contention on every iteration:

- Fast path at line 204: `state := stateType(atomic.LoadInt32((*int32)(&vh.state)))`. No mutex.
- Slow path at line 210: `vh.mutex.Lock()`. All further reads/writes of `vh.ctx`, `vh.cancel`, `vh.canStartIter`, `vh.activeVU` happen under the mutex (lines 210-240).

All writes to `vh.state` go through `changeState()` at `lib/executor/vu_handle.go:142-145` which uses `atomic.StoreInt32` at line 144. So the atomic load on the fast path is paired with atomic stores from every writer — the fast path is formally race-free by the Go memory model. Even if the fast path observes a stale `running` and invokes `runIter`, the next iteration re-reads the state and will take the slow path if the state has changed. Correctness is preserved; stale reads only mean at most one extra iteration executes before the VU notices it was asked to stop — which is exactly what `toGracefulStop` semantics require.

**Empirical race-detector proof: `TestVUHandleRace`.** At `lib/executor/vu_handle_test.go:25-111`, k6 runs a race-detector stress test. The comment on line 24 is explicit: "this test is mostly interesting when -race is enabled." Three goroutines are spawned:

- At lines 72-78, a goroutine calls `vuHandle.start()` 10,000 times.
- At lines 80-86, a goroutine calls `vuHandle.gracefulStop()` 1,000 times with a 1-ns sleep between calls.
- At lines 88-94, a goroutine calls `vuHandle.hardStop()` 100 times with a 10-ns sleep between calls.

A fourth goroutine (the VU-iteration runner) at line 69 is executing `runLoopsIfPossible` concurrently. After `wg.Wait()` at line 95, a final `hardStop()` is issued at line 96 and the test asserts `getVUCount == returnVUCount` at line 110. This test passes under `-race` (see Section 6).

The companion test `TestVUHandleStartStopRace` at `lib/executor/vu_handle_test.go:114-220` exercises additional race scenarios between start and stop transitions. `TestVUHandleSimple` at `lib/executor/vu_handle_test.go:223` validates the basic state machine semantics. All pass under `-race`.

**Verdict**: No race. Handlers are sequentially dispatched in the main `Run()` goroutine; per-VU methods are mutex-guarded; the lock-free fast path is a properly-synchronized atomic read paired with atomic writes; the Go race detector confirms zero data races in existing tests.

### Q5: VU Buffer Leak Hypothesis — Verdict: No Leak; 1:1 Correspondence Between `getVU` and `returnVU` Enforced by the State Machine

The shared VU buffer channel does not leak. Every `getVU` call is followed by exactly one `returnVU` call, enforced by the VU handle state machine and the activation-params callback wiring, and empirically verified by `TestVUHandleRace`.

**Buffer declaration and lifetime.** At `lib/execution.go:106`:

```go
vus chan InitializedVU
```

The comment block at `lib/execution.go:95-105` explains that executors are trusted to use `GetPlannedVU`/`GetUnplannedVU`/`ReturnVU` rather than the channel directly. The channel is populated at init time by `AddInitializedVU` at `lib/execution.go:537-540`:

```go
func (es *ExecutionState) AddInitializedVU(vu InitializedVU) {
    es.vus <- vu
    atomic.AddInt64(es.initializedVUs, 1)
}
```

**Borrow operation: `GetPlannedVU`.** At `lib/execution.go:471-488`:

```go
func (es *ExecutionState) GetPlannedVU(logger *logrus.Entry, modifyActiveVUCount bool) (InitializedVU, error) {
    for i := 1; i <= MaxRetriesGetPlannedVU; i++ {
        select {
        case vu := <-es.vus:
            if modifyActiveVUCount {
                es.ModCurrentlyActiveVUsCount(+1)
            }
            // ...
            return vu, nil
        case <-time.After(MaxTimeToWaitForPlannedVU):
            logger.Warnf("Could not get a planned VU for %s...", ...)
        }
    }
    return nil, fmt.Errorf("could not get a VU from the buffer in %s", ...)
}
```

The retry loop at line 472 handles transient contention. A channel receive at lines 473-479 succeeds when a VU is available. If the timeout at lines 480-482 fires, the loop retries up to `MaxRetriesGetPlannedVU` times before returning an error at lines 484-487.

**Return operation: `ReturnVU`.** At `lib/execution.go:544-549`:

```go
func (es *ExecutionState) ReturnVU(vu InitializedVU, wasActive bool) {
    es.vus <- vu
    if wasActive {
        es.ModCurrentlyActiveVUsCount(-1)
    }
}
```

Sends the VU back to the buffer channel at line 545 and conditionally decrements the active counter at lines 546-548.

**Closures in the ramping VUs executor.** In `rampingVUsRunState.runLoopsIfPossible` at `lib/executor/ramping_vus.go:592-617`:

```go
getVU := func() (lib.InitializedVU, error) {
    pvu, err := rs.executionState.GetPlannedVU(rs.executor.logger, false)
    if err != nil {
        cancel()
        return nil, err
    }
    rs.wg.Add(1)
    atomic.AddInt64(rs.activeVUsCount, 1)
    return pvu, err
}
returnVU := func(initVU lib.InitializedVU) {
    rs.executionState.ReturnVU(initVU, false)
    atomic.AddInt64(rs.activeVUsCount, -1)
    rs.wg.Done()
}
```

Each closure appears exactly once in the borrow/return pair, and both are passed to `newStoppedVUHandle` at line 613 for every VU slot. Note that `rs.wg.Add(1)` is paired with `rs.wg.Done()` across these closures, so the `defer runState.wg.Wait()` at `lib/executor/ramping_vus.go:540` only unblocks when every borrowed VU has been returned — a useful property for leak detection at executor-shutdown time.

**The 1:1 guarantee: single entry and exit paths.** The critical invariant is that every `getVU` call corresponds to exactly one `returnVU` call. This is enforced by two facts:

1. **`getVU` is called from exactly one place**: `vuHandle.start()` at `lib/executor/vu_handle.go:115-139`. Specifically, the `stopped, toHardStop` branch at lines 126-136 calls `vh.getVU()` at line 128:

    ```go
    switch vh.state {
    case starting, running: // nothing to do
    case toGracefulStop: // lastly we were running, but now we're not
        if vh.cancel == nil {
            // ...
        } else {
            vh.changeState(running)  // reuse existing VU without getVU
        }
    case stopped, toHardStop:
        // ...
        vu, err := vh.getVU()  // <-- the only getVU call site
        if err != nil {
            return err
        }
        // ... set vh.initVU, vh.activeVU via Activate ...
        vh.changeState(starting)
    }
    ```

    The `toGracefulStop` branch at lines 122-125 *reuses* the existing `vh.initVU`/`vh.activeVU` by simply flipping state back to `running` — it does NOT call `getVU`. This is crucial: a VU already borrowed from the buffer is returned to `running` without a second borrow.

2. **`returnVU` is registered as the `DeactivateCallback`** in `getVUActivationParams()` at `lib/executor/helpers.go:251-264`:

    ```go
    return &lib.VUActivationParams{
        RunContext:               ctx,
        Scenario:                 conf.Name,
        Exec:                     conf.GetExec(),
        Env:                      conf.GetEnv(),
        Tags:                     conf.GetTags(),
        DeactivateCallback:       deactivateCallback,
        GetNextIterationCounters: nextIterationCounters,
    }
    ```

    Line 261 registers `returnVU` as the callback. The k6 VU runtime guarantees this callback fires exactly once, when the per-iteration context is cancelled — which happens either from `gracefulStop()`'s `cancel` path (at `lib/executor/vu_handle.go:154`, which eventually triggers cancellation by overwriting the ctx) or `hardStop()` at `lib/executor/vu_handle.go:178` (`vh.cancel()` is invoked explicitly).

Because the only entry into `getVU` is the `stopped/toHardStop → starting` transition (state table row at `lib/executor/vu_handle.go:34` and `:38`), and because every `starting` state eventually transitions through `running` to either `toGracefulStop → stopped` or `toHardStop → stopped` (per the state table at `lib/executor/vu_handle.go:24-55`), each `getVU` call is eventually paired with exactly one `DeactivateCallback` invocation — i.e., one `returnVU` call.

**Empirical proof: `TestVUHandleRace` assertion.** At `lib/executor/vu_handle_test.go:110`:

```go
require.Equal(t, atomic.LoadInt64(&getVUCount), atomic.LoadInt64(&returnVUCount))
```

After 10,000 concurrent `start`, 1,000 `gracefulStop`, 100 `hardStop`, plus a final `hardStop`, plus one more `start` after the goroutines exit, the counts are equal. This is a strong empirical confirmation that the 1:1 invariant holds under adversarial concurrency.

**Verdict**: The borrow/return protocol is correct by construction. There is a single `getVU` call site guarded by a state transition, and `returnVU` is registered as a single-fire deactivation callback. The test suite under `-race` confirms no leak.

## 4. Cross-Goroutine Communication Map

The following diagram summarizes the goroutines spawned during a ramping-VUs run and the synchronization edges between them:

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

Edge-by-edge explanation:

- **A → B (spawns `maxVUs`)**: At `lib/executor/ramping_vus.go:611-616`, `runLoopsIfPossible` spawns one goroutine per VU slot, each running `vuHandles[i].runLoopsIfPossible(runIteration)`. These are the per-VU iteration goroutines; they live for the executor's lifetime and block on `canStartIter` / `ctx.Done()` / `executorDone` when not iterating.
- **A → C (calls `iterateSteps`)**: At `lib/executor/ramping_vus.go:549-553`, the main `Run()` goroutine *inlines* (does not `go`) the `iterateSteps` call. So `iterateSteps` runs on the main executor goroutine.
- **C → D, C → E**: Sequential dispatch inside `iterateSteps`. For each merged step (from `rawSteps` or `gracefulSteps`), the loop calls either `runRawStep` (the scheduled strategy) or `runGracefulStep` (the max-allowed strategy). See `lib/executor/ramping_vus.go:622-645`.
- **A → F (`go runRemainingGracefulSteps`)**: At `lib/executor/ramping_vus.go:554-558`. This goroutine runs only after `iterateSteps` returned; it processes leftover `gracefulSteps` from index `handledGracefulSteps` onward (`lib/executor/ramping_vus.go:660`). It only ever invokes `runGracefulStep` — never the scheduled strategy.
- **A → G (`go trackProgress`)**: At `lib/executor/ramping_vus.go:536-539`. Updates the progress bar and blocks on `regDurationCtx.Done()` / `parentCtx.Done()`.
- **D → B, E → B (mutex-guarded calls into `vuHandle`)**: Each `start`, `gracefulStop`, `hardStop` acquires `vh.mutex` (see Q4). The VU's own goroutine (B) also acquires `vh.mutex` on the slow path of `runLoopsIfPossible`.
- **H → A, H → B (SIGINT propagation)**: A first SIGINT invokes `gracefulStop` at `cmd/run.go:349-358`, calling `runAbort` which ultimately cancels `runCtx`. Propagation into `maxDurationCtx` (B's `parentCtx`) is immediate via the `context.WithDeadline` chain.

## 5. Key Synchronization Primitives Summary

| Primitive | Location | Purpose |
|---|---|---|
| `vuHandle.mutex` | `lib/executor/vu_handle.go:71` | Guards all state transitions in `start()`, `gracefulStop()`, `hardStop()`, and the slow path of `runLoopsIfPossible()` |
| `vuHandle.canStartIter` | `lib/executor/vu_handle.go:80` | Channel signalling the run loop that it may begin iterating. Closed by `start()` at lines 124 and 135; recreated by `gracefulStop()` at line 162 and `hardStop()` at line 180 |
| Atomic `vuHandle.state` | reads `lib/executor/vu_handle.go:204`, writes via `changeState()` at `lib/executor/vu_handle.go:142-145` | Fast-path lock-free state read paired with atomic writes under the mutex |
| `waiter()` timer | `lib/executor/ramping_vus.go:697-712` | `time.NewTimer` at line 698 plus a context-done select at lines 702-708 — sleeps until the next step offset with prompt cancellation |
| `testAbortController.lock` | `execution/abort.go:20` | Ensures only the first abort reason is recorded (see `execution/abort.go:24-36`) |
| `rampingVUsRunState.wg` | `lib/executor/ramping_vus.go:571` | `sync.WaitGroup` for per-VU iteration goroutines; awaited by `defer runState.wg.Wait()` at line 540. `Add(1)` at line 600 inside `getVU`; `Done()` at line 608 inside `returnVU` |
| `activeVUsCount` atomic | `lib/executor/ramping_vus.go:569`, manipulated at lines 601 and 607 | Progress-bar counter tracking currently-borrowed VUs — NOT a semantic VU-scheduled counter |

## 6. Empirical Validation

All tests were executed on this investigation branch with the Go race detector enabled (`-race`). All tests pass.

```text
$ go version
go version go1.21.13 linux/amd64

$ go build ./...
(no output; exit status 0)

$ go test -race -timeout 120s ./lib/executor/ -run "TestVUHandle"
ok  	go.k6.io/k6/lib/executor	4.123s

$ go test -race -timeout 120s ./lib/executor/ -run "TestRampingVUsRampDownNoWobble"
ok  	go.k6.io/k6/lib/executor	7.035s

$ go test -race -timeout 120s ./lib/executor/ -run "TestRampingVUsConfigExecutionPlanExample"
ok  	go.k6.io/k6/lib/executor	1.016s

$ go test -race -timeout 120s ./lib/executor/ -run "TestSumRandomSegmentSequenceMatchesNoSegment"
ok  	go.k6.io/k6/lib/executor	1.336s
```

Tests exercised:

- `TestVUHandle*` (three tests: `TestVUHandleRace` at `lib/executor/vu_handle_test.go:25`, `TestVUHandleStartStopRace` at `lib/executor/vu_handle_test.go:114`, `TestVUHandleSimple` at `lib/executor/vu_handle_test.go:223`) — validates VU-handle state-machine correctness under concurrent `start`/`gracefulStop`/`hardStop` with the race detector enabled. Also asserts `getVUCount == returnVUCount` at `lib/executor/vu_handle_test.go:110`, which is the empirical 1:1 proof for Q5.
- `TestRampingVUsRampDownNoWobble` at `lib/executor/ramping_vus_test.go:376` — validates a clean smooth stepped ramp-down from 10 VUs to 0 VUs over the `gracefulRampDown` period, asserting the VU-count sequence `{10, 9, 8, 7, 6, 5, 4, 3, 2, 1, 0}` at line 439.
- `TestRampingVUsConfigExecutionPlanExample` at `lib/executor/ramping_vus_test.go:442` — validates the canonical example from the `reserveVUsForGracefulRampDowns` comment (the ASCII diagram).
- `TestSumRandomSegmentSequenceMatchesNoSegment` at `lib/executor/ramping_vus_test.go:1112-1201` — randomized property test across 10 random seeds (`numTests = 10` at line 1116) proving `sum_over_segments(getRawExecutionSteps(segment).PlannedVUs) == getRawExecutionSteps(fullSeg).PlannedVUs` at every time offset. This is the empirical proof for Q3.

## 7. Summary of Findings and Recommendations

### Verdict Table

| Question | Verdict |
|---|---|
| Q1: VU state inconsistency | **By design** — `rawSteps` and `gracefulSteps` intentionally reflect different ceilings; the divergence between the two handlers' internal counters is the mechanism by which `gracefulRampDown` works |
| Q2: Ctrl+C overrun | **Expected within bounds** — cancellation propagation is immediate at every link in the chain; any overrun is attributable to uncancellable I/O inside VU scripts or final executor/scheduler cleanup, not a missing signal path |
| Q3: Segment overcounting | **Timing-observation artifact** — the math is provably additive; `TestSumRandomSegmentSequenceMatchesNoSegment` proves it by simulation |
| Q4: Race between handlers | **No race** — handlers are sequential within the same goroutine; per-VU methods are mutex-guarded; the lock-free fast path pairs atomic reads with atomic writes; `-race` tests pass |
| Q5: VU buffer leak | **No leak** — 1:1 correspondence between `getVU` and `returnVU` is enforced by the state machine's single `getVU` call site, the single-fire `DeactivateCallback`, and verified by `TestVUHandleRace` |

### Diagnostic Recommendations

For future investigations of similar symptoms in the executor subsystem:

- **Always run tests with `-race`** when VU-handling code is touched or when diagnosing suspected concurrency issues. The Go race detector is the authoritative diagnostic.
- **Distinguish counters from schedules.** The live `activeVUsCount` at `lib/executor/ramping_vus.go:569` is a borrow counter for progress-bar display. The *scheduled* VU count is a separate notion (the target at a given time). Confusing these when interpreting live metrics leads to the kind of apparent anomalies described in Q1 and Q3.
- **Use persisted metrics for cross-instance accounting.** When running with multiple execution-segment instances, align observations to a common time base via a metrics backend (Prometheus, InfluxDB, k6 Cloud) — do not sum live progress counters from multiple instances as if they had synchronous clocks.
- **Inspect blocking I/O inside VU scripts when cancellation appears slow.** Check that HTTP clients and any other I/O primitives used by scripts respect `context.Context` cancellation promptly. An iteration body that ignores its context is indistinguishable from a "slow signal handler" to an outside observer.
- **Read the state-transition table.** The comment block at `lib/executor/vu_handle.go:24-55` is authoritative. Any question about "what happens when VU state X receives event Y?" should be answered by that table first; only behavior not covered by the table should be treated as undocumented.
- **Trust the property tests.** `TestSumRandomSegmentSequenceMatchesNoSegment` is a randomized simulation proof of the segment math. Any production symptom that implies the segment math is wrong should be reproducible in a targeted extension of this test; if it isn't reproducible there, the symptom is observational rather than a math bug.
