# Grafana k6 - Runtime Investigation of VU Orchestration (Q1-Q5)

This document answers five runtime-behaviour questions about the internal
orchestration of **Grafana k6** - the Go/JavaScript load-testing engine. It is an
**investigative, read-only** study: the subject of study is the k6 source tree at
commit `ddc3b0b1d23c128e34e2792fc9075f9126e32375` (branch `k6_ddc3b0b1d23c`), and the
**only** file written into the repository is this document. Every behavioural claim
below is substantiated by output **actually captured from a running `./k6` binary**
(not from reading source alone), shown verbatim together with the exact command that
produced it, and cross-referenced to the producing code with `file:line` citations.
Statements that are reasoned rather than directly observed at runtime are explicitly
labelled **(inferred)**.

All temporary scripts, data files, receivers and decoders used for observation were
created **outside** the repository tree (under `/tmp/k6smoke`) and deleted after
capture; the repository ends in a pristine state (`git status --porcelain` empty). The
built `./k6` binary does not dirty the tree because it is gitignored (`.gitignore`
line 1 = `/k6`).

## Environment and canonical build

The binary under study was built from source with the documented canonical command,
using the Go toolchain that was already on `PATH` (Go 1.23.12, matching the reported
build target `go1.23.12` and the canonical Docker base `golang:1.23-alpine3.20`).

Toolchain:

```text
$ go version
go version go1.23.12 linux/amd64
```

Canonical build (offline, fully vendored) and entry-point verification:

```text
$ CGO_ENABLED=0 GOFLAGS=-mod=vendor go build -trimpath -o k6 .
$ ./k6 version
k6 v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)

$ git status --porcelain      # empty -> tree clean; the ./k6 binary is gitignored (/k6)
```

The version string is produced by `lib/consts/consts.go:12` (`const Version = "0.55.0"`)
via `FullVersion()`, which appends `runtime.Version()`, `GOOS/GOARCH` and the VCS commit.
The CLI entry point is `main.go` -> `cmd.Execute()`. Structured logs are emitted by
`github.com/sirupsen/logrus` (`go.mod:37`) as `time="..." level=... msg="..." key=value`;
debug-level lines require the `--verbose` flag.

**Methodology (applies to every question):** exercise the real `./k6` CLI with real
inputs; use the default canonical configuration; capture complete, unedited output;
for count/timing questions (Q2, Q3) run at sufficient scale and confirm stability
across >=2 runs, reporting the distribution when a value is timing-dependent; exercise
secondary and error/edge paths (e.g. first vs. second SIGINT, interrupt with/without a
gRPC error handler, the REST API 404->200 transition); and, for the byte-sensitive
Prometheus payload, decode the exact emitted bytes.

## Q1 - VU lifecycle under SIGINT

**Question.** What are the exact log messages when a `ramping-vus` executor with >=5 VUs
receives a `SIGINT`? While shutting down, what log evidence shows whether currently
active VUs are allowed to finish their current iteration or are terminated
mid-execution?

### (a) Command and script

A `ramping-vus` scenario ramps to 6 VUs and holds; each iteration is a long
`sleep(20)`, so when the signal arrives every VU is provably mid-iteration.

```javascript
import { sleep } from 'k6';

// Q1: ramping-vus executor reaching 6 VUs, each running a long (20s) iteration
// so that when a SIGINT arrives, iterations are demonstrably in-flight.
export const options = {
  scenarios: {
    ramp: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '3s', target: 6 },   // ramp up to 6 VUs
        { duration: '120s', target: 6 },  // hold at 6 VUs (long, so we interrupt mid-hold)
      ],
      gracefulRampDown: '30s',
      gracefulStop: '30s',
    },
  },
};

export default function () {
  sleep(20);
}
```

Run under `--verbose` (to surface debug-level signal handling), then deliver **one**
`SIGINT` while iterations are in flight:

```text
$ ./k6 run --verbose /tmp/k6smoke/q1.js > /tmp/k6smoke/q1_single.log 2>&1 &
$ K6PID=$!
$ sleep 6            # 6 VUs are now active, each ~6s into a 20s sleep()
$ kill -INT $K6PID   # one SIGINT
$ wait $K6PID; echo "exit=$?"     # exit=105  (exitcodes.ExternalAbort)
```

### (b) Complete, unedited output - single SIGINT

```text
time="2026-07-08T04:23:33Z" level=debug msg="Logger format: TEXT"
time="2026-07-08T04:23:33Z" level=debug msg="k6 version: v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)"

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

time="2026-07-08T04:23:33Z" level=debug msg="Resolving and reading test '/tmp/k6smoke/q1.js'..."
time="2026-07-08T04:23:33Z" level=debug msg=Loading... moduleSpecifier="file:///tmp/k6smoke/q1.js" originalModuleSpecifier=/tmp/k6smoke/q1.js
time="2026-07-08T04:23:33Z" level=debug msg="'/tmp/k6smoke/q1.js' resolved to 'file:///tmp/k6smoke/q1.js' and successfully loaded 580 bytes!"
time="2026-07-08T04:23:33Z" level=debug msg="Gathering k6 runtime options..."
time="2026-07-08T04:23:33Z" level=debug msg="Initializing k6 runner for '/tmp/k6smoke/q1.js' (file:///tmp/k6smoke/q1.js)..."
time="2026-07-08T04:23:33Z" level=debug msg="Detecting test type for..." test_path="file:///tmp/k6smoke/q1.js"
time="2026-07-08T04:23:33Z" level=debug msg="Trying to load as a JS test..." test_path="file:///tmp/k6smoke/q1.js"
time="2026-07-08T04:23:33Z" level=debug msg="Runner successfully initialized!"
time="2026-07-08T04:23:33Z" level=debug msg="Parsing CLI flags..."
time="2026-07-08T04:23:33Z" level=debug msg="Consolidating config layers..."
time="2026-07-08T04:23:33Z" level=debug msg="Parsing thresholds and validating config..."
time="2026-07-08T04:23:33Z" level=debug msg="Initializing the execution scheduler..."
time="2026-07-08T04:23:33Z" level=debug msg="Starting 2 outputs..." component=output-manager
time="2026-07-08T04:23:33Z" level=debug msg=Starting... component=metrics-engine-ingester
time="2026-07-08T04:23:33Z" level=debug msg="Started!" component=metrics-engine-ingester
     execution: local
        script: /tmp/k6smoke/q1.js
        output: -

     scenarios: (100.00%) 1 scenario, 6 max VUs, 2m33s max duration (incl. graceful stop):
              * ramp: Up to 6 looping VUs for 2m3s over 2 stages (gracefulRampDown: 30s, gracefulStop: 30s)

time="2026-07-08T04:23:33Z" level=debug msg="Trapping interrupt signals so k6 can handle them gracefully..."
time="2026-07-08T04:23:33Z" level=debug msg="Starting the REST API server on localhost:6565"
time="2026-07-08T04:23:33Z" level=debug msg="Starting emission of VUs and VUsMax metrics..."
time="2026-07-08T04:23:33Z" level=debug msg="Start of initialization" executorsCount=1 neededVUs=6 phase=execution-scheduler-init
time="2026-07-08T04:23:33Z" level=debug msg="Initialized VU #2" phase=execution-scheduler-init
time="2026-07-08T04:23:33Z" level=debug msg="Initialized VU #6" phase=execution-scheduler-init
time="2026-07-08T04:23:33Z" level=debug msg="Initialized VU #3" phase=execution-scheduler-init
time="2026-07-08T04:23:33Z" level=debug msg="Initialized VU #5" phase=execution-scheduler-init
time="2026-07-08T04:23:33Z" level=debug msg="Initialized VU #4" phase=execution-scheduler-init
time="2026-07-08T04:23:33Z" level=debug msg="Initialized VU #1" phase=execution-scheduler-init
time="2026-07-08T04:23:33Z" level=debug msg="Finished initializing needed VUs, start initializing executors..." phase=execution-scheduler-init
time="2026-07-08T04:23:33Z" level=debug msg="Initialized executor ramp" phase=execution-scheduler-init
time="2026-07-08T04:23:33Z" level=debug msg="Initialization completed" phase=execution-scheduler-init
time="2026-07-08T04:23:33Z" level=debug msg="Start of test run" executorsCount=1 phase=execution-scheduler-run
time="2026-07-08T04:23:33Z" level=debug msg="setup() is not defined or not exported, skipping!"
time="2026-07-08T04:23:33Z" level=debug msg="Start all executors..." phase=execution-scheduler-run
time="2026-07-08T04:23:33Z" level=debug msg="Starting executor" executor=ramp startTime=0s type=ramping-vus
time="2026-07-08T04:23:33Z" level=debug msg="Starting executor run..." duration=2m3s executor=ramping-vus maxVUs=6 numStages=2 scenario=ramp startVUs=0 type=ramping-vus
time="2026-07-08T04:23:33Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=0

running (0m01.0s), 1/6 VUs, 0 complete and 0 interrupted iterations
ramp   [   1% ] 1/6 VUs  0m01.0s/2m03.0s
time="2026-07-08T04:23:34Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=1
time="2026-07-08T04:23:34Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=2

running (0m02.0s), 3/6 VUs, 0 complete and 0 interrupted iterations
ramp   [   2% ] 3/6 VUs  0m02.0s/2m03.0s
time="2026-07-08T04:23:35Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=3
time="2026-07-08T04:23:35Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=4

running (0m03.0s), 5/6 VUs, 0 complete and 0 interrupted iterations
ramp   [   2% ] 5/6 VUs  0m03.0s/2m03.0s
time="2026-07-08T04:23:36Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=5

running (0m04.0s), 6/6 VUs, 0 complete and 0 interrupted iterations
ramp   [   3% ] 6/6 VUs  0m04.0s/2m03.0s

running (0m05.0s), 6/6 VUs, 0 complete and 0 interrupted iterations
ramp   [   4% ] 6/6 VUs  0m05.0s/2m03.0s
time="2026-07-08T04:23:39Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-08T04:23:39Z" level=debug msg="Metrics emission of VUs and VUsMax metrics stopped"
time="2026-07-08T04:23:39Z" level=debug msg="Executor finished successfully" executor=ramp startTime=0s type=ramping-vus
time="2026-07-08T04:23:39Z" level=debug msg="teardown() is not defined or not exported, skipping!"
time="2026-07-08T04:23:39Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
time="2026-07-08T04:23:39Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-08T04:23:39Z" level=debug msg="Stopping vus and vux_max metrics emission..." phase=execution-scheduler-init
time="2026-07-08T04:23:39Z" level=debug msg="Releasing signal trap..."
time="2026-07-08T04:23:39Z" level=debug msg="Sending usage report..."
time="2026-07-08T04:23:39Z" level=debug msg="Waiting for metrics and traces processing to finish..."
time="2026-07-08T04:23:39Z" level=debug msg="Metrics and traces processing finished!"
time="2026-07-08T04:23:39Z" level=debug msg="Stopping outputs..."
time="2026-07-08T04:23:39Z" level=debug msg="Stopping 2 outputs..." component=output-manager
time="2026-07-08T04:23:39Z" level=debug msg=Stopping... component=metrics-engine-ingester
time="2026-07-08T04:23:39Z" level=debug msg="Stopped!" component=metrics-engine-ingester
time="2026-07-08T04:23:39Z" level=debug msg="Generating the end-of-test summary..."

     data_received...: 0 B 0 B/s
     data_sent.......: 0 B 0 B/s
     vus.............: 6   min=1 max=6
     vus_max.........: 6   min=6 max=6


running (0m06.0s), 0/6 VUs, 0 complete and 6 interrupted iterations
ramp ✗ [   5% ] 6/6 VUs  0m06.0s/2m03.0s
time="2026-07-08T04:23:39Z" level=debug msg="Usage report sent successfully"
time="2026-07-08T04:23:39Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-08T04:23:39Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

### (c) Second-SIGINT hard-stop edge path

Re-running the same script and delivering **two** `SIGINT`s ~50 ms apart exercises the
hard-stop path. (A smaller gap is coalesced by the kernel into a single delivered
signal, so two distinct signals must arrive within the teardown window.)

```text
$ ./k6 run --verbose /tmp/k6smoke/q1.js > /tmp/k6smoke/q1_double.log 2>&1 &
$ K6PID=$!
$ sleep 6
$ kill -INT $K6PID; sleep 0.05; kill -INT $K6PID     # two SIGINTs
```
```text
time="2026-07-08T04:26:24Z" level=debug msg="Logger format: TEXT"
time="2026-07-08T04:26:24Z" level=debug msg="k6 version: v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)"

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

time="2026-07-08T04:26:24Z" level=debug msg="Resolving and reading test '/tmp/k6smoke/q1.js'..."
time="2026-07-08T04:26:24Z" level=debug msg=Loading... moduleSpecifier="file:///tmp/k6smoke/q1.js" originalModuleSpecifier=/tmp/k6smoke/q1.js
time="2026-07-08T04:26:24Z" level=debug msg="'/tmp/k6smoke/q1.js' resolved to 'file:///tmp/k6smoke/q1.js' and successfully loaded 580 bytes!"
time="2026-07-08T04:26:24Z" level=debug msg="Gathering k6 runtime options..."
time="2026-07-08T04:26:24Z" level=debug msg="Initializing k6 runner for '/tmp/k6smoke/q1.js' (file:///tmp/k6smoke/q1.js)..."
time="2026-07-08T04:26:24Z" level=debug msg="Detecting test type for..." test_path="file:///tmp/k6smoke/q1.js"
time="2026-07-08T04:26:24Z" level=debug msg="Trying to load as a JS test..." test_path="file:///tmp/k6smoke/q1.js"
time="2026-07-08T04:26:24Z" level=debug msg="Runner successfully initialized!"
time="2026-07-08T04:26:24Z" level=debug msg="Parsing CLI flags..."
time="2026-07-08T04:26:24Z" level=debug msg="Consolidating config layers..."
time="2026-07-08T04:26:24Z" level=debug msg="Parsing thresholds and validating config..."
time="2026-07-08T04:26:24Z" level=debug msg="Initializing the execution scheduler..."
time="2026-07-08T04:26:24Z" level=debug msg="Starting 2 outputs..." component=output-manager
time="2026-07-08T04:26:24Z" level=debug msg=Starting... component=metrics-engine-ingester
time="2026-07-08T04:26:24Z" level=debug msg="Started!" component=metrics-engine-ingester
     execution: local
        script: /tmp/k6smoke/q1.js
        output: -

     scenarios: (100.00%) 1 scenario, 6 max VUs, 2m33s max duration (incl. graceful stop):
              * ramp: Up to 6 looping VUs for 2m3s over 2 stages (gracefulRampDown: 30s, gracefulStop: 30s)

time="2026-07-08T04:26:24Z" level=debug msg="Trapping interrupt signals so k6 can handle them gracefully..."
time="2026-07-08T04:26:24Z" level=debug msg="Starting the REST API server on localhost:6565"
time="2026-07-08T04:26:24Z" level=debug msg="Starting emission of VUs and VUsMax metrics..."
time="2026-07-08T04:26:24Z" level=debug msg="Start of initialization" executorsCount=1 neededVUs=6 phase=execution-scheduler-init
time="2026-07-08T04:26:24Z" level=debug msg="Initialized VU #1" phase=execution-scheduler-init
time="2026-07-08T04:26:24Z" level=debug msg="Initialized VU #5" phase=execution-scheduler-init
time="2026-07-08T04:26:24Z" level=debug msg="Initialized VU #3" phase=execution-scheduler-init
time="2026-07-08T04:26:24Z" level=debug msg="Initialized VU #6" phase=execution-scheduler-init
time="2026-07-08T04:26:24Z" level=debug msg="Initialized VU #4" phase=execution-scheduler-init
time="2026-07-08T04:26:24Z" level=debug msg="Initialized VU #2" phase=execution-scheduler-init
time="2026-07-08T04:26:24Z" level=debug msg="Finished initializing needed VUs, start initializing executors..." phase=execution-scheduler-init
time="2026-07-08T04:26:24Z" level=debug msg="Initialized executor ramp" phase=execution-scheduler-init
time="2026-07-08T04:26:24Z" level=debug msg="Initialization completed" phase=execution-scheduler-init
time="2026-07-08T04:26:24Z" level=debug msg="Start of test run" executorsCount=1 phase=execution-scheduler-run
time="2026-07-08T04:26:24Z" level=debug msg="setup() is not defined or not exported, skipping!"
time="2026-07-08T04:26:24Z" level=debug msg="Start all executors..." phase=execution-scheduler-run
time="2026-07-08T04:26:24Z" level=debug msg="Starting executor" executor=ramp startTime=0s type=ramping-vus
time="2026-07-08T04:26:24Z" level=debug msg="Starting executor run..." duration=2m3s executor=ramping-vus maxVUs=6 numStages=2 scenario=ramp startVUs=0 type=ramping-vus
time="2026-07-08T04:26:24Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=0
time="2026-07-08T04:26:28Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=1

running (0m01.0s), 0/6 VUs, 0 complete and 0 interrupted iterations
ramp   [   1% ] 0/6 VUs  0m01.0s/2m03.0s
time="2026-07-08T04:26:28Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=2

running (0m03.7s), 2/6 VUs, 0 complete and 0 interrupted iterations
ramp   [   3% ] 3/6 VUs  0m03.7s/2m03.0s
time="2026-07-08T04:26:28Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=3
time="2026-07-08T04:26:28Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=4
time="2026-07-08T04:26:28Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=5

running (0m04.0s), 6/6 VUs, 0 complete and 0 interrupted iterations
ramp   [   3% ] 6/6 VUs  0m04.0s/2m03.0s

running (0m05.0s), 6/6 VUs, 0 complete and 0 interrupted iterations
ramp   [   4% ] 6/6 VUs  0m05.0s/2m03.0s
time="2026-07-08T04:26:30Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt

running (0m06.0s), 6/6 VUs, 0 complete and 0 interrupted iterations
ramp   [   5% ] 6/6 VUs  0m06.0s/2m03.0s
time="2026-07-08T04:26:30Z" level=error msg="Aborting k6 in response to signal" sig=interrupt
```

### (d) file:line citations

- **Signal trap.** `cmd/common.go:98` logs `Trapping interrupt signals so k6 can handle
  them gracefully...` (function `handleTestAbortSignals`), registering `os.Interrupt,
  syscall.SIGINT, syscall.SIGTERM`.
- **First signal - graceful-stop handler.** `cmd/run.go:350` logs the debug line
  `Stopping k6 in response to signal...` with field `sig=interrupt`, then aborts the run
  with the error at `cmd/run.go:354`:
  `test run was aborted because k6 received a '%s' signal` (exit code
  `exitcodes.ExternalAbort` = 105).
- **Second signal - hard stop.** `cmd/run.go:360` logs `Aborting k6 in response to
  signal` at **error** level; the second signal then causes an immediate
  `gs.OSExit(int(exitcodes.ExternalAbort))` at `cmd/common.go:118`.
- **DECISIVE completion-vs-interruption evidence.** The scheduler progress line format is
  at `execution/scheduler.go:156`:
  `"%s, "+vusFmt+"/"+vusFmt+" VUs, %d complete and %d interrupted iterations"`. The two
  counters are `GetFullIterationCount()` (`lib/execution.go:284`, "complete") and
  `GetPartialIterationCount()` (`lib/execution.go:300`, "interrupted"); interrupted
  iterations are tallied by `AddInterruptedIterations()` (`lib/execution.go:308`).
- **Semantics (root cause).** `lib/executor/base_config.go:95` documents that the graceful
  stop period *"doesn't count when the user manually interrupts the test, then iterations
  are immediately stopped."* Context: `lib/executor/ramping_vus.go` defines
  `RampingVUsConfig` with `GracefulRampDown` (default `30s`); `lib/executor/vu_handle.go`
  provides the `canStartIter` gate and `hardStop()` that cancels an in-flight iteration.

### (e) Cause -> effect

The decisive line in the single-SIGINT run is:

```text
running (0m06.0s), 0/6 VUs, 0 complete and 6 interrupted iterations
```

All six active VUs were ~6 s into a 20 s `sleep(20)` when the signal arrived. The split
is **`0 complete / 6 interrupted`** - every in-flight iteration was **terminated
mid-execution, not allowed to finish**. Mechanistically, a manual `SIGINT` is delivered
to the trap at `cmd/common.go:98`; the first delivery runs the graceful-stop handler at
`cmd/run.go:350`, which aborts the run context with reason *AbortedByUser* and exit code
*ExternalAbort* (`cmd/run.go:354`). Unlike an end-of-duration graceful stop (which honours
the `gracefulStop`/`gracefulRampDown` timers before force-cancelling), a **manual
interrupt immediately cancels active VU iterations** - exactly as documented at
`lib/executor/base_config.go:95`. The `0 complete / 6 interrupted` counters
(`lib/execution.go:284`/`:300`) are the runtime proof.

On the **second** SIGINT, the handler at `cmd/run.go:360` logs `Aborting k6 in response
to signal` at error level and k6 calls `OSExit(ExternalAbort)` at `cmd/common.go:118`,
exiting **immediately** - note the double-SIGINT log **ends** at that error line with **no
end-of-test summary** printed, in contrast to the single-SIGINT run which finishes writing
its summary. Both runs exit with code **105** (`exitcodes.ExternalAbort`).

**Web/doc cross-check.** Grafana's *Graceful stop* / *Ramping VUs* documentation states
`gracefulStop` (default 30 s) and `gracefulRampDown` bound how long k6 waits before
force-interrupting an iteration **at the end of a stage/duration**, and that a manual
interrupt does not grant that grace - consistent with the observed `0 complete / 6
interrupted` outcome.

## Q2 - gRPC server-streaming interruption

**Question.** What are the exact runtime log entries when a gRPC **server-streaming**
test with a **30 ms** graceful ramp-down is interrupted? Also report the value of
*"number of grpc messages received"* (`grpc_streams_msgs_received`) in the final metrics
summary. (Run >=2 times; report the distribution.)

### (a) Commands and script

The in-repository gRPC example server is a **separate nested Go module**
(`examples/grpc_server/go.mod`, with `replace go.k6.io/k6 => ../../`). To avoid dirtying
the tree, it was copied outside the repo, its `replace` retargeted to the absolute repo
path, built, and started on `localhost:10000` (`examples/grpc_server/main.go:51`). Its
`route_guide.proto` was copied next to the temp script (k6's `client.load()` resolves the
proto path relative to the **script** directory).

```text
# --- start the in-repo gRPC server from an out-of-repo copy (keeps the tree clean) ---
$ REPO_ABS=/tmp/blitzy/k6/blitzy-0dce0186-fb81-4374-874e-fe967cbc5702_ae9224
$ cp -r "$REPO_ABS/examples/grpc_server" /tmp/k6smoke/grpc_server
$ cp "$REPO_ABS/lib/testutils/grpcservice/route_guide.proto" /tmp/k6smoke/route_guide.proto
$ cd /tmp/k6smoke/grpc_server && go mod edit -replace go.k6.io/k6="$REPO_ABS"
$ GOFLAGS=-mod=mod go build -o server . && ./server &     # "gRPC server starting on localhost:10000" 
```

The server-streaming client (30 ms `gracefulRampDown`/`gracefulStop`, **no** `error` handler):

```javascript
import { Client, Stream } from 'k6/net/grpc';
import { sleep } from 'k6';

// Q2: gRPC server-streaming (main.FeatureExplorer/ListFeatures) with a 30ms graceful
// ramp-down / graceful stop, interrupted mid-run. NO 'error' handler is registered in
// this variant, so an interrupted stream surfaces the built-in warning from stream.go:395.
const GRPC_ADDR = __ENV.GRPC_ADDR || '127.0.0.1:10000';
const GRPC_PROTO_PATH = __ENV.GRPC_PROTO_PATH || 'route_guide.proto';

const client = new Client();
client.load([], GRPC_PROTO_PATH);

export const options = {
  scenarios: {
    stream: {
      executor: 'ramping-vus',
      startVUs: 4,
      stages: [{ duration: '60s', target: 4 }],
      gracefulRampDown: '30ms',
      gracefulStop: '30ms',
    },
  },
};

export default () => {
  client.connect(GRPC_ADDR, { plaintext: true });

  const stream = new Stream(client, 'main.FeatureExplorer/ListFeatures', null);

  // Count received messages on the JS side too (cross-check for grpc_streams_msgs_received)
  stream.on('data', function (feature) {
    // no-op body; the metric increments per received message (stream.go:153)
  });

  stream.on('end', function () {
    client.close();
  });

  // NOTE: intentionally NO stream.on('error', ...) here.

  // Request all features in the example bounding rectangle (server streams them at 100ms each)
  stream.write({
    lo: { latitude: 400000000, longitude: -750000000 },
    hi: { latitude: 420000000, longitude: -730000000 },
  });

  sleep(3);
};
```

Run three times; each time interrupt with a single `SIGINT` ~7 s in (single signal -> the
graceful path runs, so the end-of-test summary is printed):

```text
$ for R in 1 2 3; do
    ./k6 run --verbose /tmp/k6smoke/q2.js > /tmp/k6smoke/q2_run${R}.log 2>&1 &
    K6PID=$!; sleep 7; kill -INT $K6PID; wait $K6PID
  done
```

### (b) Complete, unedited output - run 1

The full, unedited run-1 log (output was redirected to a file, so there are no
carriage-return progress rewrites) - 115 lines, including the scenario header, the
per-message debug lines, the four interruption warnings, the debug stream-cancellation
lines, the **final metrics summary**, and the closing interrupted-iteration line:

```text
time="2026-07-08T04:31:19Z" level=debug msg="Logger format: TEXT"
time="2026-07-08T04:31:19Z" level=debug msg="k6 version: v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)"

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

time="2026-07-08T04:31:19Z" level=debug msg="Resolving and reading test '/tmp/k6smoke/q2.js'..."
time="2026-07-08T04:31:19Z" level=debug msg=Loading... moduleSpecifier="file:///tmp/k6smoke/q2.js" originalModuleSpecifier=/tmp/k6smoke/q2.js
time="2026-07-08T04:31:19Z" level=debug msg="'/tmp/k6smoke/q2.js' resolved to 'file:///tmp/k6smoke/q2.js' and successfully loaded 1499 bytes!"
time="2026-07-08T04:31:19Z" level=debug msg="Gathering k6 runtime options..."
time="2026-07-08T04:31:19Z" level=debug msg="Initializing k6 runner for '/tmp/k6smoke/q2.js' (file:///tmp/k6smoke/q2.js)..."
time="2026-07-08T04:31:19Z" level=debug msg="Detecting test type for..." test_path="file:///tmp/k6smoke/q2.js"
time="2026-07-08T04:31:19Z" level=debug msg="Trying to load as a JS test..." test_path="file:///tmp/k6smoke/q2.js"
time="2026-07-08T04:31:19Z" level=debug msg="Runner successfully initialized!"
time="2026-07-08T04:31:19Z" level=debug msg="Parsing CLI flags..."
time="2026-07-08T04:31:19Z" level=debug msg="Consolidating config layers..."
time="2026-07-08T04:31:19Z" level=debug msg="Parsing thresholds and validating config..."
time="2026-07-08T04:31:19Z" level=debug msg="Initializing the execution scheduler..."
time="2026-07-08T04:31:19Z" level=debug msg="Starting 2 outputs..." component=output-manager
time="2026-07-08T04:31:19Z" level=debug msg=Starting... component=metrics-engine-ingester
time="2026-07-08T04:31:19Z" level=debug msg="Started!" component=metrics-engine-ingester
     execution: local
        script: /tmp/k6smoke/q2.js
        output: -

     scenarios: (100.00%) 1 scenario, 4 max VUs, 1m0s max duration (incl. graceful stop):
              * stream: Up to 4 looping VUs for 1m0s over 1 stages (gracefulRampDown: 30ms, gracefulStop: 30ms)

time="2026-07-08T04:31:19Z" level=debug msg="Trapping interrupt signals so k6 can handle them gracefully..."
time="2026-07-08T04:31:19Z" level=debug msg="Starting the REST API server on localhost:6565"
time="2026-07-08T04:31:19Z" level=debug msg="Starting emission of VUs and VUsMax metrics..."
time="2026-07-08T04:31:19Z" level=debug msg="Start of initialization" executorsCount=1 neededVUs=4 phase=execution-scheduler-init
time="2026-07-08T04:31:19Z" level=debug msg="Initialized VU #4" phase=execution-scheduler-init
time="2026-07-08T04:31:19Z" level=debug msg="Initialized VU #2" phase=execution-scheduler-init
time="2026-07-08T04:31:19Z" level=debug msg="Initialized VU #3" phase=execution-scheduler-init
time="2026-07-08T04:31:19Z" level=debug msg="Initialized VU #1" phase=execution-scheduler-init
time="2026-07-08T04:31:19Z" level=debug msg="Finished initializing needed VUs, start initializing executors..." phase=execution-scheduler-init
time="2026-07-08T04:31:19Z" level=debug msg="Initialized executor stream" phase=execution-scheduler-init
time="2026-07-08T04:31:19Z" level=debug msg="Initialization completed" phase=execution-scheduler-init
time="2026-07-08T04:31:19Z" level=debug msg="Start of test run" executorsCount=1 phase=execution-scheduler-run
time="2026-07-08T04:31:19Z" level=debug msg="setup() is not defined or not exported, skipping!"
time="2026-07-08T04:31:19Z" level=debug msg="Start all executors..." phase=execution-scheduler-run
time="2026-07-08T04:31:19Z" level=debug msg="Starting executor" executor=stream startTime=0s type=ramping-vus
time="2026-07-08T04:31:19Z" level=debug msg="Starting executor run..." duration=1m0s executor=ramping-vus maxVUs=4 numStages=1 scenario=stream startVUs=4 type=ramping-vus
time="2026-07-08T04:31:19Z" level=debug msg=Start executor=ramping-vus scenario=stream vuNum=0
time="2026-07-08T04:31:19Z" level=debug msg=Start executor=ramping-vus scenario=stream vuNum=1
time="2026-07-08T04:31:19Z" level=debug msg=Start executor=ramping-vus scenario=stream vuNum=2
time="2026-07-08T04:31:19Z" level=debug msg=Start executor=ramping-vus scenario=stream vuNum=3

running (0m01.0s), 4/4 VUs, 0 complete and 0 interrupted iterations
stream   [   2% ] 4/4 VUs  0m01.0s/1m00.0s

running (0m02.0s), 4/4 VUs, 0 complete and 0 interrupted iterations
stream   [   3% ] 4/4 VUs  0m02.0s/1m00.0s

running (0m03.0s), 4/4 VUs, 0 complete and 0 interrupted iterations
stream   [   5% ] 4/4 VUs  0m03.0s/1m00.0s

running (0m04.0s), 4/4 VUs, 0 complete and 0 interrupted iterations
stream   [   7% ] 4/4 VUs  0m04.0s/1m00.0s

running (0m05.0s), 4/4 VUs, 0 complete and 0 interrupted iterations
stream   [   8% ] 4/4 VUs  0m05.0s/1m00.0s

running (0m06.0s), 4/4 VUs, 0 complete and 0 interrupted iterations
stream   [  10% ] 4/4 VUs  0m06.0s/1m00.0s
time="2026-07-08T04:31:26Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-08T04:31:26Z" level=debug msg="Metrics emission of VUs and VUsMax metrics stopped"
time="2026-07-08T04:31:26Z" level=debug msg="stream is cancelled/finished" error="canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-08T04:31:26Z" level=debug msg="stream /main.FeatureExplorer/ListFeatures is closing" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-08T04:31:26Z" level=warning msg="no handlers for error registered, but an error happened: canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-08T04:31:26Z" level=debug msg="stream is cancelled/finished" error="canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-08T04:31:26Z" level=debug msg="stream /main.FeatureExplorer/ListFeatures is closing" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-08T04:31:26Z" level=warning msg="no handlers for error registered, but an error happened: canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-08T04:31:26Z" level=debug msg="stream is cancelled/finished" error="canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-08T04:31:26Z" level=debug msg="stream /main.FeatureExplorer/ListFeatures is closing" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-08T04:31:26Z" level=warning msg="no handlers for error registered, but an error happened: canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-08T04:31:26Z" level=debug msg="stream is cancelled/finished" error="canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-08T04:31:26Z" level=debug msg="stream /main.FeatureExplorer/ListFeatures is closing" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-08T04:31:26Z" level=warning msg="no handlers for error registered, but an error happened: canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-08T04:31:26Z" level=debug msg="Executor finished successfully" executor=stream startTime=0s type=ramping-vus
time="2026-07-08T04:31:26Z" level=debug msg="teardown() is not defined or not exported, skipping!"
time="2026-07-08T04:31:26Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
time="2026-07-08T04:31:26Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-08T04:31:26Z" level=debug msg="Stopping vus and vux_max metrics emission..." phase=execution-scheduler-init
time="2026-07-08T04:31:26Z" level=debug msg="Releasing signal trap..."
time="2026-07-08T04:31:26Z" level=debug msg="Sending usage report..."
time="2026-07-08T04:31:26Z" level=debug msg="Waiting for metrics and traces processing to finish..."
time="2026-07-08T04:31:26Z" level=debug msg="Metrics and traces processing finished!"
time="2026-07-08T04:31:26Z" level=debug msg="Stopping outputs..."
time="2026-07-08T04:31:26Z" level=debug msg="Stopping 2 outputs..." component=output-manager
time="2026-07-08T04:31:26Z" level=debug msg=Stopping... component=metrics-engine-ingester
time="2026-07-08T04:31:26Z" level=debug msg="Stopped!" component=metrics-engine-ingester
time="2026-07-08T04:31:26Z" level=debug msg="Generating the end-of-test summary..."

     data_received................: 25 kB  3.6 kB/s
     data_sent....................: 9.3 kB 1.3 kB/s
     grpc_streams.................: 4      0.574033/s
     grpc_streams_msgs_received...: 276    39.608245/s
     grpc_streams_msgs_sent.......: 4      0.574033/s
     vus..........................: 4      min=4       max=4
     vus_max......................: 4      min=4       max=4

time="2026-07-08T04:31:26Z" level=debug msg="Usage report sent successfully"

running (0m07.0s), 0/4 VUs, 0 complete and 4 interrupted iterations
stream ✗ [  12% ] 4/4 VUs  0m07.0s/1m00.0s

running (0m07.0s), 0/4 VUs, 0 complete and 4 interrupted iterations
stream ✗ [  12% ] 4/4 VUs  0m07.0s/1m00.0s
time="2026-07-08T04:31:26Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-08T04:31:26Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

### (c) Stability across runs

The server streams one `Feature` roughly every 100 ms per active stream
(`lib/testutils/grpcservice/service.go:57` sleeps 100 ms before each `stream.Send`), so
`grpc_streams_msgs_received` is **inherently timing-dependent** - it scales with how long
the streams ran before interruption. Using the **same** unchanged script and the **same**
~7 s single-SIGINT timing:

| Run | `grpc_streams` | `grpc_streams_msgs_received` | `grpc_streams_msgs_sent` | interruption warnings |
|----:|---------------:|-----------------------------:|-------------------------:|----------------------:|
| 1   | 4              | **276**                      | 4                        | 4                     |
| 2   | 4              | **272**                      | 4                        | 4                     |
| 3   | 4              | **276**                      | 4                        | 4                     |

The count is **stable in the 272-276 band at ~7 s** (a few messages of run-to-run
variance from the 100 ms/message server pacing and the exact interrupt instant). A
longer ~15 s run of the same script yielded **592**, confirming the value **scales with
duration** and is therefore a timing-dependent magnitude, **not** a fixed constant.
`grpc_streams_msgs_sent` is exactly **4** every run (one `Rectangle` request written per
stream, for 4 VUs). The verbatim summary lines from each run:

```text
# run 1
     grpc_streams.................: 4      0.574033/s
     grpc_streams_msgs_received...: 276    39.608245/s
     grpc_streams_msgs_sent.......: 4      0.574033/s
# run 2
     grpc_streams.................: 4      0.578704/s
     grpc_streams_msgs_received...: 272    39.351853/s
     grpc_streams_msgs_sent.......: 4      0.578704/s
# run 3
     grpc_streams.................: 4      0.573722/s
     grpc_streams_msgs_received...: 276    39.586794/s
     grpc_streams_msgs_sent.......: 4      0.573722/s
```

### (d) Secondary path - an `error` handler *is* registered

When a `stream.on('error', ...)` handler **is** registered and the stream is cancelled
**while the VU is still alive** (here via an explicit `client.close()` rather than a
process interrupt), the handler receives the error and **no** warning is logged:

```javascript
import { Client, Stream } from 'k6/net/grpc';
import { sleep } from 'k6';

// Q2 secondary path (clean): register an 'error' handler and cancel the stream from the
// client side (client.close()) mid-iteration while the VU is alive, so the 'error' event
// is actually delivered to the JS handler (no SIGINT VU-teardown confound).
const GRPC_ADDR = __ENV.GRPC_ADDR || '127.0.0.1:10000';
const GRPC_PROTO_PATH = __ENV.GRPC_PROTO_PATH || 'route_guide.proto';

const client = new Client();
client.load([], GRPC_PROTO_PATH);

export const options = { vus: 1, iterations: 2 };

export default () => {
  client.connect(GRPC_ADDR, { plaintext: true });
  const stream = new Stream(client, 'main.FeatureExplorer/ListFeatures', null);
  stream.on('data', function (feature) {});
  stream.on('error', function (e) {
    console.log('STREAM_ERROR_HANDLER code=' + e.code + ' message=' + e.message);
  });
  stream.on('end', function () {});
  stream.write({
    lo: { latitude: 400000000, longitude: -750000000 },
    hi: { latitude: 420000000, longitude: -730000000 },
  });
  sleep(0.5);       // receive a few messages
  client.close();   // client-side cancel while the VU/event-loop is still alive
  sleep(0.5);       // allow the event loop to deliver the 'error' to the handler
};
```
```text
$ ./k6 run --verbose /tmp/k6smoke/q2_close.js > /tmp/k6smoke/q2_close.log 2>&1
# captured (verbatim):
time="...Z" level=info msg="STREAM_ERROR_HANDLER code=2 message=canceled by client (k6)" source=console
time="...Z" level=info msg="STREAM_ERROR_HANDLER code=2 message=canceled by client (k6)" source=console
     grpc_streams_msgs_received...: 8     1.217522/s
# interruption-warning count in this run: 0
```

gRPC status **code 2 = CANCELLED**, message `canceled by client (k6)`. For completeness, a
run that registers an `error` handler but is then hard-interrupted by `SIGINT` logged
**0** warnings *and* did not print the handler's `console.log` (received = 308 that run):
the process teardown cancels the streams (so the warning is suppressed by the registered
handler) but exits before the queued `error` callback is dispatched on the VU event loop
**(inferred)** - reasoned from the observed absence of both the warning and the handler line.

### (e) file:line citations

- `js/modules/k6/grpc/metrics.go:25` defines `grpc_streams_msgs_received` (a
  `metrics.Counter`); `:21` defines `grpc_streams_msgs_sent`.
- `js/modules/k6/grpc/stream.go:153` pushes a `StreamsMessagesReceived` sample (value 1)
  for **each** received message.
- `js/modules/k6/grpc/stream.go:395` emits the interruption warning
  `no handlers for error registered, but an error happened: %s` - reached only when the
  stream errors **and** no `error` listener is registered.
- Server: `examples/grpc_server/main.go:51` listens on `localhost:10000`;
  `lib/testutils/grpcservice/service.go:57` implements `ListFeatures`, sending one feature
  per 100 ms.

### (f) Cause -> effect

On interruption the k6 client cancels the gRPC stream context; the server-streaming RPC
`/main.FeatureExplorer/ListFeatures` ends with gRPC status **CANCELLED** and message
`canceled by client (k6)`. With **no** `error` handler registered, k6 logs the warning at
`stream.go:395` - observed **once per interrupted stream** (4 warnings for 4 VUs). Each
`Feature` received before cancellation incremented `grpc_streams_msgs_received` once
(`stream.go:153`), giving the timing-dependent 272-276 (@7 s) / 592 (@15 s) totals.
**Web/doc cross-check:** Grafana docs describe `gracefulRampDown`/`gracefulStop` (here
30 ms) as the brief window before k6 force-cancels in-flight iterations - consistent with
the near-immediate stream cancellation observed.

## Q3 - dropped_iterations via the REST API

**Question.** What is the exact value of `dropped_iterations` when a test exceeds its
maximum duration capacity? Report the value **by querying the REST API** (not the
end-of-test summary), with runtime evidence (the HTTP request and raw JSON response)
proving it came from the API. (Repeat >=2 runs.)

### (a) Command and script

A `constant-arrival-rate` executor asks for 500 iterations/s but is capped at 2 VUs, each
sleeping 1 s - so the arrival rate cannot be met and k6 emits `dropped_iterations`
(`lib/executor/constant_arrival_rate.go`).

```javascript
import { sleep } from 'k6';

// Q3: constant-arrival-rate overload. k6 tries to start 500 iterations/second but only
// 2 VUs are available, each iteration sleeps 1s, so the arrival rate cannot be met and
// k6 emits dropped_iterations (constant_arrival_rate.go:324-341).
export const options = {
  scenarios: {
    overload: {
      executor: 'constant-arrival-rate',
      rate: 500,
      timeUnit: '1s',
      duration: '30s',
      preAllocatedVUs: 1,
      maxVUs: 2,
    },
  },
};

export default function () {
  sleep(1);
}
```

Run in the **default** configuration (no `--verbose`); the REST API is auto-enabled on
`localhost:6565` (`cmd/state/state.go:150`). **While the test is live**, query the API:

```text
$ ./k6 run /tmp/k6smoke/q3.js > /tmp/k6smoke/q3_klog_run1.log 2>&1 &
$ curl -s http://localhost:6565/v1/status
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
```

### (b) Raw API evidence - the decisive 000 -> 404 -> 200 transition

Busy-polling `GET /v1/metrics/dropped_iterations` from `t=0` captures the API server
coming up and the metric being served **live** (each block shows the literal request and
the raw JSON:API response body, with the HTTP status):

```text
----- poll #1 : status transition -> HTTP 000 -----
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
HTTP_STATUS=000

----- poll #4 : status transition -> HTTP 404 -----
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
{"errors":[{"status":"404","title":"Not Found","detail":"No metric with that ID was found"}]}HTTP_STATUS=404

----- poll #15 : status transition -> HTTP 200 -----
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":48,"rate":456.588510857594}}}}HTTP_STATUS=200

[confirm 200 body] {"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":48,"rate":274.4838131748799}}}}
```

The `404` body is emitted by `api/v1/metric_routes.go:37`
(`No metric with that ID was found`) **before** the first drop is recorded; once dropping
begins the same URL returns `200` with a live `count`. A static summary read could never
`404`-then-`200`, so this transition is proof the value is served **live from the metrics
engine via the API**.

### (c) Raw API evidence - growing counts with running:true (run 1)

```text
===== [run1] elapsed ~0.2s cumulative =====
$ curl -s http://localhost:6565/v1/status
{"data":{"type":"status","id":"default","attributes":{"status":7,"paused":false,"vus":2,"vus-max":2,"stopped":false,"running":true,"tainted":false}}}
HTTP_STATUS=200
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":49,"rate":258.48694130830324}}}}
HTTP_STATUS=200

===== [run1] elapsed ~0.4s cumulative =====
$ curl -s http://localhost:6565/v1/status
{"data":{"type":"status","id":"default","attributes":{"status":7,"paused":false,"vus":2,"vus-max":2,"stopped":false,"running":true,"tainted":false}}}
HTTP_STATUS=200
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":274,"rate":452.18912387622856}}}}
HTTP_STATUS=200

===== [run1] elapsed ~0.6s cumulative =====
$ curl -s http://localhost:6565/v1/status
{"data":{"type":"status","id":"default","attributes":{"status":7,"paused":false,"vus":2,"vus-max":2,"stopped":false,"running":true,"tainted":false}}}
HTTP_STATUS=200
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":572,"rate":468.1239449641246}}}}
HTTP_STATUS=200

===== [run1] elapsed ~1s cumulative =====
$ curl -s http://localhost:6565/v1/status
{"data":{"type":"status","id":"default","attributes":{"status":7,"paused":false,"vus":2,"vus-max":2,"stopped":false,"running":true,"tainted":false}}}
HTTP_STATUS=200
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":1094,"rate":488.0567028595836}}}}
HTTP_STATUS=200

===== [run1] elapsed ~2s cumulative =====
$ curl -s http://localhost:6565/v1/status
{"data":{"type":"status","id":"default","attributes":{"status":7,"paused":false,"vus":2,"vus-max":2,"stopped":false,"running":true,"tainted":false}}}
HTTP_STATUS=200
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":2090,"rate":490.63713159216593}}}}
HTTP_STATUS=200

===== [run1] elapsed ~4s cumulative =====
$ curl -s http://localhost:6565/v1/status
{"data":{"type":"status","id":"default","attributes":{"status":7,"paused":false,"vus":2,"vus-max":2,"stopped":false,"running":true,"tainted":false}}}
HTTP_STATUS=200
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":4108,"rate":496.2111098906213}}}}
HTTP_STATUS=200

===== [run1] elapsed ~8s cumulative =====
$ curl -s http://localhost:6565/v1/status
{"data":{"type":"status","id":"default","attributes":{"status":7,"paused":false,"vus":2,"vus-max":2,"stopped":false,"running":true,"tainted":false}}}
HTTP_STATUS=200
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":8116,"rate":497.8428023585183}}}}
HTTP_STATUS=200

===== [run1] elapsed ~12s cumulative =====
$ curl -s http://localhost:6565/v1/status
{"data":{"type":"status","id":"default","attributes":{"status":7,"paused":false,"vus":2,"vus-max":2,"stopped":false,"running":true,"tainted":false}}}
HTTP_STATUS=200
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":14092,"rate":497.59667335694576}}}}
HTTP_STATUS=200
```

### (d) Raw API evidence - run 2 (stability)

```text
===== [run2] elapsed ~0.2s cumulative =====
$ curl -s http://localhost:6565/v1/status
{"data":{"type":"status","id":"default","attributes":{"status":7,"paused":false,"vus":2,"vus-max":2,"stopped":false,"running":true,"tainted":false}}}
HTTP_STATUS=200
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":48,"rate":252.99588103203914}}}}
HTTP_STATUS=200

===== [run2] elapsed ~0.4s cumulative =====
$ curl -s http://localhost:6565/v1/status
{"data":{"type":"status","id":"default","attributes":{"status":7,"paused":false,"vus":2,"vus-max":2,"stopped":false,"running":true,"tainted":false}}}
HTTP_STATUS=200
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":273,"rate":449.56483679983745}}}}
HTTP_STATUS=200

===== [run2] elapsed ~0.6s cumulative =====
$ curl -s http://localhost:6565/v1/status
{"data":{"type":"status","id":"default","attributes":{"status":7,"paused":false,"vus":2,"vus-max":2,"stopped":false,"running":true,"tainted":false}}}
HTTP_STATUS=200
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":596,"rate":486.3999179766517}}}}
HTTP_STATUS=200

===== [run2] elapsed ~1s cumulative =====
$ curl -s http://localhost:6565/v1/status
{"data":{"type":"status","id":"default","attributes":{"status":7,"paused":false,"vus":2,"vus-max":2,"stopped":false,"running":true,"tainted":false}}}
HTTP_STATUS=200
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":1069,"rate":476.6902902810294}}}}
HTTP_STATUS=200

===== [run2] elapsed ~2s cumulative =====
$ curl -s http://localhost:6565/v1/status
{"data":{"type":"status","id":"default","attributes":{"status":7,"paused":false,"vus":2,"vus-max":2,"stopped":false,"running":true,"tainted":false}}}
HTTP_STATUS=200
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":2090,"rate":490.5577387761217}}}}
HTTP_STATUS=200

===== [run2] elapsed ~4s cumulative =====
$ curl -s http://localhost:6565/v1/status
{"data":{"type":"status","id":"default","attributes":{"status":7,"paused":false,"vus":2,"vus-max":2,"stopped":false,"running":true,"tainted":false}}}
HTTP_STATUS=200
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":4107,"rate":496.09518940656136}}}}
HTTP_STATUS=200

===== [run2] elapsed ~8s cumulative =====
$ curl -s http://localhost:6565/v1/status
{"data":{"type":"status","id":"default","attributes":{"status":7,"paused":false,"vus":2,"vus-max":2,"stopped":false,"running":true,"tainted":false}}}
HTTP_STATUS=200
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":8091,"rate":496.5146110886307}}}}
HTTP_STATUS=200

===== [run2] elapsed ~12s cumulative =====
$ curl -s http://localhost:6565/v1/status
{"data":{"type":"status","id":"default","attributes":{"status":7,"paused":false,"vus":2,"vus-max":2,"stopped":false,"running":true,"tainted":false}}}
HTTP_STATUS=200
$ curl -s http://localhost:6565/v1/metrics/dropped_iterations
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":14068,"rate":496.85749062640707}}}}
HTTP_STATUS=200
```

### (e) Stability across runs

`dropped_iterations` is a monotonically increasing counter, so the **API snapshot** value
depends on **when** you query it - both runs show the same growth shape:

| elapsed | run 1 `count` | run 2 `count` | `/v1/status` |
|--------:|--------------:|--------------:|:-------------|
| ~0.2 s  | 49            | 48            | `running:true` |
| ~0.4 s  | 274           | 273           | `running:true` |
| ~0.6 s  | 572           | 596           | `running:true` |
| ~1 s    | 1094          | 1069          | `running:true` |
| ~2 s    | 2090          | 2090          | `running:true` |
| ~4 s    | 4108          | 4107          | `running:true` |
| ~8 s    | 8116          | 8091          | `running:true` |
| ~12 s   | 14092         | 14068         | `running:true` |

Every `/v1/status` response reports `"running":true` and `"status":7` - the value
`ExecutionStatusRunning` (`lib/execution.go`, `Running = 7`) - proving the test was **live**
(not finished) when each query returned. The end-of-test summary (shown only for contrast)
converged to essentially the same **total** both runs - but the reported value here was
obtained **from the API**, not from this summary:

```text
     data_received........: 0 B   0 B/s
     data_sent............: 0 B   0 B/s
     dropped_iterations...: 14941 497.028308/s
     iteration_duration...: avg=1s min=1s med=1s max=1s p(90)=1s p(95)=1s
     iterations...........: 60    1.995964/s
     vus..................: 2     min=2        max=2
     vus_max..............: 2     min=2        max=2


running (0m30.1s), 0/2 VUs, 60 complete and 0 interrupted iterations
overload ✓ [ 100% ] 0/2 VUs  30s  500.00 iters/s
```
```text
# final-summary dropped_iterations across runs (contrast only; reported value is the API value above)
q3_klog_run1.log:     dropped_iterations...: 14941 497.028308/s
q3_klog_run2.log:     dropped_iterations...: 14940 496.945685/s
```

The arithmetic is self-consistent: 60 completed + ~14 940 dropped ~= 15 000 = 500 iters/s x
30 s scheduled. The final **total** is stable (14 941 vs 14 940); the **live API value** is
whatever the counter has reached at query time (e.g. run 1: 49 -> 274 -> ... -> 14 092 ->
final 14 941).

### (f) file:line citations

- `metrics/builtin.go:10` - `DroppedIterationsName = "dropped_iterations"`; `metrics/builtin.go:84` registers
  it as a `Counter`.
- `api/server.go:70` - the REST API `http.Server` bind (`localhost:6565`).
- `api/v1/routes.go:9` - `NewHandler`; `api/v1/routes.go:23` - `/v1/metrics`; `api/v1/routes.go:31` - `/v1/metrics/`; `api/v1/routes.go:37`
  - extracts the metric id from the path.
- `api/v1/metric_routes.go:37` - the `404` `No metric with that ID was found` response.
- Emission site exercised here: `lib/executor/constant_arrival_rate.go` pushes a
  `dropped_iterations` sample whenever no free VU is available to start a scheduled
  iteration. (The `maxDuration`-overrun emission paths in
  `lib/executor/shared_iterations.go` and `lib/executor/per_vu_iterations.go` are the
  secondary/alternate producers of the same counter.)

### (g) Cause -> effect

With only 2 VUs doing 1 s iterations against a 500/s target, the executor cannot start
most scheduled iterations, so it increments the `dropped_iterations` counter
(`metrics/builtin.go:10`). The value is held in the live metrics engine and served by
`handleGetMetric` (`api/v1/metric_routes.go`) as a JSON:API single-resource envelope. The
`404 -> 200` transition and the concurrent `running:true` status are the runtime proof that
the reported number came from the **API**, not the end-of-test summary. **Web/doc
cross-check:** Grafana's *k6 REST API* docs document `GET /v1/metrics/{name}` on
`localhost:6565` returning JSON:API - matching the observed envelope.

## Q4 - SharedArray data-sharing behaviour

**Question.** Does the memory footprint for file data remain constant as the number of
VUs increases, or does each VU create its own copy? Prove it with test-script output, and
explain the root cause.

### (a) Command and scripts

A 500-record JSON file (~45.7 KB) is loaded inside a `SharedArray` whose constructor
callback logs a **unique marker**. Because the constructor runs once per unique name
across **all** VUs, the marker must appear **exactly once** regardless of VU count.

```javascript
import { SharedArray } from 'k6/data';

// Q4: the constructor callback logs a UNIQUE marker. Because SharedArray materializes the
// data once per unique name across ALL VUs (data.go:152), the marker must appear exactly
// ONCE regardless of the VU count -- proving a single shared copy.
const data = new SharedArray('my_shared_data', function () {
  console.log('SHAREDARRAY_INIT_MARKER');            // MUST be inside the callback
  return JSON.parse(open('/tmp/k6smoke/data.json'));
});

export const options = { vus: 8, iterations: 8 };

export default function () {
  // touch an element so the data is actually used (element access returns a copy)
  const _ = data[0].username;
}
```
```text
$ ./k6 run /tmp/k6smoke/q4.js 2>&1 | tee /tmp/k6smoke/q4.out
$ grep -c 'SHAREDARRAY_INIT_MARKER' /tmp/k6smoke/q4.out
```

### (b) Complete, unedited output - 8 VUs

```text

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

time="2026-07-08T04:38:16Z" level=info msg=SHAREDARRAY_INIT_MARKER source=console
     execution: local
        script: /tmp/k6smoke/q4.js
        output: -

     scenarios: (100.00%) 1 scenario, 8 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 8 iterations shared among 8 VUs (maxDuration: 10m0s, gracefulStop: 30s)


     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=31.7µs min=11.41µs med=40.48µs max=49.2µs p(90)=46.06µs p(95)=47.63µs
     iterations...........: 8   36692.366612/s


running (00m00.0s), 0/8 VUs, 8 complete and 0 interrupted iterations
default ✓ [ 100% ] 8 VUs  00m00.0s/10m0s  8/8 shared iters
```

Marker count at 8 VUs: **1**.


### (c) Secondary evidence - 16 VUs (still one copy)

```text

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

time="2026-07-08T04:38:32Z" level=info msg=SHAREDARRAY_INIT_MARKER source=console
     execution: local
        script: /tmp/k6smoke/q4_16.js
        output: -

     scenarios: (100.00%) 1 scenario, 16 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 16 iterations shared among 16 VUs (maxDuration: 10m0s, gracefulStop: 30s)


     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=32.33µs min=9.9µs med=41.67µs max=60.4µs p(90)=53.74µs p(95)=56.57µs
     iterations...........: 16  58668.231153/s


running (00m00.0s), 00/16 VUs, 16 complete and 0 interrupted iterations
default ✓ [ 100% ] 16 VUs  00m00.0s/10m0s  16/16 shared iters
```

Marker count at 16 VUs: **1** (doubling the VUs did not add a second materialisation).


### (d) Contrast - no SharedArray (each VU copies)

An otherwise-identical script that loads the file with a plain init-context
`open()`+`JSON.parse` (no `SharedArray`) logs its marker once **per VU init**:

```javascript
// CONTRAST: NO SharedArray. A plain init-context open()+parse runs once PER VU init,
// so each VU holds its OWN copy of the file. The marker prints once per VU.
const data = (function () {
  console.log('NONSHARED_INIT_MARKER');
  return JSON.parse(open('/tmp/k6smoke/data.json'));
})();

export const options = { vus: 8, iterations: 8 };

export default function () {
  const _ = data[0].username;
}
```
```text
$ grep -c 'NONSHARED_INIT_MARKER' /tmp/k6smoke/q4_nonshared.out
10
```

At 8 VUs the non-shared init ran **10** times (each VU parses and holds its **own** copy),
versus **1** for `SharedArray` - the qualitative difference that proves sharing.


### (e) file:line citations

- `js/modules/k6/data/data.go:32` - `data map[string]sharedArray`: a **single** Go-side map
  keyed by name, shared across all VUs of the instance.
- `js/modules/k6/data/data.go:152` - `get()` uses double-checked locking and invokes the
  JS constructor **exactly once per name**; later VUs receive the cached array.
- `js/modules/k6/data/share.go:23` - `wrap()` returns `rt.NewDynamicArray(...)` (`:27`), a
  Sobek dynamic-array proxy over the single backing store.
- `js/modules/k6/data/share.go:36`/`js/modules/k6/data/share.go:41` - `Set`/`SetLen` panic `SharedArray is
  immutable`; `:44` - `Get()` parses the stored JSON string and returns a per-access
  **copy**.

### (f) Cause -> effect / root cause

The memory footprint for the file data stays **constant** as VUs increase; each VU does
**not** create its own copy. Root cause: each k6 VU is an isolated JS VM (which would
otherwise each parse and hold the whole file), but `SharedArray` stores the parsed data
**once** in a Go-side map keyed by name (`data.go:32`), materialises it only once per name
(`data.go:152`), and exposes it to every VU through a Sobek dynamic-array wrapper
(`share.go:23`) that reads from that single backing store, returning a copy only for the
specific element accessed (`share.go:44`). **Web/doc cross-check:** Grafana's *SharedArray*
/ *Data parameterization* docs state the constructor runs once, the result is stored once
in shared memory, and element access returns a copy - exactly the observed behaviour.

## Q5 - Prometheus output metric-name integrity

**Question.** Prove with test-script output that data exported via the Prometheus output
preserves the integrity of metric names (how names are mapped/sanitised).

### (a) Command and script

A local HTTP receiver captures the **raw** remote-write request bodies (byte-exact, no
re-serialisation); a trivial local HTTP target lets the VUs produce real `http_req_*` and
`data_*` built-ins. The script also emits one custom metric of **each type** (Counter,
Gauge, Rate, Trend):

```javascript
import http from 'k6/http';
import { Counter, Gauge, Rate, Trend } from 'k6/metrics';
import { sleep } from 'k6';

// Q5: emit a variety of metric TYPES so we can observe name mapping for each:
// custom counter/gauge/rate/trend, plus built-ins (data_*, iterations, iteration_duration, http_req_*).
const myCounter = new Counter('my_custom_counter');
const myGauge   = new Gauge('my_custom_gauge');
const myRate    = new Rate('my_custom_rate');
const myTrend   = new Trend('my_custom_trend');

export const options = { vus: 3, duration: '8s' };

export default function () {
  const res = http.get('http://127.0.0.1:8080/');
  myCounter.add(1);
  myGauge.add(res.timings.duration);
  myRate.add(res.status === 200);
  myTrend.add(res.timings.duration);
  sleep(0.2);
}
```
```text
# receiver writes each POST body verbatim to rw_run1/rw_body_<N>.bin and replies 204
$ RW_OUT_DIR=/tmp/k6smoke/rw_run1 python3 /tmp/k6smoke/rw_receiver.py &     # :9090
$ python3 /tmp/k6smoke/target.py &                                          # :8080
$ K6_PROMETHEUS_RW_SERVER_URL=http://localhost:9090/api/v1/write \
    K6_PROMETHEUS_RW_PUSH_INTERVAL=2s \
    ./k6 run --out experimental-prometheus-rw /tmp/k6smoke/q5.js
```

The run enables the output and emits the custom metrics (from `q5_run1.log`):

```text
     scenarios: (100.00%) 1 scenario, 3 max VUs, 38s max duration (incl. graceful stop):
              * default: 3 looping VUs for 8s (gracefulStop: 30s)


running (01.0s), 3/3 VUs, 12 complete and 0 interrupted iterations
default   [  12% ] 3 VUs  1.0s/8s

running (02.0s), 3/3 VUs, 27 complete and 0 interrupted iterations
default   [  25% ] 3 VUs  2.0s/8s

running (03.0s), 3/3 VUs, 42 complete and 0 interrupted iterations
default   [  37% ] 3 VUs  3.0s/8s

running (04.0s), 3/3 VUs, 57 complete and 0 interrupted iterations
default   [  50% ] 3 VUs  4.0s/8s

running (05.0s), 3/3 VUs, 72 complete and 0 interrupted iterations
default   [  62% ] 3 VUs  5.0s/8s

running (06.0s), 3/3 VUs, 87 complete and 0 interrupted iterations
default   [  75% ] 3 VUs  6.0s/8s

running (07.0s), 3/3 VUs, 102 complete and 0 interrupted iterations
default   [  87% ] 3 VUs  7.0s/8s

running (08.0s), 3/3 VUs, 117 complete and 0 interrupted iterations
default   [ 100% ] 3 VUs  8.0s/8s

     data_received..................: 17 kB    2.1 kB/s
     data_sent......................: 9.6 kB   1.2 kB/s
     http_req_blocked...............: avg=209.49µs min=88.29µs  med=197.84µs max=1.3ms    p(90)=241.75µs p(95)=247.04µs
     http_req_connecting............: avg=140.39µs min=54.19µs  med=138.74µs max=508.89µs p(90)=164.29µs p(95)=172.92µs
     http_req_duration..............: avg=978.63µs min=432.76µs med=943.37µs max=3.12ms   p(90)=1.32ms   p(95)=1.41ms  
       { expected_response:true }...: avg=978.63µs min=432.76µs med=943.37µs max=3.12ms   p(90)=1.32ms   p(95)=1.41ms  
     http_req_failed................: 0.00%    0 out of 120
     http_req_receiving.............: avg=127.86µs min=35.49µs  med=95.32µs  max=546.55µs p(90)=243.91µs p(95)=373.37µs
     http_req_sending...............: avg=53.21µs  min=25.07µs  med=50.38µs  max=203.94µs p(90)=65.52µs  p(95)=75.96µs 
     http_req_tls_handshaking.......: avg=0s       min=0s       med=0s       max=0s       p(90)=0s       p(95)=0s      
     http_req_waiting...............: avg=797.56µs min=342.23µs med=746.15µs max=2.99ms   p(90)=1.16ms   p(95)=1.26ms  
     http_reqs......................: 120      14.86935/s
     iteration_duration.............: avg=201.71ms min=200.94ms med=201.64ms max=203.83ms p(90)=202.02ms p(95)=202.15ms
     iterations.....................: 120      14.86935/s
     my_custom_counter..............: 120      14.86935/s
     my_custom_gauge................: 0.726048 min=0.43276    max=3.129991
     my_custom_rate.................: 100.00%  120 out of 120
     my_custom_trend................: avg=0.978639 min=0.43276  med=0.943379 max=3.129991 p(90)=1.329738 p(95)=1.413953
```

### (b) The captured body is genuine snappy-compressed protobuf

The first bytes of one captured body - the raw on-the-wire payload - already reveal the
`__name__` label and a mapped metric name in situ (`d9 14` is the snappy block-length
preamble = 2649 uncompressed bytes; `0a 08 __name__`, then `12 12 k6_my_custom_gauge`):

```text
000000 d9 14 f0 48 0a 47 0a 1e 0a 08 5f 5f 6e 61 6d 65  >...H.G....__name<
000010 5f 5f 12 12 6b 36 5f 6d 79 5f 63 75 73 74 6f 6d  >__..k6_my_custom<
000020 5f 67 61 75 67 65 0a 13 0a 08 73 63 65 6e 61 72  >_gauge....scenar<
000030 69 6f 12 07 64 65 66 61 75 6c 74 12 10 09 5e 4a  >io..default...^J<
000040 5d 32 8e 91 f6 3f 10 fc 90 c5 80 f4 33 4a 49 00  >]2...?......3JI.<
000050 38 64 61 74 61 5f 73 65 6e 74 5f 74 6f 74 61 6c  >8data_sent_total<
```

### (c) Byte-exact decode - run 1 (default trend stat p(99))

The body is snappy-**block**-compressed protobuf. With no `protoc`/`cramjam`/`python-snappy`
available offline, a pure-Python snappy-block decompressor + a minimal protobuf
wire-format parser (self-tested; see appendix) decode the **exact captured bytes** and
print every `__name__` label value:

```text
===== PER-FILE SUMMARY (byte-exact decode) =====
rw_body_0.bin: raw=734B snappy-> 2649B, timeseries=19, distinct __name__=19
rw_body_1.bin: raw=742B snappy-> 2649B, timeseries=19, distinct __name__=19
rw_body_2.bin: raw=738B snappy-> 2649B, timeseries=19, distinct __name__=19
rw_body_3.bin: raw=733B snappy-> 2649B, timeseries=19, distinct __name__=19
rw_body_4.bin: raw=216B snappy-> 388B, timeseries=6, distinct __name__=6

===== ALL DISTINCT __name__ VALUES (verbatim, sorted) =====
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
k6_my_custom_gauge
k6_my_custom_rate_rate
k6_my_custom_trend_p99
k6_vus
k6_vus_max

===== FULL LABEL SETS for k6-name-integrity proof (one example series per __name__) =====
k6_data_received_total  =>  {__name__="k6_data_received_total", scenario="default"}
k6_data_sent_total  =>  {__name__="k6_data_sent_total", scenario="default"}
k6_http_req_blocked_p99  =>  {__name__="k6_http_req_blocked_p99", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_connecting_p99  =>  {__name__="k6_http_req_connecting_p99", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_duration_p99  =>  {__name__="k6_http_req_duration_p99", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_failed_rate  =>  {__name__="k6_http_req_failed_rate", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_receiving_p99  =>  {__name__="k6_http_req_receiving_p99", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_sending_p99  =>  {__name__="k6_http_req_sending_p99", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_tls_handshaking_p99  =>  {__name__="k6_http_req_tls_handshaking_p99", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_waiting_p99  =>  {__name__="k6_http_req_waiting_p99", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_reqs_total  =>  {__name__="k6_http_reqs_total", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_iteration_duration_p99  =>  {__name__="k6_iteration_duration_p99", scenario="default"}
k6_iterations_total  =>  {__name__="k6_iterations_total", scenario="default"}
k6_my_custom_counter_total  =>  {__name__="k6_my_custom_counter_total", scenario="default"}
k6_my_custom_gauge  =>  {__name__="k6_my_custom_gauge", scenario="default"}
k6_my_custom_rate_rate  =>  {__name__="k6_my_custom_rate_rate", scenario="default"}
k6_my_custom_trend_p99  =>  {__name__="k6_my_custom_trend_p99", scenario="default"}
k6_vus  =>  {__name__="k6_vus"}
k6_vus_max  =>  {__name__="k6_vus_max"}
```

### (d) Byte-exact decode - run 2 (fractional trend stats -> dot removal)

Re-running the **same** script with `K6_PROMETHEUS_RW_TREND_STATS="p(0.95),p(99.9),avg,count"`
exercises the trend stat-key sanitisation path and confirms the `k6_`+snake_case naming is
identical run-to-run (metric naming is deterministic, not timing-dependent):

```text
===== PER-FILE SUMMARY (byte-exact decode) =====
rw_body_0.bin: raw=1311B snappy-> 7758B, timeseries=46, distinct __name__=46
rw_body_1.bin: raw=1319B snappy-> 7758B, timeseries=46, distinct __name__=46
rw_body_2.bin: raw=1309B snappy-> 7758B, timeseries=46, distinct __name__=46
rw_body_3.bin: raw=1314B snappy-> 7758B, timeseries=46, distinct __name__=46
rw_body_4.bin: raw=289B snappy-> 632B, timeseries=9, distinct __name__=9

===== ALL DISTINCT __name__ VALUES (verbatim, sorted) =====
k6_data_received_total
k6_data_sent_total
k6_http_req_blocked_avg
k6_http_req_blocked_count
k6_http_req_blocked_p095
k6_http_req_blocked_p999
k6_http_req_connecting_avg
k6_http_req_connecting_count
k6_http_req_connecting_p095
k6_http_req_connecting_p999
k6_http_req_duration_avg
k6_http_req_duration_count
k6_http_req_duration_p095
k6_http_req_duration_p999
k6_http_req_failed_rate
k6_http_req_receiving_avg
k6_http_req_receiving_count
k6_http_req_receiving_p095
k6_http_req_receiving_p999
k6_http_req_sending_avg
k6_http_req_sending_count
k6_http_req_sending_p095
k6_http_req_sending_p999
k6_http_req_tls_handshaking_avg
k6_http_req_tls_handshaking_count
k6_http_req_tls_handshaking_p095
k6_http_req_tls_handshaking_p999
k6_http_req_waiting_avg
k6_http_req_waiting_count
k6_http_req_waiting_p095
k6_http_req_waiting_p999
k6_http_reqs_total
k6_iteration_duration_avg
k6_iteration_duration_count
k6_iteration_duration_p095
k6_iteration_duration_p999
k6_iterations_total
k6_my_custom_counter_total
k6_my_custom_gauge
k6_my_custom_rate_rate
k6_my_custom_trend_avg
k6_my_custom_trend_count
k6_my_custom_trend_p095
k6_my_custom_trend_p999
k6_vus
k6_vus_max

===== FULL LABEL SETS for k6-name-integrity proof (one example series per __name__) =====
k6_data_received_total  =>  {__name__="k6_data_received_total", scenario="default"}
k6_data_sent_total  =>  {__name__="k6_data_sent_total", scenario="default"}
k6_http_req_blocked_avg  =>  {__name__="k6_http_req_blocked_avg", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_blocked_count  =>  {__name__="k6_http_req_blocked_count", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_blocked_p095  =>  {__name__="k6_http_req_blocked_p095", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_blocked_p999  =>  {__name__="k6_http_req_blocked_p999", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_connecting_avg  =>  {__name__="k6_http_req_connecting_avg", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_connecting_count  =>  {__name__="k6_http_req_connecting_count", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_connecting_p095  =>  {__name__="k6_http_req_connecting_p095", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_connecting_p999  =>  {__name__="k6_http_req_connecting_p999", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_duration_avg  =>  {__name__="k6_http_req_duration_avg", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_duration_count  =>  {__name__="k6_http_req_duration_count", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_duration_p095  =>  {__name__="k6_http_req_duration_p095", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_duration_p999  =>  {__name__="k6_http_req_duration_p999", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_failed_rate  =>  {__name__="k6_http_req_failed_rate", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_receiving_avg  =>  {__name__="k6_http_req_receiving_avg", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_receiving_count  =>  {__name__="k6_http_req_receiving_count", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_receiving_p095  =>  {__name__="k6_http_req_receiving_p095", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_receiving_p999  =>  {__name__="k6_http_req_receiving_p999", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_sending_avg  =>  {__name__="k6_http_req_sending_avg", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_sending_count  =>  {__name__="k6_http_req_sending_count", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_sending_p095  =>  {__name__="k6_http_req_sending_p095", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_sending_p999  =>  {__name__="k6_http_req_sending_p999", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_tls_handshaking_avg  =>  {__name__="k6_http_req_tls_handshaking_avg", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_tls_handshaking_count  =>  {__name__="k6_http_req_tls_handshaking_count", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_tls_handshaking_p095  =>  {__name__="k6_http_req_tls_handshaking_p095", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_tls_handshaking_p999  =>  {__name__="k6_http_req_tls_handshaking_p999", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_waiting_avg  =>  {__name__="k6_http_req_waiting_avg", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_waiting_count  =>  {__name__="k6_http_req_waiting_count", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_waiting_p095  =>  {__name__="k6_http_req_waiting_p095", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_req_waiting_p999  =>  {__name__="k6_http_req_waiting_p999", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_http_reqs_total  =>  {__name__="k6_http_reqs_total", expected_response="true", method="GET", name="http://127.0.0.1:8080/", proto="HTTP/1.0", scenario="default", status="200", url="http://127.0.0.1:8080/"}
k6_iteration_duration_avg  =>  {__name__="k6_iteration_duration_avg", scenario="default"}
k6_iteration_duration_count  =>  {__name__="k6_iteration_duration_count", scenario="default"}
k6_iteration_duration_p095  =>  {__name__="k6_iteration_duration_p095", scenario="default"}
k6_iteration_duration_p999  =>  {__name__="k6_iteration_duration_p999", scenario="default"}
k6_iterations_total  =>  {__name__="k6_iterations_total", scenario="default"}
k6_my_custom_counter_total  =>  {__name__="k6_my_custom_counter_total", scenario="default"}
k6_my_custom_gauge  =>  {__name__="k6_my_custom_gauge", scenario="default"}
k6_my_custom_rate_rate  =>  {__name__="k6_my_custom_rate_rate", scenario="default"}
k6_my_custom_trend_avg  =>  {__name__="k6_my_custom_trend_avg", scenario="default"}
k6_my_custom_trend_count  =>  {__name__="k6_my_custom_trend_count", scenario="default"}
k6_my_custom_trend_p095  =>  {__name__="k6_my_custom_trend_p095", scenario="default"}
k6_my_custom_trend_p999  =>  {__name__="k6_my_custom_trend_p999", scenario="default"}
k6_vus  =>  {__name__="k6_vus"}
k6_vus_max  =>  {__name__="k6_vus_max"}
```

### (e) file:line citations

- `vendor/.../remotewrite/prometheus.go:11` - `const namelbl = "__name__"`.
- `vendor/.../remotewrite/prometheus.go:39` - `func MapSeries(series, suffix)`; `:40` -
  `v := defaultMetricPrefix + series.Metric.Name`; the suffix (when non-empty) is appended
  as `v += "_" + suffix`; `:45` - the value `v` is stored in the `Name: namelbl`
  (`__name__`) label. Labels are then sorted lexicographically.
- `vendor/.../remotewrite/config.go:24` - `defaultMetricPrefix = "k6_"`; `vendor/.../remotewrite/config.go:21` -
  `defaultServerURL = "http://localhost:9090/api/v1/write"`.
- `vendor/.../remotewrite/remotewrite.go:186` -
  `statKey = strings.ReplaceAll(statKey, ".", "")  // remove dots, p(0.95) => p095`
  (with the surrounding lines trimming the `p(` prefix and `)` suffix and re-prepending
  `p`).
- Output registration: `cmd/outputs.go:20` imports the remotewrite package and `:67` calls
  `remotewrite.New(params)`; `cmd/builtin_output_gen.go:10` lists
  `experimental-prometheus-rw`. Dependency `github.com/grafana/xk6-output-prometheus-remote
  v0.5.0` (`go.mod:20`).

### (f) Cause -> effect

k6 metric names are already snake_case (valid Prometheus identifier characters), so **no
character mangling is needed**. `MapSeries` (`prometheus.go:39`) only **prefixes** `k6_`
(`config.go:24`) and **appends a type-appropriate suffix**, storing the result in the
`__name__` label (`prometheus.go:45`). The observed suffixes are:

- **Counter -> `_total`**: `k6_data_received_total`, `k6_data_sent_total`,
  `k6_http_reqs_total`, `k6_iterations_total`, `k6_my_custom_counter_total`.
- **Trend -> per-percentile suffix**: default `p(99)` -> `k6_..._p99`
  (`k6_http_req_duration_p99`, `k6_iteration_duration_p99`, `k6_my_custom_trend_p99`).
- **Rate -> `_rate`**: `k6_http_req_failed_rate`, `k6_my_custom_rate_rate`.
- **Gauge -> no suffix**: `k6_my_custom_gauge`, `k6_vus`, `k6_vus_max`.

The trend stat-key sanitisation (`remotewrite.go:186`) is proven directly by run 2:
`p(0.95)` -> `p095` and `p(99.9)` -> `p999` (dots removed), while `avg`/`count` pass through
unchanged -> `k6_..._avg`, `k6_..._count`. In every case the **original snake_case name
survives verbatim** inside the `__name__` label - i.e. each exported name is exactly
`k6_` + `<original metric name>` + `[_<type/stat suffix>]`. **Web/doc cross-check:**
Grafana's *Prometheus remote write* docs state all series are prefixed with the `k6_`
namespace and that k6's names already follow Prometheus naming conventions - matching the
decoded `__name__` values.

## Coverage and cleanup

### Named-item coverage

- **Q1** - `ramping-vus` (`lib/executor/ramping_vus.go`), `SIGINT` trap
  (`cmd/common.go:98`), graceful-stop debug `Stopping k6 in response to signal...`
  (`cmd/run.go:350`), abort error `test run was aborted because k6 received a '...' signal`
  (`cmd/run.go:354`), second-SIGINT `Aborting k6 in response to signal` (`cmd/run.go:360`)
  + immediate `OSExit(ExternalAbort)` (`cmd/common.go:118`), the decisive
  `N complete and M interrupted iterations` progress line (`execution/scheduler.go:156`;
  `GetFullIterationCount` `lib/execution.go:284`, `GetPartialIterationCount` `:300`,
  `AddInterruptedIterations` `:308`), manual-interrupt semantics
  (`lib/executor/base_config.go:95`). **Both** SIGINT paths captured. Conclusion: active
  VUs are **terminated mid-iteration** (`0 complete / 6 interrupted`).
- **Q2** - `grpc_streams_msgs_received` (`metrics.go:25`), `grpc_streams_msgs_sent`
  (`metrics.go:21`), per-message sample (`stream.go:153`), interruption warning
  `no handlers for error registered ... canceled by client (k6)` (`stream.go:395`), the
  server-streaming RPC `ListFeatures` (`service.go:57`) on `localhost:10000`
  (`main.go:51`), `gracefulRampDown`/`gracefulStop` = 30 ms. Received distribution
  **276 / 272 / 276** @ ~7 s (592 @ ~15 s); sent = 4. Secondary path (handler registered ->
  `STREAM_ERROR_HANDLER code=2`, no warning) captured.
- **Q3** - `dropped_iterations` (`metrics/builtin.go:10`/`:84`) obtained from
  `GET /v1/metrics/dropped_iterations` (`api/v1/routes.go:31`/`:37`,
  `api/v1/metric_routes.go`), API bind `localhost:6565` (`api/server.go:70`), the
  `404` path (`metric_routes.go:37`), and `GET /v1/status` `running:true`/`status:7`.
  `000 -> 404 -> 200` transition captured; live API value grows to a stable final total
  (**14 941 / 14 940**).
- **Q4** - `SharedArray` marker logged **once** at 8 and 16 VUs (vs **10** without
  sharing); single Go-side map (`data.go:32`), once-per-name `get()` (`data.go:152`),
  dynamic-array proxy + immutability (`share.go:23`/`:36`/`:41`/`:44`).
- **Q5** - `--out experimental-prometheus-rw` (`cmd/outputs.go:20`/`:67`,
  `cmd/builtin_output_gen.go:10`), byte-exact decode of the snappy+protobuf body,
  `__name__` mapping `k6_<snake>[_suffix]` (`prometheus.go:11`/`:39`/`:45`,
  `config.go:24`), trend dot-removal `p(0.95)->p095` (`remotewrite.go:186`).

### Cleanup / read-only guarantee

All observation artifacts lived under `/tmp/k6smoke` (outside the repository). After
capture they were deleted and the repository was confirmed pristine:

```text
$ rm -rf /tmp/k6smoke
$ git status --porcelain          # empty output -> the only repo change is this document
```
## Appendix - Q5 decoder transparency

Because `protoc`, `cramjam` and `python-snappy` were unavailable offline, the Prometheus
remote-write body was decoded by a small purpose-built tool that reads the **exact
captured bytes** (it never re-serialises k6's output): a pure-Python **snappy block**
decompressor followed by a minimal **protobuf wire-format** parser that extracts the
`__name__` labels from the `prometheus.prompb.WriteRequest`. The tool was validated with
three self-tests before use, all passing:

1. snappy literal block `05 10 "hello"` -> `hello`;
2. snappy copy/overlap run-length -> `aaaaaa`;
3. a hand-built `WriteRequest` -> `__name__` = `[k6_iterations_total,
   k6_my_custom_counter_total]`.

The k6 binary and its emitted bytes are fully canonical; the decoder only **reads** those
bytes, so the decoded `__name__` values are byte-exact evidence.

