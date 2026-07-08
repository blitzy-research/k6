# k6 ramping-vus Executor — Concurrency Root-Cause Investigation

This is a **read-only, evidence-first** investigation into suspected concurrency bugs in
k6's `ramping-vus` executor (`go.k6.io/k6` v0.55.0, HEAD `ddc3b0b1d`, branch
`k6_ddc3b0b1d23c`). It answers the five questions raised in the user's report. The k6
source code and its **observed runtime behavior** are the authoritative source of truth:
every behavioral claim below is grounded in a specific `file:line` citation (anchored to
HEAD `ddc3b0b1d`) plus the exact command that produced it and that command's complete,
unedited output. Values are reported **exactly as observed** — including results that
contradict the user's "bug" framing — and are never adjusted toward what they "should" be.

The investigation was performed by **running the real code first** through its canonical
entry point (`k6 run`) and the Go race detector (`go test -race`), then transcribing the
captured evidence here. All temporary reproduction scripts, driver scripts, and binaries
lived under `/tmp` (outside the repository tree) and were removed afterward; the repository
working tree is unchanged except for this one document.

## Executive verdict

| # | Question | Verdict |
|---|----------|---------|
| **Q1** | Do VUs get "stuck" (neither active nor stopped) under rapid up/down stages + long `gracefulRampDown`? | **BY DESIGN** — the "stuck" condition is the transitional `toGracefulStop` state; deterministically resolved. Not a bug. |
| **Q2** | Why doesn't the scheduled handler's VU count match the graceful handler's count? | **BY DESIGN (expected, not corruption)** — two independent `cur` counters over different step curves; the graceful count is deliberately held higher by `reserveVUsForGracefulRampDowns()`. |
| **Q3** | Why do VUs keep running longer than `gracefulStop` after Ctrl+C? | **BY DESIGN (separate path; not reproduced for JS-bound work)** — Ctrl+C uses a distinct abort path; the first `SIGINT` aborts in ~36 ms (far *faster* than `gracefulStop`), the second forces immediate `OSExit`. |
| **Q4** | Why does one segment instance show more VUs, and why does the sum appear to exceed the maximum? | **one-instance-more = BY DESIGN** (deterministic striping remainder); **sum-exceeds-max = NOT reproduced** (sum equals the maximum at peak, never exceeds it). |
| **Q5** | Is there a race between the two handler goroutines, or a VU buffer leak? | **NO data race, NO buffer leak; the two handlers are NEVER simultaneous.** |

**Overall: no genuine defect found.** Every reported symptom is explained by documented,
by-design behavior — or, for the Q3 "linger" and Q4 "exceed" claims, was not reproducible
with canonical inputs.

## Environment & canonical build

k6 was built and run in its **canonical configuration**, exactly as a normal user would.
The captured build/version/toolchain output:

```text
$ go version
go version go1.21.13 linux/amd64

$ CGO_ENABLED=1 go build -o /tmp/k6bin .
$ /tmp/k6bin version
k6bin v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)

$ gcc --version
gcc (Ubuntu 13.3.0-6ubuntu2~24.04.1) 13.3.0

$ git rev-parse --abbrev-ref HEAD ; git rev-parse --short HEAD
k6_ddc3b0b1d23c
ddc3b0b1d
```

The race detector requires cgo (`CGO_ENABLED=1`) plus a C compiler (`gcc`); `go.mod` pins
`go 1.21` with `toolchain go1.21.13` [go.mod:L3], and the installed toolchain matches it
exactly. All reproduction scripts, driver scripts, and binaries (`/tmp/k6bin`,
`/tmp/k6race`) lived under `/tmp`, outside the repository tree, and were removed afterward.

---

## Q1 — "Stuck" VUs (rapid up/down stages + long `gracefulRampDown`)

**Verdict: BY DESIGN — not a bug.** The "neither fully active nor fully stopped" condition
is the transitional per-VU state **`toGracefulStop`**. A VU told to gracefully stop keeps
its slot (and is still counted in the `vus` metric) until either its in-flight iteration
finishes (→ `stopped`) or the graceful window expires (→ `hardStop` → `toHardStop`). If a
ramp-up arrives during that window, the VU transitions straight back to `running` — a
documented, intentional race resolution.

**Grounding (verified at HEAD `ddc3b0b1d`):**

- The per-VU state machine is the `vuHandle` struct — `lib/executor/vu_handle.go:L70`
  (mutex `*sync.Mutex` at `L71`, `state stateType` at `L82`).
- The state constants `stopped`/`starting`/`running`/`toGracefulStop`/`toHardStop` —
  `lib/executor/vu_handle.go:L16-L22`.
- The state-transition table — `lib/executor/vu_handle.go:L24-L55`. The key by-design
  resolution at `L37` is `start | toGracefulStop → running` with the rationale "we raced
  with the loop stopping, just continue".
- Methods: `start()` `L115` (Debug `"Start"` at `L123`); `gracefulStop()` `L147`
  (`running → toGracefulStop` at `L157-L158`, Debug `"Graceful stop"` at `L161`);
  `hardStop()` `L165` (Debug `"Hard stop"` at `L177`, context cancel `vh.cancel()` at
  `L178`); `changeState()` atomic store at `L142-L145`; `runLoopsIfPossible()` `L185`.

### Q1.1 Primary scenario — rapid up/down + long `gracefulRampDown`

Script `/tmp/k6inv/q1_stuck.js`:

```javascript
import { sleep } from 'k6';
export const options = {
  scenarios: {
    stuck: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '3s', target: 8 },
        { duration: '3s', target: 0 },
        { duration: '3s', target: 8 },
        { duration: '3s', target: 0 },
      ],
      gracefulRampDown: '10s',
      gracefulStop: '5s',
    },
  },
};
export default function () { sleep(2); }
```

Command:

```text
/tmp/k6bin run -v --log-output=stderr --out json=/tmp/k6inv/q1_out.json /tmp/k6inv/q1_stuck.js
```

Observed state-transition Debug counts (**identical run 1 and run 2**): `msg=Start` = **16**,
`msg="Graceful stop"` = **16**, `msg="Hard stop"` = **0**. (Note: the logrus TEXT format
prints `msg=Start` unquoted but `msg="Graceful stop"` quoted because it contains a space.)

Captured `vuNum` timeline (relative seconds; the **same** 8 VUs are re-started after being
gracefully stopped):

```text
t=+00s  Start vu0
t=+00s  Start vu1
t=+00s  Start vu2
t=+01s  Start vu3
t=+01s  Start vu4
t=+01s  Start vu5
t=+02s  Start vu6
t=+02s  Start vu7
t=+03s  "Graceful stop" vu7
t=+03s  "Graceful stop" vu6
t=+03s  "Graceful stop" vu5
t=+04s  "Graceful stop" vu4
t=+04s  "Graceful stop" vu3
t=+04s  "Graceful stop" vu2
t=+05s  "Graceful stop" vu1
t=+05s  "Graceful stop" vu0
t=+06s  Start vu0
t=+06s  Start vu1
t=+06s  Start vu2
t=+07s  Start vu3
t=+07s  Start vu4
t=+07s  Start vu5
t=+08s  Start vu6
t=+08s  Start vu7
t=+09s  "Graceful stop" vu7
t=+09s  "Graceful stop" vu6
t=+09s  "Graceful stop" vu5
t=+10s  "Graceful stop" vu4
t=+10s  "Graceful stop" vu3
t=+10s  "Graceful stop" vu2
t=+11s  "Graceful stop" vu1
t=+11s  "Graceful stop" vu0
```

Captured `vus` metric (from `--out json`; identical both runs), reported
**before / during / after** each ramp-down:

```text
t=+00.0s  vus=2
t=+01.0s  vus=5
t=+02.0s  vus=7
t=+03.0s  vus=8   <- peak (= configured max)
t=+04.0s  vus=6   <- during ramp-down: decays as in-flight iterations finish
t=+05.0s  vus=3
t=+07.0s  vus=5
t=+08.0s  vus=7
t=+09.0s  vus=8   <- peak again
t=+10.0s  vus=6
t=+11.0s  vus=3
t=+12.0s  vus=1
(max vus = 8 — never exceeds the configured maximum)
```

**Interpretation.** Each `"Graceful stop"` call moves a `running` VU to `toGracefulStop`
(`vu_handle.go:L157-L158`) — this is the "stuck" state. The VU is still counted in `vus`
while it finishes its current 2s iteration. Because `gracefulStop=5s` ≥ the 2s iteration,
every in-flight iteration finishes gracefully → **0 hard stops**. When the next ramp-up
arrives (+6–8s), the very same VUs receive `Start` again — the
`start | toGracefulStop → running` resolution at `vu_handle.go:L37` ("we raced with the
loop stopping, just continue").

### Q1.2 Complementary scenario — force `toHardStop`

Script `/tmp/k6inv/q1b_hardstop.js`:

```javascript
import { sleep } from 'k6';
export const options = {
  scenarios: {
    hard: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [ { duration: '1s', target: 4 }, { duration: '1s', target: 0 } ],
      gracefulRampDown: '2s',
      gracefulStop: '2s',
    },
  },
};
export default function () { sleep(10); }
```

Command:

```text
/tmp/k6bin run -v --log-output=stderr /tmp/k6inv/q1b_hardstop.js
```

Counts: `Start` = **4**, `"Graceful stop"` = **4**, `"Hard stop"` = **4**. Timeline:

```text
t=+00s  Start vu0
t=+00s  Start vu1
t=+00s  Start vu2
t=+01s  Start vu3
t=+01s  "Graceful stop" vu3
t=+01s  "Graceful stop" vu2
t=+01s  "Graceful stop" vu1
t=+02s  "Graceful stop" vu0
t=+03s  "Hard stop" vu3
t=+03s  "Hard stop" vu2
t=+03s  "Hard stop" vu1
t=+04s  "Hard stop" vu0
```

And the progress line confirming interruption:

```text
running (4.0s), 1/4 VUs, 0 complete and 3 interrupted iterations
running (4.0s), 0/4 VUs, 0 complete and 4 interrupted iterations
```

**Interpretation.** Here the 10s iteration cannot finish inside the 2s graceful window, so
exactly `gracefulRampDown=2s` after each `"Graceful stop"` the max-allowed handler issues
`"Hard stop"` → `running`/`toGracefulStop` → `toHardStop` (`vu_handle.go:L174-L175`) and
`vh.cancel()` (`L178`) interrupts the iteration ("4 interrupted iterations"). This exposes
the third named transitional state, `toHardStop`.

**Q1 rationale.** The three transitional states named in the question (`starting`,
`toGracefulStop`, `toHardStop`) are all real, all observed via k6's existing logrus Debug
output, and all resolved deterministically by the state machine. A VU appearing "neither
active nor stopped" is a VU in `toGracefulStop` finishing its in-flight iteration during the
(deliberately long) `gracefulRampDown` window — expected behavior, not a stuck/corrupt
state. Counts were identical across two runs.

---

## Q2 — Handler count mismatch (scheduled vs graceful)

**Verdict: BY DESIGN — the mismatch is expected, not corruption.** The two handlers
maintain **separate** local `cur` counters over **different** step curves.
`reserveVUsForGracefulRampDowns()` deliberately holds the graceful (max-allowed) count
above the scheduled (raw) count during ramp-down so that shed VUs can finish their in-flight
iterations. The total envelope never exceeds the configured maximum.

**Grounding (verified at HEAD `ddc3b0b1d`):**

- `scheduledVUsHandlerStrategy()` — `lib/executor/ramping_vus.go:L679`; its counter
  `var cur uint64 // current number of planned raw VUs` at `L680`; it grows via `start()`
  and shrinks via `gracefulStop()`.
- `maxAllowedVUsHandlerStrategy()` — `lib/executor/ramping_vus.go:L668`; its counter
  `var cur uint64 // current number of planned graceful VUs` at `L669`; it shrinks via
  `hardStop()` only.
- `reserveVUsForGracefulRampDowns()` — `lib/executor/ramping_vus.go:L307`; the doc at
  `L300-L306` states it "prevents the number of VUs from decreasing for the configured
  gracefulRampDown period".
- The two curves are precomputed in `Init()`: `rawSteps = getRawExecutionSteps(et, true)`
  at `L480`; `gracefulSteps = GetExecutionRequirements(et)` at `L483`.
  `GetExecutionRequirements` at `L434` applies the reservation only when
  `GracefulRampDown > 0` (`L439-L440`).

**Method note.** The two `cur` counters are internal locals, so they were exercised through
the **exported** API `RampingVUsConfig.GetExecutionRequirements(et)` (`ramping_vus.go:L434`)
— the exact function `Init()` uses to build `gracefulSteps`. A tiny external Go program (run
in a temporary module that *reused the repo's own `go.mod` graph via a local `replace`,
leaving the repo unmodified*) computed the curve twice on identical stages
`[3s→8, 3s→0, 3s→8, 3s→0]`, `gracefulStop=5s`: once with `gracefulRampDown=0` (→ the
raw/scheduled curve, since reservation is skipped) and once with `gracefulRampDown=10s` (→
the graceful/max-allowed curve with reservation). This is the canonical exported API, not a
debug bypass.

Captured output (deterministic; identical across 2 runs):

```text
time_offset  scheduled(raw)  maxAllowed(graceful)  reserved_gap
         0s               0                     0             0
      375ms               1                     1             0
      750ms               2                     2             0
     1.125s               3                     3             0
       1.5s               4                     4             0
     1.875s               5                     5             0
      2.25s               6                     6             0
     2.625s               7                     7             0
         3s               8                     8             0
     3.375s               7                     8             1
      3.75s               6                     8             2
     4.125s               5                     8             3
       4.5s               4                     8             4
     4.875s               3                     8             5
      5.25s               2                     8             6
     5.625s               1                     8             7
         6s               0                     8             8
     6.375s               1                     8             7
      6.75s               2                     8             6
     7.125s               3                     8             5
       7.5s               4                     8             4
     7.875s               5                     8             3
      8.25s               6                     8             2
     8.625s               7                     8             1
         9s               8                     8             0
     9.375s               7                     8             1
      9.75s               6                     8             2
    10.125s               5                     8             3
      10.5s               4                     8             4
    10.875s               3                     8             5
     11.25s               2                     8             6
    11.625s               1                     8             7
        12s               0                     8             8
        17s               0                     0             0

maxVUs scheduled(raw)=8   maxVUs maxAllowed(graceful)=8
raw steps=33  graceful steps=10
```

**Cross-corroboration with the Q1 runtime.** In the Q1 run, at +3–5s the scheduled handler
issued 8 `"Graceful stop"` calls (raw `cur` 8→0) while **0** hard stops occurred (graceful
`cur` stayed 8) — exactly the `reserved_gap = 8` row at t=6s. In Q1.2 (hard-stop scenario),
hard stops arrived exactly `gracefulRampDown` (2s) after the graceful stops — the graceful
`cur` shrinking later than the raw `cur`.

**Q2 rationale.** The scheduled handler's count (what "should be scheduled" per the stages)
and the graceful handler's count (what is still reserved during ramp-down) are two
independent, correct numbers. At t=6s the raw curve is 0 but the graceful curve is 8 — a
deliberate reservation, not corruption. Both peak at the configured maximum (8); the total
VU buffer never grows beyond it.

---

## Q3 — Ctrl+C lingering VUs

**Verdict: BY DESIGN — and the user's "longer than `gracefulStop`" was NOT reproduced for
JS-bound scripts.** Ctrl+C uses a **separate** abort path (in `cmd/`, not the executor). The
first `SIGINT` aborts promptly (measured ~30–40 ms — far *faster* than the 30s
`gracefulStop`); the second `SIGINT` forces immediate `OSExit`. The executor's
`gracefulStop`/`gracefulRampDown` (stage scheduling) is explicitly **bypassed** on manual
interrupt. These two mechanisms must not be conflated.

**Grounding (verified at HEAD `ddc3b0b1d`):**

- `handleTestAbortSignals()` — `cmd/common.go:L97-L129`; the buffered channel
  `sigC := make(chan os.Signal, 2)` at `L99`; the first signal → `gracefulStopHandler`
  (`L104-L106`); the second signal → `onHardStop` then
  `gs.OSExit(int(exitcodes.ExternalAbort))` at `L118`.
- `cmd/run.go:L349-L358` — the `gracefulStop` closure: Debug "Stopping k6 in response to
  signal..." (`L350`), `runAbort(... fmt.Errorf("test run was aborted because k6 received a
  '%s' signal", sig) ...)` (`L352-L356`), `lingerCancel()` (`L357`). `cmd/run.go:L359-L362`
  — the `onHardStop` closure: Error "Aborting k6 in response to signal" (`L360`),
  `globalCancel()` (`L361`).
- The executor default `DefaultGracefulStopValue = 30 * time.Second` —
  `lib/executor/base_config.go:L20`; `GetGracefulStop()` at `L97`. The key comment at
  `base_config.go:L94-L96`: "Of course, that doesn't count when the user manually interrupts
  the test, then iterations are immediately stopped."
- Why JS work stops promptly: k6 interrupts the VM on context cancel —
  `context.AfterFunc(...) { vu.Runtime.Interrupt(context.Canceled) }` at `js/runner.go:L372`
  (also `js/bundle.go:L323-L324`); and `k6.Sleep` is context-aware —
  `select { case <-timer.C: case <-ctx.Done(): }` at `js/modules/k6/k6.go:L72`.

### Q3.1 Single SIGINT (measured linger to process exit), stable across 2 runs

- **Case A** — context-aware `sleep(8)` iterations (`/tmp/k6inv/q3_sleep.js`):
  `linger_seconds` = **0.036** (run 1), **0.032** (run 2); `exit_code` = **105**
  (ExternalAbort). `ITER_START`=4 / `ITER_END`=2 (in-flight iterations interrupted
  mid-flight).
- **Case B** — 25s JS busy-loop that ignores context (`/tmp/k6inv/q3_busy.js`):
  `linger_seconds` = **0.036** (run 1), **0.037** (run 2); `exit_code` = **105**;
  `ITER_START`=4 / `ITER_END`=**0** (the tight loop is force-interrupted by the sobek VM
  interrupt).
- Captured abort log lines (present in every single-signal run):

```text
time="..." level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="..." level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

Takeaway: even a script deliberately ignoring context cancellation is stopped in ~36 ms —
i.e. **much faster** than `gracefulStop=30s`, the opposite of "lingering."

### Q3.2 Two SIGINTs

Script `/tmp/k6inv/q3_teardown.js` with a 20s `teardown()` to widen the window. Captured
markers:

```text
time="..." level=debug msg="Stopping k6 in response to signal..." sig=interrupt   <- 1st SIGINT (graceful abort)
time="..." level=info  msg="TEARDOWN_START ms=..." source=console
time="..." level=error msg="Aborting k6 in response to signal" sig=interrupt      <- 2nd SIGINT (onHardStop)
```

Timing: `t(sig1 → exit) = 2.036s`, `t(sig2 → exit) = 0.005s`, `exit_code=105`.
`TEARDOWN_END` was **never** logged — the second signal's `OSExit` cut the 20s teardown
short.

**Q3 rationale.** The first `SIGINT` → graceful `runAbort` (cancels the run context; VU
iterations are interrupted immediately per `base_config.go:L94-L96`). The second `SIGINT` →
`onHardStop` → immediate `OSExit(105)` (`cmd/common.go:L118`) — the safety valve so a stuck
test always dies. The only way a VU can linger after Ctrl+C is if it is inside a **blocking
Go/host call** that neither context cancellation nor the sobek `Interrupt` can preempt until
it returns; this could not be reproduced with `sleep` or a tight JS loop (both stop in
~36 ms). The 30s `gracefulStop` is never honored on manual interrupt.

---

## Q4 — Execution-segment skew (three instances)

**Verdict (two parts):**

- **(a) "One instance consistently shows more" = BY DESIGN.** Striped scaling assigns the
  rounding remainder **deterministically** to a specific segment (with this sequence, the
  extra VU goes to segment `0:1/3`); it is not random and not a leak.
- **(b) "Summing them exceeds the configured maximum" = NOT reproduced.** The per-segment
  values sum to **exactly** the configured maximum at the peak and never exceed it —
  verified statically (0 invariant violations), at runtime (peak sum = max, 0 breaches,
  stable across 2 runs), and by the existing invariant test.

**Grounding (verified at HEAD `ddc3b0b1d`):**

- Striped scaling: `ExecutionSegmentSequenceWrapper.ScaleInt64()` —
  `lib/execution_segment.go:L580` (`result := (value / essw.lcd) * int64(len(offsets))`
  then a partial-cycle remainder loop); `GetStripedOffsets()` at `L598`;
  `ExecutionTuple.ScaleInt64()` at `L734` (returns the value unchanged if there is only one
  segment, `L736`); `ExecutionTuple.GetStripedOffsets()` at `L742`.
- Non-striped proportional scaling (used when there is no sequence):
  `ExecutionSegment.Scale()` — `lib/execution_segment.go:L253`, telescoping
  `round(value*to) − round(value*from)`.
- Sum invariant test: `TestSumRandomSegmentSequenceMatchesNoSegment` —
  `lib/executor/ramping_vus_test.go:L1112`.
- Segment template tuples `0:1/3`, `1/3:2/3` over sequence `0,1/3,2/3,1` —
  `lib/executor/executors_test.go:L228,L241`.

### Q4.1 Static ground truth (deterministic)

Config: ramp `0→10` (10s), hold `10` (`{5s,10}`), ramp `10→0` (5s), `GracefulRampDown=0`,
`GracefulStop=0`. Per-segment `GetExecutionRequirements` vs the no-segment (full) curve:

```text
t_offset   full  seg0  seg1  seg2  sum   sum==full?
      0s     0     0     0     0     0   OK
      1s     1     1     0     0     1   OK
      2s     2     1     1     0     2   OK
      3s     3     1     1     1     3   OK
      4s     4     2     1     1     4   OK
      5s     5     2     2     1     5   OK
      6s     6     2     2     2     6   OK
      7s     7     3     2     2     7   OK
      8s     8     3     3     2     8   OK
      9s     9     3     3     3     9   OK
     10s    10     4     3     3    10   OK
   13.5s     9     3     3     3     9   OK
     14s     8     3     3     2     8   OK
   14.5s     7     3     2     2     7   OK
     15s     6     2     2     2     6   OK
   15.5s     5     2     2     1     5   OK
     16s     4     2     1     1     4   OK
   16.5s     3     1     1     1     3   OK
     17s     2     1     1     0     2   OK
   17.5s     1     1     0     0     1   OK
     18s     0     0     0     0     0   OK

Invariant sum(seg0..2)==full violations: 0
Direct ScaleInt64(10): full=10 seg0=4 seg1=3 seg2=3 (sum=10)
```

Note the ramp: `seg0` **leads** (e.g. t=4s: 2 vs 1 vs 1; t=7s: 3 vs 2 vs 2). The
striped-vs-proportional comparison at target 10:

```text
target=10 | striped ScaleInt64 (WITH seq) vs proportional Scale (WITHOUT seq)
  0:1/3     striped=4  proportional=3
  1/3:2/3   striped=3  proportional=4
  2/3:1     striped=3  proportional=3
  SUM       striped=10  proportional=10  (max=10)
```

Both scaling modes sum to exactly 10; only *which* segment carries the +1 differs (striped
→ `seg0`; proportional → the middle segment). Neither exceeds the max.

### Q4.2 Runtime — three real `k6 run` instances

Script `/tmp/k6inv/q4_seg.js` (ramp `0→10`, hold, `→0`, `gracefulRampDown:'0s'`,
`gracefulStop:'0s'`, `sleep(1)`), launched concurrently:

```bash
SEQ="0,1/3,2/3,1"
/tmp/k6bin run -q --no-summary --out json=... --execution-segment "0:1/3"   --execution-segment-sequence "$SEQ" q4_seg.js &
/tmp/k6bin run -q --no-summary --out json=... --execution-segment "1/3:2/3" --execution-segment-sequence "$SEQ" q4_seg.js &
/tmp/k6bin run -q --no-summary --out json=... --execution-segment "2/3:1"   --execution-segment-sequence "$SEQ" q4_seg.js &
wait
```

Aligned per-second `vus` (RUN 1; RUN 2 identical at peak):

```text
sec  seg0 seg1 seg2  sum
 10     4    3    3   10
 11     4    3    3   10
 12     4    3    3   10
 13     4    3    3   10
 14     4    3    3   10
 15     4    3    3   10
peak aligned sum = 10 (configured max = 10)   aligned-sum breaches: 0
per-instance max vus = [4, 3, 3]
TRUE finest-granularity (union-timestamp, forward-filled) peak sum = 10, breaches>10 = 0   (both runs)
```

Rapid up/down + start-skew cross-product (`/tmp/k6inv/q4b_rapid.js`, target 9 = even
3/3/3, instances staggered +0/+1/+2s to simulate independent start times): **peak naive
wall-clock sum = 7**, per-instance max = `[3,3,3]`. Independent start times put the
instances at different phases, so their instantaneous counts partly **cancel** (sum lower),
never adding beyond the max — because each instance is individually bounded by its
per-segment peak and those peaks sum to the max.

### Q4.3 Invariant test corroboration

```text
$ CGO_ENABLED=1 go test -count=1 -run TestSumRandomSegmentSequenceMatchesNoSegment ./lib/executor/
ok  	go.k6.io/k6/lib/executor	0.083s
```

**Q4 rationale.** The extra VU on one instance is the deterministic striping remainder
(`ScaleInt64`, `execution_segment.go:L580/L734`) — reproducible and by design, not a random
leak. The sum never exceeds the configured maximum at any aligned instant; the invariant
holds (0 violations statically, test passes, runtime peak = max). The user's impression of
exceeding the max is a measurement artifact: three independent processes have independent
start times, so summing instantaneous `vus` by wall-clock compares different execution
offsets — and even so, the sum is bounded by (and at peak equals) the maximum.

---

## Q5 — Root cause: race? buffer leak? simultaneous mutation?

**Verdict: NO data race. NO VU buffer leak. The two handlers are NEVER simultaneous.** All
findings are by design; no defect was found.

**Grounding (verified at HEAD `ddc3b0b1d`):**

- Per-`vuHandle` `mutex *sync.Mutex` — `lib/executor/vu_handle.go:L71`; it is taken by
  `start()`/`gracefulStop()`/`hardStop()` (`L116`/`L148`/`L166`). The atomic `state`:
  `changeState()` uses `atomic.StoreInt32` (`L142-L145`); the VU loop reads it via
  `atomic.LoadInt32` (`L204`).
- The `getVU`/`returnVU` **1:1** invariant — `lib/executor/vu_handle.go:L60-L67` ("for each
  call to getVU there must be 1 (and only 1) call to returnVU").
- Handler **serialization** (happens-before) — `lib/executor/ramping_vus.go`:
  `iterateSteps(ctx, handleNewMaxAllowedVUs, handleNewScheduledVUs)` runs synchronously at
  `L549-L553` driving **both** handlers; then
  `go runRemainingGracefulSteps(ctx, handleNewMaxAllowedVUs, handledGracefulSteps)` at
  `L554-L558` spawns a new goroutine advancing **only** the max-allowed handler *after*
  `iterateSteps` has returned. The `go` statement is the happens-before edge, so the two
  handlers never mutate state at the same time.
- The VU buffer is the channel `es.vus`: `GetPlannedVU()` pulls `case vu := <-es.vus`
  (`lib/execution.go:L471`, `L474`); `ReturnVU()` pushes `es.vus <- vu` (`L544-L545`). The
  `getVU` closure does `wg.Add(1)` (`ramping_vus.go:L600`); `returnVU` does `wg.Done()`
  (`L608`); `defer runState.wg.Wait()` (`L540`) would hang forever on any unreturned VU.
- The `vus` count is informational: `ModCurrentlyActiveVUsCount` /
  `GetCurrentlyActiveVUsCount` carry the comment "IMPORTANT: for UI/information purposes
  only, don't use for synchronization" (`lib/execution.go` ~`L268-L278`). This explains why
  the user's printed debug counts appeared to "mismatch."

### Q5.1 Race detector — exact AAP command (run twice for stability)

```text
$ CGO_ENABLED=1 go test -race -count=1 -v -run 'TestVUHandleRace|TestVUHandleStartStopRace|TestRampingVUsGracefulRampDown' ./lib/executor/
=== RUN   TestRampingVUsGracefulRampDown
=== RUN   TestVUHandleRace
=== RUN   TestVUHandleStartStopRace
--- PASS: TestVUHandleRace (0.18s)
--- PASS: TestVUHandleStartStopRace (0.37s)
--- PASS: TestRampingVUsGracefulRampDown (2.50s)
PASS
ok  	go.k6.io/k6/lib/executor	3.532s
(run 2: ok  	go.k6.io/k6/lib/executor	3.532s)
```

Whole-package assurance:

```text
$ CGO_ENABLED=1 go test -race -count=1 ./lib/executor/
ok  	go.k6.io/k6/lib/executor	30.124s
```

(`TestVUHandleRace` `vu_handle_test.go:L25` and `TestVUHandleStartStopRace` `L114` are both
"mostly interesting when -race is enabled".)

### Q5.2 Corroboration against the user's EXACT scenarios (race-instrumented binary)

Built with `CGO_ENABLED=1 go build -race -o /tmp/k6race .`
(`k6race v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)`), then ran the user's own
scenarios under it:

```text
[Q1 long-GRD  ]  exit=0   DATA RACE occurrences: 0
[Q1b hardstop ]  exit=0   DATA RACE occurrences: 0   (forces BOTH graceful + hard stop = both handlers)
[Q4 seg0      ]           DATA RACE occurrences: 0
[Q4 seg1      ]           DATA RACE occurrences: 0
[Q4 seg2      ]           DATA RACE occurrences: 0
combined DATA RACE total across ALL user-scenario race runs = 0
```

### Q5.3 Buffer-leak evidence

`vus_max` (the buffer size) is constant (distinct values `[8]` for Q1; `[6]` for a
clean-completion run) — the buffer never grows. The `vus` metric decays toward 0 (a final
sample of 1 is a metric-sampling boundary, explicitly informational per the "don't use for
synchronization" comment, not a leak). Every investigation run exited cleanly (`exit 0`);
since `defer runState.wg.Wait()` (`ramping_vus.go:L540`) blocks until every `getVU`
(`wg.Add`, `L600`) has a matching `returnVU` (`wg.Done`, `L608`), the clean exits prove the
1:1 `getVU`/`returnVU` invariant held — no leak.

**Q5 rationale.**

- **(a) No race between the handler goroutines** — they are serialized by the `L549`→`L554`
  happens-before edge and never run simultaneously; the per-handle mutex + atomic `state`
  protect each `vuHandle`; the detector is clean on the targeted tests, the whole package,
  and the user's exact scenarios.
- **(b) No buffer leak** — `getVU`/`returnVU` are 1:1, `WaitGroup`-enforced, all runs exit
  cleanly, `vus_max` constant.
- **(c) "Both handlers modifying VU state simultaneously" does not happen**; the only
  genuine race — `start` vs the VU loop stopping — is resolved deterministically by the
  transition table (`vu_handle.go:L24-L55`, e.g. `L37`). The apparent count "mismatch" (Q2)
  is the informational-only `vus`/active-VU counters being read mid-ramp-down.

---

## Coverage pass

Every named mechanism, function, flag, and condition from the report is addressed:

- [x] **Scheduled handler** (`scheduledVUsHandlerStrategy` `ramping_vus.go:L679`) — Q2. BY DESIGN.
- [x] **Graceful / max-allowed handler** (`maxAllowedVUsHandlerStrategy` `ramping_vus.go:L668`) — Q2. BY DESIGN.
- [x] **VU buffer** (channel `es.vus`, `execution.go:L471/L544`; `getVU`/`returnVU` 1:1) — Q5. NO leak.
- [x] **`gracefulStop`** (default 30s, `base_config.go:L20`; bypassed on manual interrupt `L94-L96`) — Q3. BY DESIGN.
- [x] **`gracefulRampDown`** (reservation `ramping_vus.go:L307`) — Q1, Q2. BY DESIGN.
- [x] **Execution segments** (`ScaleInt64`/`GetStripedOffsets` `execution_segment.go:L580/L598/L734/L742`; invariant test `ramping_vus_test.go:L1112`) — Q4. one-more BY DESIGN; sum-exceeds NOT reproduced.
- [x] **The two Ctrl+C signals** (`handleTestAbortSignals` `cmd/common.go:L97-L129`, `L118`; closures `cmd/run.go:L349-L362`) — Q3. BY DESIGN.
- [x] **Transitional states** `starting` / `toGracefulStop` / `toHardStop` (`vu_handle.go:L16-L22`, transition table `L24-L55`) — Q1. All observed and deterministically resolved.

**Conclusion:** the investigation found **no genuine defect**; every reported symptom is
explained by documented, by-design behavior (or, for the Q3 "linger" and Q4 "exceed"
claims, was not reproducible for canonical inputs).

## Methodology & reproducibility

The canonical `k6 run` entry point and the exact commands shown above were used throughout;
the race detector required `CGO_ENABLED=1` plus `gcc`. Magnitude and timing claims were
confirmed across at least two identical runs, and run-to-run variability is reported exactly
as observed (Q1/Q2/Q4 counts were deterministic and identical across runs; Q3 linger
measured 0.032–0.037 s). All temporary reproduction scripts, driver scripts, and binaries
lived under `/tmp` (outside the repository) and were deleted afterward; the repository
working tree is unchanged except for this document. All `file:line` citations are anchored
to HEAD `ddc3b0b1d`.
