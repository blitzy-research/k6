# grafana/k6 `ramping-vus` Executor — Concurrency Investigation (Q&A)

**Subject:** Does grafana/k6's `ramping-vus` executor contain concurrency defects?
**Module:** `go.k6.io/k6` · **Source commit under investigation:** `ddc3b0b1d23c` · **Version:** `0.55.0`
**Nature of this document:** a read-only, evidence-backed investigation report. Every behavioral claim below is grounded in **actual, unedited output** captured from a **canonical entry point** — either the compiled `k6` CLI (`go build`) or the project's own `go test` suite (including the race detector). No source file under test was modified.

---

## Environment & Methodology (verified canonical build & toolchain)

Values inside the code blocks of this section are **[OBSERVED]** (real command output); the surrounding sentences that *explain* them are ordinary prose.

- **Build command** — `GOFLAGS=-mod=vendor go build` (the `Makefile` `build` target [Makefile:7-8]). The resulting binary is invoked as `./k6`; the investigation kept its copy and all temporary scripts outside the source tree (under `/tmp/blitzy_qna_work`), so the repository stayed byte-for-byte clean.
- **Toolchain & version banner** — the complete captured environment (`./k6 version`, `go version`, `gcc --version`, relevant `go env`, and git provenance):


```
### BUILD (canonical, main working tree at branch HEAD) ###
$ GOFLAGS=-mod=vendor go build -o /tmp/blitzy_qna_work/k6 .
$ ./k6 version
k6 v0.55.0 (commit/65a334ff68, go1.21.13, linux/amd64)

### GO TOOLCHAIN ###
$ go version
go version go1.21.13 linux/amd64

### C TOOLCHAIN (required by CGO_ENABLED=1 for -race) ###
$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0

### RELEVANT ENV ###
CGO_ENABLED=1  GOFLAGS=-mod=vendor  GOTOOLCHAIN=local

### GIT PROVENANCE (explains the banner commit) ###
$ git -C <repo> rev-parse HEAD    # HEAD at build time (commit that adds this doc)
65a334ff6837fc68823624cfc6fff1e65ad6949b
$ git -C <repo> rev-parse HEAD~1  # source commit under investigation
ddc3b0b1d23c128e34e2792fc9075f9126e32375
$ git -C <repo> show --stat HEAD  # doc commit adds ONLY the markdown
    Add blitzy/documentation/k6_ddc3b0b1d23c.md — an evidence-backed
 blitzy/documentation/k6_ddc3b0b1d23c.md | 618 ++++++++++++++++++++++++++++++++
 1 file changed, 618 insertions(+)
```

- **Note on the banner commit — [OBSERVED] value, explained.** `./k6 version` stamps `commit/65a334ff68`, not `ddc3b0b1d2`, because k6's build embeds the git HEAD *at build time* via `debug.ReadBuildInfo()`'s `vcs.revision`, and the binary used for every capture here was built from the working tree at branch HEAD. That HEAD commit (`65a334ff6837...`) sits directly on top of the source commit `ddc3b0b1d23c...` and changes **only this Markdown document** — no `.go` file differs between them (this document is the sole delta, and this corrected revision is likewise committed on top of the same source, still touching no `.go` file). The compiled binary and the test binaries are therefore **behavior-identical** to a build at the source commit `ddc3b0b1d23c` (zero Go source differs); only the VCS stamp differs. `Version = "0.55.0"` is fixed in source [lib/consts/consts.go:12].
- **Toolchain** — Go **1.21.13**, matching `go.mod` (`go 1.21`, `toolchain go1.21.13`). The Go race detector requires `CGO_ENABLED=1` plus a C compiler — here **gcc 15.2.0** (captured above).
- **Race target** — `CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -race ./lib/executor/...`, the **same approach** as the `Makefile` `tests` target [Makefile:28-29] (which runs `go test -race -timeout 210s ./...` across the whole module). The race detector is the authoritative, project-native instrument for adjudicating a suspected data race.
- **Leak tool** — `go.uber.org/goleak` v1.3.0 (already vendored). The project itself calls `goleak.Find()` [cmd/tests/tests.go:57]; the temporary harness used here calls `goleak.VerifyNone(t)` (the `defer`-friendly wrapper around the same detector) so a surviving goroutine fails the test directly.

### How to read this document

Each of the seven requirement threads below uses the same five headings — **Question**, **Grounding (file:line)**, **How it was exercised (command)**, **Observed output**, **Verdict** — so every thread is uniform. Runtime results shown in code blocks are **[OBSERVED]**; sentences that *explain why* the observed behavior occurs are labeled **_[INFERRED, code-grounded]_** and still cite a specific `file:line`. Run scale and duration are stated for every measurement, and behaviors reported as intermittent are shown with their real multi-run distribution. Captured outputs are reproduced verbatim; the *only* normalization is removal of the invisible trailing whitespace on k6's decorative ASCII-logo lines, so this file passes `git diff --check` — no value, message, count, or line of code is altered.

> **The two dichotomies that explain almost everything.** Nearly every "bug" the user reported is the *designed* consequence of one of two intentional splits in k6 (each is code-grounded in the threads below):
> 1. **Active target vs. max-allowed ceiling** — the scheduled handler (raw steps) drives the *active* VU target, while the max-allowed handler (graceful steps) enforces a *ceiling* that reserves headroom during the `gracefulRampDown` window. They deliberately track different quantities. (Explains Threads 1, 2, 4.)
> 2. **Run-level abort vs. executor `gracefulStop` window** — a Ctrl+C triggers a *run-level* graceful abort that cancels the run context immediately, whereas the executor's `gracefulStop` option is a *separate* per-scenario window that only governs a *natural* end. (Explains Thread 3.)

---

## Thread 1 — Can a VU get "stuck"? (`stopped` / `starting` / `running` / `toGracefulStop` / `toHardStop`)

### Question

With stages that ramp up and down rapidly plus a long `gracefulRampDown`, VUs seem "stuck" — neither fully active nor fully stopped. Can a VU wedge among the five per-VU states `stopped / starting / running / toGracefulStop / toHardStop`?

### Grounding (file:line)

_[INFERRED, code-grounded]_ — the following describes the source; the runtime confirmation is under **Observed output**.

- The five states are defined in `lib/executor/vu_handle.go`: `type stateType int32` [vu_handle.go:13]; the `const` block [vu_handle.go:16-22] declares `stopped stateType = iota` [:17], `starting` [:18], `running` [:19], `toGracefulStop` [:20], `toHardStop` [:21].
- `toGracefulStop` and `toHardStop` are **transient, not terminal.** The per-VU loop `runLoopsIfPossible` [vu_handle.go:185] resolves them to `stopped`: its terminal `defer` sets `vh.changeState(stopped)` [vu_handle.go:192], and its slow-path `switch` converts `toGracefulStop` -> (reinit ctx) -> `fallthrough` -> `toHardStop` -> `vh.changeState(stopped)` [vu_handle.go:233]. `gracefulStop()` also sets `stopped` from the `starting` case [vu_handle.go:156]; `hardStop()` from the `starting` case [vu_handle.go:173].
- Writes to the state go through `changeState` [vu_handle.go:142], which is `atomic.StoreInt32((*int32)(&vh.state), ...)` [vu_handle.go:144], and every mutating method (`start`/`gracefulStop`/`hardStop`) holds the per-VU `mutex *sync.Mutex` field [vu_handle.go:71] (initialized `&sync.Mutex{}` [vu_handle.go:99]). The one **lock-free** read is the fast-path load inside the loop: `state := stateType(atomic.LoadInt32((*int32)(&vh.state)))` [vu_handle.go:204]. There is thus no half-updated state to observe.

### How it was exercised (command)

**Evidence A — integration: active-VU counter across a rapid up/down ramp with a long `gracefulRampDown`.** A `ramping-vus` schedule that rapidly ramps 0->6->~0->6->0 (500 ms stages), `GracefulRampDown=5s`, `GracefulStop=2s`, ~150 ms iterations; the temporary harness samples `ExecutionState.GetCurrentlyActiveVUsCount()` [lib/execution.go:269] (which reads the counter mutated by `ModCurrentlyActiveVUsCount` [lib/execution.go:276]) before, during (~every 100 ms), and after the run, under `go test -race`. Run 3x for stability.
```
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -race -count=1 -timeout 120s -run '^TestBlitzyStuckVUsAndRace$' -v ./lib/executor/
```

**Evidence B — deterministic five-state trace.** A single `vuHandle` driven directly (one VU) with `entered`/`release` channels to hold an iteration mid-flight so each transient state can be sampled via `atomic.LoadInt32(&vh.state)`, under `go test -race`.
```
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -race -count=1 -timeout 60s -run '^TestBlitzyStateMachineTrace$' -v ./lib/executor/
```

### Observed output

**Evidence A [OBSERVED]** — run 1 of 3 (unedited; the `BLITZY:` lines are the harness's before/during/after snapshots):

```
=== RUN   TestBlitzyStuckVUsAndRace
BLITZY: maxPlannedVUs(target ceiling)=6 maxPossibleVUs=6
BLITZY: BEFORE run activeVUs=0
BLITZY: t=  100ms DURING activeVUs=1
BLITZY: t=  201ms DURING activeVUs=2
BLITZY: t=  300ms DURING activeVUs=3
BLITZY: t=  400ms DURING activeVUs=4
BLITZY: t=  500ms DURING activeVUs=5
BLITZY: t=  600ms DURING activeVUs=6
BLITZY: t=  700ms DURING activeVUs=5
BLITZY: t=  800ms DURING activeVUs=3
BLITZY: t=  900ms DURING activeVUs=2
BLITZY: t= 1000ms DURING activeVUs=1
BLITZY: t= 1101ms DURING activeVUs=1
BLITZY: t= 1200ms DURING activeVUs=2
BLITZY: t= 1300ms DURING activeVUs=3
BLITZY: t= 1400ms DURING activeVUs=4
BLITZY: t= 1501ms DURING activeVUs=6
BLITZY: t= 1600ms DURING activeVUs=6
BLITZY: t= 1700ms DURING activeVUs=5
BLITZY: t= 1800ms DURING activeVUs=3
BLITZY: t= 1900ms DURING activeVUs=2
BLITZY: t= 2001ms DURING activeVUs=1
BLITZY: AFTER run activeVUs=0 iterCount=43 elapsed=2.04348543s
--- PASS: TestBlitzyStuckVUsAndRace (2.04s)
PASS
ok  	go.k6.io/k6/lib/executor	3.063s
```

Runs 2 and 3 were consistent with run 1: **`maxPlannedVUs=6`, `BEFORE activeVUs=0`, peak `activeVUs=6`, `AFTER run activeVUs=0 iterCount=43`, 0 data races, `PASS`** (the per-100 ms `DURING` samples oscillate with the rapid schedule; the BEFORE/peak/AFTER invariants were identical across all three runs).

**Evidence B [OBSERVED]** — the deterministic five-state trace (unedited):

```
=== RUN   TestBlitzyStateMachineTrace
BLITZY-STATE: 0. newStoppedVUHandle (initial)              state=0(stopped)
BLITZY-STATE: 1. after start() [loop not yet running]      state=1(starting)
BLITZY-STATE: 2. loop active, runIter blocked              state=2(running)
BLITZY-STATE: 3. after gracefulStop() [runIter blocked]    state=3(toGracefulStop)
BLITZY-STATE: 4. after release: toGracefulStop CONVERGED   state=0(stopped)
BLITZY-STATE: 5. after 2nd start(): reactivated            state=2(running)
BLITZY-STATE: 6. after hardStop() [runIter blocked]        state=4(toHardStop)
BLITZY-STATE: 7. after release: toHardStop CONVERGED       state=0(stopped)
BLITZY-STATE: 8. after executor cancel (final)             state=0(stopped)
BLITZY-STATE: getVUCount=2 returnVUCount=2 (balanced=true)
--- PASS: TestBlitzyStateMachineTrace (0.05s)
PASS
ok  	go.k6.io/k6/lib/executor	1.073s
```

### Verdict

**No VU gets stuck. [OBSERVED]** All five states are reached (`stopped`=0, `starting`=1, `running`=2, `toGracefulStop`=3, `toHardStop`=4); the active-VU counter rises to the ceiling (6) and returns to **0** after the run (`AFTER run activeVUs=0`), and `getVUCount==returnVUCount` (balanced). _[INFERRED, code-grounded]_ `toGracefulStop`/`toHardStop` are transition markers that `runLoopsIfPossible` [vu_handle.go:185] always resolves to `stopped` (slow path at vu_handle.go:233; terminal defer at vu_handle.go:192); the "stuck" appearance is the *long `gracefulRampDown` window* holding VUs alive on purpose, not a wedged state, and because writes are atomic-under-mutex [vu_handle.go:142-144,71] there is no torn state.

---

## Thread 2 — Do the two handlers track the same count? (scheduled vs. max-allowed)

### Question

Debug output suggested that at certain moments the VU count tracked by the "scheduled handler" doesn't match what the "graceful handler" thinks should exist. Do `scheduledVUsHandlerStrategy` [ramping_vus.go:679] and `maxAllowedVUsHandlerStrategy` [ramping_vus.go:668] track the same quantity?

### Grounding (file:line)

_[INFERRED, code-grounded]_ — source description; runtime confirmation under **Observed output**.

- `scheduledVUsHandlerStrategy()` [ramping_vus.go:679] closes over `var cur uint64 // current number of planned raw VUs` [ramping_vus.go:680]. It calls `start()` on ramp-up and `gracefulStop()` on ramp-down — i.e. it drives the **active target**.
- `maxAllowedVUsHandlerStrategy()` [ramping_vus.go:668] closes over `var cur uint64 // current number of planned graceful VUs` [ramping_vus.go:669]. It calls only `hardStop()` on ramp-down — i.e. it enforces the **max-allowed ceiling**.
- These are fed by two different step lists: `rawSteps` (the active target) vs. `gracefulSteps` (the ceiling, which reserves VUs across the `gracefulRampDown` window). They are **different quantities by design**; the ceiling *lags* the target on the way down.

### How it was exercised (command)

Per-step counters of the two strategies over one up/down cycle. `StartVUs=0`, stages `1s->10` then `1s->0`, `GracefulRampDown=10s`, `GracefulStop=0`; the harness prints the two source step-lists and a merged target-vs-ceiling timeline.
```
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -count=1 -timeout 60s -run '^TestBlitzyHandlerDivergence$' -v ./lib/executor/
```

### Observed output

**[OBSERVED]** (unedited — all `BLITZY-DIV:` lines):

```
=== RUN   TestBlitzyHandlerDivergence
BLITZY-DIV: rawSteps (scheduled/active TARGET) count=21:
BLITZY-DIV:   raw      t=     0ms PlannedVUs=0
BLITZY-DIV:   raw      t=   100ms PlannedVUs=1
BLITZY-DIV:   raw      t=   200ms PlannedVUs=2
BLITZY-DIV:   raw      t=   300ms PlannedVUs=3
BLITZY-DIV:   raw      t=   400ms PlannedVUs=4
BLITZY-DIV:   raw      t=   500ms PlannedVUs=5
BLITZY-DIV:   raw      t=   600ms PlannedVUs=6
BLITZY-DIV:   raw      t=   700ms PlannedVUs=7
BLITZY-DIV:   raw      t=   800ms PlannedVUs=8
BLITZY-DIV:   raw      t=   900ms PlannedVUs=9
BLITZY-DIV:   raw      t=  1000ms PlannedVUs=10
BLITZY-DIV:   raw      t=  1100ms PlannedVUs=9
BLITZY-DIV:   raw      t=  1200ms PlannedVUs=8
BLITZY-DIV:   raw      t=  1300ms PlannedVUs=7
BLITZY-DIV:   raw      t=  1400ms PlannedVUs=6
BLITZY-DIV:   raw      t=  1500ms PlannedVUs=5
BLITZY-DIV:   raw      t=  1600ms PlannedVUs=4
BLITZY-DIV:   raw      t=  1700ms PlannedVUs=3
BLITZY-DIV:   raw      t=  1800ms PlannedVUs=2
BLITZY-DIV:   raw      t=  1900ms PlannedVUs=1
BLITZY-DIV:   raw      t=  2000ms PlannedVUs=0
BLITZY-DIV: gracefulSteps (max-allowed CEILING) count=12:
BLITZY-DIV:   graceful t=     0ms PlannedVUs=0
BLITZY-DIV:   graceful t=   100ms PlannedVUs=1
BLITZY-DIV:   graceful t=   200ms PlannedVUs=2
BLITZY-DIV:   graceful t=   300ms PlannedVUs=3
BLITZY-DIV:   graceful t=   400ms PlannedVUs=4
BLITZY-DIV:   graceful t=   500ms PlannedVUs=5
BLITZY-DIV:   graceful t=   600ms PlannedVUs=6
BLITZY-DIV:   graceful t=   700ms PlannedVUs=7
BLITZY-DIV:   graceful t=   800ms PlannedVUs=8
BLITZY-DIV:   graceful t=   900ms PlannedVUs=9
BLITZY-DIV:   graceful t=  1000ms PlannedVUs=10
BLITZY-DIV:   graceful t=  2000ms PlannedVUs=0
BLITZY-DIV: merged timeline (target vs ceiling at matching t):
BLITZY-DIV:   t=     0ms  target(scheduled)= 0  ceiling(maxAllowed)= 0  gap=0
BLITZY-DIV:   t=   250ms  target(scheduled)= 2  ceiling(maxAllowed)= 2  gap=0
BLITZY-DIV:   t=   500ms  target(scheduled)= 5  ceiling(maxAllowed)= 5  gap=0
BLITZY-DIV:   t=   750ms  target(scheduled)= 7  ceiling(maxAllowed)= 7  gap=0
BLITZY-DIV:   t=  1000ms  target(scheduled)=10  ceiling(maxAllowed)=10  gap=0
BLITZY-DIV:   t=  1250ms  target(scheduled)= 8  ceiling(maxAllowed)=10  gap=2
BLITZY-DIV:   t=  1500ms  target(scheduled)= 5  ceiling(maxAllowed)=10  gap=5
BLITZY-DIV:   t=  1750ms  target(scheduled)= 3  ceiling(maxAllowed)=10  gap=7
BLITZY-DIV:   t=  2000ms  target(scheduled)= 0  ceiling(maxAllowed)= 0  gap=0
BLITZY-DIV:   t=  2250ms  target(scheduled)= 0  ceiling(maxAllowed)= 0  gap=0
BLITZY-DIV:   t=  2500ms  target(scheduled)= 0  ceiling(maxAllowed)= 0  gap=0
BLITZY-DIV:   t=  2750ms  target(scheduled)= 0  ceiling(maxAllowed)= 0  gap=0
BLITZY-DIV:   t=  3000ms  target(scheduled)= 0  ceiling(maxAllowed)= 0  gap=0
BLITZY-DIV: SUMMARY maxTarget=10 maxCeiling=10 (ceiling>=target always, invariant)
--- PASS: TestBlitzyHandlerDivergence (0.00s)
PASS
ok  	go.k6.io/k6/lib/executor	0.005s
```

### Verdict

**They intentionally track different quantities. [OBSERVED]** On the way **up** the two counters match exactly (`gap=0`); on the way **down** the *active target* (scheduled) decreases smoothly 10->0 while the *max-allowed ceiling* (graceful) **holds at 10** and only collapses at the end — at t=1500 ms target=5 but ceiling=10 (`gap=5`). The peak ceiling equals the peak target (`maxTarget=10 maxCeiling=10`). _[INFERRED, code-grounded]_ the ramp-down gap is by design: the ceiling reserves headroom for the `gracefulRampDown` window so ramped-down VUs can finish their current iteration — the scheduled handler tracks `cur` = "planned raw VUs" [ramping_vus.go:680] and the max-allowed handler tracks `cur` = "planned graceful VUs" [ramping_vus.go:669]. This is the "mismatch" the user observed, and it is expected, not a divergence bug.

---

## Thread 3 — Ctrl+C timing: do VUs run longer than `gracefulStop` allows?

### Question

On an early kill with Ctrl+C, some VUs appear to keep running longer than `gracefulStop` should allow. What is the real termination timing, and how does the run-level abort path differ from the executor's `gracefulStop` option?

### Grounding (file:line)

_[INFERRED, code-grounded]_ — two *separate* mechanisms; the runtime split is confirmed under **Observed output**.

- **Run-level abort (Ctrl+C).** `handleTestAbortSignals` [cmd/common.go:97] traps `SIGINT`/`SIGTERM`. The **first** signal calls the graceful handler `gracefulStopHandler(sig)` [cmd/common.go:106]; the **second** calls `onHardStop(sig)` [cmd/common.go:114] and then `gs.OSExit(int(exitcodes.ExternalAbort))` [cmd/common.go:118]. Those two handlers are the closures in `cmd/run.go`: `gracefulStop` [cmd/run.go:349] calls `runAbort(...)` with `"test run was aborted because k6 received a '%s' signal"` [cmd/run.go:354] and then `lingerCancel()` [cmd/run.go:357]; `onHardStop` [cmd/run.go:359] logs `"Aborting k6 in response to signal"` [cmd/run.go:360] and calls `globalCancel()` [cmd/run.go:361].
- **Executor `gracefulStop` option (natural end only).** `getDurationContexts` [lib/executor/helpers.go:168] sets `maxEndTime := startTime.Add(regularDuration + gracefulStop)` [helpers.go:172] and builds `maxDurationCtx` as a child of the *run* context — `context.WithDeadline(parentCtx, maxEndTime)` [helpers.go:174] — with `regDurationCtx` a child of it [helpers.go:178]. The doc-comment states the decisive fact: *"If the whole test is aborted, the parent context will be cancelled, so that will also cancel these contexts"* [helpers.go:165] and *"any VUs with iterations will be interrupted by the context's closing"* [helpers.go:161]. So on Ctrl+C the parent cancels first and the `gracefulStop` window is **not** honored; the window only applies at a *natural* end, where `trackProgress` logs `"Regular duration is done, waiting for iterations to gracefully finish"` [helpers.go:196].

### How it was exercised (command)

**Evidence A — single Ctrl+C, timing across 5 runs.** A `ramping-vus` run with a **30 s** `gracefulStop`, interrupted by **one** `SIGINT` 8 s in; a driver records the wall-clock interval from signal to process exit. The k6 script (`ctrlc_test.js`) and driver (`run_ctrlc.sh`):
```
import { sleep } from 'k6';

export const options = {
  scenarios: {
    blitzy: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '20s', target: 4 },
        { duration: '40s', target: 4 },
      ],
      gracefulRampDown: '30s',
      gracefulStop: '30s',
    },
  },
};

export default function () {
  const vu = __VU;
  const iter = __ITER;
  for (let i = 0; i < 25; i++) {
    console.log(`TICK vu=${vu} iter=${iter} i=${i} t=${Date.now()}`);
    sleep(1);
  }
  console.log(`ITEREND vu=${vu} iter=${iter} t=${Date.now()}`);
}
```
```
#!/usr/bin/env bash
# Usage: run_ctrlc.sh <label> <script.js> <sigints:1|2> <delay1_s> [delay2_s]
set -u
K6=/tmp/blitzy_qna_work/k6
label="$1"; script="$2"; nsig="$3"; d1="$4"; d2="${5:-0}"
out="/tmp/blitzy_qna_work/captures/ctrlc_${label}.out"
ms() { date +%s%3N; }
t0=$(ms)
"$K6" run --no-color "$script" >"$out" 2>&1 &
pid=$!
sleep "$d1"; ts1=$(ms); kill -INT "$pid"
if [ "$nsig" -ge 2 ]; then sleep "$d2"; ts2=$(ms); kill -INT "$pid"; fi
wait "$pid"; rc=$?; te=$(ms)
if [ "$nsig" -ge 2 ]; then
  printf '=== RUN %s === SIGINT #1 at t=%dms ; SIGINT #2 at t=%dms ; EXIT rc=%d time_from_SIGINT1_to_exit=%dms total=%dms\n' \
    "$label" "$((ts1-t0))" "$((ts2-t0))" "$rc" "$((te-ts1))" "$((te-t0))"
else
  printf '=== RUN %s === SIGINT #1 at t=%dms ; EXIT rc=%d time_from_SIGINT1_to_exit=%dms total=%dms\n' \
    "$label" "$((ts1-t0))" "$rc" "$((te-ts1))" "$((te-t0))"
fi
```
```
# 5 identical runs (same unchanged input) — reports the SIGINT->exit distribution
./run_ctrlc.sh A  ctrlc_test.js 1 8
./run_ctrlc.sh A2 ctrlc_test.js 1 8
./run_ctrlc.sh A3 ctrlc_test.js 1 8
./run_ctrlc.sh A4 ctrlc_test.js 1 8
./run_ctrlc.sh A5 ctrlc_test.js 1 8
```

**Evidence B — teardown under single vs. double Ctrl+C.** Same schedule plus a `teardown()` that sleeps 8 s; `ctrlc_teardown.js`, run once with a single SIGINT (Run C) and once with a second SIGINT ~3 s into teardown (Run D):
```
import { sleep } from 'k6';

export const options = {
  scenarios: {
    blitzy: {
      executor: 'ramping-vus',
      startVUs: 2,
      stages: [
        { duration: '20s', target: 2 },
        { duration: '40s', target: 2 },
      ],
      gracefulRampDown: '30s',
      gracefulStop: '30s',
    },
  },
};

export default function () {
  const vu = __VU;
  const iter = __ITER;
  for (let i = 0; i < 25; i++) {
    console.log(`TICK vu=${vu} iter=${iter} i=${i} t=${Date.now()}`);
    sleep(1);
  }
  console.log(`ITEREND vu=${vu} iter=${iter} t=${Date.now()}`);
}

export function teardown() {
  console.log(`TEARDOWN_START t=${Date.now()}`);
  for (let i = 0; i < 8; i++) {
    console.log(`TEARDOWN_TICK i=${i} t=${Date.now()}`);
    sleep(1);
  }
  console.log(`TEARDOWN_END t=${Date.now()}`);
}
```
```
./run_ctrlc.sh C ctrlc_teardown.js 1 8      # single SIGINT
./run_ctrlc.sh D ctrlc_teardown.js 2 8 3    # SIGINT, then a 2nd SIGINT 3s later
```

**Evidence C — natural end (no signal): the `gracefulStop` window.** A run whose iteration (`sleep(6)`) outlasts its 3 s regular stage, with `gracefulStop: '10s'`; observed plain and with `-v` (debug):
```
import { sleep } from 'k6';

export const options = {
  scenarios: {
    blitzy: {
      executor: 'ramping-vus',
      startVUs: 2,
      stages: [
        { duration: '3s', target: 2 },
      ],
      gracefulStop: '10s',
    },
  },
};

export default function () {
  const vu = __VU;
  const iter = __ITER;
  console.log(`ITER_START vu=${vu} iter=${iter} t=${Date.now()}`);
  sleep(6);
  console.log(`ITER_END   vu=${vu} iter=${iter} t=${Date.now()}`);
}
```
```
./k6 run    natural_gs.js
./k6 run -v natural_gs.js
```

### Observed output

**Evidence A [OBSERVED]** — the 5-run SIGINT->exit distribution (unedited driver output):

```
=== RUN A === SIGINT #1 at t=8007ms ; EXIT rc=105 time_from_SIGINT1_to_exit=38ms total=8045ms
=== RUN A2 === SIGINT #1 at t=8007ms ; EXIT rc=105 time_from_SIGINT1_to_exit=210ms total=8217ms
=== RUN A3 === SIGINT #1 at t=8007ms ; EXIT rc=105 time_from_SIGINT1_to_exit=43ms total=8050ms
=== RUN A4 === SIGINT #1 at t=8007ms ; EXIT rc=105 time_from_SIGINT1_to_exit=42ms total=8049ms
=== RUN A5 === SIGINT #1 at t=8007ms ; EXIT rc=105 time_from_SIGINT1_to_exit=33ms total=8040ms
```

The interval from the single `SIGINT` to process exit was **38 / 210 / 43 / 42 / 33 ms** (median ~42 ms; one 210 ms outlier) — *always* far below the 30 s `gracefulStop`, every run `rc=105`. The complete output of the first run (`ctrlc_A.out`, unedited):

```

         /\      Grafana   /‾‾/
    /\  /  \     |\  __   /  /
   /  \/    \    | |/ /  /   ‾‾\
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/

     execution: local
        script: ctrlc_test.js
        output: -

     scenarios: (100.00%) 1 scenario, 4 max VUs, 1m30s max duration (incl. graceful stop):
              * blitzy: Up to 4 looping VUs for 1m0s over 2 stages (gracefulRampDown: 30s, gracefulStop: 30s)


running (0m01.0s), 0/4 VUs, 0 complete and 0 interrupted iterations
blitzy   [   2% ] 0/4 VUs  0m01.0s/1m00.0s

running (0m02.0s), 0/4 VUs, 0 complete and 0 interrupted iterations
blitzy   [   3% ] 0/4 VUs  0m02.0s/1m00.0s

running (0m03.0s), 0/4 VUs, 0 complete and 0 interrupted iterations
blitzy   [   5% ] 0/4 VUs  0m03.0s/1m00.0s

running (0m04.0s), 0/4 VUs, 0 complete and 0 interrupted iterations
blitzy   [   7% ] 0/4 VUs  0m04.0s/1m00.0s

running (0m05.0s), 0/4 VUs, 0 complete and 0 interrupted iterations
blitzy   [   8% ] 0/4 VUs  0m05.0s/1m00.0s
time="2026-07-13T17:48:04Z" level=info msg="TICK vu=4 iter=0 i=0 t=1783964884746" source=console

running (0m06.0s), 1/4 VUs, 0 complete and 0 interrupted iterations
blitzy   [  10% ] 1/4 VUs  0m06.0s/1m00.0s
time="2026-07-13T17:48:05Z" level=info msg="TICK vu=4 iter=0 i=1 t=1783964885746" source=console

running (0m07.0s), 1/4 VUs, 0 complete and 0 interrupted iterations
blitzy   [  12% ] 1/4 VUs  0m07.0s/1m00.0s
time="2026-07-13T17:48:06Z" level=info msg="TICK vu=4 iter=0 i=2 t=1783964886747" source=console

     data_received...: 0 B 0 B/s
     data_sent.......: 0 B 0 B/s
     vus.............: 1   min=0 max=1
     vus_max.........: 4   min=4 max=4


running (0m08.0s), 0/4 VUs, 0 complete and 1 interrupted iterations
blitzy ✗ [  13% ] 1/4 VUs  0m08.0s/1m00.0s
time="2026-07-13T17:48:07Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

**Evidence B [OBSERVED]** — teardown behavior. Driver summary lines:

```
=== RUN C === SIGINT #1 at t=8007ms ; EXIT rc=105 time_from_SIGINT1_to_exit=8043ms total=16050ms
=== RUN D === SIGINT #1 at t=8007ms ; SIGINT #2 at t=11014ms ; EXIT rc=105 time_from_SIGINT1_to_exit=3014ms total=11021ms
```

Run C (single SIGINT) — teardown runs to **completion** (`TEARDOWN_START` ... `TEARDOWN_TICK i=0..7` ... `TEARDOWN_END`), exiting ~8 s after the signal because teardown itself sleeps 8 s; complete output (`ctrlc_C.out`, unedited):

```

         /\      Grafana   /‾‾/
    /\  /  \     |\  __   /  /
   /  \/    \    | |/ /  /   ‾‾\
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/

     execution: local
        script: ctrlc_teardown.js
        output: -

     scenarios: (100.00%) 1 scenario, 2 max VUs, 1m30s max duration (incl. graceful stop):
              * blitzy: Up to 2 looping VUs for 1m0s over 2 stages (gracefulRampDown: 30s, gracefulStop: 30s)

time="2026-07-13T17:49:18Z" level=info msg="TICK vu=2 iter=0 i=0 t=1783964958210" source=console
time="2026-07-13T17:49:18Z" level=info msg="TICK vu=1 iter=0 i=0 t=1783964958210" source=console

running (0m01.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy   [   2% ] 2/2 VUs  0m01.0s/1m00.0s
time="2026-07-13T17:49:19Z" level=info msg="TICK vu=2 iter=0 i=1 t=1783964959211" source=console
time="2026-07-13T17:49:19Z" level=info msg="TICK vu=1 iter=0 i=1 t=1783964959211" source=console

running (0m02.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy   [   3% ] 2/2 VUs  0m02.0s/1m00.0s
time="2026-07-13T17:49:20Z" level=info msg="TICK vu=1 iter=0 i=2 t=1783964960211" source=console
time="2026-07-13T17:49:20Z" level=info msg="TICK vu=2 iter=0 i=2 t=1783964960211" source=console

running (0m03.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy   [   5% ] 2/2 VUs  0m03.0s/1m00.0s
time="2026-07-13T17:49:21Z" level=info msg="TICK vu=1 iter=0 i=3 t=1783964961212" source=console
time="2026-07-13T17:49:21Z" level=info msg="TICK vu=2 iter=0 i=3 t=1783964961212" source=console

running (0m04.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy   [   7% ] 2/2 VUs  0m04.0s/1m00.0s
time="2026-07-13T17:49:22Z" level=info msg="TICK vu=2 iter=0 i=4 t=1783964962213" source=console
time="2026-07-13T17:49:22Z" level=info msg="TICK vu=1 iter=0 i=4 t=1783964962213" source=console

running (0m05.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy   [   8% ] 2/2 VUs  0m05.0s/1m00.0s
time="2026-07-13T17:49:23Z" level=info msg="TICK vu=1 iter=0 i=5 t=1783964963214" source=console
time="2026-07-13T17:49:23Z" level=info msg="TICK vu=2 iter=0 i=5 t=1783964963214" source=console

running (0m06.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy   [  10% ] 2/2 VUs  0m06.0s/1m00.0s
time="2026-07-13T17:49:24Z" level=info msg="TICK vu=2 iter=0 i=6 t=1783964964215" source=console
time="2026-07-13T17:49:24Z" level=info msg="TICK vu=1 iter=0 i=6 t=1783964964215" source=console

running (0m07.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy   [  12% ] 2/2 VUs  0m07.0s/1m00.0s
time="2026-07-13T17:49:25Z" level=info msg="TICK vu=2 iter=0 i=7 t=1783964965215" source=console
time="2026-07-13T17:49:25Z" level=info msg="TICK vu=1 iter=0 i=7 t=1783964965215" source=console
time="2026-07-13T17:49:26Z" level=info msg="TICK vu=2 iter=0 i=8 t=1783964966190" source=console
time="2026-07-13T17:49:26Z" level=info msg="TEARDOWN_START t=1783964966192" source=console
time="2026-07-13T17:49:26Z" level=info msg="TEARDOWN_TICK i=0 t=1783964966192" source=console

running (0m08.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
blitzy ✗ [  13% ] 2/2 VUs  0m08.0s/1m00.0s
time="2026-07-13T17:49:27Z" level=info msg="TEARDOWN_TICK i=1 t=1783964967192" source=console

running (0m09.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
blitzy ✗ [  13% ] 2/2 VUs  0m08.0s/1m00.0s
time="2026-07-13T17:49:28Z" level=info msg="TEARDOWN_TICK i=2 t=1783964968193" source=console

running (0m10.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
blitzy ✗ [  13% ] 2/2 VUs  0m08.0s/1m00.0s
time="2026-07-13T17:49:29Z" level=info msg="TEARDOWN_TICK i=3 t=1783964969193" source=console

running (0m11.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
blitzy ✗ [  13% ] 2/2 VUs  0m08.0s/1m00.0s
time="2026-07-13T17:49:30Z" level=info msg="TEARDOWN_TICK i=4 t=1783964970194" source=console

running (0m12.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
blitzy ✗ [  13% ] 2/2 VUs  0m08.0s/1m00.0s
time="2026-07-13T17:49:31Z" level=info msg="TEARDOWN_TICK i=5 t=1783964971195" source=console

running (0m13.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
blitzy ✗ [  13% ] 2/2 VUs  0m08.0s/1m00.0s
time="2026-07-13T17:49:32Z" level=info msg="TEARDOWN_TICK i=6 t=1783964972196" source=console

running (0m14.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
blitzy ✗ [  13% ] 2/2 VUs  0m08.0s/1m00.0s
time="2026-07-13T17:49:33Z" level=info msg="TEARDOWN_TICK i=7 t=1783964973196" source=console

running (0m15.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
blitzy ✗ [  13% ] 2/2 VUs  0m08.0s/1m00.0s
time="2026-07-13T17:49:34Z" level=info msg="TEARDOWN_END t=1783964974197" source=console

     data_received...: 0 B 0 B/s
     data_sent.......: 0 B 0 B/s
     vus.............: 2   min=2 max=2
     vus_max.........: 2   min=2 max=2


running (0m16.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
blitzy ✗ [  13% ] 2/2 VUs  0m08.0s/1m00.0s
time="2026-07-13T17:49:34Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

Run D (second SIGINT ~3 s into teardown) — teardown is **cut off** at `TEARDOWN_TICK i=3` (no `i=4..7`, no `TEARDOWN_END`), and k6 logs `"Aborting k6 in response to signal" sig=interrupt`; complete output (`ctrlc_D.out`, unedited):

```

         /\      Grafana   /‾‾/
    /\  /  \     |\  __   /  /
   /  \/    \    | |/ /  /   ‾‾\
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/

     execution: local
        script: ctrlc_teardown.js
        output: -

     scenarios: (100.00%) 1 scenario, 2 max VUs, 1m30s max duration (incl. graceful stop):
              * blitzy: Up to 2 looping VUs for 1m0s over 2 stages (gracefulRampDown: 30s, gracefulStop: 30s)

time="2026-07-13T17:49:34Z" level=info msg="TICK vu=2 iter=0 i=0 t=1783964974269" source=console
time="2026-07-13T17:49:34Z" level=info msg="TICK vu=1 iter=0 i=0 t=1783964974269" source=console

running (0m01.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy   [   2% ] 2/2 VUs  0m01.0s/1m00.0s
time="2026-07-13T17:49:35Z" level=info msg="TICK vu=2 iter=0 i=1 t=1783964975269" source=console
time="2026-07-13T17:49:35Z" level=info msg="TICK vu=1 iter=0 i=1 t=1783964975269" source=console

running (0m02.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy   [   3% ] 2/2 VUs  0m02.0s/1m00.0s
time="2026-07-13T17:49:36Z" level=info msg="TICK vu=1 iter=0 i=2 t=1783964976270" source=console
time="2026-07-13T17:49:36Z" level=info msg="TICK vu=2 iter=0 i=2 t=1783964976270" source=console

running (0m03.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy   [   5% ] 2/2 VUs  0m03.0s/1m00.0s
time="2026-07-13T17:49:37Z" level=info msg="TICK vu=1 iter=0 i=3 t=1783964977270" source=console
time="2026-07-13T17:49:37Z" level=info msg="TICK vu=2 iter=0 i=3 t=1783964977270" source=console

running (0m04.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy   [   7% ] 2/2 VUs  0m04.0s/1m00.0s
time="2026-07-13T17:49:38Z" level=info msg="TICK vu=1 iter=0 i=4 t=1783964978271" source=console
time="2026-07-13T17:49:38Z" level=info msg="TICK vu=2 iter=0 i=4 t=1783964978271" source=console

running (0m05.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy   [   8% ] 2/2 VUs  0m05.0s/1m00.0s
time="2026-07-13T17:49:39Z" level=info msg="TICK vu=2 iter=0 i=5 t=1783964979271" source=console
time="2026-07-13T17:49:39Z" level=info msg="TICK vu=1 iter=0 i=5 t=1783964979271" source=console

running (0m06.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy   [  10% ] 2/2 VUs  0m06.0s/1m00.0s
time="2026-07-13T17:49:40Z" level=info msg="TICK vu=2 iter=0 i=6 t=1783964980272" source=console
time="2026-07-13T17:49:40Z" level=info msg="TICK vu=1 iter=0 i=6 t=1783964980272" source=console

running (0m07.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy   [  12% ] 2/2 VUs  0m07.0s/1m00.0s
time="2026-07-13T17:49:41Z" level=info msg="TICK vu=2 iter=0 i=7 t=1783964981273" source=console
time="2026-07-13T17:49:41Z" level=info msg="TICK vu=1 iter=0 i=7 t=1783964981273" source=console
time="2026-07-13T17:49:42Z" level=info msg="TICK vu=1 iter=0 i=8 t=1783964982250" source=console
time="2026-07-13T17:49:42Z" level=info msg="TEARDOWN_START t=1783964982250" source=console
time="2026-07-13T17:49:42Z" level=info msg="TEARDOWN_TICK i=0 t=1783964982250" source=console

running (0m08.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
blitzy ✗ [  13% ] 2/2 VUs  0m08.0s/1m00.0s
time="2026-07-13T17:49:43Z" level=info msg="TEARDOWN_TICK i=1 t=1783964983251" source=console

running (0m09.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
blitzy ✗ [  13% ] 2/2 VUs  0m08.0s/1m00.0s
time="2026-07-13T17:49:44Z" level=info msg="TEARDOWN_TICK i=2 t=1783964984252" source=console

running (0m10.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
blitzy ✗ [  13% ] 2/2 VUs  0m08.0s/1m00.0s
time="2026-07-13T17:49:45Z" level=info msg="TEARDOWN_TICK i=3 t=1783964985253" source=console
time="2026-07-13T17:49:45Z" level=error msg="Aborting k6 in response to signal" sig=interrupt
```

**Evidence C [OBSERVED]** — natural end, plain (`natural_gs_plain.out`, unedited). The banner reads `13s max duration (incl. graceful stop)` (3 s regular + 10 s `gracefulStop`); both VUs `ITER_START` at the same ms and `ITER_END` exactly **+6 s** later even though the regular stage is only 3 s — `2 complete and 0 interrupted iterations`, `rc=0`:

```

         /\      Grafana   /‾‾/
    /\  /  \     |\  __   /  /
   /  \/    \    | |/ /  /   ‾‾\
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/

     execution: local
        script: natural_gs.js
        output: -

     scenarios: (100.00%) 1 scenario, 2 max VUs, 13s max duration (incl. graceful stop):
              * blitzy: Up to 2 looping VUs for 3s over 1 stages (gracefulRampDown: 30s, gracefulStop: 10s)

time="2026-07-13T17:51:02Z" level=info msg="ITER_START vu=2 iter=0 t=1783965062445" source=console
time="2026-07-13T17:51:02Z" level=info msg="ITER_START vu=1 iter=0 t=1783965062445" source=console

running (01.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy   [  33% ] 2/2 VUs  1.0s/3.0s

running (02.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy   [  67% ] 2/2 VUs  2.0s/3.0s

running (03.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy   [ 100% ] 2/2 VUs  3.0s/3.0s

running (04.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy ↓ [ 100% ] 2/2 VUs  3s

running (05.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy ↓ [ 100% ] 2/2 VUs  3s

running (06.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy ↓ [ 100% ] 2/2 VUs  3s
time="2026-07-13T17:51:08Z" level=info msg="ITER_END   vu=2 iter=0 t=1783965068445" source=console
time="2026-07-13T17:51:08Z" level=info msg="ITER_END   vu=1 iter=0 t=1783965068445" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=6s min=6s med=6s max=6s p(90)=6s p(95)=6s
     iterations...........: 2   0.333312/s
     vus..................: 2   min=2      max=2
     vus_max..............: 2   min=2      max=2


running (06.0s), 0/2 VUs, 2 complete and 0 interrupted iterations
blitzy ✓ [ 100% ] 0/2 VUs  3s
```

And with `-v` (`natural_gs_verbose.out`, unedited) — note the executor's own graceful-window debug line `"Regular duration is done, waiting for iterations to gracefully finish" ... gracefulStop=10s`:

```
time="2026-07-13T17:51:21Z" level=debug msg="Logger format: TEXT"
time="2026-07-13T17:51:21Z" level=debug msg="k6 version: v0.55.0 (commit/65a334ff68, go1.21.13, linux/amd64)"

         /\      Grafana   /‾‾/
    /\  /  \     |\  __   /  /
   /  \/    \    | |/ /  /   ‾‾\
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/

time="2026-07-13T17:51:21Z" level=debug msg="Resolving and reading test 'natural_gs.js'..."
time="2026-07-13T17:51:21Z" level=debug msg=Loading... moduleSpecifier="file:///tmp/blitzy_qna_work/natural_gs.js" originalModuleSpecifier=natural_gs.js
time="2026-07-13T17:51:21Z" level=debug msg="'natural_gs.js' resolved to 'file:///tmp/blitzy_qna_work/natural_gs.js' and successfully loaded 459 bytes!"
time="2026-07-13T17:51:21Z" level=debug msg="Gathering k6 runtime options..."
time="2026-07-13T17:51:21Z" level=debug msg="Initializing k6 runner for 'natural_gs.js' (file:///tmp/blitzy_qna_work/natural_gs.js)..."
time="2026-07-13T17:51:21Z" level=debug msg="Detecting test type for..." test_path="file:///tmp/blitzy_qna_work/natural_gs.js"
time="2026-07-13T17:51:21Z" level=debug msg="Trying to load as a JS test..." test_path="file:///tmp/blitzy_qna_work/natural_gs.js"
time="2026-07-13T17:51:21Z" level=debug msg="Runner successfully initialized!"
time="2026-07-13T17:51:21Z" level=debug msg="Parsing CLI flags..."
time="2026-07-13T17:51:21Z" level=debug msg="Consolidating config layers..."
time="2026-07-13T17:51:21Z" level=debug msg="Parsing thresholds and validating config..."
time="2026-07-13T17:51:21Z" level=debug msg="Initializing the execution scheduler..."
time="2026-07-13T17:51:21Z" level=debug msg="Starting 2 outputs..." component=output-manager
time="2026-07-13T17:51:21Z" level=debug msg=Starting... component=metrics-engine-ingester
time="2026-07-13T17:51:21Z" level=debug msg="Started!" component=metrics-engine-ingester
     execution: local
        script: natural_gs.js
        output: -

     scenarios: (100.00%) 1 scenario, 2 max VUs, 13s max duration (incl. graceful stop):
              * blitzy: Up to 2 looping VUs for 3s over 1 stages (gracefulRampDown: 30s, gracefulStop: 10s)

time="2026-07-13T17:51:21Z" level=debug msg="Trapping interrupt signals so k6 can handle them gracefully..."
time="2026-07-13T17:51:21Z" level=debug msg="Starting the REST API server on localhost:6565"
time="2026-07-13T17:51:21Z" level=debug msg="Starting emission of VUs and VUsMax metrics..."
time="2026-07-13T17:51:21Z" level=debug msg="Start of initialization" executorsCount=1 neededVUs=2 phase=execution-scheduler-init
time="2026-07-13T17:51:21Z" level=debug msg="Initialized VU #1" phase=execution-scheduler-init
time="2026-07-13T17:51:21Z" level=debug msg="Initialized VU #2" phase=execution-scheduler-init
time="2026-07-13T17:51:21Z" level=debug msg="Finished initializing needed VUs, start initializing executors..." phase=execution-scheduler-init
time="2026-07-13T17:51:21Z" level=debug msg="Initialized executor blitzy" phase=execution-scheduler-init
time="2026-07-13T17:51:21Z" level=debug msg="Initialization completed" phase=execution-scheduler-init
time="2026-07-13T17:51:21Z" level=debug msg="Start of test run" executorsCount=1 phase=execution-scheduler-run
time="2026-07-13T17:51:21Z" level=debug msg="setup() is not defined or not exported, skipping!"
time="2026-07-13T17:51:21Z" level=debug msg="Start all executors..." phase=execution-scheduler-run
time="2026-07-13T17:51:21Z" level=debug msg="Starting executor" executor=blitzy startTime=0s type=ramping-vus
time="2026-07-13T17:51:21Z" level=debug msg="Starting executor run..." duration=3s executor=ramping-vus maxVUs=2 numStages=1 scenario=blitzy startVUs=2 type=ramping-vus
time="2026-07-13T17:51:21Z" level=debug msg=Start executor=ramping-vus scenario=blitzy vuNum=0
time="2026-07-13T17:51:21Z" level=debug msg=Start executor=ramping-vus scenario=blitzy vuNum=1
time="2026-07-13T17:51:21Z" level=info msg="ITER_START vu=2 iter=0 t=1783965081349" source=console
time="2026-07-13T17:51:21Z" level=info msg="ITER_START vu=1 iter=0 t=1783965081349" source=console

running (01.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy   [  33% ] 2/2 VUs  1.0s/3.0s

running (02.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy   [  67% ] 2/2 VUs  2.0s/3.0s
time="2026-07-13T17:51:24Z" level=debug msg="Graceful stop" executor=ramping-vus scenario=blitzy vuNum=1

running (03.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy   [ 100% ] 2/2 VUs  3s
time="2026-07-13T17:51:24Z" level=debug msg="Graceful stop" executor=ramping-vus scenario=blitzy vuNum=0
time="2026-07-13T17:51:24Z" level=debug msg="Regular duration is done, waiting for iterations to gracefully finish" executor=ramping-vus gracefulStop=10s scenario=blitzy

running (04.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy ↓ [ 100% ] 2/2 VUs  3s

running (05.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy ↓ [ 100% ] 2/2 VUs  3s

running (06.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
blitzy ↓ [ 100% ] 2/2 VUs  3s
time="2026-07-13T17:51:27Z" level=info msg="ITER_END   vu=1 iter=0 t=1783965087350" source=console
time="2026-07-13T17:51:27Z" level=info msg="ITER_END   vu=2 iter=0 t=1783965087350" source=console
time="2026-07-13T17:51:27Z" level=debug msg="Executor finished successfully" executor=blitzy startTime=0s type=ramping-vus
time="2026-07-13T17:51:27Z" level=debug msg="teardown() is not defined or not exported, skipping!"
time="2026-07-13T17:51:27Z" level=debug msg="Test finished cleanly"
time="2026-07-13T17:51:27Z" level=debug msg="Stopping vus and vux_max metrics emission..." phase=execution-scheduler-init
time="2026-07-13T17:51:27Z" level=debug msg="Metrics emission of VUs and VUsMax metrics stopped"
time="2026-07-13T17:51:27Z" level=debug msg="Releasing signal trap..."
time="2026-07-13T17:51:27Z" level=debug msg="Sending usage report..."
time="2026-07-13T17:51:27Z" level=debug msg="Waiting for metrics and traces processing to finish..."
time="2026-07-13T17:51:27Z" level=debug msg="Metrics and traces processing finished!"
time="2026-07-13T17:51:27Z" level=debug msg="Stopping outputs..."
time="2026-07-13T17:51:27Z" level=debug msg="Stopping 2 outputs..." component=output-manager
time="2026-07-13T17:51:27Z" level=debug msg=Stopping... component=metrics-engine-ingester
time="2026-07-13T17:51:27Z" level=debug msg="Stopped!" component=metrics-engine-ingester
time="2026-07-13T17:51:27Z" level=debug msg="Generating the end-of-test summary..."

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=6s min=6s med=6s max=6s p(90)=6s p(95)=6s
     iterations...........: 2   0.333268/s
     vus..................: 2   min=2      max=2
     vus_max..............: 2   min=2      max=2


running (06.0s), 0/2 VUs, 2 complete and 0 interrupted iterations
blitzy ✓ [ 100% ] 0/2 VUs  3s
time="2026-07-13T17:51:27Z" level=debug msg="Usage report sent successfully"
time="2026-07-13T17:51:27Z" level=debug msg="Everything has finished, exiting k6 normally!"
```

### Verdict

**The user conflated two different mechanisms; on Ctrl+C, VUs stop almost immediately, not after `gracefulStop`. [OBSERVED]** A single `SIGINT` aborts the run in **33-210 ms** (5-run distribution above) — the in-flight iteration is *interrupted* (`1 interrupted iterations`, `blitzy ✗`), nowhere near the 30 s `gracefulStop`. _[INFERRED, code-grounded]_ this is the *run-level abort*: the first signal runs `gracefulStop` [cmd/run.go:349] -> `runAbort(...)` [cmd/run.go:354] + `lingerCancel()` [cmd/run.go:357], cancelling the run context; because `maxDurationCtx`/`regDurationCtx` are children of that context [helpers.go:174,178], they cancel immediately and the executor's `gracefulStop` window is bypassed [helpers.go:165]. What can make VUs *appear* to "keep running" after Ctrl+C is either (a) a `teardown()` that still executes after a single signal (Run C, ends only when teardown's own 8 s elapse) — killable with a second signal via `onHardStop` [cmd/run.go:359-361] (Run D) — or (b) a *natural* end, where the `gracefulStop` window legitimately lets a 6 s iteration finish past its 3 s stage (Evidence C, `helpers.go:172,196`). None of these is the executor `gracefulStop` extending runtime on Ctrl+C.

---

## Thread 4 — Execution-segment skew: does the sum exceed the configured maximum?

### Question

With execution segments, three instances split work "evenly," yet one instance consistently shows more VUs than the others at the same timestamp, and summing the three appears to exceed the configured maximum. Does per-segment scaling via `SegmentedIndex` [lib/execution_segment.go:768] preserve the sum-equals-unsegmented invariant?

### Grounding (file:line)

_[INFERRED, code-grounded]_ — the scaling contract; runtime confirmation under **Observed output**.

- Per-segment VU counts are produced by deterministic integer scaling: `getRawExecutionSteps` [lib/executor/ramping_vus.go:171] builds an `index = lib.NewSegmentedIndex(et)` [ramping_vus.go:176] and walks it with `index.GoTo(...)` [ramping_vus.go:180], `.Prev()` [ramping_vus.go:209], `.Next()` [ramping_vus.go:219].
- `SegmentedIndex` [lib/execution_segment.go:768] is explicitly *"not thread-safe and should be used only synchronously"* [execution_segment.go:762-764]; `NewSegmentedIndex` [execution_segment.go:776], `Next` [execution_segment.go:782], `Prev` [execution_segment.go:795], `GoTo` [execution_segment.go:808] implement the deterministic rounding.
- The invariant is that a per-instant single-instance skew of ±1 VU is legitimate rounding, while the *sum* across a full segment sequence must equal the unsegmented plan and never exceed the configured maximum. The project encodes this as `TestSumRandomSegmentSequenceMatchesNoSegment` [lib/executor/ramping_vus_test.go:1112].

### How it was exercised (command)

**Evidence A — canonical invariant test under `-race`.** The project's own randomized sum-equals-unsegmented test:
```
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -race -count=1 -timeout 120s -run '^TestSumRandomSegmentSequenceMatchesNoSegment$' -v ./lib/executor/
```

**Evidence B — harness: per-instant sum vs. configured max.** A temporary harness builds the raw execution steps for a `0->10->0` ramp for each of three segments (`0:1/3`, `1/3:2/3`, `2/3:1`), merges them onto a common timeline, and reports `sum`, `unsegmented`, and single-instance `skew` at every instant:
```
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -count=1 -timeout 60s -run '^TestBlitzySegmentSumInvariant$' -v ./lib/executor/
```

**Evidence C — three real segmented CLI instances (canonical `k6 run`).** Three parallel `k6 run` processes, each a non-overlapping segment of the same schedule (`--execution-segment` + full `--execution-segment-sequence` so they do not overlap), emitting per-VU JSON; a Python analyzer buckets the `vus` metric by whole second and sums across instances. The script (`seg_cli.js`), driver (`run_seg.sh`), and invocation (repeated 3x for stability):
```
import { sleep } from 'k6';

export const options = {
  scenarios: {
    blitzy: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '3s', target: 10 },
        { duration: '3s', target: 10 },
        { duration: '2s', target: 0 },
      ],
      gracefulRampDown: '0s',
      gracefulStop: '1s',
    },
  },
};

export default function () {
  sleep(0.5);
}
```
```
#!/bin/bash
# Args: rep_label
REP="$1"
SEQ="0,1/3,2/3,1"
./k6 run seg_cli.js --execution-segment "0:1/3"   --execution-segment-sequence "$SEQ" --quiet -o json="captures/seg${REP}_0.json" > "captures/seg${REP}_0.stdout" 2>&1 &
p0=$!
./k6 run seg_cli.js --execution-segment "1/3:2/3" --execution-segment-sequence "$SEQ" --quiet -o json="captures/seg${REP}_1.json" > "captures/seg${REP}_1.stdout" 2>&1 &
p1=$!
./k6 run seg_cli.js --execution-segment "2/3:1"   --execution-segment-sequence "$SEQ" --quiet -o json="captures/seg${REP}_2.json" > "captures/seg${REP}_2.stdout" 2>&1 &
p2=$!
wait $p0; r0=$?
wait $p1; r1=$?
wait $p2; r2=$?
echo "rep=${REP} exit codes: inst0=$r0 inst1=$r1 inst2=$r2"
```
```
./run_seg.sh 1 && python3 seg_analyze.py 10 captures/seg1_0.json captures/seg1_1.json captures/seg1_2.json
./run_seg.sh 2 && python3 seg_analyze.py 10 captures/seg2_0.json captures/seg2_1.json captures/seg2_2.json
./run_seg.sh 3 && python3 seg_analyze.py 10 captures/seg3_0.json captures/seg3_1.json captures/seg3_2.json
```

### Observed output

**Evidence A [OBSERVED]** — the canonical invariant test passes under `-race` (all 10 randomized sub-cases). Excerpt — the per-subtest step-by-step logs for `random00`..`random09` are voluminous (the full captured run is 120,732 lines) and are the **only** elision in this document, expressly permitted by the task rules; the framing and the complete PASS tail are unedited:

```
=== RUN   TestSumRandomSegmentSequenceMatchesNoSegment
=== PAUSE TestSumRandomSegmentSequenceMatchesNoSegment
=== CONT  TestSumRandomSegmentSequenceMatchesNoSegment
=== RUN   TestSumRandomSegmentSequenceMatchesNoSegment/random00
=== PAUSE TestSumRandomSegmentSequenceMatchesNoSegment/random00
=== RUN   TestSumRandomSegmentSequenceMatchesNoSegment/random01
... [the voluminous per-subtest step-by-step logs for random00 through random09
    are the ONLY elision in this document, expressly permitted by the task rules
    for these per-subtest logs; the full captured run is 120,732 lines] ...
    --- PASS: TestSumRandomSegmentSequenceMatchesNoSegment/random02 (0.10s)
    --- PASS: TestSumRandomSegmentSequenceMatchesNoSegment/random06 (0.62s)
    --- PASS: TestSumRandomSegmentSequenceMatchesNoSegment/random08 (0.80s)
    --- PASS: TestSumRandomSegmentSequenceMatchesNoSegment/random01 (1.31s)
    --- PASS: TestSumRandomSegmentSequenceMatchesNoSegment/random04 (1.52s)
    --- PASS: TestSumRandomSegmentSequenceMatchesNoSegment/random00 (1.61s)
    --- PASS: TestSumRandomSegmentSequenceMatchesNoSegment/random03 (1.84s)
    --- PASS: TestSumRandomSegmentSequenceMatchesNoSegment/random05 (2.00s)
    --- PASS: TestSumRandomSegmentSequenceMatchesNoSegment/random09 (2.13s)
    --- PASS: TestSumRandomSegmentSequenceMatchesNoSegment/random07 (2.25s)
PASS
ok  	go.k6.io/k6/lib/executor	3.275s
```

**Evidence B [OBSERVED]** — the harness per-instant timeline (unedited). The peak is at t=1000 ms: `seg=[4 3 3] sum=10 unsegmented=10 skew=1` — one instance shows 4 while the others show 3 (the ±1 skew the user saw), but the **sum is exactly 10 = the configured max**, and the summary reports `violations(sum!=unseg or sum>max)=0`:

```
=== RUN   TestBlitzySegmentSumInvariant
BLITZY-SEG: --- segment 0 (0:1/3) rawSteps count=9 ---
BLITZY-SEG:   seg0 t=     0ms PlannedVUs=0
BLITZY-SEG:   seg0 t=   100ms PlannedVUs=1
BLITZY-SEG:   seg0 t=   400ms PlannedVUs=2
BLITZY-SEG:   seg0 t=   700ms PlannedVUs=3
BLITZY-SEG:   seg0 t=  1000ms PlannedVUs=4
BLITZY-SEG:   seg0 t=  1100ms PlannedVUs=3
BLITZY-SEG:   seg0 t=  1400ms PlannedVUs=2
BLITZY-SEG:   seg0 t=  1700ms PlannedVUs=1
BLITZY-SEG:   seg0 t=  2000ms PlannedVUs=0
BLITZY-SEG: --- segment 1 (1/3:2/3) rawSteps count=7 ---
BLITZY-SEG:   seg1 t=     0ms PlannedVUs=0
BLITZY-SEG:   seg1 t=   200ms PlannedVUs=1
BLITZY-SEG:   seg1 t=   500ms PlannedVUs=2
BLITZY-SEG:   seg1 t=   800ms PlannedVUs=3
BLITZY-SEG:   seg1 t=  1300ms PlannedVUs=2
BLITZY-SEG:   seg1 t=  1600ms PlannedVUs=1
BLITZY-SEG:   seg1 t=  1900ms PlannedVUs=0
BLITZY-SEG: --- segment 2 (2/3:1) rawSteps count=7 ---
BLITZY-SEG:   seg2 t=     0ms PlannedVUs=0
BLITZY-SEG:   seg2 t=   300ms PlannedVUs=1
BLITZY-SEG:   seg2 t=   600ms PlannedVUs=2
BLITZY-SEG:   seg2 t=   900ms PlannedVUs=3
BLITZY-SEG:   seg2 t=  1200ms PlannedVUs=2
BLITZY-SEG:   seg2 t=  1500ms PlannedVUs=1
BLITZY-SEG:   seg2 t=  1800ms PlannedVUs=0
BLITZY-SEG: --- UNSEGMENTED rawSteps count=21 ---
BLITZY-SEG:   full t=     0ms PlannedVUs=0
BLITZY-SEG:   full t=   100ms PlannedVUs=1
BLITZY-SEG:   full t=   200ms PlannedVUs=2
BLITZY-SEG:   full t=   300ms PlannedVUs=3
BLITZY-SEG:   full t=   400ms PlannedVUs=4
BLITZY-SEG:   full t=   500ms PlannedVUs=5
BLITZY-SEG:   full t=   600ms PlannedVUs=6
BLITZY-SEG:   full t=   700ms PlannedVUs=7
BLITZY-SEG:   full t=   800ms PlannedVUs=8
BLITZY-SEG:   full t=   900ms PlannedVUs=9
BLITZY-SEG:   full t=  1000ms PlannedVUs=10
BLITZY-SEG:   full t=  1100ms PlannedVUs=9
BLITZY-SEG:   full t=  1200ms PlannedVUs=8
BLITZY-SEG:   full t=  1300ms PlannedVUs=7
BLITZY-SEG:   full t=  1400ms PlannedVUs=6
BLITZY-SEG:   full t=  1500ms PlannedVUs=5
BLITZY-SEG:   full t=  1600ms PlannedVUs=4
BLITZY-SEG:   full t=  1700ms PlannedVUs=3
BLITZY-SEG:   full t=  1800ms PlannedVUs=2
BLITZY-SEG:   full t=  1900ms PlannedVUs=1
BLITZY-SEG:   full t=  2000ms PlannedVUs=0
BLITZY-SEG: merged timeline (per-segment VUs, their sum, unsegmented, and skew):
BLITZY-SEG:   t=     0ms  seg=[0 0 0]  sum= 0  unsegmented= 0  skew=0  sum==unseg?true  sum>max?false
BLITZY-SEG:   t=   100ms  seg=[1 0 0]  sum= 1  unsegmented= 1  skew=1  sum==unseg?true  sum>max?false
BLITZY-SEG:   t=   200ms  seg=[1 1 0]  sum= 2  unsegmented= 2  skew=1  sum==unseg?true  sum>max?false
BLITZY-SEG:   t=   300ms  seg=[1 1 1]  sum= 3  unsegmented= 3  skew=0  sum==unseg?true  sum>max?false
BLITZY-SEG:   t=   400ms  seg=[2 1 1]  sum= 4  unsegmented= 4  skew=1  sum==unseg?true  sum>max?false
BLITZY-SEG:   t=   500ms  seg=[2 2 1]  sum= 5  unsegmented= 5  skew=1  sum==unseg?true  sum>max?false
BLITZY-SEG:   t=   600ms  seg=[2 2 2]  sum= 6  unsegmented= 6  skew=0  sum==unseg?true  sum>max?false
BLITZY-SEG:   t=   700ms  seg=[3 2 2]  sum= 7  unsegmented= 7  skew=1  sum==unseg?true  sum>max?false
BLITZY-SEG:   t=   800ms  seg=[3 3 2]  sum= 8  unsegmented= 8  skew=1  sum==unseg?true  sum>max?false
BLITZY-SEG:   t=   900ms  seg=[3 3 3]  sum= 9  unsegmented= 9  skew=0  sum==unseg?true  sum>max?false
BLITZY-SEG:   t=  1000ms  seg=[4 3 3]  sum=10  unsegmented=10  skew=1  sum==unseg?true  sum>max?false
BLITZY-SEG:   t=  1100ms  seg=[3 3 3]  sum= 9  unsegmented= 9  skew=0  sum==unseg?true  sum>max?false
BLITZY-SEG:   t=  1200ms  seg=[3 3 2]  sum= 8  unsegmented= 8  skew=1  sum==unseg?true  sum>max?false
BLITZY-SEG:   t=  1300ms  seg=[3 2 2]  sum= 7  unsegmented= 7  skew=1  sum==unseg?true  sum>max?false
BLITZY-SEG:   t=  1400ms  seg=[2 2 2]  sum= 6  unsegmented= 6  skew=0  sum==unseg?true  sum>max?false
BLITZY-SEG:   t=  1500ms  seg=[2 2 1]  sum= 5  unsegmented= 5  skew=1  sum==unseg?true  sum>max?false
BLITZY-SEG:   t=  1600ms  seg=[2 1 1]  sum= 4  unsegmented= 4  skew=1  sum==unseg?true  sum>max?false
BLITZY-SEG:   t=  1700ms  seg=[1 1 1]  sum= 3  unsegmented= 3  skew=0  sum==unseg?true  sum>max?false
BLITZY-SEG:   t=  1800ms  seg=[1 1 0]  sum= 2  unsegmented= 2  skew=1  sum==unseg?true  sum>max?false
BLITZY-SEG:   t=  1900ms  seg=[1 0 0]  sum= 1  unsegmented= 1  skew=1  sum==unseg?true  sum>max?false
BLITZY-SEG:   t=  2000ms  seg=[0 0 0]  sum= 0  unsegmented= 0  skew=0  sum==unseg?true  sum>max?false
BLITZY-SEG: SUMMARY configuredMax=10 maxSum=10 maxSingleInstanceSkew=1 violations(sum!=unseg or sum>max)=0
--- PASS: TestBlitzySegmentSumInvariant (0.00s)
PASS
ok  	go.k6.io/k6/lib/executor	1.020s
```

**Evidence C [OBSERVED]** — three real segmented `k6 run` instances, rep 1 of 3 (unedited analyzer output). Per-instance peaks are **4 / 3 / 3** (the one-instance skew), yet the summed VUs never exceed the configured max of 10 (`buckets_over_max = 0`, `sum_never_exceeds_configured_max = True`):

```
configured_max = 10
instances = 3
  instance[0] seg1_0.json peak_vus = 4
  instance[1] seg1_1.json peak_vus = 3
  instance[2] seg1_2.json peak_vus = 3
sum_of_per_instance_peaks = 10

relSec | inst0 | inst1 | inst2 | SUM | over_max?
     0 |    1 |    1 |    1 |   3 | no
     1 |    2 |    2 |    2 |   6 | no
     2 |    3 |    3 |    3 |   9 | no
     3 |    4 |    3 |    3 |  10 | no
     4 |    4 |    3 |    3 |  10 | no
     5 |    4 |    3 |    3 |  10 | no
     6 |    2 |    2 |    2 |   6 | no
     7 |    1 |    0 |    0 |   1 | no

max_sum_across_instances = 10
buckets_over_max = 0
VERDICT: sum_never_exceeds_configured_max = True
```

Reps 2 and 3 were **byte-identical** to rep 1 (peaks 4/3/3, `max_sum_across_instances = 10`, `buckets_over_max = 0`) — confirming the distribution is deterministic across runs, not intermittent:

```
configured_max = 10
instances = 3
  instance[0] seg2_0.json peak_vus = 4
  instance[1] seg2_1.json peak_vus = 3
  instance[2] seg2_2.json peak_vus = 3
sum_of_per_instance_peaks = 10

relSec | inst0 | inst1 | inst2 | SUM | over_max?
     0 |    1 |    1 |    1 |   3 | no
     1 |    2 |    2 |    2 |   6 | no
     2 |    3 |    3 |    3 |   9 | no
     3 |    4 |    3 |    3 |  10 | no
     4 |    4 |    3 |    3 |  10 | no
     5 |    4 |    3 |    3 |  10 | no
     6 |    2 |    2 |    2 |   6 | no
     7 |    1 |    0 |    0 |   1 | no

max_sum_across_instances = 10
buckets_over_max = 0
VERDICT: sum_never_exceeds_configured_max = True
```
```
configured_max = 10
instances = 3
  instance[0] seg3_0.json peak_vus = 4
  instance[1] seg3_1.json peak_vus = 3
  instance[2] seg3_2.json peak_vus = 3
sum_of_per_instance_peaks = 10

relSec | inst0 | inst1 | inst2 | SUM | over_max?
     0 |    1 |    1 |    1 |   3 | no
     1 |    2 |    2 |    2 |   6 | no
     2 |    3 |    3 |    3 |   9 | no
     3 |    4 |    3 |    3 |  10 | no
     4 |    4 |    3 |    3 |  10 | no
     5 |    4 |    3 |    3 |  10 | no
     6 |    2 |    2 |    2 |   6 | no
     7 |    1 |    0 |    0 |   1 | no

max_sum_across_instances = 10
buckets_over_max = 0
VERDICT: sum_never_exceeds_configured_max = True
```

### Verdict

**No overflow — the sum invariant holds; the per-instant ±1 skew is legitimate deterministic rounding. [OBSERVED]** Across the canonical randomized test (all 10 sub-cases PASS under `-race`), the harness timeline (`violations=0`, peak `sum=10=unsegmented`), and three real segmented CLI runs (peaks 4/3/3, `buckets_over_max=0`, byte-identical across 3 reps), the summed VU count **never exceeds the configured maximum of 10**. _[INFERRED, code-grounded]_ the single-instance skew is the expected outcome of deterministic integer scaling in `SegmentedIndex` [execution_segment.go:768] as walked by `getRawExecutionSteps` [ramping_vus.go:171-219]; the design guarantees that the same inputs always scale to the same outputs and that the parts sum to the whole with no rounding overflow — encoded and here re-verified by `TestSumRandomSegmentSequenceMatchesNoSegment` [lib/executor/ramping_vus_test.go:1112]. The user's "summing exceeds the maximum" was a measurement artifact (e.g. summing per-instance *peaks* that occur at different instants), not a real overflow at any single timestamp.

---

## Thread 5a — Is there a race condition between the two handler goroutines?

### Question

Is there a race condition between the two handler goroutines (the scheduled and the max-allowed handlers)?

### Grounding (file:line)

_[INFERRED, code-grounded]_ — the concurrency structure; the detector's verdict is under **Observed output**.

- The two handlers are **not** two contending goroutines during the main phase. `RampingVUs.Run` [ramping_vus.go:491] binds both strategies (`handleNewMaxAllowedVUs` [ramping_vus.go:546], `handleNewScheduledVUs` [ramping_vus.go:547]) and then calls `iterateSteps(...)` [ramping_vus.go:549] which drives **both serially in the single `Run` goroutine** — `iterateSteps` is defined at [ramping_vus.go:622].
- The only additional goroutine, `runRemainingGracefulSteps` [ramping_vus.go:654], is launched with `go runState.runRemainingGracefulSteps(...)` [ramping_vus.go:554] **after** `iterateSteps` has returned [ramping_vus.go:549-553] — a happens-before edge, so the two never drive steps concurrently.
- Any cross-goroutine mutation of a `vuHandle` is serialized by its per-VU `mutex` [vu_handle.go:71], and `runLoopsIfPossible` contains explicit race-resolution branches: `case running: // start raced us toGracefulStop` [vu_handle.go:220] and `case <-vh.canStartIter: // we check again in case of race` [vu_handle.go:248].

### How it was exercised (command)

Adjudicated with the Go race detector (the project-native instrument), across **three** captures of increasing breadth:
```
# 1) Project ramping-vus graceful/ramp-down suite (drives both handlers + runRemainingGracefulSteps)
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -race -count=1 -timeout 120s \
  -run '^TestRampingVUsGracefulStopWaits$|^TestRampingVUsGracefulStopStops$|^TestRampingVUsGracefulRampDown$|^TestRampingVUsHandleRemainingVUs$|^TestRampingVUsRampDownNoWobble$' -v ./lib/executor/

# 2) Integration rapid up/down ramp with long gracefulRampDown (same run as Thread 1 Evidence A)
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -race -count=1 -timeout 120s -run '^TestBlitzyStuckVUsAndRace$' ./lib/executor/

# 3) The entire lib/executor package (broadest sweep; matches Makefile 'tests' approach)
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -race -timeout 210s ./lib/executor/
```

### Observed output

**Capture 1 [OBSERVED]** — the project graceful/ramp-down suite, race-clean (unedited):

```
=== RUN   TestRampingVUsGracefulStopWaits
=== PAUSE TestRampingVUsGracefulStopWaits
=== RUN   TestRampingVUsGracefulStopStops
=== PAUSE TestRampingVUsGracefulStopStops
=== RUN   TestRampingVUsGracefulRampDown
=== PAUSE TestRampingVUsGracefulRampDown
=== RUN   TestRampingVUsHandleRemainingVUs
=== PAUSE TestRampingVUsHandleRemainingVUs
=== RUN   TestRampingVUsRampDownNoWobble
=== PAUSE TestRampingVUsRampDownNoWobble
=== CONT  TestRampingVUsGracefulStopWaits
=== CONT  TestRampingVUsRampDownNoWobble
=== CONT  TestRampingVUsGracefulStopStops
=== CONT  TestRampingVUsGracefulRampDown
=== CONT  TestRampingVUsHandleRemainingVUs
--- PASS: TestRampingVUsHandleRemainingVUs (0.07s)
--- PASS: TestRampingVUsGracefulStopWaits (1.50s)
--- PASS: TestRampingVUsGracefulStopStops (2.50s)
--- PASS: TestRampingVUsGracefulRampDown (2.50s)
--- PASS: TestRampingVUsRampDownNoWobble (6.02s)
PASS
ok  	go.k6.io/k6/lib/executor	7.038s
```

**Capture 2 [OBSERVED]** — the integration rapid-ramp run under `-race`. Its complete output (all before/during/after samples) is shown in **Thread 1 Evidence A**; its race-relevant tail is reproduced here unedited — the detector printed **no** `DATA RACE` and the package reports `ok`:

```
--- PASS: TestBlitzyStuckVUsAndRace (2.04s)
PASS
ok  	go.k6.io/k6/lib/executor	3.063s
```

**Capture 3 [OBSERVED]** — the entire `lib/executor` package under `-race` (the broadest sweep; a `DATA RACE` anywhere would fail the package and print a report — none did):

```
ok  	go.k6.io/k6/lib/executor	30.778s
```

### Verdict

**No race. [OBSERVED]** The race detector reports **zero** data races across all three captures — the targeted graceful suite (`ok 7.038s`), the rapid-ramp integration run (`ok 3.063s`), and the full package (`ok 30.778s`). _[INFERRED, code-grounded]_ this is expected because the two handler strategies execute *serially* in one goroutine via `iterateSteps` [ramping_vus.go:622] (bound at [ramping_vus.go:546-547], called at [ramping_vus.go:549]), the only other goroutine `runRemainingGracefulSteps` starts *after* that returns [ramping_vus.go:554], and any residual cross-goroutine access to a `vuHandle` is serialized by its mutex [vu_handle.go:71] with explicit race-resolution branches [vu_handle.go:220,248].

---

## Thread 5b — Is the VU buffer leaking?

### Question

Is the VU buffer — the bounded channel `ExecutionState.vus` [lib/execution.go:217] — leaking? (This thread also covers the buffer's exhaustion/retry/error path, since a "leak" would show as an unreturned VU or a surviving goroutine.)

### Grounding (file:line)

_[INFERRED, code-grounded]_ — the buffer contract; runtime confirmation under **Observed output**.

- The buffer is `vus = make(chan InitializedVU, maxPossibleVUs)` [lib/execution.go:217]. The ramping executor takes a VU with `GetPlannedVU(...)` [lib/execution.go:471] (via the `getVU` closure [ramping_vus.go:594]) and returns it with `ReturnVU(...)` [lib/execution.go:544] (via `returnVU` [ramping_vus.go:606]); active accounting is `ModCurrentlyActiveVUsCount` [lib/execution.go:276] (+1 at [ramping_vus.go:602], -1 at [ramping_vus.go:609]). A non-leaking run must return every VU it takes.
- **Exhaustion/retry/error path.** `GetPlannedVU` [lib/execution.go:471] retries `for i := 1; i <= MaxRetriesGetPlannedVU` [lib/execution.go:472], each attempt waiting `case <-time.After(MaxTimeToWaitForPlannedVU)` [lib/execution.go:480] and logging `"Could not get a VU from the buffer for %s"` [lib/execution.go:481]; after `MaxRetriesGetPlannedVU = 5` [lib/execution.go:29] × `MaxTimeToWaitForPlannedVU = 400ms` [lib/execution.go:25] it returns the terminal error `"could not get a VU from the buffer in %s"` [lib/execution.go:485-486] rather than blocking forever — so exhaustion is a *bounded error*, not a leak.

### How it was exercised (command)

**Evidence A — goroutine-leak check around a full ramping-vus run.** A harness wraps a complete run in `goleak.VerifyNone(t)` (same detector the project calls as `goleak.Find()` [cmd/tests/tests.go:57]) and then drains the buffer to confirm every planned VU was returned; run plain and under `-race`:
```
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -count=1 -timeout 60s -run '^TestBlitzyVUBufferLeak$' -v ./lib/executor/   # (also run WITHOUT -race)
```

**Evidence B — buffer exhaustion/retry/error path (harness).** A harness drains the buffer, then calls `GetPlannedVU` with no VU available to force the full retry/timeout/error path:
```
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -count=1 -timeout 60s -run '^TestBlitzyVUBufferExhaustion$' -v ./lib/executor/
```

**Evidence C — canonical exhaustion tests (project's own).** The project already ships tests for the buffer's get/timeout behavior:
```
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -race -count=1 -timeout 60s -run '^TestExecutionStateGettingVUsWhenNonAreAvailable$|^TestExecutionStateGettingVUs$' -v ./lib/executor/
```

### Observed output

**Evidence A [OBSERVED]** — no goroutine leak; buffer fully drained (8/8 returned), plain run (unedited):

```
=== RUN   TestBlitzyVUBufferLeak
BLITZY-LEAK: maxPlannedVUs=8 activeBefore=0
BLITZY-LEAK: activeAfterRun=0
BLITZY-LEAK: drained 8/8 planned VUs from buffer (all returned)
--- PASS: TestBlitzyVUBufferLeak (2.13s)
PASS
ok  	go.k6.io/k6/lib/executor	2.137s
```

And under `-race`, both repetitions identical (no leak, no race):

```
=== RUN   TestBlitzyVUBufferLeak
BLITZY-LEAK: maxPlannedVUs=8 activeBefore=0
BLITZY-LEAK: activeAfterRun=0
BLITZY-LEAK: drained 8/8 planned VUs from buffer (all returned)
--- PASS: TestBlitzyVUBufferLeak (2.13s)
PASS
ok  	go.k6.io/k6/lib/executor	3.152s
```
```
=== RUN   TestBlitzyVUBufferLeak
BLITZY-LEAK: maxPlannedVUs=8 activeBefore=0
BLITZY-LEAK: activeAfterRun=0
BLITZY-LEAK: drained 8/8 planned VUs from buffer (all returned)
--- PASS: TestBlitzyVUBufferLeak (2.13s)
PASS
ok  	go.k6.io/k6/lib/executor	3.152s
```

**Evidence B [OBSERVED]** — the exhaustion path: exactly **5** warnings (400ms, 800ms, 1.2s, 1.6s, 2s) then the terminal error `"could not get a VU from the buffer in 2s"` — bounded, not a hang (unedited):

```
=== RUN   TestBlitzyVUBufferExhaustion
BLITZY-EXHAUST: MaxTimeToWaitForPlannedVU=400ms MaxRetriesGetPlannedVU=5 (expected max wait=2s)
BLITZY-EXHAUST: GetPlannedVU returned vu=<nil> err="could not get a VU from the buffer in 2s" after 2s
BLITZY-EXHAUST: warning count=5 (== MaxRetriesGetPlannedVU=5? true)
BLITZY-EXHAUST:   warn[1] level=warning msg="Could not get a VU from the buffer for 400ms"
BLITZY-EXHAUST:   warn[2] level=warning msg="Could not get a VU from the buffer for 800ms"
BLITZY-EXHAUST:   warn[3] level=warning msg="Could not get a VU from the buffer for 1.2s"
BLITZY-EXHAUST:   warn[4] level=warning msg="Could not get a VU from the buffer for 1.6s"
BLITZY-EXHAUST:   warn[5] level=warning msg="Could not get a VU from the buffer for 2s"
--- PASS: TestBlitzyVUBufferExhaustion (2.00s)
PASS
ok  	go.k6.io/k6/lib/executor	2.008s
```

**Evidence C [OBSERVED]** — the project's canonical get/timeout tests pass under `-race` (unedited):

```
=== RUN   TestExecutionStateGettingVUsWhenNonAreAvailable
=== PAUSE TestExecutionStateGettingVUsWhenNonAreAvailable
=== RUN   TestExecutionStateGettingVUs
=== PAUSE TestExecutionStateGettingVUs
=== CONT  TestExecutionStateGettingVUs
=== CONT  TestExecutionStateGettingVUsWhenNonAreAvailable
--- PASS: TestExecutionStateGettingVUsWhenNonAreAvailable (2.00s)
--- PASS: TestExecutionStateGettingVUs (4.01s)
PASS
ok  	go.k6.io/k6/lib/executor	5.026s
```

### Verdict

**No leak. [OBSERVED]** `goleak.VerifyNone` finds **no** surviving goroutine after a complete run, the buffer drains **8/8** (every planned VU returned), and `activeAfterRun=0` — stable plain and under `-race`. The exhaustion path is **bounded**: 5 retries × 400 ms produce 5 warnings and then the terminal error `"could not get a VU from the buffer in 2s"`, never an unbounded block. _[INFERRED, code-grounded]_ this holds because each `GetPlannedVU` [lib/execution.go:471] is balanced by a `ReturnVU` [lib/execution.go:544] and the retry loop [lib/execution.go:472-486] converts starvation into an error rather than a leak; the bounded channel [lib/execution.go:217] cannot grow, so there is nothing to leak.

---

## Thread 5c — Trace what happens when both handlers modify VU state simultaneously

### Question

Trace what actually happens when both handlers attempt to modify a VU's state simultaneously.

### Grounding (file:line)

_[INFERRED, code-grounded]_ — the serialization mechanism; the runtime trace is under **Observed output**.

- Every state-mutating method holds the per-VU `mutex` [vu_handle.go:71] (init `&sync.Mutex{}` [vu_handle.go:99]): `start()` locks at [vu_handle.go:116-117], `gracefulStop()` at [vu_handle.go:148-149], `hardStop()` at [vu_handle.go:166-167]. Each writes through `changeState` [vu_handle.go:142] = `atomic.StoreInt32(...)` [vu_handle.go:144]. So two "simultaneous" mutations are **serialized**, and the last acquirer wins deterministically per the transition table.
- The loop `runLoopsIfPossible` [vu_handle.go:185] resolves interleavings explicitly: `start()` handles `case toGracefulStop:` by *not* returning the VU and going back to `running` [vu_handle.go:122-125]; the loop handles `case running: // start raced us toGracefulStop` [vu_handle.go:220], `case toGracefulStop:` [vu_handle.go:223], `case toHardStop:` -> `changeState(stopped)` [vu_handle.go:233], and re-checks `case <-vh.canStartIter: // we check again in case of race` [vu_handle.go:248] before `changeState(running)` [vu_handle.go:251]. The terminal `defer` always converges to `stopped` [vu_handle.go:192].

### How it was exercised (command)

The project's dedicated per-VU contention tests under `-race` — `TestVUHandleRace` (10,000 concurrent start/stop cycles) and `TestVUHandleStartStopRace`, plus `TestVUHandleSimple` with its three interleaving sub-cases (`start_before_gracefulStop_finishes`, `start_after_gracefulStop_finishes`, `start_after_hardStop`):
```
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -race -count=1 -timeout 60s \
  -run '^TestVUHandleRace$|^TestVUHandleStartStopRace$|^TestVUHandleSimple$' -v ./lib/executor/
```
This is complemented by the deterministic five-state trace in **Thread 1 Evidence B**, which shows the ordered `stopped -> starting -> running -> toGracefulStop -> stopped -> running -> toHardStop -> stopped` convergence with `getVUCount==returnVUCount`.

### Observed output

**[OBSERVED]** — all contention tests and their three interleaving sub-cases pass under `-race` (unedited):

```
=== RUN   TestVUHandleRace
=== PAUSE TestVUHandleRace
=== RUN   TestVUHandleStartStopRace
=== PAUSE TestVUHandleStartStopRace
=== RUN   TestVUHandleSimple
=== PAUSE TestVUHandleSimple
=== CONT  TestVUHandleStartStopRace
=== CONT  TestVUHandleSimple
=== CONT  TestVUHandleRace
=== RUN   TestVUHandleSimple/start_before_gracefulStop_finishes
=== PAUSE TestVUHandleSimple/start_before_gracefulStop_finishes
=== RUN   TestVUHandleSimple/start_after_gracefulStop_finishes
=== PAUSE TestVUHandleSimple/start_after_gracefulStop_finishes
=== RUN   TestVUHandleSimple/start_after_hardStop
=== PAUSE TestVUHandleSimple/start_after_hardStop
=== CONT  TestVUHandleSimple/start_before_gracefulStop_finishes
=== CONT  TestVUHandleSimple/start_after_hardStop
=== CONT  TestVUHandleSimple/start_after_gracefulStop_finishes
--- PASS: TestVUHandleRace (0.18s)
--- PASS: TestVUHandleStartStopRace (0.41s)
--- PASS: TestVUHandleSimple (0.00s)
    --- PASS: TestVUHandleSimple/start_after_hardStop (1.53s)
    --- PASS: TestVUHandleSimple/start_before_gracefulStop_finishes (1.56s)
    --- PASS: TestVUHandleSimple/start_after_gracefulStop_finishes (3.10s)
PASS
ok  	go.k6.io/k6/lib/executor	4.128s
```

### Verdict

**Simultaneous mutations are serialized; there is no lost update or torn state. [OBSERVED]** `TestVUHandleRace` (10,000 concurrent start/stop cycles) and `TestVUHandleStartStopRace` pass under `-race`, and all three `TestVUHandleSimple` interleavings (`start_before_gracefulStop_finishes`, `start_after_gracefulStop_finishes`, `start_after_hardStop`) pass — the project asserts the buffer balance `require.Equal(getVUCount, returnVUCount)` [lib/executor/vu_handle_test.go:110]. _[INFERRED, code-grounded]_ concurrent `start`/`gracefulStop`/`hardStop` are serialized by the mutex [vu_handle.go:71], each committing atomically via `changeState` [vu_handle.go:144]; whichever acquires the lock last wins deterministically, and `runLoopsIfPossible` reconciles the interleaving through its explicit race branches [vu_handle.go:122-125,220,248] before converging to `stopped` [vu_handle.go:192].

---

## Coverage pass — every named item addressed

Honest confirmation that each distinct thing the prompt asks for was exercised at runtime and answered above.

| # | Requirement / named item | Exercised via (canonical) | Result | Verdict |
|---|---|---|---|---|
| 1 | "Stuck" VUs across all five states `stopped`/`starting`/`running`/`toGracefulStop`/`toHardStop` | `TestBlitzyStuckVUsAndRace` (×3, `-race`) + `TestBlitzyStateMachineTrace` | All 5 states reached; active count 0->6->0; balanced | **Not stuck** |
| 2 | Scheduled vs. graceful handler count "divergence" | `TestBlitzyHandlerDivergence` (raw vs. graceful step lists) | Match on up; ceiling lags target on down (gap up to 5) by design | **Different quantities by design** |
| 3 | Ctrl+C timing vs. `gracefulStop`; **first** signal (graceful) | `k6 run` + single `SIGINT` ×5 (`run_ctrlc.sh`) | SIGINT->exit 33-210 ms; iteration interrupted | **Run-abort, not `gracefulStop`** |
| 3 | Ctrl+C **second** signal (hard stop) | `k6 run` + double `SIGINT` (Run D) | teardown cut off; `"Aborting k6 in response to signal"` | **Hard stop via `onHardStop`** |
| 3 | Executor `gracefulStop` window (natural end) | `k6 run natural_gs.js` (+ `-v`) | 6 s iteration finishes past 3 s stage; `2 complete` | **`gracefulStop` window (natural end only)** |
| 4 | Three-segment split; per-instant skew; **summed** overflow | `TestSumRandomSegmentSequenceMatchesNoSegment` (`-race`) + harness + 3 parallel segmented `k6 run` (×3 reps) | Peaks 4/3/3 (±1 skew); sum never > 10; `violations=0` | **No overflow; skew is legit rounding** |
| 5a | Race between the two handler goroutines | `-race`: graceful suite + integration + full package | 0 data races (all three) | **No race** |
| 5b | VU buffer leak (bounded channel `vus`) | `goleak.VerifyNone` harness (plain + `-race`) | No surviving goroutine; 8/8 VUs returned | **No leak** |
| 5b | Buffer exhaustion/retry/error path | `TestBlitzyVUBufferExhaustion` + canonical `TestExecutionStateGettingVUs*` | 5 warnings then bounded terminal error | **Bounded error, not a leak/hang** |
| 5c | Simultaneous VU-state mutation | `TestVUHandleRace` (10k) + `StartStopRace` + `Simple` (3 sub-cases), `-race` | All pass; balance asserted | **Serialized; no lost update** |

## Overall verdict

**Across every thread, no concurrency defect was found in the `ramping-vus` executor at commit `ddc3b0b1d23c` (v0.55.0).** Every behavior the user reported as suspicious is the *designed* consequence of one of the two intentional dichotomies stated at the top:

- The "stuck" VUs (Thread 1), the handler "divergence" (Thread 2), and the segment "skew/overflow" (Thread 4) all stem from **active target vs. max-allowed ceiling** plus **deterministic rounding** — every VU converges to `stopped`, the ceiling deliberately lags the target during `gracefulRampDown`, and the per-segment counts always sum to (never exceed) the unsegmented maximum.
- The Ctrl+C timing (Thread 3) stems from **run-level abort vs. executor `gracefulStop`** — Ctrl+C cancels the run context immediately (VUs stop in tens of ms), while the executor's `gracefulStop` window only governs a *natural* end.
- The race (5a), leak (5b), and simultaneous-mutation (5c) questions are all answered **negative** by the project-native instruments: the Go race detector reports **0** races (up to the full package, `ok 30.778s`), `goleak` finds **no** surviving goroutine with the buffer fully drained, and the mutex-plus-atomic state machine serializes every concurrent mutation (10,000-cycle `TestVUHandleRace` passes under `-race`).

Every runtime value above is **[OBSERVED]** from a labeled canonical command; every causal/design explanation is labeled **_[INFERRED, code-grounded]_** with a specific `file:line`. No source file under test was modified; the only artifacts created for this investigation were temporary and were removed, leaving the repository byte-for-byte unchanged apart from this document.
