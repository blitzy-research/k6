# Investigating "stuck VUs", handler count mismatches, Ctrl+C overrun, and segment distribution in k6's `ramping-vus` executor

**Repository:** `grafana/k6` (module `go.k6.io/k6`) &nbsp;·&nbsp; **Commit:** `ddc3b0b1d23c128e34e2792fc9075f9126e32375` (`ddc3b0b1d23c`) &nbsp;·&nbsp; **Built binary:** `k6 v0.55.0`

---

## TL;DR (bottom line up front)

At commit `ddc3b0b1d23c`, an empirical investigation — building the binary, running the executor, exercising the segment math, and running the Go race detector — found:

- **No data race between the two handler goroutines.** They cannot run concurrently on shared state: the scheduled handler runs only inside a **synchronous** `iterateSteps(...)` call (`lib/executor/ramping_vus.go:549`) that fully returns *before* the graceful goroutine is spawned with `go runState.runRemainingGracefulSteps(...)` (`lib/executor/ramping_vus.go:554`). The Go race detector reports **`DATA RACE occurrences: 0`**.
- **No VU-buffer leak.** VU checkout/return is balanced (`wg.Add(1)` at `ramping_vus.go:600`, `wg.Done()` at `ramping_vus.go:608`) and `Run()` blocks on `defer runState.wg.Wait()` (`ramping_vus.go:540`) before returning. The error string the user calls the "VU buffer" problem — `"Cannot get a VU from the buffer"` (`ramping_vus.go:596`) — is a shutdown-time guard, not a leak.
- The reported symptoms are **intended-by-design behavior**: transitional graceful-draining states, three deliberately-distinct VU counters, a cooperative Ctrl+C **abort** (not the `gracefulStop` timer), and deterministic execution-segment striping.
- The **one factually incorrect belief** — that per-instance segment counts sum to *more* than the configured maximum — is corrected with observed output: the per-instance counts are **asymmetric** (one instance gets one extra VU) but the sum **always equals** the maximum and never exceeds it.

Each of the five sub-parts of the question is answered explicitly below, every claim is tied to a `file:line` reference or verbatim runtime output, and a closing coverage pass plus an honesty/caveats section are included.

---

## How this was verified

All findings were produced by **building and running** the code at this commit, not by reading alone.

- **Repo / commit under investigation:** the source under investigation is commit `ddc3b0b1d23c128e34e2792fc9075f9126e32375` (`ddc3b0b1d23c`). The delivered branch adds only this answer document on top of that commit — `git diff --name-status ddc3b0b1d23c HEAD` reports exactly `A  blitzy/documentation/k6_ddc3b0b1d23c.md` (no source file differs), so every source `file:line` cited below is valid and byte-identical at both revisions. The module is `go.k6.io/k6` (`go.mod:1`), which declares `go 1.21` (`go.mod:3`) and `toolchain go1.21.13` (`go.mod:5`).
- **Toolchain used:** `go1.23.12` (installed at `/usr/local/go`), `gcc 15.2.0` (required by the CGO-based `-race` detector), dependencies consumed from the vendored tree (`GOFLAGS=-mod=vendor`, fully offline; `GOPROXY=off`).
- **Build command** (binary intentionally built *outside* the repository so the working tree stays clean):

  ```text
  export PATH=$PATH:/usr/local/go/bin GOFLAGS=-mod=vendor
  go build -o /tmp/k6bin .
  ```

- **Observed version string.** k6 stamps the commit into its version string from Go's build metadata: `FullVersion()` reads the `vcs.revision` build setting and keeps its first 10 characters (`commitLen := 10` at `lib/consts/consts.go:31`; `commit = s.Value[:commitLen]` at `consts.go:35`), then formats `"%s (commit/%s, %s)"` (`consts.go:52`) with `Version = "0.55.0"` (`consts.go:12`). Building the **commit under investigation** (`ddc3b0b1d23c…`, whose embedded `vcs.revision` is that full hash) therefore produces, verbatim from `/tmp/k6bin version`:

  ```text
  k6bin v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)
  ```

  Here `ddc3b0b1d2` is exactly the first 10 characters of the target commit `ddc3b0b1d23c…`. The `commit/…` field simply reflects whichever commit is built: this branch adds only the answer document on top of the source commit (no source file changes), so building a later commit on the branch reports that commit's 10-character prefix instead — for example the doc-only commit `46e149114…` yields `commit/46e149114e` — while `v0.55.0`, `go1.23.12`, and `linux/amd64` stay unchanged and the investigated source is byte-identical (`git diff --name-status ddc3b0b1d23c HEAD` shows only the added document). The basename reads `k6bin` (not `k6`) purely because k6 derives its self-name from `os.Args[0]` and the binary was built to the path `/tmp/k6bin`; this is a cosmetic artifact, not a version discrepancy.

Every command in this document is re-runnable; the exact command precedes each output block so a reviewer can reproduce it.

---

## Terminology map (the user's words → the code)

| The user says… | In the code it is… | Location |
|---|---|---|
| "scheduled handler" | `handleNewScheduledVUs`, built by `scheduledVUsHandlerStrategy()` | `lib/executor/ramping_vus.go:679` |
| "graceful handler" | `handleNewMaxAllowedVUs`, built by `maxAllowedVUsHandlerStrategy()` | `lib/executor/ramping_vus.go:668` |
| "two handler goroutines" | the **synchronous** `iterateSteps(...)` loop, then the later-spawned `go runState.runRemainingGracefulSteps(...)` | `ramping_vus.go:549` and `ramping_vus.go:554` |
| "VU buffer" | the pool of `vuHandles`; the error `"Cannot get a VU from the buffer"` | `ramping_vus.go:596` |
| "execution segments across 3 instances" | an `ExecutionSegmentSequence` such as `0,1/3,2/3,1`, scaled per segment via `ExecutionTuple.ScaleInt64` | `lib/execution_segment.go:734` |

A key nuance up front: the user speaks of "two handler **goroutines**," but at this commit the scheduled handler never runs in its own goroutine. It runs synchronously in `Run()`; only the graceful handler continues in a spawned goroutine afterwards. That single fact is what makes a race between the two impossible — see sub-part #5.

---

## Sub-part #1 — "Stuck" VUs are transitional graceful-draining states (by design)

**Question:** *"I'm seeing stuck VUs … When stages ramp up and down quickly and `gracefulRampDown` is long …"*

**Answer:** The VUs are **not stuck and not leaked**. `toGracefulStop` and `toHardStop` are *transitional draining states*: a VU in `toGracefulStop` finishes its in-flight iteration (no new iteration is started) and then transitions to `stopped`. When `gracefulRampDown` is long, VUs legitimately remain in a draining/reserved state for up to that window **by design** — which is exactly what looks like "stuck."

### The state machine

The VU lifecycle is a five-state machine declared in `lib/executor/vu_handle.go`. The type is `type stateType int32` (`vu_handle.go:13`), and the states are (verbatim, `vu_handle.go:15-22`):

```go
// states
const (
	stopped stateType = iota
	starting
	running
	toGracefulStop
	toHardStop
)
```

The file documents every legal transition in a state-transition table written as a comment (`vu_handle.go:24-55`). Three rows prove the draining states are intentional (verbatim):

```text
| start | stopped         | starting          | normal                                            |
| loop  | toGracefulStop  | stopped           | cancel the context and make new one               |
| grace | running         | toGracefulStop    | normal one, the actual work is in the loop        |
```

Reading the relevant rows: the `gracefulStop()` method moves a `running` VU to `toGracefulStop` (`vu_handle.go:157-158`), and the VU's own run loop later moves `toGracefulStop → stopped` after the in-flight iteration ends. So a VU sitting in `toGracefulStop` is *draining*, not wedged.

The methods that drive these transitions are all present and mutex-guarded:

- `start()` — `vu_handle.go:115`
- `changeState(...)` — `vu_handle.go:142`, which persists state atomically via `atomic.StoreInt32((*int32)(&vh.state), int32(newState))` (`vu_handle.go:144`)
- `gracefulStop()` — `vu_handle.go:147` (takes `vh.mutex.Lock()`, then `running → toGracefulStop`)
- `hardStop()` — `vu_handle.go:165` (takes `vh.mutex.Lock()`, then `running`/`toGracefulStop → toHardStop`)
- `runLoopsIfPossible(...)` — `vu_handle.go:185`

### Why a long `gracefulRampDown` keeps VUs alive on purpose

During a downward slope, the executor deliberately keeps the *max-allowed ceiling* high for the `gracefulRampDown` window so a ramping-down VU keeps its slot long enough to finish its current iteration. That is implemented by `reserveVUsForGracefulRampDowns(...)` (`lib/executor/ramping_vus.go:307`), whose doc comment states the intent verbatim (`ramping_vus.go:299-301`):

```text
// their last iterations is pretty simple. It just traverses the raw execution
// steps and whenever there's a scaling down of VUs, it prevents the number of
// VUs from decreasing for the configured gracefulRampDown period.
```

This reservation is applied only when a graceful ramp-down is configured — the guard is `if vlvc.GracefulRampDown.Duration > 0 {` (`ramping_vus.go:439`) calling `steps = vlvc.reserveVUsForGracefulRampDowns(steps, executorEndOffset)` (`ramping_vus.go:440`).

The `gracefulRampDown` option itself is defined on the config as `GracefulRampDown types.NullDuration` (`ramping_vus.go:44`), defaults to `types.NewNullDuration(30*time.Second, false)` (`ramping_vus.go:52`), and is read through `GetGracefulRampDown()` (`ramping_vus.go:66`).

**Rationale / verdict:** a VU that lingers in `toGracefulStop` for up to a long configured `gracefulRampDown` is the feature working as intended — the whole point of `gracefulRampDown` is to let in-flight iterations finish while VUs ramp down. This is **intended-by-design behavior**, not a leak or a wedged VU.

---

## Sub-part #2 — Three distinct VU counts exist by design

**Question:** *"…the VU count the scheduled handler tracks doesn't match what the graceful handler thinks should exist."*

**Answer:** They are **supposed to differ** during ramp-downs. There are **three different numbers on purpose**, tracked in three different places:

**1. Scheduled / raw-planned count** — a closure-local `var cur uint64` inside `scheduledVUsHandlerStrategy()` (`ramping_vus.go:679`). It tracks the *raw planned* VUs: it goes **up** via `start()` and **down** via `gracefulStop()`. Verbatim (`ramping_vus.go:679-690`):

```go
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

So this counter's downward movement calls `gracefulStop()` (`ramping_vus.go:687`) — i.e. it *schedules* graceful draining, it does not hard-kill.

**2. Max-allowed / ceiling count** — a *separate* closure-local `var cur uint64` inside `maxAllowedVUsHandlerStrategy()` (`ramping_vus.go:668`). It only ever calls `hardStop()`, bringing VUs down to the *graceful* planned ceiling — a ceiling that **intentionally stays higher during ramp-downs**. Verbatim (`ramping_vus.go:668-677`):

```go
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
```

**3. Progress-only count** — the atomic field `activeVUsCount *int64` (`ramping_vus.go:569`), whose comment states its purpose verbatim: `// the current number of active VUs, used only for the progress display`. It is read by the progress function via `cur := atomic.LoadInt64(rs.activeVUsCount)` (`ramping_vus.go:582`), and moved by `atomic.AddInt64(rs.activeVUsCount, 1)` on checkout (`ramping_vus.go:601`) and `atomic.AddInt64(rs.activeVUsCount, -1)` on return (`ramping_vus.go:607`).

**Rationale / verdict:** the "mismatch" the user observes between the scheduled handler and the graceful handler is the **designed gap** between the *raw planned* count (which drops immediately as a stage ramps down) and the *ceiling* count (which is deliberately held higher through `gracefulRampDown` so draining VUs keep their slots). The number shown on the progress bar is a third, display-only atomic counter. Three disagreeing numbers here is **expected by design**, not evidence of a bug.

---

## Sub-part #3 — Ctrl+C is an ABORT, not a `gracefulStop` wind-down

**Question:** *"If I kill the test early with Ctrl+C, some VUs run longer than `gracefulStop` allows."*

**Answer:** Ctrl+C does **not** engage the per-executor `gracefulStop` timer at all. The first `SIGINT`/`SIGTERM` triggers a **cooperative abort** that cancels the run context; how quickly an interrupted iteration then ends is governed by **cooperative context cancellation**, not by the `gracefulStop` duration. So an iteration whose work does not promptly observe context cancellation *can* keep running past the `gracefulStop` boundary — because on abort, `gracefulStop` is simply not the mechanism in play.

### The signal path, traced end to end

**CLI signal trap — `cmd/common.go`.** `handleTestAbortSignals(...)` (`common.go:97`) registers for interrupt signals with `gs.SignalNotify(sigC, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)` (`common.go:101`). The **first** signal calls the graceful-stop handler `gracefulStopHandler(sig)` (`common.go:106`); the **second** signal calls `onHardStop(sig)` (`common.go:114`) and then immediately exits with `gs.OSExit(int(exitcodes.ExternalAbort))` (`common.go:118`). The second-signal path even carries a comment referencing the historical issue it prevents:

```text
// If we get a second signal, we immediately exit, so something like
// https://github.com/k6io/k6/issues/971 never happens again
```

**Signal closures — `cmd/run.go`.** The context tree is built as `globalCtx, globalCancel := context.WithCancel(c.gs.Ctx)` (`run.go:70`), `lingerCtx, lingerCancel := context.WithCancel(globalCtx)` (`run.go:75`), and `runCtx, runAbort := execution.NewTestRunContext(lingerCtx, logger)` (`run.go:81`). The first-signal handler is `gracefulStop := func(sig os.Signal) {` (`run.go:349`); it aborts the run with a wrapped error whose format string is (verbatim, `run.go:354`):

```text
test run was aborted because k6 received a '%s' signal
```

wrapped with `exitcodes.ExternalAbort` and `errext.AbortedByUser`, then calls `lingerCancel()` (`run.go:357`). The second-signal handler is `onHardStop := func(sig os.Signal) {` (`run.go:359`); it logs `"Aborting k6 in response to signal"` (`run.go:360`) and calls `globalCancel()` (`run.go:361`). The trap is wired at `run.go:363` and the scheduler is started at `execScheduler.Run(globalCtx, runCtx, samples)` (`run.go:397`).

**Abort controller — `execution/abort.go`.** `NewTestRunContext(...)` (`abort.go:49`) returns a context plus an abort func. Internally `testAbortController.abort(err error)` keeps **only the first reason** (comment at `abort.go:20`: `// only the first reason will be kept, other will be logged`) and then calls `tac.cancel()` (`abort.go:35`). `AbortTestRun` (`abort.go:64`) and `GetCancelReasonIfTestAborted` (`abort.go:77`) expose the mechanism.

**Scheduler — `execution/scheduler.go`.** `Run(...)` (`scheduler.go:419`) hands executors a cancellable child context: `executorsRunCtx, executorsRunCancel := context.WithCancel(withExecStateCtx)` (`scheduler.go:497`). A deferred block detects abort and reclassifies the run as interrupted: `GetCancelReasonIfTestAborted(runCtx)` (`scheduler.go:426`) → `e.state.SetExecutionStatus(lib.ExecutionStatusInterrupted)` (`scheduler.go:428`). The progress line uses the format `"%d complete and %d interrupted iterations"` (part of the format at `scheduler.go:156`).

**Duration contexts — `lib/executor/helpers.go`.** `getDurationContexts(...)` (`helpers.go:168`) computes `maxEndTime := startTime.Add(regularDuration + gracefulStop)` (`helpers.go:172`), then `maxDurationCtx, maxDurationCancel = context.WithDeadline(parentCtx, maxEndTime)` (`helpers.go:174`) and `regDurationCtx, _ = context.WithDeadline(maxDurationCtx, startTime.Add(regularDuration))` (`helpers.go:178`). Crucially, its comment explains the abort short-circuit (verbatim, `helpers.go:165-167`):

```text
//   - If the whole test is aborted, the parent context will be cancelled, so
//     that will also cancel these contexts, thus the "general abort" case is
//     handled transparently.
```

Ctrl+C cancels an **ancestor** of `maxDurationCtx`, so the whole `regDurationCtx → maxDurationCtx → parent` deadline chain is short-circuited immediately — the `gracefulStop`-derived deadline (`regularDuration + gracefulStop`) never gets a chance to be the governing bound.

The default `gracefulStop` value is defined as `var DefaultGracefulStopValue = 30 * time.Second` (`lib/executor/base_config.go:20`), wired as the default at `base_config.go:45` and read through `GetGracefulStop()` (`base_config.go:97`). The abort exit code `ExternalAbort` is `105` (`errext/exitcodes/codes.go:41`).

### Verbatim live SIGINT reproduction

Script `/tmp/ramp_long.js` (written **outside** the repository):

```js
import { sleep } from 'k6';
export const options = {
  scenarios: { ramp: { executor: 'ramping-vus', startVUs: 0,
    stages: [ { duration: '2s', target: 3 }, { duration: '60s', target: 3 } ],
    gracefulStop: '1s' } },
};
export default function () { sleep(30); }
```

`/tmp/k6bin run /tmp/ramp_long.js` was launched in the background and `kill -INT <pid>` was sent after ~5 seconds. **Observed output (verbatim):**

```text
running (0m01.0s), 1/3 VUs, 0 complete and 0 interrupted iterations
running (0m02.0s), 2/3 VUs, 0 complete and 0 interrupted iterations
running (0m03.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
running (0m04.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
running (0m05.0s), 0/3 VUs, 0 complete and 3 interrupted iterations
time="2026-07-01T03:58:18Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

- Process exit code was **`105`** (`wait rc=105`), which equals `exitcodes.ExternalAbort` (`errext/exitcodes/codes.go:41`).
- The error text matches the `run.go:354` format string exactly — `'interrupt'` is the `%s` signal name.
- `3 interrupted iterations` is rendered by the scheduler progress format (`scheduler.go:156`).

**Honest interpretation.** In *this* reproduction the default function is `sleep(30)`, and k6's `sleep()` **is** interruptible, so all three iterations were cut immediately on abort — the VU count dropped `3/3 → 0/3` within the same second and the process exited promptly (well under the `1s` `gracefulStop`). This run therefore demonstrates the **abort path and its exit code**, and shows the abort is **not bounded by `gracefulStop`**.

The user's "runs longer than `gracefulStop`" happens when an iteration's work does **not** promptly observe context cancellation — e.g. a long blocking call that ignores `ctx`. Because Ctrl+C is an *abort* (cooperative cancellation) rather than the `gracefulStop` timer, nothing forcibly kills such an iteration at the `gracefulStop` boundary; it ends only when the work returns or when the process exits on a second signal. This is a **design consequence of cooperative cancellation**, not a leak or a bug.

**Flagged as not directly reproduced here:** we did **not** exhibit a single iteration provably outliving `gracefulStop`, because our `sleep()` was cooperative and cancelled promptly. That specific overrun is *explained* from the context-cancellation code above rather than *timed* in a measurement — see the Caveats section.

---

## Sub-part #4 — Execution-segment striping: the per-instance counts always sum to the maximum

**Question:** *"…with execution segments across 3 instances, one instance shows more VUs than the others, with the sum exceeding the configured maximum."*

**Answer (correcting the user):** With a 3-instance sequence such as `0,1/3,2/3,1`, **one instance does receive one more VU than the others** — that asymmetry is real and is what the user noticed — **but the per-instance counts always sum to exactly the configured maximum and never exceed it.** The belief that the sum "exceeds the configured maximum" is **incorrect**, and the observed output below proves it.

### The code that distributes VUs across segments

In `lib/execution_segment.go` (`package lib`):

- `ExecutionSegmentSequence` is `type ExecutionSegmentSequence []*ExecutionSegment` (`execution_segment.go:300`).
- Sequences are parsed by `NewExecutionSegmentSequenceFromString(...)` (`execution_segment.go:324`).
- A per-instance tuple is built by `NewExecutionTuple(...)` (`execution_segment.go:723`).
- The scaling entry point is `ExecutionTuple.ScaleInt64(value int64) int64` (`execution_segment.go:734`), which short-circuits when there is no segmentation and otherwise delegates to the sequence's rational-arithmetic scaler (verbatim, `execution_segment.go:734-739`):

  ```go
  func (et *ExecutionTuple) ScaleInt64(value int64) int64 {
  	if len(et.Sequence.ExecutionSegmentSequence) == 1 {
  		return value // if we don't have any segmentation, just return the original value
  	}
  	return et.Sequence.ScaleInt64(et.SegmentIndex, value)
  }
  ```

- The underlying striping math lives in `ExecutionSegmentSequenceWrapper.ScaleInt64(segmentIndex int, value int64) int64` (`execution_segment.go:580`).
- The iterator helper `SegmentedIndex` (`execution_segment.go:768`) carries an explicit thread-safety note in the comment immediately preceding it (verbatim, `execution_segment.go:762-764`):

  ```text
  // SegmentedIndex is an iterator that returns both the scaled and the unscaled
  // sequential values according to the given ExecutionTuple. It is not
  // thread-safe, concurrent access has to be externally synchronized.
  ```

  The relevant nuance: the *distribution math* is deterministic and coordination-free across instances (each instance computes its own share from rational arithmetic), while the *iterator struct* requires external synchronization only if a single instance shares one iterator across goroutines.

### Verbatim segment-sum observation

A temporary test was written into the repo, run, and then **deleted** (leaving the tree clean). Its body:

```go
package lib

import (
	"fmt"
	"testing"
)

func TestBlitzyObsSegmentSum(t *testing.T) {
	ess, err := NewExecutionSegmentSequenceFromString("0,1/3,2/3,1")
	if err != nil {
		t.Fatal(err)
	}
	for _, maxVUs := range []int64{100, 10, 7, 5, 1} {
		perSegment := make([]int64, len(ess))
		var sum int64
		for i := range ess {
			et, err := NewExecutionTuple(ess[i], &ess)
			if err != nil {
				t.Fatal(err)
			}
			perSegment[i] = et.ScaleInt64(maxVUs)
			sum += perSegment[i]
		}
		fmt.Printf("BLITZYOBS maxVUs=%d perSegment=%v sum=%d equalsMax=%v\n", maxVUs, perSegment, sum, sum == maxVUs)
	}
}
```

Command:

```text
CGO_ENABLED=0 GOFLAGS=-mod=vendor GOPROXY=off go test -count=1 -run '^TestBlitzyObsSegmentSum$' -v ./lib/
```

**Observed output (verbatim):**

```text
=== RUN   TestBlitzyObsSegmentSum
BLITZYOBS maxVUs=100 perSegment=[34 33 33] sum=100 equalsMax=true
BLITZYOBS maxVUs=10 perSegment=[4 3 3] sum=10 equalsMax=true
BLITZYOBS maxVUs=7 perSegment=[3 2 2] sum=7 equalsMax=true
BLITZYOBS maxVUs=5 perSegment=[2 2 1] sum=5 equalsMax=true
BLITZYOBS maxVUs=1 perSegment=[1 0 0] sum=1 equalsMax=true
--- PASS: TestBlitzyObsSegmentSum (0.00s)
PASS
ok  	go.k6.io/k6/lib	0.004s
```

**The exact asymmetry-but-exact-sum:**

| `maxVUs` | per-segment `[seg0 seg1 seg2]` | sum | `equalsMax` |
|---:|:---|---:|:---:|
| 100 | `[34 33 33]` | 100 | `true` |
| 10 | `[4 3 3]` | 10 | `true` |
| 7 | `[3 2 2]` | 7 | `true` |
| 5 | `[2 2 1]` | 5 | `true` |
| 1 | `[1 0 0]` | 1 | `true` |

The first instance gets **34** while the other two get **33** (and `4` vs `3`, `3` vs `2`, `2` vs `1`), so one instance legitimately shows more VUs. But in **every** case `sum == maxVUs` and `equalsMax=true`.

**Rationale / verdict:** this is deterministic rational-arithmetic ("striping") distribution — elements are maximally spaced across segments, producing unequal-but-exact partitions whose parts sum to the whole; the same test launched repeatedly is deterministic (no randomness). **The sum never exceeds the configured maximum.** The user's asymmetry observation is correct; the "sum exceeds the maximum" conclusion is **incorrect**, as the `BLITZYOBS` output shows directly.

**Cleanup evidence:** after `rm -f lib/blitzy_adhoc_test_segment_test.go`, `git status --porcelain` returned empty (clean tree), demonstrating the read-only guarantee held.

---

## Sub-part #5 — Race between the two handlers? VU-buffer leak? A trace of "simultaneous" modification

**Question:** *"Is there a race condition between the two handler goroutines? Is the VU buffer leaking? Can someone trace what happens when both handlers modify VU state at the same time?"*

**Answer:** **No race** between the two handlers, and **no VU-buffer leak.** The two handler closures **never touch shared state concurrently**, because of a **happens-before ordering** in `Run()`; and the per-VU `vuHandle` is independently synchronized (atomic state + mutex) for its own loop goroutine. The Go race detector reports **zero** data races.

### The happens-before ordering (why the handlers cannot run "at the same time")

Inside `Run()` (`ramping_vus.go`), the two handler closures are created once, then used in a strict order (verbatim, `ramping_vus.go:540-559`):

```go
	defer runState.wg.Wait()
	// this will populate stopped VUs and run runLoopsIfPossible on each VU
	// handle in a new goroutine
	runState.runLoopsIfPossible(maxDurationCtx, cancel)

	var (
		handleNewMaxAllowedVUs = runState.maxAllowedVUsHandlerStrategy()
		handleNewScheduledVUs  = runState.scheduledVUsHandlerStrategy()
	)
	handledGracefulSteps := runState.iterateSteps(
		ctx,
		handleNewMaxAllowedVUs,
		handleNewScheduledVUs,
	)
	go runState.runRemainingGracefulSteps(
		ctx,
		handleNewMaxAllowedVUs,
		handledGracefulSteps,
	)
	return nil
```

The critical facts:

1. `iterateSteps(...)` is called **synchronously** — its result is assigned to `handledGracefulSteps` (`ramping_vus.go:549-553`). Being a plain function call, it **fully returns** before the next statement executes.
2. Only *after* it returns is the graceful goroutine spawned: `go runState.runRemainingGracefulSteps(...)` (`ramping_vus.go:554-558`). The `go` statement establishes a happens-before edge — everything `iterateSteps` did is visible to the new goroutine.
3. Inside `iterateSteps` (`ramping_vus.go:622`), **both** handlers are invoked, but **sequentially from the single `Run` goroutine**: the graceful/max-allowed handler at `handleNewMaxAllowedVUs(g)` (`ramping_vus.go:634`) and the scheduled handler at `handleNewScheduledVUs(r)` (`ramping_vus.go:640`); it returns `j` (`ramping_vus.go:644`).
4. Inside `runRemainingGracefulSteps` (`ramping_vus.go:654`), **only** the max-allowed handler is invoked: `handleNewMaxAllowedVUs(s)` (`ramping_vus.go:664`).

Therefore, after `iterateSteps` returns, the scheduled handler is **never called again**, and only the max-allowed handler's closure-local `cur` is mutated — by exactly **one** goroutine. There is no window in which both handlers' `cur` counters are touched concurrently. The user's premise — "both handlers modify VU state at the same time" — **does not occur** at this commit.

### No VU-buffer leak

The "VU buffer" is the pool of `vuHandles`. The getVU closure calls `GetPlannedVU(...)` (`ramping_vus.go:594`) and, on error, logs the exact string and cancels (verbatim, `ramping_vus.go:596-597`):

```go
			rs.executor.logger.WithError(err).Error("Cannot get a VU from the buffer")
			cancel()
```

VU lifecycle is balanced: `rs.wg.Add(1)` on checkout (`ramping_vus.go:600`) is paired with `rs.wg.Done()` on return (`ramping_vus.go:608`), and `Run()` blocks on `defer runState.wg.Wait()` (`ramping_vus.go:540`) before returning. Because every checked-out VU is waited on, VUs are accounted for at shutdown — they are not leaked. The `"Cannot get a VU from the buffer"` message is a **shutdown-time guard** (it triggers a cancel), not a symptom of a leak.

### Per-VU synchronization (why the VU's own loop goroutine is safe)

`runLoopsIfPossible` runs one goroutine per VU concurrently with `start`/`gracefulStop`/`hardStop`. It is synchronized with a fast atomic read plus a mutex slow path: state is read via `atomic.LoadInt32((*int32)(&vh.state))` on the fast path (`vu_handle.go:204`) and falls to `vh.mutex.Lock()` on the slow path (`vu_handle.go:210`); writes go through `changeState` → `atomic.StoreInt32` (`vu_handle.go:144`). This atomic-plus-mutex design is why a VU's loop can safely coexist with lifecycle calls.

### Verbatim race-detector run

Command:

```text
CGO_ENABLED=1 GOFLAGS=-mod=vendor GOPROXY=off go test -race -count=1 \
  -run 'TestVUHandleRace|TestVUHandleStartStopRace|TestSumRandomSegmentSequenceMatchesNoSegment|TestRampingVUsRampDownNoWobble|TestRampingVUsGracefulStopWaits|TestRampingVUsGracefulStopStops' \
  ./lib/executor/
```

**Observed result (verbatim):**

```text
ok  	go.k6.io/k6/lib/executor	7.060s
-> RETURN_CODE=0, DATA RACE occurrences: 0
```

The elapsed time varies slightly run to run (the AAP had recorded `7.058s`; this run measured `7.060s`); the invariants that matter are **`RETURN_CODE=0`** and **`0`** data races. The referenced tests are `TestVUHandleRace` (`lib/executor/vu_handle_test.go:25`), `TestVUHandleStartStopRace` (`vu_handle_test.go:114`), `TestSumRandomSegmentSequenceMatchesNoSegment` (`lib/executor/ramping_vus_test.go:1112`), `TestRampingVUsGracefulStopWaits` (`ramping_vus_test.go:148`), `TestRampingVUsGracefulStopStops` (`ramping_vus_test.go:194`), and `TestRampingVUsRampDownNoWobble` (`ramping_vus_test.go:376`).

### Supporting evidence: verbose graceful tests

Command (verbose, without `-race`):

```text
CGO_ENABLED=0 GOFLAGS=-mod=vendor GOPROXY=off go test -count=1 -v \
  -run 'TestRampingVUsGracefulStopWaits|TestRampingVUsGracefulStopStops|TestRampingVUsGracefulRampDown|TestRampingVUsRampDownNoWobble' \
  ./lib/executor/
```

**Observed (verbatim, `--- PASS` markers):**

```text
--- PASS: TestRampingVUsGracefulStopWaits (1.50s)
--- PASS: TestRampingVUsGracefulStopStops (2.50s)
--- PASS: TestRampingVUsGracefulRampDown (2.50s)
--- PASS: TestRampingVUsRampDownNoWobble (6.02s)
ok  	go.k6.io/k6/lib/executor	6.022s
```

### Supporting evidence: the normal run banner (reinforces #1 and #3)

From `/tmp/ramp.js` (a `ramping-vus` scenario: stages `2s → 5`, `2s → 0`, `gracefulRampDown: '10s'`, `gracefulStop: '5s'`), `/tmp/k6bin run /tmp/ramp.js` produced (verbatim):

```text
     execution: local
        script: /tmp/ramp.js
     scenarios: (100.00%) 1 scenario, 5 max VUs, 9s max duration (incl. graceful stop):
              * ramp: Up to 5 looping VUs for 4s over 2 stages (gracefulRampDown: 10s, gracefulStop: 5s)
```

`9s max = 4s regular + 5s gracefulStop`, which confirms `helpers.go:172` (`maxEndTime = startTime + regularDuration + gracefulStop`). The banner also shows `gracefulRampDown: 10s` and `gracefulStop: 5s` as **distinct windows** — reinforcing that #1 (draining) and #3 (abort vs. stop) concern different mechanisms.

**Rationale / verdict for #5:**
- "Is there a race between the two handler goroutines?" — **No.** Happens-before ordering means the scheduled handler stops being called before the graceful goroutine starts; the race detector confirms `0` data races.
- "Is the VU buffer leaking?" — **No.** `wg.Add`/`wg.Done` balance VU lifecycle and `wg.Wait()` gates `Run()`'s return; the buffer error is a normal shutdown guard.
- "Trace what happens when both handlers modify VU state at the same time." — At this commit they **do not** run at the same time; the trace above shows the synchronous `iterateSteps` completes before the goroutine is spawned, after which only the max-allowed handler continues. The per-VU `vuHandle` remains safe for its own concurrent loop goroutine via atomic + mutex.

---

## Corroboration with official k6 documentation and design history

These external sources corroborate the code-grounded findings above. They are supporting context only — the code and its observed behavior remain the source of truth. Short quotes (in quotation marks) are used with links; the rest is synthesized.

- **`gracefulStop` is available for all executors and defaults to 30s.** The Grafana "Graceful stop" docs state it is "available for all executors" and "The default value is 30s." — matching `DefaultGracefulStopValue = 30 * time.Second` (`base_config.go:20`). Source: [Grafana k6 — Graceful stop](https://grafana.com/docs/k6/latest/using-k6/scenarios/concepts/graceful-stop/).
- **`gracefulRampDown` is ramping-vus-specific and separate from `gracefulStop` (also 30s default).** The ramping-vus reference describes it as "Time to wait for iterations to finish when ramping down VUs" and notes "This is separate from gracefulStop." — matching `ramping_vus.go:52`. Source: [Grafana k6 — Ramping VUs](https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/ramping-vus/).
- **Graceful periods let in-flight iterations finish but start no new ones.** The k6 v0.27.0 release notes explain that for those periods "no new iterations will be started by the executors," while running iterations are allowed to finish — corroborating the #1 draining-state conclusion. Source: [k6 v0.27.0 release notes](https://github.com/grafana/k6/releases/tag/v0.27.0).
- **A blocking iteration is bounded by the context deadline (stages + graceful window), not by `gracefulStop` on abort.** In the ramping-vus refactor discussion, a maintainer notes "the context used has the deadline of the stages durations + gracefulShutDown" — corroborating the #3 trace that Ctrl+C cancels an ancestor context and that a non-cooperative iteration is bounded by the deadline chain, not the `gracefulStop` timer. Source: [grafana/k6 issue #2149 — Polish: RampingVUs](https://github.com/grafana/k6/issues/2149).
- **Segment distribution is deterministic and non-overlapping (parts sum to the whole, no randomness).** This corroborates the #4 result that per-instance counts are unequal but sum exactly to the maximum. Source: [Grafana k6 — Execution segments](https://grafana.com/docs/k6/latest/using-k6/scenarios/advanced-examples/#execution-segments) (concept) and the striping design.

**Honesty note (related upstream context, not proof of a defect here).** A *later* upstream report, [grafana/k6 issue #4661](https://github.com/grafana/k6/issues/4661) (2025), describes a superficially similar symptom — a reporter who "encountered a bug in ramping-vus executor" observed VUs looping only the first step after a hard stop, and found a workaround by "increasing gracefulRampDown … to make sure all VU iterations will be executed to the end." This is presented purely as evidence that *this symptom class has been discussed upstream*; it is **not** proof of a defect at commit `ddc3b0b1d23c`. At this commit, the race detector shows `0` races and the segment sums are exact, so the code and observed output — not a later report about a different scenario — govern this answer.

---

## Coverage pass (re-reading the question)

Confirming each distinct ask is addressed:

- [x] **#1 — "stuck VUs"** when stages ramp fast and `gracefulRampDown` is long → answered: `toGracefulStop`/`toHardStop` are transitional draining states (`vu_handle.go:15-22`), and `reserveVUsForGracefulRampDowns` (`ramping_vus.go:307`) deliberately holds the ceiling high during the `gracefulRampDown` window. By design; not a leak.
- [x] **#2 — scheduled vs. graceful count "mismatch"** → answered: three distinct counters by design — raw-planned `cur` (`ramping_vus.go:679`), ceiling `cur` (`ramping_vus.go:668`), and progress-only `activeVUsCount` (`ramping_vus.go:569`).
- [x] **#3 — Ctrl+C → VUs run longer than `gracefulStop`** → answered: Ctrl+C is a cooperative **abort** (context cancellation) via `cmd/common.go:97` → `cmd/run.go:349/354` → `execution/abort.go` → `scheduler.go:497` → `helpers.go:165-174`, *not* the `gracefulStop` timer. Live repro shows the abort message and **exit code 105**; the overrun of non-cooperative work is explained as a consequence of cooperative cancellation (and flagged as not directly timed).
- [x] **#4 — 3 instances, "sum exceeds maximum"** → answered and **corrected**: distribution is asymmetric (`[34 33 33]`, etc.) but `sum == maxVUs` in every observed case (verbatim `BLITZYOBS` table via `ExecutionTuple.ScaleInt64`, `execution_segment.go:734`). The sum never exceeds the maximum.
- [x] **#5 — race between handlers? buffer leak? trace simultaneous modification** → answered: no simultaneity (happens-before ordering, `ramping_vus.go:549` synchronous then `:554` spawned); race detector reports `DATA RACE occurrences: 0`; no leak (`wg.Add`/`wg.Done`/`wg.Wait()`, `ramping_vus.go:540/600/608`); `"Cannot get a VU from the buffer"` (`ramping_vus.go:596`) is a shutdown guard.

---

## Caveats / what was NOT verified (honesty)

- **The `gracefulStop` overrun was explained, not timed.** We did **not** exhibit a single iteration provably outliving `gracefulStop` under Ctrl+C, because the reproduction used cooperative `sleep()`, which cancels promptly (VUs dropped `3/3 → 0/3` within one second and the process exited well under the `1s` `gracefulStop`). The "overrun" for non-cooperative work is grounded in the context-cancellation code (`helpers.go:165-174`, `cmd/run.go:349-357`) and corroborated by the maintainer note in issue #2149, but it is an inference from the code path rather than a measured timing.
- **Line numbers are from the `ddc3b0b1d23c` checkout.** They were verified against this exact commit. If a reviewer opens a different revision, the numbers may drift; re-open the file to pin exact lines before quoting surrounding code.
- **The built binary self-reports basename `k6bin`, and its embedded `commit/…` depends on the built revision.** It was built to `/tmp/k6bin`, and k6 derives its name from `os.Args[0]`; that basename is cosmetic. The version fields `v0.55.0`, `go1.23.12`, and `linux/amd64` are identical regardless of revision, while the `commit/…` field is the first 10 characters of the built git `HEAD` (`consts.go:31`, `consts.go:35`): it is `commit/ddc3b0b1d2` when building the commit under investigation and reflects a later branch commit's prefix (e.g. `commit/46e149114e` for the doc-only commit `46e149114`) when building the branch tip, which changes only this document. Neither is a version discrepancy.
- **Run-to-run timing values vary.** The race-suite elapsed time (`~7.06s`) and the segment-test elapsed time (`~0.004s`) fluctuate between runs; the SIGINT timestamp is wall-clock. The invariants that carry the conclusions are stable: `sum == maxVUs`, `DATA RACE occurrences: 0`, and exit code `105`.
- **Scope was read-only.** No existing source file was modified; the only repository change is this document. Temporary observation programs were written outside the repo (`/tmp/ramp.js`, `/tmp/ramp_long.js`, `/tmp/k6bin`) or deleted from it (the segment test), and `git status --porcelain` was confirmed to report only this new file.

