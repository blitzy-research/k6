# Concurrency Investigation: k6's `ramping-vus` Executor

This document traces, analyzes, and answers five concurrency-related questions about the `ramping-vus` executor in the k6 load-testing tool. Every claim is grounded in specific file/line references from the k6 source tree on branch `k6_ddc3b0b1d23c`. The investigation is **diagnostic and explanatory**; it is not a refactoring proposal and does not analyze unrelated executors.

## 1. Introduction and Question Summary

The five questions:

1. **VU State Inconsistency** — With rapid stages and a long `gracefulRampDown`, the VU count tracked by the "scheduled handler" (`scheduledVUsHandlerStrategy`) diverges from what the "graceful handler" (`maxAllowedVUsHandlerStrategy`) believes should exist. Some VUs appear to enter a limbo state.
2. **Ctrl+C / `gracefulStop` Overrun** — Upon SIGINT, some VUs continue executing longer than the configured `gracefulStop` period should permit.
3. **Execution Segment VU Overcounting** — With three execution segments (`0:1/3`, `1/3:2/3`, `2/3:1`), one instance reports more VUs than the others, and summing all instances exceeds the configured maximum.
4. **Race Condition Hypothesis** — Whether a race exists between the two handler goroutines (`handleNewMaxAllowedVUs` and `handleNewScheduledVUs`) that both manipulate VU handle state.
5. **VU Buffer Leak Hypothesis** — Whether the shared VU channel buffer in `ExecutionState.vus` can leak VUs under concurrent access.

For each question: verdict, code trace, and empirical evidence from the existing test suite under `-race`.

## 2. Architecture Overview of the `ramping-vus` Executor

The executor runs a variable number of VUs over a set of stages, where each stage smoothly ramps toward a target VU count. Three layers:

**Configuration.** `Stage` at `lib/executor/ramping_vus.go:33-37` specifies `Duration` and `Target`. `RampingVUsConfig` at `lib/executor/ramping_vus.go:40-45` embeds `BaseConfig` and adds `StartVUs`, `Stages`, and `GracefulRampDown`. `GracefulStop` is inherited from `BaseConfig` with default 30s (`lib/executor/base_config.go:20`, `:45`).

**Planning.** During `Init()` at `lib/executor/ramping_vus.go:479-487` the executor precomputes two step sequences:

- `rawSteps`, produced by `getRawExecutionSteps()` at `lib/executor/ramping_vus.go:171-234`, encodes the *instantaneous scheduled VU target* at every time offset. It uses a `SegmentedIndex` iterator (`lib/execution_segment.go:762-842`) to produce segment-specific sequences.
- `gracefulSteps`, produced by `GetExecutionRequirements()` at `lib/executor/ramping_vus.go:434-449`, encodes the *maximum allowed VU ceiling* including slots reserved for ramping-down VUs. It calls `getRawExecutionSteps` then passes the result through `reserveVUsForGracefulRampDowns()` at `lib/executor/ramping_vus.go:307-414`.

The invariant, per the algorithm comment at `lib/executor/ramping_vus.go:298-301`, is that `reserveVUsForGracefulRampDowns` "traverses the raw execution steps and whenever there's a scaling down of VUs, it prevents the number of VUs from decreasing for the configured gracefulRampDown period." So `gracefulSteps[i].PlannedVUs >= rawSteps[j].PlannedVUs` by construction.

**Runtime.** `RampingVUs.Run()` at `lib/executor/ramping_vus.go:491-560` drives execution:

1. Creates `(startTime, maxDurationCtx, regDurationCtx)` via `getDurationContexts()` at lines 501-503. Per `lib/executor/helpers.go:168-180`, `maxDurationCtx` has deadline `startTime + regularDuration + gracefulStop` (line 174).
2. Sets up `rampingVUsRunState` at lines 562-574 containing the shared `vuHandles` slice, the `activeVUsCount *int64` atomic (line 569), and the `wg sync.WaitGroup` (line 571).
3. Spawns `trackProgress()` at lines 536-539 and defers `runState.wg.Wait()` at line 540.
4. At line 543, calls `runState.runLoopsIfPossible` (`lib/executor/ramping_vus.go:592-617`), which spawns `maxVUs` goroutines — one per VU slot — each running `vuHandle.runLoopsIfPossible` (line 615).
5. Invokes `iterateSteps()` at lines 549-553 in the main goroutine.
6. Spawns `go runState.runRemainingGracefulSteps(...)` at lines 554-558 for post-ramp cleanup.

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

Any cancellation at any level propagates to all descendants — the foundation of the Q2 answer.

## 3. Detailed Answers to Each Question

### Q1: VU State Inconsistency Between Handlers — Verdict: By Design, Not a Bug

The divergence between `scheduledVUsHandlerStrategy`'s and `maxAllowedVUsHandlerStrategy`'s internal VU counters is correct, expected, and is the *mechanism* by which graceful ramp-down works.

**Two independent closures.** The handler strategies are manufactured by separate factories, each closing over its own `var cur uint64`:

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

The `cur` in `maxAllowedVUsHandlerStrategy` tracks the graceful ceiling (fed by `gracefulSteps`); the `cur` in `scheduledVUsHandlerStrategy` tracks the scheduled target (fed by `rawSteps`). They are separate variables and are *expected* to differ.

**Why they differ.** The two step sequences encode different things. The ASCII diagram at `lib/executor/ramping_vus.go:279-289` visualizes this — stars (`*`) are actively scheduled VUs, dots (`.`) are VUs finishing iterations during `gracefulRampDown`:

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

The dotted tail width equals `gracefulRampDown`. At any time, the graceful ceiling (dots + stars) can be strictly greater than the scheduled target (stars only). That is not a bug; that is the definition of the reservation window.

**The `vuHandle` five-state machine.** Five states are defined at `lib/executor/vu_handle.go:17-22`: `stopped`, `starting`, `running`, `toGracefulStop`, `toHardStop`. The authoritative transition table is the block comment at `lib/executor/vu_handle.go:24-55`. For Q1, the critical row is `grace + running → toGracefulStop` (line 46), executed by `gracefulStop()` at `lib/executor/vu_handle.go:147-163`: the `case running:` branch at lines 157-158 sets state to `toGracefulStop` and returns without invoking `vh.cancel()`. The current iteration continues.

**Tracking the "limbo" state.** When the scheduled handler calls `gracefulStop()` on VU `cur-1`, the scheduled handler decrements its `cur` (line 686) treating the VU as retired, the graceful handler's ceiling is unchanged, and the `vuHandle` is now in `toGracefulStop`. The VU's goroutine in `runLoopsIfPossible()` (`lib/executor/vu_handle.go:185-264`) observes the change on its next loop iteration: the fast path at lines 204-207 fails because `state != running`; the slow path at lines 210-240 cancels the context and transitions to `stopped` once the iteration completes. So a VU in `toGracefulStop` cannot start new iterations but its current iteration is still in flight — it is *not* stuck, it is finishing.

**Empirical evidence.** `TestRampingVUsRampDownNoWobble` at `lib/executor/ramping_vus_test.go:376` proves the ramp-down is deterministic, asserting `vuChanges == {10, 9, 8, 7, 6, 5, 4, 3, 2, 1, 0}` at line 439.

**Verdict**: Not a bug. The divergence is the *mechanism* of `gracefulRampDown`; k6's live `activeVUsCount` reflects the broader "still borrowed from the pool" view (see Q3).

### Q2: Ctrl+C / `gracefulStop` Overrun — Verdict: Cancellation Is Immediate; Any Overrun Has a Specific, Explainable Cause

Signal propagation is **synchronous and direct**. Each link in the cancellation chain is present with no missing hop.

**Step 1 — SIGINT trap.** At `cmd/run.go:347-364`, the signal handler:

```go
gracefulStop := func(sig os.Signal) {
    runAbort(errext.WithAbortReasonIfNone(
        errext.WithExitCodeIfNone(
            fmt.Errorf("test run was aborted because k6 received a '%s' signal", sig), exitcodes.ExternalAbort,
        ), errext.AbortedByUser,
    ))
    lingerCancel()
}
onHardStop := func(sig os.Signal) { globalCancel() }
stopSignalHandling := handleTestAbortSignals(c.gs, gracefulStop, onHardStop)
```

First SIGINT invokes `runAbort` at line 352; second SIGINT invokes `onHardStop` at line 361, which calls `globalCancel()` for immediate exit.

**Step 2 — `AbortTestRun`.** At `execution/abort.go:64-72`:

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

**Step 3 — `testAbortController.abort`** at `execution/abort.go:24-36`:

```go
func (tac *testAbortController) abort(err error) {
    tac.lock.Lock()
    defer tac.lock.Unlock()
    if tac.reason != nil { return }
    tac.reason = err
    tac.cancel()
}
```

The lock at line 25 ensures only the first reason wins; `tac.cancel()` at line 35 invokes the `context.CancelFunc` captured when `NewTestRunContext` was built at `execution/abort.go:49-60` (line 52: `ctx, cancel := context.WithCancel(ctx)`).

**Step 4 — `executorsRunCtx`.** At `execution/scheduler.go:497`:

```go
executorsRunCtx, executorsRunCancel := context.WithCancel(withExecStateCtx)
```

`withExecStateCtx` is derived from signal-aware `runCtx` (line 464). Cancellation propagates to each executor's `Run()` at line 500.

**Step 5 — `maxDurationCtx`.** Inside the executor, `getDurationContexts` at `lib/executor/helpers.go:168-180`:

```go
maxDurationCtx, maxDurationCancel = context.WithDeadline(parentCtx, maxEndTime)
if gracefulStop == 0 {
    return startTime, maxDurationCtx, maxDurationCtx, maxDurationCancel
}
regDurationCtx, _ = context.WithDeadline(maxDurationCtx, startTime.Add(regularDuration)) //nolint:govet
```

`maxDurationCtx` is a child of `parentCtx` (line 174). When `parentCtx` cancels, `maxDurationCtx` cancels immediately — standard Go `context.WithDeadline` semantics.

The `//nolint:govet` at line 178 marks an intentional design choice: `regDurationCtx`'s `cancel` is discarded because its lifetime is bounded by `maxDurationCtx`, which does have its `cancel` invoked. This has no bearing on Q2: the moment `parentCtx` cancels, both children cancel simultaneously.

**Step 6 — `vuHandle` observes cancellation.** `runLoopsIfPossible` at `lib/executor/vu_handle.go:185-264` captures `executorDone = vh.parentCtx.Done()` at line 197. Two `select` blocks check this:

```go
// slow-path guard, lines 211-217
select {
case <-executorDone:
    vh.mutex.Unlock()
    return
default:
}
```

```go
// outer wait select, lines 244-262
select {
case <-canStartIter:
case <-ctx.Done():
case <-executorDone:
    return
}
```

When `executorDone` fires, the loop returns at the next visit. `vh.ctx` is a child of `vh.parentCtx` (line 96), so `vu.RunOnce()` receives cancellation via the `RunContext` registered at `lib/executor/helpers.go:256`.

**Plausible causes of perceived overrun.** Since signal propagation is immediate, any observed overrun has a mundane cause:

- **VU iteration blocked on uncancellable I/O.** Go's `net/http` client respects context cancellation at connect/handshake time but can be slow on in-flight body reads. An iteration awaiting such a read won't observe cancellation until the read returns. The iteration runner is assembled in `getIterationRunner()` at `lib/executor/helpers.go:104-141` and passed to `vuHandle.runLoopsIfPossible` at `lib/executor/ramping_vus.go:615`.
- **`defer runState.wg.Wait()` blocks `Run()`.** At `lib/executor/ramping_vus.go:540`, the executor waits for every per-VU goroutine. `rs.wg.Add(1)` at line 600 and `rs.wg.Done()` at line 608 bracket each borrowed VU; any wedged iteration blocks `Run()` until it returns.
- **Post-executor cleanup.** The scheduler's loop at `execution/scheduler.go:503-514` awaits all executor results; teardown and metrics flush happen afterward. An observer timing SIGINT → process exit sees all of this.
- **Second SIGINT.** For guaranteed termination, the second Ctrl+C triggers `onHardStop` at `cmd/run.go:359-362`, calling `globalCancel()`.

**Verdict**: Cancellation reaches each VU handle through the parent chain with no missing hop. Any ramping-VUs overrun is due to iteration bodies blocking on uncancellable I/O or `defer wg.Wait()` holding `Run()` open until those iterations return — not a delayed signal path.

### Q3: Execution Segment VU Overcounting — Verdict: Mathematically Provably Additive; Apparent Overcount Is a Timing-Observation Artifact

k6's execution-segment math *is* additive: the sum of per-segment VU counts at any logical point equals the unsegmented VU count. The apparent overcount when summing live counters across instances is a live-sampling artifact.

**`ScaleInt64` is additive by construction.** At `lib/execution_segment.go:579-588`:

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

`essw.offsets[segmentIndex][0]` is the segment's initial position within `[0, lcd)`; `[1:]` are stride gaps that cycle back to the start (so strides sum to `lcd`). `len(offsets)` equals the positions the segment owns per LCD cycle.

- **Whole-cycle portion** (line 583): `(value / lcd) * len(offsets)`. Summed across segments, this is `(value / lcd) * sum(len(strides_i))`. Because striped offsets partition `[0, lcd)` exactly once (per `GetStripedOffsets()`), `sum(len(strides_i)) == lcd`, so the whole-cycle total equals `(value / lcd) * lcd`.
- **Residual loop** (lines 584-586): starting at `i = start` and stepping by strides, counts positions strictly less than `value % lcd`. Since each position in `[0, lcd)` belongs to exactly one segment, the total across segments is exactly `value % lcd`.

Therefore `sum_over_segments ScaleInt64(i, value) == (value / lcd) * lcd + (value % lcd) == value`, exactly.

**`SegmentedIndex` iterates correctly.** The struct at `lib/execution_segment.go:768-772` carries `start`, `lcd`, `offsets`, `scaled`, `unscaled`. Methods `Next()` at `:782-791`, `Prev()` at `:795-804`, and `GoTo()` at `:808-842` advance/reverse via the offset list. `getRawExecutionSteps` at `lib/executor/ramping_vus.go:171-234` drives it via `GoTo` (line 180), `Prev` (line 209), `Next` (line 219). Each segment observes step transitions exactly at the unscaled VU counts it owns.

**Randomized property proof.** `TestSumRandomSegmentSequenceMatchesNoSegment` at `lib/executor/ramping_vus_test.go:1112-1201` runs a property test: 10 random trials (const block at `:1115-1116`, where `numTests = 10`), each generating a random stage config and segment sequence (2–15 segments). For each trial it computes `fullRawSteps := c.getRawExecutionSteps(fullSeg, false)` at line 1181, subtracts each segment's steps from the running total at lines 1187-1192, and asserts every remaining step has `PlannedVUs == 0` at lines 1194-1198. This is a direct simulation proof; it passes under `-race` (Section 6).

**Why the observer sees apparent overcount.** The math is correct; overcount arises from live-sampling artifacts:

- **Clock skew.** Each instance samples its own counter at a slightly different wall-clock time. Even sub-second skew matters during rapid ramps.
- **Scheduler jitter.** `waiter()` at `lib/executor/ramping_vus.go:697-712` creates a sentinel `timer := time.NewTimer(time.Hour * 24)` at line 698 and then sets the effective sleep via `timer.Reset(diff)` at line 702 inside a context-cancellation select (lines 703-708). Timer fires are subject to OS scheduling latency; step transitions are not exact.
- **`activeVUsCount` is a borrow counter, not a scheduled counter.** Defined at `lib/executor/ramping_vus.go:569`, incremented in `getVU` at line 601 (`atomic.AddInt64(rs.activeVUsCount, 1)`) and decremented in `returnVU` at line 607. A VU in `toGracefulStop` with an in-flight iteration has NOT yet had `returnVU` called, so it remains counted. Three instances each with a `toGracefulStop` cohort will report a summed `activeVUsCount` exceeding the instantaneous scheduled target — even though the math is additive when counting borrowed VUs.

**Recommendation.** Do not sum live `activeVUsCount` across instances. Either sum persisted VU-start/VU-end events from a metrics backend aligned by a common time base, or reason in terms of the `gracefulSteps` ceiling each graceful handler already uses.

**Verdict**: Math is correct and is proven by randomized property test. The apparent overcount is an observational artifact.

### Q4: Race Condition Between Handler Goroutines — Verdict: No Race; Handlers Are Sequential; Per-VU Methods Are Mutex-Guarded

There is no concurrent mutation of VU-handle state from the two handler strategies.

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

The parameter names `handleNewMaxAllowedVUs` and `handleNewScheduledVUs` are exactly the two handlers named in the hypothesis. `iterateSteps` invokes them one-at-a-time inside its sequential `for` loop. There is no `go` keyword inside the loop — they cannot race inside `iterateSteps` because they are never simultaneous.

**`runRemainingGracefulSteps` runs after `iterateSteps` returns.** At `lib/executor/ramping_vus.go:554-558`, spawned as a goroutine after `iterateSteps` has returned. Its body at `:654-666` only processes `rs.executor.gracefulSteps[handledGracefulSteps:]` — leftover graceful steps from where `iterateSteps` left off. At that point, the scheduled handler is no longer being invoked anywhere. So the graceful handler running in this goroutine still never races the scheduled handler.

**Per-VU methods are mutex-guarded.** The `mutex *sync.Mutex` field at `lib/executor/vu_handle.go:71` is acquired by all mutating methods: `start()` at `:115-139`, `gracefulStop()` at `:147-163`, `hardStop()` at `:165-181`. Concurrent callers from any site are serialized.

**Lock-free fast path is safe.** In `runLoopsIfPossible` at `lib/executor/vu_handle.go:185-264`, the fast path at line 204 reads `state := stateType(atomic.LoadInt32((*int32)(&vh.state)))` without the mutex; the slow path at line 210 takes `vh.mutex.Lock()` for all reads/writes of `vh.ctx`, `vh.cancel`, `vh.canStartIter`, `vh.activeVU`. All writes to `vh.state` go through `changeState()` at `:142-145` via `atomic.StoreInt32` — the atomic load is paired with atomic stores, making the fast path race-free per the Go memory model. A stale `running` read leads to at most one extra iteration before re-reading — exactly what `toGracefulStop` semantics require.

**Empirical proof.** `TestVUHandleRace` at `lib/executor/vu_handle_test.go:25-111` is a race-detector stress test (comment line 24: "this test is mostly interesting when -race is enabled") with three mutation goroutines — 10,000 `start()` (lines 72-78), 1,000 `gracefulStop()` (lines 80-86), 100 `hardStop()` (lines 88-94) — plus a fourth running `runLoopsIfPossible` at line 69. After `wg.Wait()` at line 95 and a final `hardStop()` at line 96, it asserts `getVUCount == returnVUCount` at line 110. The companion `TestVUHandleStartStopRace` at `:114-188` exercises start/stop transition races (helper `handleVUTest` struct at `:190-221` supports both); `TestVUHandleSimple` at `:223` validates basic state-machine semantics. All pass under `-race` (Section 6).

**Verdict**: No race. Handlers are sequentially dispatched in the main `Run()` goroutine; per-VU methods are mutex-guarded; the lock-free fast path pairs atomic reads with atomic writes; the Go race detector confirms zero data races.

### Q5: VU Buffer Leak Hypothesis — Verdict: No Leak; 1:1 Correspondence Enforced by the State Machine

Every `getVU` call is followed by exactly one `returnVU` call, enforced by the VU handle state machine and activation-params callback wiring, and empirically verified by `TestVUHandleRace`.

**Buffer declaration.** At `lib/execution.go:106`: `vus chan InitializedVU`. Per the comment block at `:95-105`, executors are trusted to use `GetPlannedVU`/`GetUnplannedVU`/`ReturnVU` rather than the channel directly. The channel is populated at init via `AddInitializedVU` at `lib/execution.go:537-540`, which calls `es.vus <- vu` then `es.ModInitializedVUsCount(+1)`. The `ModInitializedVUsCount` helper at `lib/execution.go:260-261` wraps the underlying `atomic.AddInt64(es.initializedVUs, mod)`.

**Borrow/return API.** `GetPlannedVU` at `lib/execution.go:471-488` receives from `es.vus` inside a retry loop (returns error after `MaxRetriesGetPlannedVU` timeouts). `ReturnVU` at `:544-549` sends the VU back into the channel and conditionally decrements `ModCurrentlyActiveVUsCount(-1)` when `wasActive`.

**Executor closures.** In `rampingVUsRunState.runLoopsIfPossible` at `lib/executor/ramping_vus.go:592-617`, `getVU` at `:593-604` calls `GetPlannedVU`, then `rs.wg.Add(1)` and `atomic.AddInt64(rs.activeVUsCount, 1)`; `returnVU` at `:605-610` calls `ReturnVU`, then `atomic.AddInt64(rs.activeVUsCount, -1)` and `rs.wg.Done()`. Both are passed to `newStoppedVUHandle` at line 613 for every VU slot. The paired `Add(1)/Done()` means `defer runState.wg.Wait()` at line 540 only unblocks when every borrowed VU has been returned — a strong executor-shutdown leak check.

**The 1:1 guarantee.** Enforced by two facts:

1. **`getVU` is called from exactly one place**: `vuHandle.start()` at `lib/executor/vu_handle.go:115-139`. The `stopped, toHardStop` branch at lines 126-136 calls `vh.getVU()` at line 128. The `toGracefulStop` branch at lines 122-125 *reuses* the existing `vh.initVU`/`vh.activeVU` by flipping state back to `running` — it does NOT call `getVU`. This is crucial: an already-borrowed VU is returned to `running` without a second borrow.

2. **`returnVU` is registered as the `DeactivateCallback`** in `getVUActivationParams()` at `lib/executor/helpers.go:251-264` (line 261: `DeactivateCallback: deactivateCallback`). The k6 VU runtime guarantees this callback fires exactly once, when the per-iteration context is cancelled — via `gracefulStop()` at `lib/executor/vu_handle.go:154` or `hardStop()` at `:178` (`vh.cancel()` invoked explicitly).

Because the only entry into `getVU` is the `stopped/toHardStop → starting` transition (state table at `lib/executor/vu_handle.go:34` and `:38`), and every `starting` eventually transitions through `running` to either `toGracefulStop → stopped` or `toHardStop → stopped` (table at `:24-55`), each `getVU` is eventually paired with exactly one `DeactivateCallback` invocation.

**Empirical proof.** `TestVUHandleRace` asserts `require.Equal(t, atomic.LoadInt64(&getVUCount), atomic.LoadInt64(&returnVUCount))` at `lib/executor/vu_handle_test.go:110`. Under 10,000 concurrent `start`, 1,000 `gracefulStop`, 100 `hardStop` plus a final `hardStop` and one more `start`, counts are equal. Strong confirmation the 1:1 invariant holds under adversarial concurrency.

**Verdict**: Borrow/return protocol is correct by construction. Single `getVU` call site guarded by a state transition; `returnVU` registered as a single-fire deactivation callback; test suite under `-race` confirms no leak.

## 4. Cross-Goroutine Communication Map

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

- **A → B**: `runLoopsIfPossible` at `lib/executor/ramping_vus.go:611-616` spawns one goroutine per VU slot running `vuHandles[i].runLoopsIfPossible(runIteration)`. Each blocks on `canStartIter` / `ctx.Done()` / `executorDone` when not iterating.
- **A → C**: The main `Run()` goroutine *inlines* `iterateSteps` at `lib/executor/ramping_vus.go:549-553` (no `go` keyword).
- **C → D, C → E**: Sequential dispatch inside `iterateSteps` (`:622-645`). Each step invokes either scheduled or max-allowed strategy, never both.
- **A → F**: `go runRemainingGracefulSteps` at `:554-558`. Runs only after `iterateSteps` returned; processes leftover `gracefulSteps` from index `handledGracefulSteps` onward (`:660`). Only invokes `runGracefulStep`.
- **A → G**: `go trackProgress` at `:536-539`. Blocks on `regDurationCtx.Done()` / `parentCtx.Done()`.
- **D → B, E → B**: All three mutators acquire `vh.mutex` (see Q4). The VU's own goroutine also acquires it on the slow path.
- **H → A, H → B**: First SIGINT invokes `gracefulStop` at `cmd/run.go:349-358`, calling `runAbort` which ultimately cancels `runCtx`. Propagation to `maxDurationCtx` (B's `parentCtx`) is immediate via the `context.WithDeadline` chain.

## 5. Key Synchronization Primitives Summary

| Primitive | Location | Purpose |
|---|---|---|
| `vuHandle.mutex` | `lib/executor/vu_handle.go:71` | Guards state transitions in `start()`, `gracefulStop()`, `hardStop()`, and the slow path of `runLoopsIfPossible()` |
| `vuHandle.canStartIter` | `lib/executor/vu_handle.go:80` | Channel signalling the loop may iterate. Closed by `start()` at :124 and :135; recreated by `gracefulStop()` at :162 and `hardStop()` at :180 |
| Atomic `vuHandle.state` | reads `:204`, writes via `changeState()` at `:142-145` | Fast-path lock-free read paired with atomic writes under mutex |
| `waiter()` timer | `lib/executor/ramping_vus.go:697-712` | Sentinel `time.NewTimer(time.Hour * 24)` at :698, then `timer.Reset(diff)` at :702, plus a context-done select at :703-708 — sleeps until the next step with prompt cancellation |
| `testAbortController.lock` | `execution/abort.go:20` | Ensures only the first abort reason is recorded (see `:24-36`) |
| `rampingVUsRunState.wg` | `lib/executor/ramping_vus.go:571` | `sync.WaitGroup` for per-VU goroutines; awaited by `defer runState.wg.Wait()` at :540. `Add(1)` at :600; `Done()` at :608 |
| `activeVUsCount` atomic | `lib/executor/ramping_vus.go:569`, ops at :601 and :607 | Progress-bar counter tracking borrowed VUs — NOT a scheduled counter |

## 6. Empirical Validation

All tests executed on this branch with `-race` enabled. All pass.

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

- `TestVUHandle*` (three tests: `TestVUHandleRace` at `lib/executor/vu_handle_test.go:25`, `TestVUHandleStartStopRace` at `:114`, `TestVUHandleSimple` at `:223`) — validates state-machine correctness under concurrent `start`/`gracefulStop`/`hardStop` with the race detector. Also asserts `getVUCount == returnVUCount` at `:110` — the empirical 1:1 proof for Q5.
- `TestRampingVUsRampDownNoWobble` at `lib/executor/ramping_vus_test.go:376` — smooth stepped ramp-down from 10 VUs to 0, asserting the sequence `{10, 9, 8, 7, 6, 5, 4, 3, 2, 1, 0}` at :439.
- `TestRampingVUsConfigExecutionPlanExample` at `lib/executor/ramping_vus_test.go:442` — validates the canonical example from the `reserveVUsForGracefulRampDowns` comment.
- `TestSumRandomSegmentSequenceMatchesNoSegment` at `lib/executor/ramping_vus_test.go:1112-1201` — randomized property test across 10 random seeds (const block at `:1115-1116`, `numTests = 10`) proving `sum_over_segments(getRawExecutionSteps(segment).PlannedVUs) == getRawExecutionSteps(fullSeg).PlannedVUs` at every time offset. Empirical proof for Q3.

## 7. Summary of Findings and Recommendations

### Verdict Table

| Question | Verdict |
|---|---|
| Q1: VU state inconsistency | **By design** — `rawSteps` and `gracefulSteps` intentionally reflect different ceilings; handler divergence is the mechanism of `gracefulRampDown` |
| Q2: Ctrl+C overrun | **Expected within bounds** — cancellation propagation is immediate at every link; overrun is due to uncancellable I/O or final cleanup, not a missing signal path |
| Q3: Segment overcounting | **Timing-observation artifact** — math is provably additive; `TestSumRandomSegmentSequenceMatchesNoSegment` proves it by simulation |
| Q4: Race between handlers | **No race** — handlers are sequential within one goroutine; per-VU methods are mutex-guarded; the lock-free fast path pairs atomic reads with atomic writes; `-race` tests pass |
| Q5: VU buffer leak | **No leak** — 1:1 correspondence enforced by the single `getVU` call site, the single-fire `DeactivateCallback`, and verified by `TestVUHandleRace` |

### Diagnostic Recommendations

- **Always run tests with `-race`** when touching VU-handling code or diagnosing suspected concurrency issues.
- **Distinguish counters from schedules.** `activeVUsCount` at `lib/executor/ramping_vus.go:569` is a borrow counter for progress display; the *scheduled* count is a separate notion.
- **Use persisted metrics for cross-instance accounting.** Align observations via a metrics backend (Prometheus, InfluxDB, k6 Cloud) — don't sum live progress counters as if clocks were synchronous.
- **Inspect blocking I/O** inside VU scripts when cancellation appears slow. HTTP clients and other I/O primitives must respect `context.Context` promptly.
- **Read the state-transition table** at `lib/executor/vu_handle.go:24-55` first for any "what happens when VU state X receives event Y?" question — it is authoritative.
- **Trust the property tests.** Any production symptom implying the segment math is wrong should reproduce in a targeted extension of `TestSumRandomSegmentSequenceMatchesNoSegment`; if it doesn't, the symptom is observational, not a math bug.
