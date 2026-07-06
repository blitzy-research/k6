# Investigating the Suspected Concurrency Bug in k6's `ramping-vus` Executor

**A run-first, evidence-driven diagnosis.** This document answers the reported
question about a suspected concurrency bug in k6's `ramping-vus` executor. Every
behavioral claim below is backed by **(a)** the exact command that was run,
**(b)** the complete, unedited output it produced, and **(c)** a `file:line`
citation into the k6 source with cause→effect reasoning. Anything that was
reasoned from reading code rather than observed at runtime is explicitly labelled
**(inferred)**.

The investigation followed a strict *run-first* methodology: the canonical k6
binary was built and exercised through its real entry point (`k6 run`), and the
answer was written from the captured output. Source reading was used only to
locate and explain what was observed.

---

## Summary / TL;DR verdict table

| # | Reported symptom / question | Verdict | One-line evidence |
|---|-----------------------------|---------|-------------------|
| 1 | VUs get "stuck" — neither fully active nor fully stopped (rapid up/down + long `gracefulRampDown`) | **Transitional, not stuck** | VUs held at `5/5`, **0 interrupted** across the rapid cycles; `toGracefulStop` is reversed to `running` before the iteration ends. |
| 2 | Scheduled-handler VU count ≠ graceful-handler count at certain moments | **By design** | Raw/scheduled plan drops to `1` by t=7s while the max-allowed plan holds `6` until t=33s — intentional reservation. |
| 3 | After `ctrl+c`, VUs "keep running for way longer than `gracefulStop`" | **Not reproduced** | SIGINT teardown measured **47.7 ms / 45.4 ms** (3 VUs) and **51.1 ms** (50 VUs); nothing runs past `maxEndTime`. |
| 4 | Execution segments: one instance shows more VUs, and the sum "exceeds my configured maximum" | **Sum = max, never exceeds** | TARGET=10 → `4+3+3 = 10`; TARGET=9 → `3+3+3 = 9`. First segment gets the extra indivisible VU by design. |
| 5 | "Is the VU buffer leaking somehow?" | **No leak** | Across ~270 get/return ops/run: **0** buffer warnings/errors; `vus_max` held at `10`. |
| A | Is there a race between the two handler goroutines? | **No** | The two handlers are closures invoked **serially from one goroutine**; `-race` clean 2/2. |
| B | Trace what happens when both handlers mutate VU state simultaneously | **Serialized (cannot happen concurrently)** | Every transition is guarded by a per-VU `sync.Mutex` + atomic state write; `-race` clean. |

**Bottom line:** none of the five symptoms is a concurrency bug. Symptoms 1, 2 and
4 are *designed* behaviours (graceful ramp-down reservation and execution-segment
striping); Symptom 3 does not reproduce (ctrl+c is *faster* than `gracefulStop`,
not slower); Symptom 5 shows a fully conserved buffer. The two handler strategies
never run concurrently, and per-VU state mutation is serialized by a mutex and
atomic writes — the `-race` detector is clean across every probe and repeated run.

---

## Methodology & environment

**Run-first rules applied throughout (per the investigation brief):**

1. **Run-first, evidence-driven** — every claim carries its command, complete
   unedited output, and a `file:line` citation. Inferred statements are labelled.
2. **Real entry point only** — the canonical `k6 run` CLI and the real
   `ramping-vus` executor (`main.go` → `cmd.Execute()`). No debug hooks or stand-ins.
3. **Canonical build/config** — the default binary a normal user would build.
4. **Reproduce inconsistency, do not stabilize** — every "sometimes/consistently"
   claim was run with the *same unchanged input* **≥ 2×** and the distribution reported.
5. **Observe real magnitude/timing** — run durations/scales are stated and
   confirmed stable across ≥ 2 runs.
6. **Every condition exercised** — primary (rapid up/down + long `gracefulRampDown`),
   edge (early `ctrl+c`/SIGINT), alternate (three-instance execution segments), and
   transitional (before/during/after ramp & interrupt) states.
7. **`-race` detector** is the primary instrument for the concurrency questions;
   its verdict is reported verbatim.

**Environment.** The work was performed in the target container at k6 HEAD
`ddc3b0b1d23c128e34e2792fc9075f9126e32375` (the branch is bound to the deliverable
name `k6_ddc3b0b1d23c.md`). The Go toolchain matches `go.mod`
(`go 1.21` at `go.mod:L3`, `toolchain go1.21.13` at `go.mod:L5`):

```
$ go version
go version go1.21.13 linux/amd64
```

**Canonical build** — the exact command used by the `Makefile` `build:` target
(`go build`, `Makefile:L7-L8`). `GOFLAGS=-mod=vendor` is set because the repo
vendors its dependencies (offline-capable):

```
$ export PATH=/usr/local/go/bin:$PATH
$ export GOFLAGS=-mod=vendor
$ go build -o /tmp/k6bin .
$ /tmp/k6bin version
k6bin v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)
```

This is canonical **k6 v0.55.0**, built at commit **`ddc3b0b1d2`** on **go1.21.13**,
which is exactly what a normal user's `go build` would produce here. Every
`k6 run` invocation below uses this `/tmp/k6bin` binary through its real entry
point. All temporary observation scripts live under `/tmp/k6obs/` (outside the
repository) and are deleted at the end (see Section 8); the source tree is left
byte-for-byte unchanged apart from this one document.

**Test/probe command family.** The `-race` probes use the canonical detector
from the `Makefile` `tests:` target (`go test -race -timeout 210s ./...`,
`Makefile:L28-L29`), narrowed to the relevant tests and package.

---

## Section 1 — Symptom 1: "Stuck" VUs → transitional, not stuck

**What the user reported.** With stages that go up and down rapidly and a long
`gracefulRampDown`, some VUs "seem to get stuck in a state where they're neither
fully active nor fully stopped."

**Observation script** (`/tmp/k6obs/rapid_ramp.js`): six 1-second stages
alternating `target: 5` and `target: 0`, with `gracefulRampDown: '30s'` and a
2-second iteration (longer than a single ramp stage):

```js
import { sleep } from 'k6';
import exec from 'k6/execution';
export const options = { scenarios: { rapid: {
  executor: 'ramping-vus', startVUs: 0,
  stages: [
    { target: 5, duration: '1s' }, { target: 0, duration: '1s' },
    { target: 5, duration: '1s' }, { target: 0, duration: '1s' },
    { target: 5, duration: '1s' }, { target: 0, duration: '1s' },
  ],
  gracefulRampDown: '30s',
}}};
export default function () {
  const t = Math.round(exec.instance.currentTestRunDuration);
  console.log(`t=${t}ms vusActive=${exec.instance.vusActive} iterInInstance=${exec.instance.iterationsCompleted}`);
  sleep(2);
}
```

**Command and complete unedited output — RUN 1:**

```
$ /tmp/k6bin run --no-color /tmp/k6obs/rapid_ramp.js

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6obs/rapid_ramp.js
        output: -

     scenarios: (100.00%) 1 scenario, 5 max VUs, 36s max duration (incl. graceful stop):
              * rapid: Up to 5 looping VUs for 6s over 6 stages (gracefulRampDown: 30s, gracefulStop: 30s)

time="2026-07-06T22:09:00Z" level=info msg="t=201ms vusActive=1 iterInInstance=0" source=console
time="2026-07-06T22:09:01Z" level=info msg="t=401ms vusActive=2 iterInInstance=0" source=console
time="2026-07-06T22:09:01Z" level=info msg="t=601ms vusActive=3 iterInInstance=0" source=console
time="2026-07-06T22:09:01Z" level=info msg="t=801ms vusActive=4 iterInInstance=0" source=console

running (01.0s), 4/5 VUs, 0 complete and 0 interrupted iterations
rapid   [  17% ] 4/5 VUs  1.0s/6.0s
time="2026-07-06T22:09:01Z" level=info msg="t=1001ms vusActive=5 iterInInstance=0" source=console

running (02.0s), 5/5 VUs, 0 complete and 0 interrupted iterations
rapid   [  33% ] 5/5 VUs  2.0s/6.0s
time="2026-07-06T22:09:02Z" level=info msg="t=2202ms vusActive=5 iterInInstance=1" source=console
time="2026-07-06T22:09:03Z" level=info msg="t=2402ms vusActive=5 iterInInstance=2" source=console
time="2026-07-06T22:09:03Z" level=info msg="t=2601ms vusActive=5 iterInInstance=3" source=console
time="2026-07-06T22:09:03Z" level=info msg="t=2801ms vusActive=5 iterInInstance=4" source=console

running (03.0s), 5/5 VUs, 4 complete and 0 interrupted iterations
rapid   [  50% ] 5/5 VUs  3.0s/6.0s
time="2026-07-06T22:09:03Z" level=info msg="t=3001ms vusActive=5 iterInInstance=5" source=console

running (04.0s), 5/5 VUs, 5 complete and 0 interrupted iterations
rapid   [  67% ] 5/5 VUs  4.0s/6.0s
time="2026-07-06T22:09:04Z" level=info msg="t=4203ms vusActive=5 iterInInstance=6" source=console
time="2026-07-06T22:09:05Z" level=info msg="t=4402ms vusActive=5 iterInInstance=7" source=console
time="2026-07-06T22:09:05Z" level=info msg="t=4602ms vusActive=5 iterInInstance=8" source=console
time="2026-07-06T22:09:05Z" level=info msg="t=4801ms vusActive=5 iterInInstance=9" source=console

running (05.0s), 5/5 VUs, 9 complete and 0 interrupted iterations
rapid   [  83% ] 5/5 VUs  5.0s/6.0s
time="2026-07-06T22:09:05Z" level=info msg="t=5001ms vusActive=5 iterInInstance=10" source=console

running (06.0s), 5/5 VUs, 10 complete and 0 interrupted iterations
rapid   [ 100% ] 5/5 VUs  6.0s/6.0s

running (07.0s), 1/5 VUs, 14 complete and 0 interrupted iterations
rapid ↓ [ 100% ] 5/5 VUs  6s

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=2s min=2s med=2s max=2.03s p(90)=2s p(95)=2.01s
     iterations...........: 15  2.142293/s
     vus..................: 1   min=1      max=5
     vus_max..............: 5   min=5      max=5


running (07.0s), 0/5 VUs, 15 complete and 0 interrupted iterations
rapid ✓ [ 100% ] 0/5 VUs  6s
```

**Command and complete unedited output — RUN 2** (same unchanged input, to test
the "sometimes" claim):

```
$ /tmp/k6bin run --no-color /tmp/k6obs/rapid_ramp.js

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6obs/rapid_ramp.js
        output: -

     scenarios: (100.00%) 1 scenario, 5 max VUs, 36s max duration (incl. graceful stop):
              * rapid: Up to 5 looping VUs for 6s over 6 stages (gracefulRampDown: 30s, gracefulStop: 30s)

time="2026-07-06T22:09:17Z" level=info msg="t=201ms vusActive=1 iterInInstance=0" source=console
time="2026-07-06T22:09:17Z" level=info msg="t=400ms vusActive=2 iterInInstance=0" source=console
time="2026-07-06T22:09:17Z" level=info msg="t=600ms vusActive=3 iterInInstance=0" source=console
time="2026-07-06T22:09:17Z" level=info msg="t=801ms vusActive=4 iterInInstance=0" source=console

running (01.0s), 4/5 VUs, 0 complete and 0 interrupted iterations
rapid   [  17% ] 4/5 VUs  1.0s/6.0s
time="2026-07-06T22:09:18Z" level=info msg="t=1001ms vusActive=5 iterInInstance=0" source=console

running (02.0s), 5/5 VUs, 0 complete and 0 interrupted iterations
rapid   [  33% ] 5/5 VUs  2.0s/6.0s
time="2026-07-06T22:09:19Z" level=info msg="t=2201ms vusActive=5 iterInInstance=1" source=console
time="2026-07-06T22:09:19Z" level=info msg="t=2401ms vusActive=5 iterInInstance=2" source=console
time="2026-07-06T22:09:19Z" level=info msg="t=2601ms vusActive=5 iterInInstance=3" source=console
time="2026-07-06T22:09:19Z" level=info msg="t=2802ms vusActive=5 iterInInstance=4" source=console

running (03.0s), 5/5 VUs, 4 complete and 0 interrupted iterations
rapid   [  50% ] 5/5 VUs  3.0s/6.0s
time="2026-07-06T22:09:20Z" level=info msg="t=3001ms vusActive=5 iterInInstance=5" source=console

running (04.0s), 5/5 VUs, 5 complete and 0 interrupted iterations
rapid   [  67% ] 5/5 VUs  4.0s/6.0s
time="2026-07-06T22:09:21Z" level=info msg="t=4242ms vusActive=5 iterInInstance=6" source=console
time="2026-07-06T22:09:21Z" level=info msg="t=4401ms vusActive=5 iterInInstance=7" source=console
time="2026-07-06T22:09:21Z" level=info msg="t=4602ms vusActive=5 iterInInstance=8" source=console
time="2026-07-06T22:09:21Z" level=info msg="t=4803ms vusActive=5 iterInInstance=9" source=console

running (05.0s), 5/5 VUs, 9 complete and 0 interrupted iterations
rapid   [  83% ] 5/5 VUs  5.0s/6.0s
time="2026-07-06T22:09:22Z" level=info msg="t=5002ms vusActive=5 iterInInstance=10" source=console

running (06.0s), 5/5 VUs, 10 complete and 0 interrupted iterations
rapid   [ 100% ] 5/5 VUs  6.0s/6.0s

running (07.0s), 1/5 VUs, 14 complete and 0 interrupted iterations
rapid ↓ [ 100% ] 5/5 VUs  6s

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=2s min=2s med=2s max=2.04s p(90)=2.02s p(95)=2.03s
     iterations...........: 15  2.130827/s
     vus..................: 1   min=1      max=5
     vus_max..............: 5   min=5      max=5


running (07.0s), 0/5 VUs, 15 complete and 0 interrupted iterations
rapid ✓ [ 100% ] 0/5 VUs  6s
```

**Distribution across runs.** RUN 1 and RUN 2 are identical on every VU count and
on the `complete`/`interrupted` totals (`15` complete, `0` interrupted, `vus_max=5`).
The only differences are sub-millisecond iteration timing (`max=2.03s` vs `2.04s`)
and per-second rate (`2.142293/s` vs `2.130827/s`). **The behaviour is deterministic**,
so the user's "sometimes" is not a nondeterministic race — it is the same designed
behaviour every time.

**Cause → effect (why VUs look "stuck" but are not).**

- The progress line format is `active/max VUs`. During the rapid up/down cycles the
  **active** count *holds at `5/5`* instead of dropping to `0` between cycles, and
  iterations keep completing with **`0 interrupted`** throughout. A hung/stuck VU
  would show up as `interrupted` iterations or a frozen count — neither occurs.
- The apparent "stuck" state maps to the two transitional states in the per-VU state
  machine, `toGracefulStop` and `toHardStop` (constants at `lib/executor/vu_handle.go:L20`
  and `:L21`). These are **transient**, not terminal: the run-loop clears them back to
  `stopped`. The state-transition table encodes this explicitly —
  `| loop | toGracefulStop | stopped |` (`lib/executor/vu_handle.go:L42`) and
  `| loop | toHardStop | stopped |` (`:L43`) — and `runLoopsIfPossible`
  (`lib/executor/vu_handle.go:L185`) performs that clear/reinit on its slow path,
  ending in `changeState(stopped)`.
- With a *long* `gracefulRampDown` and *rapid* re-starts, a VU that has just been asked
  to graceful-stop is re-started before its current iteration ends. The table row
  `| start | toGracefulStop | running |` with the comment *"we raced with the loop
  stopping, just continue"* (`lib/executor/vu_handle.go:L37`) shows the design: the
  `start()` reverses the pending graceful stop, so the VU **stays legitimately active**
  and finishes its 2-second iteration. That is exactly the observed `5/5` hold with
  `0 interrupted`.

**Verdict:** the "stuck between active and stopped" appearance is the *designed
graceful-hold* produced by a long `gracefulRampDown`, not a hang. Confirmed
identical across 2 runs.

---

## Section 2 — Symptom 2: scheduled vs graceful / max-allowed divergence → by design

**What the user reported.** Debug output showed that "the number of VUs being
tracked by the scheduled handler doesn't match what the graceful handler thinks
should exist" at certain moments.

**Runtime evidence (from Section 1).** In the rapid-ramp run above, the k6 progress
line makes the reservation visible: at t=7s the run shows `1/5 VUs`. That is k6's
**active/max VU display** — `active=1` (VUs currently executing an iteration) over
`max=5` (the scenario's VU allocation, whose peak is `5`). The active count has
dropped to `1` while the max VU pool is still held at `5` because the long
`gracefulRampDown` reservation has not yet expired; the two reconcile only at the very
end (`0/5 VUs, 15 complete`). This progress line demonstrates **active-vs-max
divergence** — the runtime *symptom* of the reservation — but it is **not** a direct
readout of the `scheduledVUsHandlerStrategy` private `cur` counter. The precise
scheduled-handler-vs-max-allowed-handler count divergence the user described is
established instead by the config's computed step plan below.

**Step-plan evidence (the config's computed plan).** The two handlers consume two
*different* step streams. The exact numbers are computed by the config and asserted
by `TestRampingVUsConfigExecutionPlanExample` (`lib/executor/ramping_vus_test.go:L442`).
Running it confirms the config produces those arrays:

```
$ go test -count=1 -run 'TestRampingVUsConfigExecutionPlanExample$' -v ./lib/executor/
=== RUN   TestRampingVUsConfigExecutionPlanExample
=== PAUSE TestRampingVUsConfigExecutionPlanExample
=== CONT  TestRampingVUsConfigExecutionPlanExample
--- PASS: TestRampingVUsConfigExecutionPlanExample (0.00s)
PASS
ok  	go.k6.io/k6/lib/executor	0.005s
```

The arrays asserted by that passing test (quoted here as the config's *computed
plan*, i.e. source evidence, not runtime output) are:

- **Raw / scheduled steps** (`conf.getRawExecutionSteps(et, false)` at
  `lib/executor/ramping_vus.go:L171`), asserted in full by the passing test as
  `expRawStepsNoZeroEnd` (`lib/executor/ramping_vus_test.go:L459-L480`):
  `{0s:4, 1s:5, 2s:6, 3s:5, 4s:4, 5s:3, 6s:2, 7s:1, 8s:2, 9s:3, 10s:4, 11s:5,
  12s:4, 13s:3, 14s:2, 15s:1, 16s:2, 17s:3, 18s:4, 20s:1}` — peak `6` at t=2s,
  dropping to **`1` by t=7s** (there is no `19s` entry; the plan jumps `18s→20s`).
- **Graceful / max-allowed steps** (`conf.GetExecutionRequirements(et)` at
  `lib/executor/ramping_vus.go:L434`), with the default 30s `gracefulStop` and 30s
  `gracefulRampDown`: `{0s:4, 1s:5, 2s:6, 33s:5, 42s:4, 50s:1, 53s:0}`.

At **t=7s** the scheduled plan says `1` but the max-allowed plan still says **`6`**
(it does not begin decreasing until t=33s). That gap of `6 − 1 = 5` is precisely the
"handler count divergence" the user saw.

**Cause → effect (why the divergence is intentional).**

- The two handlers are distinct strategies consuming distinct streams:
  `scheduledVUsHandlerStrategy` (`lib/executor/ramping_vus.go:L679`) consumes the
  **raw** steps, and `maxAllowedVUsHandlerStrategy` (`:L668`) consumes the **graceful**
  steps. Inside `iterateSteps` they are called on the two arrays
  `rs.executor.rawSteps` and `rs.executor.gracefulSteps` (see Section 6).
- The graceful steps are produced by `reserveVUsForGracefulRampDowns`
  (`lib/executor/ramping_vus.go:L307`), whose own comment states its purpose:
  *"whenever there's a scaling down of VUs, it prevents the number of VUs from
  decreasing for the configured gracefulRampDown period"*
  (`lib/executor/ramping_vus.go:L300-L301`). `GracefulRampDown` defaults to
  `30 * time.Second` (`lib/executor/ramping_vus.go:L52`).
- `GetExecutionRequirements` (`lib/executor/ramping_vus.go:L434`) also appends a final
  0-VU step at `sumStagesDuration(Stages) + GracefulStop`. The same passing test shows
  this end-offset shifting with `GracefulStop`: default → final 0-VU step at `53s`;
  `GracefulStop=80s` → `103s`; `GracefulStop=3s` → `26s`; `GracefulStop=0` → `23s`.

**Verdict:** the scheduled-vs-max-allowed divergence is **intentional reservation**,
not a defect. The max-allowed count is deliberately kept at or above the scheduled
count for the `gracefulRampDown` window so that a VU asked to stop can still finish
(or be re-used by) an in-flight iteration.

---

## Section 3 — Symptom 3: `ctrl+c` overrun vs `gracefulStop`

**What the user reported.** After killing the test early with `ctrl+c`, "some VUs
keep running for way longer than `gracefulStop` should allow."

This requires distinguishing **user-abort** (SIGINT) from **natural end**, because
`gracefulStop` only governs the natural end. Both are exercised below.

### 3a — Early SIGINT, timed (the reported scenario)

**Observation script** (`/tmp/k6obs/longiter.js`): 3 VUs, a 30-second stage, default
`gracefulStop: '30s'`, and a 10-second iteration, so a VU is always mid-iteration
when interrupted:

```js
import { sleep } from 'k6';
import exec from 'k6/execution';
export const options = { scenarios: { li: {
  executor: 'ramping-vus', startVUs: 3, stages: [{ target: 3, duration: '30s' }], gracefulStop: '30s',
}}};
export default function () {
  console.log(`ITER_START t=${Math.round(exec.instance.currentTestRunDuration)}ms vu=${exec.vu.idInTest} vusActive=${exec.instance.vusActive}`);
  sleep(10);
  console.log(`ITER_END   t=${Math.round(exec.instance.currentTestRunDuration)}ms vu=${exec.vu.idInTest}`);
}
```

**Command (spawn, wait 3s, send SIGINT, time the teardown) and complete unedited
output — RUN 1:**

```
$ /tmp/k6bin run --no-color /tmp/k6obs/longiter.js > /tmp/k6obs/s3_run1.txt 2>&1 &
$ K6PID=$!; sleep 3
$ T0=$(python3 -c 'import time;print(time.time())'); kill -INT $K6PID; wait $K6PID; code=$?
$ T1=$(python3 -c 'import time;print(time.time())')
$ echo "k6 exit code after SIGINT = $code"
k6 exit code after SIGINT = 105
$ python3 -c "print(f'SIGINT->exit teardown = {($T1-$T0)*1000:.1f} ms')"
SIGINT->exit teardown = 47.7 ms
```

Full k6 output for RUN 1 (`$ cat /tmp/k6obs/s3_run1.txt`):

```

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6obs/longiter.js
        output: -

     scenarios: (100.00%) 1 scenario, 3 max VUs, 1m0s max duration (incl. graceful stop):
              * li: Up to 3 looping VUs for 30s over 1 stages (gracefulRampDown: 30s, gracefulStop: 30s)

time="2026-07-06T23:02:20Z" level=info msg="ITER_START t=0ms vu=3 vusActive=3" source=console
time="2026-07-06T23:02:20Z" level=info msg="ITER_START t=0ms vu=1 vusActive=3" source=console
time="2026-07-06T23:02:20Z" level=info msg="ITER_START t=0ms vu=2 vusActive=3" source=console

running (0m01.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
li     [   3% ] 3/3 VUs  01.0s/30.0s

running (0m02.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
li     [   7% ] 3/3 VUs  02.0s/30.0s
time="2026-07-06T23:02:23Z" level=info msg="ITER_END   t=2994ms vu=3" source=console

     data_received...: 0 B 0 B/s
     data_sent.......: 0 B 0 B/s
     vus.............: 3   min=3 max=3
     vus_max.........: 3   min=3 max=3


running (0m03.0s), 0/3 VUs, 0 complete and 3 interrupted iterations
li   ✗ [  10% ] 3/3 VUs  03.0s/30.0s
time="2026-07-06T23:02:23Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

**Command and complete unedited output — RUN 2** (same unchanged input):

```
$ /tmp/k6bin run --no-color /tmp/k6obs/longiter.js > /tmp/k6obs/s3_run2.txt 2>&1 &
$ K6PID=$!; sleep 3
$ T0=$(python3 -c 'import time;print(time.time())'); kill -INT $K6PID; wait $K6PID; code=$?
$ T1=$(python3 -c 'import time;print(time.time())')
$ echo "k6 exit code after SIGINT = $code"
k6 exit code after SIGINT = 105
$ python3 -c "print(f'SIGINT->exit teardown = {($T1-$T0)*1000:.1f} ms')"
SIGINT->exit teardown = 45.4 ms
$ cat /tmp/k6obs/s3_run2.txt

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6obs/longiter.js
        output: -

     scenarios: (100.00%) 1 scenario, 3 max VUs, 1m0s max duration (incl. graceful stop):
              * li: Up to 3 looping VUs for 30s over 1 stages (gracefulRampDown: 30s, gracefulStop: 30s)

time="2026-07-06T23:02:33Z" level=info msg="ITER_START t=0ms vu=1 vusActive=3" source=console
time="2026-07-06T23:02:33Z" level=info msg="ITER_START t=0ms vu=3 vusActive=3" source=console
time="2026-07-06T23:02:33Z" level=info msg="ITER_START t=0ms vu=2 vusActive=3" source=console

running (0m01.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
li     [   3% ] 3/3 VUs  01.0s/30.0s

running (0m02.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
li     [   7% ] 3/3 VUs  02.0s/30.0s

     data_received...: 0 B 0 B/s
     data_sent.......: 0 B 0 B/s
     vus.............: 3   min=3 max=3
     vus_max.........: 3   min=3 max=3


running (0m03.0s), 0/3 VUs, 0 complete and 3 interrupted iterations
li   ✗ [  10% ] 3/3 VUs  03.0s/30.0s
time="2026-07-06T23:02:36Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

**Distribution across runs.** SIGINT→exit teardown = **47.7 ms** (RUN 1) and
**45.4 ms** (RUN 2) — stable at ≈45–48 ms. Both runs end with `0/3 VUs, 0 complete
and 3 interrupted iterations` and the error `test run was aborted because k6
received a 'interrupt' signal`. (Whether a stray `ITER_END` line appears is itself
run-to-run variable and benign: RUN 1 shows one — `ITER_END t=2994ms vu=3` — while
RUN 2 shows none. k6's `sleep()` is context-interruptible, so on abort a VU's
post-`sleep` line can race out just before process exit; the iteration is still
counted as **interrupted**, not complete, in both runs.)

### 3b — 50-VU variant (abort is prompt regardless of scale)

**Observation script** (`/tmp/k6obs/many_vu_longiter.js`): 50 VUs (`startVUs: 50`,
held for a 60-second stage), `gracefulStop: '30s'`, and 20-second iterations — so all
50 VUs are active and mid-iteration when interrupted. The command captures the exit
code and times the teardown exactly as in 3a; the complete unedited k6 output follows:

```
$ /tmp/k6bin run --no-color /tmp/k6obs/many_vu_longiter.js > /tmp/k6obs/s3_many.txt 2>&1 &
$ K6PID=$!; sleep 3
$ T0=$(python3 -c 'import time;print(time.time())'); kill -INT $K6PID; wait $K6PID; code=$?
$ T1=$(python3 -c 'import time;print(time.time())')
$ echo "k6 exit code after SIGINT = $code"
k6 exit code after SIGINT = 105
$ python3 -c "print(f'SIGINT->exit teardown = {($T1-$T0)*1000:.1f} ms')"
SIGINT->exit teardown = 51.1 ms
$ cat /tmp/k6obs/s3_many.txt

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6obs/many_vu_longiter.js
        output: -

     scenarios: (100.00%) 1 scenario, 50 max VUs, 1m30s max duration (incl. graceful stop):
              * m: Up to 50 looping VUs for 1m0s over 1 stages (gracefulRampDown: 30s, gracefulStop: 30s)

time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=6 vusActive=14" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=4 vusActive=17" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=22 vusActive=33" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=26 vusActive=21" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=38 vusActive=36" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=35 vusActive=30" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=30 vusActive=40" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=41 vusActive=37" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=20 vusActive=33" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=19 vusActive=46" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=34 vusActive=49" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=5 vusActive=23" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=8 vusActive=25" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=47 vusActive=50" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=33 vusActive=50" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=44 vusActive=50" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=28 vusActive=27" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=40 vusActive=28" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=43 vusActive=50" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=27 vusActive=28" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=36 vusActive=29" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=16 vusActive=28" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=15 vusActive=16" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=3 vusActive=16" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=23 vusActive=30" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=24 vusActive=31" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=37 vusActive=32" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=10 vusActive=32" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=7 vusActive=14" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=9 vusActive=15" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=21 vusActive=33" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=1 vusActive=19" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=12 vusActive=14" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=11 vusActive=20" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=29 vusActive=41" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=14 vusActive=42" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=39 vusActive=43" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=13 vusActive=43" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=31 vusActive=43" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=25 vusActive=44" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=42 vusActive=47" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=32 vusActive=48" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=48 vusActive=48" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=50 vusActive=50" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=46 vusActive=50" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=45 vusActive=50" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=17 vusActive=26" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=2 vusActive=50" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=1ms vu=49 vusActive=50" source=console
time="2026-07-06T23:02:48Z" level=info msg="ITER_START t=0ms vu=18 vusActive=22" source=console

running (0m01.0s), 50/50 VUs, 0 complete and 0 interrupted iterations
m      [   2% ] 50/50 VUs  0m01.0s/1m00.0s

running (0m02.0s), 50/50 VUs, 0 complete and 0 interrupted iterations
m      [   3% ] 50/50 VUs  0m02.0s/1m00.0s
time="2026-07-06T23:02:51Z" level=info msg="ITER_END   t=2985ms vu=39" source=console
time="2026-07-06T23:02:51Z" level=info msg="ITER_END   t=2985ms vu=2" source=console
time="2026-07-06T23:02:51Z" level=info msg="ITER_END   t=2985ms vu=27" source=console
time="2026-07-06T23:02:51Z" level=info msg="ITER_END   t=2985ms vu=45" source=console
time="2026-07-06T23:02:51Z" level=info msg="ITER_END   t=2985ms vu=18" source=console

     data_received...: 0 B 0 B/s
     data_sent.......: 0 B 0 B/s
     vus.............: 50  min=50 max=50
     vus_max.........: 50  min=50 max=50


running (0m03.0s), 00/50 VUs, 0 complete and 50 interrupted iterations
m    ✗ [   5% ] 49/50 VUs  0m03.0s/1m00.0s
time="2026-07-06T23:02:51Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

Even with 50 VUs each mid-way through a 20-second iteration, teardown is **51.1 ms**
and all 50 iterations are **interrupted** — the abort does not wait for `gracefulStop`.
(As in 3a, a handful of VUs — here 5 — race out an `ITER_END` line at `t≈2985ms` just
before process exit; those iterations are still counted `interrupted`, not complete.)

### 3c — Natural end, LONG `gracefulStop` (contrast)

**Observation script** (`/tmp/k6obs/natural_long_gs.js`): 2 VUs, 3-second stage,
`gracefulStop: '30s'`, 5-second iteration. Here there is **no** SIGINT — the run ends
naturally.

The wall time is measured with an explicit `T0`/`T1` pair (as in 3a) so the timing line
is genuinely printed *after* the run output — there is **no** `ITER_END`/abort here:

```
$ T0=$(python3 -c 'import time;print(time.time())')
$ /tmp/k6bin run --no-color /tmp/k6obs/natural_long_gs.js

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6obs/natural_long_gs.js
        output: -

     scenarios: (100.00%) 1 scenario, 2 max VUs, 33s max duration (incl. graceful stop):
              * n: Up to 2 looping VUs for 3s over 1 stages (gracefulRampDown: 30s, gracefulStop: 30s)

time="2026-07-06T23:03:08Z" level=info msg="START t=0ms vu=2" source=console
time="2026-07-06T23:03:08Z" level=info msg="START t=0ms vu=1" source=console

running (01.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
n      [  33% ] 2/2 VUs  1.0s/3.0s

running (02.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
n      [  67% ] 2/2 VUs  2.0s/3.0s

running (03.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
n      [ 100% ] 2/2 VUs  3.0s/3.0s

running (04.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
n    ↓ [ 100% ] 2/2 VUs  3s

running (05.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
n    ↓ [ 100% ] 2/2 VUs  3s
time="2026-07-06T23:03:13Z" level=info msg="END   t=5001ms vu=1" source=console
time="2026-07-06T23:03:13Z" level=info msg="END   t=5001ms vu=2" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=5s min=5s med=5s max=5s p(90)=5s p(95)=5s
     iterations...........: 2   0.399874/s
     vus..................: 2   min=2      max=2
     vus_max..............: 2   min=2      max=2


running (05.0s), 0/2 VUs, 2 complete and 0 interrupted iterations
n    ✓ [ 100% ] 0/2 VUs  3s
$ T1=$(python3 -c 'import time;print(time.time())')
$ python3 -c "print(f'total wall time = {($T1-$T0):.2f} s')"
total wall time = 5.08 s
```

At natural end, the mid-flight 5-second iteration is **allowed to finish** (`END
t=5001ms`, `2 complete and 0 interrupted`) because the 30s `gracefulStop` window is
wide enough. Total wall = **5.08 s** (the run ends as soon as the last VU finishes,
well inside the `33s` max duration).

### 3d — Natural end, SHORT `gracefulStop` (the deadline in action)

**Observation script** (`/tmp/k6obs/natural_short_gs.js`): 2 VUs, 3-second stage,
`gracefulStop: '2s'`, 10-second iteration.

Timed the same way (explicit `T0`/`T1`, timing line printed after the run):

```
$ T0=$(python3 -c 'import time;print(time.time())')
$ /tmp/k6bin run --no-color /tmp/k6obs/natural_short_gs.js

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6obs/natural_short_gs.js
        output: -

     scenarios: (100.00%) 1 scenario, 2 max VUs, 5s max duration (incl. graceful stop):
              * n: Up to 2 looping VUs for 3s over 1 stages (gracefulRampDown: 30s, gracefulStop: 2s)

time="2026-07-06T23:03:22Z" level=info msg="START t=0ms vu=1" source=console
time="2026-07-06T23:03:22Z" level=info msg="START t=0ms vu=2" source=console

running (1.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
n      [  33% ] 2/2 VUs  1.0s/3.0s

running (2.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
n      [  67% ] 2/2 VUs  2.0s/3.0s

running (3.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
n      [ 100% ] 2/2 VUs  3.0s/3.0s

running (4.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
n    ↓ [ 100% ] 2/2 VUs  3s

running (5.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
n    ↓ [ 100% ] 2/2 VUs  3s
time="2026-07-06T23:03:27Z" level=warning msg="No script iterations fully finished, consider making the test duration longer"

     data_received...: 0 B 0 B/s
     data_sent.......: 0 B 0 B/s
     vus.............: 2   min=2 max=2
     vus_max.........: 2   min=2 max=2


running (5.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
n    ✓ [ 100% ] 2/2 VUs  3s
$ T1=$(python3 -c 'import time;print(time.time())')
$ python3 -c "print(f'total wall time = {($T1-$T0):.2f} s')"
total wall time = 5.07 s
```

The banner reports `5s max duration (incl. graceful stop)` = `regularDuration(3s) +
gracefulStop(2s)`, and the 10-second iteration is **interrupted at exactly t=5s** —
the deadline, not any longer.

**Cause → effect.**

- A single SIGINT is trapped by `handleTestAbortSignals` (`cmd/common.go:L97`), which
  calls `gs.SignalNotify(sigC, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)`
  (`cmd/common.go:L101`). The first signal invokes the `gracefulStop` closure in
  `cmd/run.go:L349`, which calls `runAbort(...)` (`cmd/run.go:L352`) and then
  `lingerCancel()` (`cmd/run.go:L357`, commented *"cancel this context as well, since
  the user did Ctrl+C"*). This aborts the run **immediately** — iterations are
  cancelled and counted `interrupted`, *not* granted the `gracefulStop` grace period.
  (A *second* signal would invoke `onHardStop` → `globalCancel()` at `cmd/run.go:L359,L361`.)
- At *natural* end, teardown is bounded by the duration context built in
  `getDurationContexts` (`lib/executor/helpers.go:L168`):
  `maxEndTime := startTime.Add(regularDuration + gracefulStop)`
  (`lib/executor/helpers.go:L172`), wired into `maxDurationCtx` (`:L174`) and
  `regDurationCtx` (`:L178`). `GracefulStop` defaults to `30 * time.Second`
  (`lib/executor/base_config.go:L20`). A long `gracefulStop` lets a mid-iteration VU
  finish (3c); a short one interrupts it at exactly `maxEndTime` (3d).

**Verdict (reported honestly):** the user's "VUs run *longer* than `gracefulStop`
after `ctrl+c`" is **not reproduced** on the local executor. `ctrl+c` stops VUs
*faster* than `gracefulStop` (≈45–51 ms here), and at natural end nothing runs past
`maxEndTime = startTime + regularDuration + gracefulStop`. The only way a VU legitimately
runs *up to* (never beyond) `gracefulStop` after the regular duration is the *natural*
end shown in 3c/3d — which is by design, not an overrun.

---

## Section 4 — Symptom 4: execution-segment imbalance & sum vs max

**What the user reported.** Three instances split by execution segments that
"should split the load evenly," yet "one instance consistently shows more VUs than
the others at the same timestamp, and if I sum them up they exceed my configured
maximum."

**Observation script** (`/tmp/k6obs/seg.js`): a steady VU plateau at `TARGET`
(parametrized via `-e TARGET=<n>`), `gracefulRampDown: '0s'`:

```js
import { sleep } from 'k6';
import exec from 'k6/execution';
const TARGET = parseInt(__ENV.TARGET || '10');
export const options = { scenarios: { seg: {
  executor: 'ramping-vus', startVUs: 0,
  stages: [ { target: TARGET, duration: '1s' }, { target: TARGET, duration: '4s' } ],
  gracefulRampDown: '0s',
}}};
export default function () {
  console.log(`t=${Math.round(exec.instance.currentTestRunDuration)}ms vusActive=${exec.instance.vusActive}`);
  sleep(1);
}
```

### 4a — Cross-product for `TARGET=10` (single machine + three segments)

The canonical three-instance CLI uses `--execution-segment` with the shared
`--execution-segment-sequence "0,1/3,2/3,1"`. The **complete unedited** single-machine
baseline output is shown here; the three per-segment `TARGET=10` runs (with their own
complete output) are in **4c**, where they are launched concurrently.

**Single machine, `TARGET=10` — complete unedited output** (`$ /tmp/k6bin run --no-color -e TARGET=10 /tmp/k6obs/seg.js`):

```

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6obs/seg.js
        output: -

     scenarios: (100.00%) 1 scenario, 10 max VUs, 35s max duration (incl. graceful stop):
              * seg: Up to 10 looping VUs for 5s over 2 stages (gracefulRampDown: 0s, gracefulStop: 30s)

time="2026-07-06T23:09:56Z" level=info msg="t=100ms vusActive=1" source=console
time="2026-07-06T23:09:56Z" level=info msg="t=201ms vusActive=2" source=console
time="2026-07-06T23:09:56Z" level=info msg="t=300ms vusActive=3" source=console
time="2026-07-06T23:09:56Z" level=info msg="t=401ms vusActive=4" source=console
time="2026-07-06T23:09:56Z" level=info msg="t=500ms vusActive=5" source=console
time="2026-07-06T23:09:56Z" level=info msg="t=601ms vusActive=6" source=console
time="2026-07-06T23:09:56Z" level=info msg="t=700ms vusActive=7" source=console
time="2026-07-06T23:09:57Z" level=info msg="t=801ms vusActive=8" source=console
time="2026-07-06T23:09:57Z" level=info msg="t=900ms vusActive=9" source=console

running (01.0s), 09/10 VUs, 0 complete and 0 interrupted iterations
seg    [  20% ] 09/10 VUs  1.0s/5.0s
time="2026-07-06T23:09:57Z" level=info msg="t=1001ms vusActive=10" source=console
time="2026-07-06T23:09:57Z" level=info msg="t=1101ms vusActive=10" source=console
time="2026-07-06T23:09:57Z" level=info msg="t=1201ms vusActive=10" source=console
time="2026-07-06T23:09:57Z" level=info msg="t=1301ms vusActive=10" source=console
time="2026-07-06T23:09:57Z" level=info msg="t=1401ms vusActive=10" source=console
time="2026-07-06T23:09:57Z" level=info msg="t=1501ms vusActive=10" source=console
time="2026-07-06T23:09:57Z" level=info msg="t=1601ms vusActive=10" source=console
time="2026-07-06T23:09:57Z" level=info msg="t=1701ms vusActive=10" source=console
time="2026-07-06T23:09:58Z" level=info msg="t=1802ms vusActive=10" source=console
time="2026-07-06T23:09:58Z" level=info msg="t=1901ms vusActive=10" source=console

running (02.0s), 10/10 VUs, 9 complete and 0 interrupted iterations
seg    [  40% ] 10/10 VUs  2.0s/5.0s
time="2026-07-06T23:09:58Z" level=info msg="t=2002ms vusActive=10" source=console
time="2026-07-06T23:09:58Z" level=info msg="t=2102ms vusActive=10" source=console
time="2026-07-06T23:09:58Z" level=info msg="t=2202ms vusActive=10" source=console
time="2026-07-06T23:09:58Z" level=info msg="t=2302ms vusActive=10" source=console
time="2026-07-06T23:09:58Z" level=info msg="t=2402ms vusActive=10" source=console
time="2026-07-06T23:09:58Z" level=info msg="t=2501ms vusActive=10" source=console
time="2026-07-06T23:09:58Z" level=info msg="t=2602ms vusActive=10" source=console
time="2026-07-06T23:09:58Z" level=info msg="t=2701ms vusActive=10" source=console
time="2026-07-06T23:09:59Z" level=info msg="t=2802ms vusActive=10" source=console
time="2026-07-06T23:09:59Z" level=info msg="t=2901ms vusActive=10" source=console

running (03.0s), 10/10 VUs, 19 complete and 0 interrupted iterations
seg    [  60% ] 10/10 VUs  3.0s/5.0s
time="2026-07-06T23:09:59Z" level=info msg="t=3003ms vusActive=10" source=console
time="2026-07-06T23:09:59Z" level=info msg="t=3104ms vusActive=10" source=console
time="2026-07-06T23:09:59Z" level=info msg="t=3203ms vusActive=10" source=console
time="2026-07-06T23:09:59Z" level=info msg="t=3303ms vusActive=10" source=console
time="2026-07-06T23:09:59Z" level=info msg="t=3403ms vusActive=10" source=console
time="2026-07-06T23:09:59Z" level=info msg="t=3501ms vusActive=10" source=console
time="2026-07-06T23:09:59Z" level=info msg="t=3602ms vusActive=10" source=console
time="2026-07-06T23:09:59Z" level=info msg="t=3702ms vusActive=10" source=console
time="2026-07-06T23:10:00Z" level=info msg="t=3803ms vusActive=10" source=console
time="2026-07-06T23:10:00Z" level=info msg="t=3902ms vusActive=10" source=console

running (04.0s), 10/10 VUs, 29 complete and 0 interrupted iterations
seg    [  80% ] 10/10 VUs  4.0s/5.0s
time="2026-07-06T23:10:00Z" level=info msg="t=4004ms vusActive=10" source=console
time="2026-07-06T23:10:00Z" level=info msg="t=4104ms vusActive=10" source=console
time="2026-07-06T23:10:00Z" level=info msg="t=4204ms vusActive=10" source=console
time="2026-07-06T23:10:00Z" level=info msg="t=4304ms vusActive=10" source=console
time="2026-07-06T23:10:00Z" level=info msg="t=4403ms vusActive=10" source=console
time="2026-07-06T23:10:00Z" level=info msg="t=4502ms vusActive=10" source=console
time="2026-07-06T23:10:00Z" level=info msg="t=4603ms vusActive=10" source=console
time="2026-07-06T23:10:00Z" level=info msg="t=4702ms vusActive=10" source=console
time="2026-07-06T23:10:01Z" level=info msg="t=4804ms vusActive=10" source=console
time="2026-07-06T23:10:01Z" level=info msg="t=4903ms vusActive=10" source=console

running (05.0s), 10/10 VUs, 39 complete and 0 interrupted iterations
seg    [ 100% ] 10/10 VUs  5.0s/5.0s

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=1s min=1s med=1s max=1s p(90)=1s p(95)=1s
     iterations...........: 49  8.299344/s
     vus..................: 10  min=9      max=10
     vus_max..............: 10  min=10     max=10


running (05.9s), 00/10 VUs, 49 complete and 0 interrupted iterations
seg  ✓ [ 100% ] 00/10 VUs  5s
```

**Derived cross-product** (from `$ grep vus_max` over the single-machine run above and
the three concurrent per-segment runs in 4c — placed after the raw output, as a
summary of it):

| instance | `--execution-segment` | `vus_max` |
|---|---|---|
| single machine | (none) | `10` |
| A | `0:1/3` | `4` |
| B | `1/3:2/3` | `3` |
| C | `2/3:1` | `3` |

`single = 10`; segments `0:1/3 → 4`, `1/3:2/3 → 3`, `2/3:1 → 3`;
**SUM = 4 + 3 + 3 = 10 = single-machine max.**

### 4b — Cross-product for `TARGET=9`

All four runs are shown with **complete unedited output**.

**Single machine, `TARGET=9`** (`$ /tmp/k6bin run --no-color -e TARGET=9 /tmp/k6obs/seg.js`):

```

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6obs/seg.js
        output: -

     scenarios: (100.00%) 1 scenario, 9 max VUs, 35s max duration (incl. graceful stop):
              * seg: Up to 9 looping VUs for 5s over 2 stages (gracefulRampDown: 0s, gracefulStop: 30s)

time="2026-07-06T23:11:06Z" level=info msg="t=112ms vusActive=1" source=console
time="2026-07-06T23:11:06Z" level=info msg="t=223ms vusActive=2" source=console
time="2026-07-06T23:11:06Z" level=info msg="t=334ms vusActive=3" source=console
time="2026-07-06T23:11:06Z" level=info msg="t=445ms vusActive=4" source=console
time="2026-07-06T23:11:06Z" level=info msg="t=557ms vusActive=5" source=console
time="2026-07-06T23:11:06Z" level=info msg="t=667ms vusActive=6" source=console
time="2026-07-06T23:11:06Z" level=info msg="t=779ms vusActive=7" source=console
time="2026-07-06T23:11:06Z" level=info msg="t=890ms vusActive=8" source=console

running (01.0s), 8/9 VUs, 0 complete and 0 interrupted iterations
seg    [  20% ] 8/9 VUs  1.0s/5.0s
time="2026-07-06T23:11:06Z" level=info msg="t=1000ms vusActive=9" source=console
time="2026-07-06T23:11:07Z" level=info msg="t=1113ms vusActive=9" source=console
time="2026-07-06T23:11:07Z" level=info msg="t=1223ms vusActive=9" source=console
time="2026-07-06T23:11:07Z" level=info msg="t=1335ms vusActive=9" source=console
time="2026-07-06T23:11:07Z" level=info msg="t=1446ms vusActive=9" source=console
time="2026-07-06T23:11:07Z" level=info msg="t=1558ms vusActive=9" source=console
time="2026-07-06T23:11:07Z" level=info msg="t=1668ms vusActive=9" source=console
time="2026-07-06T23:11:07Z" level=info msg="t=1780ms vusActive=9" source=console
time="2026-07-06T23:11:07Z" level=info msg="t=1890ms vusActive=9" source=console

running (02.0s), 9/9 VUs, 8 complete and 0 interrupted iterations
seg    [  40% ] 9/9 VUs  2.0s/5.0s
time="2026-07-06T23:11:07Z" level=info msg="t=2002ms vusActive=9" source=console
time="2026-07-06T23:11:08Z" level=info msg="t=2114ms vusActive=9" source=console
time="2026-07-06T23:11:08Z" level=info msg="t=2224ms vusActive=9" source=console
time="2026-07-06T23:11:08Z" level=info msg="t=2335ms vusActive=9" source=console
time="2026-07-06T23:11:08Z" level=info msg="t=2447ms vusActive=9" source=console
time="2026-07-06T23:11:08Z" level=info msg="t=2559ms vusActive=9" source=console
time="2026-07-06T23:11:08Z" level=info msg="t=2669ms vusActive=9" source=console
time="2026-07-06T23:11:08Z" level=info msg="t=2781ms vusActive=9" source=console
time="2026-07-06T23:11:08Z" level=info msg="t=2891ms vusActive=9" source=console

running (03.0s), 9/9 VUs, 17 complete and 0 interrupted iterations
seg    [  60% ] 9/9 VUs  3.0s/5.0s
time="2026-07-06T23:11:08Z" level=info msg="t=3002ms vusActive=9" source=console
time="2026-07-06T23:11:09Z" level=info msg="t=3115ms vusActive=9" source=console
time="2026-07-06T23:11:09Z" level=info msg="t=3225ms vusActive=9" source=console
time="2026-07-06T23:11:09Z" level=info msg="t=3337ms vusActive=9" source=console
time="2026-07-06T23:11:09Z" level=info msg="t=3448ms vusActive=9" source=console
time="2026-07-06T23:11:09Z" level=info msg="t=3560ms vusActive=9" source=console
time="2026-07-06T23:11:09Z" level=info msg="t=3670ms vusActive=9" source=console
time="2026-07-06T23:11:09Z" level=info msg="t=3782ms vusActive=9" source=console
time="2026-07-06T23:11:09Z" level=info msg="t=3892ms vusActive=9" source=console

running (04.0s), 9/9 VUs, 26 complete and 0 interrupted iterations
seg    [  80% ] 9/9 VUs  4.0s/5.0s
time="2026-07-06T23:11:09Z" level=info msg="t=4003ms vusActive=9" source=console
time="2026-07-06T23:11:10Z" level=info msg="t=4116ms vusActive=9" source=console
time="2026-07-06T23:11:10Z" level=info msg="t=4226ms vusActive=9" source=console
time="2026-07-06T23:11:10Z" level=info msg="t=4337ms vusActive=9" source=console
time="2026-07-06T23:11:10Z" level=info msg="t=4448ms vusActive=9" source=console
time="2026-07-06T23:11:10Z" level=info msg="t=4561ms vusActive=9" source=console
time="2026-07-06T23:11:10Z" level=info msg="t=4671ms vusActive=9" source=console
time="2026-07-06T23:11:10Z" level=info msg="t=4783ms vusActive=9" source=console
time="2026-07-06T23:11:10Z" level=info msg="t=4893ms vusActive=9" source=console

running (05.0s), 9/9 VUs, 35 complete and 0 interrupted iterations
seg    [ 100% ] 9/9 VUs  5.0s/5.0s

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=1s min=1s med=1s max=1s p(90)=1s p(95)=1s
     iterations...........: 44  7.464691/s
     vus..................: 9   min=8      max=9
     vus_max..............: 9   min=9      max=9


running (05.9s), 0/9 VUs, 44 complete and 0 interrupted iterations
seg  ✓ [ 100% ] 0/9 VUs  5s
```

**Segment `0:1/3`, `TARGET=9`** (`$ /tmp/k6bin run --no-color -e TARGET=9 --execution-segment "0:1/3" --execution-segment-sequence "0,1/3,2/3,1" /tmp/k6obs/seg.js`):

```

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6obs/seg.js
        output: -

     scenarios: (33.33%) 1 scenario, 3 max VUs, 35s max duration (incl. graceful stop):
              * seg: Up to 3 looping VUs for 5s over 2 stages (gracefulRampDown: 0s, gracefulStop: 30s)

time="2026-07-06T23:11:11Z" level=info msg="t=112ms vusActive=1" source=console
time="2026-07-06T23:11:12Z" level=info msg="t=445ms vusActive=2" source=console
time="2026-07-06T23:11:12Z" level=info msg="t=778ms vusActive=3" source=console

running (01.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
seg    [  20% ] 3/3 VUs  1.0s/5.0s
time="2026-07-06T23:11:12Z" level=info msg="t=1112ms vusActive=3" source=console
time="2026-07-06T23:11:13Z" level=info msg="t=1446ms vusActive=3" source=console
time="2026-07-06T23:11:13Z" level=info msg="t=1779ms vusActive=3" source=console

running (02.0s), 3/3 VUs, 3 complete and 0 interrupted iterations
seg    [  40% ] 3/3 VUs  2.0s/5.0s
time="2026-07-06T23:11:13Z" level=info msg="t=2112ms vusActive=3" source=console
time="2026-07-06T23:11:14Z" level=info msg="t=2447ms vusActive=3" source=console
time="2026-07-06T23:11:14Z" level=info msg="t=2780ms vusActive=3" source=console

running (03.0s), 3/3 VUs, 6 complete and 0 interrupted iterations
seg    [  60% ] 3/3 VUs  3.0s/5.0s
time="2026-07-06T23:11:14Z" level=info msg="t=3113ms vusActive=3" source=console
time="2026-07-06T23:11:15Z" level=info msg="t=3448ms vusActive=3" source=console
time="2026-07-06T23:11:15Z" level=info msg="t=3781ms vusActive=3" source=console

running (04.0s), 3/3 VUs, 9 complete and 0 interrupted iterations
seg    [  80% ] 3/3 VUs  4.0s/5.0s
time="2026-07-06T23:11:15Z" level=info msg="t=4114ms vusActive=3" source=console
time="2026-07-06T23:11:16Z" level=info msg="t=4448ms vusActive=3" source=console
time="2026-07-06T23:11:16Z" level=info msg="t=4782ms vusActive=3" source=console

running (05.0s), 3/3 VUs, 12 complete and 0 interrupted iterations
seg    [ 100% ] 3/3 VUs  5.0s/5.0s

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=1s min=1s med=1s max=1s p(90)=1s p(95)=1s
     iterations...........: 15  2.593987/s
     vus..................: 3   min=3      max=3
     vus_max..............: 3   min=3      max=3


running (05.8s), 0/3 VUs, 15 complete and 0 interrupted iterations
seg  ✓ [ 100% ] 0/3 VUs  5s
```

**Segment `1/3:2/3`, `TARGET=9`** (`$ /tmp/k6bin run --no-color -e TARGET=9 --execution-segment "1/3:2/3" --execution-segment-sequence "0,1/3,2/3,1" /tmp/k6obs/seg.js`):

```

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6obs/seg.js
        output: -

     scenarios: (33.33%) 1 scenario, 3 max VUs, 35s max duration (incl. graceful stop):
              * seg: Up to 3 looping VUs for 5s over 2 stages (gracefulRampDown: 0s, gracefulStop: 30s)

time="2026-07-06T23:11:17Z" level=info msg="t=223ms vusActive=1" source=console
time="2026-07-06T23:11:18Z" level=info msg="t=557ms vusActive=2" source=console
time="2026-07-06T23:11:18Z" level=info msg="t=889ms vusActive=3" source=console

running (01.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
seg    [  20% ] 3/3 VUs  1.0s/5.0s
time="2026-07-06T23:11:18Z" level=info msg="t=1224ms vusActive=3" source=console
time="2026-07-06T23:11:19Z" level=info msg="t=1558ms vusActive=3" source=console
time="2026-07-06T23:11:19Z" level=info msg="t=1890ms vusActive=3" source=console

running (02.0s), 3/3 VUs, 3 complete and 0 interrupted iterations
seg    [  40% ] 3/3 VUs  2.0s/5.0s
time="2026-07-06T23:11:19Z" level=info msg="t=2225ms vusActive=3" source=console
time="2026-07-06T23:11:20Z" level=info msg="t=2558ms vusActive=3" source=console
time="2026-07-06T23:11:20Z" level=info msg="t=2890ms vusActive=3" source=console

running (03.0s), 3/3 VUs, 6 complete and 0 interrupted iterations
seg    [  60% ] 3/3 VUs  3.0s/5.0s
time="2026-07-06T23:11:20Z" level=info msg="t=3226ms vusActive=3" source=console
time="2026-07-06T23:11:21Z" level=info msg="t=3559ms vusActive=3" source=console
time="2026-07-06T23:11:21Z" level=info msg="t=3892ms vusActive=3" source=console

running (04.0s), 3/3 VUs, 9 complete and 0 interrupted iterations
seg    [  80% ] 3/3 VUs  4.0s/5.0s
time="2026-07-06T23:11:21Z" level=info msg="t=4227ms vusActive=3" source=console
time="2026-07-06T23:11:22Z" level=info msg="t=4560ms vusActive=3" source=console
time="2026-07-06T23:11:22Z" level=info msg="t=4893ms vusActive=3" source=console

running (05.0s), 3/3 VUs, 12 complete and 0 interrupted iterations
seg    [ 100% ] 3/3 VUs  5s

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=1s min=1s med=1s max=1s p(90)=1s p(95)=1s
     iterations...........: 15  2.545062/s
     vus..................: 3   min=3      max=3
     vus_max..............: 3   min=3      max=3


running (05.9s), 0/3 VUs, 15 complete and 0 interrupted iterations
seg  ✓ [ 100% ] 0/3 VUs  5s
```

**Segment `2/3:1`, `TARGET=9`** (`$ /tmp/k6bin run --no-color -e TARGET=9 --execution-segment "2/3:1" --execution-segment-sequence "0,1/3,2/3,1" /tmp/k6obs/seg.js`):

```

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6obs/seg.js
        output: -

     scenarios: (33.33%) 1 scenario, 3 max VUs, 35s max duration (incl. graceful stop):
              * seg: Up to 3 looping VUs for 5s over 2 stages (gracefulRampDown: 0s, gracefulStop: 30s)

time="2026-07-06T23:11:24Z" level=info msg="t=334ms vusActive=1" source=console
time="2026-07-06T23:11:24Z" level=info msg="t=667ms vusActive=2" source=console

running (01.0s), 2/3 VUs, 0 complete and 0 interrupted iterations
seg    [  20% ] 2/3 VUs  1.0s/5.0s
time="2026-07-06T23:11:24Z" level=info msg="t=1000ms vusActive=3" source=console
time="2026-07-06T23:11:25Z" level=info msg="t=1335ms vusActive=3" source=console
time="2026-07-06T23:11:25Z" level=info msg="t=1669ms vusActive=3" source=console

running (02.0s), 3/3 VUs, 2 complete and 0 interrupted iterations
seg    [  40% ] 3/3 VUs  2.0s/5.0s
time="2026-07-06T23:11:25Z" level=info msg="t=2001ms vusActive=3" source=console
time="2026-07-06T23:11:26Z" level=info msg="t=2336ms vusActive=3" source=console
time="2026-07-06T23:11:26Z" level=info msg="t=2670ms vusActive=3" source=console

running (03.0s), 3/3 VUs, 5 complete and 0 interrupted iterations
seg    [  60% ] 3/3 VUs  3.0s/5.0s
time="2026-07-06T23:11:26Z" level=info msg="t=3002ms vusActive=3" source=console
time="2026-07-06T23:11:27Z" level=info msg="t=3337ms vusActive=3" source=console
time="2026-07-06T23:11:27Z" level=info msg="t=3670ms vusActive=3" source=console

running (04.0s), 3/3 VUs, 8 complete and 0 interrupted iterations
seg    [  80% ] 3/3 VUs  4.0s/5.0s
time="2026-07-06T23:11:27Z" level=info msg="t=4003ms vusActive=3" source=console
time="2026-07-06T23:11:28Z" level=info msg="t=4338ms vusActive=3" source=console
time="2026-07-06T23:11:28Z" level=info msg="t=4671ms vusActive=3" source=console

running (05.0s), 3/3 VUs, 11 complete and 0 interrupted iterations
seg    [ 100% ] 3/3 VUs  5.0s/5.0s

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=1s min=1s med=1s max=1s p(90)=1s p(95)=1s
     iterations...........: 14  2.468277/s
     vus..................: 3   min=2      max=3
     vus_max..............: 3   min=3      max=3


running (05.7s), 0/3 VUs, 14 complete and 0 interrupted iterations
seg  ✓ [ 100% ] 0/3 VUs  5s
```

**Derived cross-product** (`$ grep vus_max`, summarizing the four raw runs above):

| instance | `--execution-segment` | `vus_max` |
|---|---|---|
| single machine | (none) | `9` |
| A | `0:1/3` | `3` |
| B | `1/3:2/3` | `3` |
| C | `2/3:1` | `3` |

`single = 9`; each segment `→ 3`; **SUM = 3 + 3 + 3 = 9 = single-machine max**
(evenly divisible, so no remainder and no "extra" instance).

### 4c — Identical-timestamp `vusActive` (three segments run concurrently)

To answer "one instance shows more … at the same timestamp," the three `TARGET=10`
segments were run **concurrently** (background `&` + `wait`), each redirected to its own
file:

```
$ SEQ="0,1/3,2/3,1"
$ /tmp/k6bin run --no-color -e TARGET=10 --execution-segment "0:1/3"   --execution-segment-sequence "$SEQ" /tmp/k6obs/seg.js > /tmp/k6obs/segc_A.txt 2>&1 &
$ /tmp/k6bin run --no-color -e TARGET=10 --execution-segment "1/3:2/3" --execution-segment-sequence "$SEQ" /tmp/k6obs/seg.js > /tmp/k6obs/segc_B.txt 2>&1 &
$ /tmp/k6bin run --no-color -e TARGET=10 --execution-segment "2/3:1"   --execution-segment-sequence "$SEQ" /tmp/k6obs/seg.js > /tmp/k6obs/segc_C.txt 2>&1 &
$ wait
```

**Instance A (`0:1/3`) — complete unedited output** (`$ cat /tmp/k6obs/segc_A.txt`):

```

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6obs/seg.js
        output: -

     scenarios: (33.33%) 1 scenario, 4 max VUs, 35s max duration (incl. graceful stop):
              * seg: Up to 4 looping VUs for 5s over 2 stages (gracefulRampDown: 0s, gracefulStop: 30s)

time="2026-07-06T23:11:29Z" level=info msg="t=101ms vusActive=1" source=console
time="2026-07-06T23:11:29Z" level=info msg="t=401ms vusActive=2" source=console
time="2026-07-06T23:11:30Z" level=info msg="t=701ms vusActive=3" source=console

running (01.0s), 3/4 VUs, 0 complete and 0 interrupted iterations
seg    [  20% ] 3/4 VUs  1.0s/5.0s
time="2026-07-06T23:11:30Z" level=info msg="t=1001ms vusActive=4" source=console
time="2026-07-06T23:11:30Z" level=info msg="t=1102ms vusActive=4" source=console
time="2026-07-06T23:11:30Z" level=info msg="t=1402ms vusActive=4" source=console
time="2026-07-06T23:11:31Z" level=info msg="t=1701ms vusActive=4" source=console

running (02.0s), 4/4 VUs, 3 complete and 0 interrupted iterations
seg    [  40% ] 4/4 VUs  2.0s/5.0s
time="2026-07-06T23:11:31Z" level=info msg="t=2002ms vusActive=4" source=console
time="2026-07-06T23:11:31Z" level=info msg="t=2102ms vusActive=4" source=console
time="2026-07-06T23:11:31Z" level=info msg="t=2403ms vusActive=4" source=console
time="2026-07-06T23:11:32Z" level=info msg="t=2702ms vusActive=4" source=console

running (03.0s), 4/4 VUs, 7 complete and 0 interrupted iterations
seg    [  60% ] 4/4 VUs  3.0s/5.0s
time="2026-07-06T23:11:32Z" level=info msg="t=3002ms vusActive=4" source=console
time="2026-07-06T23:11:32Z" level=info msg="t=3103ms vusActive=4" source=console
time="2026-07-06T23:11:32Z" level=info msg="t=3404ms vusActive=4" source=console
time="2026-07-06T23:11:33Z" level=info msg="t=3703ms vusActive=4" source=console

running (04.0s), 4/4 VUs, 11 complete and 0 interrupted iterations
seg    [  80% ] 4/4 VUs  4.0s/5.0s
time="2026-07-06T23:11:33Z" level=info msg="t=4002ms vusActive=4" source=console
time="2026-07-06T23:11:33Z" level=info msg="t=4104ms vusActive=4" source=console
time="2026-07-06T23:11:33Z" level=info msg="t=4405ms vusActive=4" source=console
time="2026-07-06T23:11:34Z" level=info msg="t=4703ms vusActive=4" source=console

running (05.0s), 4/4 VUs, 15 complete and 0 interrupted iterations
seg    [ 100% ] 4/4 VUs  5.0s/5.0s

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=1s min=1s med=1s max=1s p(90)=1s p(95)=1s
     iterations...........: 19  3.331127/s
     vus..................: 4   min=3      max=4
     vus_max..............: 4   min=4      max=4


running (05.7s), 0/4 VUs, 19 complete and 0 interrupted iterations
seg  ✓ [ 100% ] 0/4 VUs  5s
```

**Instance B (`1/3:2/3`) — complete unedited output** (`$ cat /tmp/k6obs/segc_B.txt`):

```

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6obs/seg.js
        output: -

     scenarios: (33.33%) 1 scenario, 3 max VUs, 35s max duration (incl. graceful stop):
              * seg: Up to 3 looping VUs for 5s over 2 stages (gracefulRampDown: 0s, gracefulStop: 30s)

time="2026-07-06T23:11:29Z" level=warning msg="Error from API server" error="listen tcp 127.0.0.1:6565: bind: address already in use"
time="2026-07-06T23:11:29Z" level=info msg="t=201ms vusActive=1" source=console
time="2026-07-06T23:11:29Z" level=info msg="t=500ms vusActive=2" source=console
time="2026-07-06T23:11:30Z" level=info msg="t=801ms vusActive=3" source=console

running (01.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
seg    [  20% ] 3/3 VUs  1.0s/5.0s
time="2026-07-06T23:11:30Z" level=info msg="t=1202ms vusActive=3" source=console
time="2026-07-06T23:11:30Z" level=info msg="t=1501ms vusActive=3" source=console
time="2026-07-06T23:11:31Z" level=info msg="t=1801ms vusActive=3" source=console

running (02.0s), 3/3 VUs, 3 complete and 0 interrupted iterations
seg    [  40% ] 3/3 VUs  2.0s/5.0s
time="2026-07-06T23:11:31Z" level=info msg="t=2202ms vusActive=3" source=console
time="2026-07-06T23:11:31Z" level=info msg="t=2502ms vusActive=3" source=console
time="2026-07-06T23:11:32Z" level=info msg="t=2802ms vusActive=3" source=console

running (03.0s), 3/3 VUs, 6 complete and 0 interrupted iterations
seg    [  60% ] 3/3 VUs  3.0s/5.0s
time="2026-07-06T23:11:32Z" level=info msg="t=3203ms vusActive=3" source=console
time="2026-07-06T23:11:32Z" level=info msg="t=3503ms vusActive=3" source=console
time="2026-07-06T23:11:33Z" level=info msg="t=3803ms vusActive=3" source=console

running (04.0s), 3/3 VUs, 9 complete and 0 interrupted iterations
seg    [  80% ] 3/3 VUs  4.0s/5.0s
time="2026-07-06T23:11:33Z" level=info msg="t=4204ms vusActive=3" source=console
time="2026-07-06T23:11:33Z" level=info msg="t=4503ms vusActive=3" source=console
time="2026-07-06T23:11:34Z" level=info msg="t=4804ms vusActive=3" source=console

running (05.0s), 3/3 VUs, 12 complete and 0 interrupted iterations
seg    [ 100% ] 3/3 VUs  5.0s/5.0s

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=1s min=1s med=1s max=1s p(90)=1s p(95)=1s
     iterations...........: 15  2.584032/s
     vus..................: 3   min=3      max=3
     vus_max..............: 3   min=3      max=3


running (05.8s), 0/3 VUs, 15 complete and 0 interrupted iterations
seg  ✓ [ 100% ] 0/3 VUs  5s
```

**Instance C (`2/3:1`) — complete unedited output** (`$ cat /tmp/k6obs/segc_C.txt`):

```

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6obs/seg.js
        output: -

     scenarios: (33.33%) 1 scenario, 3 max VUs, 35s max duration (incl. graceful stop):
              * seg: Up to 3 looping VUs for 5s over 2 stages (gracefulRampDown: 0s, gracefulStop: 30s)

time="2026-07-06T23:11:29Z" level=warning msg="Error from API server" error="listen tcp 127.0.0.1:6565: bind: address already in use"
time="2026-07-06T23:11:29Z" level=info msg="t=301ms vusActive=1" source=console
time="2026-07-06T23:11:29Z" level=info msg="t=600ms vusActive=2" source=console
time="2026-07-06T23:11:30Z" level=info msg="t=901ms vusActive=3" source=console

running (01.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
seg    [  20% ] 3/3 VUs  1.0s/5.0s
time="2026-07-06T23:11:30Z" level=info msg="t=1301ms vusActive=3" source=console
time="2026-07-06T23:11:30Z" level=info msg="t=1601ms vusActive=3" source=console
time="2026-07-06T23:11:31Z" level=info msg="t=1902ms vusActive=3" source=console

running (02.0s), 3/3 VUs, 3 complete and 0 interrupted iterations
seg    [  40% ] 3/3 VUs  2.0s/5.0s
time="2026-07-06T23:11:31Z" level=info msg="t=2302ms vusActive=3" source=console
time="2026-07-06T23:11:31Z" level=info msg="t=2601ms vusActive=3" source=console
time="2026-07-06T23:11:32Z" level=info msg="t=2903ms vusActive=3" source=console

running (03.0s), 3/3 VUs, 6 complete and 0 interrupted iterations
seg    [  60% ] 3/3 VUs  3.0s/5.0s
time="2026-07-06T23:11:32Z" level=info msg="t=3303ms vusActive=3" source=console
time="2026-07-06T23:11:32Z" level=info msg="t=3602ms vusActive=3" source=console
time="2026-07-06T23:11:33Z" level=info msg="t=3904ms vusActive=3" source=console

running (04.0s), 3/3 VUs, 9 complete and 0 interrupted iterations
seg    [  80% ] 3/3 VUs  4.0s/5.0s
time="2026-07-06T23:11:33Z" level=info msg="t=4304ms vusActive=3" source=console
time="2026-07-06T23:11:33Z" level=info msg="t=4603ms vusActive=3" source=console
time="2026-07-06T23:11:34Z" level=info msg="t=4905ms vusActive=3" source=console

running (05.0s), 3/3 VUs, 12 complete and 0 interrupted iterations
seg    [ 100% ] 3/3 VUs  5.0s/5.0s

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=1s min=1s med=1s max=1s p(90)=1s p(95)=1s
     iterations...........: 15  2.540016/s
     vus..................: 3   min=3      max=3
     vus_max..............: 3   min=3      max=3


running (05.9s), 0/3 VUs, 15 complete and 0 interrupted iterations
seg  ✓ [ 100% ] 0/3 VUs  5s
```

**Additional analysis (derived from the complete output above).** Extracting just the
`t=…ms vusActive=…` samples from each instance with `grep -oE`:

```
$ grep -oE 't=[0-9]+ms vusActive=[0-9]+' /tmp/k6obs/segc_A.txt
t=101ms vusActive=1
t=401ms vusActive=2
t=701ms vusActive=3
t=1001ms vusActive=4
t=1102ms vusActive=4
t=1402ms vusActive=4
t=1701ms vusActive=4
t=2002ms vusActive=4
t=2102ms vusActive=4
t=2403ms vusActive=4
t=2702ms vusActive=4
t=3002ms vusActive=4
t=3103ms vusActive=4
t=3404ms vusActive=4
t=3703ms vusActive=4
t=4002ms vusActive=4
t=4104ms vusActive=4
t=4405ms vusActive=4
t=4703ms vusActive=4

$ grep -oE 't=[0-9]+ms vusActive=[0-9]+' /tmp/k6obs/segc_B.txt
t=201ms vusActive=1
t=500ms vusActive=2
t=801ms vusActive=3
t=1202ms vusActive=3
t=1501ms vusActive=3
t=1801ms vusActive=3
t=2202ms vusActive=3
t=2502ms vusActive=3
t=2802ms vusActive=3
t=3203ms vusActive=3
t=3503ms vusActive=3
t=3803ms vusActive=3
t=4204ms vusActive=3
t=4503ms vusActive=3
t=4804ms vusActive=3

$ grep -oE 't=[0-9]+ms vusActive=[0-9]+' /tmp/k6obs/segc_C.txt
t=301ms vusActive=1
t=600ms vusActive=2
t=901ms vusActive=3
t=1301ms vusActive=3
t=1601ms vusActive=3
t=1902ms vusActive=3
t=2302ms vusActive=3
t=2601ms vusActive=3
t=2903ms vusActive=3
t=3303ms vusActive=3
t=3602ms vusActive=3
t=3904ms vusActive=3
t=4304ms vusActive=3
t=4603ms vusActive=3
t=4905ms vusActive=3
```

At steady state (e.g. around t≈2000 ms — A `t=2002ms vusActive=4`, B `t=2202ms
vusActive=3`, C `t=2302ms vusActive=3`) the three instances read **A=4, B=3, C=3**
simultaneously — **sum = 10 = the configured maximum**. The concurrent run reproduced
the same per-segment `vus_max` (`4`, `3`, `3`) as the single-machine cross-product in
4a, so the split is deterministic across two independent runs.

**Cause → effect.**

- Segment scaling is done by `SegmentedIndex` (`lib/execution_segment.go:L768`;
  `NewSegmentedIndex` `:L776`, `Next` `:L782`, `Prev` `:L795`, `GoTo` `:L808`) via
  `ExecutionTuple.ScaleInt64` (`:L734`) and
  `ExecutionSegmentSequenceWrapper.ScaleInt64` (`:L580`). Indivisible VUs are *striped*
  across the sequence so that **summing the per-segment values reproduces the
  single-machine total exactly** (the k6 issue #997 invariant — see **External
  references**). The first segment in
  the sequence takes the remainder, which is why `0:1/3` shows `4` while the others
  show `3` when the total is `10`.
- This is validated independently by `TestSumRandomSegmentSequenceMatchesNoSegment`
  (`lib/executor/ramping_vus_test.go:L1112`), which passes cleanly under `-race`
  (Section 6, Probe 3).
- (inferred, from k6 issue #1308 — see **External references**) for `ramping-vus` only the *number of VUs* is
  partitioned across segments — consistent with observing only VU-count striping here.

**Verdict:** "one instance consistently shows more" is **true and expected** — the
first segment deterministically receives the extra indivisible VU. But "the sum
exceeds my configured maximum" is **false**: the per-segment values **sum exactly to**
the single-machine max (`10` and `9` here) and never exceed it. No VUs are lost or
duplicated.

---

## Section 5 — Symptom 5: VU buffer leak hypothesis → no leak, buffer conserved

**What the user reported.** "Maybe the VU buffer is leaking somehow?"

The "buffer" is the shared `vus chan InitializedVU` in the execution state
(`lib/execution.go:L106`). VUs are taken from it via `GetPlannedVU`
(`lib/execution.go:L471`), which retries up to `MaxRetriesGetPlannedVU = 5`
(`lib/execution.go:L29`) and logs `"Could not get a VU from the buffer for %s"`
(`lib/execution.go:L481`) if it ever *starves*. A genuine leak (a VU taken and never
returned) would eventually trigger that warning and/or grow `vus_max`.

**Observation script** (`/tmp/k6obs/buffer_stress.js`): 15 back-to-back
`0 → 10 → 0` cycles (30 stages, 400 ms each) with `gracefulRampDown: '0s'` to force
maximal get/return churn:

```js
import { sleep } from 'k6';
const cyc = [];
for (let i = 0; i < 15; i++) { cyc.push({ target: 10, duration: '400ms' }); cyc.push({ target: 0, duration: '400ms' }); }
export const options = { scenarios: { churn: {
  executor: 'ramping-vus', startVUs: 0, stages: cyc, gracefulRampDown: '0s',
}}};
export default function () { sleep(0.3); }
```

**Command and complete unedited output — RUN 1:**

```
$ /tmp/k6bin run --no-color /tmp/k6obs/buffer_stress.js > /tmp/k6obs/s5_run1.txt 2>&1
$ cat /tmp/k6obs/s5_run1.txt

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6obs/buffer_stress.js
        output: -

     scenarios: (100.00%) 1 scenario, 10 max VUs, 12s max duration (incl. graceful stop):
              * churn: Up to 10 looping VUs for 12s over 30 stages (gracefulRampDown: 0s, gracefulStop: 30s)


running (01.0s), 04/10 VUs, 8 complete and 10 interrupted iterations
churn   [   8% ] 04/10 VUs  01.0s/12.0s

running (02.0s), 09/10 VUs, 18 complete and 20 interrupted iterations
churn   [  17% ] 09/10 VUs  02.0s/12.0s

running (03.0s), 06/10 VUs, 31 complete and 34 interrupted iterations
churn   [  25% ] 06/10 VUs  03.0s/12.0s

running (04.0s), 01/10 VUs, 41 complete and 49 interrupted iterations
churn   [  33% ] 01/10 VUs  04.0s/12.0s

running (05.0s), 04/10 VUs, 49 complete and 60 interrupted iterations
churn   [  42% ] 04/10 VUs  05.0s/12.0s

running (06.0s), 09/10 VUs, 59 complete and 70 interrupted iterations
churn   [  50% ] 09/10 VUs  06.0s/12.0s

running (07.0s), 06/10 VUs, 71 complete and 84 interrupted iterations
churn   [  58% ] 06/10 VUs  07.0s/12.0s

running (08.0s), 01/10 VUs, 81 complete and 99 interrupted iterations
churn   [  67% ] 01/10 VUs  08.0s/12.0s

running (09.0s), 04/10 VUs, 89 complete and 110 interrupted iterations
churn   [  75% ] 04/10 VUs  09.0s/12.0s

running (10.0s), 09/10 VUs, 99 complete and 120 interrupted iterations
churn   [  83% ] 09/10 VUs  10.0s/12.0s

running (11.0s), 06/10 VUs, 111 complete and 134 interrupted iterations
churn   [  92% ] 06/10 VUs  11.0s/12.0s

running (12.0s), 01/10 VUs, 121 complete and 149 interrupted iterations
churn   [ 100% ] 01/10 VUs  12.0s/12.0s

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=300.56ms min=300.03ms med=300.53ms max=301.09ms p(90)=301ms p(95)=301.05ms
     iterations...........: 121 10.082502/s
     vus..................: 1   min=1       max=9 
     vus_max..............: 10  min=10      max=10


running (12.0s), 00/10 VUs, 121 complete and 150 interrupted iterations
churn ✓ [ 100% ] 01/10 VUs  12s
```

**Command and complete unedited output — RUN 2 (same unchanged input):**

```
$ /tmp/k6bin run --no-color /tmp/k6obs/buffer_stress.js > /tmp/k6obs/s5_run2.txt 2>&1
$ cat /tmp/k6obs/s5_run2.txt

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6obs/buffer_stress.js
        output: -

     scenarios: (100.00%) 1 scenario, 10 max VUs, 12s max duration (incl. graceful stop):
              * churn: Up to 10 looping VUs for 12s over 30 stages (gracefulRampDown: 0s, gracefulStop: 30s)


running (01.0s), 04/10 VUs, 8 complete and 10 interrupted iterations
churn   [   8% ] 04/10 VUs  01.0s/12.0s

running (02.0s), 09/10 VUs, 18 complete and 20 interrupted iterations
churn   [  17% ] 09/10 VUs  02.0s/12.0s

running (03.0s), 06/10 VUs, 30 complete and 34 interrupted iterations
churn   [  25% ] 06/10 VUs  03.0s/12.0s

running (04.0s), 01/10 VUs, 40 complete and 49 interrupted iterations
churn   [  33% ] 01/10 VUs  04.0s/12.0s

running (05.0s), 04/10 VUs, 48 complete and 60 interrupted iterations
churn   [  42% ] 04/10 VUs  05.0s/12.0s

running (06.0s), 09/10 VUs, 58 complete and 70 interrupted iterations
churn   [  50% ] 09/10 VUs  06.0s/12.0s

running (07.0s), 06/10 VUs, 70 complete and 84 interrupted iterations
churn   [  58% ] 06/10 VUs  07.0s/12.0s

running (08.0s), 01/10 VUs, 80 complete and 99 interrupted iterations
churn   [  67% ] 01/10 VUs  08.0s/12.0s

running (09.0s), 04/10 VUs, 88 complete and 110 interrupted iterations
churn   [  75% ] 04/10 VUs  09.0s/12.0s

running (10.0s), 09/10 VUs, 98 complete and 120 interrupted iterations
churn   [  83% ] 09/10 VUs  10.0s/12.0s

running (11.0s), 06/10 VUs, 110 complete and 134 interrupted iterations
churn   [  92% ] 06/10 VUs  11.0s/12.0s

running (12.0s), 01/10 VUs, 120 complete and 149 interrupted iterations
churn   [ 100% ] 01/10 VUs  12.0s/12.0s

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=300.52ms min=300.03ms med=300.5ms max=301.11ms p(90)=300.96ms p(95)=301.07ms
     iterations...........: 120 9.999287/s
     vus..................: 1   min=1      max=9 
     vus_max..............: 10  min=10     max=10


running (12.0s), 00/10 VUs, 120 complete and 150 interrupted iterations
churn ✓ [ 100% ] 00/10 VUs  12s
```

**Buffer-warning check (derived from the two complete runs above).** Each captured
file was grepped for the starvation warning and its corresponding error:

```
$ grep -c "Could not get a VU from the buffer" /tmp/k6obs/s5_run1.txt
0
$ grep -c "could not get a VU from the buffer in" /tmp/k6obs/s5_run1.txt
0
$ grep -c "Could not get a VU from the buffer" /tmp/k6obs/s5_run2.txt
0
$ grep -c "could not get a VU from the buffer in" /tmp/k6obs/s5_run2.txt
0
```

**Distribution across runs (same unchanged input, run twice).** The two runs did
**not** produce byte-identical iteration counts: RUN 1 ended with
**`121 complete and 150 interrupted`** iterations and RUN 2 with
**`120 complete and 150 interrupted`** — the `complete` count varies by ±1
(`120–121`) as a normal effect of sub-millisecond timing jitter in how many 300 ms
iterations happen to finish inside the fixed 12 s window. The buffer-conservation
signals, however, were **stable and identical across both runs**: **`0`** occurrences
of the starvation warning (`lib/execution.go:L481`) *and* **`0`** of the corresponding
error, and `vus_max` pinned at **`10`** (`min=10 max=10`) with no growth. The leak
verdict rests on those stable signals, not on the jittering `complete` count.

**Cause → effect.**

- Over the run there are ~270–271 get/return operations per run (`120–121` completed +
  `150` interrupted iterations, each pairing one `GetPlannedVU` with one `ReturnVU`), yet
  the starvation warning at `lib/execution.go:L481` never fired and `GetPlannedVU`
  (`lib/execution.go:L471`) never exhausted its 5 retries (`lib/execution.go:L29`).
- `vus_max` never grew beyond the configured `10`, which means no VU was taken from the
  `vus` channel (`lib/execution.go:L106`) and left unreturned. Every get is paired with
  exactly one return — the thread-safe start/stop contract at `lib/executor/vu_handle.go:L61`.
- The `150 interrupted` iterations are **not** a leak: with `gracefulRampDown: 0s`, a
  VU that is scaled down is cut off mid-iteration by design (this is the documented
  effect of `gracefulRampDown: 0`), so those iterations are counted `interrupted`
  rather than `complete`. The buffer itself is conserved.

**Verdict:** the buffer-leak hypothesis is **denied**. The VU buffer is fully
conserved across rapid ramp cycles; there is no starvation and no growth of `vus_max`.

---

## Section 6 — Question A: is there a race between the two handler goroutines? → No

**What the user asked.** "Is there a race condition between the two handler goroutines?"

**`-race` evidence (Probe 1 — VU-handle concurrency, run twice).** Verdict reported
verbatim:

```
$ go test -race -count=1 -run 'TestVUHandleRace|TestVUHandleStartStopRace|TestVUHandleSimple' -v ./lib/executor/     # RUN 1
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
=== CONT  TestVUHandleSimple/start_after_gracefulStop_finishes
=== CONT  TestVUHandleSimple/start_after_hardStop
=== CONT  TestVUHandleSimple/start_before_gracefulStop_finishes
--- PASS: TestVUHandleRace (0.18s)
--- PASS: TestVUHandleStartStopRace (0.39s)
--- PASS: TestVUHandleSimple (0.00s)
    --- PASS: TestVUHandleSimple/start_after_hardStop (1.53s)
    --- PASS: TestVUHandleSimple/start_before_gracefulStop_finishes (1.56s)
    --- PASS: TestVUHandleSimple/start_after_gracefulStop_finishes (3.10s)
PASS
ok  	go.k6.io/k6/lib/executor	4.131s
```

```
$ go test -race -count=1 -run 'TestVUHandleRace|TestVUHandleStartStopRace|TestVUHandleSimple' -v ./lib/executor/     # RUN 2
=== RUN   TestVUHandleRace
=== PAUSE TestVUHandleRace
=== RUN   TestVUHandleStartStopRace
=== PAUSE TestVUHandleStartStopRace
=== RUN   TestVUHandleSimple
=== PAUSE TestVUHandleSimple
=== CONT  TestVUHandleRace
=== CONT  TestVUHandleSimple
=== CONT  TestVUHandleStartStopRace
=== RUN   TestVUHandleSimple/start_before_gracefulStop_finishes
=== PAUSE TestVUHandleSimple/start_before_gracefulStop_finishes
=== RUN   TestVUHandleSimple/start_after_gracefulStop_finishes
=== PAUSE TestVUHandleSimple/start_after_gracefulStop_finishes
=== RUN   TestVUHandleSimple/start_after_hardStop
=== PAUSE TestVUHandleSimple/start_after_hardStop
=== CONT  TestVUHandleSimple/start_after_gracefulStop_finishes
=== CONT  TestVUHandleSimple/start_after_hardStop
=== CONT  TestVUHandleSimple/start_before_gracefulStop_finishes
--- PASS: TestVUHandleRace (0.18s)
--- PASS: TestVUHandleStartStopRace (0.40s)
--- PASS: TestVUHandleSimple (0.00s)
    --- PASS: TestVUHandleSimple/start_after_hardStop (1.53s)
    --- PASS: TestVUHandleSimple/start_before_gracefulStop_finishes (1.56s)
    --- PASS: TestVUHandleSimple/start_after_gracefulStop_finishes (3.10s)
PASS
ok  	go.k6.io/k6/lib/executor	4.134s
```

Both runs are **CLEAN**: no `WARNING: DATA RACE`, all `PASS`, `ok` in 4.131s / 4.134s.

**`-race` evidence (Probe 2 — the executor `Run` path itself).** This exercises
`RampingVUs.Run` → `iterateSteps` + `go runRemainingGracefulSteps` under the detector:

```
$ go test -race -count=1 -run 'TestRampingVUsGracefulStopWaits|TestRampingVUsGracefulStopStops|TestRampingVUsGracefulRampDown|TestRampingVUsHandleRemainingVUs|TestRampingVUsRampDownNoWobble' -v ./lib/executor/
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
=== CONT  TestRampingVUsGracefulRampDown
=== CONT  TestRampingVUsHandleRemainingVUs
=== CONT  TestRampingVUsGracefulStopStops
--- PASS: TestRampingVUsHandleRemainingVUs (0.07s)
--- PASS: TestRampingVUsGracefulStopWaits (1.50s)
--- PASS: TestRampingVUsGracefulStopStops (2.50s)
--- PASS: TestRampingVUsGracefulRampDown (2.50s)
--- PASS: TestRampingVUsRampDownNoWobble (6.02s)
PASS
ok  	go.k6.io/k6/lib/executor	7.053s
```

Clean again: all `PASS`, no data race.

**`-race` evidence (Probe 3 — the execution-segment sum-invariant, run twice).**
This is the independent check behind Symptom 4: that summing the per-segment VU
counts reproduces the single-machine total exactly.
`TestSumRandomSegmentSequenceMatchesNoSegment`
(`lib/executor/ramping_vus_test.go:L1112`) generates random execution-segment
sequences and asserts the per-segment sums equal the no-segment shape; running it
under `-race` confirms the striping arithmetic (`SegmentedIndex`,
`lib/execution_segment.go:L768`) is free of data races as well:

```
$ go test -race -count=1 -run 'TestSumRandomSegmentSequenceMatchesNoSegment' ./lib/executor/     # RUN 1
ok  	go.k6.io/k6/lib/executor	1.468s
```

```
$ go test -race -count=1 -run 'TestSumRandomSegmentSequenceMatchesNoSegment' ./lib/executor/     # RUN 2
ok  	go.k6.io/k6/lib/executor	1.384s
```

Both runs report `ok` with **no `WARNING: DATA RACE`** and no `FAIL` — the
sum-invariant test passes cleanly under the race detector across repeated runs,
corroborating the Section 4 finding that the per-segment counts sum exactly to the
single-machine maximum.

**Structural finding (why no race is possible between the two handlers).**

- The two "handlers" are closures built in `RampingVUs.Run`:
  `handleNewMaxAllowedVUs = runState.maxAllowedVUsHandlerStrategy()`
  (`lib/executor/ramping_vus.go:L546`) and
  `handleNewScheduledVUs = runState.scheduledVUsHandlerStrategy()` (`:L547`).
- During the regular duration they are invoked **serially from a single goroutine**
  inside `iterateSteps` (`lib/executor/ramping_vus.go:L622-L645`). That function is one
  `for` loop that merges `rawSteps` and `gracefulSteps` by `TimeOffset` and calls
  **either** `handleNewMaxAllowedVUs(g)` (`:L634`) **or** `handleNewScheduledVUs(r)`
  (`:L640`) per iteration — never both at once, never from two goroutines. `iterateSteps`
  is called **synchronously** (`lib/executor/ramping_vus.go:L549-L553`).
- Only **after** `iterateSteps` returns does `Run` spawn the single extra goroutine:
  `go runState.runRemainingGracefulSteps(...)` (`lib/executor/ramping_vus.go:L554`).
  That goroutine (`:L654-L666`) calls **only** `handleNewMaxAllowedVUs(s)` (`:L664`) —
  it never touches the scheduled handler. So the scheduled handler has already stopped
  running by the time the graceful-continuation goroutine starts.
- Each handler closure owns a **private** `cur` counter (`maxAllowedVUsHandlerStrategy`
  at `:L668-L676` with `var cur uint64`; `scheduledVUsHandlerStrategy` at `:L679-L690`
  with its own `var cur uint64`). They share no mutable handler state.

**Verdict:** there is **no race between the two handler goroutines**, because there are
not two handler goroutines running the two handlers concurrently. The handlers are
called serially from one goroutine during the regular duration; the only separately
spawned goroutine runs *only* the max-allowed handler and *only after* the scheduled
handler has finished. The `-race` detector confirms this with a clean verdict across
repeated runs.

---

## Section 7 — Question B: trace simultaneous VU-state mutation → serialized

**What the user asked.** "Trace through what actually happens when both handlers try
to modify VU state simultaneously."

**Grounding from Section 6.** Because the two handlers are never invoked concurrently,
"both handlers mutating a VU simultaneously" cannot occur. The only *real* concurrency
on a single `vuHandle` is between a **handler call** (e.g. `gracefulStop()`/`hardStop()`/
`start()`) and the VU's own **run-loop** (`runLoopsIfPossible`). The `-race` probes in
Section 6 exercise exactly that overlap and come back clean.

**How that overlap is serialized (traced against source).**

- Each `vuHandle` carries a `mutex *sync.Mutex` (`lib/executor/vu_handle.go:L71`) and
  documents a thread-safe contract at `lib/executor/vu_handle.go:L61` (*"it needs to be
  able to start and stop VUs in thread safe fashion"*).
- Every externally-driven transition takes that lock first:
  `start()` → `vh.mutex.Lock()` (`lib/executor/vu_handle.go:L115-L116`),
  `gracefulStop()` → `vh.mutex.Lock()` (`:L147-L148`),
  `hardStop()` → `vh.mutex.Lock()` (`:L165-L167`). So a handler-initiated stop and a
  start cannot interleave — the mutex serializes them.
- The state field itself is written atomically: `changeState` is
  `atomic.StoreInt32((*int32)(&vh.state), int32(newState))`
  (`lib/executor/vu_handle.go:L142-L144`), and the run-loop reads it atomically via
  `atomic.LoadInt32((*int32)(&vh.state))` (`:L204`). This guarantees a consistent
  (non-torn) read even on the loop's fast path outside the lock.
- The run-loop re-checks under the lock to close the start/stop race window —
  `case <-vh.canStartIter: // we check again in case of race`
  (`lib/executor/vu_handle.go:L248`) — and clears the transient `toGracefulStop`/
  `toHardStop` states back to `stopped` on its slow path (`runLoopsIfPossible` at
  `:L185`).
- The full behaviour is specified by the transition table at
  `lib/executor/vu_handle.go:L34-L53`. Walking the relevant rows: if the scheduled
  handler calls `gracefulStop()` while the loop is running, the table gives
  `| grace | running | toGracefulStop |` (*"the actual work is in the loop"*); the loop
  then applies `| loop | toGracefulStop | stopped |` (`:L42`). If a `start()` arrives in
  between, `| start | toGracefulStop | running |` (`:L37`, *"we raced with the loop
  stopping, just continue"*) reverses it — all under the same mutex, so the outcome is
  deterministic, not a torn/"stuck" state.

**Verdict:** simultaneous VU-state mutation is **fully serialized**. The per-VU
`sync.Mutex` serializes handler-vs-loop transitions and the atomic state
write/read guarantees consistent reads; the transition table makes every
(action, state) pair well-defined. The `-race` detector's clean verdict across
repeated runs (Section 6) is the runtime confirmation that no unsynchronized state
mutation occurs.

---

## External references

The design invariants that the runtime observations above **confirm** are grounded in
the following authoritative sources. These corroborate the *expected* behaviours; they
are **not** the source of the observed values (which come from the runs shown in each
section above):

1. **k6 issue #997 — execution-segment partitioning / sum-invariant.**
   <https://github.com/grafana/k6/issues/997>. This is the issue that introduced the
   `--execution-segment` / `--execution-segment-sequence` options; the striping
   distributes indivisible VUs so that the per-segment values **sum to the
   single-machine total exactly**. Corroborated by the k6 v0.27.0 release notes, which
   state the new execution-segment options were added for partitioning test runs across
   instances ("See #997 for more details"). → grounds **Section 4** (Symptom 4).
2. **k6 issue #1308 — VU-count-only partitioning for looping-VU executors.**
   <https://github.com/grafana/k6/issues/1308>. For `ramping-vus` (internally
   "variable-looping-vus") only the *number of VUs* is partitioned across segments; the
   per-iteration work is not further subdivided (unlike the arrival-rate /
   shared-iterations executors). → grounds the "(inferred, from k6 issue #1308)" note in
   **Section 4**.
3. **Execution segments — k6 options reference.**
   <https://grafana.com/docs/k6/latest/using-k6/k6-options/reference/>. Documents the
   `--execution-segment` and `--execution-segment-sequence` options exercised in
   **Section 4**.
4. **`ramping-vus` executor — official k6 docs.**
   <https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/ramping-vus/>.
   Confirms `gracefulRampDown` (default `30s`, "separate from `gracefulStop`") and that
   with `gracefulRampDown: 0s` some iterations may be interrupted during ramp-down. →
   grounds **Sections 1, 2, 4, 5**.
5. **Graceful stop concept — official k6 docs.**
   <https://grafana.com/docs/k6/latest/using-k6/scenarios/concepts/graceful-stop/>.
   Confirms `gracefulStop` (default `30s`, available to all executors except
   externally-controlled) is the duration k6 waits before forcefully interrupting an
   iteration. → grounds **Section 3**.

All `file:line` citations throughout this document refer to the canonical k6 source
under investigation (`k6 v0.55.0`); the URLs above are supplementary design context
only.

---

## Section 8 — Coverage pass + final repository state

### 8a — Coverage checklist (every named item addressed by name)

| Named item (from the question) | Where addressed | Verdict / evidence |
|---|---|---|
| **Symptom 1** — "stuck" VUs | §1 | Transitional; `5/5` held, `0 interrupted`, 2/2 deterministic |
| **Symptom 2** — scheduled ≠ graceful count | §2 | By design; computed step plan: scheduled=1 vs max-allowed=6 at t=7s (runtime progress shows active=1/max=5) |
| **Symptom 3** — `ctrl+c` overrun | §3 | Not reproduced; SIGINT ≈45–51 ms; nothing past `maxEndTime` |
| **Symptom 4** — segment imbalance / sum > max | §4 | Sum = max (10 and 9); first segment gets remainder |
| **Symptom 5** — VU buffer leak | §5 | No leak; `0` warnings, `vus_max`=10 conserved |
| **Question A** — race between handler goroutines | §6 | No; handlers serial in one goroutine; `-race` clean 2/2 |
| **Question B** — simultaneous VU-state mutation | §7 | Serialized by mutex + atomic; `-race` clean |
| `iterateSteps` | §6 | `ramping_vus.go:L622-L645`; serial calls at L634 / L640 |
| `runRemainingGracefulSteps` | §6 | `ramping_vus.go:L654-L666`; only max-allowed at L664; goroutine at L554 |
| `maxAllowedVUsHandlerStrategy` | §2, §6 | `ramping_vus.go:L668-L676`; private `cur`; only `hardStop()` |
| `scheduledVUsHandlerStrategy` | §2, §6 | `ramping_vus.go:L679-L690`; private `cur`; `start()`/`gracefulStop()` |
| `reserveVUsForGracefulRampDowns` | §2 | `ramping_vus.go:L307`; comment L300-L301 |
| `GetExecutionRequirements` | §2 | `ramping_vus.go:L434`; final 0-VU step at `sumStagesDuration + GracefulStop` |
| `getDurationContexts` / `maxEndTime` | §3 | `helpers.go:L168` / `:L172`; `regDurationCtx` L178 |
| `vuHandle.start` / `gracefulStop` / `hardStop` | §3, §7 | `vu_handle.go:L115` / `:L147` / `:L165` (each `mutex.Lock()`) |
| `vuHandle.changeState` | §7 | `vu_handle.go:L142-L144`; `atomic.StoreInt32` |
| `vuHandle.runLoopsIfPossible` | §1, §7 | `vu_handle.go:L185`; atomic load L204; race re-check L248 |
| `GetPlannedVU` / `vus` chan | §5 | `execution.go:L471` / `:L106`; retries L29; warning L481 |
| `SegmentedIndex` | §4 | `execution_segment.go:L768` (+ L776/L782/L795/L808); `ScaleInt64` L734/L580 |
| file `ramping_vus.go` | §1, §2, §6 | executor + handlers + Run |
| file `vu_handle.go` | §1, §7 | per-VU state machine |
| file `helpers.go` | §3 | duration contexts |
| file `base_config.go` | §3 | `DefaultGracefulStopValue` L20; `GracefulStop` L31; `GetGracefulStop` L97 |
| file `execution.go` | §5 | VU buffer |
| file `execution_segment.go` | §4 | segment scaler |
| file `cmd/run.go` | §3 | interrupt handlers L349/L352/L357/L359/L361/L363 |
| file `cmd/common.go` | §3 | `handleTestAbortSignals` L97; `SignalNotify` L101 |
| file `main.go` | Methodology, §6 | real entry point `func main(){ cmd.Execute() }` |
| flag `--execution-segment` | §4 | three-instance split |
| flag `--execution-segment-sequence` | §4 | `"0,1/3,2/3,1"` |
| option `gracefulRampDown` | §1, §2, §4, §5 | 30s (reservation) and 0s (churn) |
| option `gracefulStop` | §3 | 30s / 2s natural-end deadline |
| `-race` detector | §6, §7 | clean verdicts, reported verbatim |

Every symptom (1–5), every question (A–B), and every named mechanism, function,
file, and flag is present and answered above.

### 8b — Final cleanup and repository integrity

All observation scripts and the built binary live outside the repository, under
`/tmp`. They are removed, and `git status --porcelain` is used to confirm the source
tree is unchanged apart from this single new document. (The Go toolchain at
`/usr/local/go` was pre-installed in the environment and is intentionally left in
place.)

```
$ rm -rf /tmp/k6obs /tmp/k6bin
$ git status --porcelain --untracked-files=all
?? blitzy/documentation/k6_ddc3b0b1d23c.md
```

(`--untracked-files=all` is used so git prints the exact new file path rather than
collapsing it to the untracked directory `?? blitzy/`; the pre-existing empty
`blitzy/screenshots` and `blitzy/screen_recordings` directories are not shown because
git does not track empty directories.)

The only change to the repository is the addition of this answer document,
`blitzy/documentation/k6_ddc3b0b1d23c.md` — exactly as required. No existing source,
test, configuration, or documentation file was modified, and no other file was added.

