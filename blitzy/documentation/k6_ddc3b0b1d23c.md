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
  the real transcript, not an annotation added after the fact. A first normalization applied to the
  captured output is that trailing whitespace (a cosmetic artifact of fixed-width column padding in the
  probe's and harnesses' formatting) has been trimmed for lint-cleanliness; no value, log message,
  count, timing, or footer was altered. The substantive content of every fenced block — the values, the
  log-message text, the counts, the timings, and the `PASS`/`ok` footers — is verbatim from the run that
  produced it. A second normalization reconciles the source-line-number prefix that Go's `t.Logf`
  automatically prepends to each probe log line (e.g., `blitzy_adhoc_test_probe_test.go:320:`) to the
  consolidated §9.1 listing, so that each numeric prefix names the exact `t.Logf` call in §9.1 that
  emitted the line; this was verified programmatically - all 414 transcript prefixes in §4-§7 resolve to
  a real `t.Logf` statement in the §9.1 source, and the message text after each prefix is unaltered. The
  only quantities that are inherently run-specific are the goroutine identifiers in the handler-ordering
  trace (§4, e.g., `g36`/`g42`), the wall-clock durations in the `--- PASS`/`ok` footers, and a few
  timing-sampled values (the per-transition offsets in §7-B and the 100 ms active-VU trajectory samples
  in §7-A); these vary slightly between otherwise-identical runs, while every other value, count, table,
  and message is deterministic and reproduces exactly across runs (confirmed under `-race -count=2`).

---

## 1. Summary (top-line answer)

The reported symptoms do **not** stem from an open data race between two competing handler goroutines,
nor from a VU-buffer leak. On the canonical `ramping-vus` path, observed at runtime:

1. **There are not two concurrently-running handler goroutines to race.** The two "handlers" are the
   `maxAllowedVUsHandlerStrategy` `lib/executor/ramping_vus.go:668` and the
   `scheduledVUsHandlerStrategy` `lib/executor/ramping_vus.go:679`. They are invoked **sequentially**,
   one call per loop iteration, from a single `for` loop inside `iterateSteps`
   `lib/executor/ramping_vus.go:622-643`, which runs in the main `Run` goroutine
   `lib/executor/ramping_vus.go:491`. This is not inferred from reading: a direct runtime trace that
   wraps the two **real** strategies and drives them through the **real** `iterateSteps` sequencer
   records every scheduling call on a **single** goroutine with **`max handler bodies executing
   simultaneously = 1`**, the graceful tail contributing exactly one later `maxAllowed` call on a
   *separate* goroutine only after `iterateSteps` returns (§4.1). `go test -race` over both the
   VU-handle race suite and the full ramping-vus suite is additionally clean across two iterations
   each (§4.4).
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
| **C — VUs outrun `gracefulStop` on ctrl+c** | Re-characterized — **not** reproduced as a VU/`gracefulStop` defect | A single real `SIGINT` cancels VU iteration contexts and interrupts the JS VM, so a sleeping iteration, a CPU-busy iteration, an iteration at a stage edge, and an iteration already in `toGracefulStop` all stop in **~7–8 ms** (exit code 105); a no-signal natural stage-end exits `0`. The only way to observe "keeps running for seconds" is non-VU **graceful-phase work** such as `teardown()` (~3 s here) running to completion during the graceful abort — which is **not** a per-VU iteration overrunning `gracefulStop`. A second `SIGINT` escalates to an immediate hard stop (~7–8 ms, teardown truncated). The user's literal "VUs keep running longer than `gracefulStop`" was not reproduced on the canonical path **(INFERRED:** the user likely observed graceful-phase work, or a non-yielding native call inside an iteration**)**. (§7-C) |
| **D — one instance shows more VUs; sum exceeds max** | Yes — deterministically | With a **shared** `--execution-segment-sequence`, three synchronized instances split a global max of 11 as canonical `vus_max = [4, 4, 3]`; the remainder `11 % 3 = 2` deterministically lands on the **two earliest** segments (`0:1/3`, `1/3:2/3`), so **two instances tie** at the top (not one), sum **== 11** (never exceeds). The number of top instances equals the remainder — at max 1 it is a single one (`[1, 0, 0]`). The sum **exceeds** the configured max only when instances are run **without** a shared sequence (`[4, 4, 4]`, all three tied, sum **12 > 11**), because each independently fills its own sequence and rounds up. All cases are deterministic per the striping arithmetic — not random, not a race. (§7-D) |

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
state concurrently. The sequential premise is confirmed directly at runtime: a trace that wraps the two
**real** strategies and drives them through the **real** `iterateSteps` sequencer records every
scheduling call on a **single** goroutine with **`max handler bodies executing simultaneously = 1`** —
the two strategies never overlap — with the graceful tail contributing exactly one later `maxAllowed`
call on a *separate* goroutine only after `iterateSteps` returns (§4.1). Running the VU-handle race
suite and the full ramping-vus suite under `go test -race` (two iterations each) additionally reports
**zero** data races (§4.4). (Caveat: the race detector only observes exercised interleavings.)

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
run fully offline. (Rather than pin an ephemeral, self-referential documentation-commit hash — a file
cannot contain the hash of the commit that will contain it — the block below records the *stable
invariant* directly: baseline→`HEAD` adds only this one markdown file, so no k6 source differs from the
investigated commit `ddc3b0b1d23c`. §3.2 elaborates and shows this is independently re-verifiable.)

```text
### git branch --show-current
blitzy-cfb43038-9f4d-4c7a-9f83-818b7e754ba5
exit=0

### git diff --name-only ddc3b0b1d23c HEAD -- . ':(exclude)blitzy/documentation/k6_ddc3b0b1d23c.md'
exit=0  (empty output above => no k6 source file differs from the investigated commit ddc3b0b1d23c)

### git diff ddc3b0b1d23c..HEAD --name-status
A	blitzy/documentation/k6_ddc3b0b1d23c.md
exit=0

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

This branch carries the k6 source at the investigated commit `ddc3b0b1d23c` ("Update comment") plus a
sequence of documentation-only commits that each add or revise **only** this one markdown file. Because
those commits touch **no** k6 source, the working tree's source stays byte-identical to `ddc3b0b1d23c`
regardless of which documentation commit is checked out — so every `file:line` reference in this document
resolves equally against the current `HEAD` and against `ddc3b0b1d23c`. This is the stable invariant that
makes the self-referential `HEAD` hash irrelevant, and it is independently re-verifiable at any time with
`git diff --name-only ddc3b0b1d23c HEAD -- . ':(exclude)blitzy/documentation/k6_ddc3b0b1d23c.md'`, which
prints nothing (empty output, exit 0), and with `git diff ddc3b0b1d23c..HEAD --name-status`, which lists
exactly one added path — the document itself. The `go.mod` minimum (`go 1.21`) is satisfied by the local toolchain (Go 1.23.6), which is
also the highest CI-supported line (`DEFAULT_GO_VERSION: "1.23.x"`). The exit code used by the ctrl+c path
(§7-C), `ExternalAbort = 105`, is defined here as well.

```text
### Stable invariant: only the documentation file differs from investigated commit ddc3b0b1d23c
--- git diff ddc3b0b1d23c..HEAD --name-status (baseline->HEAD adds only this one file) ---
A	blitzy/documentation/k6_ddc3b0b1d23c.md

--- git diff --name-only ddc3b0b1d23c HEAD -- . ':(exclude)blitzy/documentation/k6_ddc3b0b1d23c.md' ---
(empty output => no k6 source file differs from ddc3b0b1d23c)

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

`go mod verify` confirms the vendored modules are intact, and k6 builds offline from the vendored tree
in ~2.5 s (timing below). The binary under test for the real-binary reproductions (§7-C, §7-D) is the
canonical **baseline** build — `k6 v0.55.0`, `commit/ddc3b0b1d2` (the investigated commit
`ddc3b0b1d23c`) — produced offline during environment setup and kept **outside** the repository tree at
`/tmp/k6` (the setup also produced the byte-identical copy `/tmp/k6_bin`; `sha256` of the two matches at
`eb304828f77bfd14…`), so the repo is left unchanged. `/tmp/k6` is removed during cleanup (§9.4). The
`commit/ddc3b0b1d2` suffix in the version string is the build-time `HEAD`, here matching the
investigated commit exactly.

```text
### go mod verify
all modules verified
exit=0

### vendor mode check — head -5 vendor/modules.txt
# buf.build/gen/go/gogo/protobuf/protocolbuffers/go v1.31.0-20210810001428-4df00b267f94.1
## explicit
buf.build/gen/go/gogo/protobuf/protocolbuffers/go/gogoproto
# buf.build/gen/go/prometheus/prometheus/protocolbuffers/go v1.31.0-20230627135113-9a12bc2590d2.1
## explicit

### explicit module count — grep -c '^## explicit' vendor/modules.txt
94

### Offline build from vendored tree (reproducibility check)

real	0m2.534s
user	0m3.634s
sys	0m2.493s
build exit=0

### /tmp/k6 version
k6 v0.55.0 (commit/ddc3b0b1d2, go1.23.6, linux/amd64)
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

**Direct runtime confirmation (run-first, canonical, under `-race`).** The serialization above is not
asserted from reading `iterateSteps` alone — it is confirmed by a direct trace that wraps the two **real**
handler strategies (`maxAllowedVUsHandlerStrategy` `lib/executor/ramping_vus.go:668` and
`scheduledVUsHandlerStrategy` `lib/executor/ramping_vus.go:679`) in a recorder and drives them through the
**real** sequencer `iterateSteps` `lib/executor/ramping_vus.go:622`, followed by the **real** graceful tail
`runRemainingGracefulSteps` `lib/executor/ramping_vus.go:654` launched in its own goroutine exactly as `Run`
does (`go runState.runRemainingGracefulSteps(...)` `lib/executor/ramping_vus.go:554`). Each invocation is
timestamped and tagged with its goroutine id and phase; an atomic depth counter records the maximum number
of handler bodies ever executing simultaneously. The input is the canonical rapid up/down config
(8↔0 ×4, `GracefulRampDown=3s`). Reproduced ×2 under `-race` with no `WARNING: DATA RACE`.

**Command:**

```text
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -race -run 'TestBlitzyProbeHandlerOrdering' ./lib/executor/ -count=2 -v
```

Complete output of one representative run (the second `-count=2` iteration is structurally identical — same
two-goroutine split, same `max simultaneous = 1`, same `0` scheduled calls in the tail; only the runtime
goroutine ids and exact offsets differ):

```text
=== RUN   TestBlitzyProbeHandlerOrdering
=== PAUSE TestBlitzyProbeHandlerOrdering
=== CONT  TestBlitzyProbeHandlerOrdering
    blitzy_adhoc_test_probe_test.go:901: Q(a)/M1 direct handler-invocation trace (rawSteps=33 gracefulSteps=10 maxVUs=8):
    blitzy_adhoc_test_probe_test.go:903:   seq | phase         | handler    | goroutine | TimeOffset | PlannedVUs
    blitzy_adhoc_test_probe_test.go:905:      1 | iterateSteps | scheduled  | g36       |         0s | 0
    blitzy_adhoc_test_probe_test.go:905:      2 | iterateSteps | maxAllowed | g36       |         0s | 0
    blitzy_adhoc_test_probe_test.go:905:      3 | iterateSteps | scheduled  | g36       |      125ms | 1
    blitzy_adhoc_test_probe_test.go:905:      4 | iterateSteps | maxAllowed | g36       |      125ms | 1
    blitzy_adhoc_test_probe_test.go:905:      5 | iterateSteps | scheduled  | g36       |      250ms | 2
    blitzy_adhoc_test_probe_test.go:905:      6 | iterateSteps | maxAllowed | g36       |      250ms | 2
    blitzy_adhoc_test_probe_test.go:905:      7 | iterateSteps | scheduled  | g36       |      375ms | 3
    blitzy_adhoc_test_probe_test.go:905:      8 | iterateSteps | maxAllowed | g36       |      375ms | 3
    blitzy_adhoc_test_probe_test.go:905:      9 | iterateSteps | scheduled  | g36       |      500ms | 4
    blitzy_adhoc_test_probe_test.go:905:     10 | iterateSteps | maxAllowed | g36       |      500ms | 4
    blitzy_adhoc_test_probe_test.go:905:     11 | iterateSteps | scheduled  | g36       |      625ms | 5
    blitzy_adhoc_test_probe_test.go:905:     12 | iterateSteps | maxAllowed | g36       |      625ms | 5
    blitzy_adhoc_test_probe_test.go:905:     13 | iterateSteps | scheduled  | g36       |      750ms | 6
    blitzy_adhoc_test_probe_test.go:905:     14 | iterateSteps | maxAllowed | g36       |      750ms | 6
    blitzy_adhoc_test_probe_test.go:905:     15 | iterateSteps | scheduled  | g36       |      875ms | 7
    blitzy_adhoc_test_probe_test.go:905:     16 | iterateSteps | maxAllowed | g36       |      875ms | 7
    blitzy_adhoc_test_probe_test.go:905:     17 | iterateSteps | scheduled  | g36       |         1s | 8
    blitzy_adhoc_test_probe_test.go:905:     18 | iterateSteps | maxAllowed | g36       |         1s | 8
    blitzy_adhoc_test_probe_test.go:905:     19 | iterateSteps | scheduled  | g36       |     1.125s | 7
    blitzy_adhoc_test_probe_test.go:905:     20 | iterateSteps | scheduled  | g36       |      1.25s | 6
    blitzy_adhoc_test_probe_test.go:905:     21 | iterateSteps | scheduled  | g36       |     1.375s | 5
    blitzy_adhoc_test_probe_test.go:905:     22 | iterateSteps | scheduled  | g36       |       1.5s | 4
    blitzy_adhoc_test_probe_test.go:905:     23 | iterateSteps | scheduled  | g36       |     1.625s | 3
    blitzy_adhoc_test_probe_test.go:905:     24 | iterateSteps | scheduled  | g36       |      1.75s | 2
    blitzy_adhoc_test_probe_test.go:905:     25 | iterateSteps | scheduled  | g36       |     1.875s | 1
    blitzy_adhoc_test_probe_test.go:905:     26 | iterateSteps | scheduled  | g36       |         2s | 0
    blitzy_adhoc_test_probe_test.go:905:     27 | iterateSteps | scheduled  | g36       |     2.125s | 1
    blitzy_adhoc_test_probe_test.go:905:     28 | iterateSteps | scheduled  | g36       |      2.25s | 2
    blitzy_adhoc_test_probe_test.go:905:     29 | iterateSteps | scheduled  | g36       |     2.375s | 3
    blitzy_adhoc_test_probe_test.go:905:     30 | iterateSteps | scheduled  | g36       |       2.5s | 4
    blitzy_adhoc_test_probe_test.go:905:     31 | iterateSteps | scheduled  | g36       |     2.625s | 5
    blitzy_adhoc_test_probe_test.go:905:     32 | iterateSteps | scheduled  | g36       |      2.75s | 6
    blitzy_adhoc_test_probe_test.go:905:     33 | iterateSteps | scheduled  | g36       |     2.875s | 7
    blitzy_adhoc_test_probe_test.go:905:     34 | iterateSteps | scheduled  | g36       |         3s | 8
    blitzy_adhoc_test_probe_test.go:905:     35 | iterateSteps | scheduled  | g36       |     3.125s | 7
    blitzy_adhoc_test_probe_test.go:905:     36 | iterateSteps | scheduled  | g36       |      3.25s | 6
    blitzy_adhoc_test_probe_test.go:905:     37 | iterateSteps | scheduled  | g36       |     3.375s | 5
    blitzy_adhoc_test_probe_test.go:905:     38 | iterateSteps | scheduled  | g36       |       3.5s | 4
    blitzy_adhoc_test_probe_test.go:905:     39 | iterateSteps | scheduled  | g36       |     3.625s | 3
    blitzy_adhoc_test_probe_test.go:905:     40 | iterateSteps | scheduled  | g36       |      3.75s | 2
    blitzy_adhoc_test_probe_test.go:905:     41 | iterateSteps | scheduled  | g36       |     3.875s | 1
    blitzy_adhoc_test_probe_test.go:905:     42 | iterateSteps | scheduled  | g36       |         4s | 0
    blitzy_adhoc_test_probe_test.go:905:     43 | tail         | maxAllowed | g42       |         5s | 0
    blitzy_adhoc_test_probe_test.go:907: distinct goroutines that invoked ANY handler = 2 (expect 1 during iterateSteps + at most 1 for the tail)
    blitzy_adhoc_test_probe_test.go:908: max handler bodies executing simultaneously = 1 (expect 1 => no concurrent handler execution)
    blitzy_adhoc_test_probe_test.go:909: scheduled-handler calls during tail = 0 (expect 0); maxAllowed calls during tail = 1
    blitzy_adhoc_test_probe_test.go:910: totals: scheduled=33 maxAllowed=10 ; handledGracefulSteps(returned by iterateSteps)=9
--- PASS: TestBlitzyProbeHandlerOrdering (5.00s)
PASS
ok  	go.k6.io/k6/lib/executor	11.029s
```

This trace **directly answers Question (a)**. Every one of the 42 scheduling calls (both `scheduled` and
`maxAllowed`) executed on a **single** goroutine (`g36` in this run) during `iterateSteps`, strictly one at
a time — `max handler bodies executing simultaneously = 1`, so the two strategies **never** run
concurrently. The trace does record **2** distinct handler-invoking goroutines, but the second (`g42`) is
**only** the graceful tail: it fires a **single** `maxAllowed` call (`scheduled-handler calls during tail = 0`),
and only *after* `iterateSteps` has returned. So the "two handler goroutines racing on shared VU state"
premise does not hold: there is no instant at which the scheduled and max-allowed strategies are both
executing. (Goroutine ids and exact offsets differ between the two `-count=2` iterations; the structural
facts — one scheduling goroutine, `max simultaneous = 1`, `0` scheduled calls in the tail — are identical
across both.)

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
CGO_ENABLED=1 GOFLAGS=-mod=vendor go test -race -run 'TestBlitzyProbeStateTransitions' ./lib/executor/ -count=2 -v
```

This probe drives one real `vuHandle` (constructed via `newStoppedVUHandle`, exactly as the executor
does) and **directly samples the state at each transition point**, capturing all five states — including
the three *transient* ones (`starting`, `toGracefulStop`, `toHardStop`), not just the stable endpoints.
Determinism comes from a channel-gated iteration callback: `runIter` blocks on an unbuffered `proceed`
channel, so while a VU is `running` the loop is parked *inside* `runIter` and cannot advance the state
machine. That lets the probe call `gracefulStop()` / `hardStop()` and read the resulting **transient**
state *before* releasing the iteration; `starting` is captured by calling `start()` **before** launching
the run-loop goroutine (nothing has advanced it to `running` yet). The surplus VUs that linger in
`toGracefulStop` are the same lag mechanism that makes VUs *look* "stuck" in Symptom A (§7-A). State is
read via the real lock-free fast-path (`atomic.LoadInt32`), the actual DebugLevel transition lines are
captured, and VU acquire/return is counted. All five source state names
(`lib/executor/vu_handle.go:17-21`) are observed **directly**; acquire/return is one-to-one
(`getVU=2 == returnVU=2`), the invariant noted at `lib/executor/vu_handle.go:62`. Reproduced ×2 under
`-race` with no `WARNING: DATA RACE` (both iterations are shown; they are identical):

```text
=== RUN   TestBlitzyProbeStateTransitions
=== PAUSE TestBlitzyProbeStateTransitions
=== CONT  TestBlitzyProbeStateTransitions
    blitzy_adhoc_test_probe_test.go:224: Q(c)/M2 DIRECT five-state capture (one real vuHandle, channel-gated):
    blitzy_adhoc_test_probe_test.go:227:   initial                                    -> stopped
    blitzy_adhoc_test_probe_test.go:227:   start() [loop not launched]                -> starting
    blitzy_adhoc_test_probe_test.go:227:   loop entered iteration                     -> running
    blitzy_adhoc_test_probe_test.go:227:   gracefulStop() [iteration held]            -> toGracefulStop
    blitzy_adhoc_test_probe_test.go:227:   after iteration finishes                   -> stopped
    blitzy_adhoc_test_probe_test.go:227:   start() again -> loop entered iteration    -> running
    blitzy_adhoc_test_probe_test.go:227:   hardStop() [iteration held]                -> toHardStop
    blitzy_adhoc_test_probe_test.go:227:   after hardStop settles                     -> stopped
    blitzy_adhoc_test_probe_test.go:227:   after cancel()                             -> stopped
    blitzy_adhoc_test_probe_test.go:235: distinct states directly observed = [stopped starting running toGracefulStop toHardStop]
    blitzy_adhoc_test_probe_test.go:238: captured vuHandle debug transition lines:
    blitzy_adhoc_test_probe_test.go:240:     level=debug msg="Start" vuNum=0
    blitzy_adhoc_test_probe_test.go:240:     level=debug msg="Graceful stop" vuNum=0
    blitzy_adhoc_test_probe_test.go:240:     level=debug msg="Start" vuNum=0
    blitzy_adhoc_test_probe_test.go:240:     level=debug msg="Hard stop" vuNum=0
    blitzy_adhoc_test_probe_test.go:242: vuHandle acquire/return accounting: getVU=2 returnVU=2 (must be equal, invariant vu_handle.go:62)
--- PASS: TestBlitzyProbeStateTransitions (0.08s)
=== RUN   TestBlitzyProbeStateTransitions
=== PAUSE TestBlitzyProbeStateTransitions
=== CONT  TestBlitzyProbeStateTransitions
    blitzy_adhoc_test_probe_test.go:224: Q(c)/M2 DIRECT five-state capture (one real vuHandle, channel-gated):
    blitzy_adhoc_test_probe_test.go:227:   initial                                    -> stopped
    blitzy_adhoc_test_probe_test.go:227:   start() [loop not launched]                -> starting
    blitzy_adhoc_test_probe_test.go:227:   loop entered iteration                     -> running
    blitzy_adhoc_test_probe_test.go:227:   gracefulStop() [iteration held]            -> toGracefulStop
    blitzy_adhoc_test_probe_test.go:227:   after iteration finishes                   -> stopped
    blitzy_adhoc_test_probe_test.go:227:   start() again -> loop entered iteration    -> running
    blitzy_adhoc_test_probe_test.go:227:   hardStop() [iteration held]                -> toHardStop
    blitzy_adhoc_test_probe_test.go:227:   after hardStop settles                     -> stopped
    blitzy_adhoc_test_probe_test.go:227:   after cancel()                             -> stopped
    blitzy_adhoc_test_probe_test.go:235: distinct states directly observed = [stopped starting running toGracefulStop toHardStop]
    blitzy_adhoc_test_probe_test.go:238: captured vuHandle debug transition lines:
    blitzy_adhoc_test_probe_test.go:240:     level=debug msg="Start" vuNum=0
    blitzy_adhoc_test_probe_test.go:240:     level=debug msg="Graceful stop" vuNum=0
    blitzy_adhoc_test_probe_test.go:240:     level=debug msg="Start" vuNum=0
    blitzy_adhoc_test_probe_test.go:240:     level=debug msg="Hard stop" vuNum=0
    blitzy_adhoc_test_probe_test.go:242: vuHandle acquire/return accounting: getVU=2 returnVU=2 (must be equal, invariant vu_handle.go:62)
--- PASS: TestBlitzyProbeStateTransitions (0.09s)
PASS
ok  	go.k6.io/k6/lib/executor	1.190s
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
`rs.activeVUsCount` `lib/executor/ramping_vus.go:569`. The progress-counter update and the observable
update are two *distinct* atomic operations on adjacent lines within the same closure — the pair `:601`+`:602`
on acquire and `:607`+`:609` on return — so the two counters are **not** guaranteed equal at an arbitrary
instant: a concurrent reader can observe one store before its sibling and momentarily see them differ.
They **reconcile after each *completed* acquire or return**, and both settle to **zero** once `Run`
returns — by which point `defer runState.wg.Wait()` `lib/executor/ramping_vus.go:540` has released every
VU. INFERRED from the source pairing; the net-zero endpoint is confirmed by the observed run below.)
Therefore the evidence below pairs **(i)** net-zero active count with **(ii)**
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
    blitzy_adhoc_test_probe_test.go:292: Q(b) buffer accounting — config: rapid up/down 8<->0 @1s x4, GracefulRampDown=3s, GracefulStop=1s
    blitzy_adhoc_test_probe_test.go:293:   initialized VUs in buffer at start: 8
    blitzy_adhoc_test_probe_test.go:294:   active-VU count BEFORE Run: 0
    blitzy_adhoc_test_probe_test.go:320:   active-VU count AFTER Run returns: 0 (peak observed during run: 8)
    blitzy_adhoc_test_probe_test.go:339:   buffer restoration: drained 8/8 planned VUs in 0s (all present => none orphaned)
--- PASS: TestBlitzyProbeBufferAccounting (4.06s)
PASS
ok  	go.k6.io/k6/lib/executor	4.061s
```

Re-running the identical input with `-count=2` shows the same result both times:

```text
=== RUN   TestBlitzyProbeBufferAccounting
=== PAUSE TestBlitzyProbeBufferAccounting
=== CONT  TestBlitzyProbeBufferAccounting
    blitzy_adhoc_test_probe_test.go:292: Q(b) buffer accounting — config: rapid up/down 8<->0 @1s x4, GracefulRampDown=3s, GracefulStop=1s
    blitzy_adhoc_test_probe_test.go:293:   initialized VUs in buffer at start: 8
    blitzy_adhoc_test_probe_test.go:294:   active-VU count BEFORE Run: 0
    blitzy_adhoc_test_probe_test.go:320:   active-VU count AFTER Run returns: 0 (peak observed during run: 8)
    blitzy_adhoc_test_probe_test.go:339:   buffer restoration: drained 8/8 planned VUs in 0s (all present => none orphaned)
--- PASS: TestBlitzyProbeBufferAccounting (4.05s)
=== RUN   TestBlitzyProbeBufferAccounting
=== PAUSE TestBlitzyProbeBufferAccounting
=== CONT  TestBlitzyProbeBufferAccounting
    blitzy_adhoc_test_probe_test.go:292: Q(b) buffer accounting — config: rapid up/down 8<->0 @1s x4, GracefulRampDown=3s, GracefulStop=1s
    blitzy_adhoc_test_probe_test.go:293:   initialized VUs in buffer at start: 8
    blitzy_adhoc_test_probe_test.go:294:   active-VU count BEFORE Run: 0
    blitzy_adhoc_test_probe_test.go:320:   active-VU count AFTER Run returns: 0 (peak observed during run: 8)
    blitzy_adhoc_test_probe_test.go:339:   buffer restoration: drained 8/8 planned VUs in 0s (all present => none orphaned)
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
    blitzy_adhoc_test_probe_test.go:369: Q(b) depleted-buffer failure path — calling GetPlannedVU on an EMPTY buffer (0 initialized VUs):
    blitzy_adhoc_test_probe_test.go:374:     Could not get a VU from the buffer for 400ms
    blitzy_adhoc_test_probe_test.go:374:     Could not get a VU from the buffer for 800ms
    blitzy_adhoc_test_probe_test.go:374:     Could not get a VU from the buffer for 1.2s
    blitzy_adhoc_test_probe_test.go:374:     Could not get a VU from the buffer for 1.6s
    blitzy_adhoc_test_probe_test.go:374:     Could not get a VU from the buffer for 2s
    blitzy_adhoc_test_probe_test.go:376:   returned: vu==nil? true ; err="could not get a VU from the buffer in 2s" ; elapsed=2s (expect ~5x400ms=2s)
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
    blitzy_adhoc_test_probe_test.go:531: Symptom A — config: rapid up/down 8<->0 @1s x4 (StartVUs=0), GracefulRampDown=3s, GracefulStop=1s, iteration sleep=300ms; 10 identical (unchanged-input) runs
    blitzy_adhoc_test_probe_test.go:533: Symptom A — OBSERVED signal = GetCurrentlyActiveVUsCount [lib/execution.go:269]; 'target' column = scheduled PlannedVUs from rawSteps (the scheduledVUsHandlerStrategy goal).
    blitzy_adhoc_test_probe_test.go:535: Symptom A — INFERRED (from vu_handle.go:19,24-55, not directly sampled): when active > target during a down-stage, the surplus VUs are in the transient toGracefulStop state — mid-iteration and finishing their current ~300ms iteration before returnVU; this is the 'stuck'-looking lag.
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 1/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 2/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 3/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 4/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 5/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 6/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 7/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 8/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 9/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 10/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:642: Symptom A — DISTRIBUTION over 10 identical runs (value×count):
    blitzy_adhoc_test_probe_test.go:643:   peak active-VU count:                 8×10
    blitzy_adhoc_test_probe_test.go:644:   final active-VU count (after Run):    0×10
    blitzy_adhoc_test_probe_test.go:645:   max transient surplus (active-target): 2×10
    blitzy_adhoc_test_probe_test.go:646:   runs leaving a permanently STUCK VU (final != 0): 0/10
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
    blitzy_adhoc_test_probe_test.go:531: Symptom A — config: rapid up/down 8<->0 @1s x4 (StartVUs=0), GracefulRampDown=3s, GracefulStop=1s, iteration sleep=300ms; 10 identical (unchanged-input) runs
    blitzy_adhoc_test_probe_test.go:533: Symptom A — OBSERVED signal = GetCurrentlyActiveVUsCount [lib/execution.go:269]; 'target' column = scheduled PlannedVUs from rawSteps (the scheduledVUsHandlerStrategy goal).
    blitzy_adhoc_test_probe_test.go:535: Symptom A — INFERRED (from vu_handle.go:19,24-55, not directly sampled): when active > target during a down-stage, the surplus VUs are in the transient toGracefulStop state — mid-iteration and finishing their current ~300ms iteration before returnVU; this is the 'stuck'-looking lag.
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 1/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 2/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 3/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 4/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 5/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 6/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 7/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 8/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 9/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 10/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:642: Symptom A — DISTRIBUTION over 10 identical runs (value×count):
    blitzy_adhoc_test_probe_test.go:643:   peak active-VU count:                 8×10
    blitzy_adhoc_test_probe_test.go:644:   final active-VU count (after Run):    0×10
    blitzy_adhoc_test_probe_test.go:645:   max transient surplus (active-target): 2×10
    blitzy_adhoc_test_probe_test.go:646:   runs leaving a permanently STUCK VU (final != 0): 0/10
--- PASS: TestBlitzyProbeStuckVUsDistribution (40.55s)
=== RUN   TestBlitzyProbeStuckVUsDistribution
=== PAUSE TestBlitzyProbeStuckVUsDistribution
=== CONT  TestBlitzyProbeStuckVUsDistribution
    blitzy_adhoc_test_probe_test.go:531: Symptom A — config: rapid up/down 8<->0 @1s x4 (StartVUs=0), GracefulRampDown=3s, GracefulStop=1s, iteration sleep=300ms; 10 identical (unchanged-input) runs
    blitzy_adhoc_test_probe_test.go:533: Symptom A — OBSERVED signal = GetCurrentlyActiveVUsCount [lib/execution.go:269]; 'target' column = scheduled PlannedVUs from rawSteps (the scheduledVUsHandlerStrategy goal).
    blitzy_adhoc_test_probe_test.go:535: Symptom A — INFERRED (from vu_handle.go:19,24-55, not directly sampled): when active > target during a down-stage, the surplus VUs are in the transient toGracefulStop state — mid-iteration and finishing their current ~300ms iteration before returnVU; this is the 'stuck'-looking lag.
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 1/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 2/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 3/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 4/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 5/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 6/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 7/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 8/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 9/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:606: Symptom A run 10/10: before(t=0)=0  peak=8  final(after Run)=0  transitions[Start=16 GracefulStop=16 HardStop=0]
    blitzy_adhoc_test_probe_test.go:619:   active-VU trajectory (100ms samples; '*' = active>target, the 'stuck'-looking surplus):
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
    blitzy_adhoc_test_probe_test.go:642: Symptom A — DISTRIBUTION over 10 identical runs (value×count):
    blitzy_adhoc_test_probe_test.go:643:   peak active-VU count:                 8×10
    blitzy_adhoc_test_probe_test.go:644:   final active-VU count (after Run):    0×10
    blitzy_adhoc_test_probe_test.go:645:   max transient surplus (active-target): 2×10
    blitzy_adhoc_test_probe_test.go:646:   runs leaving a permanently STUCK VU (final != 0): 0/10
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
    blitzy_adhoc_test_probe_test.go:405: Symptom B — raw steps (scheduledVUsHandlerStrategy target, cur := raw.PlannedVUs):
    blitzy_adhoc_test_probe_test.go:406:     idx |  TimeOffset | PlannedVUs
    blitzy_adhoc_test_probe_test.go:408:       0 |         0s | 0
    blitzy_adhoc_test_probe_test.go:408:       1 |      125ms | 1
    blitzy_adhoc_test_probe_test.go:408:       2 |      250ms | 2
    blitzy_adhoc_test_probe_test.go:408:       3 |      375ms | 3
    blitzy_adhoc_test_probe_test.go:408:       4 |      500ms | 4
    blitzy_adhoc_test_probe_test.go:408:       5 |      625ms | 5
    blitzy_adhoc_test_probe_test.go:408:       6 |      750ms | 6
    blitzy_adhoc_test_probe_test.go:408:       7 |      875ms | 7
    blitzy_adhoc_test_probe_test.go:408:       8 |         1s | 8
    blitzy_adhoc_test_probe_test.go:408:       9 |     1.125s | 7
    blitzy_adhoc_test_probe_test.go:408:      10 |      1.25s | 6
    blitzy_adhoc_test_probe_test.go:408:      11 |     1.375s | 5
    blitzy_adhoc_test_probe_test.go:408:      12 |       1.5s | 4
    blitzy_adhoc_test_probe_test.go:408:      13 |     1.625s | 3
    blitzy_adhoc_test_probe_test.go:408:      14 |      1.75s | 2
    blitzy_adhoc_test_probe_test.go:408:      15 |     1.875s | 1
    blitzy_adhoc_test_probe_test.go:408:      16 |         2s | 0
    blitzy_adhoc_test_probe_test.go:408:      17 |     2.125s | 1
    blitzy_adhoc_test_probe_test.go:408:      18 |      2.25s | 2
    blitzy_adhoc_test_probe_test.go:408:      19 |     2.375s | 3
    blitzy_adhoc_test_probe_test.go:408:      20 |       2.5s | 4
    blitzy_adhoc_test_probe_test.go:408:      21 |     2.625s | 5
    blitzy_adhoc_test_probe_test.go:408:      22 |      2.75s | 6
    blitzy_adhoc_test_probe_test.go:408:      23 |     2.875s | 7
    blitzy_adhoc_test_probe_test.go:408:      24 |         3s | 8
    blitzy_adhoc_test_probe_test.go:408:      25 |     3.125s | 7
    blitzy_adhoc_test_probe_test.go:408:      26 |      3.25s | 6
    blitzy_adhoc_test_probe_test.go:408:      27 |     3.375s | 5
    blitzy_adhoc_test_probe_test.go:408:      28 |       3.5s | 4
    blitzy_adhoc_test_probe_test.go:408:      29 |     3.625s | 3
    blitzy_adhoc_test_probe_test.go:408:      30 |      3.75s | 2
    blitzy_adhoc_test_probe_test.go:408:      31 |     3.875s | 1
    blitzy_adhoc_test_probe_test.go:408:      32 |         4s | 0
    blitzy_adhoc_test_probe_test.go:410: Symptom B — graceful steps (maxAllowedVUsHandlerStrategy ceiling, cur := graceful.PlannedVUs):
    blitzy_adhoc_test_probe_test.go:411:     idx |  TimeOffset | PlannedVUs
    blitzy_adhoc_test_probe_test.go:413:       0 |         0s | 0
    blitzy_adhoc_test_probe_test.go:413:       1 |      125ms | 1
    blitzy_adhoc_test_probe_test.go:413:       2 |      250ms | 2
    blitzy_adhoc_test_probe_test.go:413:       3 |      375ms | 3
    blitzy_adhoc_test_probe_test.go:413:       4 |      500ms | 4
    blitzy_adhoc_test_probe_test.go:413:       5 |      625ms | 5
    blitzy_adhoc_test_probe_test.go:413:       6 |      750ms | 6
    blitzy_adhoc_test_probe_test.go:413:       7 |      875ms | 7
    blitzy_adhoc_test_probe_test.go:413:       8 |         1s | 8
    blitzy_adhoc_test_probe_test.go:413:       9 |         5s | 0
    blitzy_adhoc_test_probe_test.go:417: Symptom B — reconstructed ceiling-vs-target over time (INFERRED from steps above):
    blitzy_adhoc_test_probe_test.go:418:     time(ms) | scheduled target | max-allowed ceiling | ceiling>=target?
    blitzy_adhoc_test_probe_test.go:423:            0 |                0 |                   0 | true
    blitzy_adhoc_test_probe_test.go:423:          125 |                1 |                   1 | true
    blitzy_adhoc_test_probe_test.go:423:          250 |                2 |                   2 | true
    blitzy_adhoc_test_probe_test.go:423:          375 |                3 |                   3 | true
    blitzy_adhoc_test_probe_test.go:423:          500 |                4 |                   4 | true
    blitzy_adhoc_test_probe_test.go:423:          625 |                5 |                   5 | true
    blitzy_adhoc_test_probe_test.go:423:          750 |                6 |                   6 | true
    blitzy_adhoc_test_probe_test.go:423:          875 |                7 |                   7 | true
    blitzy_adhoc_test_probe_test.go:423:         1000 |                8 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         1125 |                7 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         1250 |                6 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         1375 |                5 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         1500 |                4 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         1625 |                3 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         1750 |                2 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         1875 |                1 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         2000 |                0 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         2125 |                1 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         2250 |                2 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         2375 |                3 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         2500 |                4 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         2625 |                5 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         2750 |                6 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         2875 |                7 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         3000 |                8 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         3125 |                7 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         3250 |                6 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         3375 |                5 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         3500 |                4 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         3625 |                3 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         3750 |                2 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         3875 |                1 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         4000 |                0 |                   8 | true
    blitzy_adhoc_test_probe_test.go:423:         5000 |                0 |                   0 | true
    blitzy_adhoc_test_probe_test.go:459: Symptom B — observed vuHandle transitions (real debug lines, relative to Run start):
    blitzy_adhoc_test_probe_test.go:461:     t=   126ms  Start         vuNum=0
    blitzy_adhoc_test_probe_test.go:461:     t=   250ms  Start         vuNum=1
    blitzy_adhoc_test_probe_test.go:461:     t=   376ms  Start         vuNum=2
    blitzy_adhoc_test_probe_test.go:461:     t=   501ms  Start         vuNum=3
    blitzy_adhoc_test_probe_test.go:461:     t=   626ms  Start         vuNum=4
    blitzy_adhoc_test_probe_test.go:461:     t=   750ms  Start         vuNum=5
    blitzy_adhoc_test_probe_test.go:461:     t=   876ms  Start         vuNum=6
    blitzy_adhoc_test_probe_test.go:461:     t=  1.001s  Start         vuNum=7
    blitzy_adhoc_test_probe_test.go:461:     t=  1.125s  Graceful stop vuNum=7
    blitzy_adhoc_test_probe_test.go:461:     t=   1.25s  Graceful stop vuNum=6
    blitzy_adhoc_test_probe_test.go:461:     t=  1.375s  Graceful stop vuNum=5
    blitzy_adhoc_test_probe_test.go:461:     t=  1.501s  Graceful stop vuNum=4
    blitzy_adhoc_test_probe_test.go:461:     t=  1.625s  Graceful stop vuNum=3
    blitzy_adhoc_test_probe_test.go:461:     t=   1.75s  Graceful stop vuNum=2
    blitzy_adhoc_test_probe_test.go:461:     t=  1.876s  Graceful stop vuNum=1
    blitzy_adhoc_test_probe_test.go:461:     t=  2.001s  Graceful stop vuNum=0
    blitzy_adhoc_test_probe_test.go:461:     t=  2.125s  Start         vuNum=0
    blitzy_adhoc_test_probe_test.go:461:     t=  2.251s  Start         vuNum=1
    blitzy_adhoc_test_probe_test.go:461:     t=  2.376s  Start         vuNum=2
    blitzy_adhoc_test_probe_test.go:461:     t=    2.5s  Start         vuNum=3
    blitzy_adhoc_test_probe_test.go:461:     t=  2.626s  Start         vuNum=4
    blitzy_adhoc_test_probe_test.go:461:     t=  2.751s  Start         vuNum=5
    blitzy_adhoc_test_probe_test.go:461:     t=  2.875s  Start         vuNum=6
    blitzy_adhoc_test_probe_test.go:461:     t=  3.001s  Start         vuNum=7
    blitzy_adhoc_test_probe_test.go:461:     t=  3.126s  Graceful stop vuNum=7
    blitzy_adhoc_test_probe_test.go:461:     t=  3.251s  Graceful stop vuNum=6
    blitzy_adhoc_test_probe_test.go:461:     t=  3.375s  Graceful stop vuNum=5
    blitzy_adhoc_test_probe_test.go:461:     t=    3.5s  Graceful stop vuNum=4
    blitzy_adhoc_test_probe_test.go:461:     t=  3.626s  Graceful stop vuNum=3
    blitzy_adhoc_test_probe_test.go:461:     t=  3.751s  Graceful stop vuNum=2
    blitzy_adhoc_test_probe_test.go:461:     t=  3.876s  Graceful stop vuNum=1
    blitzy_adhoc_test_probe_test.go:461:     t=  4.001s  Graceful stop vuNum=0
    blitzy_adhoc_test_probe_test.go:463: Symptom B — transition TOTALS: Start(scheduled grow)=16  Graceful stop(scheduled shrink)=16  Hard stop(max-allowed ceiling shrink)=0
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
iteration contexts and interrupts the JS VM, so even a CPU-busy iteration stops in **~7–8 ms** — VU
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
  `cmd/common.go:97` (`gs.SignalNotify` for `os.Interrupt`/`SIGINT`/`SIGTERM` `cmd/common.go:101`) invokes
  the `gracefulStop` signal closure `cmd/run.go:349` on the first signal (Debug "Stopping k6 in response
  to signal…" `cmd/run.go:350`) and the `onHardStop` closure `cmd/run.go:359` on the second (Error
  "Aborting k6 in response to signal" `cmd/run.go:360`), then `gs.OSExit(int(exitcodes.ExternalAbort))`
  `cmd/common.go:118`. VU iteration `sleep` is interruptible because it selects on `ctx.Done()`
  `js/modules/k6/k6.go:71-80`.

**Method.** The canonical baseline `k6` binary (`v0.55.0`, `commit/ddc3b0b1d2` — the investigated commit
`ddc3b0b1d23c`) is launched via `k6 run`; a real `SIGINT` is delivered to the exact spawned PID, and the
SIGINT→exit latency is measured with a high-resolution wall clock (`date +%s.%N`) using a busy-spin
`kill -0` exit poll (sub-millisecond detection). **Nine** conditions are each run twice (Rule 1),
spanning the natural path, the primary single-SIGINT path, and the secondary/edge paths the symptom
implies:

- **`natural_stage_end`** — a short ramp that ends **with no signal** (the ordinary stage-end baseline);
- **`natural_forced_grace`** — a no-signal ramp in which VUs are mid-iteration at the down-stage boundary,
  so `gracefulRampDown` extends their finish (still a clean exit);
- **`C1_sleep_singleSIGINT`** — sleeping iterations + a single SIGINT;
- **`C2_busy_singleSIGINT`** — CPU-busy (non-yielding) iterations + a single SIGINT;
- **`near_transition_SIGINT`** — a SIGINT delivered right at a stage boundary (edge timing);
- **`graceful_state_SIGINT`** — a SIGINT delivered only *after* an actual `Graceful stop` state log
  (`lib/executor/vu_handle.go:161`) has been observed in the live stderr stream (VUs already in
  `toGracefulStop`);
- **`C3_teardown_singleSIGINT`** — a script with a ~3 s busy `teardown()` + a single SIGINT;
- **`C4_teardown_doubleSIGINT`** — the same teardown script + a double SIGINT;
- **`rapid_double_SIGINT`** — a ~10 ms-spaced double SIGINT delivered during the teardown.

The hardened harness (`set -u`; strict `RUNS>=2` integer validation before any work; `mktemp -d` workdir;
exact-PID `kill -INT`; **bounded** `wait` with TERM→KILL escalation and removal of each reaped PID from
the cleanup-ownership array; `EXIT`/`INT`/`TERM` trap cleanup) and the JS scripts are embedded in §9.2.

**Command (per run, as echoed in the transcript):**

```text
/tmp/k6 run --verbose --no-summary --no-usage-report <scenario>.js
```

```text
################ k6 build under test ################
k6 v0.55.0 (commit/ddc3b0b1d2, go1.23.6, linux/amd64)
workdir: /tmp/k6probe_symC.DVoT3p ; RUNS per condition: 2
binary: /tmp/k6 (canonical baseline build, commit ddc3b0b1d2 == investigated ddc3b0b1d23c)

===== CONDITION natural_stage_end — run 1/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report natural.js   (first=none second=none wait_log=none)
spawned k6 pid=466717
wait result: pid=466717 exit_code=0  (0 == clean natural completion)
elapsed spawn -> natural exit: 1207.7 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
  running (1.0s), 1/3 VUs, 8 complete and 0 interrupted iterations
  running (1.2s), 0/3 VUs, 9 complete and 0 interrupted iterations
  time="2026-07-15T03:59:14Z" level=debug msg="Everything has finished, exiting k6 normally!"
--- state markers --- Graceful stop log count=3 ; Hard stop log count=0
--- teardown marker (console) ---
--- iteration progress (final two states) ---
  running (1.0s), 1/3 VUs, 8 complete and 0 interrupted iterations
  running (1.2s), 0/3 VUs, 9 complete and 0 interrupted iterations
  alive_after_wait=no
===== end natural_stage_end run 1: exit=0 =====

===== CONDITION natural_stage_end — run 2/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report natural.js   (first=none second=none wait_log=none)
spawned k6 pid=466820
wait result: pid=466820 exit_code=0  (0 == clean natural completion)
elapsed spawn -> natural exit: 1215.3 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
  running (1.0s), 1/3 VUs, 8 complete and 0 interrupted iterations
  running (1.2s), 0/3 VUs, 9 complete and 0 interrupted iterations
  time="2026-07-15T03:59:15Z" level=debug msg="Everything has finished, exiting k6 normally!"
--- state markers --- Graceful stop log count=3 ; Hard stop log count=0
--- teardown marker (console) ---
--- iteration progress (final two states) ---
  running (1.0s), 1/3 VUs, 8 complete and 0 interrupted iterations
  running (1.2s), 0/3 VUs, 9 complete and 0 interrupted iterations
  alive_after_wait=no
===== end natural_stage_end run 2: exit=0 =====

===== CONDITION natural_forced_grace — run 1/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report natural_forced.js   (first=none second=none wait_log=none)
spawned k6 pid=466924
wait result: pid=466924 exit_code=0  (0 == clean natural completion)
elapsed spawn -> natural exit: 2579.3 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
  running (1.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
  running (2.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
  running (2.5s), 0/3 VUs, 3 complete and 0 interrupted iterations
  time="2026-07-15T03:59:17Z" level=debug msg="Everything has finished, exiting k6 normally!"
--- state markers --- Graceful stop log count=3 ; Hard stop log count=0
--- teardown marker (console) ---
--- iteration progress (final two states) ---
  running (2.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
  running (2.5s), 0/3 VUs, 3 complete and 0 interrupted iterations
  alive_after_wait=no
===== end natural_forced_grace run 1: exit=0 =====

===== CONDITION natural_forced_grace — run 2/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report natural_forced.js   (first=none second=none wait_log=none)
spawned k6 pid=467099
wait result: pid=467099 exit_code=0  (0 == clean natural completion)
elapsed spawn -> natural exit: 2538.4 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
  running (1.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
  running (2.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
  running (2.5s), 0/3 VUs, 3 complete and 0 interrupted iterations
  time="2026-07-15T03:59:20Z" level=debug msg="Everything has finished, exiting k6 normally!"
--- state markers --- Graceful stop log count=3 ; Hard stop log count=0
--- teardown marker (console) ---
--- iteration progress (final two states) ---
  running (2.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
  running (2.5s), 0/3 VUs, 3 complete and 0 interrupted iterations
  alive_after_wait=no
===== end natural_forced_grace run 2: exit=0 =====

===== CONDITION C1_sleep_singleSIGINT — run 1/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report sleep_iter.js   (first=2 second=none wait_log=none)
spawned k6 pid=467273
first  SIGINT: sent_at_mono=1784087962.505132375 kill_status=0 alive_before=yes
wait result: pid=467273 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 7.1 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
  running (01.0s), 4/5 VUs, 0 complete and 0 interrupted iterations
  time="2026-07-15T03:59:22Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
  time="2026-07-15T03:59:22Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
  time="2026-07-15T03:59:22Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
  running (02.0s), 0/5 VUs, 0 complete and 5 interrupted iterations
  time="2026-07-15T03:59:22Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
--- state markers --- Graceful stop log count=0 ; Hard stop log count=0
--- teardown marker (console) ---
--- iteration progress (final two states) ---
  running (01.0s), 4/5 VUs, 0 complete and 0 interrupted iterations
  running (02.0s), 0/5 VUs, 0 complete and 5 interrupted iterations
  alive_after_wait=no
===== end C1_sleep_singleSIGINT run 1: exit=105 =====

===== CONDITION C1_sleep_singleSIGINT — run 2/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report sleep_iter.js   (first=2 second=none wait_log=none)
spawned k6 pid=467310
first  SIGINT: sent_at_mono=1784087964.535946015 kill_status=0 alive_before=yes
wait result: pid=467310 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 6.9 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
  running (01.0s), 4/5 VUs, 0 complete and 0 interrupted iterations
  time="2026-07-15T03:59:24Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
  time="2026-07-15T03:59:24Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
  time="2026-07-15T03:59:24Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
  running (02.0s), 0/5 VUs, 0 complete and 5 interrupted iterations
  time="2026-07-15T03:59:24Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
--- state markers --- Graceful stop log count=0 ; Hard stop log count=0
--- teardown marker (console) ---
--- iteration progress (final two states) ---
  running (01.0s), 4/5 VUs, 0 complete and 0 interrupted iterations
  running (02.0s), 0/5 VUs, 0 complete and 5 interrupted iterations
  alive_after_wait=no
===== end C1_sleep_singleSIGINT run 2: exit=105 =====

===== CONDITION C2_busy_singleSIGINT — run 1/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report busy_iter.js   (first=2.5 second=none wait_log=none)
spawned k6 pid=467347
first  SIGINT: sent_at_mono=1784087967.069693630 kill_status=0 alive_before=yes
wait result: pid=467347 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 7.8 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
  running (01.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
  running (02.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
  time="2026-07-15T03:59:27Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
  time="2026-07-15T03:59:27Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
  time="2026-07-15T03:59:27Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
  running (02.5s), 0/3 VUs, 0 complete and 3 interrupted iterations
--- state markers --- Graceful stop log count=0 ; Hard stop log count=0
--- teardown marker (console) ---
--- iteration progress (final two states) ---
  running (02.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
  running (02.5s), 0/3 VUs, 0 complete and 3 interrupted iterations
  alive_after_wait=no
===== end C2_busy_singleSIGINT run 1: exit=105 =====

===== CONDITION C2_busy_singleSIGINT — run 2/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report busy_iter.js   (first=2.5 second=none wait_log=none)
spawned k6 pid=467405
first  SIGINT: sent_at_mono=1784087969.602845462 kill_status=0 alive_before=yes
wait result: pid=467405 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 7.6 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
  running (01.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
  running (02.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
  time="2026-07-15T03:59:29Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
  time="2026-07-15T03:59:29Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
  time="2026-07-15T03:59:29Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
  running (02.5s), 0/3 VUs, 0 complete and 3 interrupted iterations
--- state markers --- Graceful stop log count=0 ; Hard stop log count=0
--- teardown marker (console) ---
--- iteration progress (final two states) ---
  running (02.0s), 3/3 VUs, 0 complete and 0 interrupted iterations
  running (02.5s), 0/3 VUs, 0 complete and 3 interrupted iterations
  alive_after_wait=no
===== end C2_busy_singleSIGINT run 2: exit=105 =====

===== CONDITION near_transition_SIGINT — run 1/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report rampdown.js   (first=1 second=none wait_log=none)
spawned k6 pid=467463
first  SIGINT: sent_at_mono=1784087970.636082302 kill_status=0 alive_before=yes
wait result: pid=467463 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 7.7 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
  time="2026-07-15T03:59:30Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
  time="2026-07-15T03:59:30Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
  time="2026-07-15T03:59:30Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
  running (01.0s), 0/6 VUs, 0 complete and 5 interrupted iterations
  time="2026-07-15T03:59:30Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
  time="2026-07-15T03:59:30Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
--- state markers --- Graceful stop log count=0 ; Hard stop log count=0
--- teardown marker (console) ---
--- iteration progress (final two states) ---
  running (01.0s), 0/6 VUs, 0 complete and 5 interrupted iterations
  alive_after_wait=no
===== end near_transition_SIGINT run 1: exit=105 =====

===== CONDITION near_transition_SIGINT — run 2/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report rampdown.js   (first=1 second=none wait_log=none)
spawned k6 pid=467501
first  SIGINT: sent_at_mono=1784087971.669550244 kill_status=0 alive_before=yes
wait result: pid=467501 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 6.7 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
  time="2026-07-15T03:59:31Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
  time="2026-07-15T03:59:31Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
  time="2026-07-15T03:59:31Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
  running (01.0s), 0/6 VUs, 0 complete and 5 interrupted iterations
  time="2026-07-15T03:59:31Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
  time="2026-07-15T03:59:31Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
--- state markers --- Graceful stop log count=0 ; Hard stop log count=0
--- teardown marker (console) ---
--- iteration progress (final two states) ---
  running (01.0s), 0/6 VUs, 0 complete and 5 interrupted iterations
  alive_after_wait=no
===== end near_transition_SIGINT run 2: exit=105 =====

===== CONDITION graceful_state_SIGINT — run 1/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report rampdown.js   (first=0 second=none wait_log=Graceful stop)
spawned k6 pid=467540
trigger 'Graceful stop' observed_before_signal=yes after 1.10s
first  SIGINT: sent_at_mono=1784087972.934658535 kill_status=0 alive_before=yes
wait result: pid=467540 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 7.4 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
  running (01.0s), 5/6 VUs, 0 complete and 0 interrupted iterations
  time="2026-07-15T03:59:32Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
  time="2026-07-15T03:59:32Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
  time="2026-07-15T03:59:32Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
  running (01.2s), 0/6 VUs, 0 complete and 6 interrupted iterations
  time="2026-07-15T03:59:32Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
--- state markers --- Graceful stop log count=1 ; Hard stop log count=0
--- teardown marker (console) ---
--- iteration progress (final two states) ---
  running (01.0s), 5/6 VUs, 0 complete and 0 interrupted iterations
  running (01.2s), 0/6 VUs, 0 complete and 6 interrupted iterations
  alive_after_wait=no
===== end graceful_state_SIGINT run 1: exit=105 =====

===== CONDITION graceful_state_SIGINT — run 2/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report rampdown.js   (first=0 second=none wait_log=Graceful stop)
spawned k6 pid=467669
trigger 'Graceful stop' observed_before_signal=yes after 1.10s
first  SIGINT: sent_at_mono=1784087974.204903951 kill_status=0 alive_before=yes
wait result: pid=467669 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 7.5 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
  running (01.0s), 5/6 VUs, 0 complete and 0 interrupted iterations
  time="2026-07-15T03:59:34Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
  time="2026-07-15T03:59:34Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
  time="2026-07-15T03:59:34Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
  running (01.2s), 0/6 VUs, 0 complete and 6 interrupted iterations
  time="2026-07-15T03:59:34Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
--- state markers --- Graceful stop log count=1 ; Hard stop log count=0
--- teardown marker (console) ---
--- iteration progress (final two states) ---
  running (01.0s), 5/6 VUs, 0 complete and 0 interrupted iterations
  running (01.2s), 0/6 VUs, 0 complete and 6 interrupted iterations
  alive_after_wait=no
===== end graceful_state_SIGINT run 2: exit=105 =====

===== CONDITION C3_teardown_singleSIGINT — run 1/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report teardown_busy.js   (first=2 second=none wait_log=none)
spawned k6 pid=467796
first  SIGINT: sent_at_mono=1784087976.238119822 kill_status=0 alive_before=yes
wait result: pid=467796 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 3008.9 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
  running (01.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
  time="2026-07-15T03:59:36Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
  running (02.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
  running (03.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
  running (04.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
  time="2026-07-15T03:59:39Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
--- state markers --- Graceful stop log count=0 ; Hard stop log count=0
--- teardown marker (console) ---
  time="2026-07-15T03:59:36Z" level=info msg=TEARDOWN_START source=console
  time="2026-07-15T03:59:39Z" level=info msg=TEARDOWN_END source=console
--- iteration progress (final two states) ---
  running (04.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
  running (05.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
  alive_after_wait=no
===== end C3_teardown_singleSIGINT run 1: exit=105 =====

===== CONDITION C3_teardown_singleSIGINT — run 2/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report teardown_busy.js   (first=2 second=none wait_log=none)
spawned k6 pid=467852
first  SIGINT: sent_at_mono=1784087981.272714868 kill_status=0 alive_before=yes
wait result: pid=467852 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 3008.1 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
  running (01.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
  time="2026-07-15T03:59:41Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
  running (02.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
  running (03.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
  running (04.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
  time="2026-07-15T03:59:44Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
--- state markers --- Graceful stop log count=0 ; Hard stop log count=0
--- teardown marker (console) ---
  time="2026-07-15T03:59:41Z" level=info msg=TEARDOWN_START source=console
  time="2026-07-15T03:59:44Z" level=info msg=TEARDOWN_END source=console
--- iteration progress (final two states) ---
  running (04.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
  running (05.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
  alive_after_wait=no
===== end C3_teardown_singleSIGINT run 2: exit=105 =====

===== CONDITION C4_teardown_doubleSIGINT — run 1/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report teardown_busy.js   (first=2 second=0.6 wait_log=none)
spawned k6 pid=467910
first  SIGINT: sent_at_mono=1784087986.309258274 kill_status=0 alive_before=yes
second SIGINT: sent_at_mono=1784087986.915993033 kill_status=0 alive_before=yes
wait result: pid=467910 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 614.6 ms
elapsed second_SIGINT -> process_exit: 7.8 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
  running (01.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
  time="2026-07-15T03:59:46Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
  running (02.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
  time="2026-07-15T03:59:46Z" level=error msg="Aborting k6 in response to signal" sig=interrupt
--- state markers --- Graceful stop log count=0 ; Hard stop log count=0
--- teardown marker (console) ---
  time="2026-07-15T03:59:46Z" level=info msg=TEARDOWN_START source=console
--- iteration progress (final two states) ---
  running (01.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
  running (02.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
  alive_after_wait=no
===== end C4_teardown_doubleSIGINT run 1: exit=105 =====

===== CONDITION C4_teardown_doubleSIGINT — run 2/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report teardown_busy.js   (first=2 second=0.6 wait_log=none)
spawned k6 pid=467959
first  SIGINT: sent_at_mono=1784087988.948519915 kill_status=0 alive_before=yes
second SIGINT: sent_at_mono=1784087989.555541354 kill_status=0 alive_before=yes
wait result: pid=467959 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 614.4 ms
elapsed second_SIGINT -> process_exit: 7.4 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
  running (01.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
  time="2026-07-15T03:59:48Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
  running (02.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
  time="2026-07-15T03:59:49Z" level=error msg="Aborting k6 in response to signal" sig=interrupt
--- state markers --- Graceful stop log count=0 ; Hard stop log count=0
--- teardown marker (console) ---
  time="2026-07-15T03:59:48Z" level=info msg=TEARDOWN_START source=console
--- iteration progress (final two states) ---
  running (01.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
  running (02.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
  alive_after_wait=no
===== end C4_teardown_doubleSIGINT run 2: exit=105 =====

===== CONDITION rapid_double_SIGINT — run 1/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report teardown_busy.js   (first=2 second=0.01 wait_log=none)
spawned k6 pid=468010
first  SIGINT: sent_at_mono=1784087991.591862132 kill_status=0 alive_before=yes
second SIGINT: sent_at_mono=1784087991.608255979 kill_status=0 alive_before=yes
wait result: pid=468010 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 23.1 ms
elapsed second_SIGINT -> process_exit: 6.7 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
  running (01.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
  time="2026-07-15T03:59:51Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
  time="2026-07-15T03:59:51Z" level=error msg="Aborting k6 in response to signal" sig=interrupt
--- state markers --- Graceful stop log count=0 ; Hard stop log count=0
--- teardown marker (console) ---
  time="2026-07-15T03:59:51Z" level=info msg=TEARDOWN_START source=console
--- iteration progress (final two states) ---
  running (01.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
  alive_after_wait=no
===== end rapid_double_SIGINT run 1: exit=105 =====

===== CONDITION rapid_double_SIGINT — run 2/2 =====
$ /tmp/k6 run --verbose --no-summary --no-usage-report teardown_busy.js   (first=2 second=0.01 wait_log=none)
spawned k6 pid=468050
first  SIGINT: sent_at_mono=1784087993.640998933 kill_status=0 alive_before=yes
second SIGINT: sent_at_mono=1784087993.657432379 kill_status=0 alive_before=yes
wait result: pid=468050 exit_code=105  (105 == ExternalAbort)
elapsed first_SIGINT  -> process_exit: 24.7 ms
elapsed second_SIGINT -> process_exit: 8.2 ms
--- signal-relevant log lines (k6 stderr, --verbose) ---
  running (01.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
  time="2026-07-15T03:59:53Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
  time="2026-07-15T03:59:53Z" level=error msg="Aborting k6 in response to signal" sig=interrupt
--- state markers --- Graceful stop log count=0 ; Hard stop log count=0
--- teardown marker (console) ---
  time="2026-07-15T03:59:53Z" level=info msg=TEARDOWN_START source=console
--- iteration progress (final two states) ---
  running (01.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
  alive_after_wait=no
===== end rapid_double_SIGINT run 2: exit=105 =====

################ SYMPTOM C HARNESS COMPLETE (RUNS=2) ################
```

**Two-run summary (all nine conditions; both runs consistent).**

| Condition | Signal | Exit (run1,run2) | Key latency (run1,run2) | State markers |
|---|---|---:|---|---|
| `natural_stage_end` | none | 0, 0 | runtime 1207.7, 1215.3 ms | Graceful stop=3; Hard stop=0 |
| `natural_forced_grace` | none | 0, 0 | runtime 2579.3, 2538.4 ms | Graceful stop=3; Hard stop=0 |
| `C1_sleep_singleSIGINT` | 1×SIGINT | 105, 105 | 1st→exit 7.1, 6.9 ms | 5 interrupted iters |
| `C2_busy_singleSIGINT` | 1×SIGINT | 105, 105 | 1st→exit 7.8, 7.6 ms | 3 interrupted iters (busy loop still interrupted) |
| `near_transition_SIGINT` | 1×SIGINT | 105, 105 | 1st→exit 7.7, 6.7 ms | signal at stage boundary |
| `graceful_state_SIGINT` | 1×SIGINT after `Graceful stop` log | 105, 105 | 1st→exit 7.4, 7.5 ms | `Graceful stop` observed before signal = yes |
| `C3_teardown_singleSIGINT` | 1×SIGINT | 105, 105 | 1st→exit 3008.9, 3008.1 ms | `TEARDOWN_START→TEARDOWN_END` (ran to completion) |
| `C4_teardown_doubleSIGINT` | 2×SIGINT (~0.6 s apart) | 105, 105 | 2nd→exit 7.8, 7.4 ms | `TEARDOWN_START` only (truncated); "Aborting…" |
| `rapid_double_SIGINT` | 2×SIGINT (~10 ms apart) | 105, 105 | 2nd→exit 6.7, 8.2 ms | `TEARDOWN_START` only (truncated); immediate hard stop |

**Reading the nine conditions (both runs each are consistent).**

- **`natural_stage_end` / `natural_forced_grace` (no signal — the baseline).** With no Ctrl+C the run ends
  cleanly (**exit 0**): iterations complete (`… complete and 0 interrupted`), the debug log ends with
  `Everything has finished, exiting k6 normally!`, and — when VUs are still mid-iteration at a down-stage
  boundary — the scheduled-target drop emits ordinary `Graceful stop` logs (count **3**, `Hard stop=0`)
  while `gracefulRampDown` lets those VUs finish. This is the reference against which the signalled cases
  are compared: a clean stage end is bounded by the stage/graceful window, **not** by `gracefulStop`.
- **C1 (sleeping iterations, single SIGINT):** exit **105** in **7.1 ms / 6.9 ms**; the progress line goes
  from `4/5 VUs, 0 … interrupted` to `0/5 VUs, … 5 interrupted iterations`. Only the graceful "Stopping…"
  (Debug) appears. Interruptible sleeps stop promptly.
- **C2 (CPU-busy, non-yielding iterations, single SIGINT):** exit **105** in **7.8 ms / 7.6 ms**;
  `3 interrupted iterations`. **Even a busy JS loop is interrupted** — k6 cancels the iteration context and
  the VM stops — so ordinary VU iterations do not overrun `gracefulStop`.
- **`near_transition_SIGINT` (SIGINT at a stage boundary):** exit **105** in **7.7 ms / 6.7 ms**. Delivering
  the signal exactly at the up→down transition changes nothing about the latency — the abort still cancels
  iteration contexts immediately. (Edge timing covered.)
- **`graceful_state_SIGINT` (SIGINT *after* an actual `Graceful stop` state log):** the harness waits until
  a real `Graceful stop` line (`lib/executor/vu_handle.go:161`) is observed (`observed_before_signal=yes`) —
  i.e. VUs are already in `toGracefulStop` — *then* sends the SIGINT. Exit **105** in **7.4 ms / 7.5 ms**:
  VUs sitting in the graceful-ramp-down state are cancelled just as promptly; the graceful window does not
  delay the abort.
- **C3 (script with a ~3 s busy `teardown()`, single SIGINT):** exit **105** in **3008.9 ms / 3008.1 ms**.
  The VU iterations are interrupted immediately, but `TEARDOWN_START → TEARDOWN_END` then runs to
  **completion** during the graceful abort. This is the demonstrated mechanism behind "keeps running longer
  than expected": it is **graceful-phase work**, not a per-VU iteration overrunning `gracefulStop`.
- **C4 (same teardown script, double SIGINT ~0.6 s apart):** the first SIGINT begins the graceful "Stopping…"
  and `TEARDOWN_START`; the second SIGINT triggers "Aborting k6 in response to signal" (Error, `onHardStop`)
  and an immediate `OSExit(105)`. The second-SIGINT→exit latency is **7.8 ms / 7.4 ms** and `TEARDOWN_END`
  is **absent** — the hard stop cut teardown short. This is the by-design escalation escape hatch (the
  `cmd/common.go:116-117` comment references k6 issue #971).
- **`rapid_double_SIGINT` (two SIGINTs ~10 ms apart):** even with the two signals nearly coincident, the
  second one still lands during the teardown and escalates to an immediate hard stop: first-SIGINT→exit
  **23.1 ms / 24.7 ms**, second-SIGINT→exit **6.7 ms / 8.2 ms**, `TEARDOWN_END` absent. The escalation is
  robust to signal spacing.

**Demonstrated mechanism vs. the user's diagnosis.** What is *demonstrated* is: (a) VU iterations,
including CPU-busy ones, are interrupted in ~7–8 ms on a single Ctrl+C; (b) a single Ctrl+C then runs
graceful-phase work such as `teardown()` to completion; (c) a second Ctrl+C forces an immediate hard
stop. The user's literal wording — *VUs keep running longer than `gracefulStop`* — was **not reproduced**
as a VU/`gracefulStop` violation on the canonical path. **(INFERRED)** the user most likely observed
graceful-phase work (like `teardown()`), or a long non-yielding native/host call inside an iteration
that does not observe context cancellation, rather than the `ramping-vus` scheduler letting VU iterations
exceed `gracefulStop`.


### 7-D. Symptom D — one instance consistently shows more VUs; summed across instances they exceed the configured maximum

**Direct answer:** The imbalance is **deterministic striping**, not a race. With a **shared**
`--execution-segment-sequence`, three synchronized instances split a global max of 11 as
`vus_max = [4, 4, 3]` (canonical JSON metric, §7-D.2) — the **two earliest** segments (`0:1/3` and
`1/3:2/3`) **tie** at the higher count 4 while `2/3:1` holds 3, because the remainder `11 % 3 = 2` is
handed to the two lowest-offset segments — and the per-instance counts **sum to exactly 11**, never
exceeding it. (More generally the number of "higher" instances equals the remainder: at a global max of
1, `1 % 3 = 1`, so only the single earliest segment is higher — `[1, 0, 0]` — which is the special case
where the user's literal "one instance" holds.) The sum **exceeds** the configured maximum only when the
instances are run **without** a shared sequence: each instance then fills its *own* segment sequence and
rounds up independently, giving `[4, 4, 4]` (all three tied), sum **12 > 11**. Both outcomes are fully
determined by the striping arithmetic (the same test yields the same split every time) — there is no
randomness and no race.

**Why specific (tied) instances are consistently higher (source trace).** The `--execution-segment` /
`--execution-segment-sequence` flags `cmd/options.go:31-32` feed `NewExecutionTuple`
`lib/execution_segment.go:723`, which calls `GetFilledExecutionSegmentSequence`
`lib/execution_segment.go:445`. Scaling a global count for a segment uses the striped
`ExecutionSegmentSequenceWrapper.ScaleInt64` `lib/execution_segment.go:580`: it computes the floor share
`result = (value / lcd) * len(offsets)` `lib/execution_segment.go:583` and then, in the remainder loop
`for gi, i := 0, start; i < value%lcd; …` `lib/execution_segment.go:584-586`, adds **one** extra to every
segment whose striped `start` position is below `value % lcd`. Those are precisely the **earliest-offset**
segments, so the surplus lands on them deterministically — and there are exactly `value % lcd` of them.
For a global max of 11 (`11 % 3 = 2`) that is the **two** earliest segments (`0:1/3`, `1/3:2/3`), which
therefore **tie** at 4; for a global max of 1 (`1 % 3 = 1`) it is only the single earliest segment
(`0:1/3`), giving `[1, 0, 0]`. (`ExecutionTuple.ScaleInt64` `lib/execution_segment.go:734` short-circuits
to the raw value when the sequence has a single segment `lib/execution_segment.go:735-737`, and otherwise
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
    blitzy_adhoc_test_probe_test.go:675: Symptom D (shared sequence "0,1/3,2/3,1") — deterministic striped ScaleInt64 [execution_segment.go:580]:
    blitzy_adhoc_test_probe_test.go:676:       T | 0:1/3     | 1/3:2/3     | 2/3:1     | sum | sum vs T
    blitzy_adhoc_test_probe_test.go:694:       1 |         1 |           0 |         0 |   1 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:       2 |         1 |           1 |         0 |   2 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:       3 |         1 |           1 |         1 |   3 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:       4 |         2 |           1 |         1 |   4 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:       5 |         2 |           2 |         1 |   5 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:       6 |         2 |           2 |         2 |   6 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:       7 |         3 |           2 |         2 |   7 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:       8 |         3 |           3 |         2 |   8 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:       9 |         3 |           3 |         3 |   9 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:      10 |         4 |           3 |         3 |  10 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:      11 |         4 |           4 |         3 |  11 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:      12 |         4 |           4 |         4 |  12 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:696: Symptom D (shared sequence) — OVER rows: 0, UNDER rows: 0 (both must be 0; sum==T always)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:750: Symptom D (independent) — segment 0:1/3    filled sequence: 0,1/3,1
    blitzy_adhoc_test_probe_test.go:750: Symptom D (independent) — segment 1/3:2/3  filled sequence: 0,1/3,2/3,1
    blitzy_adhoc_test_probe_test.go:750: Symptom D (independent) — segment 2/3:1    filled sequence: 0,2/3,1
    blitzy_adhoc_test_probe_test.go:753: Symptom D (NO shared sequence, each instance independent) — full under/equal/over table:
    blitzy_adhoc_test_probe_test.go:754:       T | 0:1/3     | 1/3:2/3     | 2/3:1     | sum | sum vs global max T
    blitzy_adhoc_test_probe_test.go:774:       1 |         0 |           0 |         0 |   0 | UNDER (sum-T=-1)
    blitzy_adhoc_test_probe_test.go:774:       2 |         1 |           1 |         1 |   3 | OVER (sum-T=+1)
    blitzy_adhoc_test_probe_test.go:774:       3 |         1 |           1 |         1 |   3 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:774:       4 |         1 |           1 |         1 |   3 | UNDER (sum-T=-1)
    blitzy_adhoc_test_probe_test.go:774:       5 |         2 |           2 |         2 |   6 | OVER (sum-T=+1)
    blitzy_adhoc_test_probe_test.go:774:       6 |         2 |           2 |         2 |   6 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:774:       7 |         2 |           2 |         2 |   6 | UNDER (sum-T=-1)
    blitzy_adhoc_test_probe_test.go:774:       8 |         3 |           3 |         3 |   9 | OVER (sum-T=+1)
    blitzy_adhoc_test_probe_test.go:774:       9 |         3 |           3 |         3 |   9 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:774:      10 |         3 |           3 |         3 |   9 | UNDER (sum-T=-1)
    blitzy_adhoc_test_probe_test.go:774:      11 |         4 |           4 |         4 |  12 | OVER (sum-T=+1)
    blitzy_adhoc_test_probe_test.go:774:      12 |         4 |           4 |         4 |  12 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:776: Symptom D (independent) — classification across T=1..12: OVER=4 EQUAL=4 UNDER=4
    blitzy_adhoc_test_probe_test.go:777: Symptom D (independent) — OVER rows are real: without a shared --execution-segment-sequence the per-instance shares can sum ABOVE the configured maximum (deterministic per T, still never random).
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:718: Symptom D (shared sequence) — recomputed 5×: identical every time (deterministic, coordination-free).
--- PASS: TestBlitzyProbeIndependentUnderEqualOver (0.00s)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:728: Symptom D (shared sequence) — at T=1 the surplus VU goes to segment index 0 (0:1/3): [1 0 0] => one instance is CONSISTENTLY higher (deterministic), never random.
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
    blitzy_adhoc_test_probe_test.go:750: Symptom D (independent) — segment 0:1/3    filled sequence: 0,1/3,1
    blitzy_adhoc_test_probe_test.go:750: Symptom D (independent) — segment 1/3:2/3  filled sequence: 0,1/3,2/3,1
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:675: Symptom D (shared sequence "0,1/3,2/3,1") — deterministic striped ScaleInt64 [execution_segment.go:580]:
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:750: Symptom D (independent) — segment 2/3:1    filled sequence: 0,2/3,1
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:676:       T | 0:1/3     | 1/3:2/3     | 2/3:1     | sum | sum vs T
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:753: Symptom D (NO shared sequence, each instance independent) — full under/equal/over table:
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:694:       1 |         1 |           0 |         0 |   1 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:       2 |         1 |           1 |         0 |   2 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:       3 |         1 |           1 |         1 |   3 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:       4 |         2 |           1 |         1 |   4 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:754:       T | 0:1/3     | 1/3:2/3     | 2/3:1     | sum | sum vs global max T
    blitzy_adhoc_test_probe_test.go:774:       1 |         0 |           0 |         0 |   0 | UNDER (sum-T=-1)
    blitzy_adhoc_test_probe_test.go:774:       2 |         1 |           1 |         1 |   3 | OVER (sum-T=+1)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:694:       5 |         2 |           2 |         1 |   5 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:774:       3 |         1 |           1 |         1 |   3 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:774:       4 |         1 |           1 |         1 |   3 | UNDER (sum-T=-1)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:694:       6 |         2 |           2 |         2 |   6 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:       7 |         3 |           2 |         2 |   7 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:       8 |         3 |           3 |         2 |   8 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:774:       5 |         2 |           2 |         2 |   6 | OVER (sum-T=+1)
    blitzy_adhoc_test_probe_test.go:774:       6 |         2 |           2 |         2 |   6 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:774:       7 |         2 |           2 |         2 |   6 | UNDER (sum-T=-1)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:694:       9 |         3 |           3 |         3 |   9 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:      10 |         4 |           3 |         3 |  10 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:      11 |         4 |           4 |         3 |  11 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:774:       8 |         3 |           3 |         3 |   9 | OVER (sum-T=+1)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:694:      12 |         4 |           4 |         4 |  12 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:774:       9 |         3 |           3 |         3 |   9 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:774:      10 |         3 |           3 |         3 |   9 | UNDER (sum-T=-1)
    blitzy_adhoc_test_probe_test.go:774:      11 |         4 |           4 |         4 |  12 | OVER (sum-T=+1)
    blitzy_adhoc_test_probe_test.go:774:      12 |         4 |           4 |         4 |  12 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:696: Symptom D (shared sequence) — OVER rows: 0, UNDER rows: 0 (both must be 0; sum==T always)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:776: Symptom D (independent) — classification across T=1..12: OVER=4 EQUAL=4 UNDER=4
    blitzy_adhoc_test_probe_test.go:777: Symptom D (independent) — OVER rows are real: without a shared --execution-segment-sequence the per-instance shares can sum ABOVE the configured maximum (deterministic per T, still never random).
--- PASS: TestBlitzyProbeIndependentUnderEqualOver (0.00s)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:718: Symptom D (shared sequence) — recomputed 5×: identical every time (deterministic, coordination-free).
    blitzy_adhoc_test_probe_test.go:728: Symptom D (shared sequence) — at T=1 the surplus VU goes to segment index 0 (0:1/3): [1 0 0] => one instance is CONSISTENTLY higher (deterministic), never random.
--- PASS: TestBlitzyProbeSegmentScaleArithmetic (0.00s)
=== RUN   TestBlitzyProbeSegmentScaleArithmetic
=== PAUSE TestBlitzyProbeSegmentScaleArithmetic
=== RUN   TestBlitzyProbeIndependentUnderEqualOver
=== PAUSE TestBlitzyProbeIndependentUnderEqualOver
=== CONT  TestBlitzyProbeSegmentScaleArithmetic
=== CONT  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:750: Symptom D (independent) — segment 0:1/3    filled sequence: 0,1/3,1
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:675: Symptom D (shared sequence "0,1/3,2/3,1") — deterministic striped ScaleInt64 [execution_segment.go:580]:
    blitzy_adhoc_test_probe_test.go:676:       T | 0:1/3     | 1/3:2/3     | 2/3:1     | sum | sum vs T
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:750: Symptom D (independent) — segment 1/3:2/3  filled sequence: 0,1/3,2/3,1
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:694:       1 |         1 |           0 |         0 |   1 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:       2 |         1 |           1 |         0 |   2 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:750: Symptom D (independent) — segment 2/3:1    filled sequence: 0,2/3,1
    blitzy_adhoc_test_probe_test.go:753: Symptom D (NO shared sequence, each instance independent) — full under/equal/over table:
    blitzy_adhoc_test_probe_test.go:754:       T | 0:1/3     | 1/3:2/3     | 2/3:1     | sum | sum vs global max T
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:694:       3 |         1 |           1 |         1 |   3 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:       4 |         2 |           1 |         1 |   4 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:       5 |         2 |           2 |         1 |   5 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:       6 |         2 |           2 |         2 |   6 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:774:       1 |         0 |           0 |         0 |   0 | UNDER (sum-T=-1)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:694:       7 |         3 |           2 |         2 |   7 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:694:       8 |         3 |           3 |         2 |   8 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:774:       2 |         1 |           1 |         1 |   3 | OVER (sum-T=+1)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:694:       9 |         3 |           3 |         3 |   9 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:774:       3 |         1 |           1 |         1 |   3 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:694:      10 |         4 |           3 |         3 |  10 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:774:       4 |         1 |           1 |         1 |   3 | UNDER (sum-T=-1)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:694:      11 |         4 |           4 |         3 |  11 | EQUAL (sum-T=+0)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:774:       5 |         2 |           2 |         2 |   6 | OVER (sum-T=+1)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:694:      12 |         4 |           4 |         4 |  12 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:696: Symptom D (shared sequence) — OVER rows: 0, UNDER rows: 0 (both must be 0; sum==T always)
=== NAME  TestBlitzyProbeIndependentUnderEqualOver
    blitzy_adhoc_test_probe_test.go:774:       6 |         2 |           2 |         2 |   6 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:774:       7 |         2 |           2 |         2 |   6 | UNDER (sum-T=-1)
    blitzy_adhoc_test_probe_test.go:774:       8 |         3 |           3 |         3 |   9 | OVER (sum-T=+1)
    blitzy_adhoc_test_probe_test.go:774:       9 |         3 |           3 |         3 |   9 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:774:      10 |         3 |           3 |         3 |   9 | UNDER (sum-T=-1)
    blitzy_adhoc_test_probe_test.go:774:      11 |         4 |           4 |         4 |  12 | OVER (sum-T=+1)
    blitzy_adhoc_test_probe_test.go:774:      12 |         4 |           4 |         4 |  12 | EQUAL (sum-T=+0)
    blitzy_adhoc_test_probe_test.go:776: Symptom D (independent) — classification across T=1..12: OVER=4 EQUAL=4 UNDER=4
    blitzy_adhoc_test_probe_test.go:777: Symptom D (independent) — OVER rows are real: without a shared --execution-segment-sequence the per-instance shares can sum ABOVE the configured maximum (deterministic per T, still never random).
--- PASS: TestBlitzyProbeIndependentUnderEqualOver (0.00s)
=== NAME  TestBlitzyProbeSegmentScaleArithmetic
    blitzy_adhoc_test_probe_test.go:718: Symptom D (shared sequence) — recomputed 5×: identical every time (deterministic, coordination-free).
    blitzy_adhoc_test_probe_test.go:728: Symptom D (shared sequence) — at T=1 the surplus VU goes to segment index 0 (0:1/3): [1 0 0] => one instance is CONSISTENTLY higher (deterministic), never random.
--- PASS: TestBlitzyProbeSegmentScaleArithmetic (0.00s)
PASS
ok  	go.k6.io/k6/lib/executor	1.034s
```

#### 7-D.2 Genuine synchronized three-process CLI runs (canonical `k6` binary; ≥2 runs per mode)

To reproduce the user's distributed setup faithfully, three real `k6 run` processes are launched
**simultaneously**, one per segment, sharing a global ramp with a configured maximum of 11 VUs. Each
mode is run twice. Per-instance counts are taken from the **canonical JSON metric stream** each instance
emits via `--out json=<file>`: `vus_max` (the scaled configured max for that segment) and `vus` (the peak
active count), read directly from the metric `Point` records — **not** scraped from stdout progress
lines. The full harness, the JS scenarios, and the JSON parser are embedded in §9.3.

- **SHARED** mode passes the same `--execution-segment-sequence 0,1/3,2/3,1` to all three instances.
- **INDEPENDENT** mode passes only `--execution-segment` to each (no shared sequence).
- **SHARED_LOW** mode is `SHARED` with a global max of **1**, included to demonstrate the remainder-of-1
  case where exactly one instance is higher (`[1, 0, 0]`).

**Commands (as echoed in the transcript):**

```text
# SHARED (per instance)
/tmp/k6 run --execution-segment 0:1/3   --execution-segment-sequence 0,1/3,2/3,1 --out json=<f> seg.js
/tmp/k6 run --execution-segment 1/3:2/3 --execution-segment-sequence 0,1/3,2/3,1 --out json=<f> seg.js
/tmp/k6 run --execution-segment 2/3:1   --execution-segment-sequence 0,1/3,2/3,1 --out json=<f> seg.js
# INDEPENDENT (per instance)
/tmp/k6 run --execution-segment 0:1/3   --out json=<f> seg.js
/tmp/k6 run --execution-segment 1/3:2/3 --out json=<f> seg.js
/tmp/k6 run --execution-segment 2/3:1   --out json=<f> seg.js
# SHARED_LOW (per instance; global max target 1 via seg_low.js)
/tmp/k6 run --execution-segment 0:1/3   --execution-segment-sequence 0,1/3,2/3,1 --out json=<f> seg_low.js
```

```text
################ k6 build under test ################
k6 v0.55.0 (commit/ddc3b0b1d2, go1.23.6, linux/amd64)
workdir: /tmp/k6probe_symD.FZnDUC ; RUNS per mode: 2 ; shared sequence: 0,1/3,2/3,1 ; segments: 0:1/3 1/3:2/3 2/3:1

-------- SHARED triple (gmax=11), SYNCHRONIZED 3-process run 1/2 --------
$ /tmp/k6 run --execution-segment 0:1/3 --execution-segment-sequence 0,1/3,2/3,1 --out json=<f> seg.js
$ /tmp/k6 run --execution-segment 1/3:2/3 --execution-segment-sequence 0,1/3,2/3,1 --out json=<f> seg.js
$ /tmp/k6 run --execution-segment 2/3:1 --execution-segment-sequence 0,1/3,2/3,1 --out json=<f> seg.js
exit codes: seg0=0 seg1=0 seg2=0
  per-instance CANONICAL metrics (JSON --out json; vus_max = scaled configured max, vus = peak active):
    seg0 0:1/3    => vus_max=4  peak_vus=4
    seg1 1/3:2/3  => vus_max=4  peak_vus=4
    seg2 2/3:1    => vus_max=3  peak_vus=3
    SUM of per-instance vus_max = 11  vs configured global max = 11  => EQUAL
  per-instance vus_max = [4, 4, 3]
  maximum per-instance share = 4 ; segment(s) AT the maximum (TIED): ['seg0(0:1/3)', 'seg1(1/3:2/3)']  (2 of 3 instances)
  remainder distribution: 2 surplus VU(s) assigned DETERMINISTICALLY to the EARLIEST-offset segment(s) ['seg0(0:1/3)', 'seg1(1/3:2/3)'] (striped ScaleInt64 remainder loop, execution_segment.go:584-586) => those instance(s) are consistently higher; the rest tie at the floor 3
-------- end SHARED run 1 --------

-------- SHARED triple (gmax=11), SYNCHRONIZED 3-process run 2/2 --------
$ /tmp/k6 run --execution-segment 0:1/3 --execution-segment-sequence 0,1/3,2/3,1 --out json=<f> seg.js
$ /tmp/k6 run --execution-segment 1/3:2/3 --execution-segment-sequence 0,1/3,2/3,1 --out json=<f> seg.js
$ /tmp/k6 run --execution-segment 2/3:1 --execution-segment-sequence 0,1/3,2/3,1 --out json=<f> seg.js
exit codes: seg0=0 seg1=0 seg2=0
  per-instance CANONICAL metrics (JSON --out json; vus_max = scaled configured max, vus = peak active):
    seg0 0:1/3    => vus_max=4  peak_vus=4
    seg1 1/3:2/3  => vus_max=4  peak_vus=4
    seg2 2/3:1    => vus_max=3  peak_vus=3
    SUM of per-instance vus_max = 11  vs configured global max = 11  => EQUAL
  per-instance vus_max = [4, 4, 3]
  maximum per-instance share = 4 ; segment(s) AT the maximum (TIED): ['seg0(0:1/3)', 'seg1(1/3:2/3)']  (2 of 3 instances)
  remainder distribution: 2 surplus VU(s) assigned DETERMINISTICALLY to the EARLIEST-offset segment(s) ['seg0(0:1/3)', 'seg1(1/3:2/3)'] (striped ScaleInt64 remainder loop, execution_segment.go:584-586) => those instance(s) are consistently higher; the rest tie at the floor 3
-------- end SHARED run 2 --------

-------- INDEPENDENT triple (gmax=11), SYNCHRONIZED 3-process run 1/2 --------
$ /tmp/k6 run --execution-segment 0:1/3 --out json=<f> seg.js   (NO --execution-segment-sequence)
$ /tmp/k6 run --execution-segment 1/3:2/3 --out json=<f> seg.js   (NO --execution-segment-sequence)
$ /tmp/k6 run --execution-segment 2/3:1 --out json=<f> seg.js   (NO --execution-segment-sequence)
exit codes: seg0=0 seg1=0 seg2=0
  per-instance CANONICAL metrics (JSON --out json; vus_max = scaled configured max, vus = peak active):
    seg0 0:1/3    => vus_max=4  peak_vus=4
    seg1 1/3:2/3  => vus_max=4  peak_vus=4
    seg2 2/3:1    => vus_max=4  peak_vus=4
    SUM of per-instance vus_max = 12  vs configured global max = 11  => OVER (exceeds configured max!)
  per-instance vus_max = [4, 4, 4]
  maximum per-instance share = 4 ; segment(s) AT the maximum (TIED): ['seg0(0:1/3)', 'seg1(1/3:2/3)', 'seg2(2/3:1)']  (3 of 3 instances)
  remainder distribution: ALL 3 segments tie at 4 — each instance rounds ITS OWN share up (independent per-instance ceiling, no shared floor) => the sum can exceed the global max
-------- end INDEPENDENT run 1 --------

-------- INDEPENDENT triple (gmax=11), SYNCHRONIZED 3-process run 2/2 --------
$ /tmp/k6 run --execution-segment 0:1/3 --out json=<f> seg.js   (NO --execution-segment-sequence)
$ /tmp/k6 run --execution-segment 1/3:2/3 --out json=<f> seg.js   (NO --execution-segment-sequence)
$ /tmp/k6 run --execution-segment 2/3:1 --out json=<f> seg.js   (NO --execution-segment-sequence)
exit codes: seg0=0 seg1=0 seg2=0
  per-instance CANONICAL metrics (JSON --out json; vus_max = scaled configured max, vus = peak active):
    seg0 0:1/3    => vus_max=4  peak_vus=4
    seg1 1/3:2/3  => vus_max=4  peak_vus=4
    seg2 2/3:1    => vus_max=4  peak_vus=4
    SUM of per-instance vus_max = 12  vs configured global max = 11  => OVER (exceeds configured max!)
  per-instance vus_max = [4, 4, 4]
  maximum per-instance share = 4 ; segment(s) AT the maximum (TIED): ['seg0(0:1/3)', 'seg1(1/3:2/3)', 'seg2(2/3:1)']  (3 of 3 instances)
  remainder distribution: ALL 3 segments tie at 4 — each instance rounds ITS OWN share up (independent per-instance ceiling, no shared floor) => the sum can exceed the global max
-------- end INDEPENDENT run 2 --------

-------- SHARED_LOW triple (gmax=1), SYNCHRONIZED 3-process run 1/2 --------
$ /tmp/k6 run --execution-segment 0:1/3 --execution-segment-sequence 0,1/3,2/3,1 --out json=<f> seg_low.js
$ /tmp/k6 run --execution-segment 1/3:2/3 --execution-segment-sequence 0,1/3,2/3,1 --out json=<f> seg_low.js
$ /tmp/k6 run --execution-segment 2/3:1 --execution-segment-sequence 0,1/3,2/3,1 --out json=<f> seg_low.js
exit codes: seg0=0 seg1=0 seg2=0
  per-instance CANONICAL metrics (JSON --out json; vus_max = scaled configured max, vus = peak active):
    seg0 0:1/3    => vus_max=1  peak_vus=1
    seg1 1/3:2/3  => vus_max=0  peak_vus=0
    seg2 2/3:1    => vus_max=0  peak_vus=0
    SUM of per-instance vus_max = 1  vs configured global max = 1  => EQUAL
  per-instance vus_max = [1, 0, 0]
  maximum per-instance share = 1 ; segment(s) AT the maximum (TIED): ['seg0(0:1/3)']  (1 of 3 instances)
  remainder distribution: 1 surplus VU(s) assigned DETERMINISTICALLY to the EARLIEST-offset segment(s) ['seg0(0:1/3)'] (striped ScaleInt64 remainder loop, execution_segment.go:584-586) => those instance(s) are consistently higher; the rest tie at the floor 0
-------- end SHARED_LOW run 1 --------

-------- SHARED_LOW triple (gmax=1), SYNCHRONIZED 3-process run 2/2 --------
$ /tmp/k6 run --execution-segment 0:1/3 --execution-segment-sequence 0,1/3,2/3,1 --out json=<f> seg_low.js
$ /tmp/k6 run --execution-segment 1/3:2/3 --execution-segment-sequence 0,1/3,2/3,1 --out json=<f> seg_low.js
$ /tmp/k6 run --execution-segment 2/3:1 --execution-segment-sequence 0,1/3,2/3,1 --out json=<f> seg_low.js
exit codes: seg0=0 seg1=0 seg2=0
  per-instance CANONICAL metrics (JSON --out json; vus_max = scaled configured max, vus = peak active):
    seg0 0:1/3    => vus_max=1  peak_vus=1
    seg1 1/3:2/3  => vus_max=0  peak_vus=0
    seg2 2/3:1    => vus_max=0  peak_vus=0
    SUM of per-instance vus_max = 1  vs configured global max = 1  => EQUAL
  per-instance vus_max = [1, 0, 0]
  maximum per-instance share = 1 ; segment(s) AT the maximum (TIED): ['seg0(0:1/3)']  (1 of 3 instances)
  remainder distribution: 1 surplus VU(s) assigned DETERMINISTICALLY to the EARLIEST-offset segment(s) ['seg0(0:1/3)'] (striped ScaleInt64 remainder loop, execution_segment.go:584-586) => those instance(s) are consistently higher; the rest tie at the floor 0
-------- end SHARED_LOW run 2 --------

################ SYMPTOM D HARNESS COMPLETE (RUNS=2) ################
```

**Reading the CLI evidence.**

- **SHARED, both runs:** canonical `vus_max = [4, 4, 3]`, sum **= 11 = configured max** (`EQUAL`); the
  **two earliest** segments (`0:1/3`, `1/3:2/3`) **tie** at 4 — the parser labels them the deterministic
  remainder recipients (`2` surplus VUs → the two lowest offsets) — and `2/3:1` holds 3; the sum peaks at
  11 and **never** exceeds it. This is the user's "one instance consistently shows more," refined to *the
  earliest `value % n` instances tie at the top* — and it is correct: it sums to exactly the configured
  max.
- **INDEPENDENT, both runs:** canonical `vus_max = [4, 4, 4]` (**all three tied**), sum **= 12 > 11**
  (`OVER — exceeds configured max`). This reproduces the user's "sum them up and they exceed my
  configured maximum" — and it happens specifically because the instances were **not** given a shared
  `--execution-segment-sequence`, so each rounds *its own* share up independently.
- **SHARED_LOW, both runs:** canonical `vus_max = [1, 0, 0]`, sum **= 1** (`EQUAL`); with remainder
  `1 % 3 = 1`, only the single earliest segment (`0:1/3`) is higher — the special case where the user's
  literal "one instance" is exactly true.

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
genuine synchronized multi-process CLI runs with canonical JSON `vus_max`/`vus` metrics (§7-D.2). The
user's "one instance consistently higher" is expected — refined by the evidence to *the earliest
`value % n` instances tie at the top* (two of three at a global max of 11, `[4, 4, 3]`; a single one at a
global max of 1, `[1, 0, 0]`); "sum exceeds the configured maximum" occurs specifically when instances
lack a shared `--execution-segment-sequence` (`[4, 4, 4]`, all three tied). Neither is a race.


---

## 8. Coverage summary

Every named symptom, question, and sub-item from the request is addressed, with the section holding its
runtime evidence:

| Request item | Verdict | Evidence |
|---|---|---|
| **Question (a)** — race between the two handler goroutines? | **No** — direct handler-ordering trace shows both strategies invoked on one goroutine with `max simultaneous = 1` (they never overlap); graceful tail adds one later call on a separate goroutine after `iterateSteps` returns; `-race` clean ×2 on both suites | §2.1, §4.1, §4.4 |
| **Question (b)** — is the VU buffer leaking? | **No** on the observed paths — buffer fully restored; failure path fails cleanly | §2.2, §6 |
| **Question (c)** — trace both handlers modifying VU state "simultaneously" | Serialized on one per-VU mutex; mixed atomic/mutex model; no unsafe interleaving | §2.3, §5 |
| **Symptom A** — "stuck" VUs (rapid up/down + long `gracefulRampDown`) | Transient, bounded lag (max +2), never permanent (final `0×10`) | §7-A |
| **Symptom B** — scheduled vs. graceful count mismatch | Expected by design; `ceiling >= target` always; `Hard stop=0` observed | §7-B |
| **Symptom C** — VUs outrun `gracefulStop` on ctrl+c | Not reproduced as a VU/`gracefulStop` defect. Real-binary, nine conditions ×2: natural stage-end exits `0` (no signal); a single Ctrl+C interrupts sleeping iterations, CPU-busy iterations, iterations at a stage edge, and iterations already in `toGracefulStop` alike in ~7–8 ms (exit `105`); the long "keeps running" is graceful-phase work (`teardown` runs ~3.0 s to completion during graceful abort, not per-iteration overrun); a 2nd Ctrl+C escalates to an immediate hard stop (~7–8 ms, teardown truncated) | §7-C |
| **Symptom D** — one instance higher; sum exceeds max | Deterministic striping. Canonical JSON `vus_max`: shared `[4,4,3]` sum `=11` (remainder 2 → **two earliest segments tie** at top, not one; at max 1 it is `[1,0,0]`, a single one); independent `[4,4,4]` all tied, sum `=12>11` | §7-D |
| Named component: `maxAllowedVUsHandlerStrategy` `:668` (graceful/max-allowed) | Traced; shrinks ceiling via `hardStop`, grows implicitly | §4.1, §7-B |
| Named component: `scheduledVUsHandlerStrategy` `:679` (scheduled) | Traced; `start`/`gracefulStop` on target change | §4.1, §7-B |
| Named component: per-VU `mutex` `:71`; `start`/`gracefulStop`/`hardStop` | All lock the same mutex; serialize | §5.1 |
| Named states: `stopped`/`starting`/`running`/`toGracefulStop`/`toHardStop` `:17-21` | Used by real name; **all five directly captured** in one channel-gated VU trace (`[stopped starting running toGracefulStop toHardStop]`) | §5.3, §7-A |
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
	"runtime"
	"sort"
	"strconv"
	"strings"
	"sync"
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

// TestBlitzyProbeStateTransitions drives ONE real vuHandle through a deterministic,
// channel-gated sequence and DIRECTLY samples the state at each transition point —
// including the three TRANSIENT states (starting, toGracefulStop, toHardStop), not
// just the stable endpoints. It answers Q(c): all three control methods lock the SAME
// per-VU mutex (start lib/executor/vu_handle.go:116, gracefulStop :148, hardStop :166)
// and the documented transition table (lib/executor/vu_handle.go:24-55) governs every
// move. It also confirms getVU/returnVU are 1:1 at the vuHandle level.
//
// Determinism: the iteration callback (runIter) blocks on an unbuffered `proceed`
// channel, so while a VU is "running" the loop is parked INSIDE runIter and cannot
// advance the state machine. This lets the test call gracefulStop()/hardStop() and read
// the resulting TRANSIENT state BEFORE releasing the iteration. `starting` is captured
// by calling start() BEFORE launching the run-loop goroutine (nothing advances it yet).
func TestBlitzyProbeStateTransitions(t *testing.T) {
	t.Parallel()
	parentCtx, cancelParent := context.WithCancel(context.Background())
	defer cancelParent()

	logHook := testutils.NewLogHook(logrus.DebugLevel)
	testLog := logrus.New()
	testLog.AddHook(logHook)
	testLog.SetOutput(io.Discard)
	testLog.SetLevel(logrus.DebugLevel)
	logEntry := logrus.NewEntry(testLog).WithField("vuNum", 0)

	runner := simpleRunner(func(rctx context.Context, _ *lib.State) error {
		select {
		case <-rctx.Done():
		case <-time.After(10 * time.Millisecond):
		}
		return nil
	})
	require.NoError(t, runner.SetOptions(lib.Options{}))

	var getVUCount, returnVUCount int64
	getVU := func() (lib.InitializedVU, error) {
		atomic.AddInt64(&getVUCount, 1)
		return runner.NewVU(parentCtx, uint64(atomic.LoadInt64(&getVUCount)), 0, nil)
	}
	returnVU := func(_ lib.InitializedVU) {
		atomic.AddInt64(&returnVUCount, 1)
	}

	vh := newStoppedVUHandle(parentCtx, getVU, returnVU, mockNextIterations, &BaseConfig{}, logEntry)

	// Gated iteration callback: hold the loop INSIDE the running state until released,
	// so the test can observe the transient states deterministically.
	entered := make(chan struct{}, 64)
	proceed := make(chan struct{})
	testDone := make(chan struct{})
	defer close(testDone)
	runIter := func(_ context.Context, avu lib.ActiveVU) bool {
		if avu != nil {
			_ = avu.RunOnce()
		}
		entered <- struct{}{} // announce: state is running and one real iteration ran
		select {
		case <-proceed: // test lets this iteration finish
		case <-testDone: // safety: never hang at cleanup
		}
		return true
	}

	type step struct {
		label string
		state string
	}
	var trace []step
	record := func(label string) {
		trace = append(trace, step{label, stateName(readState(vh))})
	}
	waitState := func(want stateType, timeout time.Duration) {
		deadline := time.Now().Add(timeout)
		for time.Now().Before(deadline) {
			if readState(vh) == want {
				return
			}
			time.Sleep(time.Millisecond)
		}
	}

	record("initial")                     // stopped
	require.NoError(t, vh.start())        // stopped -> starting (loop not launched)
	record("start() [loop not launched]") // starting
	go vh.runLoopsIfPossible(runIter)     // starting -> running, enters runIter
	<-entered
	record("loop entered iteration")          // running
	vh.gracefulStop()                         // running -> toGracefulStop (iteration held)
	record("gracefulStop() [iteration held]") // toGracefulStop
	proceed <- struct{}{}                     // release; loop: toGracefulStop -> stopped
	waitState(stopped, 2*time.Second)
	record("after iteration finishes") // stopped
	require.NoError(t, vh.start())     // stopped -> ... -> running
	<-entered
	record("start() again -> loop entered iteration") // running
	vh.hardStop()                                     // running -> toHardStop (iteration held)
	record("hardStop() [iteration held]")             // toHardStop
	proceed <- struct{}{}                             // release; loop: toHardStop -> stopped
	waitState(stopped, 2*time.Second)
	record("after hardStop settles") // stopped
	cancelParent()
	time.Sleep(60 * time.Millisecond)
	record("after cancel()") // stopped

	// wait for async returnVU (VU deactivation) to settle before reading accounting
	deadline := time.Now().Add(2 * time.Second)
	for atomic.LoadInt64(&returnVUCount) < atomic.LoadInt64(&getVUCount) && time.Now().Before(deadline) {
		time.Sleep(time.Millisecond)
	}

	seen := map[string]bool{}
	t.Logf("Q(c)/M2 DIRECT five-state capture (one real vuHandle, channel-gated):")
	for _, st := range trace {
		seen[st.state] = true
		t.Logf("  %-42s -> %s", st.label, st.state)
	}
	var distinct []string
	for _, name := range []string{"stopped", "starting", "running", "toGracefulStop", "toHardStop"} {
		if seen[name] {
			distinct = append(distinct, name)
		}
	}
	t.Logf("distinct states directly observed = %v", distinct)

	entries := logHook.Drain()
	t.Logf("captured vuHandle debug transition lines:")
	for _, e := range entries {
		t.Logf("    level=%s msg=%q vuNum=%v", e.Level, e.Message, e.Data["vuNum"])
	}
	t.Logf("vuHandle acquire/return accounting: getVU=%d returnVU=%d (must be equal, invariant vu_handle.go:62)",
		atomic.LoadInt64(&getVUCount), atomic.LoadInt64(&returnVUCount))

	require.Equal(t, []string{"stopped", "starting", "running", "toGracefulStop", "toHardStop"}, distinct,
		"all five VU states must be directly observed")
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
	t.Logf("Symptom A — OBSERVED signal = GetCurrentlyActiveVUsCount [lib/execution.go:269]; " +
		"'target' column = scheduled PlannedVUs from rawSteps (the scheduledVUsHandlerStrategy goal).")
	t.Logf("Symptom A — INFERRED (from vu_handle.go:19,24-55, not directly sampled): when active > target " +
		"during a down-stage, the surplus VUs are in the transient toGracefulStop state — mid-iteration and " +
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
	t.Logf("Symptom D (independent) — OVER rows are real: without a shared --execution-segment-sequence the " +
		"per-instance shares can sum ABOVE the configured maximum (deterministic per T, still never random).")
	require.Greater(t, over, 0,
		"independent fill must exhibit at least one OVER row (sum above global max) to explain Symptom D")
}

// blitzyGoID extracts the current goroutine's numeric id from the runtime stack
// header ("goroutine <id> [running]:"). Used only to prove how many distinct
// goroutines ever invoke the two handler strategies.
func blitzyGoID() int {
	var buf [64]byte
	n := runtime.Stack(buf[:], false)
	fields := strings.Fields(string(buf[:n]))
	if len(fields) < 2 {
		return -1
	}
	id, _ := strconv.Atoi(fields[1])
	return id
}

// TestBlitzyProbeHandlerOrdering answers Q(a) DIRECTLY. It builds a real
// rampingVUsRunState over the canonical rapid up/down config, wraps the two REAL
// handler strategies (maxAllowedVUsHandlerStrategy lib/executor/ramping_vus.go:668 and
// scheduledVUsHandlerStrategy :679) in a recorder, and drives them through the REAL
// sequencer iterateSteps (:622) followed by the REAL graceful tail
// runRemainingGracefulSteps (:654) launched in its own goroutine exactly as Run does
// (go ... :554). Every handler invocation is timestamped and tagged with its goroutine
// id and phase, so the trace shows: (i) how many goroutines invoke handlers, (ii)
// whether any two handler bodies ever overlap in time, and (iii) that the scheduled
// handler runs ONLY during iterateSteps while the tail invokes ONLY maxAllowed.
func TestBlitzyProbeHandlerOrdering(t *testing.T) {
	t.Parallel()
	config := blitzyRapidConfig()
	runner := simpleRunner(func(_ context.Context, _ *lib.State) error { return nil })
	require.NoError(t, runner.SetOptions(lib.Options{}))
	ctx, cancel, executorIface, _, _ := blitzyProbeExecutor(t, config, "", "", runner)
	defer cancel()
	rv, ok := executorIface.(*RampingVUs)
	require.True(t, ok, "executor must be *RampingVUs")

	maxVUs := lib.GetMaxPlannedVUs(rv.gracefulSteps)
	discard := logrus.New()
	discard.SetOutput(io.Discard)
	logEntry := logrus.NewEntry(discard)

	getVU := func() (lib.InitializedVU, error) { return runner.NewVU(ctx, 1, 0, nil) }
	returnVU := func(_ lib.InitializedVU) {}

	rs := &rampingVUsRunState{
		executor:       rv,
		vuHandles:      make([]*vuHandle, maxVUs),
		maxVUs:         maxVUs,
		activeVUsCount: new(int64),
	}
	for i := uint64(0); i < maxVUs; i++ {
		rs.vuHandles[i] = newStoppedVUHandle(
			ctx, getVU, returnVU, rv.nextIterationCounters,
			&rv.config.BaseConfig, logEntry.WithField("vuNum", i))
	}

	type ev struct {
		seq            int
		goid           int
		phase, handler string
		off            time.Duration
		pv             uint64
	}
	var (
		mu      sync.Mutex
		evs     []ev
		seq     int
		inH     int32
		maxConc int32
		phase   string
	)
	wrap := func(name string, real func(lib.ExecutionStep)) func(lib.ExecutionStep) {
		return func(s lib.ExecutionStep) {
			c := atomic.AddInt32(&inH, 1)
			for {
				m := atomic.LoadInt32(&maxConc)
				if c <= m || atomic.CompareAndSwapInt32(&maxConc, m, c) {
					break
				}
			}
			mu.Lock()
			seq++
			evs = append(evs, ev{seq, blitzyGoID(), phase, name, s.TimeOffset, s.PlannedVUs})
			mu.Unlock()
			real(s)
			atomic.AddInt32(&inH, -1)
		}
	}
	handleMax := wrap("maxAllowed", rs.maxAllowedVUsHandlerStrategy())
	handleSched := wrap("scheduled", rs.scheduledVUsHandlerStrategy())

	rs.started = time.Now()
	phase = "iterateSteps"
	handled := rs.iterateSteps(ctx, handleMax, handleSched)
	phase = "tail"
	done := make(chan struct{})
	go func() {
		rs.runRemainingGracefulSteps(ctx, handleMax, handled)
		close(done)
	}()
	<-done

	// analysis
	goids := map[int]bool{}
	schedInTail, maxInTail, schedTotal, maxTotal := 0, 0, 0, 0
	for _, e := range evs {
		goids[e.goid] = true
		switch e.handler {
		case "scheduled":
			schedTotal++
			if e.phase == "tail" {
				schedInTail++
			}
		case "maxAllowed":
			maxTotal++
			if e.phase == "tail" {
				maxInTail++
			}
		}
	}
	t.Logf("Q(a)/M1 direct handler-invocation trace (rawSteps=%d gracefulSteps=%d maxVUs=%d):",
		len(rv.rawSteps), len(rv.gracefulSteps), maxVUs)
	t.Logf("  seq | phase         | handler    | goroutine | TimeOffset | PlannedVUs")
	for _, e := range evs {
		t.Logf("  %4d | %-12s | %-10s | g%-8d | %10s | %d", e.seq, e.phase, e.handler, e.goid, e.off, e.pv)
	}
	t.Logf("distinct goroutines that invoked ANY handler = %d (expect 1 during iterateSteps + at most 1 for the tail)", len(goids))
	t.Logf("max handler bodies executing simultaneously = %d (expect 1 => no concurrent handler execution)", maxConc)
	t.Logf("scheduled-handler calls during tail = %d (expect 0); maxAllowed calls during tail = %d", schedInTail, maxInTail)
	t.Logf("totals: scheduled=%d maxAllowed=%d ; handledGracefulSteps(returned by iterateSteps)=%d", schedTotal, maxTotal, handled)

	require.Equal(t, int32(1), maxConc, "the two handler strategies must never execute concurrently")
	require.Equal(t, 0, schedInTail, "the scheduled handler must never run during the graceful tail")
	require.Greater(t, maxInTail, 0, "the graceful tail must invoke maxAllowed at least once")
}
```

### 9.2 Symptom C harness (`symptomC_run.sh`) — real SIGINT to the `k6` binary

This Bash harness produced §7-C. It uses `set -u`; **strict `RUNS>=2` integer validation that fails fast
(`exit 2`) before any work** (lines 22–30, addressing the earlier gap where `RUNS=0` or a non-numeric
`RUNS` ran silently); a `mktemp -d` workdir; and `EXIT`/`INT`/`TERM` trap cleanup (`trap cleanup EXIT INT
TERM`, line 81) that kills only the exact spawned PIDs it still owns and removes the workdir (no broad
`pkill`). Each `SIGINT` is delivered to the exact spawned PID with `kill -INT`; the SIGINT→exit latency
is measured on a high-resolution wall clock (`now() { date +%s.%N; }`, line 83) via a busy-spin `kill -0`
poll with a 30 s deadline (`spin_until_exit`, line 87). Child reaping is **bounded** (`reap_bounded`,
line 51): `wait` runs in the main shell and escalates `TERM`→`KILL` if a child overruns, after which the
reaped PID is removed from the cleanup-ownership array (`remove_child`, line 41) so a later PID reuse
cannot be signalled by mistake. The six JavaScript scenarios (`sleep_iter.js`, `busy_iter.js`,
`teardown_busy.js`, `rampdown.js`, `natural.js`, `natural_forced.js`) are created inline by
`cat > … <<'JS'` heredocs (lines 100, 110, 119, 130, 140, 150), so the complete JS source is visible in
the listing below.

```bash
#!/usr/bin/env bash
# symptomC_run.sh — TEMPORARY, EPHEMERAL investigation harness for User Symptom C
# ("when I kill the test early with ctrl+c, some VUs keep running for way longer
#  than gracefulStop should allow").
#
# It launches the REAL k6 binary through its canonical `k6 run` entry point,
# delivers real SIGINT(s) to the exact spawned PID, and measures the
# SIGINT->exit latency with a high-resolution wall clock. Every condition is
# run RUNS times (Rule 1: repeat identical input). Output is fully self-labeled.
#
# Hardening (QA m5/m6):
#   * RUNS is validated as an integer >= 2 BEFORE any work or completion print.
#   * Child waits are bounded with TERM->KILL escalation; the reaped PID is
#     removed from the cleanup ownership array (no PID-reuse hazard). The real
#     exit code is captured by `wait` in the MAIN shell (never a subshell).
set -u

# ---- configuration -------------------------------------------------------
K6="${K6:-/tmp/k6}"          # canonical baseline k6 (v0.55.0, commit ddc3b0b1d2)
RUNS="${RUNS:-2}"

# ---- m5: strict RUNS validation (fail fast, before ANY work) -------------
case "$RUNS" in
  ''|*[!0-9]*)
    echo "ERROR(m5): RUNS must be a non-negative integer, got: '$RUNS'" >&2
    exit 2 ;;
esac
if [ "$RUNS" -lt 2 ]; then
  echo "ERROR(m5): RUNS must be >= 2 (Rule 1 requires repeating identical input), got: $RUNS" >&2
  exit 2
fi

command -v "$K6" >/dev/null 2>&1 || { echo "ERROR: k6 binary not found at $K6" >&2; exit 3; }

WORKDIR="$(mktemp -d /tmp/k6probe_symC.XXXXXX)"

# ---- m6: bounded reap with TERM->KILL escalation + PID removal -----------
CHILDREN=()
REAP_RC=0

remove_child() {  # delete a pid from the CHILDREN ownership array
  local pid="$1" c newc=()
  for c in ${CHILDREN[@]+"${CHILDREN[@]}"}; do
    [ "$c" = "$pid" ] || newc+=("$c")
  done
  CHILDREN=(${newc[@]+"${newc[@]}"})
}

# reap_bounded PID GRACE_SECS : sets global REAP_RC; removes pid from CHILDREN.
# Runs in the MAIN shell so `wait` returns the child's real exit status.
reap_bounded() {
  local pid="$1" grace="${2:-10}" waited=0
  while kill -0 "$pid" 2>/dev/null; do
    awk -v w="$waited" -v g="$grace" 'BEGIN{exit !(w>=g)}' && break
    sleep 0.05; waited=$(awk -v w="$waited" 'BEGIN{printf "%.3f", w+0.05}')
  done
  if kill -0 "$pid" 2>/dev/null; then
    kill -TERM "$pid" 2>/dev/null || true
    waited=0
    while kill -0 "$pid" 2>/dev/null; do
      awk -v w="$waited" 'BEGIN{exit !(w>=2)}' && break
      sleep 0.05; waited=$(awk -v w="$waited" 'BEGIN{printf "%.3f", w+0.05}')
    done
  fi
  if kill -0 "$pid" 2>/dev/null; then
    kill -KILL "$pid" 2>/dev/null || true
    sleep 0.2
  fi
  wait "$pid"; REAP_RC=$?
  remove_child "$pid"
}

cleanup() {
  local c
  for c in ${CHILDREN[@]+"${CHILDREN[@]}"}; do
    kill -KILL "$c" 2>/dev/null || true
    wait "$c" 2>/dev/null || true
  done
  rm -rf "$WORKDIR"
}
trap cleanup EXIT INT TERM

now() { date +%s.%N; }
ms()  { awk -v a="$1" -v b="$2" 'BEGIN{printf "%.1f", (b-a)*1000}'; }

# busy-spin (high-resolution) until PID exits, bounded by a wall-clock deadline
spin_until_exit() {
  local pid="$1" t0 i=0 cur
  t0="$(now)"
  while kill -0 "$pid" 2>/dev/null; do
    i=$((i+1))
    if [ $((i % 200000)) -eq 0 ]; then
      cur="$(now)"
      awk -v c="$cur" -v a="$t0" 'BEGIN{exit !((c-a)>30)}' && { echo "SPIN_TIMEOUT" >&2; break; }
    fi
  done
}

# ---- scenario scripts ----------------------------------------------------
cat > "$WORKDIR/sleep_iter.js" <<'JS'
import { sleep } from 'k6';
export const options = { scenarios: { s: {
  executor: 'ramping-vus', startVUs: 0,
  stages: [{ duration: '1s', target: 5 }, { duration: '30s', target: 5 }],
  gracefulRampDown: '5s', gracefulStop: '5s',
} } };
export default function () { sleep(30); }
JS

cat > "$WORKDIR/busy_iter.js" <<'JS'
export const options = { scenarios: { s: {
  executor: 'ramping-vus', startVUs: 0,
  stages: [{ duration: '0.5s', target: 3 }, { duration: '30s', target: 3 }],
  gracefulRampDown: '5s', gracefulStop: '5s',
} } };
export default function () { const end = Date.now() + 60000; while (Date.now() < end) {} }
JS

cat > "$WORKDIR/teardown_busy.js" <<'JS'
import { sleep } from 'k6';
export const options = { scenarios: { s: {
  executor: 'ramping-vus', startVUs: 0,
  stages: [{ duration: '0.5s', target: 2 }, { duration: '30s', target: 2 }],
  gracefulRampDown: '5s', gracefulStop: '5s',
} } };
export default function () { sleep(30); }
export function teardown () { console.log('TEARDOWN_START'); const end = Date.now() + 3000; while (Date.now() < end) {} console.log('TEARDOWN_END'); }
JS

cat > "$WORKDIR/rampdown.js" <<'JS'
import { sleep } from 'k6';
export const options = { scenarios: { s: {
  executor: 'ramping-vus', startVUs: 0,
  stages: [{ duration: '1s', target: 6 }, { duration: '1s', target: 1 }, { duration: '30s', target: 1 }],
  gracefulRampDown: '10s', gracefulStop: '5s',
} } };
export default function () { sleep(30); }
JS

cat > "$WORKDIR/natural.js" <<'JS'
import { sleep } from 'k6';
export const options = { scenarios: { s: {
  executor: 'ramping-vus', startVUs: 0,
  stages: [{ duration: '0.5s', target: 3 }, { duration: '0.5s', target: 0 }],
  gracefulRampDown: '2s', gracefulStop: '2s',
} } };
export default function () { sleep(0.2); }
JS

cat > "$WORKDIR/natural_forced.js" <<'JS'
import { sleep } from 'k6';
export const options = { scenarios: { s: {
  executor: 'ramping-vus', startVUs: 0,
  stages: [{ duration: '0.5s', target: 3 }, { duration: '0.5s', target: 0 }],
  gracefulRampDown: '3s', gracefulStop: '3s',
} } };
export default function () { sleep(2); }
JS

K6VER="$("$K6" version 2>&1 | head -1)"
echo "################ k6 build under test ################"
echo "$K6VER"
echo "workdir: $WORKDIR ; RUNS per condition: $RUNS"
echo "binary: $K6 (canonical baseline build, commit ddc3b0b1d2 == investigated ddc3b0b1d23c)"
echo

# run_case NAME SCRIPT FIRST_DELAY SECOND_DELAY WAIT_LOG
#   FIRST_DELAY  : seconds before first SIGINT, or "none" for no signal
#   SECOND_DELAY : seconds after first SIGINT before second SIGINT, or "none"
#   WAIT_LOG     : a stderr substring to wait for before first SIGINT, or "none"
run_case() {
  local name="$1" script="$2" first="$3" second="$4" waitlog="$5"
  local r log pid t_spawn t_first t_second t_exit rc g h
  for r in $(seq 1 "$RUNS"); do
    log="$WORKDIR/${name}.run${r}.log"
    echo "===== CONDITION ${name} — run ${r}/${RUNS} ====="
    echo "\$ $K6 run --verbose --no-summary --no-usage-report ${script##*/}   (first=${first} second=${second} wait_log=${waitlog})"
    t_spawn="$(now)"; t_first=""; t_second=""
    "$K6" run --verbose --no-summary --no-usage-report "$script" >"$log" 2>&1 &
    pid=$!
    CHILDREN+=("$pid")
    echo "spawned k6 pid=$pid"

    if [ "$waitlog" != "none" ]; then
      local w=0 seen=no
      while awk -v w="$w" 'BEGIN{exit !(w<15)}'; do
        if grep -q -- "$waitlog" "$log" 2>/dev/null; then seen=yes; break; fi
        if ! kill -0 "$pid" 2>/dev/null; then break; fi
        sleep 0.05; w=$(awk -v w="$w" 'BEGIN{printf "%.2f", w+0.05}')
      done
      echo "trigger '$waitlog' observed_before_signal=$seen after ${w}s"
    fi

    if [ "$first" = "none" ]; then
      reap_bounded "$pid" 40; rc=$REAP_RC
      t_exit="$(now)"
      echo "wait result: pid=$pid exit_code=$rc  (0 == clean natural completion)"
      echo "elapsed spawn -> natural exit: $(ms "$t_spawn" "$t_exit") ms"
    else
      if [ "$waitlog" = "none" ]; then sleep "$first"; fi
      t_first="$(now)"
      if kill -0 "$pid" 2>/dev/null; then
        kill -INT "$pid"; echo "first  SIGINT: sent_at_mono=$t_first kill_status=$? alive_before=yes"
      else
        echo "first  SIGINT: process already exited before signal"; t_first=""
      fi
      if [ "$second" != "none" ]; then
        sleep "$second"
        t_second="$(now)"
        if kill -0 "$pid" 2>/dev/null; then
          kill -INT "$pid"; echo "second SIGINT: sent_at_mono=$t_second kill_status=$? alive_before=yes"
        else
          echo "second SIGINT: process already exited before second signal"; t_second=""
        fi
      fi
      spin_until_exit "$pid"
      t_exit="$(now)"
      reap_bounded "$pid" 5; rc=$REAP_RC
      echo "wait result: pid=$pid exit_code=$rc  (105 == ExternalAbort)"
      [ -n "$t_first" ]  && echo "elapsed first_SIGINT  -> process_exit: $(ms "$t_first" "$t_exit") ms"
      [ -n "$t_second" ] && echo "elapsed second_SIGINT -> process_exit: $(ms "$t_second" "$t_exit") ms"
    fi

    echo "--- signal-relevant log lines (k6 stderr, --verbose) ---"
    grep -E 'Stopping k6 in response|Aborting k6 in response|interrupted|aborted because|finished with an error|exiting k6' "$log" 2>/dev/null | sed 's/^/  /' | head -6
    g=$(grep -c 'Graceful stop' "$log" 2>/dev/null); g=${g:-0}
    h=$(grep -c 'Hard stop' "$log" 2>/dev/null); h=${h:-0}
    echo "--- state markers --- Graceful stop log count=$g ; Hard stop log count=$h"
    echo "--- teardown marker (console) ---"
    grep -E 'TEARDOWN' "$log" 2>/dev/null | sed 's/^/  /' | head -2
    echo "--- iteration progress (final two states) ---"
    grep -E 'VUs, .*complete' "$log" 2>/dev/null | tail -2 | sed 's/^/  /'
    echo "  alive_after_wait=$(kill -0 "$pid" 2>/dev/null && echo yes || echo no)"
    echo "===== end ${name} run ${r}: exit=$rc ====="
    echo
  done
}

# ---- M3 baseline: natural stage-end (NO signal) --------------------------
run_case natural_stage_end        "$WORKDIR/natural.js"        none  none none
run_case natural_forced_grace     "$WORKDIR/natural_forced.js" none  none none
# ---- single-SIGINT conditions -------------------------------------------
run_case C1_sleep_singleSIGINT    "$WORKDIR/sleep_iter.js"     2     none none
run_case C2_busy_singleSIGINT     "$WORKDIR/busy_iter.js"      2.5   none none
# ---- M3 edge: SIGINT near a stage transition ----------------------------
run_case near_transition_SIGINT   "$WORKDIR/rampdown.js"       1     none none
# ---- M3 edge: SIGINT AFTER an actual "Graceful stop" state log ----------
run_case graceful_state_SIGINT    "$WORKDIR/rampdown.js"       0     none "Graceful stop"
# ---- teardown (graceful-phase work) single + double ---------------------
run_case C3_teardown_singleSIGINT "$WORKDIR/teardown_busy.js"  2     none none
run_case C4_teardown_doubleSIGINT "$WORKDIR/teardown_busy.js"  2     0.6  none
# ---- rapid double SIGINT (~10ms spacing) escalation during teardown -----
run_case rapid_double_SIGINT      "$WORKDIR/teardown_busy.js"  2     0.01 none

echo "################ SYMPTOM C HARNESS COMPLETE (RUNS=$RUNS) ################"
```

### 9.3 Symptom D harness (`symptomD_run.sh`) — synchronized 3-process CLI runs

This Bash harness produced §7-D.2. It launches three real `k6 run` processes simultaneously (one per
segment) in three modes — **SHARED**, **INDEPENDENT**, and **SHARED_LOW** (global max 1) — twice each.
It uses `set -u` (line 25); **strict `RUNS>=2` integer validation that fails fast (`exit 2`) before any
work** (lines 33–42); a `mktemp -d` workdir; and `EXIT`/`INT`/`TERM` trap cleanup (`trap cleanup EXIT INT
TERM`, line 74) that kills only the exact spawned PIDs it still owns (no broad `pkill`). Child reaping is
**bounded** (`reap_bounded`, line 58): `wait` runs in the main shell and escalates `TERM`→`KILL` if a
child overruns, after which the reaped PID is removed from the cleanup-ownership array (`remove_child`,
line 50). Each instance emits **canonical metrics** via `--out json=<file>`; the embedded `python3`
parser (heredoc at line 102) reads the `vus_max` (scaled configured max) and `vus` (peak active) metric
`Point` records — **not** stdout progress lines — and reports the **tied maxima** plus the deterministic
remainder recipients (earliest offsets). The two JS scenarios `seg.js` and `seg_low.js` are created
inline by the `cat > … <<'JS'` heredocs at lines 77 and 91.

```bash
#!/usr/bin/env bash
#
# symptomD_run.sh — TEMPORARY, EPHEMERAL investigation harness for User Symptom D
# ("one instance consistently shows more VUs than the others at the same timestamp,
#  and if I sum them up they exceed my configured maximum").
#
# Launches THREE REAL k6 processes SIMULTANEOUSLY through the canonical `k6 run`
# entry point, each pinned to a non-overlapping --execution-segment. Modes:
#   shared      : all three share ONE --execution-segment-sequence 0,1/3,2/3,1 (canonical)
#   independent : NO sequence supplied (each instance fills its own)
#   shared_low  : shared sequence, but global max target = 1 (remainder=1 -> single higher [1,0,0])
# Each instance emits CANONICAL metrics via `--out json=<file>`; the python parser reads the
# `vus_max` (scaled configured max) and `vus` (peak active) metric Points — NOT stdout progress lines.
#
# m5: RUNS is validated as an integer >= 2 BEFORE any work.
# m6: child waits are BOUNDED with TERM->KILL escalation; each reaped PID is removed from the
#     cleanup-ownership array (no unbounded wait; no signalling of a reused PID).
# m2: the parser reports TIED maxima and the deterministic remainder recipients (earliest offsets),
#     not a single arbitrary "highest".
#
# Flags: cmd/options.go:31 --execution-segment ; :32 --execution-segment-sequence.
# Scaling: ExecutionSegmentSequenceWrapper.ScaleInt64 [lib/execution_segment.go:580] (remainder loop
#          :584-586); GetFilledExecutionSegmentSequence [lib/execution_segment.go:445].
#
set -u

K6="${K6:-/tmp/k6}"
export K6_NO_USAGE_REPORT=true
RUNS="${RUNS:-2}"
SEQ="0,1/3,2/3,1"
SEGS=("0:1/3" "1/3:2/3" "2/3:1")

# ---- m5: strict RUNS validation (fail fast, before ANY work) -------------
case "$RUNS" in
    ''|*[!0-9]*)
        echo "ERROR(m5): RUNS must be a non-negative integer, got: '$RUNS'" >&2
        exit 2 ;;
esac
if [ "$RUNS" -lt 2 ]; then
    echo "ERROR(m5): RUNS must be >= 2 (Rule 1 requires repeating identical input), got: $RUNS" >&2
    exit 2
fi

command -v "$K6" >/dev/null 2>&1 || { echo "ERROR: k6 binary not found at '$K6'" >&2; exit 3; }
WORKDIR="$(mktemp -d /tmp/k6probe_symD.XXXXXX)"

# ---- m6: bounded reap with TERM->KILL escalation + PID removal ------------
declare -a CHILDREN=()
REAP_RC=0
remove_child() {   # delete a pid from the CHILDREN ownership array
    local pid="$1" i; local -a keep=()
    for i in "${CHILDREN[@]:-}"; do
        [ -n "$i" ] && [ "$i" != "$pid" ] && keep+=("$i")
    done
    CHILDREN=("${keep[@]:-}")
}
# reap_bounded PID GRACE_SECS : sets global REAP_RC; removes pid from CHILDREN.
reap_bounded() {
    local pid="$1" grace="${2:-40}" waited=0
    while kill -0 "$pid" 2>/dev/null && [ "$waited" -lt "$grace" ]; do sleep 1; waited=$((waited + 1)); done
    if kill -0 "$pid" 2>/dev/null; then kill -TERM "$pid" 2>/dev/null || true; sleep 2; fi
    if kill -0 "$pid" 2>/dev/null; then kill -KILL "$pid" 2>/dev/null || true; sleep 1; fi
    wait "$pid" 2>/dev/null; REAP_RC=$?
    remove_child "$pid"
}
cleanup() {
    local c
    for c in "${CHILDREN[@]:-}"; do
        [ -n "${c:-}" ] || continue
        if kill -0 "$c" 2>/dev/null; then kill -KILL "$c" 2>/dev/null || true; wait "$c" 2>/dev/null || true; fi
    done
    rm -rf "$WORKDIR"
}
trap cleanup EXIT INT TERM

# ---- JS scenarios (canonical ramping-vus) --------------------------------
cat > "$WORKDIR/seg.js" <<'JS'
import { sleep } from 'k6';
// Global peak target 11 VUs across 3 equal thirds; 11 % 3 != 0 so the split is uneven.
export const options = {
  scenarios: { d: { executor: 'ramping-vus', startVUs: 0,
    stages: [
      { duration: '3s', target: 11 },
      { duration: '2s', target: 11 },
      { duration: '3s', target: 0  },
    ] } },
};
export default function () { sleep(1); }
JS

cat > "$WORKDIR/seg_low.js" <<'JS'
import { sleep } from 'k6';
// Global peak target 1 VU: remainder 1 -> exactly ONE segment (earliest offset) gets it -> [1,0,0].
export const options = {
  scenarios: { d: { executor: 'ramping-vus', startVUs: 0,
    stages: [ { duration: '2s', target: 1 }, { duration: '2s', target: 0 } ] } },
};
export default function () { sleep(1); }
JS

# ---- canonical JSON parser (m2-correct: tied maxima + remainder recipients) ----
cat > "$WORKDIR/parse.py" <<'PY'
import sys, json
mode, gmax = sys.argv[1], int(sys.argv[2])
segs = sys.argv[3:6]
jsonfiles = sys.argv[6:9]
vmax, vpeak = [], []
for f in jsonfiles:
    mx = 0; pk = 0
    with open(f) as fh:
        for line in fh:
            line = line.strip()
            if not line or '"Point"' not in line:
                continue
            try:
                obj = json.loads(line)
            except Exception:
                continue
            if obj.get("type") != "Point":
                continue
            if obj.get("metric") == "vus_max":
                mx = max(mx, int(obj["data"]["value"]))
            elif obj.get("metric") == "vus":
                pk = max(pk, int(obj["data"]["value"]))
    vmax.append(mx); vpeak.append(pk)

print("  per-instance CANONICAL metrics (JSON --out json; vus_max = scaled configured max, vus = peak active):")
for i, seg in enumerate(segs):
    print("    seg%d %-8s => vus_max=%d  peak_vus=%d" % (i, seg, vmax[i], vpeak[i]))
tot = sum(vmax)
rel = "OVER (exceeds configured max!)" if tot > gmax else ("EQUAL" if tot == gmax else "UNDER")
print("    SUM of per-instance vus_max = %d  vs configured global max = %d  => %s" % (tot, gmax, rel))

# m2: report TIED maxima + which segments received the striping remainder (earliest offsets).
mx = max(vmax) if vmax else 0
mn = min(vmax) if vmax else 0
tied = ["seg%d(%s)" % (i, segs[i]) for i, v in enumerate(vmax) if v == mx]
recipients = ["seg%d(%s)" % (i, segs[i]) for i, v in enumerate(vmax) if v > mn]
print("  per-instance vus_max = %s" % vmax)
print("  maximum per-instance share = %d ; segment(s) AT the maximum (TIED): %s  (%d of %d instances)"
      % (mx, tied, len([1 for v in vmax if v == mx]), len(segs)))
if mn == mx:
    print("  remainder distribution: ALL %d segments tie at %d — each instance rounds ITS OWN share up "
          "(independent per-instance ceiling, no shared floor) => the sum can exceed the global max" % (len(segs), mx))
else:
    surplus = sum(1 for v in vmax if v > mn)
    print("  remainder distribution: %d surplus VU(s) assigned DETERMINISTICALLY to the EARLIEST-offset "
          "segment(s) %s (striped ScaleInt64 remainder loop, execution_segment.go:584-586) => those "
          "instance(s) are consistently higher; the rest tie at the floor %d" % (surplus, recipients, mn))
PY

# ---- one synchronized triple ---------------------------------------------
# run_triple MODE RUN_NO SCRIPT GMAX USE_SEQ
run_triple() {
    local mode="$1" run_no="$2" script="$3" gmax="$4" use_seq="$5"
    local -a pids=() jsons=() rc=()
    local i js
    echo "-------- ${mode} triple (gmax=${gmax}), SYNCHRONIZED 3-process run ${run_no}/${RUNS} --------"
    for i in 0 1 2; do
        js="$WORKDIR/${mode}_run${run_no}_seg${i}.json"
        jsons[i]="$js"
        if [ "$use_seq" = "yes" ]; then
            echo "\$ $K6 run --execution-segment ${SEGS[i]} --execution-segment-sequence $SEQ --out json=<f> $script"
            "$K6" run --execution-segment "${SEGS[i]}" --execution-segment-sequence "$SEQ" \
                --no-summary --out json="$js" "$WORKDIR/$script" \
                >"$WORKDIR/${mode}_run${run_no}_seg${i}.out" 2>"$WORKDIR/${mode}_run${run_no}_seg${i}.err" &
        else
            echo "\$ $K6 run --execution-segment ${SEGS[i]} --out json=<f> $script   (NO --execution-segment-sequence)"
            "$K6" run --execution-segment "${SEGS[i]}" \
                --no-summary --out json="$js" "$WORKDIR/$script" \
                >"$WORKDIR/${mode}_run${run_no}_seg${i}.out" 2>"$WORKDIR/${mode}_run${run_no}_seg${i}.err" &
        fi
        pids[i]=$!
        CHILDREN+=("${pids[i]}")
    done
    for i in 0 1 2; do reap_bounded "${pids[i]}" 40; rc[i]=$REAP_RC; done
    echo "exit codes: seg0=${rc[0]} seg1=${rc[1]} seg2=${rc[2]}"
    python3 "$WORKDIR/parse.py" "$mode" "$gmax" "${SEGS[0]}" "${SEGS[1]}" "${SEGS[2]}" \
        "${jsons[0]}" "${jsons[1]}" "${jsons[2]}"
    echo "-------- end ${mode} run ${run_no} --------"
    echo
}

echo "################ k6 build under test ################"
"$K6" version
echo "workdir: $WORKDIR ; RUNS per mode: $RUNS ; shared sequence: $SEQ ; segments: ${SEGS[*]}"
echo

for r in $(seq 1 "$RUNS"); do run_triple "SHARED"      "$r" "seg.js"     11 "yes"; done
for r in $(seq 1 "$RUNS"); do run_triple "INDEPENDENT" "$r" "seg.js"     11 "no";  done
for r in $(seq 1 "$RUNS"); do run_triple "SHARED_LOW"  "$r" "seg_low.js"  1 "yes"; done

echo "################ SYMPTOM D HARNESS COMPLETE (RUNS=$RUNS) ################"
```

### 9.4 Cleanup attestation and repository integrity

This was a read-only investigation. The only change committed to the repository is this single document
(`blitzy/documentation/k6_ddc3b0b1d23c.md`); the k6 source tree is left byte-for-byte identical to the
investigated baseline commit `ddc3b0b1d23c` ("Update comment"). The temporary artifacts used to capture
the evidence, and their disposal, are:

- **Binary under test (outside the repo tree):** `/tmp/k6` — the canonical **baseline** build
  (`commit/ddc3b0b1d2`, `65,574,496` bytes) used for the real-`SIGINT` (§7-C) and three-process segment
  (§7-D.2) reproductions. It was **removed** after evidence capture. It is distinct from the
  environment-setup-owned `/tmp/k6_bin` (the byte-identical baseline build, same size and `sha256`), which
  this investigation did not create and does **not** remove.
- **Harness scripts + captured transcripts (outside the repo tree):** `symptomC_run.sh`,
  `symptomD_run.sh`, their captured transcripts (embedded verbatim in §7-C and §7-D.2), and the small
  helper scripts — all under a scratch directory outside the repo tree; **removed** after the transcripts
  were embedded. Each harness's JS scenarios lived in its own `mktemp -d` workdir and were deleted by that
  harness's `EXIT`/`INT`/`TERM` trap the moment it finished (no `/tmp/k6probe_*` workdirs remain).
- **In-package probe (inside the tree, untracked, never committed):**
  `lib/executor/blitzy_adhoc_test_probe_test.go` (the source embedded in §9.1) — carried the
  `blitzy_adhoc_test_` prefix and was **removed** after capture. With it gone, `go build ./lib/executor/`
  and a test-package compile both still succeed, confirming the source tree remains intact.

**Verification (actual output captured at cleanup time):**

```text
### /tmp/k6 removed; /tmp/k6_bin (setup-owned) preserved
$ test -e /tmp/k6 && echo present || echo absent
absent
$ test -f /tmp/k6_bin && echo present || echo missing
present

### no investigation probe / adhoc files remain in the repo tree
$ find . -name 'blitzy_adhoc_test_*' -not -path './.git/*'
(no output)

### source tree byte-for-byte unchanged vs baseline (restricted to source dirs)
$ git diff ddc3b0b1d23c -- lib execution cmd | wc -l
0

### the sole addition vs the investigated baseline is this document
$ git diff ddc3b0b1d23c..HEAD --name-status
A	blitzy/documentation/k6_ddc3b0b1d23c.md
```

**Working-tree status vs. baseline→HEAD diff — they answer different questions.** On the *committed*
result, `git status --porcelain` is **empty**: a clean working tree carries no entry for an
already-committed file. The one-document change is therefore **not** shown by `git status`; it is visible
only in the baseline→HEAD comparison above (`A  blitzy/documentation/k6_ddc3b0b1d23c.md`). (Before the
final commit, the same document appears exactly once in `git status --porcelain` as a single changed path
and nowhere else; committing it folds that single delta into `HEAD`, leaving the working tree clean.)

No existing k6 source or test file was modified, refactored, or deleted at any point in this
investigation.
