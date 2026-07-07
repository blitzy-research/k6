# Grafana k6 v0.55.0 — Internal Runtime Behavior Investigation

This document answers a five-part investigation into the internal runtime behavior of **Grafana k6** (a Go + JavaScript load-testing engine). Every reported value, log line, metric, and API body below was captured by **building and running k6 from source** — not by reading code alone. The k6 source tree was treated as strictly read-only; all experiments ran under `/tmp`, and all temporary artifacts were deleted afterward, leaving the repository byte-for-byte unchanged.

Each section leads with a **Direct answer**, shows the exact command that produced each piece of evidence, pastes the **unedited** runtime output, and grounds every factual claim in a `[path:line]` citation resolving to the checked-out commit `ddc3b0b1d23c128e34e2792fc9075f9126e32375`.

---

## Section 0 — Environment & Canonical Build

k6 was built from source in its default, canonical configuration using a hermetic Go environment (toolchain, module cache, and build cache pinned outside the source tree). The binary was emitted to `/tmp` so the repository tree stayed pristine.

**The commit substring in the banner is the git `HEAD` commit at build time — it is not a hard-coded string.** `consts.FullVersion()` [lib/consts/consts.go:16] reads Go's VCS build stamp via `debug.ReadBuildInfo()` [lib/consts/consts.go:19], takes the `vcs.revision` setting [lib/consts/consts.go:30], and truncates it to the first **10** characters (`commitLen := 10`; `commit = s.Value[:commitLen]`) [lib/consts/consts.go:31-35] before formatting `"%s (commit/%s, %s)"` [lib/consts/consts.go:52] (`Version` is the compile-time constant `"0.55.0"` [lib/consts/consts.go:12]). The leading `k6` is the invoked binary's own name (its `argv[0]` basename), which is why the binary is built as `-o /tmp/k6`.

The investigated commit is `ddc3b0b1d23c128e34e2792fc9075f9126e32375`, whose 10-character truncation is exactly `ddc3b0b1d2`; the canonical banner is therefore reproduced by building **from that commit**. The sequence below is fully reproducible from a clean checkout and leaves the source tree untouched — the clone lives entirely under `/tmp`, and building from an out-of-tree clone makes `HEAD` equal the investigated commit so the stamp is deterministic:

```bash
# hermetic Go env — keeps GOPATH/GOCACHE out of the repo, pins the local toolchain,
# and uses the repository's vendored modules (offline-capable)
export GOTOOLCHAIN=local GOFLAGS=-mod=vendor GOPATH=/tmp/gopath GOCACHE=/tmp/gocache

# build the canonical commit in an out-of-tree clone so HEAD == the investigated commit
git clone --quiet . /tmp/k6src
cd /tmp/k6src && git checkout --quiet ddc3b0b1d23c128e34e2792fc9075f9126e32375
go build -o /tmp/k6 .        # binary lands in /tmp; the repository tree is never modified
```

The version banner (recorded verbatim):

```bash
$ /tmp/k6 version
k6 v0.55.0 (commit/ddc3b0b1d2, go1.23.4, linux/amd64)
```

> **On the commit substring and this answer document.** Because k6 stamps the *checked-out* `HEAD` commit, building directly from the branch tip that carries this answer document stamps that tip's commit substring instead (e.g. `commit/2054f7c70b`) rather than `ddc3b0b1d2`. That is expected: the answer document is Markdown under `blitzy/documentation/` and is **not** compiled into the binary, so the k6 version (`v0.55.0`), Go toolchain (`go1.23.4`), platform (`linux/amd64`), and every compiled code path are byte-for-byte identical to the canonical build above. All runtime evidence in this document was produced with the `commit/ddc3b0b1d2` binary built by the sequence shown.

**Toolchain / dependency grounding** (from `go.mod`): the module is `go.k6.io/k6` [go.mod:1] with a declared minimum of `go 1.21` [go.mod:3]; it was built with the Go **1.23.4** toolchain. Key packages exercised in this investigation, at the exact versions pinned in the manifest:

| Package | Version | `go.mod` | Used for |
|---------|---------|----------|----------|
| `google.golang.org/grpc` | v1.67.1 | [go.mod:55] | gRPC transport (Req 2) |
| `google.golang.org/protobuf` | v1.35.1 | [go.mod:56] | Protobuf runtime (Req 2) |
| `github.com/grafana/xk6-output-prometheus-remote` | v0.5.0 | [go.mod:20] | Remote-write mapping / `k6_` prefix (Req 5) |
| `github.com/sirupsen/logrus` | v1.9.3 | [go.mod:37] | Structured logger emitting the log lines (Req 1/2) |
| `github.com/grafana/sobek` | v0.0.0-20241024150027-d91f02b05e9b | [go.mod:78] | JS runtime; SharedArray deep-freeze (Req 4) |
| `github.com/spf13/afero` | v1.1.2 | [go.mod:38] | Filesystem abstraction for data loading (Req 4) |
| `github.com/fatih/color` | v1.18.0 | [go.mod:13] | End-of-test summary rendering (Req 1–3) |

**Environment constraints honored throughout:**

- **Loopback-only network.** All servers ran on localhost: the in-repo gRPC RouteGuide server (`127.0.0.1:10000`), the k6 REST API (`127.0.0.1:6565`), a Prometheus remote-write receiver (`127.0.0.1:9091`/`9092`), and an HTTP target for `http.get` (`127.0.0.1:8088`).
- **Peak RSS** was sampled from `/proc/<pid>/status` (`VmHWM`, in kB), because `/usr/bin/time -v` is unavailable (Req 4).
- **Process control** used numeric `kill <pid>` only (PIDs captured via `$!`).

Every experiment's exact invocation command is shown inline in its section. After evidence collection, all temporary scripts, logs, data files, and the compiled binary were removed from `/tmp`, and `git status --porcelain` was verified to contain only this answer document.

---

## Section 1 — Requirement 1: Ramping-VUs orchestration under SIGINT

**Direct answer:** On a manual `SIGINT`, currently active VUs are **terminated mid-execution — they are NOT allowed to finish their current iteration.** The graceful-finish window (`gracefulStop` / `gracefulRampDown`) applies only at a scenario's **natural duration end**, never on a manual interrupt. The decisive runtime evidence is the end-of-run progress counter, which reports the in-flight iterations as **interrupted** (here: `6 interrupted iterations` for the 6 VUs that were mid-`sleep(3)` when the signal arrived), corroborated by the source doc-comment that "iterations are immediately stopped" on manual interrupt [lib/executor/base_config.go:95-96]. A single `SIGINT` performs a graceful abort (exit code **105**); a second `SIGINT` triggers an immediate hard stop.

### Reproduction script

`ramping-vus` rising to 6 VUs, with iterations (`sleep(3)`) longer than the observation window so VUs are guaranteed to be mid-iteration when the signal is delivered. `gracefulRampDown`/`gracefulStop` are overridden to short values (the field default is `30s` [lib/executor/ramping_vus.go:52]):

```javascript
// /tmp/k6inv/req1.js
import { sleep } from 'k6';
export const options = {
  scenarios: { ramp: { executor: 'ramping-vus', startVUs: 0,
    stages: [{ duration: '2s', target: 6 }, { duration: '20s', target: 6 }],
    gracefulRampDown: '2s', gracefulStop: '3s' } },
};
export default function () {
  console.log(`ITER_START vu=${__VU} iter=${__ITER}`);
  sleep(3);
  console.log(`ITER_END vu=${__VU} iter=${__ITER}`);
}
```

The `ramping-vus` executor is configured by `RampingVUsConfig`, whose `GracefulRampDown` field is declared at [lib/executor/ramping_vus.go:44].

### Invocation (single SIGINT)

STDOUT (progress counter + end-of-test summary) and STDERR (logrus logs + `console.log` markers) are captured to **separate files from one and the same run**, so every figure below is mutually consistent:

```bash
NO_COLOR=1 /tmp/k6 run --verbose --no-color /tmp/k6inv/req1.js \
  >/tmp/k6inv/req1.out 2>/tmp/k6inv/req1.err &   # STDOUT->req1.out (progress+summary); STDERR->req1.err (logs+markers)
K6PID=$!
sleep 5
kill -INT "$K6PID"        # single numeric SIGINT at ~t=5s (6 VUs, each mid-sleep(3))
wait "$K6PID"; echo "exit=$?"
```

Result: `exit=105`. All evidence in (a), (b) below is from this single run (`req1.out` / `req1.err`).

### (a) Signal-pipeline log lines (the three key lines extracted from the `--verbose` STDERR stream of the same run)

`--verbose` emits many `level=debug` lines; the command below extracts exactly the three signal-pipeline lines (unedited; logrus `TextFormatter` → STDERR). The third alternative is anchored on `level=error` so it matches only the error-level abort line itself, not the redundant `level=debug` lines that merely reference the same abort message in a `msg=` prose string or an `error=` field:

```bash
$ grep -E 'msg="Trapping interrupt signals|msg="Stopping k6 in response to signal|level=error msg="test run was aborted' /tmp/k6inv/req1.err
time="2026-07-06T23:29:58Z" level=debug msg="Trapping interrupt signals so k6 can handle them gracefully..."
time="2026-07-06T23:30:03Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-06T23:30:03Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

- The interrupt-trap line is emitted by `handleTestAbortSignals` at [cmd/common.go:98].
- `"Stopping k6 in response to signal..."` is emitted by the `gracefulStop` closure at [cmd/run.go:350].
- The abort error string is formatted at [cmd/run.go:354] (`fmt.Errorf("test run was aborted because k6 received a '%s' signal", sig)`), tagged with `exitcodes.ExternalAbort`.
- **Exit code 105** is `ExternalAbort`, defined at [errext/exitcodes/codes.go:41].

### (b) DECISIVE evidence — the progress iteration counter (before / during / after, unedited from STDOUT)

```
running (01.0s), 2/6 VUs, 0 complete and 0 interrupted iterations
running (02.0s), 5/6 VUs, 0 complete and 0 interrupted iterations
running (03.0s), 6/6 VUs, 0 complete and 0 interrupted iterations
running (04.0s), 6/6 VUs, 2 complete and 0 interrupted iterations
running (05.0s), 0/6 VUs, 5 complete and 6 interrupted iterations   ← at SIGINT: 6 in-flight iterations INTERRUPTED
```

At the instant of the signal (t≈5s), the counter jumps to **`5 complete and 6 interrupted iterations`**: the 6 VUs that were active (each mid-`sleep(3)`) had their iterations **interrupted**, not finished. This "N complete and M interrupted iterations" line is built by the scheduler's progress formatter at [execution/scheduler.go:156] (format string `"%s, <vus>/<vus> VUs, %d complete and %d interrupted iterations"`; the interrupted count comes from `GetPartialIterationCount()` [execution/scheduler.go:158]).

> Note: the `scenarios:` header printed at the top of a run is a different line, rendered at [cmd/ui.go:149-150]; it is **not** the interrupted-iteration counter.

The end-of-test summary from the **same run** (unedited from `req1.out`) reports only the **complete** count, which equals the progress counter's `5 complete`:

```
     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=3s min=3s med=3s max=3s p(90)=3s p(95)=3s
     iterations...........: 5   1.004826/s
     vus..................: 6   min=3      max=6
     vus_max..............: 6   min=6      max=6
```

Cross-checking the `console.log` markers from the **same run** (`req1.err`) independently confirms mid-iteration termination — iterations that started (`ITER_START`) but were killed mid-`sleep(3)` never reached their `ITER_END`:

```bash
$ grep -c ITER_START /tmp/k6inv/req1.err
11
$ grep -c ITER_END /tmp/k6inv/req1.err
5
```

These figures tie out exactly for this run: `ITER_END` (5) = `iterations` summary (5) = the progress counter's `5 complete`; and `ITER_START` (11) − `ITER_END` (5) = **6** = the `6 interrupted iterations` the engine reports — i.e. the 6 VUs caught mid-`sleep(3)` began an iteration (`ITER_START`) but were terminated before returning.

**Stability across runs.** The engine counter is the authoritative, stable figure. A second run (`req1_run2.*`) reproduced the identical engine result — `running (05.0s), 0/6 VUs, 5 complete and 6 interrupted iterations`, summary `iterations: 5`, and `ITER_START` `11` — with only the `ITER_END` marker count differing (`5` in run 1, `6` in run 2). That single-marker variance is a `console.log`-flush timing artifact: `console.log('ITER_END')` fires at the JS-function tail a hair before the Go runner finalizes the iteration-complete accounting, so when the abort lands in that window the marker can print for an iteration the engine still tallies as *interrupted*. The decisive facts — `5 complete`, **`6 interrupted`**, and `ITER_START` (11) far exceeding the completed work — are stable across both runs and prove active VUs are terminated mid-execution rather than drained.

### (c) Root-cause corroboration from source

- The authoritative doc-comment on `GetGracefulStop` states the graceful window is honored only "at the end of the normal executor duration", and: "Of course, that doesn't count when the user manually interrupts the test, then iterations are immediately stopped" [lib/executor/base_config.go:95-96].
- Interrupted iterations are tallied by `executionState.AddInterruptedIterations(1)`, called from the iteration runner both when the context is cancelled [lib/executor/helpers.go:117] and on a handled interrupt [lib/executor/helpers.go:122].
- The **contrasting** normal-duration path only fires at natural scenario end: after the regular duration context is done [lib/executor/helpers.go:191], it logs `"Regular duration is done, waiting for iterations to gracefully finish"` [lib/executor/helpers.go:195-196]. This branch is **not** taken on a manual abort, which is why in-flight iterations are killed rather than drained.

### (d) Second-SIGINT hard stop (secondary path)

Delivering a **second** `SIGINT` while k6 is still shutting down from the first exercises the hard-stop path (the OS-signal channel is buffered with capacity 2 at [cmd/common.go:99]). A naive `kill -INT $PID ; kill -INT $PID` is timing-fragile: on a manual interrupt k6 kills its iterations immediately and returns fast, so the second signal can miss the brief shutdown window; and the kernel coalesces two identical pending signals into one if the process has not yet handled the first. A reliable pattern delivers the first signal, then **tightly re-sends `SIGINT` with no gap until the process exits**, guaranteeing a distinct second signal lands in the buffered channel during the shutdown window:

```bash
NO_COLOR=1 /tmp/k6 run --verbose --no-color /tmp/k6inv/req1.js 2>/tmp/k6inv/req1_hard.err &
K6PID=$!
sleep 5
kill -INT "$K6PID"                                 # 1st SIGINT -> graceful stop
while kill -INT "$K6PID" 2>/dev/null; do :; done   # tight burst -> distinct 2nd SIGINT lands during shutdown -> hard stop
wait "$K6PID"; echo "exit=$?"
```

Result: `exit=105`. This pattern reproduced the hard stop **8 out of 8** attempts (the fragile sequential form was intermittent). Unedited log — both handlers fire (extracted from `req1_hard.err`):

```bash
$ grep -E "Stopping k6 in response to signal|Aborting k6 in response to signal" /tmp/k6inv/req1_hard.err
time="2026-07-06T23:34:35Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-06T23:34:35Z" level=error msg="Aborting k6 in response to signal" sig=interrupt
```

The first signal runs the `gracefulStop` closure; the second is read by the second `select` in `handleTestAbortSignals` [cmd/common.go:111], which invokes the `onHardStop` closure — `"Aborting k6 in response to signal"` at [cmd/run.go:360] — and then immediately calls `gs.OSExit(int(exitcodes.ExternalAbort))` at [cmd/common.go:118]. Exit code: **105** ([errext/exitcodes/codes.go:41]).

---

## Section 2 — Requirement 2: gRPC server-streaming interruption + received-message count

**Direct answer:** For the deterministic completion window, `grpc_streams_msgs_received` = **200** (2 iterations × 100 features per stream), stable across runs. An **interrupted** run yields a labeled **PARTIAL** snapshot — observed **29** for a ~3s interrupt (timing-dependent). The interruption is visible in the logs as the stream-cancellation debug line `"stream is cancelled/finished"` carrying `error="canceled by client (k6)"`; the **same** debug line fires on normal completion but with `error=EOF`.

### Server setup (repo untouched — copy out, don't edit in place)

`examples/grpc_server` is a **separate** Go module with a `replace go.k6.io/k6 => ../../` directive, so it was copied out of the tree and its replace target rewritten to an absolute path before building — leaving the tracked `go.mod`/`go.sum` unchanged:

```bash
REPO=$(pwd)                        # run this from the repository root; $REPO is its absolute path
mkdir -p /tmp/grpcsrv && cp -r "$REPO"/examples/grpc_server/* /tmp/grpcsrv/
cd /tmp/grpcsrv && go mod edit -replace go.k6.io/k6="$REPO"
GOFLAGS= go build -mod=mod -o /tmp/grpcserver . && /tmp/grpcserver -port 10000 &
# server log: "gRPC server starting on localhost:10000"
```

The server registers `FeatureExplorer` at [examples/grpc_server/main.go:80], listens on `localhost:<port>` (default port `10000`) at [examples/grpc_server/main.go:59], and logs its startup at [examples/grpc_server/main.go:57]. Its `ListFeatures` server-streaming RPC — declared in the proto as `rpc ListFeatures(Rectangle) returns (stream Feature)` [lib/testutils/grpcservice/route_guide.proto:38] — throttles one feature per `100ms` via `time.Sleep(100 * time.Millisecond)` [lib/testutils/grpcservice/service.go:62] before each `stream.Send` [lib/testutils/grpcservice/service.go:63]. The embedded dataset has 100 features inside the reference rectangle, so a full stream takes ~10s and yields exactly 100 messages.

### (a) Deterministic completion window → 200

`shared-iterations` with `vus:2 iterations:2`; each iteration opens a `Stream` on `main.FeatureExplorer/ListFeatures`, sends the reference `Rectangle`, and `sleep(12)`s so the full 100-message stream completes:

```javascript
// /tmp/k6inv/req2_complete.js
import { Client, Stream } from 'k6/net/grpc';
import { sleep } from 'k6';
export const options = {
  scenarios: { s: { executor: 'shared-iterations', vus: 2, iterations: 2, maxDuration: '60s' } },
};
const client = new Client();
client.load([], 'route_guide.proto');
export default function () {
  client.connect('127.0.0.1:10000', { plaintext: true });
  const stream = new Stream(client, 'main.FeatureExplorer/ListFeatures', null);
  let n = 0;
  stream.on('data', () => { n++; });
  stream.on('end', () => { client.close(); console.log(`STREAM_END received=${n}`); });
  stream.on('error', (e) => { console.log('STREAM_ERR ' + JSON.stringify(e)); });
  stream.write({ lo: { latitude: 400000000, longitude: -750000000 },
                 hi: { latitude: 420000000, longitude: -730000000 } });
  sleep(12);
}
```

```bash
NO_COLOR=1 /tmp/k6 run --verbose --no-color /tmp/k6inv/req2_complete.js
```

Unedited summary excerpt (exit code 0):

```
     grpc_streams.................: 2      0.166619/s
     grpc_streams_msgs_received...: 200    16.661935/s     ← DETERMINISTIC ANSWER = 200
     grpc_streams_msgs_sent.......: 2      0.166619/s
```

Normal-completion cancellation log (unedited; fires once per stream, so twice here — `error=EOF`):

```
time="2026-07-06T23:38:26Z" level=debug msg="stream is cancelled/finished" error=EOF streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-06T23:38:26Z" level=debug msg="stream is cancelled/finished" error=EOF streamMethod=/main.FeatureExplorer/ListFeatures
```

The console markers confirm 2 × 100 = 200:

```
time="2026-07-06T23:38:28Z" level=info msg="STREAM_END received=100" source=console
time="2026-07-06T23:38:28Z" level=info msg="STREAM_END received=100" source=console
```

`grpc_streams_msgs_received` is registered as a **Counter** at [js/modules/k6/grpc/metrics.go:25]. The value was stable across two runs (RUN1 `200 16.661935/s`, RUN2 `200 16.664145/s`); the count is deterministically **200**, only the sub-0.01/s rate varies.

### (b) Interruption case — 30ms graceful ramp-down window → PARTIAL

`ramping-vus` with a single VU, a `30s` up-stage, and a **`30ms` graceful ramp-down / graceful stop**; the single long stream is interrupted with a `SIGINT` at ~t=3s:

```javascript
// /tmp/k6inv/req2_interrupt.js — options block
export const options = {
  scenarios: { s: { executor: 'ramping-vus', startVUs: 1,
    stages: [{ duration: '30s', target: 1 }],
    gracefulRampDown: '30ms', gracefulStop: '30ms' } },
};
// default fn: connect, open Stream on main.FeatureExplorer/ListFeatures, write the Rectangle, sleep(30)
```

```bash
NO_COLOR=1 /tmp/k6 run --verbose --no-color /tmp/k6inv/req2_interrupt.js \
  >/tmp/k6inv/req2i.out 2>/tmp/k6inv/req2i.err &   # STDOUT->req2i.out (summary); STDERR->req2i.err (logs)
K6PID=$!
sleep 3
kill -INT "$K6PID"
wait "$K6PID"; echo "exit=$?"
```

Unedited logs (exit code 105) — the four key lines extracted from the `--verbose` STDERR stream of the same run:

```bash
$ grep -E "Stopping k6 in response to signal|stream is cancelled/finished|STREAM_ERR|test run was aborted because" /tmp/k6inv/req2i.err
time="2026-07-06T23:39:14Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-06T23:39:14Z" level=debug msg="stream is cancelled/finished" error="canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-06T23:39:14Z" level=info msg="STREAM_ERR {\"code\":2,\"details\":[],\"message\":\"canceled by client (k6)\"}" source=console
time="2026-07-06T23:39:14Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

Unedited summary line (from `req2i.out`):

```
     grpc_streams_msgs_received...: 29     9.746054/s      ← PARTIAL / timing-dependent
```

This value is **PARTIAL** and timing-dependent (it reflects however many of the 100ms-throttled features arrived before the interrupt). For a ~3s interrupt it was **29** in both runs (RUN1 `29 9.746054/s`, RUN2 `29 9.748080/s`) — approximately 3s / 100ms minus connection setup — but it must be read as a snapshot, not a deterministic total.

### (c) Key distinction

The **same** debug log message `"stream is cancelled/finished"` fires in both the normal and the interrupted case; only the `error=` field differs — `EOF` on normal completion vs `"canceled by client (k6)"` on interrupt. That debug call is at [js/modules/k6/grpc/stream.go:201], guarded by `isRegularClosing(err)` at [js/modules/k6/grpc/stream.go:200]; the predicate treats both `io.EOF` and `grpcext.ErrCanceled` as regular closings [js/modules/k6/grpc/stream.go:216-218]. The literal string `"canceled by client (k6)"` originates from `var ErrCanceled = errors.New("canceled by client (k6)")` at [lib/netext/grpcext/stream.go:29]. The `grpc_streams_*` lines are part of the end-of-test text summary, which is rendered by the embedded-JS summarizer: `summarizeMetrics` [js/summary.js:243] emits one line per metric [js/summary.js:378] inside `generateTextSummary` [js/summary.js:384], and the result is written to STDOUT by `handleSummaryResult` [cmd/run.go:508]. (The `scenarios:` header at the top of a run — a different line — is the one rendered at [cmd/ui.go:149-150].)

---

## Section 3 — Requirement 3: dropped_iterations reported via the REST API

**Direct answer:** `dropped_iterations` = **1850** for the exercised configuration, and this value was **obtained by querying the k6 REST API v1** — `GET /v1/metrics/dropped_iterations` returned `"count":1850` in its JSON:API body, not merely the end-of-test summary. The count is deterministic across runs.

### Mechanism

For the `shared-iterations` executor, dropped iterations = `totalIters − attemptedIters`, pushed in a deferred function **after** `activeVUs.Wait()` at [lib/executor/shared_iterations.go:218-227] (the `DroppedIterations` metric is referenced at [lib/executor/shared_iterations.go:222] and the value `float64(totalIters - attemptedIters)` at [lib/executor/shared_iterations.go:225]). `attemptedIters` is incremented atomically when an iteration **starts**, at [lib/executor/shared_iterations.go:254]. The metric's name constant is `DroppedIterationsName = "dropped_iterations"` [metrics/builtin.go:10], registered as a Counter at [metrics/builtin.go:84].

### Reproduction

`shared-iterations`, `vus:150`, `iterations:2000`, `maxDuration:'3s'`, default fn `sleep(60)`. Because each `sleep(60)` far exceeds `maxDuration`, each of the 150 VUs starts exactly one iteration → `attempted = 150` → `dropped = 2000 − 150 = 1850`. `--linger` keeps the REST API queryable after the run completes:

```javascript
// /tmp/k6inv/req3.js
import { sleep } from 'k6';
export const options = {
  scenarios: { s: { executor: 'shared-iterations', vus: 150, iterations: 2000, maxDuration: '3s' } },
};
export default function () { sleep(60); }
```

```bash
NO_COLOR=1 /tmp/k6 run --linger --address 127.0.0.1:6565 --no-color /tmp/k6inv/req3.js >/tmp/k6inv/req3.log 2>&1 &
K6PID=$!
```

### Runtime proof from the API (the required evidence)

**Before** the executor emits the metric (queried ~1s into the run), the single-metric endpoint returns **404** — proving the value is not yet present:

```bash
$ curl -s http://127.0.0.1:6565/v1/metrics/dropped_iterations
{"errors":[{"status":"404","title":"Not Found","detail":"No metric with that ID was found"}]}
```

**After** the run finishes (with `--linger` holding the API up), the single-metric endpoint returns the value — `GET /v1/metrics/{id}` is routed at [api/v1/routes.go:31] to `handleGetMetric` [api/v1/metric_routes.go:27]:

```bash
$ curl -s http://127.0.0.1:6565/v1/metrics/dropped_iterations
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":1850,"rate":56.059533852371416}}}}
```

The all-metrics endpoint — `GET /v1/metrics`, routed at [api/v1/routes.go:23] to `handleGetMetrics` [api/v1/metric_routes.go:9] — includes the same object (`"count":1850`):

```bash
$ curl -s http://127.0.0.1:6565/v1/metrics
{"data":[{"type":"metrics","id":"data_received","attributes":{"type":"counter","contains":"data","tainted":null,"sample":{"count":0,"rate":0}}},{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":1850,"rate":56.059533852371416}}},{"type":"metrics","id":"vus","attributes":{"type":"gauge","contains":"default","tainted":null,"sample":{"value":0}}},{"type":"metrics","id":"vus_max","attributes":{"type":"gauge","contains":"default","tainted":null,"sample":{"value":150}}},{"type":"metrics","id":"data_sent","attributes":{"type":"counter","contains":"data","tainted":null,"sample":{"count":0,"rate":0}}}]}
```

Confirming `--linger` kept the server up (the run has ended — `"running":false`):

```bash
$ curl -s http://127.0.0.1:6565/v1/status
{"data":{"type":"status","id":"default","attributes":{"status":9,"paused":false,"vus":0,"vus-max":150,"stopped":false,"running":false,"tainted":false}}}
```

### Cross-check against the summary and progress counter

The end-of-test summary line equals the API `sample.count`:

```
     dropped_iterations...: 1850 56.059534/s
```

The progress counter independently corroborates the arithmetic — 150 iterations were attempted and all were interrupted, out of a 2000 budget (`0000/2000 shared iters`), so dropped = 2000 − 150 = 1850:

```
running (33.0s), 000/150 VUs, 0 complete and 150 interrupted iterations
```

(The run ends at ~33s = `maxDuration` 3s + the default `gracefulStop` 30s, at which point the still-sleeping VUs are killed.)

### Stability

The count is deterministically **1850** across runs; only the rate differs slightly. RUN1: `count=1850, rate=56.059533852371416` (summary `56.059534/s`). RUN2: `count=1850, rate=56.05863596138503` (summary `56.058636/s`). The sub-0.01/s rate difference is reported rather than smoothed away.

---

## Section 4 — Requirement 4: SharedArray memory footprint + root cause

**Direct answer:** With `SharedArray`, the process memory footprint stays **~constant as the number of VUs increases — each VU does NOT create its own copy of the data.** A per-VU parse control (top-level `JSON.parse(open(...))`) DOES copy the data once per VU, growing roughly linearly. **Root cause:** there is a single shared backing store (`[]string`) held once in Go memory and referenced by **pointer** from every VU's module instance; elements are parsed lazily and deep-frozen only when accessed.

### Setup

A ~12.8 MB-class data file was generated deterministically (fixed RNG seed) as a 60,000-row JSON array of six-field user records:

```python
# gen_data.py — deterministic ~12.8 MB fixture (60,000 rows)
import json, random, string
random.seed(42)
def rs(n): return ''.join(random.choice(string.ascii_letters) for _ in range(n))
rows = [{
    "id": i,
    "name": f"User {i:06d}",
    "email": f"user{i:06d}@example.com",
    "phone": f"+1-555-{100 + i // 1000}-{1000 + i % 1000}",
    "company": rs(18) + " Inc",
    "bio": rs(24) + " " + rs(48),
} for i in range(60000)]
json.dump(rows, open("/tmp/k6inv/data.json", "w"))
```

```console
$ python3 gen_data.py && stat -c '%s bytes' /tmp/k6inv/data.json && \
    python3 -c "import json;print(len(json.load(open('/tmp/k6inv/data.json'))),'rows')"
13308890 bytes
60000 rows
```

The fixture is **13,308,890 bytes (12.692 MiB, 60,000 rows, ≈221.8 B/row)**. Peak RSS was sampled as `VmHWM` from `/proc/<pid>/status` while each run executed (k6 runs as a single process — VUs are goroutines — so process `VmHWM` captures the total peak):

```bash
#!/bin/bash
# measure.sh <script> <vus> <duration>: run k6, poll /proc/<pid>/status for peak VmHWM
SCRIPT="$1"; VUS="$2"; DUR="$3"
NO_COLOR=1 /tmp/k6 run -u "$VUS" -d "$DUR" -q --no-color "$SCRIPT" >/dev/null 2>&1 &
K6PID=$!
PEAK=0
while kill -0 "$K6PID" 2>/dev/null; do
  HWM=$(awk '/VmHWM/{print $2}' /proc/$K6PID/status 2>/dev/null)
  [ -n "$HWM" ] && [ "$HWM" -gt "$PEAK" ] && PEAK=$HWM
  sleep 0.1
done
wait "$K6PID" 2>/dev/null
echo "VUs=$VUS peak_VmHWM=${PEAK} kB"
```

The two scripts differ only in how the data is loaded:

```javascript
// req4_shared.js — single shared backing store
import { SharedArray } from 'k6/data';
import { sleep } from 'k6';
const data = new SharedArray('users', function () { return JSON.parse(open('/tmp/k6inv/data.json')); });
export default function () { const row = data[(__VU + __ITER) % data.length]; if (!row) throw new Error('no row'); sleep(1); }
```

```javascript
// req4_copy.js — CONTROL: top-level parse runs once PER VU init → one full copy per VU
import { sleep } from 'k6';
const data = JSON.parse(open('/tmp/k6inv/data.json'));
export default function () { const row = data[(__VU + __ITER) % data.length]; if (!row) throw new Error('no row'); sleep(1); }
```

### Measured peak `VmHWM` (test-script output)

Each mode was measured at increasing VU counts, two runs each (8 s per run):

```bash
for vus in 1 100 300; do for run in 1 2; do
  echo "SharedArray RUN${run} $(./measure.sh req4_shared.js "$vus" 8s)"
done; done
for vus in 1 5 10; do for run in 1 2; do
  echo "per-VU-copy RUN${run} $(./measure.sh req4_copy.js "$vus" 8s)"
done; done
```

Complete, unedited output:

```text
### SharedArray (req4_shared.js) ###
SharedArray RUN1 VUs=1 peak_VmHWM=203372 kB
SharedArray RUN2 VUs=1 peak_VmHWM=202060 kB
SharedArray RUN1 VUs=100 peak_VmHWM=205760 kB
SharedArray RUN2 VUs=100 peak_VmHWM=211320 kB
SharedArray RUN1 VUs=300 peak_VmHWM=218160 kB
SharedArray RUN2 VUs=300 peak_VmHWM=221760 kB
### per-VU-copy CONTROL (req4_copy.js) ###
per-VU-copy RUN1 VUs=1 peak_VmHWM=200648 kB
per-VU-copy RUN2 VUs=1 peak_VmHWM=219508 kB
per-VU-copy RUN1 VUs=5 peak_VmHWM=594124 kB
per-VU-copy RUN2 VUs=5 peak_VmHWM=685232 kB
per-VU-copy RUN1 VUs=10 peak_VmHWM=1078364 kB
per-VU-copy RUN2 VUs=10 peak_VmHWM=1090428 kB
```

Summarized (kB → MB at ÷1024):

| Mode | VUs | RUN1 | RUN2 |
|------|-----|------|------|
| SharedArray | 1 | 203372 kB (~199 MB) | 202060 kB (~197 MB) |
| SharedArray | 100 | 205760 kB (~201 MB) | 211320 kB (~206 MB) |
| SharedArray | 300 | 218160 kB (~213 MB) | 221760 kB (~217 MB) |
| per-VU-copy | 1 | 200648 kB (~196 MB) | 219508 kB (~214 MB) |
| per-VU-copy | 5 | 594124 kB (~580 MB) | 685232 kB (~669 MB) |
| per-VU-copy | 10 | 1078364 kB (~1053 MB) | 1090428 kB (~1065 MB) |

**Interpretation:** `SharedArray` is **flat** from 1 → 300 VUs (~197–199 MB → ~213–217 MB; a small, sub-linear increase of ≈17 MB across all 300 VUs, attributable to per-VU goroutine/JS-runtime overhead — not to copying the dataset). The per-VU-copy control is **linear** — roughly +95 MB per VU (~196–214 MB at 1 VU → ~1053–1065 MB at 10 VUs). `SharedArray` at **300** VUs (~213–217 MB) uses far less than the control at merely **10** VUs (~1053–1065 MB); extrapolating the control to 300 VUs would be on the order of tens of GB (infeasible) — which is precisely why `SharedArray` exists. The behavior was stable across two runs.

### Root cause (code-confirmed)

- **One backing store.** The data is held once as a `[]string`: `type sharedArray struct { arr []string }` [js/modules/k6/data/share.go:10-12].
- **Shared by pointer per VU.** The root module holds a single `shared sharedArrays` [js/modules/k6/data/data.go:21-23]; each VU's instance holds a **pointer** `shared *sharedArrays` [js/modules/k6/data/data.go:26-29]. `NewModuleInstance` — invoked once per VU — returns `&Data{ vu: vu, shared: &rm.shared }` [js/modules/k6/data/data.go:53-57], and the shared pointer `&rm.shared` is taken at [js/modules/k6/data/data.go:56]. Every VU therefore references the **same** store rather than allocating its own.
- **Lazy, copy-on-read into JS only.** `wrappedSharedArray.Get(index)` parses just the requested element on demand via `JSON.parse` [js/modules/k6/data/share.go:48] and deep-freezes the result [js/modules/k6/data/share.go:53] (function body at [js/modules/k6/data/share.go:44-58]). The raw rows live once in Go memory as strings; only individually accessed elements are materialized on a VU's JS heap, so the full dataset is never duplicated N times.

---

## Section 5 — Requirement 5: Prometheus output metric-name integrity

**Direct answer:** Exported series preserve name integrity. Every `__name__` equals `"k6_"` + the original metric name carried **verbatim** + an optional per-type/stat suffix. There is no character mangling and no truncation; underscores are preserved, and **custom** metrics receive the same `k6_` prefix as built-ins.

### Setup

The built-in `experimental-prometheus-rw` output was run against a minimal loopback remote-write receiver. To guarantee the payload is decoded exactly as k6 encodes it, the receiver is a small **Go** program that imports the **same** libraries k6 uses to *encode* — the `klauspost/compress` snappy **block** codec, the Prometheus `prompb.WriteRequest` protobuf, and `google.golang.org/protobuf` — so it decompresses the body, unmarshals the `WriteRequest`, records every distinct `__name__` label, and dumps the sorted unique set on `SIGINT`:

```go
// /tmp/rwrecv/main.go — remote-write receiver using k6's exact encode libraries
// Standalone Prometheus remote-write receiver for observing k6's exported
// series names. It uses the SAME libraries k6 uses to ENCODE (klauspost snappy
// block codec + prometheus prompb + protobuf), guaranteeing correct decoding.
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"os/signal"
	"sort"
	"sync"
	"syscall"

	prompb "buf.build/gen/go/prometheus/prometheus/protocolbuffers/go"
	"github.com/klauspost/compress/snappy"
	"google.golang.org/protobuf/proto"
)

var (
	mu          sync.Mutex
	names       = map[string]bool{}
	examples    = map[string]string{}
	reqCount    int
	seriesCount int
)

func handler(w http.ResponseWriter, r *http.Request) {
	body, err := io.ReadAll(r.Body)
	if err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}
	dec, err := snappy.Decode(nil, body) // snappy BLOCK decode (matches k6's snappy.Encode)
	if err != nil {
		http.Error(w, "snappy: "+err.Error(), http.StatusBadRequest)
		return
	}
	var wr prompb.WriteRequest
	if err := proto.Unmarshal(dec, &wr); err != nil {
		http.Error(w, "proto: "+err.Error(), http.StatusBadRequest)
		return
	}
	mu.Lock()
	reqCount++
	for _, ts := range wr.GetTimeseries() {
		seriesCount++
		labels := map[string]string{}
		var name string
		for _, l := range ts.GetLabels() {
			labels[l.GetName()] = l.GetValue()
			if l.GetName() == "__name__" {
				name = l.GetValue()
			}
		}
		if name != "" {
			names[name] = true
			if _, ok := examples[name]; !ok {
				b, _ := json.Marshal(labels)
				examples[name] = string(b)
			}
		}
	}
	mu.Unlock()
	w.WriteHeader(http.StatusNoContent) // 204: standard remote-write ack
}

func main() {
	addr := "127.0.0.1:9091"
	if len(os.Args) > 1 {
		addr = os.Args[1]
	}
	mux := http.NewServeMux()
	mux.HandleFunc("/api/v1/write", handler)
	srv := &http.Server{Addr: addr, Handler: mux}
	go func() {
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			fmt.Fprintln(os.Stderr, "listen error:", err)
			os.Exit(1)
		}
	}()
	fmt.Printf("remote-write receiver listening on http://%s/api/v1/write\n", addr)

	sig := make(chan os.Signal, 1)
	signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)
	<-sig

	mu.Lock()
	defer mu.Unlock()
	uniq := make([]string, 0, len(names))
	for n := range names {
		uniq = append(uniq, n)
	}
	sort.Strings(uniq)
	fmt.Printf("\n=== RECEIVER SUMMARY: requests=%d series_samples=%d unique __name__=%d ===\n", reqCount, seriesCount, len(uniq))
	for _, n := range uniq {
		fmt.Println(n)
	}
	fmt.Println("--- full example label sets ---")
	for _, want := range []string{"k6_my_custom_counter_total", "k6_my_custom_trend_p99", "k6_http_req_duration_p99", "k6_vus", "k6_http_req_failed_rate"} {
		if ex, ok := examples[want]; ok {
			fmt.Printf("EXAMPLE %s => %s\n", want, ex)
		}
	}
}
```

Its `go.mod` pins the exact encode-side library versions (matching k6's own — see [go.mod:20] and the vendored tree), so decoding is byte-faithful:

```text
module rwrecv

go 1.23.4

require (
	buf.build/gen/go/prometheus/prometheus/protocolbuffers/go v1.31.0-20230627135113-9a12bc2590d2.1
	github.com/klauspost/compress v1.17.11
	google.golang.org/protobuf v1.35.1
)
```

```bash
# Build the receiver offline from the warm module cache
cd /tmp/rwrecv
GOPROXY=off GOSUMDB=off GOMODCACHE=/root/go/pkg/mod GOFLAGS=-mod=mod GOTOOLCHAIN=local \
  go build -o /tmp/rwreceiver .
```

The k6 script defines a custom `Counter('my_custom_counter')` and a custom `Trend('my_custom_trend')`, plus an `http.get(...)` to a loopback target to generate built-in HTTP metrics:

```javascript
// /tmp/k6inv/req5.js
import http from 'k6/http';
import { Counter, Trend } from 'k6/metrics';
import { sleep } from 'k6';
const myCounter = new Counter('my_custom_counter');
const myTrend = new Trend('my_custom_trend');
export const options = { vus: 3, duration: '12s' };
export default function () {
  http.get('http://127.0.0.1:8088/');
  myCounter.add(1);
  myTrend.add(Math.random() * 100);
  sleep(1);
}
```

```bash
# 1) start the receiver
/tmp/rwreceiver 127.0.0.1:9091 &        # prints: remote-write receiver listening on ...
# 2) run k6 with the experimental-prometheus-rw output pointed at it
K6_PROMETHEUS_RW_SERVER_URL="http://127.0.0.1:9091/api/v1/write" \
  NO_COLOR=1 /tmp/k6 run -o experimental-prometheus-rw --no-color /tmp/k6inv/req5.js
# 3) after k6 exits, SIGINT the receiver to dump the sorted unique __name__ set
kill -INT %1
```

k6 confirmed the output was active (exit code 0):

```
        output: Prometheus remote write (http://127.0.0.1:9091/api/v1/write)
```

The receiver then printed its complete, **unedited** dump (RUN1 shown; the 17-name set was byte-identical on RUN2):

```text
remote-write receiver listening on http://127.0.0.1:9091/api/v1/write

=== RECEIVER SUMMARY: requests=3 series_samples=51 unique __name__=17 ===
k6_data_received_total
k6_data_sent_total
k6_http_req_blocked_p99
k6_http_req_connecting_p99
k6_http_req_duration_p99
k6_http_req_failed_rate
k6_http_req_receiving_p99
k6_http_req_sending_p99
k6_http_req_tls_handshaking_p99
k6_http_req_waiting_p99
k6_http_reqs_total
k6_iteration_duration_p99
k6_iterations_total
k6_my_custom_counter_total
k6_my_custom_trend_p99
k6_vus
k6_vus_max
--- full example label sets ---
EXAMPLE k6_my_custom_counter_total => {"__name__":"k6_my_custom_counter_total","scenario":"default"}
EXAMPLE k6_my_custom_trend_p99 => {"__name__":"k6_my_custom_trend_p99","scenario":"default"}
EXAMPLE k6_http_req_duration_p99 => {"__name__":"k6_http_req_duration_p99","expected_response":"true","method":"GET","name":"http://127.0.0.1:8088/","proto":"HTTP/1.0","scenario":"default","status":"200","url":"http://127.0.0.1:8088/"}
EXAMPLE k6_vus => {"__name__":"k6_vus"}
EXAMPLE k6_http_req_failed_rate => {"__name__":"k6_http_req_failed_rate","expected_response":"true","method":"GET","name":"http://127.0.0.1:8088/","proto":"HTTP/1.0","scenario":"default","status":"200","url":"http://127.0.0.1:8088/"}
```

### Captured `__name__` values (17 unique, IDENTICAL across 2 runs)

Grouped by metric type (the suffix is a function of the type/stat, the base name is verbatim):

- **Gauge → no suffix:** `k6_vus`, `k6_vus_max`
- **Counter → `_total`:** `k6_iterations_total`, `k6_http_reqs_total`, `k6_data_sent_total`, `k6_data_received_total`, `k6_my_custom_counter_total`
- **Rate → `_rate`:** `k6_http_req_failed_rate`
- **Trend → `_p99` (default stat):** `k6_http_req_duration_p99`, `k6_http_req_blocked_p99`, `k6_http_req_connecting_p99`, `k6_http_req_waiting_p99`, `k6_http_req_sending_p99`, `k6_http_req_receiving_p99`, `k6_http_req_tls_handshaking_p99`, `k6_iteration_duration_p99`, `k6_my_custom_trend_p99`

The full decoded label sets appear verbatim in the receiver dump above (the `EXAMPLE …` lines). They confirm the base metric name is carried **verbatim** beneath the `k6_` prefix plus the per-type/stat suffix: the custom counter `my_custom_counter` → `k6_my_custom_counter_total` (`{"__name__":"k6_my_custom_counter_total","scenario":"default"}`), and the built-in `http_req_duration` → `k6_http_req_duration_p99` while carrying its full HTTP label set (`method`, `status`, `url`, `proto`, …).

### Integrity proof

Original names are carried verbatim under the prefix: `http_req_duration` → `k6_http_req_duration_p99`; the custom `my_custom_counter` → `k6_my_custom_counter_total`. Underscores are preserved, no dots or dashes are introduced, and there is no truncation. Custom metrics receive the same `k6_` prefix as built-ins. The set of 17 names was **identical across both runs** (empty symmetric difference).

### Root cause / citations

- The prefix constant is `defaultMetricPrefix = "k6_"` [vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/config.go:24].
- The name is constructed in `MapSeries`: `v := defaultMetricPrefix + series.Metric.Name` [vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/prometheus.go:40], then `if suffix != "" { v += "_" + suffix }` [vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/prometheus.go:42], and this `v` is set as the `__name__` label [vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/prometheus.go:44-47]. The metric name is otherwise carried verbatim — no sanitization or truncation is applied.
- The per-type suffix is chosen in `seriesWithMeasure.MapPrompb` [vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/remotewrite.go:316]: Counter → `"total"` [vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/remotewrite.go:330-333], Gauge → `""` [vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/remotewrite.go:335-338], Rate → `"rate"` [vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/remotewrite.go:340-344].
- Trend metrics get a stat suffix: the stats loop calls `tg.Append(stat, ...)` [vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/trend.go:53-54], and the suffix is appended to the name label via `ts.Labels[tg.ixname].Value += "_" + suffix` [vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/trend.go:89]. The default stat set is `defaultTrendStats = []string{"p(99)"}` [vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/config.go:28], and `p(99)` is normalized to `p99` (parentheses trimmed, `"p"` re-prefixed) at [vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/remotewrite.go:184-188].
- The output is registered under the identifier `experimental-prometheus-rw` [cmd/builtin_output_gen.go:10].

---

## Final Coverage Pass

Confirming every named sub-item across the five requirements is explicitly answered with runtime evidence and grounded citations:

- **Req 1 — Ramping-VUs under SIGINT:**
  - ✔ Exact signal-pipeline log lines captured (trap [cmd/common.go:98], "Stopping k6 in response to signal..." [cmd/run.go:350], abort error [cmd/run.go:354]).
  - ✔ Decisive interrupted-iteration proof (one self-consistent run): progress counter `5 complete and 6 interrupted iterations` ([execution/scheduler.go:156]) = summary `iterations: 5` = `ITER_END` (5); `ITER_START` (11) − `ITER_END` (5) = the 6 interrupted → active VUs terminated **mid-execution**, not allowed to finish. Engine counter (`5 complete / 6 interrupted`, `ITER_START` 11) stable across both runs; only the `ITER_END` marker varies (5→6, console-flush timing).
  - ✔ Before/during/after counter shown; root cause corroborated ([lib/executor/base_config.go:95-96], [lib/executor/helpers.go:117,122,195-196]).
  - ✔ Second-SIGINT hard stop ("Aborting k6 in response to signal" [cmd/run.go:360]); exit code **105** ([errext/exitcodes/codes.go:41]).
- **Req 2 — gRPC server-streaming interruption:**
  - ✔ Interruption log entries captured (`"stream is cancelled/finished" error="canceled by client (k6)"` [js/modules/k6/grpc/stream.go:201], `ErrCanceled` [lib/netext/grpcext/stream.go:29]).
  - ✔ `grpc_streams_msgs_received` value: **200** deterministic (2×100), **29** PARTIAL on interrupt (labeled timing-dependent); Counter registration [js/modules/k6/grpc/metrics.go:25].
  - ✔ 30ms graceful-ramp-down configuration exercised; key distinction (EOF vs canceled) stated.
- **Req 3 — dropped_iterations via API:**
  - ✔ Value = **1850**, obtained via the REST API v1 — raw JSON:API bodies from `GET /v1/metrics` [api/v1/metric_routes.go:9] and `GET /v1/metrics/dropped_iterations` [api/v1/metric_routes.go:27].
  - ✔ Before/after evidence (404 during run vs `count:1850` after); `--linger` + `/v1/status` `"running":false`; summary cross-check; name constant [metrics/builtin.go:10].
- **Req 4 — SharedArray memory footprint:**
  - ✔ Memory table (SharedArray flat 1→300 VUs vs per-VU-copy linear 1→10 VUs) from measured `VmHWM`.
  - ✔ Test-script output shown; root cause: single `[]string` backing store [js/modules/k6/data/share.go:10-12] shared by pointer `&rm.shared` [js/modules/k6/data/data.go:56], lazy per-element parse + deep-freeze [js/modules/k6/data/share.go:44-58].
- **Req 5 — Prometheus name integrity:**
  - ✔ All 17 `__name__` values captured (identical across 2 runs) + raw decoded label sets; `k6_` prefix + verbatim names + per-type/stat suffixes proven.
  - ✔ Root cause: `defaultMetricPrefix = "k6_"` [vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/config.go:24], `MapSeries` name construction [vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/prometheus.go:40-47]; output id `experimental-prometheus-rw` [cmd/builtin_output_gen.go:10].

**Reproducibility & cleanup:** k6 was built once as `k6 v0.55.0 (commit/ddc3b0b1d2, go1.23.4, linux/amd64)`; every value was confirmed stable across ≥2 runs (run-to-run variability reported where present). All experiments ran under `/tmp` via k6's real entry points (the `k6 run` CLI, the REST API v1, and the `k6/net/grpc` module against the in-repo RouteGuide server). All temporary scripts, logs, data files, and the compiled binary were deleted, leaving the repository tree unchanged.
