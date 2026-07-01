# k6 `ramping-vus` Executor — Concurrency Investigation (`k6_ddc3b0b1d23c`)

## Preamble

**What is being answered.** This document answers a reported concurrency question about k6's `ramping-vus` executor, observed during rapid ramp up/down with a long `gracefulRampDown`, under early `ctrl+c` termination, and under distributed execution with execution segments. The question is decomposed into six sub-questions:

- **Q1** — VUs appear "stuck" in a state where they are neither fully active nor fully stopped.
- **Q2** — the VU count tracked by the "scheduled" handler does not match what the "graceful"/max-allowed handler thinks should exist.
- **Q3** — after `ctrl+c`, some VUs reportedly keep running for far longer than `gracefulStop` should allow.
- **Q4** — with execution segments across three instances, one instance consistently shows more VUs than the others at the same timestamp, and summing them appears to exceed the configured maximum.
- **Q5** — is there a race condition between the two handler goroutines, or is the VU buffer leaking?
- **Q6** — a step-by-step trace of what actually happens when both handlers try to modify VU state simultaneously.

**Methodology (run-first).** Following the `SWE-AtlasQnA-Repo` rule, the investigation built k6 from this checkout and ran real scenarios plus the Go race detector *before* writing anything. Every behavioral claim below is placed immediately next to the specific, verbatim observed output line that demonstrates it, and every literal (identifier, string, number, default) is grounded in a `file:line` citation. Where an observation contradicts the user's expectation, it is reported exactly as observed (this happens in **Q3**).

**Read-only scope.** No existing source file was modified. The repository is byte-for-byte unchanged apart from this one document. Temporary investigation scripts were used only for observation and were then removed.

**Bottom-line summary.** The two "handlers" do **not** modify VU state concurrently: `iterateSteps` dispatches both handler closures **serially from a single goroutine**, and only the max-allowed handler continues alone afterward in `runRemainingGracefulSteps`. Under the race detector there is **no data race**, and there is **no VU/goroutine leak** — the `getVU`/`returnVU` closures are paired 1:1. Most reported symptoms are **by-design** grace-period behavior. The one symptom that is **not** reproduced is "VUs keep running longer than `gracefulStop`" after `ctrl+c`: the observed behavior is the opposite — a near-instant (~31–35 ms) stop.

### Build & environment identity

`go version`:

```
go version go1.23.12 linux/amd64
```

Build command (vendored dependencies, CGO enabled so the race detector can compile):

```
GOFLAGS=-mod=vendor CGO_ENABLED=1 go build -o k6 .
```

`k6 version`:

```
k6 v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)
```

The version literal in source is `const Version = "0.55.0"` [`lib/consts/consts.go:12`]. The external-abort exit code observed in Q3 is `ExternalAbort ExitCode = 105` [`errext/exitcodes/codes.go:41`]. Toolchain / dependency context: `go 1.21` [`go.mod:3`], `toolchain go1.21.13` [`go.mod:5`]; test dependencies `github.com/stretchr/testify v1.9.0` [`go.mod:41`] and `go.uber.org/goleak v1.3.0` [`go.mod:49`]; `github.com/sirupsen/logrus v1.9.3` [`go.mod:37`] produces the `--verbose` log output quoted throughout. The race detector requires CGO plus a C compiler; `gcc 13.3.0` was installed for this.

---

## Q1 — The "stuck" VU state (neither fully active nor fully stopped)

**Answer.** The "stuck" state is the **`toGracefulStop`** state. A VU enters it during a graceful ramp-down while it finishes its in-flight iteration: it will not begin a new iteration, but it has not been hard-stopped either — and it can be pulled back to `running` if a ramp-up arrives within the reservation window. **This is by design, not a defect.**

The five VU states are declared `stopped, starting, running, toGracefulStop, toHardStop` [`lib/executor/vu_handle.go:16-22`]; `toGracefulStop` is precisely the "neither fully active nor fully stopped" state.

`gracefulStop()` transitions `running → toGracefulStop` via `vh.changeState(toGracefulStop)` [`lib/executor/vu_handle.go:158`] and logs `"Graceful stop"` [`lib/executor/vu_handle.go:161`] (the method spans [`lib/executor/vu_handle.go:147-161`]).

The window during which such a VU is kept alive is produced by `reserveVUsForGracefulRampDowns` [`lib/executor/ramping_vus.go:307-417`], and the relevant default is `GracefulRampDown: types.NewNullDuration(30*time.Second, false)` [`lib/executor/ramping_vus.go:52`] — i.e. `30s`. The raw active-target steps that drive it come from `getRawExecutionSteps` [`lib/executor/ramping_vus.go:171`].

**Observed evidence.** A temporary `ramping-vus` scenario mirroring the user's config (`startVUs` `0`, `gracefulRampDown` `'30s'`, `stages` `3s→10, 2s→1, 3s→10, 2s→0`, iteration body `sleep(5)`) was run with `--verbose`:

```
./k6 run --verbose oscillate.js
```

Across the whole run the debug log contained **20× `"Start"`**, **19× `"Graceful stop"`**, and **0× `"Hard stop"`** — a non-interrupt ramp-down **never hard-stops**, which is exactly why the count of hard stops is `0`.

The lifecycle of VU 1 shows the lingering `toGracefulStop → running` transition directly (verbatim):

```
time="2026-07-01T19:53:49Z" level=debug msg=Start executor=ramping-vus scenario=oscillate vuNum=1
time="2026-07-01T19:53:53Z" level=debug msg="Graceful stop" executor=ramping-vus scenario=oscillate vuNum=1
time="2026-07-01T19:53:54Z" level=debug msg=Start executor=ramping-vus scenario=oscillate vuNum=1
time="2026-07-01T19:53:58Z" level=debug msg="Graceful stop" executor=ramping-vus scenario=oscillate vuNum=1
```

**Claim ↔ evidence.** VU 1 receives `"Graceful stop"` at `:53` (entering `toGracefulStop`) and is **re-`Start`ed at `:54`** — within the `30s` reserve. That is exactly the documented transition `| start | toGracefulStop | running | we raced with the loop stopping, just continue |` [`lib/executor/vu_handle.go:37`]. The interval between the graceful-stop and the next start is precisely the "neither fully active nor fully stopped" window the user observed. This is expected behavior of the graceful ramp-down + reservation design — not a stuck or leaked VU.

---

## Q2 — Handler-count mismatch (scheduled handler vs graceful/max-allowed handler)

**Answer.** The two handlers **track different quantities by design**, so their counts are *supposed* to differ during a ramp-down:

- `scheduledVUsHandlerStrategy` tracks **active-target** VUs (surfaced as the `vus` metric); it calls `start()` to raise the active count and `gracefulStop()` to lower it — cite [`lib/executor/ramping_vus.go:678-689`].
- `maxAllowedVUsHandlerStrategy` tracks **reserved / max-allowed** VUs (surfaced as the `vus_max` metric); it calls `hardStop()` only when the reserve itself shrinks — cite [`lib/executor/ramping_vus.go:668-676`].

During a ramp-down with a long `gracefulRampDown`, `vus` drops while `vus_max` stays high; the gap equals the number of VUs still finishing iterations inside the graceful window. This is **not** a defect — the two counters measure two different things.

**Observed evidence.** The end-of-test summary of the same run as Q1 (verbatim):

```
     vus..................: 1   min=1     max=10
     vus_max..............: 10  min=10    max=10
```

**Claim ↔ evidence.** `vus` ranges `min=1 max=10` — the scheduled/active-target count driven by `scheduledVUsHandlerStrategy` [`lib/executor/ramping_vus.go:678-689`]. `vus_max` is steady `10` (`min=10 max=10`) — the reserved count driven by `maxAllowedVUsHandlerStrategy` [`lib/executor/ramping_vus.go:668-676`]. The gap (up to 9) is exactly the `gracefulRampDown` reserve, i.e. the VUs allowed to finish their current iteration. The counts "not matching" is the intended relationship `vus ≤ vus_max`, not a bug.

---

## Q3 — Interrupt (`ctrl+c`) vs `gracefulStop`

**Answer — this CONTRADICTS the user's expectation; it is reported exactly anyway.** A manual interrupt does **NOT** let VUs run longer than `gracefulStop`. It **bypasses `gracefulStop`** and stops iterations **near-instantly**. The user's reported symptom — "some VUs keep running for way longer than `gracefulStop` should allow" — is **not reproduced** in the v0.55.0 baseline; the observed behavior is the **opposite**, a ~31–35 ms stop. Per the rule to report exactly what is observed even when it contradicts expectations, this is stated plainly here.

The documented rule is the docstring on `GetGracefulStop`: "Of course, that doesn't count when the user manually interrupts the test, then iterations are immediately stopped." — cite [`lib/executor/base_config.go:95-96`]. The default it is compared against is `DefaultGracefulStopValue = 30 * time.Second` [`lib/executor/base_config.go:20`].

The interrupt path is a two-stage abort. The **first** signal runs the `gracefulStop` func, which calls `runAbort` with `fmt.Errorf("test run was aborted because k6 received a '%s' signal", sig)` and `exitcodes.ExternalAbort` [`cmd/run.go:354`] plus `errext.AbortedByUser` [`cmd/run.go:355`], then `lingerCancel()` [`cmd/run.go:357`] (the func spans [`cmd/run.go:349-358`]). A **second** signal runs `onHardStop`, which calls `globalCancel()` [`cmd/run.go:361`] (the func spans [`cmd/run.go:359-362`]). The trapped signal set is `os.Interrupt, syscall.SIGINT, syscall.SIGTERM` [`cmd/common.go:101`].

Iteration cancellation is checked **only after `RunOnce` returns**, via a `select` on `case <-ctx.Done():` that returns `false` in `getIterationRunner` [`lib/executor/helpers.go:114-120`] (function defined at [`lib/executor/helpers.go:104`]). The max-duration deadline `maxEndTime := startTime.Add(regularDuration + gracefulStop)` is set in `getDurationContexts` [`lib/executor/helpers.go:172`] (function defined at [`lib/executor/helpers.go:168`]) — this governs the *natural* end plus `gracefulStop`; a manual interrupt cancels the parent context immediately, so this deadline is not what bounds a `ctrl+c`.

**Observed evidence — Run A** (interruptible `sleep` iteration, `constant-vus`, `gracefulStop` `'60s'`): `SIGINT` was sent at `19:56:06.252` and the process exited at `19:56:06.283` → **latency 0.031 s** versus the configured **60 s**, with `K6_EXIT_CODE=105`.

Last VU console line before the interrupt (verbatim):

```
level=info msg="VU 1 tick 4 @ 2026-07-01T19:56:05.279Z" source=console
```

`tick 5` never printed — the VU was interrupted mid-`sleep`.

Abort line (verbatim):

```
time="2026-07-01T19:56:06Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

**Observed evidence — Run B** (NON-cancelable 12 s busy-loop iteration, `gracefulStop` `'1s'`): `SIGINT` was sent at `19:57:08.635` and the process exited at `19:57:08.672` → **latency 0.035 s**, with `EXIT_CODE=105`. The `busy-iteration DONE` line **never printed** (k6 exited during the busy loop); the abort message was identical to Run A.

**Claim ↔ evidence.**

- "Interrupt bypasses `gracefulStop`" ↔ Run A's **0.031 s** stop versus the configured **60 s**, and `K6_EXIT_CODE=105` (matching `ExternalAbort ExitCode = 105` [`errext/exitcodes/codes.go:41`]).
- "Even a non-cancelable iteration does not delay process exit" ↔ Run B's **0.035 s** stop and the missing `busy-iteration DONE` line.
- **Conclusion:** a **~31–35 ms** stop regardless of iteration cancelability; `gracefulStop` (default `30s` [`lib/executor/base_config.go:20`]) is **not applied** to a manual interrupt — matching the docstring [`lib/executor/base_config.go:95-96`], the two-stage abort [`cmd/run.go:349-364`], and the observed exit code `105`.
- **Rationale (the "why"):** the process aborts on the first signal (propagating the error and cancelling contexts); a VU only lingers until its current `RunOnce` returns and `getIterationRunner` observes `ctx.Done()` [`lib/executor/helpers.go:114-120`], so the effective bound is the iteration's own cancellation responsiveness, not `gracefulStop`.

---

## Q4 — Execution-segment imbalance and sum-over-max

**Answer (two parts).**

- **(a) The per-segment skew is REAL and deterministic.** When the total is not divisible by 3, the **first** segment `0:1/3` receives the round-up extra VU — this is the "one instance consistently shows more VUs than the others".
- **(b) A true simultaneous sum-over-max is NOT reproduced.** Striping guarantees the per-segment maxes sum **exactly** to the global max. The perceived overflow is the ±1 skew read at **non-identical sampling instants** (different instances sampled at slightly different times).

Per-segment striping is seeded by `index = lib.NewSegmentedIndex(et)` [`lib/executor/ramping_vus.go:176`]. Scaling rounds up: `Scale` ends `return roundUp(toValue).Int64()` [`lib/execution_segment.go:253-273`] (helper `roundUp` at [`lib/execution_segment.go:242`]). The striping primitive constructor is `NewSegmentedIndex` [`lib/execution_segment.go:776`]. Each instance builds its tuple with `lib.NewExecutionTuple(options.ExecutionSegment, options.ExecutionSegmentSequence)` [`execution/scheduler.go:40`]; an executor with no work on a segment is disabled with `"Executor '%s' is disabled for segment %s due to lack of work!"` [`execution/scheduler.go:57`] (guarded at [`execution/scheduler.go:54-58`]).

**Observed evidence.** The same script was run three times, once per simulated instance on a single host, with the flags:

```
--execution-segment '0:1/3' | '1/3:2/3' | '2/3:1'   with   --execution-segment-sequence '0,1/3,2/3,1'
```

Ramping to a peak of 10, the per-segment `vus_max` values were **4, 3, 3** for segments `0:1/3`, `1/3:2/3`, `2/3:1` respectively, while the **global `vus_max` = 10** (`4+3+3 = 10` — the first segment gets the round-up extra).

Aligning the per-second `vus` time-series across the three instances, the **maximum simultaneous sum across segments = 9** (≤ 10); e.g. at `20:00:01` the three instances read `[4,3,2] = 9`. There is **no true overflow**.

A `constant-vus` sweep of the sum of per-segment `vus_max` against the global `N` (verbatim):

```
N=1→[1,0,0]=1; N=2→[1,1,0]=2; N=4→[2,1,1]=4; N=5→[2,2,1]=5; N=7→[3,2,2]=7; N=8→[3,3,2]=8; N=10→[4,3,3]=10; N=11→[4,4,3]=11
```

Every sum **exactly equals `N`** — the per-segment maxes always sum to the global maximum.

The invariant is corroborated by the test `TestSumRandomSegmentSequenceMatchesNoSegment` [`lib/executor/ramping_vus_test.go:1112`], which **passes** under the race detector (all 10 random subtests PASS).

**Claim ↔ evidence.**

- "The first segment gets the extra VU (deterministic skew)" ↔ the per-segment `vus_max = 4, 3, 3` and the sweep line (`N=10→[4,3,3]=10`, `N=1→[1,0,0]=1`, …).
- "No true instantaneous sum-over-max" ↔ the aligned time-series max sum `= 9 (≤ 10)` at `20:00:01 → [4,3,2]=9`, the sweep where every sum equals `N`, and the `TestSumRandomSegmentSequenceMatchesNoSegment` PASS.
- **Rationale (the "why"):** `Scale` rounds up [`lib/execution_segment.go:253-273`] and striping via `NewSegmentedIndex` [`lib/executor/ramping_vus.go:176`, `lib/execution_segment.go:776`] deterministically assigns the extra VU to the earliest segment; because the segments partition `(0,1]` exactly, the rounded per-segment counts still sum to the global `N`. A naive observer summing instances sampled at different instants sees a transient ±1 that *looks* like "over max" but is not simultaneous.

---

## Q5 — Race between the two handler goroutines? Or a VU-buffer leak?

**Answer.** **NO data race and NO VU/goroutine leak.** The two handlers are **not concurrent on shared state** (proven in Q6). The "VU buffer" is the `ExecutionState` pool; the `getVU`/`returnVU` closures are paired **1:1**, so it does not leak.

The VU buffer is `ExecutionState`, managed via `GetPlannedVU` [`lib/execution.go:471`] and `ReturnVU` [`lib/execution.go:544`]; active-count bookkeeping is `ModCurrentlyActiveVUsCount` [`lib/execution.go:276`] and `AddInitializedVU` [`lib/execution.go:537`]. The 1:1 pairing lives in the `getVU`/`returnVU` closures [`lib/executor/ramping_vus.go:595-606`] — `getVU` calls `ModCurrentlyActiveVUsCount(+1)` and `returnVU` calls `executionState.ReturnVU(...)`. The documented invariant is "for each call to getVU there must be 1 (and only 1) call to returnVU" [`lib/executor/vu_handle.go:57-69`] (text at L62).

**Accuracy nuance (reported exactly — not overclaimed).** `go.uber.org/goleak` is used **ONLY** in `cmd/tests/tests.go` — `goleak.Find()` [`cmd/tests/tests.go:57`], imported at [`cmd/tests/tests.go:10`]; it is **NOT imported anywhere under `lib/`**. Therefore `go test -race ./lib/executor/` does **not** run the goleak harness. The executor tests instead verify VU accounting via `GetCurrentlyActiveVUsCount()` [`lib/executor/ramping_vus_test.go:139,416,422`]. This document does **not** claim goleak runs inside the executor package.

**Observed evidence.** The race detector (CGO / `gcc`) was run:

```
go test -race -count=1
```

Targeted run of the two concurrency tests (verbatim):

```
--- PASS: TestVUHandleRace (0.19s)
--- PASS: TestVUHandleStartStopRace (0.42s)
ok  	go.k6.io/k6/lib/executor	1.447s
```

Full package under `-race` (verbatim):

```
ok  	go.k6.io/k6/lib/executor	29.951s
```

There was **NO `WARNING: DATA RACE`** and no failures.

**Claim ↔ evidence.**

- "No data race" ↔ the `--- PASS` lines and the absence of any `WARNING: DATA RACE`, from `TestVUHandleRace` [`lib/executor/vu_handle_test.go:25`] and `TestVUHandleStartStopRace` [`lib/executor/vu_handle_test.go:114`].
- "No VU-buffer leak" ↔ the 1:1 `getVU`/`returnVU` design [`lib/executor/ramping_vus.go:595-606`], the invariant comment [`lib/executor/vu_handle.go:57-69`], and the clean full-package run `ok ... 29.951s`.
- **Rationale (the "why"):** because the buffer is the `ExecutionState` pool and every checkout (`GetPlannedVU` / `ModCurrentlyActiveVUsCount(+1)`) is matched by a return (`ReturnVU`), the pool cannot leak; and because the handlers do not share mutable iteration state (Q6), there is nothing for `-race` to flag.

---

## Q6 — Trace: what happens when both handlers try to modify VU state "simultaneously"

**Central finding.** They do **NOT** modify shared iteration state simultaneously. `iterateSteps` dispatches **both** handler closures **serially from a single goroutine**, ordered by `TimeOffset`, prioritizing raw steps [`lib/executor/ramping_vus.go:620-642`]. This call runs **synchronously** in `Run` (no `go` keyword) [`lib/executor/ramping_vus.go:549-553`]. Only **after** `iterateSteps` returns (raw steps exhausted) does `Run` launch `go runState.runRemainingGracefulSteps(...)` [`lib/executor/ramping_vus.go:554-558`], and that lone goroutine calls **only** `handleNewMaxAllowedVUs` [`lib/executor/ramping_vus.go:654-666`]. Hence the `scheduledVUsHandlerStrategy` and `maxAllowedVUsHandlerStrategy` closures **never race for the shared iteration cursor**.

The only real concurrency is between the handler goroutine(s) and each VU's own `runLoopsIfPossible` loop [`lib/executor/vu_handle.go:185-263`] (the fast path reads state with `atomic.LoadInt32` at [`lib/executor/vu_handle.go:204`]). Every state mutation is guarded by `vh.mutex` and published via `atomic.StoreInt32` — the `changeState` helper does `atomic.StoreInt32((*int32)(&vh.state), int32(newState))` [`lib/executor/vu_handle.go:142-145`], and `start`/`gracefulStop`/`hardStop` all take the mutex [`lib/executor/vu_handle.go:115-181`]. The documented transition table [`lib/executor/vu_handle.go:24-55`] enumerates each raced (input × state) case and its **intentional** resolution.

**Corroboration.** The `-race` PASS output from Q5 (no `WARNING: DATA RACE`) confirms the per-VU guards hold under concurrency.

**Step-by-step trace** (tied to code and to the Q1 VU-1 log lines):

1. A ramp-down calls `gracefulStop()` on the highest-indexed active VU handle → `running → toGracefulStop` [`lib/executor/vu_handle.go:147-161`], logging `"Graceful stop"` — this is the `:53` line for VU 1 in Q1.
2. The VU's `runLoopsIfPossible` loop observes `toGracefulStop` on its slow path, cancels its context, and transitions to `stopped` [`lib/executor/vu_handle.go:185-263`; table row `| loop | toGracefulStop | stopped |` at `lib/executor/vu_handle.go:24-55`].
3. An arriving ramp-up calls `start()`; finding the handle in `toGracefulStop`, it transitions **back to `running`** — "we raced with the loop stopping, just continue" [`lib/executor/vu_handle.go:37`] (the `start` method begins at [`lib/executor/vu_handle.go:115`]), logging `"Start"` — this is the `:54` line for VU 1 in Q1.
4. A `hardStop()` — issued **only** by `maxAllowedVUsHandlerStrategy` once the reserve itself shrinks [`lib/executor/ramping_vus.go:668-676`] — forces `toHardStop` and cancels the context [`lib/executor/vu_handle.go:165-181`]. In the Q1 non-interrupt run this path is not exercised (0× `"Hard stop"`), consistent with "a graceful ramp-down never hard-stops".

**Claim ↔ evidence.**

- "Handlers are serialized, not concurrent" ↔ `iterateSteps` is called without `go` [`lib/executor/ramping_vus.go:549-553`] and dispatches both closures in one loop [`lib/executor/ramping_vus.go:620-642`], while `runRemainingGracefulSteps` is the lone `go` routine receiving only the max-allowed handler [`lib/executor/ramping_vus.go:554-558,654-666`].
- "Per-VU state changes are safe under concurrency" ↔ the mutex + atomic guards [`lib/executor/vu_handle.go:115-181`] and the `-race` PASS from Q5.
- The concrete lifecycle steps map to the VU-1 `"Start"` / `"Graceful stop"` lines already quoted in Q1.

Named symbols exercised in this section: `scheduledVUsHandlerStrategy`, `maxAllowedVUsHandlerStrategy`, `iterateSteps`, `runRemainingGracefulSteps`, `runLoopsIfPossible`, `start`, `gracefulStop`, `hardStop`.

---

## Coverage pass

| # | Item to cover | Where answered | Grounding |
|---|---------------|----------------|-----------|
| 1 | Symptom: "stuck" VU | Q1 | `toGracefulStop` [`lib/executor/vu_handle.go:16-22`] + VU-1 `:53`/`:54` log lines |
| 2 | Symptom: handler-count mismatch | Q2 | `vus` vs `vus_max` block; strategies [`ramping_vus.go:678-689`, `:668-676`] |
| 3 | Symptom: interrupt outruns `gracefulStop` | Q3 | **contradicted**; Run A `0.031 s` vs `60 s`, Run B `0.035 s` |
| 4 | Symptom: segment skew | Q4 | `vus_max = 4, 3, 3`; sweep `N=10→[4,3,3]=10` |
| 5 | Symptom: sum-over-max | Q4 | aligned max sum `= 9 (≤10)`; every sweep sum `= N` |
| 6 | Symptom: race vs leak | Q5 | `--- PASS` + no `WARNING: DATA RACE`; 1:1 `getVU`/`returnVU` |
| 7 | Deliverable: simultaneous-modification trace | Q6 | serial dispatch [`ramping_vus.go:620-642`, `:549-558`] + mutex/atomic [`vu_handle.go:115-181`] |
| 8 | Functions: `scheduledVUsHandlerStrategy`, `maxAllowedVUsHandlerStrategy`, `iterateSteps`, `runRemainingGracefulSteps`, `reserveVUsForGracefulRampDowns`, `getRawExecutionSteps`, `NewSegmentedIndex`, `Scale`, `GetPlannedVU`, `ReturnVU`, `ModCurrentlyActiveVUsCount`, `runLoopsIfPossible`, `start`/`gracefulStop`/`hardStop`, `getIterationRunner`, `getDurationContexts`, `GetGracefulStop` | Q1–Q6 | each cited at `file:line` in-section |
| 9 | States: `stopped`, `starting`, `running`, `toGracefulStop`, `toHardStop` | Q1, Q6 | [`lib/executor/vu_handle.go:16-22`] |
| 10 | Flags/config: `--verbose`, `--execution-segment`, `--execution-segment-sequence`, `gracefulStop`, `gracefulRampDown`, `startVUs`, `stages` | Q1, Q3, Q4 | in-section |
| 11 | Signals: `os.Interrupt`, `syscall.SIGINT`, `syscall.SIGTERM` | Q3 | [`cmd/common.go:101`] |
| 12 | Literals: `30s` defaults (`GracefulRampDown`, `gracefulStop`), exit code `105`, version `v0.55.0` | Q1, Q3, preamble | [`ramping_vus.go:52`], [`base_config.go:20`], [`codes.go:41`], [`consts.go:12`] |
| 13 | Accuracy nuance: goleak only in `cmd/tests/tests.go`, not under `lib/` | Q5 | [`cmd/tests/tests.go:10,57`] |
| 14 | Nuance: `maxEndTime` at `helpers.go:172` (def at `:168`) | Q3 | [`lib/executor/helpers.go:172`] |
| 15 | Read-only scope affirmed | Preamble | repository unchanged apart from this file; temp scripts removed |

**Read-only scope note (restated).** No existing source file in the repository was modified, and no code was added other than this answer document. Temporary observation scripts were used and then removed, leaving the repository byte-for-byte unchanged apart from this single Markdown file at `blitzy/documentation/k6_ddc3b0b1d23c.md`.
