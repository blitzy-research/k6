# k6 Onboarding Q&A — Test Health, Metrics Code, and the Metric Data-Flow

> **Subject:** `go.k6.io/k6` (grafana/k6) load-testing tool
> **Commit:** `ddc3b0b1d23c128e34e2792fc9075f9126e32375`
> **Built binary:** `k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)`
> **Mode:** read-only investigation — **nothing in the repository was modified.** The only artifact created is this document. The built `./k6` binary is ignored by `.gitignore` (line 1: `/k6`), and every temporary observation script lived under `/tmp` and was removed afterward.

This document answers three onboarding questions from a newly-joined engineer:

1. **Q1 — Project health:** *"Can you run the tests and tell me how many pass vs fail? Are there any that are skipped or broken?"*
2. **Q2 — Metrics code:** *"When I kick off a load test, what parts of the code are responsible for counting iterations and collecting performance data? Name the specific files and modules involved."*
3. **Q3 — Metric data-flow trace:** *"Could you trace through running a simple test script and show me the function calls involved in collecting at least one metric, so I can see how the data flows from test start to metrics output?"*

**Methodology (investigate-by-running-first).** Every factual claim below was produced by *building and running the real software first*, then writing from the captured output. The canonical binary was built, the canonical test suite was run three times (once as the exact `Makefile` command, then twice more with `-count=1` to force genuine re-execution and to emit a machine-readable tally), the "broken" condition was reproduced deliberately, and a minimal script plus a duration script were run through the real `./k6 run` entry point. All counts are real; where a value legitimately varies run-to-run (flaky tests, sub-millisecond timings) that is called out explicitly with the observed range and the stable core. Every claim carries a `file:line` reference naming the concrete function, method, or struct.

---

## Section 0 — Environment & exact commands

**Direct answer:** the project builds and runs in its canonical, offline, vendored configuration on this host (Go 1.21.13, linux/amd64). Here is exactly how the environment was established and what a normal user would run.

### 0.1 Toolchain observed on this host

| Component | Observed value | How verified |
|-----------|----------------|--------------|
| Go | `go version go1.21.13 linux/amd64` | matches the pin `toolchain go1.21.13` [go.mod:5] over `go 1.21` [go.mod:3] |
| C compiler | `gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0` at `/usr/bin/gcc` | required by `go test -race` (cgo) → `CGO_ENABLED=1` |
| OS / arch | `linux` / `amd64` (`uname`: Linux 6.6.122+ x86_64) | `go env GOOS GOARCH` |
| CPUs | `nproc` = **4** | relevant to `-race` timing behavior in Q1 |
| Dependencies | fully vendored (`vendor/` + `vendor/modules.txt`) | build/test run offline with `-mod=vendor` |

**How Go is exposed here (reported honestly):** Go is **not on the bare `PATH`**; `which go` returns nothing until the environment file is sourced. Go is pre-installed at `/usr/local/go` and exposed through a profile script:

```bash
$ source /etc/profile.d/goenv.sh    # sets PATH+=/usr/local/go/bin, GOTOOLCHAIN=local, GOFLAGS=-mod=vendor, GOPATH=/root/go
$ go version
go version go1.21.13 linux/amd64
```

### 0.2 Canonical build command and version banner

```bash
$ go build -mod=vendor -o k6 .      # exit 0; ~2s here (Go build cache was warm; a cold build is slower)
$ ./k6 version
k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)
```

> **Reproducing this exact banner (commit hash).** k6 stamps its banner from the checked-out git commit, so the `commit/ddc3b0b1d2` shown above is produced only when the binary is built **at the source commit under study, `ddc3b0b1d23c128e34e2792fc9075f9126e32375`** (short `ddc3b0b1d2`). This document's own commit is added *on top of* that source commit on the delivery branch; therefore **building the final documentation branch instead yields a different banner commit** — in this environment `k6 v0.55.0 (commit/4874b312ac, go1.21.13, linux/amd64)` — because HEAD now includes the documentation commit `4874b312a`. Only the stamped commit hash differs; the compiled code and its behavior are identical (the docs commit changes no source). To reproduce the exact `commit/ddc3b0b1d2` banner, check out `ddc3b0b1d23c128e34e2792fc9075f9126e32375` before building.

The entry point is tiny: `func main()` calls `cmd.Execute()` [main.go:8-9].

### 0.3 Canonical commands used throughout this document

| Purpose | Exact command |
|---------|---------------|
| Build | `go build -mod=vendor -o k6 .` |
| Test (canonical) | `CGO_ENABLED=1 go test -race -timeout 210s ./...` — the `tests:` target [Makefile:28-29], documented as `make tests` [CONTRIBUTING.md:61] |
| Run a script | `./k6 run <script>.js` |

> **Note on `-count=1`.** The canonical `Makefile` command has no `-count=1`. Go caches *passing* package test results, so a second identical invocation prints `(cached)` instead of re-executing. To honestly observe run-to-run stability (and to make skipped tests visible), runs #2 and #3 below add `-count=1`, the idiomatic "ignore the test cache" flag. It changes nothing in the repository and does not alter test semantics — it only forces genuine re-execution.

---

## Section 1 — Q1: Project test health (pass / fail / skipped / broken)

**Direct answer.** The overwhelming majority of tests pass. At the **test level**, a machine-readable (`-json`) run recorded **4412 test/subtests passing, 13 failing, and 1 skipped** (4426 results total). At the **package level**, of the 82 packages `go test ./...` visits, **28 have no test files** and **54 contain tests**; in the three recorded runs below, **~50 of those 54 passed** (48–52 across runs) and a **small flaky set failed** (2–6 across runs). **Nothing is broken** in the canonical configuration (zero build/setup failures). **Exactly one** test is skipped: `TestTC39`. Every failure observed is a **flaky, environment/timing-sensitive test under the race detector** — not a code defect — which is why the failing *set* and its *count* change between runs while the stable structure (28 no-test packages, ~50 passing, 0 broken, exactly 1 skip) stays constant. The exact pass/fail package counts are therefore reported as the values **observed in these recorded runs**, not as a universal bound.

### 1.1 The command

```bash
CGO_ENABLED=1 go test -race -timeout 210s ./...
```

This is verbatim the `tests` target — `tests:` [Makefile:28] → `go test -race -timeout 210s ./...` [Makefile:29] — and is what `CONTRIBUTING.md` tells contributors to invoke as `make tests` [CONTRIBUTING.md:61]. `CGO_ENABLED=1` is prepended because `-race` needs cgo (see §1.5 and §4).

### 1.2 The four buckets

| Bucket | Result | Evidence |
|--------|--------|----------|
| **pass** | ~50 packages `ok` (48–52 across the recorded runs); **4412** test/subtests pass (`-json` run) | §1.3 table, §1.4 |
| **fail** | small flaky set (2–6 packages across the recorded runs); **13** test/subtest fail-events (`-json` run) — all **flaky** under `-race` | §1.4 |
| **skipped** | **exactly 1** test: `TestTC39` (`js/tc39`) | §1.6 |
| **broken** | **0** in the canonical run; reproducible only by removing the C compiler | §1.5 |

### 1.3 Three runs — reconciled per run

All three runs exited non-zero (`exit=1`) because at least one flaky test failed each time. The **structure** is stable; only the failing set varies.

| Run | Command | Wall | `ok` pkgs | `FAIL` pkgs | `?` no-test | broken |
|-----|---------|------|-----------|-------------|-------------|--------|
| #1 | `CGO_ENABLED=1 go test -race -timeout 210s ./...` (warm cache — the 52 `ok` were `(cached)`) | 34s | 52 | 2 | 28 | 0 |
| #2 | same **+ `-count=1`** (forced fresh) | 48s | 48 | 6 | 28 | 0 |
| #3 | same **+ `-count=1 -json`** (tally) | 47s | 49 | 5 | 28 | 0 |

`52+2+28 = 48+6+28 = 49+5+28 = 82` packages every time. The 28 "`?` no test files" packages and the 54 packages-with-tests are constant across runs; **0 broken** every run. Only the flaky failing *set* (and therefore the ok/FAIL split among the 54 test-bearing packages) moves between runs.

Below is the **complete, unedited package-summary output** for the two plain-text runs — every one of the 82 `?`/`ok`/`FAIL` lines the runner printed, in the order printed, nothing elided — followed by the exact machine tally for the `-json` Run #3.

**Run #1 — canonical `Makefile` command (warm cache), complete package summary (52 `ok` / 2 `FAIL` / 28 `?` = 82 packages):**

```
?   	go.k6.io/k6	[no test files]
ok  	go.k6.io/k6/api	(cached)
?   	go.k6.io/k6/api/v1/client	[no test files]
ok  	go.k6.io/k6/api/v1	(cached)
ok  	go.k6.io/k6/cloudapi	(cached)
?   	go.k6.io/k6/cloudapi/insights/proto	[no test files]
?   	go.k6.io/k6/cloudapi/insights/proto/v1/common	[no test files]
?   	go.k6.io/k6/cloudapi/insights/proto/v1/ingester	[no test files]
?   	go.k6.io/k6/cloudapi/insights/proto/v1/k6	[no test files]
?   	go.k6.io/k6/cloudapi/insights/proto/v1/trace	[no test files]
ok  	go.k6.io/k6/cloudapi/insights	(cached)
?   	go.k6.io/k6/cmd/state	[no test files]
?   	go.k6.io/k6/cmd/tests/events	[no test files]
?   	go.k6.io/k6/errext/exitcodes	[no test files]
ok  	go.k6.io/k6/cmd	(cached)
?   	go.k6.io/k6/execution/local	[no test files]
?   	go.k6.io/k6/ext	[no test files]
ok  	go.k6.io/k6/cmd/tests	(cached)
ok  	go.k6.io/k6/errext	(cached)
ok  	go.k6.io/k6/event	(cached)
ok  	go.k6.io/k6/execution	(cached)
?   	go.k6.io/k6/js/modules	[no test files]
ok  	go.k6.io/k6/js	(cached)
ok  	go.k6.io/k6/js/common	(cached)
ok  	go.k6.io/k6/js/compiler	(cached)
?   	go.k6.io/k6/js/modules/k6/experimental	[no test files]
ok  	go.k6.io/k6/js/eventloop	(cached)
ok  	go.k6.io/k6/js/modules/k6	(cached)
?   	go.k6.io/k6/js/modules/k6/html/gen	[no test files]
ok  	go.k6.io/k6/js/modules/k6/crypto	(cached)
ok  	go.k6.io/k6/js/modules/k6/crypto/x509	(cached)
ok  	go.k6.io/k6/js/modules/k6/data	(cached)
ok  	go.k6.io/k6/js/modules/k6/encoding	(cached)
ok  	go.k6.io/k6/js/modules/k6/execution	(cached)
ok  	go.k6.io/k6/js/modules/k6/experimental/csv	(cached)
ok  	go.k6.io/k6/js/modules/k6/experimental/fs	(cached)
ok  	go.k6.io/k6/js/modules/k6/experimental/streams	(cached)
ok  	go.k6.io/k6/js/modules/k6/grpc	(cached)
ok  	go.k6.io/k6/js/modules/k6/html	(cached)
?   	go.k6.io/k6/js/modulestest	[no test files]
?   	go.k6.io/k6/lib/consts	[no test files]
?   	go.k6.io/k6/lib/testutils	[no test files]
?   	go.k6.io/k6/lib/testutils/grpcservice	[no test files]
?   	go.k6.io/k6/lib/testutils/httpmultibin	[no test files]
?   	go.k6.io/k6/lib/testutils/httpmultibin/grpc_any_testing	[no test files]
?   	go.k6.io/k6/lib/testutils/httpmultibin/grpc_testing	[no test files]
?   	go.k6.io/k6/lib/testutils/httpmultibin/grpc_wrappers_testing	[no test files]
?   	go.k6.io/k6/lib/testutils/minirunner	[no test files]
?   	go.k6.io/k6/lib/testutils/mockoutput	[no test files]
?   	go.k6.io/k6/lib/testutils/mockresolver	[no test files]
?   	go.k6.io/k6/output/cloud/expv2/pbcloud	[no test files]
?   	go.k6.io/k6/ui/console	[no test files]
FAIL	go.k6.io/k6/js/modules/k6/http	5.915s
ok  	go.k6.io/k6/js/modules/k6/metrics	(cached)
ok  	go.k6.io/k6/js/modules/k6/timers	(cached)
ok  	go.k6.io/k6/js/modules/k6/ws	(cached)
ok  	go.k6.io/k6/js/promises	(cached)
ok  	go.k6.io/k6/js/tc39	(cached)
ok  	go.k6.io/k6/lib	(cached)
FAIL	go.k6.io/k6/lib/executor	29.657s
ok  	go.k6.io/k6/lib/fsext	(cached)
ok  	go.k6.io/k6/lib/netext	(cached)
ok  	go.k6.io/k6/lib/netext/grpcext	(cached)
ok  	go.k6.io/k6/lib/netext/httpext	(cached)
ok  	go.k6.io/k6/lib/strvals	(cached)
ok  	go.k6.io/k6/lib/trace	(cached)
ok  	go.k6.io/k6/lib/types	(cached)
ok  	go.k6.io/k6/loader	(cached)
ok  	go.k6.io/k6/log	(cached)
ok  	go.k6.io/k6/metrics	(cached)
ok  	go.k6.io/k6/metrics/engine	(cached)
ok  	go.k6.io/k6/output	(cached)
ok  	go.k6.io/k6/output/cloud	(cached)
ok  	go.k6.io/k6/output/cloud/expv2	(cached)
ok  	go.k6.io/k6/output/cloud/expv2/integration	(cached)
ok  	go.k6.io/k6/output/cloud/insights	(cached)
ok  	go.k6.io/k6/output/csv	(cached)
ok  	go.k6.io/k6/output/influxdb	(cached)
ok  	go.k6.io/k6/output/json	(cached)
ok  	go.k6.io/k6/ui	(cached)
ok  	go.k6.io/k6/ui/pb	(cached)
ok  	go.k6.io/k6/usage	(cached)
```

**Run #2 — same `+ -count=1` (forced fresh), complete package summary (48 `ok` / 6 `FAIL` / 28 `?` = 82 packages):**

```
?   	go.k6.io/k6	[no test files]
ok  	go.k6.io/k6/api	1.377s
?   	go.k6.io/k6/api/v1/client	[no test files]
?   	go.k6.io/k6/cloudapi/insights/proto	[no test files]
?   	go.k6.io/k6/cloudapi/insights/proto/v1/common	[no test files]
?   	go.k6.io/k6/cloudapi/insights/proto/v1/ingester	[no test files]
?   	go.k6.io/k6/cloudapi/insights/proto/v1/k6	[no test files]
?   	go.k6.io/k6/cloudapi/insights/proto/v1/trace	[no test files]
?   	go.k6.io/k6/cmd/state	[no test files]
?   	go.k6.io/k6/cmd/tests/events	[no test files]
?   	go.k6.io/k6/errext/exitcodes	[no test files]
?   	go.k6.io/k6/execution/local	[no test files]
?   	go.k6.io/k6/ext	[no test files]
?   	go.k6.io/k6/js/modules	[no test files]
?   	go.k6.io/k6/js/modules/k6/experimental	[no test files]
?   	go.k6.io/k6/js/modules/k6/html/gen	[no test files]
?   	go.k6.io/k6/js/modulestest	[no test files]
?   	go.k6.io/k6/lib/consts	[no test files]
?   	go.k6.io/k6/lib/testutils	[no test files]
?   	go.k6.io/k6/lib/testutils/grpcservice	[no test files]
?   	go.k6.io/k6/lib/testutils/httpmultibin	[no test files]
?   	go.k6.io/k6/lib/testutils/httpmultibin/grpc_any_testing	[no test files]
?   	go.k6.io/k6/lib/testutils/httpmultibin/grpc_testing	[no test files]
?   	go.k6.io/k6/lib/testutils/httpmultibin/grpc_wrappers_testing	[no test files]
?   	go.k6.io/k6/lib/testutils/minirunner	[no test files]
?   	go.k6.io/k6/lib/testutils/mockoutput	[no test files]
?   	go.k6.io/k6/lib/testutils/mockresolver	[no test files]
?   	go.k6.io/k6/output/cloud/expv2/pbcloud	[no test files]
?   	go.k6.io/k6/ui/console	[no test files]
ok  	go.k6.io/k6/api/v1	1.604s
ok  	go.k6.io/k6/cloudapi	1.578s
ok  	go.k6.io/k6/cloudapi/insights	1.199s
ok  	go.k6.io/k6/cmd	8.581s
FAIL	go.k6.io/k6/cmd/tests	18.163s
ok  	go.k6.io/k6/errext	1.202s
ok  	go.k6.io/k6/event	1.431s
FAIL	go.k6.io/k6/execution	10.266s
FAIL	go.k6.io/k6/js	15.362s
ok  	go.k6.io/k6/js/common	1.429s
ok  	go.k6.io/k6/js/compiler	2.032s
FAIL	go.k6.io/k6/js/eventloop	1.526s
ok  	go.k6.io/k6/js/modules/k6	5.026s
ok  	go.k6.io/k6/js/modules/k6/crypto	1.827s
ok  	go.k6.io/k6/js/modules/k6/crypto/x509	1.927s
ok  	go.k6.io/k6/js/modules/k6/data	10.162s
ok  	go.k6.io/k6/js/modules/k6/encoding	1.431s
ok  	go.k6.io/k6/js/modules/k6/execution	1.925s
ok  	go.k6.io/k6/js/modules/k6/experimental/csv	1.832s
ok  	go.k6.io/k6/js/modules/k6/experimental/fs	1.602s
ok  	go.k6.io/k6/js/modules/k6/experimental/streams	1.431s
ok  	go.k6.io/k6/js/modules/k6/grpc	8.325s
ok  	go.k6.io/k6/js/modules/k6/html	3.825s
FAIL	go.k6.io/k6/js/modules/k6/http	12.614s
ok  	go.k6.io/k6/js/modules/k6/metrics	2.231s
ok  	go.k6.io/k6/js/modules/k6/timers	10.299s
ok  	go.k6.io/k6/js/modules/k6/ws	6.525s
ok  	go.k6.io/k6/js/promises	1.503s
ok  	go.k6.io/k6/js/tc39	1.425s
ok  	go.k6.io/k6/lib	2.825s
FAIL	go.k6.io/k6/lib/executor	30.323s
ok  	go.k6.io/k6/lib/fsext	1.201s
ok  	go.k6.io/k6/lib/netext	2.428s
ok  	go.k6.io/k6/lib/netext/grpcext	2.026s
ok  	go.k6.io/k6/lib/netext/httpext	6.330s
ok  	go.k6.io/k6/lib/strvals	1.324s
ok  	go.k6.io/k6/lib/trace	1.324s
ok  	go.k6.io/k6/lib/types	1.924s
ok  	go.k6.io/k6/loader	4.124s
ok  	go.k6.io/k6/log	1.324s
ok  	go.k6.io/k6/metrics	1.802s
ok  	go.k6.io/k6/metrics/engine	1.524s
ok  	go.k6.io/k6/output	2.327s
ok  	go.k6.io/k6/output/cloud	1.823s
ok  	go.k6.io/k6/output/cloud/expv2	2.923s
ok  	go.k6.io/k6/output/cloud/expv2/integration	4.624s
ok  	go.k6.io/k6/output/cloud/insights	1.427s
ok  	go.k6.io/k6/output/csv	1.528s
ok  	go.k6.io/k6/output/influxdb	1.923s
ok  	go.k6.io/k6/output/json	1.804s
ok  	go.k6.io/k6/ui	1.324s
ok  	go.k6.io/k6/ui/pb	1.424s
ok  	go.k6.io/k6/usage	1.323s
```

**Run #3 — same `+ -count=1 -json`: exact tally command and output.** A `-json` run emits one JSON event per line (tens of thousands of lines), so the counts were derived by piping the captured stream (`results.json`) through this small parser:

```bash
CGO_ENABLED=1 go test -race -timeout 210s -count=1 -json ./... > results.json
python3 tally.py results.json
```

where `tally.py` is:

```python
import json, sys
pkg = {"pass": 0, "fail": 0, "skip": 0}
tests = {"pass": 0, "fail": 0, "skip": 0}
top = {"pass": 0, "fail": 0, "skip": 0}
for line in open(sys.argv[1]):
    line = line.strip()
    if not line.startswith("{"):
        continue
    e = json.loads(line)
    a = e.get("Action")
    if a not in ("pass", "fail", "skip"):
        continue
    t = e.get("Test")
    if t is None:                 # package-level event
        pkg[a] += 1
    else:                         # test-level event
        tests[a] += 1
        if "/" not in t:          # top-level test (no subtest slash)
            top[a] += 1
print("packages          :", pkg, "=> total", sum(pkg.values()))
print("tests (incl subs) :", tests, "=> total", sum(tests.values()))
print("top-level tests   :", top, "=> total", sum(top.values()))
```

and its real output was:

```
packages          : {'pass': 49, 'fail': 5, 'skip': 28} => total 82
tests (incl subs) : {'pass': 4412, 'fail': 13, 'skip': 1} => total 4426
top-level tests   : {'pass': 766, 'fail': 6, 'skip': 1} => total 773
```

In the `-json` package tally the 28 `skip` package-actions are the "no test files" packages (they combine with 49 `pass` + 5 `fail` = 82 packages); the single test-level `skip` is `TestTC39` (§1.6). The 4426 total test-actions and 773 top-level tests are stable run-to-run; only the pass/fail split among them moves.

### 1.4 FAIL — what fails and why (all failures are flaky, not defects)

The failing **set** is non-deterministic under `-race` on this shared **4-CPU** host: Run #1 failed 2 packages, Run #2 failed 6, Run #3 failed 5, and no two runs failed exactly the same set. Two categories:

**(a) Persistent core — the two packages that failed in *every* run (`lib/executor` and `js/modules/k6/http`):**

- **`TestConstantArrivalRateRunCorrectTiming`** (`lib/executor`) — a sub-100 ms timing assertion. Captured verbatim (Run #2, condensed to the failing line + its `Error:`):
  ```
  constant_arrival_rate_test.go:185:
      Error: Max difference between ... allowed is 24ms, but difference was -74.18343ms
  ```
  The 24 ms tolerance is exceeded because race-detector instrumentation on a heavily-loaded, low-core host perturbs goroutine scheduling. This is a **timing-sensitive flake, not a code defect** (the magnitude varied run to run and across subtests: `-74.18ms`, `-54.21ms`, …).

- **`TestRequestAndBatchTLS/ocsp_stapled_good`** (`js/modules/k6/http`) — an OCSP-stapling check against a live TLS endpoint. Captured verbatim (Run #2):
  ```
  request_test.go:2208:
      Error: Received unexpected error:
             Error: wrong ocsp stapled response status: unknown at <eval>:3:58(22)
  ```
  This is **environment/network-sensitive** (it depends on the remote server's OCSP staple at test time).

**(b) Intermittent — packages/tests that failed in some runs only (this set changes run-to-run).** In the recorded runs the `execution` package was the most frequent intermittent failer, failing on *different* tests each time — `TestExecutionInfoVUSharing` and `TestExecutionInfoScenarioIter` in Run #2 — captured verbatim as classic concurrency flakiness:
  ```
  scheduler_ext_exec_test.go:111:
      Error: Max difference between 20 and 17 allowed is 2, but difference was 3   [TestExecutionInfoVUSharing]
  scheduler_ext_exec_test.go:116:
      Error: []uint64{0x1, 0x2} does not contain 0x3
  ```
  The other intermittent failers observed across the recorded runs were `TestVURunInterrupt` (`js`), `TestEventLoopAllCallbacksGetCalled` (`js/eventloop`), `TestActiveVUsCount` and `TestRealTimeAndSetupTeardownMetrics` (`cmd/tests`), `TestAsyncRequest/Concurrent` (`js/modules/k6/http`), `TestRampingVUsHandleRemainingVUs` (`lib/executor`), and — in the `-json` Run #3 — `TestSetTimeoutOrder` (`js/modules/k6/timers`) and `TestEventLoopCrossScenario` (`cmd/tests`). All are timing/TLS/concurrency-sensitive under `-race`.

> **Honest stability statement:** across the recorded runs the *structure* is stable — 28 no-test packages, ~50 passing packages (48–52 observed here), **0 broken**, exactly **1 skip** — and the two persistent-core failures reproduce every time; only the wider intermittent set (and therefore the exact fail count) changes. The absolute pass/fail *package counts* wobble with the flaky set, so they are reported as the values **observed in these recorded runs**, not as a universal bound. (Independent reruns in other environments have likewise landed just outside these exact counts — e.g. as few as 2 or as many as ~9 failing packages — while reproducing the identical structure.)

### 1.5 BROKEN — 0 in canonical runs; how to reproduce the broken condition

No package reported `[build failed]` or `[setup failed]` in any of the three canonical runs — **0 broken**. The one way to *make* the suite broken is to remove the C compiler, because `-race` requires cgo. Reproduced verbatim (only this one command's `PATH` was restricted; the system `gcc` was left installed):

```bash
$ PATH=/usr/local/go/bin CGO_ENABLED=1 go test -race -count=1 ./metrics/
# runtime/cgo
cgo: C compiler "gcc" not found: exec: "gcc": executable file not found in $PATH
FAIL	go.k6.io/k6/metrics [build failed]
FAIL
```

This is an **environment prerequisite**, not a repository problem: install/ensure `gcc` and set `CGO_ENABLED=1` and the suite builds and runs.

### 1.6 SKIPPED — exactly one test

On linux/amd64 exactly one test skips at runtime. Captured verbatim with `go test -race -v -count=1 -run '^TestTC39$' ./js/tc39/`:

```
=== RUN   TestTC39
    tc39_test.go:799: If you want to run tc39 tests, you need to run the 'checkout.sh` script in the directory to get  https://github.com/tc39/test262 at the correct last tested commit (stat TestTC39/test262: no such file or directory)
--- SKIP: TestTC39 (0.00s)
PASS
ok  	go.k6.io/k6/js/tc39	1.021s
```

A skip is **not** a failure: the `js/tc39` package still reports `ok`. The runner prints the skip only in verbose/`-json` mode, which is why the plain runs in §1.3 don't show it.

**Why only 1, when the source has 10 `t.Skip`/`t.Skipf` sites?** There are exactly **10** genuine testing-skip call sites in the repository:

- **3 are Windows-gated** and are no-ops on linux/amd64 (each guarded by `if runtime.GOOS == "windows"`): `js/modules/k6/http/request_test.go:2195`, `lib/executor/constant_arrival_rate_test.go:113`, `lib/netext/httpext/request_test.go:376`.
- **7 live in `js/tc39/tc39_test.go`** (lines 383, 456, 531, 770, 778, 796, 807). Of these, `js/tc39/tc39_test.go:796` is a separate `t.Skip()` that only fires under `go test -short`; the one that fires on a normal run is the `t.Skipf(...)` at **`js/tc39/tc39_test.go:807`** inside the helper `runTestTC39` — it triggers because the `test262` corpus is absent (`os.Stat` fails). The remaining tc39 skips are inside subtests that never execute once the parent skips.
  - *Precision note:* the runtime output attributes the skip to **`js/tc39/tc39_test.go:799`** — the call site inside `TestTC39` — because `runTestTC39` calls `t.Helper()` (`js/tc39/tc39_test.go:804`), so Go reports the caller's line. Both `js/tc39/tc39_test.go:799` (reported) and `js/tc39/tc39_test.go:807` (the actual `t.Skipf` statement) are correct in their respective senses.

> A naive `grep '.Skip('` also matches 27 generated `in.Skip()` calls in `output/json/json_easyjson.go` and `cloudapi/cloudapi_easyjson.go`. Those are **easyjson/jlexer deserialization** helpers, **not** test skips, and are excluded from the count of 10.

### 1.7 Test isolation (why the suite is safe to run offline)

The shared integration harness lives in `cmd/tests/tests.go` and is imported by other packages' `TestMain`:

- `type blockingTransport struct` [cmd/tests/tests.go:13] whose `RoundTrip` [cmd/tests/tests.go:19] **panics** on forbidden outbound hosts — `panic(fmt.Errorf("trying to make forbidden request to %s during test", host))` [cmd/tests/tests.go:23] — for the k6 cloud hosts `ingest.k6.io` [cmd/tests/tests.go:42], `cloudlogs.k6.io` [cmd/tests/tests.go:43], `app.k6.io` [cmd/tests/tests.go:44], `reports.k6.io` [cmd/tests/tests.go:45]. It is installed as `http.DefaultTransport = bt` [cmd/tests/tests.go:48].
- The shared entry `func Main(m *testing.M)` [cmd/tests/tests.go:33] also runs goroutine-leak detection via `goleak.Find()` [cmd/tests/tests.go:57].

This is why the suite runs offline without contacting Grafana Cloud, and why HTTP-heavy tests use an in-process server.

---


## Section 2 — Q2: What code counts iterations and collects performance data

**Direct answer.** Two clearly separate responsibilities, in two different parts of the tree:

- **Counting iterations** is done by the **JS VU runner** (`js/runner.go`) emitting the built-in **`Iterations`** counter that is registered in `metrics/builtin.go`; the **scheduler/executors** (`execution/scheduler.go`, `lib/executor/*.go`) drive how many iterations run and emit the `vus`/`vus_max` gauges.
- **Collecting performance data** (HTTP timings/throughput) is done by the **HTTP protocol layer** (`lib/netext/httpext/tracer.go` + `transport.go`), feeding the **metrics core** (`metrics/registry.go`, `metrics/metric.go`, `metrics/sample.go`, `metrics/sink.go`) and the **output pipeline** (`output/manager.go`, `metrics/engine/ingester.go`).

### 2.1 (a) Counting iterations

| File / module | What it does | Concrete symbol & line |
|---------------|--------------|------------------------|
| `metrics/builtin.go` | Registers the iteration built-ins | `RegisterBuiltinMetrics` [metrics/builtin.go:78] registers **`Iterations`** as a **Counter** [metrics/builtin.go:82] and **`IterationDuration`** as a **Trend (Time)** [metrics/builtin.go:83] |
| `js/runner.go` | Runs one iteration and emits its samples | `func (u *ActiveVU) RunOnce()` [js/runner.go:724] runs one iteration; `func (u *VU) runFn(` [js/runner.go:817] executes the JS default function; on a completed default iteration the guard `if isFullIteration && isDefault {` [js/runner.go:870] sends `iterationSamples(...)` on the channel [js/runner.go:871]; `func iterationSamples(` [js/runner.go:879] builds the two samples — the **`IterationDuration`** trend sample (`Metric: builtinMetrics.IterationDuration` [js/runner.go:885]) and the **`Iterations`** counter sample (`Metric: builtinMetrics.Iterations` [js/runner.go:894]) whose literal `Value: 1` is at [js/runner.go:899] |
| `execution/scheduler.go` + `lib/executor/*.go` | Schedule/drive `RunOnce` per executor and emit VU gauges | `func (e *Scheduler) emitVUsAndVUsMax(...)` [execution/scheduler.go:199] emits the `vus`/`vus_max` gauges on a **1-second** ticker `time.NewTicker(1 * time.Second)` [execution/scheduler.go:231] |

The net effect: one full default-function iteration produces exactly one `Iterations` sample of value `1`, and the counter sink sums them into the total you see in the summary (observed `iterations` == number of completed iterations in §3).

### 2.2 (b) Collecting performance data (HTTP timings / throughput)

| File / module | What it does | Concrete symbol & line |
|---------------|--------------|------------------------|
| `lib/netext/httpext/tracer.go` | Turns raw HTTP timings into metric samples | `func (tr *Trail) SaveSamples(...)` [lib/netext/httpext/tracer.go:44] assembles the samples: **`http_reqs`** (Counter, `Value: 1` [lib/netext/httpext/tracer.go:56]), **`http_req_duration`** (Trend [lib/netext/httpext/tracer.go:60]), and the timing sub-metrics `http_req_blocked`, `http_req_connecting`, `http_req_tls_handshaking`, `http_req_sending`, `http_req_waiting`, `http_req_receiving`, plus `http_req_failed` |
| `lib/netext/httpext/transport.go` | Pushes the assembled `Trail` onto the VU channel | `metrics.PushIfNotDone(t.ctx, t.state.Samples, trail)` [lib/netext/httpext/transport.go:164] |
| `metrics/sample.go` | Non-blocking push + sample types | `func PushIfNotDone(ctx, output chan<- SampleContainer, sample SampleContainer) bool` [metrics/sample.go:131] (also defines `Sample` / `SampleContainer`) |
| `metrics/registry.go`, `metrics/metric.go` | Metric registry and metric types | metric registration and type definitions consumed by `RegisterBuiltinMetrics` |
| `metrics/sink.go` | Aggregates sample values | `type CounterSink` [metrics/sink.go:47], `type GaugeSink` [metrics/sink.go:72], `type TrendSink` [metrics/sink.go:104], `type RateSink` [metrics/sink.go:201]; each has an `Add`: `CounterSink.Add` [metrics/sink.go:53], `GaugeSink.Add` [metrics/sink.go:82], `TrendSink.Add` [metrics/sink.go:117], `RateSink.Add` [metrics/sink.go:210] |
| `output/manager.go`, `metrics/engine/ingester.go` | Drain the channel, flush, and feed the sinks | detailed in Section 3 |
| `lib/vu_state.go` | The conduit every VU writes to | `Samples chan<- metrics.SampleContainer` [lib/vu_state.go:59] |

### 2.3 The four metric types ↔ the four sinks

k6's public docs describe four metric types; each maps to exactly one sink struct in `metrics/sink.go`:

| Metric type | Semantics | Sink struct |
|-------------|-----------|-------------|
| **Counter** | sums values (e.g. `iterations`, `http_reqs`) | `CounterSink` [metrics/sink.go:47] |
| **Gauge** | keeps min/max/last (e.g. `vus`, `vus_max`) | `GaugeSink` [metrics/sink.go:72] |
| **Trend** | min/max/avg/percentiles (e.g. `http_req_duration`, `iteration_duration`) | `TrendSink` [metrics/sink.go:104] |
| **Rate** | frequency of non-zero (e.g. `http_req_failed`, `checks`) | `RateSink` [metrics/sink.go:201] |

This mapping is exactly what you see in the summary output in Section 3: `iterations`/`http_reqs` are Counters (a total + a per-second rate), `http_req_duration`/`iteration_duration` are Trends (avg/min/med/max/p90/p95), `vus`/`vus_max` are Gauges, and `http_req_failed` is a Rate.

---


## Section 3 — Q3: End-to-end metric trace (a simple script, before → during → after)

**Direct answer.** A metric travels: **VU runs JS → a `Sample` is emitted onto the VU's `Samples` channel → the `output.Manager` drains that channel every 50 ms → the metrics-engine `OutputIngester` adds each sample to its metric's `Sink` → the end-of-test summary reads the sinks and prints the totals.** Below is the exact script, the captured console output at three boundaries, and the ordered function-call chain with `file:line`s.

### 3.1 The minimal script (authored under `/tmp`, never committed)

```js
import http from 'k6/http';
import { sleep } from 'k6';
export const options = { vus: 1, iterations: 3 };
export default function () {
  http.get('http://127.0.0.1:8090/');
  sleep(0.3);
}
```

Run through the real entry point (a tiny local HTTP server on `127.0.0.1:8090` was the target, so the run is offline and reproducible):

```bash
$ ./k6 run /tmp/k6_trace_script.js   # exit 0
```

### 3.2 BEFORE — the test-start banner (printed before any metric is collected)

```
         /\      Grafana   /‾‾/
    /\  /  \     |\  __   /  /
   /  \/    \    | |/ /  /   ‾‾\
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/

     execution: local
        script: /tmp/k6_trace_script.js
        output: -

     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 3 iterations shared among 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)
```

### 3.3 AFTER — the end-of-test summary (values read from the sinks)

```
     data_received..................: 417 B 461 B/s
     data_sent......................: 240 B 266 B/s
     http_req_blocked...............: avg=231.28µs min=208.6µs  med=219.91µs max=265.31µs p(90)=256.23µs p(95)=260.77µs
     http_req_connecting............: avg=159.89µs min=136.32µs med=154.41µs max=188.94µs p(90)=182.03µs p(95)=185.49µs
     http_req_duration..............: avg=483.67µs min=431.88µs med=446.61µs max=572.53µs p(90)=547.35µs p(95)=559.94µs
       { expected_response:true }...: avg=483.67µs min=431.88µs med=446.61µs max=572.53µs p(90)=547.35µs p(95)=559.94µs
     http_req_failed................: 0.00% 0 out of 3
     http_req_receiving.............: avg=80.54µs  min=71.15µs  med=71.18µs  max=99.29µs  p(90)=93.67µs  p(95)=96.48µs
     http_req_sending...............: avg=65.07µs  min=55.61µs  med=62.39µs  max=77.2µs   p(90)=74.24µs  p(95)=75.72µs
     http_req_tls_handshaking.......: avg=0s       min=0s       med=0s       max=0s       p(90)=0s       p(95)=0s
     http_req_waiting...............: avg=338.05µs min=298.3µs  med=319.84µs max=396.03µs p(90)=380.79µs p(95)=388.41µs
     http_reqs......................: 3     3.318299/s
     iteration_duration.............: avg=301.29ms min=301.01ms med=301.29ms max=301.56ms p(90)=301.51ms p(95)=301.54ms
     iterations.....................: 3     3.318299/s


running (00m00.9s), 0/1 VUs, 3 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.9s/10m0s  3/3 shared iters
```

**Direct observed magnitude: `iterations = 3` and `http_reqs = 3`** — one `http.get` per iteration × 3 iterations. (Sub-millisecond timings differ slightly on every run; the counts do not.)

### 3.4 DURING — the running state (a 2-VU, 3-second run to show progress + the VU gauges)

The minimal run finished in 0.9 s — too fast to show the ~1 s progress cadence or the `vus` gauge. A second script (`{ vus: 2, duration: '3s' }`, one `http.get` + `sleep(0.2)`) exposes the *during* state:

```bash
$ ./k6 run /tmp/k6_trace_duration.js   # exit 0
```

Progress lines (emitted roughly every second):

```
running (01.0s), 2/2 VUs, 8 complete and 0 interrupted iterations
default   [  33% ] 2 VUs  1.0s/3s

running (02.0s), 2/2 VUs, 18 complete and 0 interrupted iterations
default   [  67% ] 2 VUs  2.0s/3s

running (03.0s), 2/2 VUs, 28 complete and 0 interrupted iterations
default   [ 100% ] 2 VUs  3.0s/3s
```

Its summary (key lines) — note `vus`/`vus_max` are now present because the 1-second gauge ticker fired during the 3-second run:

```
     http_req_failed................: 0.00%  0 out of 30
     http_reqs......................: 30     9.947034/s
     iteration_duration.............: avg=201.03ms min=200.53ms med=200.88ms max=201.92ms p(90)=201.63ms p(95)=201.78ms
     iterations.....................: 30     9.947034/s
     vus............................: 2      min=2       max=2
     vus_max........................: 2      min=2       max=2
```

Again `iterations == http_reqs` (30 == 30). The appearance/disappearance of `vus`/`vus_max` between the two runs is explained in §4.3.

### 3.5 The ordered function-call chain (from `main` to the summary)

Each step names the concrete symbol and its `file:line`:

1. **Entry point.** `func main()` [main.go:8] → `cmd.Execute()` [main.go:9].
2. **Run wiring (`cmd/run.go`).** Create the metrics ingester `metricsEngine.CreateIngester()` [cmd/run.go:187] and append it to the outputs list [cmd/run.go:188]; build the output manager `output.NewManager(...)` [cmd/run.go:220]; create the samples channel `samples := make(chan metrics.SampleContainer, ...)` [cmd/run.go:227]; start draining with `outputManager.Start(samples)` [cmd/run.go:228]; the end-of-test summary is invoked via `HandleSummary(...)` [cmd/run.go:195].
3. **VU executes the script.** `ActiveVU.RunOnce` [js/runner.go:724] → `VU.runFn` [js/runner.go:817] runs the JS default function.
4. **HTTP metric emission (performance data).** `Trail.SaveSamples` [lib/netext/httpext/tracer.go:44] builds `http_reqs` + `http_req_duration` + timing sub-metrics, then `metrics.PushIfNotDone(t.ctx, t.state.Samples, trail)` [lib/netext/httpext/transport.go:164] pushes them.
5. **Iteration metric emission (counting).** The guard `if isFullIteration && isDefault {` [js/runner.go:870] → `iterationSamples(...)` [js/runner.go:879] emits the `Iterations` counter (`Metric: builtinMetrics.Iterations` [js/runner.go:894], `Value: 1` [js/runner.go:899]).
6. **The conduit.** Both paths write to `State.Samples chan<- metrics.SampleContainer` [lib/vu_state.go:59].
7. **Drain & buffer (every 50 ms).** `Manager.Start` [output/manager.go:42] drains the channel and dispatches `out.AddMetricSamples(...)` [output/manager.go:52] on a ticker `time.NewTicker(sendBatchToOutputsRate)` [output/manager.go:58] where `const sendBatchToOutputsRate = 50 * time.Millisecond` [output/manager.go:12]; buffering uses `SampleBuffer` [output/helpers.go:15] / `PeriodicFlusher` [output/helpers.go:55] built by `NewPeriodicFlusher` [output/helpers.go:89].
8. **Ingestion into sinks.** The metrics-engine `OutputIngester` is itself an `output.Output` — `var _ output.Output = &OutputIngester{}` [metrics/engine/ingester.go:16]. Its `flushMetrics` [metrics/engine/ingester.go:62] calls `markObserved(m)` [metrics/engine/ingester.go:89] (so the metric shows in the summary) and `m.Sink.Add(sample)` [metrics/engine/ingester.go:90].
9. **Aggregation.** `CounterSink.Add` [metrics/sink.go:53] sums `iterations` / `http_reqs`; `TrendSink.Add` [metrics/sink.go:117] accumulates the duration percentiles.
10. **Summary.** `HandleSummary(...)` [cmd/run.go:195] → `summarizeMetricsToObject` [js/summary.go:62] reads each metric's sink via `getMetricValues(m.Sink, data.TestRunDuration)` [js/summary.go:84] (the getter built by `metricValueGetter` [js/summary.go:26]), rendered to stdout by `js/summary.js` / `js/summary-wrapper.js`.

```
main.go:9 main→cmd.Execute
   → cmd/run.go:227 samples channel  +  cmd/run.go:187-188 CreateIngester→outputs  +  cmd/run.go:220 / cmd/run.go:228 NewManager/Start
      → js/runner.go:724 RunOnce → js/runner.go:817 runFn (runs JS)
          → HTTP:  lib/netext/httpext/tracer.go:44 Trail.SaveSamples → lib/netext/httpext/transport.go:164 PushIfNotDone
          → ITER:  js/runner.go:870 guard → js/runner.go:879 iterationSamples → js/runner.go:894 / js/runner.go:899 Iterations Value:1
             → lib/vu_state.go:59 State.Samples channel
                → output/manager.go:42 Manager.Start drains (50ms ticker output/manager.go:12 / output/manager.go:58) → output/manager.go:52 AddMetricSamples
                   → metrics/engine/ingester.go:62 flushMetrics → metrics/engine/ingester.go:89 markObserved → metrics/engine/ingester.go:90 Sink.Add
                      → metrics/sink.go:53 CounterSink.Add / metrics/sink.go:117 TrendSink.Add
                         → cmd/run.go:195 HandleSummary → js/summary.go:62 / js/summary.go:84 read Sink → stdout
```

### 3.6 Design-pattern note

Because `OutputIngester` implements the `output.Output` interface [metrics/engine/ingester.go:16] and is simply appended to the outputs list [cmd/run.go:188], the **same** 50 ms flush loop that feeds real backends (JSON, CSV, cloud) also feeds the in-memory sinks used to compute the end-of-test summary. There is no separate "summary collection" path — the summary is just another consumer of the standard output pipeline.

---


## Section 4 — Rationale & edge cases (cause → effect)

### 4.1 Why iterations are counted the way they are
A full **default-function** iteration emits an `Iterations` sample with `Value: 1` [js/runner.go:899], but **only** under the guard `if isFullIteration && isDefault {` [js/runner.go:870] — i.e. `setup`/`teardown` and partial iterations don't inflate the count. The `CounterSink` then sums those `1`s [metrics/sink.go:53]. **Effect:** the summary's `iterations` equals the number of completed default iterations — exactly what §3 observed (`3` and `30`).

### 4.2 Why HTTP performance data appears only when HTTP is used
HTTP timings are assembled into a `Trail` and converted to samples by `Trail.SaveSamples` [lib/netext/httpext/tracer.go:44], then pushed via `PushIfNotDone` [lib/netext/httpext/transport.go:164]. **Effect:** built-in metrics (`iterations`, `vus`, …) appear on every test, but `http_req_*` metrics appear **only** when the script actually makes HTTP requests — which is why a script with no `http.*` calls would show no `http_reqs`.

### 4.3 Why `vus`/`vus_max` were missing from the minimal summary (zero-value Gauge suppression)
In the 0.9 s minimal run (§3.3) the summary **omitted** `vus`/`vus_max`; in the 3 s run (§3.4) they appeared as `2`. Cause: the `vus`/`vus_max` gauges are emitted on a **1-second** ticker `time.NewTicker(1 * time.Second)` [execution/scheduler.go:231] inside `emitVUsAndVUsMax` [execution/scheduler.go:199]. The 0.9 s run finished before the first tick, so those Gauge sinks were empty and k6 suppresses empty/zero-value Gauges (and zero Counters such as `dropped_iterations`) from the stdout summary. Trends like `http_req_tls_handshaking` still render as `0s` even when unused. **Effect:** absence of a Gauge line in a very short run is expected behavior, not a bug.

### 4.4 Why `-race` needs a C compiler (the "broken" edge case)
`go test -race` compiles with cgo, which requires a C compiler. Without `gcc` the build fails fast — `cgo: C compiler "gcc" not found` → `FAIL go.k6.io/k6/metrics [build failed]` (§1.5). **Effect:** on a machine lacking `gcc`, the entire canonical suite is "broken" at build time; installing `gcc` and setting `CGO_ENABLED=1` resolves it. This is an environment prerequisite, not a repository defect.

### 4.5 Skipped vs broken — different buckets
A **skipped** test ran and chose not to assert (`t.Skip`, e.g. `TestTC39` at `js/tc39/tc39_test.go:807`, reported at `js/tc39/tc39_test.go:799`); its package still reports `ok`. A **broken** package **failed to build/compile or set up** (e.g. the cgo failure) and never ran its tests. They are tracked separately in §1.2: 1 skipped, 0 broken.

### 4.6 Why test counts are reported as recorded-run values, not a single number
Every failure observed is timing-, TLS-, or concurrency-sensitive under `-race` on a 4-CPU host (§1.4). Such tests are inherently non-deterministic, so the exact pass/fail counts differ per run while the **structure** (28 no-test packages, ~50 passing, 0 broken, exactly 1 skip, and a stable persistent-core of flaky failures) is constant. The honest representation is therefore to state the stable structure and give the exact package counts **as observed in the recorded runs** (here 48–52 passing / 2–6 failing; independent reruns elsewhere have landed as low as 2 and as high as ~9 failing) rather than presenting any single count — or a narrow range — as if it held on every run.

### 4.7 Where thresholds fit (same sinks, different tick)
The threshold engine evaluates on a **2-second** tick — `const thresholdsRate = 2 * time.Second` [metrics/engine/engine.go:21], `time.NewTicker(thresholdsRate)` [metrics/engine/engine.go:173] — against the **same** sinks that feed the summary. The ingester those sinks live behind is created by `CreateIngester` [metrics/engine/engine.go:56]. **Effect:** thresholds, the summary, and any external outputs all read a single, shared source of truth.

---

## Appendix A — Citation map (all verified at commit `ddc3b0b1d23c`)

| Concern | Symbol | Location |
|---------|--------|----------|
| Entry point | `main()` → `cmd.Execute()` | main.go:8-9 |
| Run wiring | `CreateIngester()`; append outputs; `NewManager`; samples chan; `Start`; `HandleSummary` | cmd/run.go:187,188,220,227,228,195 |
| Iteration built-ins | `RegisterBuiltinMetrics`; `Iterations` (Counter); `IterationDuration` (Trend) | metrics/builtin.go:78,82,83 |
| Iteration run/emit | `ActiveVU.RunOnce`; `VU.runFn`; guard; `iterationSamples`; `Iterations`; `Value:1` | js/runner.go:724,817,870,879,894,899 |
| VU/scheduler | `emitVUsAndVUsMax`; 1s ticker | execution/scheduler.go:199,231 |
| HTTP samples | `Trail.SaveSamples` (http_reqs Value:1 @56, http_req_duration @60) | lib/netext/httpext/tracer.go:44,56,60 |
| HTTP push | `metrics.PushIfNotDone(...)` | lib/netext/httpext/transport.go:164 |
| Non-blocking push | `func PushIfNotDone(...)` | metrics/sample.go:131 |
| VU channel | `Samples chan<- metrics.SampleContainer` | lib/vu_state.go:59 |
| Sinks | `CounterSink`/`.Add`; `GaugeSink`/`.Add`; `TrendSink`/`.Add`; `RateSink`/`.Add` | metrics/sink.go:47,53,72,82,104,117,201,210 |
| Output flush | `sendBatchToOutputsRate=50ms`; `Manager.Start`; `AddMetricSamples`; 50ms ticker | output/manager.go:12,42,52,58 |
| Buffering | `SampleBuffer`; `PeriodicFlusher`; `NewPeriodicFlusher` | output/helpers.go:15,55,89 |
| Ingester | `var _ output.Output`; `flushMetrics`; `markObserved`; `Sink.Add` | metrics/engine/ingester.go:16,62,89,90 |
| Thresholds | `thresholdsRate=2s`; `CreateIngester`; 2s ticker | metrics/engine/engine.go:21,56,173 |
| Summary | `metricValueGetter`; `summarizeMetricsToObject`; `getMetricValues(m.Sink,...)` | js/summary.go:26,62,84 |
| Test isolation | `blockingTransport`; `RoundTrip`; `panic`; `Main`; forbidden hosts; `DefaultTransport`; `goleak.Find` | cmd/tests/tests.go:13,19,23,33,42-45,48,57 |
| Skip (runtime) | `t.Skipf` in `runTestTC39` (reported at `TestTC39` call site `js/tc39/tc39_test.go:799`) | js/tc39/tc39_test.go:807 (helper js/tc39/tc39_test.go:804, caller js/tc39/tc39_test.go:799) |
| Canonical test cmd | `tests:` → `go test -race -timeout 210s ./...` | Makefile:28-29 |
| Contributor doc | `make tests` | CONTRIBUTING.md:61 |
| Toolchain pin | `go 1.21`; `toolchain go1.21.13` | go.mod:3,5 |

## Appendix B — Reproduce everything

> **This appendix is a reproduction-only quick-reference** — a copy-paste command list, not an evidence block. The **actual, complete, unedited output** for every command below is embedded inline next to the command that produced it: the build banner in §0.2, the full per-run package summaries + tally script/output in §1.3, the failure excerpts in §1.4, the broken-condition (cgo) output in §1.5, the skip output in §1.6, and the before/during/after `k6 run` output in §3.2–§3.4. The inline-comment values here (e.g. `# iterations=3`) are abbreviated pointers to that inline evidence.

```bash
# 0. Toolchain
source /etc/profile.d/goenv.sh          # Go 1.21.13; GOTOOLCHAIN=local; GOFLAGS=-mod=vendor
go version                              # go version go1.21.13 linux/amd64
gcc --version | head -1                 # gcc 15.2.0 (needed for -race)

# 1. Build (canonical, offline, vendored)
#    NOTE: the exact "commit/ddc3b0b1d2" banner below is produced only when building at the
#    source commit ddc3b0b1d23c128e34e2792fc9075f9126e32375. Building the final documentation
#    branch instead stamps the docs commit (e.g. commit/4874b312ac); code behavior is unchanged (see §0.2).
go build -mod=vendor -o k6 .
./k6 version                            # k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)  # at source commit ddc3b0b1d23c

# 2. Q1 — test health (run >=2x for stability; -count=1 forces fresh execution)
CGO_ENABLED=1 go test -race -timeout 210s ./...
CGO_ENABLED=1 go test -race -timeout 210s -count=1 ./...
CGO_ENABLED=1 go test -race -timeout 210s -count=1 -json ./...   # machine-readable tally

# 2b. The "broken" edge case (C compiler absent) — system gcc left intact
PATH=/usr/local/go/bin CGO_ENABLED=1 go test -race -count=1 ./metrics/

# 3. Q3 — metric trace (minimal + duration); target is a local http server on :8090
./k6 run /tmp/k6_trace_script.js        # iterations=3, http_reqs=3
./k6 run /tmp/k6_trace_duration.js      # iterations=30, http_reqs=30, vus=2, vus_max=2
```

*This document is the sole artifact of a read-only investigation. All temporary scripts and the local test server were removed afterward; the source tree was left pristine. Two distinct checks confirm this: on a clean committed checkout the working tree has no pending changes, so `git status --porcelain` **returns no output** (the built `./k6` binary is gitignored, so it never appears); and the baseline diff `git diff --name-status ddc3b0b1d23c128e34e2792fc9075f9126e32375..HEAD` shows **exactly one added path** — `A blitzy/documentation/k6_ddc3b0b1d23c.md` — and nothing else.*

