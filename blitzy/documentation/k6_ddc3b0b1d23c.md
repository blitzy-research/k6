# k6 `ramping-vus` Concurrency Investigation — Answer Document

This document answers a read-only investigation into a suspected concurrency defect in k6's
`ramping-vus` executor. It is **run-first**: every behavioral claim below is paired with the actual,
unedited output of a program that was built and executed against the real k6 code, and every factual
claim about the system carries a `file:line` reference. Statements that were reasoned from reading the
code but not directly sampled at runtime are explicitly labeled **(INFERRED)**.

## 0. Scope, methodology, and how to read the evidence

- **Subject under investigation:** the `ramping-vus` executor `lib/executor/ramping_vus.go` and the
  per-VU state machine it drives `lib/executor/vu_handle.go`.
- **Investigated source tree:** k6 commit `ddc3b0b1d23c` ("Update comment"), product version
  `k6 v0.55.0`. All `file:line` references resolve against this tree. (See §3 for how the working
  tree's `HEAD` relates to this commit.)
- **Canonical entry points only.** Evidence comes from either (a) the public
  `go.k6.io/k6/lib/executor` package driven exactly as the in-repo tests drive it (via an in-package
  probe that reuses the real, unexported test harness), or (b) the compiled `k6` binary run through its
  real `k6 run` command line. No debug hooks, mocks, fallbacks, or synthetic stand-ins were used to
  produce any measured value.
- **Race detector.** The concurrency claim (Question a) is backed by `CGO_ENABLED=1 go test -race`.
  A clean `-race` run is strong — but not absolute — evidence: it only proves the absence of a data
  race on the interleavings that were actually exercised. Every `-race` result below was reproduced
  across at least two iterations.
- **Run-to-run inconsistency.** For the symptoms the user described as intermittent ("sometimes"),
  the *same unchanged input* was executed repeatedly and the observed distribution of outcomes is
  reported, rather than a single representative run.
- **Reproducibility artifacts.** The complete source of the temporary Go probe, the JavaScript test
  scripts, and the Bash harnesses used to capture every block below are embedded verbatim in §9, so
  that any reader can regenerate this evidence. Per the investigation's cleanup rule, these temporary
  artifacts are removed after capture and are **not** committed to the repository (see §9.4).
- **A note on the fenced blocks.** Everything inside a fenced block is verbatim captured output or
  verbatim source. All explanatory annotations are in prose *outside* the fenced blocks. Where a log
  line begins with `###` or `----`, that marker was echoed by the capture script itself and is part of
  the real transcript, not an annotation added after the fact. The only normalization applied to the
  captured output is that trailing whitespace (a cosmetic artifact of fixed-width column padding in the
  probe's and harnesses' formatting) has been trimmed for lint-cleanliness; no value, log message,
  count, timing, or footer was altered. The substantive content of every fenced block — the values, the
  log-message text, the counts, the timings, and the `PASS`/`ok` footers — is verbatim from the run that
  produced it. The one exception is the source-line-number prefix that Go's `t.Logf` automatically
  prepends to each probe log line (e.g., `blitzy_adhoc_test_probe_test.go:484:`): because the probe was
  lightly edited between capture runs, those auto-generated numeric prefixes can differ by a few lines
  from the consolidated §9.1 listing; the message text after each prefix is unaltered.

---

## 1. Summary (top-line answer)

The reported symptoms do **not** stem from an open data race between two competing handler goroutines,
nor from a VU-buffer leak. On the canonical `ramping-vus` path, observed at runtime:

1. **There are not two concurrently-running handler goroutines to race.** The two "handlers" are the
   `maxAllowedVUsHandlerStrategy` `lib/executor/ramping_vus.go:668` and the
   `scheduledVUsHandlerStrategy` `lib/executor/ramping_vus.go:679`. They are invoked **sequentially**,
   one call per loop iteration, from a single `for` loop inside `iterateSteps`
   `lib/executor/ramping_vus.go:622-643`, which runs in the main `Run` goroutine
   `lib/executor/ramping_vus.go:491`. `go test -race` over both the VU-handle race suite and the full
   ramping-vus suite is clean across two iterations each (§4).
2. **Per-VU state cannot be modified unsafely "simultaneously."** `start`, `gracefulStop`, and
   `hardStop` each lock the *same* per-VU `mutex` `lib/executor/vu_handle.go:71`, so their bodies
   cannot interleave; state writes go through an atomic store and reads are either performed under that
   mutex or via a single lock-free atomic load on the hot path (§5). This is a **mixed** synchronization
   model, not "atomics only."
3. **The VU buffer is not leaking on the observed paths.** After a rapid up/down run, the active-VU
   counter returns to 0 and every planned VU is back in the buffer (8 of 8 drained), and the
   depleted-buffer failure path fails cleanly and boundedly rather than orphaning a handle (§6).
4. **Segment imbalance is deterministic striping, not a race** (§7-D).

### 1.1 Per-symptom reproduction status

Each of the four named symptoms was reproduced against the real code; the precise observed behavior
(and, where relevant, the gap between the literal symptom and what the runtime actually does) is:

| Symptom | Reproduced? | What the runtime actually does (evidence) |
|---|---|---|
| **A — "stuck" VUs** during rapid up/down + long `gracefulRampDown` | The *appearance* is reproduced; permanent stuck state is **not** | Across 10 identical runs, active VUs transiently exceed the scheduled target by a **bounded** amount (max surplus **+2**) during each rapid down-stage, then always return to **0** by the end (final `0×10`). No run left a permanently stuck VU. The surplus is graceful-ramp-down lag: VUs mid-iteration in the transient `toGracefulStop` state **(INFERRED)** finishing their short iteration. (§7-A) |
| **B — scheduled vs. graceful count mismatch** | Yes — and it is **by design** | The max-allowed *ceiling* (graceful handler) legitimately holds **above** the scheduled *target* (scheduled handler) throughout every down-stage; `ceiling >= target` holds for **every** sampled instant. The two closures track different quantities on purpose; the divergence is expected, not a defect. (§7-B) |
| **C — VUs outrun `gracefulStop` on ctrl+c** | Re-characterized — **not** reproduced as a VU/`gracefulStop` defect | A single real `SIGINT` cancels VU iteration contexts and interrupts the JS VM, so even a CPU-busy iteration stops in **~17–19 ms** (exit code 105). The only way to observe "keeps running for seconds" is non-VU **graceful-phase work** such as `teardown()` (~3 s here) running to completion during the graceful abort — which is **not** a per-VU iteration overrunning `gracefulStop`. A second `SIGINT` escalates to an immediate hard stop (~17–19 ms). The user's literal "VUs keep running longer than `gracefulStop`" was not reproduced on the canonical path **(INFERRED:** the user likely observed graceful-phase work, or a non-yielding native call inside an iteration**)**. (§7-C) |
| **D — one instance shows more VUs; sum exceeds max** | Yes — deterministically | With a **shared** `--execution-segment-sequence`, three synchronized instances split a global max of 11 as `[4, 4, 3]`, one instance consistently higher, sum **== 11** (never exceeds). The sum **exceeds** the configured max only when instances are run **without** a shared sequence (`[4, 4, 4]`, sum **12 > 11**), because each instance independently fills a different sequence and rounds up. Both cases are deterministic per the striping arithmetic — not random, not a race. (§7-D) |

Every conclusion above is grounded in the runtime evidence in §3–§7 and the source trace at the cited
`file:line` locations. Items that were reasoned from the source rather than directly sampled are marked
**(INFERRED)**.

---

## 2. Direct answers to the three questions

The user asked three explicit questions. Each is answered directly first, then substantiated in the
referenced section.

### 2.1 Question (a) — "Is there a race condition between the two handler goroutines?"

**Direct answer: No — and the premise that there are *two handler goroutines* does not hold.** The two
handler strategies are **not** run as competing goroutines. They are two closures invoked
**sequentially** from one `for` loop in `iterateSteps` `lib/executor/ramping_vus.go:622-643`: each loop
iteration handles a single step and calls **either** `handleNewMaxAllowedVUs(g)`
`lib/executor/ramping_vus.go:634` **or** `handleNewScheduledVUs(r)` `lib/executor/ramping_vus.go:640`,
never both at once, and that loop runs in the single main `Run` goroutine
`lib/executor/ramping_vus.go:491`.

`Run` *does* spawn other goroutines — a progress-tracking goroutine
`lib/executor/ramping_vus.go:536`, one VU-loop goroutine per planned VU
`lib/executor/ramping_vus.go:615`, and, only after `iterateSteps` returns, a graceful-tail goroutine
`go runState.runRemainingGracefulSteps` `lib/executor/ramping_vus.go:554` — but those interact with
per-VU state only through the mutex/atomic discipline of §5, not by two handlers mutating the same
state concurrently. Running the VU-handle race suite and the full ramping-vus suite under
`go test -race` (two iterations each) reports **zero** data races (§4). (Caveat: the race detector only
observes exercised interleavings.)

### 2.2 Question (b) — "Or maybe the VU buffer is leaking somehow?"

**Direct answer: No VU-buffer leak was observed on the exercised paths.** After a rapid up/down run,
the active-VU counter returns to **0** and **8 of 8** planned VUs are drainable from the buffer
afterward — none orphaned (§6). This is authoritative because `Run` blocks on
`defer runState.wg.Wait()` `lib/executor/ramping_vus.go:540`, so by the time it returns every acquired
VU has been returned. Acquisition and return are one-to-one in the accounting closures
`getVU`/`returnVU` `lib/executor/ramping_vus.go:592-610`, and the acquisition **failure** path returns
before it ever increments any counter `lib/executor/ramping_vus.go:595-600`, so a failed acquire cannot
be mis-paired with a spurious return. The depleted-buffer failure path was exercised directly: it fails
cleanly and boundedly after 5 retries (`MaxRetriesGetPlannedVU = 5` `lib/execution.go:29`), returning a
`nil` VU and an error rather than leaking (§6). (Scope: "no leak" is asserted for the observed paths; the
active-VU counter itself is a UI-only signal `lib/execution.go:268` — see the caveat in §6.)

### 2.3 Question (c) — "Trace what actually happens when both handlers try to modify VU state simultaneously."

**Direct answer: They cannot modify a given VU's state unsafely at the same time, because all three
control methods serialize on that VU's single mutex.** `start` `lib/executor/vu_handle.go:115`,
`gracefulStop` `lib/executor/vu_handle.go:147`, and `hardStop` `lib/executor/vu_handle.go:165` each
acquire the same `vh.mutex` `lib/executor/vu_handle.go:71` (locked at lines `:116`, `:148`, `:166`
respectively) before touching state, so their critical sections are mutually exclusive — even if the
callers were concurrent, the bodies run one at a time. State transitions use a **mixed** synchronization
model (not "atomics only", as one might assume): the `state` field is initialized by a plain assignment
at construction `lib/executor/vu_handle.go:106` (before any goroutine can see it), every write goes
through an atomic store in `changeState` `lib/executor/vu_handle.go:144` while holding the mutex, reads
inside the control methods are plain reads performed **under** the mutex
(`lib/executor/vu_handle.go:119`, `:150`, `:169`), and the VU run-loop has exactly one **lock-free**
atomic-load fast path `lib/executor/vu_handle.go:204` plus a mutex-guarded slow path
`lib/executor/vu_handle.go:219`. The documented transition table `lib/executor/vu_handle.go:24-55`
explicitly resolves the raced orderings (e.g., a `start` arriving while a VU is in `toGracefulStop`
just continues running). A deterministic single-VU trace and the clean `-race` results corroborate this
(§5). So "both handlers modifying VU state simultaneously" resolves to *serialized, mutually-exclusive*
modifications of a mutex-protected state machine — no unsafe interleaving occurs.

---

## 3. Environment and build evidence

All evidence in this document was produced in the environment captured below. Annotations are in prose;
the fenced blocks are the verbatim transcript written by the capture commands (each `###` line is the
command label the capture script echoed, followed by that command's real stdout and its captured
`exit=` code).

### 3.1 Toolchain, branch, and race-detector prerequisites

The Go race detector requires `CGO_ENABLED=1` and a C compiler; both are present (`go env CGO_ENABLED`
prints `1`; `gcc` is **15.2.0**). Vendored modules are used (`GOFLAGS=-mod=vendor`), so builds and tests
run fully offline. (The `git rev-parse HEAD` value in the block below is a point-in-time capture taken at
the initial documentation draft commit; §3.2 explains why this self-referential value does not affect any
`file:line` reference in this document.)

```text
### git branch --show-current
blitzy-cfb43038-9f4d-4c7a-9f83-818b7e754ba5
exit=0

### git rev-parse HEAD
128744ee44b8af37d9ede6e620ce87fc6c2c7fca
exit=0

### git status --porcelain --untracked-files=all
exit=0  (empty output above => clean)

### go version
go version go1.23.6 linux/amd64
exit=0

### gcc --version
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
Copyright (C) 2025 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

exit=0

### go env CGO_ENABLED GOFLAGS GOTOOLCHAIN
1
-mod=vendor
auto
exit=0
```

### 3.2 Relationship of the working tree to the investigated commit

The git evidence shown in this section was captured at the initial documentation **draft** commit
`128744ee4` (at that point the file was a 1,008-line draft, which is exactly what the `git diff … --stat`
below reports). The document was subsequently expanded to its present form by a later commit on the same
branch, so the current `HEAD` is a *later* documentation commit than the one quoted in the block below.
This self-referential, point-in-time capture changes none of the analysis, thanks to a stable invariant:
**every** documentation commit on this branch adds only this one markdown file and touches **no** k6
source, so the working tree's source stays byte-identical to the investigated k6 source commit
`ddc3b0b1d23c` ("Update comment", the parent of the first documentation commit) regardless of which
documentation commit is checked out — i.e., all `file:line` references in this document resolve equally
against the current `HEAD` and against `ddc3b0b1d23c`. This invariant is stable across re-commits and is
independently verifiable at any time with
`git diff --name-only ddc3b0b1d23c HEAD -- . ':(exclude)blitzy/documentation/k6_ddc3b0b1d23c.md'`, which
prints nothing. The `go.mod` minimum (`go 1.21`) is satisfied by the local toolchain (Go 1.23.6), which is
also the highest CI-supported line (`DEFAULT_GO_VERSION: "1.23.x"`). The exit code used by the ctrl+c path
(§7-C), `ExternalAbort = 105`, is defined here as well.

```text
### Relationship of current HEAD to investigated commit ddc3b0b1d23c
--- git log --oneline -2 ---
128744ee4 docs: add k6 ramping-vus concurrency investigation answer (k6_ddc3b0b1d23c)
ddc3b0b1d Update comment

--- git diff ddc3b0b1d23c..HEAD --stat (what the doc commit changed) ---
 blitzy/documentation/k6_ddc3b0b1d23c.md | 1008 +++++++++++++++++++++++++++++++
 1 file changed, 1008 insertions(+)

--- git diff ddc3b0b1d23c..HEAD --name-status ---
A	blitzy/documentation/k6_ddc3b0b1d23c.md

### go.mod (module/go/toolchain lines)
1:module go.k6.io/k6
3:go 1.21
5:toolchain go1.21.13

### CI supported Go version (.github/workflows/build.yml DEFAULT_GO_VERSION)
27:  DEFAULT_GO_VERSION: "1.23.x"
64:            GO_VERSION="${DEFAULT_GO_VERSION}"

### exit code authority (errext/exitcodes/codes.go ExternalAbort)
	InvalidConfig ExitCode = 104

	// ExternalAbort indicates the test was aborted by an external signal
	// (e.g. SIGINT, SIGTERM, etc.) and should be considered aborted rather
	// than a failure.
	ExternalAbort ExitCode = 105

```

### 3.3 Offline build of the k6 binary under test

`go mod verify` confirms the vendored modules are intact, and the investigation binary builds offline in
~2.7 s. It is written to `/tmp/k6` — **outside** the repository tree — so the repo is left unchanged.
(The environment setup separately produced a binary at `/tmp/k6_bin`; the binary this investigation
built and used is the distinct `/tmp/k6`.) The product version string is `k6 v0.55.0`; the
`commit/128744ee44` suffix in the version string is the build-time `HEAD` and is environment-specific
(it reflects whatever commit is checked out at build time, here the documentation commit).

```text
### go mod verify
all modules verified
exit=0

### vendor mode check (head of vendor/modules.txt + count)
# buf.build/gen/go/gogo/protobuf/protocolbuffers/go v1.31.0-20210810001428-4df00b267f94.1
## explicit
buf.build/gen/go/gogo/protobuf/protocolbuffers/go/gogoproto
... (total explicit modules:)
94

### Build k6 (investigation binary, kept OUTSIDE repo at /tmp/k6)

real	0m2.713s
user	0m3.880s
sys	0m2.342s
build exit=0

### /tmp/k6 version
k6 v0.55.0 (commit/128744ee44, go1.23.6, linux/amd64)
exit=0
```

---

## 4. Question (a): the two-handler concurrency model, and the race-detector result

### 4.1 The two "handlers" are invoked sequentially from one goroutine

`Run` constructs the two handler strategies as closures over shared run state
`lib/executor/ramping_vus.go:546-547` and hands them to `iterateSteps`
`lib/executor/ramping_vus.go:622`. `iterateSteps` merge-walks the executor's two pre-computed step
lists (`rawSteps`, the scheduled targets, and `gracefulSteps`, the max-allowed ceiling) in a single
`for` loop `lib/executor/ramping_vus.go:628-643`. On each iteration it advances to the next step in
time and then calls **exactly one** of the two handlers:

- a **graceful** step → `handleNewMaxAllowedVUs(g)` `lib/executor/ramping_vus.go:634`, which runs
  `maxAllowedVUsHandlerStrategy` `lib/executor/ramping_vus.go:668`; that closure only ever *shrinks*
  the ceiling by calling `hardStop` on now-excess VUs `lib/executor/ramping_vus.go:672-674` and grows
  it implicitly `lib/executor/ramping_vus.go:675` (no `start`, no log on growth);
- a **raw** step → `handleNewScheduledVUs(r)` `lib/executor/ramping_vus.go:640`, which runs
  `scheduledVUsHandlerStrategy` `lib/executor/ramping_vus.go:679`; that closure `start`s VUs when the
  target grows `lib/executor/ramping_vus.go:683-685` and `gracefulStop`s them when it shrinks
  `lib/executor/ramping_vus.go:686-688`.

Because both calls happen from the same loop in the same goroutine, **the two strategies never execute
concurrently with each other** — the "race between two handler goroutines" premise does not apply. Each
handler keeps its *own* closure-local `cur` counter (`lib/executor/ramping_vus.go:669` and `:680`),
which is why they can report different counts at the same instant (this is Symptom B, §7-B) without any
shared-memory hazard.

### 4.2 The goroutines `Run` actually starts (so the claim is scoped precisely)

To avoid overstating the "no concurrency" result, here is the complete set of goroutines in a
`ramping-vus` run and how each touches VU state:

- **Main `Run` goroutine** `lib/executor/ramping_vus.go:491` — runs `iterateSteps`, i.e. both handler
  strategies, sequentially (§4.1).
- **Progress goroutine** `lib/executor/ramping_vus.go:536` — reports progress; does not mutate VU
  state.
- **One VU-loop goroutine per planned VU** `go rs.vuHandles[i].runLoopsIfPossible`
  `lib/executor/ramping_vus.go:615` — these run concurrently with the handlers, but they touch each
  `vuHandle` only through its mutex/atomic discipline (§5), which is exactly what the `-race` run
  exercises.
- **Graceful-tail goroutine** `go runState.runRemainingGracefulSteps`
  `lib/executor/ramping_vus.go:554` — launched **after** `iterateSteps` returns, it drives only the
  remaining max-allowed (graceful) steps `lib/executor/ramping_vus.go:664`. By the time it runs, the
  scheduled handler has already finished, so it does not run *concurrently* with the scheduled handler
  either.

Coordination is via `defer runState.wg.Wait()` `lib/executor/ramping_vus.go:540` and the cancel /
progress-drain deferred in `Run`. **(INFERRED** from the source structure above; the *safety* of the
concurrent VU-loop goroutines is confirmed empirically by the clean `-race` runs below.**)**

### 4.3 Normal completion vs. early cancellation

`iterateSteps` returns in two distinct ways, which matters for reasoning about the graceful tail:

- **Normal exhaustion:** the loop ends after the last step is consumed. This is the common case the
  informal description ("returns after the raw steps are done") refers to.
- **Early cancellation:** each iteration first waits for the step's time offset, and that wait returns
  early if the context is cancelled (`if wait(...) { break }` at `lib/executor/ramping_vus.go:632` and
  `:638`; the waiter returns `true` on `ctx.Done()` `lib/executor/ramping_vus.go:698-711`). On
  cancellation the loop `break`s and `iterateSteps` can return with steps still unhandled.

### 4.4 Race-detector evidence (canonical, reproduced ×2)

**Command (VU-handle race/state suite):**

```text
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -race -run 'TestVUHandle' ./lib/executor/ -count=2 -v
```

The suite includes `TestVUHandleRace` and `TestVUHandleStartStopRace`, which deliberately hammer
`start`/`gracefulStop`/`hardStop` on a single `vuHandle` from competing goroutines. Both `-count=2`
iterations pass with **no** `WARNING: DATA RACE`:

```text
=== RUN   TestVUHandleRace
=== PAUSE TestVUHandleRace
=== RUN   TestVUHandleStartStopRace
=== PAUSE TestVUHandleStartStopRace
=== RUN   TestVUHandleSimple
=== PAUSE TestVUHandleSimple
=== CONT  TestVUHandleSimple
=== CONT  TestVUHandleStartStopRace
=== RUN   TestVUHandleSimple/start_before_gracefulStop_finishes
=== PAUSE TestVUHandleSimple/start_before_gracefulStop_finishes
=== CONT  TestVUHandleRace
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
=== RUN   TestVUHandleRace
=== PAUSE TestVUHandleRace
=== RUN   TestVUHandleStartStopRace
=== PAUSE TestVUHandleStartStopRace
=== RUN   TestVUHandleSimple
=== PAUSE TestVUHandleSimple
=== CONT  TestVUHandleRace
=== CONT  TestVUHandleStartStopRace
=== CONT  TestVUHandleSimple
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
--- PASS: TestVUHandleStartStopRace (0.42s)
--- PASS: TestVUHandleSimple (0.00s)
    --- PASS: TestVUHandleSimple/start_after_hardStop (1.53s)
    --- PASS: TestVUHandleSimple/start_before_gracefulStop_finishes (1.56s)
    --- PASS: TestVUHandleSimple/start_after_gracefulStop_finishes (3.10s)
PASS
ok  	go.k6.io/k6/lib/executor	7.234s
```

**Command (full ramping-vus suite, including the graceful-ramp-down "no wobble" regression test):**

```text
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -race -run 'TestRampingVUs' ./lib/executor/ -count=2 -v
```

Both iterations pass with **no** `WARNING: DATA RACE`. This exercises the whole concurrent structure of
a real run — the sequential handlers plus the concurrent per-VU loop goroutines plus the progress and
graceful-tail goroutines — and includes `TestRampingVUsRampDownNoWobble`, the regression test for the
historical "no wobble of VUs during graceful ramp-down" fix that directly matches Symptoms A/B:

```text
=== RUN   TestRampingVUsConfigValidation
=== PAUSE TestRampingVUsConfigValidation
=== RUN   TestRampingVUsRun
=== PAUSE TestRampingVUsRun
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
=== RUN   TestRampingVUsConfigExecutionPlanExample
=== PAUSE TestRampingVUsConfigExecutionPlanExample
=== RUN   TestRampingVUsConfigExecutionPlanExampleOneThird
=== PAUSE TestRampingVUsConfigExecutionPlanExampleOneThird
=== RUN   TestRampingVUsExecutionTupleTests
=== PAUSE TestRampingVUsExecutionTupleTests
=== RUN   TestRampingVUsGetRawExecutionStepsCornerCases
=== PAUSE TestRampingVUsGetRawExecutionStepsCornerCases
=== CONT  TestRampingVUsConfigValidation
=== CONT  TestRampingVUsRampDownNoWobble
=== RUN   TestRampingVUsConfigValidation/no_stages
=== CONT  TestRampingVUsGracefulStopStops
=== PAUSE TestRampingVUsConfigValidation/no_stages
=== RUN   TestRampingVUsConfigValidation/basic_1_stage
=== PAUSE TestRampingVUsConfigValidation/basic_1_stage
=== CONT  TestRampingVUsGracefulStopWaits
=== CONT  TestRampingVUsGetRawExecutionStepsCornerCases
=== CONT  TestRampingVUsRun
=== CONT  TestRampingVUsConfigExecutionPlanExampleOneThird
=== RUN   TestRampingVUsGetRawExecutionStepsCornerCases/going_up_then_down_straight_away
=== CONT  TestRampingVUsConfigExecutionPlanExample
=== CONT  TestRampingVUsHandleRemainingVUs
=== CONT  TestRampingVUsGracefulRampDown
--- PASS: TestRampingVUsConfigExecutionPlanExampleOneThird (0.00s)
=== RUN   TestRampingVUsConfigValidation/If_startVUs_are_larger_than_maxConcurrentVUs,_the_validation_should_return_an_error
=== CONT  TestRampingVUsExecutionTupleTests
=== PAUSE TestRampingVUsGetRawExecutionStepsCornerCases/going_up_then_down_straight_away
=== PAUSE TestRampingVUsConfigValidation/If_startVUs_are_larger_than_maxConcurrentVUs,_the_validation_should_return_an_error
--- PASS: TestRampingVUsConfigExecutionPlanExample (0.00s)
=== RUN   TestRampingVUsGetRawExecutionStepsCornerCases/jump_up_then_go_up_again
=== RUN   TestRampingVUsConfigValidation/For_multiple_VU_values_larger_than_maxConcurrentVUs,_multiple_errors_are_returned
=== PAUSE TestRampingVUsGetRawExecutionStepsCornerCases/jump_up_then_go_up_again
=== RUN   TestRampingVUsExecutionTupleTests/0:1_in_0,1
=== RUN   TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down
=== PAUSE TestRampingVUsConfigValidation/For_multiple_VU_values_larger_than_maxConcurrentVUs,_multiple_errors_are_returned
=== PAUSE TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down
=== PAUSE TestRampingVUsExecutionTupleTests/0:1_in_0,1
=== RUN   TestRampingVUsConfigValidation/VU_values_below_maxConcurrentVUs_will_pass_validation
=== PAUSE TestRampingVUsConfigValidation/VU_values_below_maxConcurrentVUs_will_pass_validation
=== RUN   TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_half
=== CONT  TestRampingVUsConfigValidation/no_stages
=== CONT  TestRampingVUsConfigValidation/If_startVUs_are_larger_than_maxConcurrentVUs,_the_validation_should_return_an_error
=== CONT  TestRampingVUsConfigValidation/basic_1_stage
=== CONT  TestRampingVUsConfigValidation/VU_values_below_maxConcurrentVUs_will_pass_validation
=== CONT  TestRampingVUsConfigValidation/For_multiple_VU_values_larger_than_maxConcurrentVUs,_multiple_errors_are_returned
=== RUN   TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,1
=== PAUSE TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_half
=== PAUSE TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,1
--- PASS: TestRampingVUsConfigValidation (0.01s)
    --- PASS: TestRampingVUsConfigValidation/no_stages (0.00s)
    --- PASS: TestRampingVUsConfigValidation/If_startVUs_are_larger_than_maxConcurrentVUs,_the_validation_should_return_an_error (0.00s)
    --- PASS: TestRampingVUsConfigValidation/basic_1_stage (0.00s)
    --- PASS: TestRampingVUsConfigValidation/For_multiple_VU_values_larger_than_maxConcurrentVUs,_multiple_errors_are_returned (0.00s)
    --- PASS: TestRampingVUsConfigValidation/VU_values_below_maxConcurrentVUs_will_pass_validation (0.00s)
=== RUN   TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_the_other_half
=== PAUSE TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_the_other_half
=== RUN   TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_with_nothing
=== PAUSE TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_with_nothing
=== RUN   TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,1#01
=== RUN   TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_with_funky_sequence
=== PAUSE TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,1#01
=== PAUSE TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_with_funky_sequence
=== RUN   TestRampingVUsGetRawExecutionStepsCornerCases/strange
=== PAUSE TestRampingVUsGetRawExecutionStepsCornerCases/strange
=== RUN   TestRampingVUsExecutionTupleTests/1/3:2/3_in_0,1/3,2/3,1
=== RUN   TestRampingVUsGetRawExecutionStepsCornerCases/more_up_and_down
=== PAUSE TestRampingVUsExecutionTupleTests/1/3:2/3_in_0,1/3,2/3,1
=== PAUSE TestRampingVUsGetRawExecutionStepsCornerCases/more_up_and_down
=== CONT  TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down
=== CONT  TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_the_other_half
=== CONT  TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_with_funky_sequence
=== CONT  TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_half
=== CONT  TestRampingVUsGetRawExecutionStepsCornerCases/jump_up_then_go_up_again
=== CONT  TestRampingVUsGetRawExecutionStepsCornerCases/going_up_then_down_straight_away
=== CONT  TestRampingVUsGetRawExecutionStepsCornerCases/strange
=== CONT  TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_with_nothing
=== RUN   TestRampingVUsExecutionTupleTests/2/3:1_in_0,2/3,1
=== CONT  TestRampingVUsGetRawExecutionStepsCornerCases/more_up_and_down
=== PAUSE TestRampingVUsExecutionTupleTests/2/3:1_in_0,2/3,1
--- PASS: TestRampingVUsGetRawExecutionStepsCornerCases (0.01s)
    --- PASS: TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down (0.00s)
    --- PASS: TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_the_other_half (0.00s)
    --- PASS: TestRampingVUsGetRawExecutionStepsCornerCases/jump_up_then_go_up_again (0.00s)
    --- PASS: TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_half (0.00s)
    --- PASS: TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_with_funky_sequence (0.00s)
    --- PASS: TestRampingVUsGetRawExecutionStepsCornerCases/going_up_then_down_straight_away (0.00s)
    --- PASS: TestRampingVUsGetRawExecutionStepsCornerCases/strange (0.00s)
    --- PASS: TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_with_nothing (0.00s)
    --- PASS: TestRampingVUsGetRawExecutionStepsCornerCases/more_up_and_down (0.00s)
=== RUN   TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,2/3,1
=== PAUSE TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,2/3,1
=== RUN   TestRampingVUsExecutionTupleTests/1/3:2/3_in_0,1/3,2/3,1#01
=== PAUSE TestRampingVUsExecutionTupleTests/1/3:2/3_in_0,1/3,2/3,1#01
=== RUN   TestRampingVUsExecutionTupleTests/2/3:1_in_0,1/3,2/3,1
=== PAUSE TestRampingVUsExecutionTupleTests/2/3:1_in_0,1/3,2/3,1
=== CONT  TestRampingVUsExecutionTupleTests/0:1_in_0,1
=== CONT  TestRampingVUsExecutionTupleTests/2/3:1_in_0,2/3,1
=== CONT  TestRampingVUsExecutionTupleTests/1/3:2/3_in_0,1/3,2/3,1#01
=== CONT  TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,1#01
=== CONT  TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,1
=== CONT  TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,2/3,1
=== CONT  TestRampingVUsExecutionTupleTests/1/3:2/3_in_0,1/3,2/3,1
=== CONT  TestRampingVUsExecutionTupleTests/2/3:1_in_0,1/3,2/3,1
--- PASS: TestRampingVUsExecutionTupleTests (0.00s)
    --- PASS: TestRampingVUsExecutionTupleTests/0:1_in_0,1 (0.00s)
    --- PASS: TestRampingVUsExecutionTupleTests/1/3:2/3_in_0,1/3,2/3,1#01 (0.00s)
    --- PASS: TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,1#01 (0.00s)
    --- PASS: TestRampingVUsExecutionTupleTests/2/3:1_in_0,2/3,1 (0.00s)
    --- PASS: TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,1 (0.00s)
    --- PASS: TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,2/3,1 (0.00s)
    --- PASS: TestRampingVUsExecutionTupleTests/1/3:2/3_in_0,1/3,2/3,1 (0.00s)
    --- PASS: TestRampingVUsExecutionTupleTests/2/3:1_in_0,1/3,2/3,1 (0.00s)
--- PASS: TestRampingVUsHandleRemainingVUs (0.07s)
--- PASS: TestRampingVUsGracefulStopWaits (1.51s)
--- PASS: TestRampingVUsRun (2.41s)
--- PASS: TestRampingVUsGracefulStopStops (2.50s)
--- PASS: TestRampingVUsGracefulRampDown (2.50s)
--- PASS: TestRampingVUsRampDownNoWobble (6.02s)
=== RUN   TestRampingVUsConfigValidation
=== PAUSE TestRampingVUsConfigValidation
=== RUN   TestRampingVUsRun
=== PAUSE TestRampingVUsRun
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
=== RUN   TestRampingVUsConfigExecutionPlanExample
=== PAUSE TestRampingVUsConfigExecutionPlanExample
=== RUN   TestRampingVUsConfigExecutionPlanExampleOneThird
=== PAUSE TestRampingVUsConfigExecutionPlanExampleOneThird
=== RUN   TestRampingVUsExecutionTupleTests
=== PAUSE TestRampingVUsExecutionTupleTests
=== RUN   TestRampingVUsGetRawExecutionStepsCornerCases
=== PAUSE TestRampingVUsGetRawExecutionStepsCornerCases
=== CONT  TestRampingVUsGracefulStopWaits
=== CONT  TestRampingVUsExecutionTupleTests
=== CONT  TestRampingVUsGracefulRampDown
=== CONT  TestRampingVUsRampDownNoWobble
=== CONT  TestRampingVUsConfigExecutionPlanExample
=== CONT  TestRampingVUsRun
=== CONT  TestRampingVUsConfigExecutionPlanExampleOneThird
=== CONT  TestRampingVUsGracefulStopStops
=== CONT  TestRampingVUsHandleRemainingVUs
=== CONT  TestRampingVUsConfigValidation
=== CONT  TestRampingVUsGetRawExecutionStepsCornerCases
=== RUN   TestRampingVUsConfigValidation/no_stages
=== RUN   TestRampingVUsExecutionTupleTests/0:1_in_0,1
=== PAUSE TestRampingVUsConfigValidation/no_stages
--- PASS: TestRampingVUsConfigExecutionPlanExample (0.00s)
=== PAUSE TestRampingVUsExecutionTupleTests/0:1_in_0,1
--- PASS: TestRampingVUsConfigExecutionPlanExampleOneThird (0.00s)
=== RUN   TestRampingVUsConfigValidation/basic_1_stage
=== PAUSE TestRampingVUsConfigValidation/basic_1_stage
=== RUN   TestRampingVUsGetRawExecutionStepsCornerCases/going_up_then_down_straight_away
=== RUN   TestRampingVUsConfigValidation/If_startVUs_are_larger_than_maxConcurrentVUs,_the_validation_should_return_an_error
=== RUN   TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,1
=== PAUSE TestRampingVUsGetRawExecutionStepsCornerCases/going_up_then_down_straight_away
=== PAUSE TestRampingVUsConfigValidation/If_startVUs_are_larger_than_maxConcurrentVUs,_the_validation_should_return_an_error
=== PAUSE TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,1
=== RUN   TestRampingVUsConfigValidation/For_multiple_VU_values_larger_than_maxConcurrentVUs,_multiple_errors_are_returned
=== RUN   TestRampingVUsGetRawExecutionStepsCornerCases/jump_up_then_go_up_again
=== PAUSE TestRampingVUsConfigValidation/For_multiple_VU_values_larger_than_maxConcurrentVUs,_multiple_errors_are_returned
=== PAUSE TestRampingVUsGetRawExecutionStepsCornerCases/jump_up_then_go_up_again
=== RUN   TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,1#01
=== RUN   TestRampingVUsConfigValidation/VU_values_below_maxConcurrentVUs_will_pass_validation
=== PAUSE TestRampingVUsConfigValidation/VU_values_below_maxConcurrentVUs_will_pass_validation
=== PAUSE TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,1#01
=== CONT  TestRampingVUsConfigValidation/basic_1_stage
=== RUN   TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down
=== CONT  TestRampingVUsConfigValidation/If_startVUs_are_larger_than_maxConcurrentVUs,_the_validation_should_return_an_error
=== PAUSE TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down
=== CONT  TestRampingVUsConfigValidation/no_stages
=== CONT  TestRampingVUsConfigValidation/VU_values_below_maxConcurrentVUs_will_pass_validation
=== CONT  TestRampingVUsConfigValidation/For_multiple_VU_values_larger_than_maxConcurrentVUs,_multiple_errors_are_returned
=== RUN   TestRampingVUsExecutionTupleTests/1/3:2/3_in_0,1/3,2/3,1
=== PAUSE TestRampingVUsExecutionTupleTests/1/3:2/3_in_0,1/3,2/3,1
=== RUN   TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_half
--- PASS: TestRampingVUsConfigValidation (0.00s)
    --- PASS: TestRampingVUsConfigValidation/basic_1_stage (0.00s)
    --- PASS: TestRampingVUsConfigValidation/If_startVUs_are_larger_than_maxConcurrentVUs,_the_validation_should_return_an_error (0.00s)
    --- PASS: TestRampingVUsConfigValidation/no_stages (0.00s)
    --- PASS: TestRampingVUsConfigValidation/VU_values_below_maxConcurrentVUs_will_pass_validation (0.00s)
    --- PASS: TestRampingVUsConfigValidation/For_multiple_VU_values_larger_than_maxConcurrentVUs,_multiple_errors_are_returned (0.00s)
=== PAUSE TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_half
=== RUN   TestRampingVUsExecutionTupleTests/2/3:1_in_0,2/3,1
=== PAUSE TestRampingVUsExecutionTupleTests/2/3:1_in_0,2/3,1
=== RUN   TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_the_other_half
=== RUN   TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,2/3,1
=== PAUSE TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_the_other_half
=== PAUSE TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,2/3,1
=== RUN   TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_with_nothing
=== RUN   TestRampingVUsExecutionTupleTests/1/3:2/3_in_0,1/3,2/3,1#01
=== PAUSE TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_with_nothing
=== PAUSE TestRampingVUsExecutionTupleTests/1/3:2/3_in_0,1/3,2/3,1#01
=== RUN   TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_with_funky_sequence
=== RUN   TestRampingVUsExecutionTupleTests/2/3:1_in_0,1/3,2/3,1
=== PAUSE TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_with_funky_sequence
=== PAUSE TestRampingVUsExecutionTupleTests/2/3:1_in_0,1/3,2/3,1
=== RUN   TestRampingVUsGetRawExecutionStepsCornerCases/strange
=== CONT  TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,1
=== CONT  TestRampingVUsExecutionTupleTests/2/3:1_in_0,1/3,2/3,1
=== CONT  TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,1#01
=== CONT  TestRampingVUsExecutionTupleTests/2/3:1_in_0,2/3,1
=== CONT  TestRampingVUsExecutionTupleTests/1/3:2/3_in_0,1/3,2/3,1
=== CONT  TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,2/3,1
=== PAUSE TestRampingVUsGetRawExecutionStepsCornerCases/strange
=== CONT  TestRampingVUsExecutionTupleTests/0:1_in_0,1
=== CONT  TestRampingVUsExecutionTupleTests/1/3:2/3_in_0,1/3,2/3,1#01
=== RUN   TestRampingVUsGetRawExecutionStepsCornerCases/more_up_and_down
=== PAUSE TestRampingVUsGetRawExecutionStepsCornerCases/more_up_and_down
=== CONT  TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_the_other_half
=== CONT  TestRampingVUsGetRawExecutionStepsCornerCases/strange
--- PASS: TestRampingVUsExecutionTupleTests (0.00s)
    --- PASS: TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,1 (0.00s)
    --- PASS: TestRampingVUsExecutionTupleTests/2/3:1_in_0,1/3,2/3,1 (0.00s)
    --- PASS: TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,1#01 (0.00s)
    --- PASS: TestRampingVUsExecutionTupleTests/2/3:1_in_0,2/3,1 (0.00s)
    --- PASS: TestRampingVUsExecutionTupleTests/0:1/3_in_0,1/3,2/3,1 (0.00s)
    --- PASS: TestRampingVUsExecutionTupleTests/1/3:2/3_in_0,1/3,2/3,1 (0.00s)
    --- PASS: TestRampingVUsExecutionTupleTests/1/3:2/3_in_0,1/3,2/3,1#01 (0.00s)
    --- PASS: TestRampingVUsExecutionTupleTests/0:1_in_0,1 (0.00s)
=== CONT  TestRampingVUsGetRawExecutionStepsCornerCases/more_up_and_down
=== CONT  TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_with_funky_sequence
=== CONT  TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_with_nothing
=== CONT  TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down
=== CONT  TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_half
=== CONT  TestRampingVUsGetRawExecutionStepsCornerCases/jump_up_then_go_up_again
=== CONT  TestRampingVUsGetRawExecutionStepsCornerCases/going_up_then_down_straight_away
--- PASS: TestRampingVUsGetRawExecutionStepsCornerCases (0.00s)
    --- PASS: TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_the_other_half (0.00s)
    --- PASS: TestRampingVUsGetRawExecutionStepsCornerCases/strange (0.00s)
    --- PASS: TestRampingVUsGetRawExecutionStepsCornerCases/more_up_and_down (0.00s)
    --- PASS: TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_with_funky_sequence (0.00s)
    --- PASS: TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_with_nothing (0.00s)
    --- PASS: TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down (0.00s)
    --- PASS: TestRampingVUsGetRawExecutionStepsCornerCases/up_down_up_down_in_half (0.00s)
    --- PASS: TestRampingVUsGetRawExecutionStepsCornerCases/jump_up_then_go_up_again (0.00s)
    --- PASS: TestRampingVUsGetRawExecutionStepsCornerCases/going_up_then_down_straight_away (0.00s)
--- PASS: TestRampingVUsHandleRemainingVUs (0.07s)
--- PASS: TestRampingVUsGracefulStopWaits (1.50s)
--- PASS: TestRampingVUsRun (2.40s)
--- PASS: TestRampingVUsGracefulStopStops (2.50s)
--- PASS: TestRampingVUsGracefulRampDown (2.50s)
--- PASS: TestRampingVUsRampDownNoWobble (6.02s)
PASS
ok  	go.k6.io/k6/lib/executor	13.063s
```

**Conclusion for Question (a):** the two handler strategies are serialized by construction (§4.1), and
the broader concurrent machinery (per-VU loops, progress, graceful tail) is exercised under `-race`
with zero data races across two iterations of each suite. No data race was found on the exercised
paths. (Standard caveat: `-race` instruments only the interleavings that actually occur during the
run, so a clean result is strong but not an absolute proof of race-freedom for all possible schedules.)


---

## 5. Question (c): the per-VU state machine and its (mixed) synchronization model

### 5.1 The five states and the single per-VU mutex

Each VU is driven by a `vuHandle` whose `state` field `lib/executor/vu_handle.go:82` takes one of five
values declared as an `iota` block `lib/executor/vu_handle.go:17-21`: `stopped`, `starting`, `running`,
`toGracefulStop`, `toHardStop`. The handle carries one `mutex` `lib/executor/vu_handle.go:71`. The
three control methods each lock that same mutex before touching state:

- `start` locks at `lib/executor/vu_handle.go:116`,
- `gracefulStop` locks at `lib/executor/vu_handle.go:148`,
- `hardStop` locks at `lib/executor/vu_handle.go:166`.

Because it is the **same** lock, the three methods' critical sections are mutually exclusive: even if
their callers were concurrent, the bodies execute one at a time. That is the core of the answer to
Question (c) — there is no window in which "both handlers modify VU state simultaneously."

### 5.2 The synchronization model is mixed, not "atomics only"

A natural assumption is that the state is guarded purely by atomics. That is **not** what the code does;
the actual model is a deliberate mix:

- **Plain initialization** at construction: `state: stopped` is a plain field assignment
  `lib/executor/vu_handle.go:106`, done before any goroutine can observe the handle (safe by
  happens-before of goroutine creation).
- **Atomic writes**: every state change goes through `changeState`, which does an
  `atomic.StoreInt32` `lib/executor/vu_handle.go:144`; every caller of `changeState` holds the mutex.
- **Plain reads under the mutex**: the `switch vh.state` dispatch inside `start` `:119`,
  `gracefulStop` `:150`, `hardStop` `:169`, and the run-loop slow path `:219` are ordinary
  (non-atomic) reads — safe because they are performed while holding the mutex and all writes are
  atomic stores.
- **One lock-free atomic read on the hot path**: the VU run loop's fast path reads the state with
  `atomic.LoadInt32` **without** taking the mutex `lib/executor/vu_handle.go:204`; this is safe against
  the atomic stores and is the performance-sensitive path that avoids lock contention per iteration.

So the precise answer is: *atomic stores + plain reads performed under the mutex + a single lock-free
atomic-load fast path + a plain pre-goroutine initialization*. The transition table
`lib/executor/vu_handle.go:24-55` documents how deliberately-raced orderings resolve — for example, a
`start` that arrives while the VU is in `toGracefulStop` simply continues running ("we raced with the
loop stopping, just continue").

### 5.3 Runtime trace of a single VU through the state machine (canonical, DebugLevel)

**Command:**

```text
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -run 'TestBlitzyProbeStateTransitions' ./lib/executor/ -v
```

This probe drives one real `vuHandle` (constructed via `newStoppedVUHandle` with its run loop started
on its own goroutine, exactly as the executor does) through `start → gracefulStop → start → hardStop`,
observing the state via the real lock-free fast-path read (`atomic.LoadInt32`), capturing the actual
DebugLevel transition log lines, and counting VU acquire/return calls. The `gracefulStop` passes
through the transient `toGracefulStop` state — the VU finishes its in-flight ~50 ms iteration and only
then settles to `stopped` via the slow path — which is the same lag mechanism that makes VUs *look*
"stuck" in Symptom A (§7-A). Acquire/return is one-to-one (`getVU=2 == returnVU=2`), the invariant noted
at `lib/executor/vu_handle.go:62`:

```text
=== RUN   TestBlitzyProbeStateTransitions
=== PAUSE TestBlitzyProbeStateTransitions
=== CONT  TestBlitzyProbeStateTransitions
    blitzy_adhoc_test_probe_test.go:165: Q(c) single-vuHandle state-machine trace (initial state=stopped):
    blitzy_adhoc_test_probe_test.go:170:   start()        : stopped        -> running
    blitzy_adhoc_test_probe_test.go:175:   gracefulStop() : running        -> stopped
    blitzy_adhoc_test_probe_test.go:180:   start()        : stopped        -> running
    blitzy_adhoc_test_probe_test.go:185:   hardStop()     : running        -> stopped
    blitzy_adhoc_test_probe_test.go:190:   after cancel() : final state=stopped
    blitzy_adhoc_test_probe_test.go:194: Captured vuHandle debug transition lines (Message @ level):
    blitzy_adhoc_test_probe_test.go:196:     level=debug msg="Start" vuNum=0
    blitzy_adhoc_test_probe_test.go:196:     level=debug msg="Graceful stop" vuNum=0
    blitzy_adhoc_test_probe_test.go:196:     level=debug msg="Start" vuNum=0
    blitzy_adhoc_test_probe_test.go:196:     level=debug msg="Hard stop" vuNum=0
    blitzy_adhoc_test_probe_test.go:198: vuHandle-level acquire/return accounting: getVU=2 returnVU=2 (must be equal, invariant vu_handle.go:62)
--- PASS: TestBlitzyProbeStateTransitions (0.46s)
PASS
ok  	go.k6.io/k6/lib/executor	0.468s
```

**Conclusion for Question (c):** the three control methods serialize on one per-VU mutex (§5.1); state
uses a mixed atomic-store / mutex-protected-read / lock-free-fast-read model (§5.2); the transition
table resolves raced orderings by design; and both the deterministic trace above and the clean
`TestVUHandleRace` / `TestVUHandleStartStopRace` results under `-race` (§4.4) corroborate that no unsafe
simultaneous modification occurs.


---

## 6. Question (b): VU-buffer accounting and the depleted-buffer failure path

### 6.1 What the buffer is, and the one-to-one accounting

The VU buffer is the `es.vus` channel inside `ExecutionState`. A VU is acquired for a scheduled slot via
`GetPlannedVU` `lib/execution.go:471` and returned via `ReturnVU`, which pushes it back onto the channel
`lib/execution.go:544-546`. In `ramping-vus`, acquisition and return are wrapped by the `getVU` /
`returnVU` accounting closures `lib/executor/ramping_vus.go:592-610`:

- `getVU` calls `GetPlannedVU`; on **error it returns before** doing `wg.Add(1)`, incrementing the
  active counter, or calling `ModCurrentlyActiveVUsCount(+1)` `lib/executor/ramping_vus.go:595-602` — so
  a *failed* acquisition is never paired with a spurious return;
- `returnVU` calls `ReturnVU`, decrements the active counter, calls `wg.Done()`, and does
  `ModCurrentlyActiveVUsCount(-1)` `lib/executor/ramping_vus.go:605-610`.

### 6.2 Caveat on the observable

The active-VU counter read by `GetCurrentlyActiveVUsCount` is explicitly documented as a UI/reporting
signal — "*don't use it for synchronization*" `lib/execution.go:268-270`. So on its own, a net-zero
active count does **not** prove one-return-per-acquire. It becomes meaningful only *together* with a
direct buffer check: the value it returns is the `es.activeVUs` field `lib/execution.go:270`, which the
`getVU` closure increments via `ModCurrentlyActiveVUsCount(+1)` `lib/executor/ramping_vus.go:602` and the
`returnVU` closure decrements via `ModCurrentlyActiveVUsCount(-1)` `lib/executor/ramping_vus.go:609` — in
the same closures that call `GetPlannedVU`/`ReturnVU`. (The adjacent `atomic.AddInt64(rs.activeVUsCount, ±1)`
at `lib/executor/ramping_vus.go:601`/`:607` maintains a *separate* progress-display counter,
`rs.activeVUsCount` `lib/executor/ramping_vus.go:569`, in lockstep on the same two lines, so the observable
and the progress counter always hold equal values.) Therefore the evidence below pairs **(i)** net-zero active count with **(ii)**
a direct drain of the buffer after the run, and **(iii)** the direct one-to-one `getVU==returnVU` count
from §5.3. "No leak" is asserted for these observed paths.

### 6.3 Buffer restoration after a rapid up/down run (canonical, reproduced ×2 and under `-race`)

**Command:**

```text
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -run 'TestBlitzyProbeBufferAccounting' ./lib/executor/ -v
```

Config (disclosed): rapid up/down 8↔0 at 1 s per stage ×4, `GracefulRampDown=3s`, `GracefulStop=1s`.
The probe records the buffer size before the run, the peak active count during, the active count after
`Run` returns, and then **drains the buffer** via the public `GetPlannedVU` to confirm every planned VU
is present. Because `Run` blocks on `defer runState.wg.Wait()` `lib/executor/ramping_vus.go:540`, when it
returns every `returnVU` has already completed, so a full 8-of-8 drain proves nothing was orphaned:

```text
=== RUN   TestBlitzyProbeBufferAccounting
=== PAUSE TestBlitzyProbeBufferAccounting
=== CONT  TestBlitzyProbeBufferAccounting
    blitzy_adhoc_test_probe_test.go:240: Q(b) buffer accounting — config: rapid up/down 8<->0 @1s x4, GracefulRampDown=3s, GracefulStop=1s
    blitzy_adhoc_test_probe_test.go:241:   initialized VUs in buffer at start: 8
    blitzy_adhoc_test_probe_test.go:242:   active-VU count BEFORE Run: 0
    blitzy_adhoc_test_probe_test.go:268:   active-VU count AFTER Run returns: 0 (peak observed during run: 8)
    blitzy_adhoc_test_probe_test.go:287:   buffer restoration: drained 8/8 planned VUs in 0s (all present => none orphaned)
--- PASS: TestBlitzyProbeBufferAccounting (4.06s)
PASS
ok  	go.k6.io/k6/lib/executor	4.061s
```

Re-running the identical input with `-count=2` shows the same result both times:

```text
=== RUN   TestBlitzyProbeBufferAccounting
=== PAUSE TestBlitzyProbeBufferAccounting
=== CONT  TestBlitzyProbeBufferAccounting
    blitzy_adhoc_test_probe_test.go:240: Q(b) buffer accounting — config: rapid up/down 8<->0 @1s x4, GracefulRampDown=3s, GracefulStop=1s
    blitzy_adhoc_test_probe_test.go:241:   initialized VUs in buffer at start: 8
    blitzy_adhoc_test_probe_test.go:242:   active-VU count BEFORE Run: 0
    blitzy_adhoc_test_probe_test.go:268:   active-VU count AFTER Run returns: 0 (peak observed during run: 8)
    blitzy_adhoc_test_probe_test.go:287:   buffer restoration: drained 8/8 planned VUs in 0s (all present => none orphaned)
--- PASS: TestBlitzyProbeBufferAccounting (4.05s)
=== RUN   TestBlitzyProbeBufferAccounting
=== PAUSE TestBlitzyProbeBufferAccounting
=== CONT  TestBlitzyProbeBufferAccounting
    blitzy_adhoc_test_probe_test.go:240: Q(b) buffer accounting — config: rapid up/down 8<->0 @1s x4, GracefulRampDown=3s, GracefulStop=1s
    blitzy_adhoc_test_probe_test.go:241:   initialized VUs in buffer at start: 8
    blitzy_adhoc_test_probe_test.go:242:   active-VU count BEFORE Run: 0
    blitzy_adhoc_test_probe_test.go:268:   active-VU count AFTER Run returns: 0 (peak observed during run: 8)
    blitzy_adhoc_test_probe_test.go:287:   buffer restoration: drained 8/8 planned VUs in 0s (all present => none orphaned)
--- PASS: TestBlitzyProbeBufferAccounting (4.06s)
PASS
ok  	go.k6.io/k6/lib/executor	8.116s
```

And under the race detector (`-race -count=2`) the same accounting reproduces with **zero** data races
(footer only shown; the per-iteration body is identical to the block above):

```text
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -race -run 'TestBlitzyProbeBufferAccounting' ./lib/executor/ -count=2
```

```text
ok  	go.k6.io/k6/lib/executor	9.150s
```

### 6.4 The depleted-buffer failure path (edge/error path, exercised directly)

The user's "leak" hypothesis motivates checking the failure path: what happens if a scheduled slot tries
to acquire a VU but the buffer is empty? `GetPlannedVU` retries up to `MaxRetriesGetPlannedVU = 5`
`lib/execution.go:29`, warning on each attempt `lib/execution.go:481`, then returns a final error
`lib/execution.go:484-487`. The probe calls `GetPlannedVU` on an execution state with **0** initialized
VUs and captures the real warnings and the terminal error:

**Command:**

```text
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -run 'TestBlitzyProbeDepletedBuffer' ./lib/executor/ -v
```

```text
=== RUN   TestBlitzyProbeDepletedBuffer
=== PAUSE TestBlitzyProbeDepletedBuffer
=== CONT  TestBlitzyProbeDepletedBuffer
    blitzy_adhoc_test_probe_test.go:317: Q(b) depleted-buffer failure path — calling GetPlannedVU on an EMPTY buffer (0 initialized VUs):
    blitzy_adhoc_test_probe_test.go:322:     Could not get a VU from the buffer for 400ms
    blitzy_adhoc_test_probe_test.go:322:     Could not get a VU from the buffer for 800ms
    blitzy_adhoc_test_probe_test.go:322:     Could not get a VU from the buffer for 1.2s
    blitzy_adhoc_test_probe_test.go:322:     Could not get a VU from the buffer for 1.6s
    blitzy_adhoc_test_probe_test.go:322:     Could not get a VU from the buffer for 2s
    blitzy_adhoc_test_probe_test.go:324:   returned: vu==nil? true ; err="could not get a VU from the buffer in 2s" ; elapsed=2s (expect ~5x400ms=2s)
--- PASS: TestBlitzyProbeDepletedBuffer (2.00s)
PASS
ok  	go.k6.io/k6/lib/executor	2.009s
```

The acquisition fails **cleanly and boundedly** — five ~400 ms retries totaling ~2 s, then a `nil` VU and
an error — rather than blocking forever or orphaning a handle. Combined with the fact that `getVU`
returns before any increment on this error path `lib/executor/ramping_vus.go:595-600`, a depleted buffer
cannot produce a mis-balanced acquire/return.

**Conclusion for Question (b):** no VU-buffer leak was observed on the exercised paths — the buffer is
fully restored after the run (§6.3), acquire/return is one-to-one (§5.3), and the depleted-buffer failure
path fails cleanly rather than leaking (§6.4).


---

## 7. Per-symptom reproductions

### 7-A. Symptom A — VUs "stuck" between active and stopped during rapid up/down with long `gracefulRampDown`

**Direct answer:** VUs are **not** permanently stuck. The "stuck" appearance is a transient, **bounded**
graceful-ramp-down lag: during a rapid down-stage the scheduled target drops faster than VUs that are
mid-iteration can finish, so those VUs stay counted as active (in the transient `toGracefulStop` state —
**INFERRED**, not directly sampled because the `vuHandles` slice lives in a `Run`-local struct, from its
definition at `lib/executor/vu_handle.go:20` within the state `iota` block `lib/executor/vu_handle.go:17-21`
and the transition table `lib/executor/vu_handle.go:24-55`) until they complete their current short
iteration and return. Because the iteration (300 ms) is far shorter than `GracefulRampDown` (3 s), they
always finish and the active count always returns to 0. (The temporary probe's own `INFERRED` log line —
preserved verbatim in the §7-A transcripts below and in the §9.1 probe source — cites the adjacent
`vu_handle.go:19`, which is the `running` state; the precise definition of `toGracefulStop` is
`vu_handle.go:20`. The captured log text is left unedited; this note supplies the exact line.)

**Method.** The user's scenario ("stages that go up and down rapidly with a long `gracefulRampDown`") is
reproduced as a `RampingVUsConfig` ramping 8↔0 at 1 s per stage ×4, `StartVUs=0`, `GracefulRampDown=3s`,
`GracefulStop=1s`, iteration sleep 300 ms. Because the user reports intermittency ("sometimes"), the
**same unchanged input is run 10 times** and the distribution of outcomes is reported (Rule 1). The
canonical observable is `GetCurrentlyActiveVUsCount` `lib/execution.go:269` — the same signal the in-repo
"no wobble" test samples — printed every 100 ms and paired with the scheduled target
(`PlannedVUs` from `rawSteps`) so the lag is visible; a `*` marks samples where active exceeds target.

**Command:**

```text
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -run 'TestBlitzyProbeStuckVUsDistribution' ./lib/executor/ -v
```

The complete output for all 10 identical runs follows (each run: `before(t=0)`, the full 100 ms-sampled
trajectory with per-sample target and lag, `peak`, `final(after Run)`, and transition counts):

```text
=== RUN   TestBlitzyProbeStuckVUsDistribution
=== PAUSE TestBlitzyProbeStuckVUsDistribution
=== CONT  TestBlitzyProbeStuckVUsDistribution
    blitzy_adhoc_test_probe_test.go:480: Symptom A — config: rapid up/down 8<->0 @1s x4 (StartVUs=0), GracefulRampDown=3s, GracefulStop=1s, iteration sleep=300ms; 10 identical (unchanged-input) runs
    blitzy_adhoc_test_probe_test.go:482: Symptom A — OBSERVED signal = GetCurrentlyActiveVUsCount [lib/execution.go:269]; 'target' column = scheduled PlannedVUs from rawSteps (the scheduledVUsHandlerStrategy goal).
    blitzy_adhoc_test_probe_test.go:484: Symptom A — INFERRED (from vu_handle.go:19,24-55, not directly sampled): when active > target during a down-stage, the surplus VUs are in the transient toGracefulStop state — mid-iteration and finishing their current ~300ms iteration before returnVU; this is the 'stuck'-looking lag.
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 1/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=8 target=8 lag=+0
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=4 target=4 lag=+0
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=8 target=8 lag=+0
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 2/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=8 target=8 lag=+0
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=7 target=6 lag=+1*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 3/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=8 target=8 lag=+0
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=7 target=6 lag=+1*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 4/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=8 target=8 lag=+0
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=7 target=6 lag=+1*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=4 target=4 lag=+0
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=8 target=8 lag=+0
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=7 target=6 lag=+1*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 5/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=8 target=8 lag=+0
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=7 target=6 lag=+1*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=8 target=8 lag=+0
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=7 target=6 lag=+1*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 6/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=8 target=8 lag=+0
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=4 target=4 lag=+0
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=8 target=8 lag=+0
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 7/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=8 target=8 lag=+0
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=4 target=4 lag=+0
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=8 target=8 lag=+0
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=7 target=6 lag=+1*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 8/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=8 target=8 lag=+0
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=4 target=4 lag=+0
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=8 target=8 lag=+0
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 9/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=8 target=8 lag=+0
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=7 target=6 lag=+1*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=4 target=4 lag=+0
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=7 target=6 lag=+1*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 10/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=4 target=4 lag=+0
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=8 target=8 lag=+0
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=7 target=6 lag=+1*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=4 target=4 lag=+0
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=8 target=8 lag=+0
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:591: Symptom A — DISTRIBUTION over 10 identical runs (value×count):
    blitzy_adhoc_test_probe_test.go:592:   peak active-VU count:                 8×10
    blitzy_adhoc_test_probe_test.go:593:   final active-VU count (after Run):    0×10
    blitzy_adhoc_test_probe_test.go:594:   max transient surplus (active-target): 2×10
    blitzy_adhoc_test_probe_test.go:595:   runs leaving a permanently STUCK VU (final != 0): 0/10
--- PASS: TestBlitzyProbeStuckVUsDistribution (40.54s)
PASS
ok  	go.k6.io/k6/lib/executor	40.542s
```

**Reading the trajectory.** Active VUs track the target exactly while ramping up; during each rapid
down-stage active exceeds target by at most **+2** (the `*` samples), then decays. Crucially, `final`
is **0** in every run, and the transition totals are `Start=16, GracefulStop=16, HardStop=0` — see §7-B
for why there are zero `Hard stop` lines. The surplus is the "stuck"-looking population, and it is
transient and bounded, never permanent.

**Reproducibility and race-freedom (Rule 1 + Question a).** Re-running the identical 10-run input under
the race detector with `-count=2` (i.e., 20 runs total) yields the **same** distribution both times and
**zero** data races:

```text
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -race -run 'TestBlitzyProbeStuckVUsDistribution' ./lib/executor/ -count=2 -v -timeout 300s
```

```text
=== RUN   TestBlitzyProbeStuckVUsDistribution
=== PAUSE TestBlitzyProbeStuckVUsDistribution
=== CONT  TestBlitzyProbeStuckVUsDistribution
    blitzy_adhoc_test_probe_test.go:480: Symptom A — config: rapid up/down 8<->0 @1s x4 (StartVUs=0), GracefulRampDown=3s, GracefulStop=1s, iteration sleep=300ms; 10 identical (unchanged-input) runs
    blitzy_adhoc_test_probe_test.go:482: Symptom A — OBSERVED signal = GetCurrentlyActiveVUsCount [lib/execution.go:269]; 'target' column = scheduled PlannedVUs from rawSteps (the scheduledVUsHandlerStrategy goal).
    blitzy_adhoc_test_probe_test.go:484: Symptom A — INFERRED (from vu_handle.go:19,24-55, not directly sampled): when active > target during a down-stage, the surplus VUs are in the transient toGracefulStop state — mid-iteration and finishing their current ~300ms iteration before returnVU; this is the 'stuck'-looking lag.
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 1/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 2/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 3/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 4/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 5/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 6/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 7/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 8/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 9/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 10/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:591: Symptom A — DISTRIBUTION over 10 identical runs (value×count):
    blitzy_adhoc_test_probe_test.go:592:   peak active-VU count:                 8×10
    blitzy_adhoc_test_probe_test.go:593:   final active-VU count (after Run):    0×10
    blitzy_adhoc_test_probe_test.go:594:   max transient surplus (active-target): 2×10
    blitzy_adhoc_test_probe_test.go:595:   runs leaving a permanently STUCK VU (final != 0): 0/10
--- PASS: TestBlitzyProbeStuckVUsDistribution (40.55s)
=== RUN   TestBlitzyProbeStuckVUsDistribution
=== PAUSE TestBlitzyProbeStuckVUsDistribution
=== CONT  TestBlitzyProbeStuckVUsDistribution
    blitzy_adhoc_test_probe_test.go:480: Symptom A — config: rapid up/down 8<->0 @1s x4 (StartVUs=0), GracefulRampDown=3s, GracefulStop=1s, iteration sleep=300ms; 10 identical (unchanged-input) runs
    blitzy_adhoc_test_probe_test.go:482: Symptom A — OBSERVED signal = GetCurrentlyActiveVUsCount [lib/execution.go:269]; 'target' column = scheduled PlannedVUs from rawSteps (the scheduledVUsHandlerStrategy goal).
    blitzy_adhoc_test_probe_test.go:484: Symptom A — INFERRED (from vu_handle.go:19,24-55, not directly sampled): when active > target during a down-stage, the surplus VUs are in the transient toGracefulStop state — mid-iteration and finishing their current ~300ms iteration before returnVU; this is the 'stuck'-looking lag.
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 1/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 2/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 3/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 4/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 5/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 6/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 7/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 8/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 9/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:555: Symptom A run 10/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:568:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
              t=100ms  active=0 target=0 lag=+0
              t=200ms  active=1 target=1 lag=+0
              t=300ms  active=2 target=2 lag=+0
              t=400ms  active=3 target=3 lag=+0
              t=500ms  active=3 target=4 lag=-1
              t=600ms  active=4 target=4 lag=+0
              t=700ms  active=5 target=5 lag=+0
              t=800ms  active=6 target=6 lag=+0
              t=900ms  active=7 target=7 lag=+0
              t=1s     active=7 target=8 lag=-1
              t=1.1s   active=8 target=8 lag=+0
              t=1.2s   active=8 target=7 lag=+1*
              t=1.3s   active=8 target=6 lag=+2*
              t=1.4s   active=7 target=5 lag=+2*
              t=1.5s   active=6 target=4 lag=+2*
              t=1.6s   active=5 target=4 lag=+1*
              t=1.7s   active=4 target=3 lag=+1*
              t=1.8s   active=3 target=2 lag=+1*
              t=1.9s   active=2 target=1 lag=+1*
              t=2s     active=2 target=0 lag=+2*
              t=2.1s   active=1 target=0 lag=+1*
              t=2.2s   active=1 target=1 lag=+0
              t=2.3s   active=2 target=2 lag=+0
              t=2.4s   active=3 target=3 lag=+0
              t=2.5s   active=3 target=4 lag=-1
              t=2.6s   active=4 target=4 lag=+0
              t=2.7s   active=5 target=5 lag=+0
              t=2.8s   active=6 target=6 lag=+0
              t=2.9s   active=7 target=7 lag=+0
              t=3s     active=7 target=8 lag=-1
              t=3.1s   active=8 target=8 lag=+0
              t=3.2s   active=8 target=7 lag=+1*
              t=3.3s   active=8 target=6 lag=+2*
              t=3.4s   active=7 target=5 lag=+2*
              t=3.5s   active=6 target=4 lag=+2*
              t=3.6s   active=5 target=4 lag=+1*
              t=3.7s   active=4 target=3 lag=+1*
              t=3.8s   active=3 target=2 lag=+1*
              t=3.9s   active=2 target=1 lag=+1*
              t=4s     active=2 target=0 lag=+2*
    blitzy_adhoc_test_probe_test.go:591: Symptom A — DISTRIBUTION over 10 identical runs (value×count):
    blitzy_adhoc_test_probe_test.go:592:   peak active-VU count:                 8×10
    blitzy_adhoc_test_probe_test.go:593:   final active-VU count (after Run):    0×10
    blitzy_adhoc_test_probe_test.go:594:   max transient surplus (active-target): 2×10
    blitzy_adhoc_test_probe_test.go:595:   runs leaving a permanently STUCK VU (final != 0): 0/10
--- PASS: TestBlitzyProbeStuckVUsDistribution (40.55s)
PASS
ok  	go.k6.io/k6/lib/executor	82.121s
```

**Distribution over the 10 identical runs** (identical across both `-race` iterations):

- peak active-VU count: **8 ×10**
- final active-VU count after `Run` returns: **0 ×10**  ← no run leaves a permanently stuck VU
- max transient surplus (active − target): **2 ×10**
- runs leaving a permanently STUCK VU (`final != 0`): **0 / 10**

This is authoritative for "not permanently stuck" because `Run` blocks on
`defer runState.wg.Wait()` `lib/executor/ramping_vus.go:540`, so the `final` sample is taken **after**
every `returnVU` has completed; `0 ×10` proves nothing was orphaned.


### 7-B. Symptom B — the scheduled handler's count doesn't match the graceful handler's

**Direct answer:** This mismatch is **expected by design**, not a defect. The two handlers track
different quantities: `scheduledVUsHandlerStrategy` tracks the **scheduled target** (how many VUs *should
be actively iterating right now*), while `maxAllowedVUsHandlerStrategy` tracks the **max-allowed ceiling**
(how many VUs are *allowed to still exist*, kept high during a graceful ramp-down so in-flight iterations
can finish). Each keeps its own closure-local `cur` counter (`lib/executor/ramping_vus.go:669` and
`:680`). During a down-stage the ceiling deliberately holds above the target, so
`ceiling >= target` — the exact divergence the user noticed.

**Method.** Same config as §7-A. The probe reads the executor's two pre-computed step lists after
`Init` (`rawSteps` and `gracefulSteps`), prints both as tables, reconstructs the ceiling-vs-target
relationship over time, and then runs the executor with a DebugLevel hook to capture the **real**
`Start` / `Graceful stop` / `Hard stop` transition log lines with timestamps relative to `Run` start.

**On what is measured vs. inferred.** The two `cur` counters are closure-local and unexported, so they
cannot be logged directly; the ceiling and target series below are **reconstructed (INFERRED)** from the
step tables the handlers consume (this is labeled as such in the output). The transition lines, by
contrast, are **real** DebugLevel log lines emitted by `start`/`gracefulStop`/`hardStop`. Note that a
handler call that is a no-op (e.g., `hardStop` on a VU that is already stopped) does **not** emit a log
line `lib/executor/vu_handle.go:169`.

**Command:**

```text
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -run 'TestBlitzyProbeHandlerDivergence' ./lib/executor/ -v
```

```text
=== RUN   TestBlitzyProbeHandlerDivergence
=== PAUSE TestBlitzyProbeHandlerDivergence
=== CONT  TestBlitzyProbeHandlerDivergence
    blitzy_adhoc_test_probe_test.go:353: Symptom B — raw steps (scheduledVUsHandlerStrategy target, cur := raw.PlannedVUs):
    blitzy_adhoc_test_probe_test.go:354:     idx |  TimeOffset | PlannedVUs
    blitzy_adhoc_test_probe_test.go:356:       0 |         0s | 0
    blitzy_adhoc_test_probe_test.go:356:       1 |      125ms | 1
    blitzy_adhoc_test_probe_test.go:356:       2 |      250ms | 2
    blitzy_adhoc_test_probe_test.go:356:       3 |      375ms | 3
    blitzy_adhoc_test_probe_test.go:356:       4 |      500ms | 4
    blitzy_adhoc_test_probe_test.go:356:       5 |      625ms | 5
    blitzy_adhoc_test_probe_test.go:356:       6 |      750ms | 6
    blitzy_adhoc_test_probe_test.go:356:       7 |      875ms | 7
    blitzy_adhoc_test_probe_test.go:356:       8 |         1s | 8
    blitzy_adhoc_test_probe_test.go:356:       9 |     1.125s | 7
    blitzy_adhoc_test_probe_test.go:356:      10 |      1.25s | 6
    blitzy_adhoc_test_probe_test.go:356:      11 |     1.375s | 5
    blitzy_adhoc_test_probe_test.go:356:      12 |       1.5s | 4
    blitzy_adhoc_test_probe_test.go:356:      13 |     1.625s | 3
    blitzy_adhoc_test_probe_test.go:356:      14 |      1.75s | 2
    blitzy_adhoc_test_probe_test.go:356:      15 |     1.875s | 1
    blitzy_adhoc_test_probe_test.go:356:      16 |         2s | 0
    blitzy_adhoc_test_probe_test.go:356:      17 |     2.125s | 1
    blitzy_adhoc_test_probe_test.go:356:      18 |      2.25s | 2
    blitzy_adhoc_test_probe_test.go:356:      19 |     2.375s | 3
    blitzy_adhoc_test_probe_test.go:356:      20 |       2.5s | 4
    blitzy_adhoc_test_probe_test.go:356:      21 |     2.625s | 5
    blitzy_adhoc_test_probe_test.go:356:      22 |      2.75s | 6
    blitzy_adhoc_test_probe_test.go:356:      23 |     2.875s | 7
    blitzy_adhoc_test_probe_test.go:356:      24 |         3s | 8
    blitzy_adhoc_test_probe_test.go:356:      25 |     3.125s | 7
    blitzy_adhoc_test_probe_test.go:356:      26 |      3.25s | 6
    blitzy_adhoc_test_probe_test.go:356:      27 |     3.375s | 5
    blitzy_adhoc_test_probe_test.go:356:      28 |       3.5s | 4
    blitzy_adhoc_test_probe_test.go:356:      29 |     3.625s | 3
    blitzy_adhoc_test_probe_test.go:356:      30 |      3.75s | 2
    blitzy_adhoc_test_probe_test.go:356:      31 |     3.875s | 1
    blitzy_adhoc_test_probe_test.go:356:      32 |         4s | 0
    blitzy_adhoc_test_probe_test.go:358: Symptom B — graceful steps (maxAllowedVUsHandlerStrategy ceiling, cur := graceful.PlannedVUs):
    blitzy_adhoc_test_probe_test.go:359:     idx |  TimeOffset | PlannedVUs
    blitzy_adhoc_test_probe_test.go:361:       0 |         0s | 0
    blitzy_adhoc_test_probe_test.go:361:       1 |      125ms | 1
    blitzy_adhoc_test_probe_test.go:361:       2 |      250ms | 2
    blitzy_adhoc_test_probe_test.go:361:       3 |      375ms | 3
    blitzy_adhoc_test_probe_test.go:361:       4 |      500ms | 4
    blitzy_adhoc_test_probe_test.go:361:       5 |      625ms | 5
    blitzy_adhoc_test_probe_test.go:361:       6 |      750ms | 6
    blitzy_adhoc_test_probe_test.go:361:       7 |      875ms | 7
    blitzy_adhoc_test_probe_test.go:361:       8 |         1s | 8
    blitzy_adhoc_test_probe_test.go:361:       9 |         5s | 0
    blitzy_adhoc_test_probe_test.go:365: Symptom B — reconstructed ceiling-vs-target over time (INFERRED from steps above):
    blitzy_adhoc_test_probe_test.go:366:     time(ms) | scheduled target | max-allowed ceiling | ceiling>=target?
    blitzy_adhoc_test_probe_test.go:371:            0 |                0 |                   0 | true
    blitzy_adhoc_test_probe_test.go:371:          125 |                1 |                   1 | true
    blitzy_adhoc_test_probe_test.go:371:          250 |                2 |                   2 | true
    blitzy_adhoc_test_probe_test.go:371:          375 |                3 |                   3 | true
    blitzy_adhoc_test_probe_test.go:371:          500 |                4 |                   4 | true
    blitzy_adhoc_test_probe_test.go:371:          625 |                5 |                   5 | true
    blitzy_adhoc_test_probe_test.go:371:          750 |                6 |                   6 | true
    blitzy_adhoc_test_probe_test.go:371:          875 |                7 |                   7 | true
    blitzy_adhoc_test_probe_test.go:371:         1000 |                8 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         1125 |                7 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         1250 |                6 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         1375 |                5 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         1500 |                4 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         1625 |                3 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         1750 |                2 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         1875 |                1 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         2000 |                0 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         2125 |                1 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         2250 |                2 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         2375 |                3 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         2500 |                4 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         2625 |                5 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         2750 |                6 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         2875 |                7 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         3000 |                8 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         3125 |                7 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         3250 |                6 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         3375 |                5 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         3500 |                4 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         3625 |                3 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         3750 |                2 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         3875 |                1 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         4000 |                0 |                   8 | true
    blitzy_adhoc_test_probe_test.go:371:         5000 |                0 |                   0 | true
    blitzy_adhoc_test_probe_test.go:407: Symptom B — observed vuHandle transitions (real debug lines, relative to Run start):
    blitzy_adhoc_test_probe_test.go:409:     t=   126ms  Start         vuNum=0
    blitzy_adhoc_test_probe_test.go:409:     t=   250ms  Start         vuNum=1
    blitzy_adhoc_test_probe_test.go:409:     t=   376ms  Start         vuNum=2
    blitzy_adhoc_test_probe_test.go:409:     t=   501ms  Start         vuNum=3
    blitzy_adhoc_test_probe_test.go:409:     t=   626ms  Start         vuNum=4
    blitzy_adhoc_test_probe_test.go:409:     t=   750ms  Start         vuNum=5
    blitzy_adhoc_test_probe_test.go:409:     t=   876ms  Start         vuNum=6
    blitzy_adhoc_test_probe_test.go:409:     t=  1.001s  Start         vuNum=7
    blitzy_adhoc_test_probe_test.go:409:     t=  1.125s  Graceful stop vuNum=7
    blitzy_adhoc_test_probe_test.go:409:     t=   1.25s  Graceful stop vuNum=6
    blitzy_adhoc_test_probe_test.go:409:     t=  1.375s  Graceful stop vuNum=5
    blitzy_adhoc_test_probe_test.go:409:     t=  1.501s  Graceful stop vuNum=4
    blitzy_adhoc_test_probe_test.go:409:     t=  1.625s  Graceful stop vuNum=3
    blitzy_adhoc_test_probe_test.go:409:     t=   1.75s  Graceful stop vuNum=2
    blitzy_adhoc_test_probe_test.go:409:     t=  1.876s  Graceful stop vuNum=1
    blitzy_adhoc_test_probe_test.go:409:     t=  2.001s  Graceful stop vuNum=0
    blitzy_adhoc_test_probe_test.go:409:     t=  2.125s  Start         vuNum=0
    blitzy_adhoc_test_probe_test.go:409:     t=  2.251s  Start         vuNum=1
    blitzy_adhoc_test_probe_test.go:409:     t=  2.376s  Start         vuNum=2
    blitzy_adhoc_test_probe_test.go:409:     t=    2.5s  Start         vuNum=3
    blitzy_adhoc_test_probe_test.go:409:     t=  2.626s  Start         vuNum=4
    blitzy_adhoc_test_probe_test.go:409:     t=  2.751s  Start         vuNum=5
    blitzy_adhoc_test_probe_test.go:409:     t=  2.875s  Start         vuNum=6
    blitzy_adhoc_test_probe_test.go:409:     t=  3.001s  Start         vuNum=7
    blitzy_adhoc_test_probe_test.go:409:     t=  3.126s  Graceful stop vuNum=7
    blitzy_adhoc_test_probe_test.go:409:     t=  3.251s  Graceful stop vuNum=6
    blitzy_adhoc_test_probe_test.go:409:     t=  3.375s  Graceful stop vuNum=5
    blitzy_adhoc_test_probe_test.go:409:     t=    3.5s  Graceful stop vuNum=4
    blitzy_adhoc_test_probe_test.go:409:     t=  3.626s  Graceful stop vuNum=3
    blitzy_adhoc_test_probe_test.go:409:     t=  3.751s  Graceful stop vuNum=2
    blitzy_adhoc_test_probe_test.go:409:     t=  3.876s  Graceful stop vuNum=1
    blitzy_adhoc_test_probe_test.go:409:     t=  4.001s  Graceful stop vuNum=0
    blitzy_adhoc_test_probe_test.go:411: Symptom B — transition TOTALS: Start(scheduled grow)=16  Graceful stop(scheduled shrink)=16  Hard stop(max-allowed ceiling shrink)=0
--- PASS: TestBlitzyProbeHandlerDivergence (4.06s)
PASS
ok  	go.k6.io/k6/lib/executor	4.061s
```

**Reading the evidence.**

- The **raw-step** table is the scheduled target: it rises 0→8 over the first second, falls 8→0 over the
  next, and repeats, ending at 0 at 4 s (the executor's regular duration).
- The **graceful-step** table is the max-allowed ceiling: it rises 0→8 by 1 s and then **holds at 8**
  until 5 s (1 s of `GracefulStop` past the 4 s regular duration) before dropping to 0.
- The reconstructed **ceiling-vs-target** table shows `ceiling >= target` is **true at every sampled
  instant** — including every down-stage where the ceiling (8) sits above a falling target. That gap is
  precisely Symptom B, and it is intended.
- The **observed transitions** are `Start ×16` (the two up-stages, 8 VUs each) and `Graceful stop ×16`
  (the two down-stages). **`Hard stop` total is 0.**

**Correction of a previously-reported (fabricated) detail.** There are **zero** `Hard stop` log lines in
the real run. The ceiling holds at 8 until 5 s — at or beyond the 4 s run end — so by the time the
ceiling finally shrinks, the affected VUs have already been gracefully stopped; the resulting `hardStop`
calls are **no-ops that do not log** `lib/executor/vu_handle.go:169`. Any claim of "`Hard stop` at
t=2.401s" is not consistent with this run (the first graceful stop occurs at ~1.125 s, and the ceiling
does not shrink until 5 s), and is corrected here to the observed `Hard stop=0`.


### 7-C. Symptom C — on ctrl+c, some VUs "keep running for way longer than `gracefulStop` should allow"

**Direct answer:** On the canonical `ramping-vus` + real `SIGINT` path, a single Ctrl+C performs a
**graceful abort** (exit code 105 = `ExternalAbort` `errext/exitcodes/codes.go:41`) that cancels VU
iteration contexts and interrupts the JS VM, so even a CPU-busy iteration stops in **~17–19 ms** — VU
iterations do **not** overrun. The only thing observed "keeping running for seconds" is non-VU
**graceful-phase work** (here, a `teardown()` that runs ~3 s) completing during the graceful abort. So
the literal report — *VU iterations outrunning `gracefulStop`* — was **not reproduced** as a defect;
what is real is that graceful-phase work runs to completion during a single-Ctrl+C graceful abort. A
**second** Ctrl+C escalates to an immediate hard stop.

**Three terms that must be kept distinct (terminology correction).**

- **`gracefulRampDown`** — the per-stage window (default 30 s) allowing VUs to finish when the scheduled
  target *drops between stages* (`GracefulRampDown` default `lib/executor/ramping_vus.go:52`). This is
  the Symptom A/B mechanism.
- **`gracefulStop`** — the **normal executor-end** grace window (default 30 s,
  `DefaultGracefulStopValue` `lib/executor/base_config.go:20`, field `lib/executor/base_config.go:31`)
  applied when the executor reaches its *own* duration; it extends the max end time via
  `getDurationContexts` (`maxEndTime = start + regularDuration + gracefulStop`
  `lib/executor/helpers.go:168,172`). It is **not** a "stage-end" grace and it does **not** bound the
  ctrl+c abort.
- **The ctrl+c abort path** — a *separate* mechanism: `handleTestAbortSignals`
  `cmd/common.go:97` (`signal.Notify` for `os.Interrupt`/`SIGINT`/`SIGTERM` `cmd/common.go:101`) invokes
  the `gracefulStop` signal closure `cmd/run.go:349` on the first signal (Debug "Stopping k6 in response
  to signal…" `cmd/run.go:350`) and the `onHardStop` closure `cmd/run.go:359` on the second (Error
  "Aborting k6 in response to signal" `cmd/run.go:360`), then `gs.OSExit(int(exitcodes.ExternalAbort))`
  `cmd/common.go:118`. VU iteration `sleep` is interruptible because it selects on `ctx.Done()`
  `js/modules/k6/k6.go:71-80`.

**Method.** The real `/tmp/k6` binary is launched via canonical `k6 run`, a real `SIGINT` is delivered
to the exact spawned PID, and the SIGINT→exit latency is measured with a **monotonic** clock
(`python3 time.monotonic()`) using ~1 ms poll-based exit detection. Four conditions are each run twice
(Rule 1): **C1** sleeping iterations + single SIGINT; **C2** CPU-busy (non-yielding) iterations + single
SIGINT; **C3** a script with a ~3 s busy `teardown()` + single SIGINT; **C4** the same teardown script +
a double SIGINT. The full harness (strict shell options, `mktemp -d` workdir, exact-PID kill + `wait`,
`EXIT` trap cleanup) and the JS scripts are embedded in §9.2.

**Command (per run, as echoed in the transcript):**

```text
/tmp/k6 run --verbose --no-summary --no-usage-report <scenario>.js
```

```text
################ k6 build under test ################
k6 v0.55.0 (commit/128744ee44, go1.23.6, linux/amd64)
workdir: /tmp/k6probe_symC.wv5iBg ; RUNS per condition: 2

===== CONDITION C1_sleep_singleSIGINT — run 1/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report sleep_iter.js   (first SIGINT @+2s)
spawned k6 pid=148888
first  SIGINT: delivered_at_mono=3150345.821976  kill_status=0  process_alive_before_send=yes
wait result: pid=148888 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 16.9 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
time="2026-07-14T20:59:57Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-14T20:59:57Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
time="2026-07-14T20:59:57Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-14T20:59:57Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-14T20:59:57Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
--- setup/teardown markers (console.log -> stderr) ---
(no teardown markers)
--- iteration progress (final states) ---
running (00m01.0s), 4/5 VUs, 0 complete and 0 interrupted iterations
running (00m02.0s), 0/5 VUs, 0 complete and 5 interrupted iterations
===== end C1_sleep_singleSIGINT run 1: exit=105 =====

===== CONDITION C1_sleep_singleSIGINT — run 2/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report sleep_iter.js   (first SIGINT @+2s)
spawned k6 pid=148917
first  SIGINT: delivered_at_mono=3150347.880923  kill_status=0  process_alive_before_send=yes
wait result: pid=148917 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 18.3 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
time="2026-07-14T20:59:59Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-14T20:59:59Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
time="2026-07-14T20:59:59Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-14T20:59:59Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-14T20:59:59Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
--- setup/teardown markers (console.log -> stderr) ---
(no teardown markers)
--- iteration progress (final states) ---
running (00m01.0s), 4/5 VUs, 0 complete and 0 interrupted iterations
running (00m02.0s), 0/5 VUs, 0 complete and 5 interrupted iterations
===== end C1_sleep_singleSIGINT run 2: exit=105 =====

===== CONDITION C2_busy_singleSIGINT — run 1/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report busy_iter.js   (first SIGINT @+2.5s)
spawned k6 pid=148946
first  SIGINT: delivered_at_mono=3150350.437208  kill_status=0  process_alive_before_send=yes
wait result: pid=148946 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 17.6 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
time="2026-07-14T21:00:01Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-14T21:00:01Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
time="2026-07-14T21:00:01Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-14T21:00:01Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-14T21:00:01Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
--- setup/teardown markers (console.log -> stderr) ---
(no teardown markers)
--- iteration progress (final states) ---
running (00m02.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
running (00m02.5s), 0/3 VUs, 0 complete and 3 interrupted iterations
===== end C2_busy_singleSIGINT run 1: exit=105 =====

===== CONDITION C2_busy_singleSIGINT — run 2/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report busy_iter.js   (first SIGINT @+2.5s)
spawned k6 pid=148990
first  SIGINT: delivered_at_mono=3150352.990707  kill_status=0  process_alive_before_send=yes
wait result: pid=148990 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 19.1 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
time="2026-07-14T21:00:04Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-14T21:00:04Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
time="2026-07-14T21:00:04Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-14T21:00:04Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-14T21:00:04Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
--- setup/teardown markers (console.log -> stderr) ---
(no teardown markers)
--- iteration progress (final states) ---
running (00m02.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
running (00m02.5s), 0/3 VUs, 0 complete and 3 interrupted iterations
===== end C2_busy_singleSIGINT run 2: exit=105 =====

===== CONDITION C3_teardown_singleSIGINT — run 1/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report teardown_iter.js   (first SIGINT @+2.5s)
spawned k6 pid=149034
first  SIGINT: delivered_at_mono=3150355.549216  kill_status=0  process_alive_before_send=yes
wait result: pid=149034 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 3018.6 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
time="2026-07-14T21:00:06Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-14T21:00:09Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
time="2026-07-14T21:00:09Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-14T21:00:09Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-14T21:00:09Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
--- setup/teardown markers (console.log -> stderr) ---
time="2026-07-14T21:00:06Z" level=info msg=TEARDOWN_START source=console
time="2026-07-14T21:00:09Z" level=info msg=TEARDOWN_END source=console
--- iteration progress (final states) ---
running (00m05.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
running (00m05.5s), 0/2 VUs, 0 complete and 2 interrupted iterations
===== end C3_teardown_singleSIGINT run 1: exit=105 =====

===== CONDITION C3_teardown_singleSIGINT — run 2/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report teardown_iter.js   (first SIGINT @+2.5s)
spawned k6 pid=149981
first  SIGINT: delivered_at_mono=3150361.103532  kill_status=0  process_alive_before_send=yes
wait result: pid=149981 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 3018.3 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
time="2026-07-14T21:00:12Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-14T21:00:15Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
time="2026-07-14T21:00:15Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-14T21:00:15Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-14T21:00:15Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
--- setup/teardown markers (console.log -> stderr) ---
time="2026-07-14T21:00:12Z" level=info msg=TEARDOWN_START source=console
time="2026-07-14T21:00:15Z" level=info msg=TEARDOWN_END source=console
--- iteration progress (final states) ---
running (00m05.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
running (00m05.5s), 0/2 VUs, 0 complete and 2 interrupted iterations
===== end C3_teardown_singleSIGINT run 2: exit=105 =====

===== CONDITION C4_teardown_doubleSIGINT — run 1/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report teardown_iter.js   (first SIGINT @+2.5s, second @+0.8s after first)
spawned k6 pid=150898
first  SIGINT: delivered_at_mono=3150366.660236  kill_status=0  process_alive_before_send=yes
second SIGINT: delivered_at_mono=3150367.477251  kill_status=0  process_alive_before_send=yes
wait result: pid=150898 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 835.6 ms
elapsed second_SIGINT -> process_exit: 18.6 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
time="2026-07-14T21:00:17Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-14T21:00:18Z" level=error msg="Aborting k6 in response to signal" sig=interrupt
--- setup/teardown markers (console.log -> stderr) ---
time="2026-07-14T21:00:17Z" level=info msg=TEARDOWN_START source=console
--- iteration progress (final states) ---
running (00m02.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
running (00m03.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
===== end C4_teardown_doubleSIGINT run 1: exit=105 =====

===== CONDITION C4_teardown_doubleSIGINT — run 2/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report teardown_iter.js   (first SIGINT @+2.5s, second @+0.8s after first)
spawned k6 pid=151186
first  SIGINT: delivered_at_mono=3150370.049739  kill_status=0  process_alive_before_send=yes
second SIGINT: delivered_at_mono=3150370.867097  kill_status=0  process_alive_before_send=yes
wait result: pid=151186 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 834.5 ms
elapsed second_SIGINT -> process_exit: 17.2 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
time="2026-07-14T21:00:21Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-14T21:00:22Z" level=error msg="Aborting k6 in response to signal" sig=interrupt
--- setup/teardown markers (console.log -> stderr) ---
time="2026-07-14T21:00:21Z" level=info msg=TEARDOWN_START source=console
--- iteration progress (final states) ---
running (00m03.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
running (00m03.3s), 0/2 VUs, 0 complete and 2 interrupted iterations
===== end C4_teardown_doubleSIGINT run 2: exit=105 =====

################ ALL SYMPTOM C CONDITIONS COMPLETE ################
```

**Reading the four conditions (both runs each are consistent).**

- **C1 (sleeping iterations, single SIGINT):** exit **105** in **16.9 ms / 18.3 ms**; the progress line
  goes from `4/5 VUs, 0 … interrupted` to `0/5 VUs, … 5 interrupted iterations`. Only the graceful
  "Stopping…" (Debug) appears. Interruptible sleeps stop promptly.
- **C2 (CPU-busy, non-yielding iterations, single SIGINT):** exit **105** in **17.6 ms / 19.1 ms**;
  `3 interrupted iterations`. **Even a busy JS loop is interrupted** — k6 cancels the iteration context
  and the VM stops — so ordinary VU iterations do not overrun `gracefulStop`.
- **C3 (script with a ~3 s busy `teardown()`, single SIGINT):** exit **105** in **3018.6 ms / 3018.3 ms**.
  The VU iterations are interrupted immediately, but `TEARDOWN_START → TEARDOWN_END` then runs to
  **completion** during the graceful abort. This is the demonstrated mechanism behind "keeps running
  longer than expected": it is **graceful-phase work**, not a per-VU iteration overrunning `gracefulStop`.
- **C4 (same teardown script, double SIGINT):** the first SIGINT begins the graceful "Stopping…" and
  `TEARDOWN_START`; the second SIGINT ~0.8 s later triggers "Aborting k6 in response to signal" (Error,
  `onHardStop`) and an immediate `OSExit(105)`. The second-SIGINT→exit latency is **18.6 ms / 17.2 ms**
  and `TEARDOWN_END` is **absent** — the hard stop cut teardown short. This is the by-design escalation
  escape hatch (the `cmd/common.go:116-117` comment references k6 issue #971).

**Demonstrated mechanism vs. the user's diagnosis.** What is *demonstrated* is: (a) VU iterations,
including CPU-busy ones, are interrupted in ~17–19 ms on a single Ctrl+C; (b) a single Ctrl+C then runs
graceful-phase work such as `teardown()` to completion; (c) a second Ctrl+C forces an immediate hard
stop. The user's literal wording — *VUs keep running longer than `gracefulStop`* — was **not reproduced**
as a VU/`gracefulStop` violation on the canonical path. **(INFERRED)** the user most likely observed
graceful-phase work (like `teardown()`), or a long non-yielding native/host call inside an iteration
that does not observe context cancellation, rather than the `ramping-vus` scheduler letting VU iterations
exceed `gracefulStop`.


### 7-D. Symptom D — one instance consistently shows more VUs; summed across instances they exceed the configured maximum

**Direct answer:** The imbalance is **deterministic striping**, not a race. With a **shared**
`--execution-segment-sequence`, three synchronized instances split a global max of 11 as `[4, 4, 3]` —
one instance (segment `0:1/3`) consistently higher — and the per-instance counts **sum to exactly 11**,
never exceeding it. The sum **exceeds** the configured maximum only when the instances are run
**without** a shared sequence: each instance then fills its *own* segment sequence and rounds up
independently, giving `[4, 4, 4]`, sum **12 > 11**. Both outcomes are fully determined by the striping
arithmetic (the same test yields the same split every time) — there is no randomness and no race.

**Why one instance is consistently higher (source trace).** The `--execution-segment` /
`--execution-segment-sequence` flags `cmd/options.go:31-32` feed `NewExecutionTuple`
`lib/execution_segment.go:723`, which calls `GetFilledExecutionSegmentSequence`
`lib/execution_segment.go:445`. Scaling a global count for a segment uses the striped
`ExecutionSegmentSequenceWrapper.ScaleInt64` `lib/execution_segment.go:580`: it computes
`result = (value / lcd) * len(offsets)` `lib/execution_segment.go:583` and then distributes the
remainder to the earliest offsets in a loop `lib/execution_segment.go:584-586`. The remainder therefore
lands on the earliest segment(s) first — deterministically — which is why segment `0:1/3` is the one
that consistently rounds up. (`ExecutionTuple.ScaleInt64` `lib/execution_segment.go:734` short-circuits to
the raw value when the sequence has a single segment `lib/execution_segment.go:735-737`, and otherwise
delegates to the striped wrapper.)

**Why a shared sequence sums to the max but independent fills can exceed it.** With a **shared** sequence
every instance builds the *same* filled sequence, so the striped shares partition the global value
exactly (they sum to it). **Without** a shared sequence, `GetFilledExecutionSegmentSequence(nil, segment)`
fills a *different* sequence around each instance's own segment, so each instance rounds **its** share up
independently and the shares can sum above the global value.

#### 7-D.1 The striping arithmetic (canonical `lib` API, deterministic; reproduced ×2 under `-race`)

**Command:**

```text
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -run 'TestBlitzyProbeSegmentScaleArithmetic|TestBlitzyProbeIndependentUnderEqualOver' ./lib/executor/ -v
```

The first test scales global totals `T = 1..12` for the shared sequence `0,1/3,2/3,1` (segments `0:1/3`,
`1/3:2/3`, `2/3:1`) and shows the per-segment split always sums **exactly** to `T` (0 OVER, 0 UNDER
rows), with the surplus at low `T` landing on segment index 0, recomputed 5× identically. The second test
builds each segment's sequence **independently** (no shared sequence) and prints the **complete**
under/equal/over table — not filtered to overshoots — showing 4 UNDER, 4 EQUAL, and 4 OVER rows across
`T=1..12`:

```text
=== RUN   TestBlitzyProbeSegmentScaleArithmetic
=== PAUSE TestBlitzyProbeSegmentScaleArithmetic
=== RUN   TestBlitzyProbeIndependentUnderEqualOver
=== PAUSE TestBlitzyProbeIndependentUnderEqualOver
=== CONT  TestBlitzyProbeSegmentScaleArithmetic
=== CONT  TestBlitzyProbeIndependentUnderEqualOver
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:624: Symptom D (shared sequence "0,1/3,2/3,1") — deterministic striped ScaleInt64 [execution_segment.go:580]:
    blitzy_adhoc_test_probe_test.go:625:       T | 0:1/3     | 1/3:2/3     | 2/3:1     | sum | sum vs T
    blitzy_adhoc_test_probe_test.go:643:       1 |         1 |           0 |         0 |   1 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:       2 |         1 |           1 |         0 |   2 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:       3 |         1 |           1 |         1 |   3 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:       4 |         2 |           1 |         1 |   4 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:       5 |         2 |           2 |         1 |   5 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:       6 |         2 |           2 |         2 |   6 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:       7 |         3 |           2 |         2 |   7 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:       8 |         3 |           3 |         2 |   8 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:       9 |         3 |           3 |         3 |   9 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:      10 |         4 |           3 |         3 |  10 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:      11 |         4 |           4 |         3 |  11 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:      12 |         4 |           4 |         4 |  12 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:645: Symptom D (shared sequence) — OVER rows: 0, UNDER rows: 0 (both must be 0; sum==T always)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:699: Symptom D (independent) — segment 0:1/3    filled sequence: 0,1/3,1
    blitzy_adhoc_test_probe_test.go:699: Symptom D (independent) — segment 1/3:2/3  filled sequence: 0,1/3,2/3,1
    blitzy_adhoc_test_probe_test.go:699: Symptom D (independent) — segment 2/3:1    filled sequence: 0,2/3,1
    blitzy_adhoc_test_probe_test.go:702: Symptom D (NO shared sequence, each instance independent) — full under/equal/over table:
    blitzy_adhoc_test_probe_test.go:703:       T | 0:1/3     | 1/3:2/3     | 2/3:1     | sum | sum vs global max T
    blitzy_adhoc_test_probe_test.go:723:       1 |         0 |           0 |         0 |   0 | UNDER (sum-T=-1)
    blitzy_adhoc_test_probe_test.go:723:       2 |         1 |           1 |         1 |   3 | OVER (sum-T=+1)
    blitzy_adhoc_test_probe_test.go:723:       3 |         1 |           1 |         1 |   3 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:723:       4 |         1 |           1 |         1 |   3 | UNDER (sum-T=-1)
    blitzy_adhoc_test_probe_test.go:723:       5 |         2 |           2 |         2 |   6 | OVER (sum-T=+1)
    blitzy_adhoc_test_probe_test.go:723:       6 |         2 |           2 |         2 |   6 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:723:       7 |         2 |           2 |         2 |   6 | UNDER (sum-T=-1)
    blitzy_adhoc_test_probe_test.go:723:       8 |         3 |           3 |         3 |   9 | OVER (sum-T=+1)
    blitzy_adhoc_test_probe_test.go:723:       9 |         3 |           3 |         3 |   9 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:723:      10 |         3 |           3 |         3 |   9 | UNDER (sum-T=-1)
    blitzy_adhoc_test_probe_test.go:723:      11 |         4 |           4 |         4 |  12 | OVER (sum-T=+1)
    blitzy_adhoc_test_probe_test.go:723:      12 |         4 |           4 |         4 |  12 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:725: Symptom D (independent) — classification across T=1..12: OVER=4 EQUAL=4 UNDER=4
    blitzy_adhoc_test_probe_test.go:726: Symptom D (independent) — OVER rows are real: without a shared --execution-segment-sequence the per-instance shares can sum ABOVE the configured maximum (deterministic per T, still never random).
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:667: Symptom D (shared sequence) — recomputed 5×: identical every time (deterministic, coordination-free).
--- PASS: TestBlitzyProbeIndependentUnderEqualOver (0.00s)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:677: Symptom D (shared sequence) — at T=1 the surplus VU goes to segment index 0 (0:1/3): [1 0 0] => one instance is CONSISTENTLY higher (deterministic), never random.
--- PASS: TestBlitzyProbeSegmentScaleArithmetic (0.00s)
PASS
ok  	go.k6.io/k6/lib/executor	0.006s
```

Re-running both arithmetic tests under the race detector with `-count=2` reproduces the identical
classification (shared: 0 OVER / 0 UNDER; independent: OVER=4 EQUAL=4 UNDER=4) with **zero** data races:

```text
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -race -run 'TestBlitzyProbeSegmentScaleArithmetic|TestBlitzyProbeIndependentUnderEqualOver' ./lib/executor/ -count=2 -v
```

```text
=== RUN   TestBlitzyProbeSegmentScaleArithmetic
=== PAUSE TestBlitzyProbeSegmentScaleArithmetic
=== RUN   TestBlitzyProbeIndependentUnderEqualOver
=== PAUSE TestBlitzyProbeIndependentUnderEqualOver
=== CONT  TestBlitzyProbeIndependentUnderEqualOver
=== CONT  TestBlitzyProbeSegmentScaleArithmetic
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:699: Symptom D (independent) — segment 0:1/3    filled sequence: 0,1/3,1
    blitzy_adhoc_test_probe_test.go:699: Symptom D (independent) — segment 1/3:2/3  filled sequence: 0,1/3,2/3,1
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:624: Symptom D (shared sequence "0,1/3,2/3,1") — deterministic striped ScaleInt64 [execution_segment.go:580]:
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:699: Symptom D (independent) — segment 2/3:1    filled sequence: 0,2/3,1
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:625:       T | 0:1/3     | 1/3:2/3     | 2/3:1     | sum | sum vs T
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:702: Symptom D (NO shared sequence, each instance independent) — full under/equal/over table:
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:643:       1 |         1 |           0 |         0 |   1 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:       2 |         1 |           1 |         0 |   2 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:       3 |         1 |           1 |         1 |   3 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:       4 |         2 |           1 |         1 |   4 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:703:       T | 0:1/3     | 1/3:2/3     | 2/3:1     | sum | sum vs global max T
    blitzy_adhoc_test_probe_test.go:723:       1 |         0 |           0 |         0 |   0 | UNDER (sum-T=-1)
    blitzy_adhoc_test_probe_test.go:723:       2 |         1 |           1 |         1 |   3 | OVER (sum-T=+1)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:643:       5 |         2 |           2 |         1 |   5 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:723:       3 |         1 |           1 |         1 |   3 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:723:       4 |         1 |           1 |         1 |   3 | UNDER (sum-T=-1)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:643:       6 |         2 |           2 |         2 |   6 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:       7 |         3 |           2 |         2 |   7 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:       8 |         3 |           3 |         2 |   8 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:723:       5 |         2 |           2 |         2 |   6 | OVER (sum-T=+1)
    blitzy_adhoc_test_probe_test.go:723:       6 |         2 |           2 |         2 |   6 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:723:       7 |         2 |           2 |         2 |   6 | UNDER (sum-T=-1)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:643:       9 |         3 |           3 |         3 |   9 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:      10 |         4 |           3 |         3 |  10 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:      11 |         4 |           4 |         3 |  11 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:723:       8 |         3 |           3 |         3 |   9 | OVER (sum-T=+1)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:643:      12 |         4 |           4 |         4 |  12 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:723:       9 |         3 |           3 |         3 |   9 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:723:      10 |         3 |           3 |         3 |   9 | UNDER (sum-T=-1)
    blitzy_adhoc_test_probe_test.go:723:      11 |         4 |           4 |         4 |  12 | OVER (sum-T=+1)
    blitzy_adhoc_test_probe_test.go:723:      12 |         4 |           4 |         4 |  12 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:645: Symptom D (shared sequence) — OVER rows: 0, UNDER rows: 0 (both must be 0; sum==T always)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:725: Symptom D (independent) — classification across T=1..12: OVER=4 EQUAL=4 UNDER=4
    blitzy_adhoc_test_probe_test.go:726: Symptom D (independent) — OVER rows are real: without a shared --execution-segment-sequence the per-instance shares can sum ABOVE the configured maximum (deterministic per T, still never random).
--- PASS: TestBlitzyProbeIndependentUnderEqualOver (0.00s)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:667: Symptom D (shared sequence) — recomputed 5×: identical every time (deterministic, coordination-free).
    blitzy_adhoc_test_probe_test.go:677: Symptom D (shared sequence) — at T=1 the surplus VU goes to segment index 0 (0:1/3): [1 0 0] => one instance is CONSISTENTLY higher (deterministic), never random.
--- PASS: TestBlitzyProbeSegmentScaleArithmetic (0.00s)
=== RUN   TestBlitzyProbeSegmentScaleArithmetic
=== PAUSE TestBlitzyProbeSegmentScaleArithmetic
=== RUN   TestBlitzyProbeIndependentUnderEqualOver
=== PAUSE TestBlitzyProbeIndependentUnderEqualOver
=== CONT  TestBlitzyProbeSegmentScaleArithmetic
=== CONT  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:699: Symptom D (independent) — segment 0:1/3    filled sequence: 0,1/3,1
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:624: Symptom D (shared sequence "0,1/3,2/3,1") — deterministic striped ScaleInt64 [execution_segment.go:580]:
    blitzy_adhoc_test_probe_test.go:625:       T | 0:1/3     | 1/3:2/3     | 2/3:1     | sum | sum vs T
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:699: Symptom D (independent) — segment 1/3:2/3  filled sequence: 0,1/3,2/3,1
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:643:       1 |         1 |           0 |         0 |   1 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:       2 |         1 |           1 |         0 |   2 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:699: Symptom D (independent) — segment 2/3:1    filled sequence: 0,2/3,1
    blitzy_adhoc_test_probe_test.go:702: Symptom D (NO shared sequence, each instance independent) — full under/equal/over table:
    blitzy_adhoc_test_probe_test.go:703:       T | 0:1/3     | 1/3:2/3     | 2/3:1     | sum | sum vs global max T
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:643:       3 |         1 |           1 |         1 |   3 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:       4 |         2 |           1 |         1 |   4 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:       5 |         2 |           2 |         1 |   5 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:       6 |         2 |           2 |         2 |   6 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:723:       1 |         0 |           0 |         0 |   0 | UNDER (sum-T=-1)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:643:       7 |         3 |           2 |         2 |   7 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:643:       8 |         3 |           3 |         2 |   8 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:723:       2 |         1 |           1 |         1 |   3 | OVER (sum-T=+1)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:643:       9 |         3 |           3 |         3 |   9 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:723:       3 |         1 |           1 |         1 |   3 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:643:      10 |         4 |           3 |         3 |  10 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:723:       4 |         1 |           1 |         1 |   3 | UNDER (sum-T=-1)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:643:      11 |         4 |           4 |         3 |  11 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:723:       5 |         2 |           2 |         2 |   6 | OVER (sum-T=+1)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:643:      12 |         4 |           4 |         4 |  12 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:645: Symptom D (shared sequence) — OVER rows: 0, UNDER rows: 0 (both must be 0; sum==T always)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:723:       6 |         2 |           2 |         2 |   6 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:723:       7 |         2 |           2 |         2 |   6 | UNDER (sum-T=-1)
    blitzy_adhoc_test_probe_test.go:723:       8 |         3 |           3 |         3 |   9 | OVER (sum-T=+1)
    blitzy_adhoc_test_probe_test.go:723:       9 |         3 |           3 |         3 |   9 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:723:      10 |         3 |           3 |         3 |   9 | UNDER (sum-T=-1)
    blitzy_adhoc_test_probe_test.go:723:      11 |         4 |           4 |         4 |  12 | OVER (sum-T=+1)
    blitzy_adhoc_test_probe_test.go:723:      12 |         4 |           4 |         4 |  12 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:725: Symptom D (independent) — classification across T=1..12: OVER=4 EQUAL=4 UNDER=4
    blitzy_adhoc_test_probe_test.go:726: Symptom D (independent) — OVER rows are real: without a shared --execution-segment-sequence the per-instance shares can sum ABOVE the configured maximum (deterministic per T, still never random).
--- PASS: TestBlitzyProbeIndependentUnderEqualOver (0.00s)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:667: Symptom D (shared sequence) — recomputed 5×: identical every time (deterministic, coordination-free).
    blitzy_adhoc_test_probe_test.go:677: Symptom D (shared sequence) — at T=1 the surplus VU goes to segment index 0 (0:1/3): [1 0 0] => one instance is CONSISTENTLY higher (deterministic), never random.
--- PASS: TestBlitzyProbeSegmentScaleArithmetic (0.00s)
PASS
ok  	go.k6.io/k6/lib/executor	1.034s
```

#### 7-D.2 Genuine synchronized three-process CLI runs (canonical `k6` binary; ≥2 runs per mode)

To reproduce the user's distributed setup faithfully, three real `k6 run` processes are launched
**simultaneously**, one per segment, sharing a global ramp with a configured maximum of 11 VUs. Each
mode is run twice. Per-instance active-VU counts are parsed from the `running (SS.s), active/max VUs`
progress lines that k6 writes to **stdout** (k6's `level=…` log lines go to stderr). The full harness and
the JS scenario are embedded in §9.3.

- **SHARED** mode passes the same `--execution-segment-sequence 0,1/3,2/3,1` to all three instances.
- **INDEPENDENT** mode passes only `--execution-segment` to each (no shared sequence).

**Commands (as echoed in the transcript):**

```text
# SHARED (per instance)
/tmp/k6 run --execution-segment 0:1/3   --execution-segment-sequence 0,1/3,2/3,1 seg.js
/tmp/k6 run --execution-segment 1/3:2/3 --execution-segment-sequence 0,1/3,2/3,1 seg.js
/tmp/k6 run --execution-segment 2/3:1   --execution-segment-sequence 0,1/3,2/3,1 seg.js
# INDEPENDENT (per instance)
/tmp/k6 run --execution-segment 0:1/3   seg.js
/tmp/k6 run --execution-segment 1/3:2/3 seg.js
/tmp/k6 run --execution-segment 2/3:1   seg.js
```

```text
################ k6 build under test ################
k6 v0.55.0 (commit/128744ee44, go1.23.6, linux/amd64)
workdir: /tmp/k6probe_symD.w5f6V3 ; RUNS per mode: 2 ; shared sequence: 0,1/3,2/3,1 ; segments: 0:1/3 1/3:2/3 2/3:1 ; global max target: 11

-------- SHARED sequence, SYNCHRONIZED 3-process triple, run 1/2 --------
$ /tmp/k6 run --execution-segment 0:1/3 --execution-segment-sequence 0,1/3,2/3,1 seg.js
$ /tmp/k6 run --execution-segment 1/3:2/3 --execution-segment-sequence 0,1/3,2/3,1 seg.js
$ /tmp/k6 run --execution-segment 2/3:1 --execution-segment-sequence 0,1/3,2/3,1 seg.js
exit codes: seg0=0 seg1=0 seg2=0
  per-instance max planned VUs (scenario header 'Up to N looping VUs'):
    seg0 0:1/3    => 4
    seg1 1/3:2/3  => 4
    seg2 2/3:1    => 3
    SUM of per-instance max VUs = 11  vs configured global max = 11  => EQUAL
  aligned active-VU series (carry-forward by k6 internal elapsed):
    elapsed | seg0 | seg1 | seg2 | sum | vs gmax
       1.0s |    1 |    1 |    1 |   3 |
       2.0s |    3 |    2 |    2 |   7 |
       3.0s |    4 |    3 |    3 |  10 |
       4.0s |    4 |    4 |    3 |  11 | =
       5.0s |    4 |    4 |    3 |  11 | =
       6.0s |    3 |    4 |    3 |  10 |
       7.0s |    2 |    2 |    1 |   5 |
       7.8s |    2 |    2 |    0 |   4 |
       8.0s |    2 |    1 |    0 |   3 |
       8.3s |    0 |    1 |    0 |   1 |
       8.6s |    0 |    0 |    0 |   0 |
  peak instantaneous active-VU sum across aligned samples = 11 (configured max 11)
  per-instance peak active VUs: [4, 4, 3] ; highest = seg0 (0:1/3) => one instance consistently higher
-------- end shared run 1 --------

-------- SHARED sequence, SYNCHRONIZED 3-process triple, run 2/2 --------
$ /tmp/k6 run --execution-segment 0:1/3 --execution-segment-sequence 0,1/3,2/3,1 seg.js
$ /tmp/k6 run --execution-segment 1/3:2/3 --execution-segment-sequence 0,1/3,2/3,1 seg.js
$ /tmp/k6 run --execution-segment 2/3:1 --execution-segment-sequence 0,1/3,2/3,1 seg.js
exit codes: seg0=0 seg1=0 seg2=0
  per-instance max planned VUs (scenario header 'Up to N looping VUs'):
    seg0 0:1/3    => 4
    seg1 1/3:2/3  => 4
    seg2 2/3:1    => 3
    SUM of per-instance max VUs = 11  vs configured global max = 11  => EQUAL
  aligned active-VU series (carry-forward by k6 internal elapsed):
    elapsed | seg0 | seg1 | seg2 | sum | vs gmax
       1.0s |    1 |    1 |    1 |   3 |
       2.0s |    3 |    2 |    2 |   7 |
       3.0s |    4 |    3 |    3 |  10 |
       4.0s |    4 |    4 |    3 |  11 | =
       5.0s |    4 |    4 |    3 |  11 | =
       6.0s |    3 |    4 |    3 |  10 |
       7.0s |    2 |    2 |    1 |   5 |
       7.8s |    2 |    2 |    0 |   4 |
       8.0s |    2 |    1 |    0 |   3 |
       8.3s |    0 |    1 |    0 |   1 |
       8.6s |    0 |    0 |    0 |   0 |
  peak instantaneous active-VU sum across aligned samples = 11 (configured max 11)
  per-instance peak active VUs: [4, 4, 3] ; highest = seg0 (0:1/3) => one instance consistently higher
-------- end shared run 2 --------

-------- INDEPENDENT sequence, SYNCHRONIZED 3-process triple, run 1/2 --------
$ /tmp/k6 run --execution-segment 0:1/3 seg.js   (NO --execution-segment-sequence)
$ /tmp/k6 run --execution-segment 1/3:2/3 seg.js   (NO --execution-segment-sequence)
$ /tmp/k6 run --execution-segment 2/3:1 seg.js   (NO --execution-segment-sequence)
exit codes: seg0=0 seg1=0 seg2=0
  per-instance max planned VUs (scenario header 'Up to N looping VUs'):
    seg0 0:1/3    => 4
    seg1 1/3:2/3  => 4
    seg2 2/3:1    => 4
    SUM of per-instance max VUs = 12  vs configured global max = 11  => OVER (exceeds configured max!)
  aligned active-VU series (carry-forward by k6 internal elapsed):
    elapsed | seg0 | seg1 | seg2 | sum | vs gmax
       1.0s |    1 |    1 |    1 |   3 |
       2.0s |    2 |    2 |    2 |   6 |
       3.0s |    3 |    3 |    3 |   9 |
       4.0s |    4 |    4 |    4 |  12 | OVER
       5.0s |    4 |    4 |    4 |  12 | OVER
       6.0s |    4 |    4 |    4 |  12 | OVER
       7.0s |    2 |    2 |    2 |   6 |
       8.0s |    1 |    1 |    1 |   3 |
       8.5s |    1 |    0 |    1 |   2 |
       8.6s |    0 |    0 |    0 |   0 |
  peak instantaneous active-VU sum across aligned samples = 12 (configured max 11)
  per-instance peak active VUs: [4, 4, 4] ; highest = seg0 (0:1/3) => one instance consistently higher
-------- end independent run 1 --------

-------- INDEPENDENT sequence, SYNCHRONIZED 3-process triple, run 2/2 --------
$ /tmp/k6 run --execution-segment 0:1/3 seg.js   (NO --execution-segment-sequence)
$ /tmp/k6 run --execution-segment 1/3:2/3 seg.js   (NO --execution-segment-sequence)
$ /tmp/k6 run --execution-segment 2/3:1 seg.js   (NO --execution-segment-sequence)
exit codes: seg0=0 seg1=0 seg2=0
  per-instance max planned VUs (scenario header 'Up to N looping VUs'):
    seg0 0:1/3    => 4
    seg1 1/3:2/3  => 4
    seg2 2/3:1    => 4
    SUM of per-instance max VUs = 12  vs configured global max = 11  => OVER (exceeds configured max!)
  aligned active-VU series (carry-forward by k6 internal elapsed):
    elapsed | seg0 | seg1 | seg2 | sum | vs gmax
       1.0s |    1 |    1 |    1 |   3 |
       2.0s |    2 |    2 |    2 |   6 |
       3.0s |    3 |    3 |    4 |  10 |
       4.0s |    4 |    4 |    4 |  12 | OVER
       5.0s |    4 |    4 |    4 |  12 | OVER
       6.0s |    4 |    4 |    4 |  12 | OVER
       7.0s |    2 |    2 |    2 |   6 |
       8.0s |    1 |    1 |    1 |   3 |
       8.5s |    1 |    1 |    0 |   2 |
       8.6s |    0 |    0 |    0 |   0 |
  peak instantaneous active-VU sum across aligned samples = 12 (configured max 11)
  per-instance peak active VUs: [4, 4, 4] ; highest = seg0 (0:1/3) => one instance consistently higher
-------- end independent run 2 --------

################ ALL SYMPTOM D TRIPLES COMPLETE ################
```

**Reading the CLI evidence.**

- **SHARED, both runs:** per-instance max planned VUs `[4, 4, 3]`, sum **= 11 = configured max**
  (`EQUAL`); segment `0:1/3` is consistently at/above the others; the aligned instantaneous active-VU
  sum peaks at 11 and **never** exceeds it. This is the user's "one instance consistently shows more" —
  and it is correct: it sums to exactly the configured max.
- **INDEPENDENT, both runs:** per-instance max planned VUs `[4, 4, 4]`, sum **= 12 > 11**
  (`OVER — exceeds configured max`); the aligned instantaneous sum reaches 12. This reproduces the
  user's "sum them up and they exceed my configured maximum" — and it happens specifically because the
  instances were **not** given a shared `--execution-segment-sequence`.

#### 7-D.3 External design context (scoped precisely)

Two k6 design discussions are directly relevant, and their scope must be delimited so neither is
over-applied:

- **Issue #997, "Execution segments for partitioning work between k6 instances"**
  (https://github.com/grafana/k6/issues/997). This is the **original design** of segment distribution.
  Its stated goals — distribute work with minimal coordination and *no* central scheduler, with **no**
  "off-by-one errors, rounding mistakes, [or] VU numbers jumping up and down," so that "launching the
  same test multiple times [produces] identical scheduling outcomes … [with no] randomness" — are
  exactly what the arithmetic in §7-D.1 demonstrates: the striping is deterministic and coordination-free,
  and with a shared sequence it sums to the configured maximum. **This issue supports the
  deterministic-striping conclusion (including why one instance is consistently, not randomly, higher).**
- **Issue #1308, "Better distribution of work with execution segments?"**
  (https://github.com/grafana/k6/issues/1308). This concerns executors that must partition **two**
  attributes — VU count *and* per-VU work — and it **explicitly excludes** `ramping-vus`: it states that
  "per-vu-iterations, constant-looping-vus, and variable-looping-vus" (variable-looping-vus being the
  historical name for `ramping-vus`) "don't have this problem, since … the only important thing we need
  to segment/partition is the number of VUs"; only `shared-iterations` and the arrival-rate executors
  are affected. **Therefore #1308 does not explain the `ramping-vus` "sum exceeds max" behavior.** That
  behavior is explained by the checked-in striping arithmetic measured in §7-D.1 (independent per-segment
  rounding), **not** by #1308.

**Conclusion for Symptom D:** the imbalance and the transient sum-above-max are deterministic
consequences of the striping algorithm (`ScaleInt64`), observed both arithmetically (§7-D.1) and through
genuine synchronized multi-process CLI runs (§7-D.2). "One instance consistently higher" is expected;
"sum exceeds the configured maximum" occurs specifically when instances lack a shared
`--execution-segment-sequence`. Neither is a race.


---

## 8. Coverage summary

Every named symptom, question, and sub-item from the request is addressed, with the section holding its
runtime evidence:

| Request item | Verdict | Evidence |
|---|---|---|
| **Question (a)** — race between the two handler goroutines? | **No** — the two strategies are invoked sequentially in one goroutine; `-race` clean ×2 on both suites | §2.1, §4 |
| **Question (b)** — is the VU buffer leaking? | **No** on the observed paths — buffer fully restored; failure path fails cleanly | §2.2, §6 |
| **Question (c)** — trace both handlers modifying VU state "simultaneously" | Serialized on one per-VU mutex; mixed atomic/mutex model; no unsafe interleaving | §2.3, §5 |
| **Symptom A** — "stuck" VUs (rapid up/down + long `gracefulRampDown`) | Transient, bounded lag (max +2), never permanent (final `0×10`) | §7-A |
| **Symptom B** — scheduled vs. graceful count mismatch | Expected by design; `ceiling >= target` always; `Hard stop=0` observed | §7-B |
| **Symptom C** — VUs outrun `gracefulStop` on ctrl+c | Not reproduced as a VU/`gracefulStop` defect; single Ctrl+C interrupts iterations in ~17–19 ms; graceful-phase work (`teardown`) explains long runtime; 2nd Ctrl+C = immediate hard stop | §7-C |
| **Symptom D** — one instance higher; sum exceeds max | Deterministic striping; shared-sequence sum `=11`; independent-fill sum `=12>11` | §7-D |
| Named component: `maxAllowedVUsHandlerStrategy` `:668` (graceful/max-allowed) | Traced; shrinks ceiling via `hardStop`, grows implicitly | §4.1, §7-B |
| Named component: `scheduledVUsHandlerStrategy` `:679` (scheduled) | Traced; `start`/`gracefulStop` on target change | §4.1, §7-B |
| Named component: per-VU `mutex` `:71`; `start`/`gracefulStop`/`hardStop` | All lock the same mutex; serialize | §5.1 |
| Named states: `stopped`/`starting`/`running`/`toGracefulStop`/`toHardStop` `:17-21` | Used by real name; transition trace captured | §5, §7-A |
| Named path: VU buffer `GetPlannedVU`/`ReturnVU`/`ModCurrentlyActiveVUsCount` | Traced + accounting + failure path | §6 |
| Named path: `gracefulStop`/`onHardStop` closures; `handleTestAbortSignals`; exit 105 | Traced; both signal closures observed | §7-C |
| Named path: `ScaleInt64` / `GetFilledExecutionSegmentSequence` striping | Traced arithmetically + via CLI | §7-D |
| "e.g." stages that go up and down rapidly with a long `gracefulRampDown` | Reproduced as 8↔0 ×4, `GracefulRampDown=3s` | §7-A/§7-B |
| "e.g." kill the test early with ctrl+c | Real `SIGINT` to the built binary | §7-C |
| "e.g." three instances, one higher, sum exceeds max | Three synchronized `k6 run` processes | §7-D |
| Reproduce run-to-run inconsistency ("sometimes") by repeating identical input | Symptom A run ×10 (+×20 under `-race`); each C/D mode ×2 | §7-A, §7-C, §7-D |


---

## 9. Reproduction artifacts and cleanup attestation

So that every block above can be regenerated independently, the complete source of the temporary probe
and harnesses is embedded here verbatim. These are **temporary investigation artifacts**: none of them
is committed to the repository (see §9.4).

### 9.1 In-package Go probe (`lib/executor/blitzy_adhoc_test_probe_test.go`)

This is the consolidated final source of the single temporary Go test file used for the in-process
reproductions (§5, §6, §7-A, §7-B, and the arithmetic in §7-D.1). Because the probe was lightly edited
between capture runs, the auto-generated `t.Logf` source-line-number prefixes in the transcripts above
reflect the exact draft at each capture and may differ by a few lines from this consolidated listing; the
message text and all reported values are identical either way. It is placed in `package executor` because it reuses the real, unexported test
harness (`simpleRunner`, `getTestRunState`, `setupExecutor`, `newStoppedVUHandle`, `NewExecutionState`,
`NewExecutionTuple`, `GetFilledExecutionSegmentSequence`, `ScaleInt64`) exactly as the in-repo tests do —
so all observations flow through the canonical package API, not a stand-in. To capture the DebugLevel
transition lines, it builds the executor with a `logrus` entry at `DebugLevel` wired to a
`testutils.NewLogHook`. The file name carries the `blitzy_adhoc_test_` prefix and is deleted after
capture.

```go
package executor

// blitzy_adhoc_test_probe_test.go is a TEMPORARY, EPHEMERAL investigation probe.
// It exists only to capture real runtime evidence for the k6 ramping-vus concurrency
// investigation and MUST be deleted after evidence capture (it is never committed).
// It lives inside package `executor` because it needs the unexported test harness
// (getTestRunState/simpleRunner) and the unexported vuHandle state machine.

import (
	"context"
	"fmt"
	"io"
	"sort"
	"strings"
	"sync/atomic"
	"testing"
	"time"

	"github.com/sirupsen/logrus"
	"github.com/stretchr/testify/require"
	"gopkg.in/guregu/null.v3"

	"go.k6.io/k6/lib"
	"go.k6.io/k6/lib/testutils"
	"go.k6.io/k6/lib/types"
	"go.k6.io/k6/metrics"
)

// stateName maps the unexported vuHandle stateType constants to their source names
// (lib/executor/vu_handle.go:17-21) for human-readable evidence.
func stateName(s stateType) string {
	switch s {
	case stopped:
		return "stopped"
	case starting:
		return "starting"
	case running:
		return "running"
	case toGracefulStop:
		return "toGracefulStop"
	case toHardStop:
		return "toHardStop"
	default:
		return fmt.Sprintf("unknown(%d)", int(s))
	}
}

// readState reads vh.state exactly as the lock-free fast path does
// (atomic.LoadInt32, lib/executor/vu_handle.go:204).
func readState(vh *vuHandle) stateType {
	return stateType(atomic.LoadInt32((*int32)(&vh.state)))
}

// blitzyProbeExecutor builds a ramping-vus executor wired to a DebugLevel log hook so
// that the vuHandle "Start"/"Graceful stop"/"Hard stop" debug lines
// (vu_handle.go:123,127,161,177) are captured. It mirrors setupExecutor
// (common_test.go:44) but forces DebugLevel and returns the ExecutionState + hook.
func blitzyProbeExecutor(
	t testing.TB, config lib.ExecutorConfig, segmentStr, sequenceStr string, runner lib.Runner,
) (context.Context, context.CancelFunc, lib.Executor, *lib.ExecutionState, *testutils.SimpleLogrusHook) {
	var err error
	var segment *lib.ExecutionSegment
	if segmentStr != "" {
		segment, err = lib.NewExecutionSegmentFromString(segmentStr)
		require.NoError(t, err)
	}
	var sequence lib.ExecutionSegmentSequence
	if sequenceStr != "" {
		sequence, err = lib.NewExecutionSegmentSequenceFromString(sequenceStr)
		require.NoError(t, err)
	}
	et, err := lib.NewExecutionTuple(segment, &sequence)
	require.NoError(t, err)

	options := lib.Options{
		ExecutionSegment:         segment,
		ExecutionSegmentSequence: &sequence,
	}.Apply(runner.GetOptions())

	testRunState := getTestRunState(t, options, runner)
	execReqs := config.GetExecutionRequirements(et)
	es := lib.NewExecutionState(testRunState, et, lib.GetMaxPlannedVUs(execReqs), lib.GetMaxPossibleVUs(execReqs))

	logHook := testutils.NewLogHook(logrus.DebugLevel)
	testLog := logrus.New()
	testLog.AddHook(logHook)
	testLog.SetOutput(io.Discard)
	testLog.SetLevel(logrus.DebugLevel)
	logEntry := logrus.NewEntry(testLog)

	ctx, cancel := context.WithCancel(context.Background())
	engineOut := make(chan metrics.SampleContainer, 100)
	initVUFunc := func(_ context.Context, _ *logrus.Entry) (lib.InitializedVU, error) {
		idl, idg := es.GetUniqueVUIdentifiers()
		return es.Test.Runner.NewVU(ctx, idl, idg, engineOut)
	}
	es.SetInitVUFunc(initVUFunc)
	maxPlannedVUs := lib.GetMaxPlannedVUs(config.GetExecutionRequirements(es.ExecutionTuple))
	for i := uint64(0); i < maxPlannedVUs; i++ {
		vu, ierr := initVUFunc(ctx, logEntry)
		require.NoError(t, ierr)
		es.AddInitializedVU(vu)
	}
	executor, err := config.NewExecutor(es, logEntry)
	require.NoError(t, err)
	require.NoError(t, executor.Init(ctx))
	return ctx, cancel, executor, es, logHook
}

// TestBlitzyProbeStateTransitions drives ONE real vuHandle through a deterministic
// sequence and observes the resulting state after each control call, answering Q(c):
// all three control methods lock the SAME per-VU mutex and the documented transition
// table governs every move. It also confirms getVU/returnVU are 1:1 at the vuHandle
// level (the invariant at vu_handle.go:62).
func TestBlitzyProbeStateTransitions(t *testing.T) {
	t.Parallel()
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	logHook := testutils.NewLogHook(logrus.DebugLevel)
	testLog := logrus.New()
	testLog.AddHook(logHook)
	testLog.SetOutput(io.Discard)
	testLog.SetLevel(logrus.DebugLevel)
	logEntry := logrus.NewEntry(testLog).WithField("vuNum", 0)

	runner := simpleRunner(func(rctx context.Context, _ *lib.State) error {
		// Short iteration so a running VU actually completes an iteration.
		select {
		case <-rctx.Done():
		case <-time.After(50 * time.Millisecond):
		}
		return nil
	})
	require.NoError(t, runner.SetOptions(lib.Options{}))

	var getVUCount, returnVUCount int64
	getVU := func() (lib.InitializedVU, error) {
		atomic.AddInt64(&getVUCount, 1)
		return runner.NewVU(ctx, uint64(atomic.LoadInt64(&getVUCount)), 0, nil)
	}
	returnVU := func(_ lib.InitializedVU) {
		atomic.AddInt64(&returnVUCount, 1)
	}

	vh := newStoppedVUHandle(ctx, getVU, returnVU, mockNextIterations, &BaseConfig{}, logEntry)
	go vh.runLoopsIfPossible(func(ir context.Context, avu lib.ActiveVU) bool {
		if avu != nil {
			_ = avu.RunOnce()
		}
		select {
		case <-ir.Done():
			return false
		default:
			return true
		}
	})

	settle := func() { time.Sleep(120 * time.Millisecond) }

	t.Logf("Q(c) single-vuHandle state-machine trace (initial state=%s):", stateName(readState(vh)))
	// stopped -> start -> (starting/running)
	before := stateName(readState(vh))
	require.NoError(t, vh.start())
	settle()
	t.Logf("  start()        : %-14s -> %-14s", before, stateName(readState(vh)))
	// running -> gracefulStop -> (toGracefulStop -> eventually stopped after iteration)
	before = stateName(readState(vh))
	vh.gracefulStop()
	settle()
	t.Logf("  gracefulStop() : %-14s -> %-14s", before, stateName(readState(vh)))
	// stopped -> start -> starting/running (reactivates the VU)
	before = stateName(readState(vh))
	require.NoError(t, vh.start())
	time.Sleep(20 * time.Millisecond)
	t.Logf("  start()        : %-14s -> %-14s", before, stateName(readState(vh)))
	// running/starting -> hardStop -> stopped
	before = stateName(readState(vh))
	vh.hardStop()
	settle()
	t.Logf("  hardStop()     : %-14s -> %-14s", before, stateName(readState(vh)))

	cancel()
	time.Sleep(80 * time.Millisecond)
	final := stateName(readState(vh))
	t.Logf("  after cancel() : final state=%s", final)

	// Emit the captured debug transition lines in order.
	entries := logHook.Drain()
	t.Logf("Captured vuHandle debug transition lines (Message @ level):")
	for _, e := range entries {
		t.Logf("    level=%s msg=%q vuNum=%v", e.Level, e.Message, e.Data["vuNum"])
	}
	t.Logf("vuHandle-level acquire/return accounting: getVU=%d returnVU=%d (must be equal, invariant vu_handle.go:62)",
		atomic.LoadInt64(&getVUCount), atomic.LoadInt64(&returnVUCount))
	require.Equal(t, atomic.LoadInt64(&getVUCount), atomic.LoadInt64(&returnVUCount),
		"getVU/returnVU must be 1:1")
}

// blitzyRapidConfig is the SINGLE unchanged input used for Symptoms A & B and the
// Q(b) accounting run: rapid up/down stages (1s each) with a LONG GracefulRampDown
// (3s, well above each stage) and an explicit GracefulStop (1s). Iterations sleep
// 300ms so a running VU performs real work inside the graceful window.
func blitzyRapidConfig() RampingVUsConfig {
	c := RampingVUsConfig{
		BaseConfig:       NewBaseConfig("blitzy_rapid", rampingVUsType),
		StartVUs:         null.IntFrom(0),
		GracefulRampDown: types.NullDurationFrom(3 * time.Second),
		Stages: []Stage{
			{Duration: types.NullDurationFrom(1 * time.Second), Target: null.IntFrom(8)},
			{Duration: types.NullDurationFrom(1 * time.Second), Target: null.IntFrom(0)},
			{Duration: types.NullDurationFrom(1 * time.Second), Target: null.IntFrom(8)},
			{Duration: types.NullDurationFrom(1 * time.Second), Target: null.IntFrom(0)},
		},
	}
	c.GracefulStop = types.NullDurationFrom(1 * time.Second)
	return c
}

func blitzyRapidRunner() lib.Runner {
	return simpleRunner(func(rctx context.Context, _ *lib.State) error {
		select {
		case <-rctx.Done():
		case <-time.After(300 * time.Millisecond):
		}
		return nil
	})
}

// TestBlitzyProbeBufferAccounting answers Q(b): it reconciles, per run, the active-VU
// trajectory (before/during/after) and the buffer restoration after Run returns
// (Run blocks on `defer runState.wg.Wait()` [ramping_vus.go:540] so every returnVU has
// completed). It confirms the buffer is fully restored (no orphaned handles).
func TestBlitzyProbeBufferAccounting(t *testing.T) {
	t.Parallel()
	config := blitzyRapidConfig()
	ctx, cancel, executor, es, _ := blitzyProbeExecutor(t, config, "", "", blitzyRapidRunner())
	defer cancel()

	initialized := es.GetInitializedVUsCount()
	t.Logf("Q(b) buffer accounting — config: rapid up/down 8<->0 @1s x4, GracefulRampDown=3s, GracefulStop=1s")
	t.Logf("  initialized VUs in buffer at start: %d", initialized)
	t.Logf("  active-VU count BEFORE Run: %d", es.GetCurrentlyActiveVUsCount())

	errCh := make(chan error, 1)
	go func() { errCh <- executor.Run(ctx, nil) }()

	var peak int64
	stop := time.After(9 * time.Second)
	ticker := time.NewTicker(200 * time.Millisecond)
	defer ticker.Stop()
	done := false
	for !done {
		select {
		case err := <-errCh:
			require.NoError(t, err)
			done = true
		case <-ticker.C:
			cur := es.GetCurrentlyActiveVUsCount()
			if cur > peak {
				peak = cur
			}
		case <-stop:
			t.Fatal("Run did not complete within 9s")
		}
	}

	finalActive := es.GetCurrentlyActiveVUsCount()
	t.Logf("  active-VU count AFTER Run returns: %d (peak observed during run: %d)", finalActive, peak)

	// Buffer restoration: after Run returns, wg.Wait has completed, so every acquired VU
	// must be back in the buffer. Drain exactly `initialized` VUs via the public
	// GetPlannedVU [lib/execution.go:471]; all must return quickly (no 5x400ms timeout).
	drainStart := time.Now()
	drained := int64(0)
	logHook := testutils.NewLogHook(logrus.DebugLevel)
	testLog := logrus.New()
	testLog.AddHook(logHook)
	testLog.SetOutput(io.Discard)
	drainLog := logrus.NewEntry(testLog)
	for i := int64(0); i < initialized; i++ {
		vu, err := es.GetPlannedVU(drainLog, false)
		require.NoError(t, err)
		require.NotNil(t, vu)
		drained++
	}
	elapsed := time.Since(drainStart)
	t.Logf("  buffer restoration: drained %d/%d planned VUs in %s (all present => none orphaned)",
		drained, initialized, elapsed.Round(time.Millisecond))
	require.Equal(t, initialized, drained, "every initialized VU must be back in the buffer")
	require.Equal(t, int64(0), finalActive, "net active-VU count must return to 0")
}

// TestBlitzyProbeDepletedBuffer answers the Q(b) FAILURE path (Finding 5): when the buffer
// is empty, GetPlannedVU [lib/execution.go:471] retries MaxRetriesGetPlannedVU=5 times
// [lib/execution.go:29], emits a "Could not get a VU from the buffer for ..." warning each
// time [lib/execution.go:481], and finally returns an error [lib/execution.go:484-487].
// getVU [ramping_vus.go:595-600] returns BEFORE incrementing on this error, so a failed
// acquisition is never paired with a spurious return.
func TestBlitzyProbeDepletedBuffer(t *testing.T) {
	t.Parallel()
	runner := simpleRunner(func(_ context.Context, _ *lib.State) error { return nil })
	require.NoError(t, runner.SetOptions(lib.Options{}))
	et, err := lib.NewExecutionTuple(nil, nil)
	require.NoError(t, err)
	trs := getTestRunState(t, lib.Options{}, runner)
	// maxPlannedVUs=1, maxPossibleVUs=1 → buffer capacity 1, but we AddInitializedVU ZERO
	// VUs, so the buffer is empty.
	es := lib.NewExecutionState(trs, et, 1, 1)

	logHook := testutils.NewLogHook(logrus.WarnLevel, logrus.DebugLevel)
	testLog := logrus.New()
	testLog.AddHook(logHook)
	testLog.SetOutput(io.Discard)
	testLog.SetLevel(logrus.DebugLevel)
	logEntry := logrus.NewEntry(testLog)

	t.Logf("Q(b) depleted-buffer failure path — calling GetPlannedVU on an EMPTY buffer (0 initialized VUs):")
	start := time.Now()
	vu, err := es.GetPlannedVU(logEntry, false)
	elapsed := time.Since(start)
	for _, line := range logHook.Lines() {
		t.Logf("    %s", line)
	}
	t.Logf("  returned: vu==nil? %v ; err=%q ; elapsed=%s (expect ~5x400ms=2s)",
		vu == nil, errStr(err), elapsed.Round(10*time.Millisecond))
	require.Error(t, err, "empty buffer must yield an error after 5 retries")
	require.Nil(t, vu)
}

func errStr(err error) string {
	if err == nil {
		return "<nil>"
	}
	return err.Error()
}

// TestBlitzyProbeHandlerDivergence answers Symptom B: it prints the exact raw and graceful
// planned-step tables (the executor's precomputed steps [ramping_vus.go:471,480-483]) and
// the reconstructed per-handler `cur` series, then runs the executor and captures the real
// Start/Graceful stop/Hard stop transitions with relative timestamps. The two handlers track
// DIFFERENT quantities: the scheduled handler [ramping_vus.go:679] follows the raw target;
// the max-allowed handler [ramping_vus.go:668] follows the graceful ceiling.
func TestBlitzyProbeHandlerDivergence(t *testing.T) {
	t.Parallel()
	config := blitzyRapidConfig()
	ctx, cancel, executor, _, logHook := blitzyProbeExecutor(t, config, "", "", blitzyRapidRunner())
	defer cancel()

	rv, ok := executor.(*RampingVUs)
	require.True(t, ok)

	// Exact planned-step tables (source of the two independent `cur` counters).
	t.Logf("Symptom B — raw steps (scheduledVUsHandlerStrategy target, cur := raw.PlannedVUs):")
	t.Logf("    idx |  TimeOffset | PlannedVUs")
	for i, s := range rv.rawSteps {
		t.Logf("    %3d | %10s | %d", i, s.TimeOffset.Round(time.Millisecond), s.PlannedVUs)
	}
	t.Logf("Symptom B — graceful steps (maxAllowedVUsHandlerStrategy ceiling, cur := graceful.PlannedVUs):")
	t.Logf("    idx |  TimeOffset | PlannedVUs")
	for i, s := range rv.gracefulSteps {
		t.Logf("    %3d | %10s | %d", i, s.TimeOffset.Round(time.Millisecond), s.PlannedVUs)
	}
	// Reconstructed divergence table (INFERRED from the steps: cur is closure-local &
	// unexported, ramping_vus.go:669,680 — we recompute what each handler's cur would be).
	t.Logf("Symptom B — reconstructed ceiling-vs-target over time (INFERRED from steps above):")
	t.Logf("    time(ms) | scheduled target | max-allowed ceiling | ceiling>=target?")
	times := mergedOffsets(rv.rawSteps, rv.gracefulSteps)
	for _, tm := range times {
		target := plannedAt(rv.rawSteps, tm)
		ceiling := plannedAt(rv.gracefulSteps, tm)
		t.Logf("    %8d | %16d | %19d | %v", tm.Milliseconds(), target, ceiling, ceiling >= target)
	}

	// Now RUN it and capture the real transitions with relative timestamps.
	start := time.Now()
	errCh := make(chan error, 1)
	go func() { errCh <- executor.Run(ctx, nil) }()
	select {
	case err := <-errCh:
		require.NoError(t, err)
	case <-time.After(9 * time.Second):
		t.Fatal("Run did not complete within 9s")
	}

	entries := logHook.Drain()
	type tr struct {
		off time.Duration
		msg string
		vu  interface{}
	}
	trs := make([]tr, 0, len(entries))
	var starts, gstops, hstops int
	for _, e := range entries {
		switch e.Message {
		case "Start":
			starts++
		case "Graceful stop":
			gstops++
		case "Hard stop":
			hstops++
		default:
			continue
		}
		trs = append(trs, tr{off: e.Time.Sub(start), msg: e.Message, vu: e.Data["vuNum"]})
	}
	sort.Slice(trs, func(i, j int) bool { return trs[i].off < trs[j].off })
	t.Logf("Symptom B — observed vuHandle transitions (real debug lines, relative to Run start):")
	for _, r := range trs {
		t.Logf("    t=%8s  %-13s vuNum=%v", r.off.Round(time.Millisecond), r.msg, r.vu)
	}
	t.Logf("Symptom B — transition TOTALS: Start(scheduled grow)=%d  Graceful stop(scheduled shrink)=%d  Hard stop(max-allowed ceiling shrink)=%d",
		starts, gstops, hstops)
}

// mergedOffsets returns the sorted unique TimeOffsets across both step slices.
func mergedOffsets(a, b []lib.ExecutionStep) []time.Duration {
	seen := map[time.Duration]bool{}
	var out []time.Duration
	for _, s := range a {
		if !seen[s.TimeOffset] {
			seen[s.TimeOffset] = true
			out = append(out, s.TimeOffset)
		}
	}
	for _, s := range b {
		if !seen[s.TimeOffset] {
			seen[s.TimeOffset] = true
			out = append(out, s.TimeOffset)
		}
	}
	sort.Slice(out, func(i, j int) bool { return out[i] < out[j] })
	return out
}

// plannedAt returns the PlannedVUs in effect at time offset tm (the last step whose
// TimeOffset <= tm), i.e. the value the corresponding handler's `cur` would hold.
func plannedAt(steps []lib.ExecutionStep, tm time.Duration) uint64 {
	var v uint64
	for _, s := range steps {
		if s.TimeOffset <= tm {
			v = s.PlannedVUs
		} else {
			break
		}
	}
	return v
}

// distStr formats an int64->count distribution as "value×count" pairs sorted by value,
// e.g. {8:10} => "8×10" (all 10 runs observed a value of 8).
func distStr(m map[int64]int) string {
	keys := make([]int64, 0, len(m))
	for k := range m {
		keys = append(keys, k)
	}
	sort.Slice(keys, func(i, j int) bool { return keys[i] < keys[j] })
	parts := make([]string, 0, len(keys))
	for _, k := range keys {
		parts = append(parts, fmt.Sprintf("%d×%d", k, m[k]))
	}
	return strings.Join(parts, ", ")
}

// TestBlitzyProbeStuckVUsDistribution answers User Symptom A ("stuck" VUs). It runs the
// SAME rapid up/down + long gracefulRampDown config 10 identical times (Rule 1: reproduce
// the "sometimes" report by repeating the unchanged input) and, for each run, captures the
// full active-VU trajectory: BEFORE Run (t=0), a 100ms-sampled series DURING Run (each
// sample paired with the scheduled target from rawSteps so the graceful lag is visible),
// and AFTER Run returns. It also correlates each run with the observed vuHandle debug
// transitions (Start/Graceful stop/Hard stop). The active-VU count is the canonical
// observable GetCurrentlyActiveVUsCount [lib/execution.go:269] — the same signal the
// no-wobble test samples [lib/executor/ramping_vus_test.go:376]. Because Run blocks on
// `defer runState.wg.Wait()` [lib/executor/ramping_vus.go:540], every acquired VU has been
// returned by the time Run returns, so a permanently "stuck" VU would show as final != 0.
func TestBlitzyProbeStuckVUsDistribution(t *testing.T) {
	t.Parallel()
	const runs = 10
	config := blitzyRapidConfig()
	t.Logf("Symptom A — config: rapid up/down 8<->0 @1s x4 (StartVUs=0), GracefulRampDown=3s, "+
		"GracefulStop=1s, iteration sleep=300ms; %d identical (unchanged-input) runs", runs)
	t.Logf("Symptom A — OBSERVED signal = GetCurrentlyActiveVUsCount [lib/execution.go:269]; "+
		"'target' column = scheduled PlannedVUs from rawSteps (the scheduledVUsHandlerStrategy goal).")
	t.Logf("Symptom A — INFERRED (from vu_handle.go:19,24-55, not directly sampled): when active > target "+
		"during a down-stage, the surplus VUs are in the transient toGracefulStop state — mid-iteration and "+
		"finishing their current ~300ms iteration before returnVU; this is the 'stuck'-looking lag.")

	type runResult struct {
		id          int
		before      int64
		peak        int64
		final       int64
		samples     []int64
		targets     []uint64
		sampleTimes []time.Duration
		starts      int
		gstops      int
		hstops      int
	}
	results := make([]runResult, 0, runs)

	for r := 1; r <= runs; r++ {
		ctx, cancel, executor, es, logHook := blitzyProbeExecutor(t, config, "", "", blitzyRapidRunner())
		rv, ok := executor.(*RampingVUs)
		require.True(t, ok)

		rr := runResult{id: r}
		rr.before = es.GetCurrentlyActiveVUsCount()

		start := time.Now()
		errCh := make(chan error, 1)
		go func() { errCh <- executor.Run(ctx, nil) }()

		ticker := time.NewTicker(100 * time.Millisecond)
		stop := time.After(12 * time.Second)
		done := false
		for !done {
			select {
			case err := <-errCh:
				require.NoError(t, err)
				done = true
			case <-ticker.C:
				off := time.Since(start)
				cur := es.GetCurrentlyActiveVUsCount()
				rr.samples = append(rr.samples, cur)
				rr.targets = append(rr.targets, plannedAt(rv.rawSteps, off))
				rr.sampleTimes = append(rr.sampleTimes, off.Round(100*time.Millisecond))
				if cur > rr.peak {
					rr.peak = cur
				}
			case <-stop:
				cancel()
				t.Fatal("Run did not complete within 12s")
			}
		}
		ticker.Stop()
		rr.final = es.GetCurrentlyActiveVUsCount()

		for _, e := range logHook.Drain() {
			switch e.Message {
			case "Start":
				rr.starts++
			case "Graceful stop":
				rr.gstops++
			case "Hard stop":
				rr.hstops++
			}
		}
		cancel()
		results = append(results, rr)
	}

	// Per-run full before/during/after trajectory with transition correlation.
	for _, rr := range results {
		t.Logf("Symptom A run %d/%d: before(t=0)=%d  peak=%d  final(after Run)=%d  "+
			"transitions[Start=%d GracefulStop=%d HardStop=%d]",
			rr.id, runs, rr.before, rr.peak, rr.final, rr.starts, rr.gstops, rr.hstops)
		var sb strings.Builder
		for i := range rr.samples {
			lag := int64(rr.samples[i]) - int64(rr.targets[i])
			marker := ""
			if lag > 0 {
				marker = "*" // active exceeds scheduled target => transient graceful lag
			}
			fmt.Fprintf(&sb, "\n      t=%-6s active=%d target=%d lag=%+d%s",
				rr.sampleTimes[i], rr.samples[i], rr.targets[i], lag, marker)
		}
		t.Logf("  active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):%s",
			sb.String())
	}

	// Run-to-run distribution.
	peaks := map[int64]int{}
	finals := map[int64]int{}
	maxLag := map[int64]int{}
	var stuck int
	for _, rr := range results {
		peaks[rr.peak]++
		finals[rr.final]++
		var ml int64
		for i := range rr.samples {
			if l := int64(rr.samples[i]) - int64(rr.targets[i]); l > ml {
				ml = l
			}
		}
		maxLag[ml]++
		if rr.final != 0 {
			stuck++
		}
	}
	t.Logf("Symptom A — DISTRIBUTION over %d identical runs (value×count):", runs)
	t.Logf("  peak active-VU count:                 %s", distStr(peaks))
	t.Logf("  final active-VU count (after Run):    %s", distStr(finals))
	t.Logf("  max transient surplus (active-target): %s", distStr(maxLag))
	t.Logf("  runs leaving a permanently STUCK VU (final != 0): %d/%d", stuck, runs)
	require.Equal(t, 0, stuck,
		"no run may leave a VU permanently stuck: Run blocks on wg.Wait [ramping_vus.go:540]")
}

// TestBlitzyProbeSegmentScaleArithmetic answers the deterministic half of Symptom D:
// with a SHARED --execution-segment-sequence, every instance builds the SAME filled
// sequence (GetFilledExecutionSegmentSequence [lib/execution_segment.go:445]) and scales
// via the striped ExecutionSegmentSequenceWrapper.ScaleInt64 [lib/execution_segment.go:580].
// For every global target T the three per-segment shares SUM EXACTLY to T, one segment
// consistently rounds up (deterministic striping), and recomputing is identical every
// time (coordination-free, no randomness — k6 issue #997). This is NOT overshoot-filtered:
// the full T=1..12 table is printed.
func TestBlitzyProbeSegmentScaleArithmetic(t *testing.T) {
	t.Parallel()
	const seqStr = "0,1/3,2/3,1"
	segStrs := []string{"0:1/3", "1/3:2/3", "2/3:1"}

	seq, err := lib.NewExecutionSegmentSequenceFromString(seqStr)
	require.NoError(t, err)
	ets := make([]*lib.ExecutionTuple, len(segStrs))
	for i, ss := range segStrs {
		seg, serr := lib.NewExecutionSegmentFromString(ss)
		require.NoError(t, serr)
		et, terr := lib.NewExecutionTuple(seg, &seq)
		require.NoError(t, terr)
		ets[i] = et
	}

	t.Logf("Symptom D (shared sequence %q) — deterministic striped ScaleInt64 [execution_segment.go:580]:", seqStr)
	t.Logf("      T | %-9s | %-11s | %-9s | sum | sum vs T", segStrs[0], segStrs[1], segStrs[2])
	overCount, underCount := 0, 0
	for tgt := int64(1); tgt <= 12; tgt++ {
		s := make([]int64, len(ets))
		var sum int64
		for i, et := range ets {
			s[i] = et.ScaleInt64(tgt)
			sum += s[i]
		}
		rel := "EQUAL"
		switch {
		case sum > tgt:
			rel = "OVER"
			overCount++
		case sum < tgt:
			rel = "UNDER"
			underCount++
		}
		t.Logf("    %3d | %9d | %11d | %9d | %3d | %s (sum-T=%+d)", tgt, s[0], s[1], s[2], sum, rel, sum-tgt)
	}
	t.Logf("Symptom D (shared sequence) — OVER rows: %d, UNDER rows: %d (both must be 0; sum==T always)",
		overCount, underCount)
	require.Equal(t, 0, overCount, "shared sequence: per-segment shares must never sum above the global target")
	require.Equal(t, 0, underCount, "shared sequence: per-segment shares must never sum below the global target")

	// Determinism: recompute the whole table 5 times; every value must be identical.
	base := make([][]int64, 0, 12)
	for tgt := int64(1); tgt <= 12; tgt++ {
		row := make([]int64, len(ets))
		for i, et := range ets {
			row[i] = et.ScaleInt64(tgt)
		}
		base = append(base, row)
	}
	for rep := 0; rep < 5; rep++ {
		for ti, tgt := 0, int64(1); tgt <= 12; ti, tgt = ti+1, tgt+1 {
			for i, et := range ets {
				require.Equal(t, base[ti][i], et.ScaleInt64(tgt),
					"ScaleInt64 must be deterministic across recomputation")
			}
		}
	}
	t.Logf("Symptom D (shared sequence) — recomputed 5×: identical every time (deterministic, coordination-free).")

	// Which segment consistently carries the surplus at the first non-divisible target?
	first := base[0] // T=1
	hi := 0
	for i := range first {
		if first[i] > first[hi] {
			hi = i
		}
	}
	t.Logf("Symptom D (shared sequence) — at T=1 the surplus VU goes to segment index %d (%s): %v => "+
		"one instance is CONSISTENTLY higher (deterministic), never random.", hi, segStrs[hi], first)
}

// TestBlitzyProbeIndependentUnderEqualOver answers the "sum exceeds my configured maximum"
// half of Symptom D. When the sequence is MISSING/mismatched, each instance is built with a
// nil sequence, so GetFilledExecutionSegmentSequence [lib/execution_segment.go:445] fills a
// DIFFERENT sequence around each segment (e.g. 0:1/3 -> [0:1/3,1/3:1]; 1/3:2/3 ->
// [0:1/3,1/3:2/3,2/3:1]; 2/3:1 -> [0:2/3,2/3:1]) with different LCDs/striping. The three
// independent shares therefore need NOT sum to T: the FULL T=1..12 table below shows the
// UNDER, EQUAL and OVER (sum > global max) cases — the checked-in-arithmetic explanation for
// "if I sum them up they exceed my configured maximum", with NO overshoot filtering.
func TestBlitzyProbeIndependentUnderEqualOver(t *testing.T) {
	t.Parallel()
	segStrs := []string{"0:1/3", "1/3:2/3", "2/3:1"}
	ets := make([]*lib.ExecutionTuple, len(segStrs))
	for i, ss := range segStrs {
		seg, serr := lib.NewExecutionSegmentFromString(ss)
		require.NoError(t, serr)
		et, terr := lib.NewExecutionTuple(seg, nil) // nil sequence => independent fill
		require.NoError(t, terr)
		ets[i] = et
		t.Logf("Symptom D (independent) — segment %-8s filled sequence: %s", ss, et.Sequence)
	}

	t.Logf("Symptom D (NO shared sequence, each instance independent) — full under/equal/over table:")
	t.Logf("      T | %-9s | %-11s | %-9s | sum | sum vs global max T", segStrs[0], segStrs[1], segStrs[2])
	over, equal, under := 0, 0, 0
	for tgt := int64(1); tgt <= 12; tgt++ {
		s := make([]int64, len(ets))
		var sum int64
		for i, et := range ets {
			s[i] = et.ScaleInt64(tgt)
			sum += s[i]
		}
		rel := "EQUAL"
		switch {
		case sum > tgt:
			rel = "OVER"
			over++
		case sum < tgt:
			rel = "UNDER"
			under++
		default:
			equal++
		}
		t.Logf("    %3d | %9d | %11d | %9d | %3d | %s (sum-T=%+d)", tgt, s[0], s[1], s[2], sum, rel, sum-tgt)
	}
	t.Logf("Symptom D (independent) — classification across T=1..12: OVER=%d EQUAL=%d UNDER=%d", over, equal, under)
	t.Logf("Symptom D (independent) — OVER rows are real: without a shared --execution-segment-sequence the "+
		"per-instance shares can sum ABOVE the configured maximum (deterministic per T, still never random).")
	require.Greater(t, over, 0,
		"independent fill must exhibit at least one OVER row (sum above global max) to explain Symptom D")
}
```

### 9.2 Symptom C harness (`symptomC_run.sh`) — real SIGINT to the `k6` binary

This Bash harness produced §7-C. It uses strict shell options (`set -euo pipefail`), a `mktemp -d`
workdir, an `EXIT` trap that kills only the exact spawned PID and removes the workdir (no broad
`pkill`), monotonic timing via `python3 time.monotonic()`, and ~1 ms poll-based exit detection. The three
JavaScript scenarios (`sleep_iter.js`, `busy_iter.js`, `teardown_iter.js`) are created inline by the
`cat > … <<'JS_EOF'` heredocs at the top of the script (lines 47, 58, 72), so the complete JS source is
visible here as well.

```bash
#!/usr/bin/env bash
#
# symptomC_run.sh — TEMPORARY, EPHEMERAL investigation harness for User Symptom C
# ("when I kill the test early with ctrl+c, some VUs keep running for way longer
#  than gracefulStop should allow").
#
# It launches the REAL k6 binary through its canonical `k6 run` entry point, delivers
# genuine os.Interrupt (SIGINT) signals to the exact child PID, detects the actual
# process-exit instant by polling, and records the SIGINT->exit latency with a MONOTONIC
# clock (python3 time.monotonic()). Each condition runs RUNS (>=2) identical times.
#
#   C1 single SIGINT, interruptible sleep() VUs   (js/modules/k6/k6.go:71-80: sleep respects ctx.Done)
#   C2 single SIGINT, CPU-busy non-yielding VUs    (JS VM interrupt still cancels them)
#   C3 single SIGINT, script WITH teardown() busy  (graceful abort runs teardown to completion)
#   C4 double SIGINT, same teardown() script       (2nd signal -> onHardStop -> OSExit, cutting teardown short)
#
# Signal path under test:
#   cmd/common.go:97 handleTestAbortSignals; SignalNotify(os.Interrupt,SIGINT,SIGTERM) :101
#     1st signal -> gracefulStop closure cmd/run.go:349-357
#                   ("Stopping k6 in response to signal..." Debug :350; runAbort ExternalAbort)
#     2nd signal -> onHardStop closure cmd/run.go:359-362
#                   ("Aborting k6 in response to signal" Error :360)
#                   then gs.OSExit(int(exitcodes.ExternalAbort)) cmd/common.go:118
#   ExternalAbort ExitCode = 105  (errext/exitcodes/codes.go:41; comment :38-40)
#
set -euo pipefail

K6="${K6:-/tmp/k6}"
export K6_NO_USAGE_REPORT=true
RUNS="${RUNS:-2}"
WORKDIR="$(mktemp -d /tmp/k6probe_symC.XXXXXX)"
declare -a CHILDREN=()

cleanup() {
    local p
    for p in "${CHILDREN[@]:-}"; do
        [ -n "${p:-}" ] || continue
        if kill -0 "$p" 2>/dev/null; then kill -KILL "$p" 2>/dev/null || true; fi
    done
    rm -rf "$WORKDIR"
}
trap cleanup EXIT

mono()  { python3 -c 'import time; print("%.6f" % time.monotonic())'; }
py_ms() { python3 -c "import sys;a=float(sys.argv[1]);b=float(sys.argv[2]);print('%.1f'%((b-a)*1000.0))" "$1" "$2"; }

cat > "$WORKDIR/sleep_iter.js" <<'JS_EOF'
import { sleep } from 'k6';
export const options = {
  scenarios: { c: { executor: 'ramping-vus', startVUs: 0,
    stages: [{ duration: '1s', target: 5 }, { duration: '600s', target: 5 }],
    gracefulStop: '30s', gracefulRampDown: '30s' } },
};
// Interruptible: k6 sleep() returns immediately on ctx.Done (js/modules/k6/k6.go:76-78).
export default function () { sleep(600); }
JS_EOF

cat > "$WORKDIR/busy_iter.js" <<'JS_EOF'
export const options = {
  scenarios: { c: { executor: 'ramping-vus', startVUs: 0,
    stages: [{ duration: '1s', target: 3 }, { duration: '600s', target: 3 }],
    gracefulStop: '30s', gracefulRampDown: '30s' } },
};
// CPU-busy, NON-yielding iteration (~6s, bounded so the harness always terminates).
export default function () {
  const end = Date.now() + 6000; let x = 0;
  while (Date.now() < end) { x += Math.sqrt(x + 1.0); }
  if (x < 0) { console.log(x); }
}
JS_EOF

cat > "$WORKDIR/teardown_iter.js" <<'JS_EOF'
export const options = {
  scenarios: { c: { executor: 'ramping-vus', startVUs: 0,
    stages: [{ duration: '1s', target: 2 }, { duration: '600s', target: 2 }],
    gracefulStop: '30s', gracefulRampDown: '30s' } },
};
export default function () {
  const end = Date.now() + 6000; let x = 0;
  while (Date.now() < end) { x += Math.sqrt(x + 1.0); }
  if (x < 0) { console.log(x); }
}
// teardown() runs during graceful abort; this ~3s busy teardown makes the graceful
// phase long enough to reliably exercise the 2nd-SIGINT hard-stop escalation.
export function teardown() {
  console.log('TEARDOWN_START');
  const end = Date.now() + 3000; let x = 0;
  while (Date.now() < end) { x += Math.sqrt(x + 1.0); }
  console.log('TEARDOWN_END');
}
JS_EOF

run_one() {
    local label="$1" script="$2" first_delay="$3" second_delay="$4" run_no="$5"
    local out="$WORKDIR/${label}_run${run_no}.out"
    local err="$WORKDIR/${label}_run${run_no}.err"
    local tsf="$WORKDIR/${label}_run${run_no}.tsecond"
    local ksf="$WORKDIR/${label}_run${run_no}.ks2"

    echo "===== CONDITION ${label} — run ${run_no}/${RUNS} ====="
    echo "\$ $K6 run --verbose --no-summary --no-usage-report ${script##*/}   (first SIGINT @+${first_delay}s${second_delay:+, second @+${second_delay}s after first})"
    "$K6" run --verbose --no-summary --no-usage-report "$script" >"$out" 2>"$err" &
    local pid=$!
    CHILDREN+=("$pid")
    echo "spawned k6 pid=$pid"

    sleep "$first_delay"
    local t_sig=""
    if kill -0 "$pid" 2>/dev/null; then
        t_sig="$(mono)"
        local ks; if kill -INT "$pid" 2>/dev/null; then ks=0; else ks=$?; fi
        echo "first  SIGINT: delivered_at_mono=$t_sig  kill_status=$ks  process_alive_before_send=yes"
    else
        echo "first  SIGINT: NOT SENT — process already exited before +${first_delay}s"
    fi

    # Background second-SIGINT sender (double-SIGINT conditions), independent of exit polling.
    local sender_pid=""
    if [ -n "$second_delay" ] && [ -n "$t_sig" ]; then
        (
            sleep "$second_delay"
            if kill -0 "$pid" 2>/dev/null; then
                mono > "$tsf"
                if kill -INT "$pid" 2>/dev/null; then echo 0 > "$ksf"; else echo $? > "$ksf"; fi
            fi
        ) &
        sender_pid=$!
        CHILDREN+=("$sender_pid")
    fi

    # Poll for the actual exit instant (~1ms resolution) — accurate SIGINT->exit latency.
    while kill -0 "$pid" 2>/dev/null; do sleep 0.001; done
    local t_exit; t_exit="$(mono)"
    [ -n "$sender_pid" ] && { wait "$sender_pid" 2>/dev/null || true; }

    local t_second=""; [ -f "$tsf" ] && t_second="$(cat "$tsf")"
    if [ -n "$second_delay" ]; then
        if [ -n "$t_second" ]; then
            local ks2="?"; [ -f "$ksf" ] && ks2="$(cat "$ksf")"
            echo "second SIGINT: delivered_at_mono=$t_second  kill_status=$ks2  process_alive_before_send=yes"
        else
            echo "second SIGINT: NOT SENT — process exited before scheduled +${second_delay}s"
        fi
    fi

    set +e; wait "$pid"; local rc=$?; set -e
    echo "wait result: pid=$pid exit_code=$rc  (105 == ExternalAbort)"
    [ -n "$t_sig" ]    && echo "elapsed first_SIGINT  -> process_exit: $(py_ms "$t_sig" "$t_exit") ms"
    [ -n "$t_second" ] && echo "elapsed second_SIGINT -> process_exit: $(py_ms "$t_second" "$t_exit") ms"
    echo "--- signal-relevant log lines (k6 stderr, --verbose) ---"
    grep -E 'Stopping k6 in response to signal|Aborting k6 in response to signal|test run was aborted because k6' "$err" || echo "(none matched)"
    echo "--- setup/teardown markers (console.log -> stderr) ---"
    grep -E 'TEARDOWN_START|TEARDOWN_END' "$err" || echo "(no teardown markers)"
    echo "--- iteration progress (final states) ---"
    grep -E 'complete and [0-9]+ interrupted iterations' "$out" | tail -2 || true
    echo "===== end ${label} run ${run_no}: exit=${rc} ====="
    echo
}

echo "################ k6 build under test ################"
"$K6" version
echo "workdir: $WORKDIR ; RUNS per condition: $RUNS"
echo

for r in $(seq 1 "$RUNS"); do run_one "C1_sleep_singleSIGINT"    "$WORKDIR/sleep_iter.js"    2   ""  "$r"; done
for r in $(seq 1 "$RUNS"); do run_one "C2_busy_singleSIGINT"     "$WORKDIR/busy_iter.js"     2.5 ""  "$r"; done
for r in $(seq 1 "$RUNS"); do run_one "C3_teardown_singleSIGINT" "$WORKDIR/teardown_iter.js" 2.5 ""  "$r"; done
for r in $(seq 1 "$RUNS"); do run_one "C4_teardown_doubleSIGINT" "$WORKDIR/teardown_iter.js" 2.5 0.8 "$r"; done

echo "################ ALL SYMPTOM C CONDITIONS COMPLETE ################"
```

### 9.3 Symptom D harness (`symptomD_run.sh`) — synchronized 3-process CLI runs

This Bash harness produced §7-D.2. It launches three real `k6 run` processes simultaneously (one per
segment), in both SHARED and INDEPENDENT modes, twice each, and parses per-instance active-VU counts
from k6's stdout progress lines with a carry-forward alignment (embedded `python3` parser). The
`seg.js` scenario is created inline by the `cat > … <<'JS_EOF'` heredoc at line 42.

```bash
#!/usr/bin/env bash
#
# symptomD_run.sh — TEMPORARY, EPHEMERAL investigation harness for User Symptom D
# ("one instance consistently shows more VUs than the others at the same timestamp,
#  and if I sum them up they exceed my configured maximum").
#
# It launches THREE REAL k6 processes SIMULTANEOUSLY through the canonical `k6 run`
# entry point, each pinned to a non-overlapping --execution-segment. Two modes, each
# RUNS (>=2) synchronized triples:
#   SHARED      : all three share ONE --execution-segment-sequence 0,1/3,2/3,1 (canonical)
#   INDEPENDENT : NO sequence supplied (each instance fills its own) -> the missing/mismatched case
# Per instance it captures the scenario-header max planned VUs ("Up to N looping VUs"),
# the active-VU progress time series ("running (Xs), a/m VUs"), and the exit code.
# A python parser aligns the three series (carry-forward step function) by k6 internal
# elapsed and reports per-timestamp sums.
#
# Flags: cmd/options.go:31 --execution-segment ; :32 --execution-segment-sequence.
# Scaling: ExecutionSegmentSequenceWrapper.ScaleInt64 [lib/execution_segment.go:580];
#          GetFilledExecutionSegmentSequence [lib/execution_segment.go:445].
#
set -euo pipefail

K6="${K6:-/tmp/k6}"
export K6_NO_USAGE_REPORT=true
RUNS="${RUNS:-2}"
GLOBAL_MAX=11
SEQ="0,1/3,2/3,1"
SEGS=("0:1/3" "1/3:2/3" "2/3:1")
WORKDIR="$(mktemp -d /tmp/k6probe_symD.XXXXXX)"
declare -a CHILDREN=()

cleanup() {
    local p
    for p in "${CHILDREN[@]:-}"; do
        [ -n "${p:-}" ] || continue
        if kill -0 "$p" 2>/dev/null; then kill -KILL "$p" 2>/dev/null || true; fi
    done
    rm -rf "$WORKDIR"
}
trap cleanup EXIT

cat > "$WORKDIR/seg.js" <<'JS_EOF'
import { sleep } from 'k6';
// Global peak target 11 VUs. With 3 equal thirds, 11 is NOT divisible by 3, so the
// per-segment split is uneven (shared: 4/4/3=11; independent: 4/4/4=12>11).
export const options = {
  scenarios: { d: { executor: 'ramping-vus', startVUs: 0,
    stages: [
      { duration: '3s', target: 11 },
      { duration: '2s', target: 11 },
      { duration: '3s', target: 0  },
    ] } },
};
export default function () { sleep(1); }
JS_EOF

cat > "$WORKDIR/parse.py" <<'PY_EOF'
import sys, re
mode, gmax = sys.argv[1], int(sys.argv[2])
segs = sys.argv[3:6]
errfiles = sys.argv[6:9]
maxvus, series = [], []
for f in errfiles:
    text = open(f).read()
    m = re.search(r'Up to (\d+) looping VUs', text)
    maxvus.append(int(m.group(1)) if m else -1)
    s = {}
    for mm in re.finditer(r'running \((?:(\d+)m)?([\d.]+)s\), (\d+)/(\d+) VUs', text):
        mins = int(mm.group(1)) if mm.group(1) else 0
        elapsed = round(mins * 60 + float(mm.group(2)), 1)
        s[elapsed] = int(mm.group(3))
    series.append(s)

print("  per-instance max planned VUs (scenario header 'Up to N looping VUs'):")
for i, seg in enumerate(segs):
    print("    seg%d %-8s => %d" % (i, seg, maxvus[i]))
tot = sum(maxvus)
rel = "OVER (exceeds configured max!)" if tot > gmax else ("EQUAL" if tot == gmax else "UNDER")
print("    SUM of per-instance max VUs = %d  vs configured global max = %d  => %s" % (tot, gmax, rel))

def at(s, t):
    v = 0
    for tt in sorted(s):
        if tt <= t + 1e-9:
            v = s[tt]
        else:
            break
    return v

times = sorted(set().union(*[set(s.keys()) for s in series])) if any(series) else []
print("  aligned active-VU series (carry-forward by k6 internal elapsed):")
print("    elapsed | seg0 | seg1 | seg2 | sum | vs gmax")
peak_sum = 0
for t in times:
    vals = [at(series[i], t) for i in range(3)]
    ssum = sum(vals)
    peak_sum = max(peak_sum, ssum)
    tag = "OVER" if ssum > gmax else ("=" if ssum == gmax else "")
    print("    %6.1fs | %4d | %4d | %4d | %3d | %s" % (t, vals[0], vals[1], vals[2], ssum, tag))
print("  peak instantaneous active-VU sum across aligned samples = %d (configured max %d)" % (peak_sum, gmax))
peaks = [max(s.values()) if s else 0 for s in series]
hi = peaks.index(max(peaks))
print("  per-instance peak active VUs: %s ; highest = seg%d (%s) => one instance consistently higher" % (peaks, hi, segs[hi]))
PY_EOF

run_triple() {
    local mode="$1" run_no="$2"
    local -a pids=() outs=() rc=()
    local i out
    echo "-------- ${mode^^} sequence, SYNCHRONIZED 3-process triple, run ${run_no}/${RUNS} --------"
    for i in 0 1 2; do
        out="$WORKDIR/${mode}_run${run_no}_seg${i}.out"
        outs[i]="$out"
        if [ "$mode" = "shared" ]; then
            echo "\$ $K6 run --execution-segment ${SEGS[i]} --execution-segment-sequence $SEQ seg.js"
            "$K6" run --execution-segment "${SEGS[i]}" --execution-segment-sequence "$SEQ" \
                --no-summary --no-usage-report "$WORKDIR/seg.js" >"$out" 2>"$WORKDIR/${mode}_run${run_no}_seg${i}.err" &
        else
            echo "\$ $K6 run --execution-segment ${SEGS[i]} seg.js   (NO --execution-segment-sequence)"
            "$K6" run --execution-segment "${SEGS[i]}" \
                --no-summary --no-usage-report "$WORKDIR/seg.js" >"$out" 2>"$WORKDIR/${mode}_run${run_no}_seg${i}.err" &
        fi
        pids[i]=$!
        CHILDREN+=("${pids[i]}")
    done
    for i in 0 1 2; do set +e; wait "${pids[i]}"; rc[i]=$?; set -e; done
    echo "exit codes: seg0=${rc[0]} seg1=${rc[1]} seg2=${rc[2]}"
    python3 "$WORKDIR/parse.py" "$mode" "$GLOBAL_MAX" "${SEGS[0]}" "${SEGS[1]}" "${SEGS[2]}" \
        "${outs[0]}" "${outs[1]}" "${outs[2]}"
    echo "-------- end ${mode} run ${run_no} --------"
    echo
}

echo "################ k6 build under test ################"
"$K6" version
echo "workdir: $WORKDIR ; RUNS per mode: $RUNS ; shared sequence: $SEQ ; segments: ${SEGS[*]} ; global max target: $GLOBAL_MAX"
echo

for r in $(seq 1 "$RUNS"); do run_triple "shared"      "$r"; done
for r in $(seq 1 "$RUNS"); do run_triple "independent" "$r"; done

echo "################ ALL SYMPTOM D TRIPLES COMPLETE ################"
```

### 9.4 Cleanup attestation and repository integrity

This was a read-only investigation. The only change committed to the repository is this single document.
The temporary artifacts used to capture the evidence are, by category:

- **Built binary (outside the repo tree):** `/tmp/k6` — the investigation binary built in §3.3. This is
  distinct from the environment-setup-owned `/tmp/k6_bin`, which this investigation neither produced nor
  relies on and does not remove.
- **Captured logs and harness scripts (outside the repo tree):** `/tmp/k6probe_out/` — the raw `.log`
  transcripts embedded above, plus `symptomC_run.sh` and `symptomD_run.sh`. The JavaScript scenarios were
  created inside each harness's own `mktemp -d` workdir and were removed by that harness's `EXIT` trap as
  soon as it finished.
- **In-package probe (inside the tree, but untracked and never committed):**
  `lib/executor/blitzy_adhoc_test_probe_test.go` — carries the `blitzy_adhoc_test_` prefix and is deleted
  after capture.

After evidence capture, these temporary artifacts are removed with the following commands, and the
repository is verified to contain only this document as an addition to the investigated source tree
(`git status --porcelain` empty apart from this file; the k6 source tree byte-for-byte unchanged):

```text
rm -f /tmp/k6
rm -rf /tmp/k6probe_out
rm -f lib/executor/blitzy_adhoc_test_probe_test.go
git status --porcelain    # expected: only blitzy/documentation/k6_ddc3b0b1d23c.md
```

No existing k6 source or test file was modified, refactored, or deleted at any point in this
investigation.
