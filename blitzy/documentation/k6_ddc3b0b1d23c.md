# k6 v0.55.0 — Runtime Behavior Investigation (branch `k6_ddc3b0b1d23c`)

This document answers five runtime-behavior questions about Grafana **k6 v0.55.0**
(`go.k6.io/k6`, repo HEAD `ddc3b0b1d23c128e34e2792fc9075f9126e32375`). Every behavioral claim
below is backed by **actual output captured from real runs** of the canonical `./k6` binary
through its normal `./k6 run` CLI entry point (`main.go:L8-L9` → `cmd.Execute()`) — nothing here
is inferred from source alone unless explicitly labelled **Inferred**. The binary under test was
built with `go build -o ./k6 .` and reports the banner
`k6 v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)` (version constant `const Version = "0.55.0"`
at `lib/consts/consts.go:L12`). Each magnitude/count/timing value was confirmed stable across at
least two runs; where a value legitimately varies run-to-run, the variance and the fixed
methodology are stated. See **Methodology & Reproducibility** at the end for the exact environment,
commands, and the final `git status` proving the repository was left unchanged.

---

## Q1 — VU lifecycle on SIGINT (ramping-vus)

**Direct answer:** On the **first** `SIGINT`, the currently-active VUs are **terminated
mid-iteration — they are NOT allowed to finish their in-progress iteration.** The proof is that a
`ramping-vus` scenario holding **6/6** active VUs (each in a `sleep(8)`) transitions, the instant the
signal arrives, from `0 complete and 0 interrupted iterations` to `0 complete and **6 interrupted**
iterations` — i.e. every one of the 6 in-flight iterations is counted as *interrupted*, not
*complete*. k6 logs `Stopping k6 in response to signal...` (Debug level) and exits with the
external-abort code **105** and the error `test run was aborted because k6 received a 'interrupt' signal`.
Despite its name, the first-signal handler `gracefulStop` immediately cancels the run context; the
graceful window applies only to the *natural* end of a scenario, never to a manual interrupt.

### Command(s) run

Temp script `/tmp/q1_ramping.js`:

```javascript
import { sleep } from 'k6';

export const options = {
  scenarios: {
    ramp: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '3s', target: 6 },   // ramp up to 6 VUs
        { duration: '60s', target: 6 },   // hold 6 VUs
      ],
      gracefulStop: '30s',
      gracefulRampDown: '30s',
    },
  },
};

export default function () {
  sleep(8); // each iteration is in-progress far longer than time-to-signal
}
```

Primary run — a single real `SIGINT` delivered to the real process once ≥5 VUs are active.
`--verbose` is **required** because the key log line is emitted at Debug level:

```bash
./k6 run --verbose /tmp/q1_ramping.js > /tmp/q1.out 2>&1 &
K6PID=$!
sleep 6                 # by ~6s the executor holds 6/6 active VUs, each mid-sleep(8)
kill -INT "$K6PID"      # a REAL SIGINT to the real process (not a programmatic abort)
wait "$K6PID"; echo "exit=$?"
cat /tmp/q1.out
```

### Observed output (complete, unedited — primary single-SIGINT run)

```text
time="2026-07-14T19:34:36Z" level=debug msg="Logger format: TEXT"
time="2026-07-14T19:34:36Z" level=debug msg="k6 version: v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)"

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

time="2026-07-14T19:34:36Z" level=debug msg="Resolving and reading test '/tmp/q1_ramping.js'..."
time="2026-07-14T19:34:36Z" level=debug msg=Loading... moduleSpecifier="file:///tmp/q1_ramping.js" originalModuleSpecifier=/tmp/q1_ramping.js
time="2026-07-14T19:34:36Z" level=debug msg="'/tmp/q1_ramping.js' resolved to 'file:///tmp/q1_ramping.js' and successfully loaded 454 bytes!"
time="2026-07-14T19:34:36Z" level=debug msg="Gathering k6 runtime options..."
time="2026-07-14T19:34:36Z" level=debug msg="Initializing k6 runner for '/tmp/q1_ramping.js' (file:///tmp/q1_ramping.js)..."
time="2026-07-14T19:34:36Z" level=debug msg="Detecting test type for..." test_path="file:///tmp/q1_ramping.js"
time="2026-07-14T19:34:36Z" level=debug msg="Trying to load as a JS test..." test_path="file:///tmp/q1_ramping.js"
time="2026-07-14T19:34:36Z" level=debug msg="Runner successfully initialized!"
time="2026-07-14T19:34:36Z" level=debug msg="Parsing CLI flags..."
time="2026-07-14T19:34:36Z" level=debug msg="Consolidating config layers..."
time="2026-07-14T19:34:36Z" level=debug msg="Parsing thresholds and validating config..."
time="2026-07-14T19:34:36Z" level=debug msg="Initializing the execution scheduler..."
time="2026-07-14T19:34:36Z" level=debug msg="Starting 2 outputs..." component=output-manager
time="2026-07-14T19:34:36Z" level=debug msg=Starting... component=metrics-engine-ingester
time="2026-07-14T19:34:36Z" level=debug msg="Started!" component=metrics-engine-ingester
     execution: local
        script: /tmp/q1_ramping.js
        output: -

     scenarios: (100.00%) 1 scenario, 6 max VUs, 1m33s max duration (incl. graceful stop):
              * ramp: Up to 6 looping VUs for 1m3s over 2 stages (gracefulRampDown: 30s, gracefulStop: 30s)

time="2026-07-14T19:34:36Z" level=debug msg="Trapping interrupt signals so k6 can handle them gracefully..."
time="2026-07-14T19:34:36Z" level=debug msg="Starting the REST API server on localhost:6565"
time="2026-07-14T19:34:36Z" level=debug msg="Starting emission of VUs and VUsMax metrics..."
time="2026-07-14T19:34:36Z" level=debug msg="Start of initialization" executorsCount=1 neededVUs=6 phase=execution-scheduler-init
time="2026-07-14T19:34:36Z" level=debug msg="Initialized VU #4" phase=execution-scheduler-init
time="2026-07-14T19:34:36Z" level=debug msg="Initialized VU #5" phase=execution-scheduler-init
time="2026-07-14T19:34:36Z" level=debug msg="Initialized VU #6" phase=execution-scheduler-init
time="2026-07-14T19:34:36Z" level=debug msg="Initialized VU #2" phase=execution-scheduler-init
time="2026-07-14T19:34:36Z" level=debug msg="Initialized VU #3" phase=execution-scheduler-init
time="2026-07-14T19:34:36Z" level=debug msg="Initialized VU #1" phase=execution-scheduler-init
time="2026-07-14T19:34:36Z" level=debug msg="Finished initializing needed VUs, start initializing executors..." phase=execution-scheduler-init
time="2026-07-14T19:34:36Z" level=debug msg="Initialized executor ramp" phase=execution-scheduler-init
time="2026-07-14T19:34:36Z" level=debug msg="Initialization completed" phase=execution-scheduler-init
time="2026-07-14T19:34:36Z" level=debug msg="Start of test run" executorsCount=1 phase=execution-scheduler-run
time="2026-07-14T19:34:36Z" level=debug msg="setup() is not defined or not exported, skipping!"
time="2026-07-14T19:34:36Z" level=debug msg="Start all executors..." phase=execution-scheduler-run
time="2026-07-14T19:34:36Z" level=debug msg="Starting executor" executor=ramp startTime=0s type=ramping-vus
time="2026-07-14T19:34:36Z" level=debug msg="Starting executor run..." duration=1m3s executor=ramping-vus maxVUs=6 numStages=2 scenario=ramp startVUs=0 type=ramping-vus
time="2026-07-14T19:34:37Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=0

running (0m01.0s), 1/6 VUs, 0 complete and 0 interrupted iterations
ramp   [   2% ] 1/6 VUs  0m01.0s/1m03.0s
time="2026-07-14T19:34:37Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=1
time="2026-07-14T19:34:38Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=2

running (0m02.0s), 3/6 VUs, 0 complete and 0 interrupted iterations
ramp   [   3% ] 3/6 VUs  0m02.0s/1m03.0s
time="2026-07-14T19:34:38Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=3
time="2026-07-14T19:34:39Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=4

running (0m03.0s), 5/6 VUs, 0 complete and 0 interrupted iterations
ramp   [   5% ] 5/6 VUs  0m03.0s/1m03.0s
time="2026-07-14T19:34:39Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=5

running (0m04.0s), 6/6 VUs, 0 complete and 0 interrupted iterations
ramp   [   6% ] 6/6 VUs  0m04.0s/1m03.0s

running (0m05.0s), 6/6 VUs, 0 complete and 0 interrupted iterations
ramp   [   8% ] 6/6 VUs  0m05.0s/1m03.0s
time="2026-07-14T19:34:42Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-14T19:34:42Z" level=debug msg="Metrics emission of VUs and VUsMax metrics stopped"
time="2026-07-14T19:34:42Z" level=debug msg="Executor finished successfully" executor=ramp startTime=0s type=ramping-vus
time="2026-07-14T19:34:42Z" level=debug msg="teardown() is not defined or not exported, skipping!"
time="2026-07-14T19:34:42Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
time="2026-07-14T19:34:42Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-14T19:34:42Z" level=debug msg="Stopping vus and vux_max metrics emission..." phase=execution-scheduler-init
time="2026-07-14T19:34:42Z" level=debug msg="Releasing signal trap..."
time="2026-07-14T19:34:42Z" level=debug msg="Sending usage report..."
time="2026-07-14T19:34:42Z" level=debug msg="Waiting for metrics and traces processing to finish..."
time="2026-07-14T19:34:42Z" level=debug msg="Metrics and traces processing finished!"
time="2026-07-14T19:34:42Z" level=debug msg="Stopping outputs..."
time="2026-07-14T19:34:42Z" level=debug msg="Stopping 2 outputs..." component=output-manager
time="2026-07-14T19:34:42Z" level=debug msg=Stopping... component=metrics-engine-ingester
time="2026-07-14T19:34:42Z" level=debug msg="Stopped!" component=metrics-engine-ingester
time="2026-07-14T19:34:42Z" level=debug msg="Generating the end-of-test summary..."

     data_received...: 0 B 0 B/s
     data_sent.......: 0 B 0 B/s
     vus.............: 6   min=1 max=6
     vus_max.........: 6   min=6 max=6


running (0m06.0s), 0/6 VUs, 0 complete and 6 interrupted iterations
ramp ✗ [   9% ] 2/6 VUs  0m06.0s/1m03.0s
time="2026-07-14T19:34:42Z" level=debug msg="Usage report sent successfully"
time="2026-07-14T19:34:42Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-14T19:34:42Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
exit=105
```

**Before → after (the stateful proof, Rule 2):**

| State | Progress line |
|-------|---------------|
| Before signal (0m05.0s) | `running (0m05.0s), 6/6 VUs, 0 complete and 0 interrupted iterations` |
| After signal (0m06.0s)  | `running (0m06.0s), 0/6 VUs, 0 complete and **6 interrupted** iterations` |

The number of **interrupted** iterations (**6**) equals the number of active VUs (**6**): each VU's
in-progress `sleep(8)` iteration was cut off, and **0** iterations completed. Process **exit code =
105**.

### Explanation & root cause (file:line)

- **Entry point:** `main.go:L8-L9` — `func main()` calls `cmd.Execute()`.
- **Signal trap:** `handleTestAbortSignals` (`cmd/common.go:L97`) registers
  `gs.SignalNotify(sigC, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)` (`cmd/common.go:L101`) on a
  buffered channel of size 2. Its goroutine calls the **graceful-stop** handler on the *first* signal
  and the **hard-stop** handler on the *second*, then `gs.OSExit(...)`.
- **First-signal handler is not actually graceful:** in `cmd/run.go`, `gracefulStop` (`cmd/run.go:L349`)
  logs `"Stopping k6 in response to signal..."` at **Debug** level (`cmd/run.go:L350`) and then
  **immediately calls `runAbort(...)`** (`cmd/run.go:L352`), cancelling the run context at once. It is
  wired via `handleTestAbortSignals(...)` at `cmd/run.go:L363`. Because the run context is cancelled,
  in-flight iterations are cut off and counted as *interrupted*.
- **Why the graceful window does not apply to a manual interrupt:** the graceful window is computed for
  the **natural** end of an executor. `getDurationContexts` (`lib/executor/helpers.go:L168`) sets
  `maxEndTime := startTime.Add(regularDuration + gracefulStop)` (`:L172`), and `trackProgress`
  (`:L184`) only logs `"Regular duration is done, waiting for iterations to gracefully finish"`
  (`:L196`) when the regular duration elapses naturally. A `SIGINT` cancels the parent context first,
  so that graceful path is never taken. The ramping start/stop loop lives in
  `lib/executor/ramping_vus.go`.
- **Exit code:** `105` = `ExternalAbort` (`errext/exitcodes/codes.go:L41`, comment: "the test was
  aborted by an external signal").

### Secondary / edge condition — second signal triggers the hard stop

Sending a **second** trapped signal makes the handler goroutine take its second branch, invoking
`onHardStop` (`cmd/run.go:L359`), which logs `"Aborting k6 in response to signal"` at **Error** level
(`cmd/run.go:L360`) before `gs.OSExit`.

```bash
./k6 run --verbose /tmp/q1_ramping.js > /tmp/q1_hard.out 2>&1 &
K6PID=$!
sleep 6
kill -INT "$K6PID"; kill -TERM "$K6PID"   # two distinct trapped signals, back-to-back
wait "$K6PID"; echo "exit=$?"
```

Observed (salient lines):

```text
running (0m05.0s), 6/6 VUs, 0 complete and 0 interrupted iterations
time="2026-07-14T19:38:03Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-14T19:38:03Z" level=error msg="Aborting k6 in response to signal" sig=terminated
exit=105
```

> **Why `SIGINT`+`SIGTERM` rather than two `SIGINT`s?** Both signals are handled identically by the
> same trap (`cmd/common.go:L101`); `onHardStop` fires on *any* second trapped signal. Two *identical*
> `SIGINT`s are unreliable here: standard (non-realtime) signals are **not queued**, so a second
> `SIGINT` delivered while the first is still pending is coalesced/dropped; and because the first
> signal's `runAbort` shuts the run down almost instantly, the handler goroutine's `done` channel
> (`cmd/common.go:L108`,`L118`) usually closes before a duplicate `SIGINT` can be observed. A *distinct*
> second signal (`SIGTERM`) is not coalesced with the `SIGINT`, so it reliably lands in the size-2
> buffered channel and drives `onHardStop`. This was confirmed empirically: two `SIGINT`s at 8–20 ms
> spacing did **not** produce the `onHardStop` line, whereas `SIGINT`→`SIGTERM` did every time.

**Inferred (not separately measured here):** `gracefulRampDown` (a ramping-vus-only setting) governs
the window for VU-count *reductions between stages*, not manual interrupts — consistent with the
observation that the 30 s `gracefulStop`/`gracefulRampDown` had no effect on the SIGINT path.

---

## Q2 — gRPC server-streaming interrupt

**Direct answer:** When a gRPC **server-streaming** test (`main.FeatureExplorer/ListFeatures`) with a
**30 ms** `gracefulStop`/`gracefulRampDown` is interrupted with `SIGINT`, k6 emits the same interrupt
log family as Q1 — `level=debug msg="Stopping k6 in response to signal..." sig=interrupt` followed by
`level=error msg="test run was aborted because k6 received a 'interrupt' signal"` (exit **105**) — plus
per-stream `stream is cancelled/finished ... error="canceled by client (k6)"` lines. The end-of-test
summary reports **`grpc_streams_msgs_received...........: 98  19.694856/s`** — i.e. **98** streamed
messages were received and counted before the interrupt. That value was **identical (98) across all
three runs**; each of the 2 VUs received the 49 `Feature` messages `ListFeatures` streams back for the
requested rectangle (2 × 49 = 98), all counted via `queueMessage` before the two in-flight iterations
were interrupted (`0 complete and 2 interrupted iterations`).

### Command(s) run

Start the in-repo RouteGuide gRPC server (the `Makefile` `grpc-server-run:` target), listening on
`localhost:10000`:

```bash
go run -mod=mod examples/grpc_server/*.go > /tmp/q2_server.out 2>&1 &
SRVPID=$!
sleep 3
cat /tmp/q2_server.out          # -> "gRPC server starting on localhost:10000"
```

Temp client `/tmp/q2_stream.js` (server-streaming, 30 ms graceful settings). Because the script lives
in `/tmp`, `client.load()` resolves the proto path relative to the script's directory, so the
self-contained proto (`lib/testutils/grpcservice/route_guide.proto`, `package main`, no imports) was
copied next to the script and loaded by a script-relative name:

```javascript
import { Client, Stream } from 'k6/net/grpc';
import { sleep } from 'k6';

const GRPC_ADDR = __ENV.GRPC_ADDR || '127.0.0.1:10000';
const GRPC_PROTO_PATH = __ENV.GRPC_PROTO_PATH; // resolved relative to the script dir (/tmp)

const client = new Client();
client.load([], GRPC_PROTO_PATH);

export const options = {
  scenarios: {
    stream: {
      executor: 'ramping-vus',
      startVUs: 2,
      stages: [{ duration: '60s', target: 2 }],
      gracefulStop: '30ms',
      gracefulRampDown: '30ms',
    },
  },
};

export default () => {
  client.connect(GRPC_ADDR, { plaintext: true });
  const stream = new Stream(client, 'main.FeatureExplorer/ListFeatures', null);
  let received = 0;
  stream.on('data', (feature) => { received++; });
  stream.on('end', () => { client.close(); });
  stream.on('error', (e) => { console.log('Error: ' + JSON.stringify(e)); });
  stream.write({
    lo: { latitude: 400000000, longitude: -750000000 },
    hi: { latitude: 420000000, longitude: -730000000 },
  });
  sleep(0.5);
};
```

```bash
cp lib/testutils/grpcservice/route_guide.proto /tmp/route_guide.proto
GRPC_PROTO_PATH="route_guide.proto" \
  ./k6 run --verbose /tmp/q2_stream.js > /tmp/q2.out 2>&1 &
K6PID=$!
sleep 5                 # let several Features stream in and be counted
kill -INT "$K6PID"
wait "$K6PID"; echo "exit=$?"
```

### Observed output (complete, unedited — run 1)

```text
time="2026-07-14T19:39:46Z" level=debug msg="Logger format: TEXT"
time="2026-07-14T19:39:46Z" level=debug msg="k6 version: v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)"

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

time="2026-07-14T19:39:46Z" level=debug msg="Resolving and reading test '/tmp/q2_stream.js'..."
time="2026-07-14T19:39:46Z" level=debug msg=Loading... moduleSpecifier="file:///tmp/q2_stream.js" originalModuleSpecifier=/tmp/q2_stream.js
time="2026-07-14T19:39:46Z" level=debug msg="'/tmp/q2_stream.js' resolved to 'file:///tmp/q2_stream.js' and successfully loaded 1178 bytes!"
time="2026-07-14T19:39:46Z" level=debug msg="Gathering k6 runtime options..."
time="2026-07-14T19:39:46Z" level=debug msg="Initializing k6 runner for '/tmp/q2_stream.js' (file:///tmp/q2_stream.js)..."
time="2026-07-14T19:39:46Z" level=debug msg="Detecting test type for..." test_path="file:///tmp/q2_stream.js"
time="2026-07-14T19:39:46Z" level=debug msg="Trying to load as a JS test..." test_path="file:///tmp/q2_stream.js"
time="2026-07-14T19:39:46Z" level=debug msg="Runner successfully initialized!"
time="2026-07-14T19:39:46Z" level=debug msg="Parsing CLI flags..."
time="2026-07-14T19:39:46Z" level=debug msg="Consolidating config layers..."
time="2026-07-14T19:39:46Z" level=debug msg="Parsing thresholds and validating config..."
time="2026-07-14T19:39:46Z" level=debug msg="Initializing the execution scheduler..."
time="2026-07-14T19:39:46Z" level=debug msg="Starting 2 outputs..." component=output-manager
time="2026-07-14T19:39:46Z" level=debug msg=Starting... component=metrics-engine-ingester
time="2026-07-14T19:39:46Z" level=debug msg="Started!" component=metrics-engine-ingester
     execution: local
        script: /tmp/q2_stream.js
        output: -

     scenarios: (100.00%) 1 scenario, 2 max VUs, 1m0s max duration (incl. graceful stop):
              * stream: Up to 2 looping VUs for 1m0s over 1 stages (gracefulRampDown: 30ms, gracefulStop: 30ms)

time="2026-07-14T19:39:46Z" level=debug msg="Trapping interrupt signals so k6 can handle them gracefully..."
time="2026-07-14T19:39:46Z" level=debug msg="Starting the REST API server on localhost:6565"
time="2026-07-14T19:39:46Z" level=debug msg="Starting emission of VUs and VUsMax metrics..."
time="2026-07-14T19:39:46Z" level=debug msg="Start of initialization" executorsCount=1 neededVUs=2 phase=execution-scheduler-init
time="2026-07-14T19:39:46Z" level=debug msg="Initialized VU #1" phase=execution-scheduler-init
time="2026-07-14T19:39:46Z" level=debug msg="Initialized VU #2" phase=execution-scheduler-init
time="2026-07-14T19:39:46Z" level=debug msg="Finished initializing needed VUs, start initializing executors..." phase=execution-scheduler-init
time="2026-07-14T19:39:46Z" level=debug msg="Initialized executor stream" phase=execution-scheduler-init
time="2026-07-14T19:39:46Z" level=debug msg="Initialization completed" phase=execution-scheduler-init
time="2026-07-14T19:39:46Z" level=debug msg="Start of test run" executorsCount=1 phase=execution-scheduler-run
time="2026-07-14T19:39:46Z" level=debug msg="setup() is not defined or not exported, skipping!"
time="2026-07-14T19:39:46Z" level=debug msg="Start all executors..." phase=execution-scheduler-run
time="2026-07-14T19:39:46Z" level=debug msg="Starting executor" executor=stream startTime=0s type=ramping-vus
time="2026-07-14T19:39:46Z" level=debug msg="Starting executor run..." duration=1m0s executor=ramping-vus maxVUs=2 numStages=1 scenario=stream startVUs=2 type=ramping-vus
time="2026-07-14T19:39:46Z" level=debug msg=Start executor=ramping-vus scenario=stream vuNum=0
time="2026-07-14T19:39:46Z" level=debug msg=Start executor=ramping-vus scenario=stream vuNum=1

running (0m01.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
stream   [   2% ] 2/2 VUs  0m01.0s/1m00.0s

running (0m02.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
stream   [   3% ] 2/2 VUs  0m02.0s/1m00.0s

running (0m03.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
stream   [   5% ] 2/2 VUs  0m03.0s/1m00.0s

running (0m04.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
stream   [   7% ] 2/2 VUs  0m04.0s/1m00.0s
time="2026-07-14T19:39:51Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-14T19:39:51Z" level=debug msg="Metrics emission of VUs and VUsMax metrics stopped"
time="2026-07-14T19:39:51Z" level=debug msg="stream is cancelled/finished" error="canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-14T19:39:51Z" level=debug msg="stream /main.FeatureExplorer/ListFeatures is closing" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-14T19:39:51Z" level=debug msg="stream is cancelled/finished" error="canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-14T19:39:51Z" level=debug msg="stream /main.FeatureExplorer/ListFeatures is closing" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-14T19:39:51Z" level=debug msg="Executor finished successfully" executor=stream startTime=0s type=ramping-vus
time="2026-07-14T19:39:51Z" level=debug msg="teardown() is not defined or not exported, skipping!"
time="2026-07-14T19:39:51Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
time="2026-07-14T19:39:51Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-14T19:39:51Z" level=debug msg="Stopping vus and vux_max metrics emission..." phase=execution-scheduler-init
time="2026-07-14T19:39:51Z" level=debug msg="Releasing signal trap..."
time="2026-07-14T19:39:51Z" level=debug msg="Sending usage report..."
time="2026-07-14T19:39:51Z" level=debug msg="Waiting for metrics and traces processing to finish..."
time="2026-07-14T19:39:51Z" level=debug msg="Metrics and traces processing finished!"
time="2026-07-14T19:39:51Z" level=debug msg="Stopping outputs..."
time="2026-07-14T19:39:51Z" level=debug msg="Stopping 2 outputs..." component=output-manager
time="2026-07-14T19:39:51Z" level=debug msg=Stopping... component=metrics-engine-ingester
time="2026-07-14T19:39:51Z" level=debug msg="Stopped!" component=metrics-engine-ingester
time="2026-07-14T19:39:51Z" level=debug msg="Generating the end-of-test summary..."

     data_received................: 8.9 kB 1.8 kB/s
     data_sent....................: 3.5 kB 708 B/s
     grpc_streams.................: 2      0.401936/s
     grpc_streams_msgs_received...: 98     19.694856/s
     grpc_streams_msgs_sent.......: 2      0.401936/s
     vus..........................: 2      min=2       max=2
     vus_max......................: 2      min=2       max=2


running (0m05.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
stream ✗ [   8% ] 2/2 VUs  0m05.0s/1m00.0s
time="2026-07-14T19:39:51Z" level=debug msg="Usage report sent successfully"
time="2026-07-14T19:39:51Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-14T19:39:51Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
exit=105
```

**The metric to report (verbatim from the summary):**

```text
grpc_streams_msgs_received...: 98     19.694856/s
```

### Explanation & root cause (file:line)

- The gRPC module registers three stream counters in `js/modules/k6/grpc/metrics.go`: `grpc_streams`
  (`:L17`), `grpc_streams_msgs_sent` (`:L21`), and **`grpc_streams_msgs_received` (`:L25`)** — all
  `metrics.Counter`.
- `queueMessage` (`js/modules/k6/grpc/stream.go:L149`) pushes a `metrics.Sample` for
  `StreamsMessagesReceived` with `Value: 1` for **every** message received from the server (push spans
  `:L151-L159`). The reported **98** is the sum of these `Value:1` samples accumulated before the
  interrupt.
- System under test: `examples/grpc_server/main.go` (the `FeatureExplorer` service). The RPC is
  server-streaming: `rpc ListFeatures(Rectangle) returns (stream Feature)`
  (`lib/testutils/grpcservice/route_guide.proto:L38`; service `FeatureExplorer` at `:L23`). Reference
  client pattern: `examples/grpc_server_streaming.js`.
- The interrupt path is exactly Q1's: `gracefulStop` → `runAbort` (`cmd/run.go:L349-L352`), exit code
  `105` (`errext/exitcodes/codes.go:L41`). The `stream is cancelled/finished ... "canceled by client
  (k6)"` lines show the open server-streams being torn down by the context cancellation.

### Secondary / edge conditions & stability

- **Messages received before the interrupt (non-zero, meaningful):** the summary shows `grpc_streams:
  2` (one stream per VU) and `grpc_streams_msgs_received: 98`; all 98 were counted before the signal.
- **In-flight iterations interrupted:** the final progress line is `0/2 VUs, 0 complete and 2
  interrupted iterations` — mirroring Q1, the 2 in-flight iterations were terminated, not completed.
- **Stability across ≥2 runs:** `grpc_streams_msgs_received` was **98** in all three runs (rates
  `19.694856/s`, `19.701401/s`, `19.690390/s`). The count is stable because `ListFeatures`
  deterministically streams the same 49 features for the fixed rectangle, and both VUs complete their
  single stream well before the 5 s interrupt. Had the interrupt landed mid-stream the count could be
  lower; here it was consistently the full 2 × 49 = 98.


---

## Q3 — Dropped iterations via the REST API

**Direct answer:** The `dropped_iterations` counter value was obtained **by querying the k6 REST API**
`GET http://localhost:6565/v1/metrics` (per the required methodology, Rule 3). For a
`constant-arrival-rate` scenario driving 200 iters/s through only 2 VUs (each `sleep(1)`), a mid-run
API query at ~10 s returned **`"count": 1970`** with `"rate": 197.32…` in the JSON:API payload; the
end-of-test terminal summary independently reported **`dropped_iterations...: 5941  197.077247/s`**.
Both are consistent — `dropped_iterations` is a cumulative counter, so the API captures its value **at
query time** (~197/s × ~10 s ≈ 1970) and the summary captures the **final** value (~197/s × 30 s ≈
5941). The absolute count therefore legitimately varies with *when* you sample; the stable, meaningful
figure is the **drop rate ≈ 197/s**, and the API-vs-summary values agree at every sampling point.

### Command(s) run

**Mechanism A — no free VU (arrival-rate).** Temp script `/tmp/q3_car.js`:

```javascript
import { sleep } from 'k6';
export const options = {
  scenarios: {
    car: {
      executor: 'constant-arrival-rate',
      rate: 200, timeUnit: '1s', duration: '30s',
      preAllocatedVUs: 2, maxVUs: 2,     // only 2 VUs, each iter sleeps 1s => ~2/s served, ~198/s dropped
    },
  },
};
export default function () { sleep(1); }
```

```bash
./k6 run /tmp/q3_car.js > /tmp/q3_car.out 2>&1 &
K6PID=$!
sleep 10
curl -s http://localhost:6565/v1/metrics > /tmp/q3_car_api.json
python3 -c "import json;d=json.load(open('/tmp/q3_car_api.json'));print(json.dumps([x for x in d['data'] if x['id']=='dropped_iterations'],indent=2))"
wait "$K6PID"
grep -E 'dropped_iterations' /tmp/q3_car.out   # terminal-summary cross-check
```

### Observed output — Mechanism A (the authoritative REST-API evidence)

Raw JSON:API object extracted from `GET /v1/metrics` at ~10 s (run 1), **unedited**:

```json
[
  {
    "type": "metrics",
    "id": "dropped_iterations",
    "attributes": {
      "type": "counter",
      "contains": "default",
      "tainted": null,
      "sample": {
        "count": 1970,
        "rate": 197.32295743862812
      }
    }
  }
]
```

The full document is a JSON:API envelope whose `data` array holds one object per observed metric, e.g.
its head:

```json
{"data":[{"type":"metrics","id":"vus_max","attributes":{"type":"gauge","contains":"default","tainted":null,"sample":{"value":2}}},{"type":"metrics","id":"data_sent","attributes":{"type":"counter","contains":"data","tainted":null,"sample":{"count":0,"rate":0}}}, ...
```

Terminal-summary cross-check (end of the same run, at 30 s):

```text
     dropped_iterations...: 5941 197.077247/s
```

### Observed output — Mechanism B (`maxDuration`, shared-iterations)

Temp script `/tmp/q3_si.js`:

```javascript
import { sleep } from 'k6';
export const options = {
  scenarios: {
    si: {
      executor: 'shared-iterations',
      vus: 2, iterations: 1000, maxDuration: '5s',   // ~10 done in 5s, ~990 dropped at maxDuration
    },
  },
};
export default function () { sleep(1); }
```

```bash
./k6 run /tmp/q3_si.js > /tmp/q3_si.out 2>&1 &
K6PID=$!
# poll the API repeatedly while the process is alive:
while kill -0 "$K6PID" 2>/dev/null; do curl -s http://localhost:6565/v1/metrics ...; sleep 0.03; done
grep -E 'dropped_iterations' /tmp/q3_si.out
```

Observed progress + summary:

```text
running (01.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
running (02.0s), 2/2 VUs, 2 complete and 0 interrupted iterations
running (03.0s), 2/2 VUs, 4 complete and 0 interrupted iterations
running (04.0s), 2/2 VUs, 6 complete and 0 interrupted iterations
running (05.0s), 2/2 VUs, 8 complete and 0 interrupted iterations
     dropped_iterations...: 990 197.839523/s
     iterations...........: 10  1.998379/s
running (05.0s), 0/2 VUs, 10 complete and 0 interrupted iterations
```

For this mechanism the live API query returns the metric as **`absent`** (even polling every 0.03 s):
the `DroppedIterations` sample is pushed **once, at executor end** when `maxDuration` is hit, after
which k6 immediately tears down (including the REST API server), so there is effectively no window in
which the value is observable via a live mid-run query. Its value is therefore taken from the terminal
summary: **`dropped_iterations...: 990`**. This is exactly why **Mechanism A (arrival-rate) is used as
the primary Rule-3 REST-API evidence** (its drops are pushed continuously and are live-observable),
and Mechanism B is reported with its summary value as the second, distinct mechanism.

### Explanation & root cause (file:line)

- **The counter:** `metrics/builtin.go` defines `DroppedIterationsName = "dropped_iterations"` (`:L10`),
  the struct field `DroppedIterations *Metric` (`:L44`), and registers it via
  `registry.MustNewMetric(DroppedIterationsName, Counter)` (`:L84`).
- **Two distinct push mechanisms:**
  - *No free VU (arrival-rate, live-observable):* `constant_arrival_rate.go:L324` (and the ramping
    equivalent `ramping_arrival_rate.go:L472`) push a dropped-iteration sample whenever no free VU is
    available to service the scheduled arrival — pushed continuously during the run.
  - *`maxDuration` reached (iterations executors, end-of-run):* `shared_iterations.go:L217-L229` pushes
    a single sample with `Value: float64(totalIters - attemptedIters)` from a deferred function when
    the executor ends; `per_vu_iterations.go:L202` is the per-VU analogue.
- **The REST API path (Rule 3):** the route is registered at `api/v1/routes.go:L23`
  (`mux.HandleFunc("/v1/metrics", …)`, GET); the handler `handleGetMetrics`
  (`api/v1/metric_routes.go:L9`) builds the JSON:API document via
  `newMetricsJSONAPI(cs.MetricsEngine.ObservedMetrics, t)` (`:L16`). The API server binds the default
  address `"localhost:6565"` (`cmd/state/state.go:L150`), which is on by default in this version.

### Secondary / edge conditions & stability

- **Both drop mechanisms covered (Rule 2):** arrival-rate no-free-VU (live API = **1970** @10 s) and
  shared-iterations `maxDuration` (summary = **990**).
- **Run-to-run variance (stated explicitly):** the absolute counter value varies with sampling time
  and scheduling jitter. Observed values:

  | Run | Mechanism A — API @~10 s | Mechanism A — summary @30 s | Mechanism B — summary |
  |-----|--------------------------|-----------------------------|-----------------------|
  | 1   | count=1970, rate=197.32295743862812 | 5941, 197.077247/s | 990, 197.839523/s |
  | 2   | count=1970, rate=197.35988903382736 | 5940, 197.038953/s | 990, 197.827834/s |

  The Mechanism-A API count at ~10 s was **identical (1970)** across both runs and the drop **rate is
  stable at ≈197/s**; the fixed, reproducible element (per Rule 3) is the **methodology** — the value
  is read from the REST API JSON:API payload.


---

## Q4 — SharedArray memory behavior

**Direct answer:** The memory footprint of a data file loaded via **`SharedArray` stays approximately
CONSTANT as the VU count grows** — VUs do **not** each copy the data. Measured peak resident memory
(`VmRSS`) for a ~30 MB / 120,000-row dataset was **~348–363 MB regardless of whether 1, 10, or 50 VUs
were running**. The classic per-VU baseline (`const data = JSON.parse(open(...))` at module-init
scope) instead **scales roughly linearly with VUs** — ~401 MB at 1 VU, ~1,946 MB at 10 VUs, and
~9,262 MB at 50 VUs — because that init code runs once *per VU*, giving each VU its own full parsed
copy. **Root cause:** `SharedArray` stores the data exactly **once** as a host-side `[]string` in the
module's single `sharedArrays` map, hands every VU a **pointer** to that same store, and materializes
only individual elements on demand through a lazy `DynamicArray` proxy — so there is no per-VU
duplication of the dataset.

### Command(s) run

Generate a ~30 MB dataset once (exact size reported below):

```bash
python3 - <<'PY'
import json
rows=[{"id":i,"name":f"user_{i}","email":f"user{i}@example.com","payload":"x"*180} for i in range(120000)]
open('/tmp/data.json','w').write(json.dumps(rows))
PY
ls -l /tmp/data.json
```

`SharedArray` script `/tmp/q4_shared.js` (data loaded **once**, shared across all VUs):

```javascript
import { SharedArray } from 'k6/data';
import { sleep } from 'k6';
const data = new SharedArray('rows', function () { return JSON.parse(open('/tmp/data.json')); });
export const options = { vus: __ENV.VUS ? parseInt(__ENV.VUS) : 50, duration: '30s' };
export default function () {
  const r = data[Math.floor(Math.random() * data.length)]; // touch one element (lazy per-element parse)
  sleep(1);
}
```

Per-VU copy baseline `/tmp/q4_copy.js` (module-init runs **once per VU** → per-VU duplication):

```javascript
import { sleep } from 'k6';
const data = JSON.parse(open('/tmp/data.json'));   // executed per-VU in init context => per-VU copy
export const options = { vus: __ENV.VUS ? parseInt(__ENV.VUS) : 50, duration: '30s' };
export default function () {
  const r = data[Math.floor(Math.random() * data.length)];
  sleep(1);
}
```

Peak resident memory was measured by polling `/proc/<pid>/status` `VmRSS` at 0.1 s intervals (GNU
`time -v` is unavailable in this environment):

```bash
measure () {  # $1=script  $2=VUS
  VUS="$2" ./k6 run "$1" > /tmp/q4.out 2>&1 &
  local pid=$! peak=0 rss
  while kill -0 "$pid" 2>/dev/null; do
    rss=$(awk '/VmRSS/{print $2}' /proc/"$pid"/status 2>/dev/null)
    [ -n "$rss" ] && [ "$rss" -gt "$peak" ] && peak=$rss
    sleep 0.1
  done
  echo "$1 VUS=$2 peakVmRSS=$((peak/1024))MB"
}
for V in 1 10 50; do measure /tmp/q4_shared.js "$V"; measure /tmp/q4_copy.js "$V"; done
# repeated ×2 for stability
```

### Observed output

Dataset size (exact):

```text
-rw-r--r-- 1 root root 31946670 /tmp/data.json      # 31,946,670 bytes; 120,000 rows
```

Peak `VmRSS` (MB), two runs each (`run1 / run2`):

| VUs | `SharedArray` (`q4_shared.js`) | per-VU copy (`q4_copy.js`) |
|----:|-------------------------------:|---------------------------:|
|   1 | 354 / 349 MB                   | 401 / 394 MB               |
|  10 | 348 / 349 MB                   | 1946 / 1900 MB             |
|  50 | 363 / 361 MB                   | 9262 / 9006 MB             |

Raw driver output (`/tmp/q4_results.txt`, unedited):

```text
q4_shared.js           VUS=1   run1 peakVmRSS=354MB (362916 kB)
q4_copy.js             VUS=1   run1 peakVmRSS=401MB (410972 kB)
q4_shared.js           VUS=10  run1 peakVmRSS=348MB (356920 kB)
q4_copy.js             VUS=10  run1 peakVmRSS=1946MB (1993192 kB)
q4_shared.js           VUS=50  run1 peakVmRSS=363MB (372588 kB)
q4_copy.js             VUS=50  run1 peakVmRSS=9262MB (9485112 kB)
q4_shared.js           VUS=1   run2 peakVmRSS=349MB (358076 kB)
q4_copy.js             VUS=1   run2 peakVmRSS=394MB (404472 kB)
q4_shared.js           VUS=10  run2 peakVmRSS=349MB (357856 kB)
q4_copy.js             VUS=10  run2 peakVmRSS=1900MB (1945664 kB)
q4_shared.js           VUS=50  run2 peakVmRSS=361MB (370004 kB)
q4_copy.js             VUS=50  run2 peakVmRSS=9006MB (9222768 kB)
```

The `SharedArray` curve is **flat** (~348–363 MB) while the copy curve grows **linearly** with VUs; at
50 VUs the difference is roughly **8.6 GB** (~362 MB vs ~9.1 GB).

### Explanation & root cause (file:line)

- **One backing store, shared by pointer.** `js/modules/k6/data/data.go`: `RootModule` embeds a single
  `sharedArrays` value (`shared sharedArrays`, `:L22`; the `sharedArrays struct` is a map wrapper at
  `:L31`). `NewModuleInstance` (`:L53`) constructs each VU's `Data` instance with a **pointer** to that
  same map (`shared: &rm.shared`, `:L56`). Thus there is exactly **one** backing store process-wide,
  not one per VU.
- **Data stringified once under lock.** The array is built a single time via `(*sharedArrays).set`
  (`:L143`) / `get` (`:L152`); `getShareArrayFromCall` (`:L169`) JSON-stringifies each element into one
  host-side `[]string`.
- **Lazy per-element proxy — no full-dataset copy per VU.** `js/modules/k6/data/share.go`:
  `type sharedArray struct { arr []string }` (`:L10`); `wrap` (`:L23`) returns a
  `rt.NewDynamicArray(...)` proxy (`:L27`); `wrappedSharedArray.Get(index)` (`:L44`) lazily
  `JSON.parse`s (and deep-freezes) an **individual** element only when it is accessed. Each VU
  therefore materializes only transient per-element values on demand — never a duplicate of the whole
  dataset — which is why the footprint stays ~constant.
- **Contrast (baseline):** the top-level `JSON.parse(open(...))` in `q4_copy.js` runs in **each VU's
  init context** (once per VU), so every VU holds a full parsed copy inside its own Sobek runtime →
  linear growth.

### Secondary / edge conditions & stability

- **Multiple VU counts (1, 10, 50):** demonstrate the flat-vs-linear contrast across the range.
- **Stability (≥2 runs):** `SharedArray` peak stayed within ~348–363 MB across all counts and both
  runs; the copy baseline reproduced ~400 MB / ~1.9 GB / ~9 GB. Absolute MB values are
  machine-dependent (they will differ on other hardware), but the **shape** — constant for
  `SharedArray`, linear for the copy — is the reproducible result.
- **Measurement note:** peak RSS via `/proc/<pid>/status` `VmRSS` because GNU `time -v` is not
  installed (see Methodology).


---

## Q5 — Prometheus (`experimental-prometheus-rw`) metric-name integrity

**Direct answer:** The `experimental-prometheus-rw` output **preserves the integrity of metric names**:
each exported time-series `__name__` is `"k6_" + <original k6 metric name, verbatim> [ + "_" +
<type-suffix> ]`. The original k6 name is never mangled — only a fixed `k6_` prefix and a
metric-type-based suffix are added. Observed suffixes by type: **Counter → `_total`, Gauge → (none),
Rate → `_rate`, Trend → one series per configured statistic** (default stat `p(99)` → `_p99`). This was
proven by pointing k6's remote-write output at a minimal receiver that snappy-decompresses and
protobuf-decodes the `prompb.WriteRequest` and prints every `__name__` label; e.g. `iterations`
(Counter) → **`k6_iterations_total`**, `vus` (Gauge) → **`k6_vus`**, `checks` (Rate) →
**`k6_checks_rate`**, `iteration_duration` (Trend) → **`k6_iteration_duration_p99`**, and
`dropped_iterations` (Counter) → **`k6_dropped_iterations_total`**.

### Command(s) run

A throwaway remote-write receiver was built under `/tmp/rwrecv` (never added to the repo), pinning the
**same** dependency versions k6 uses (from `go.mod`) so it decodes byte-identical payloads:

`/tmp/rwrecv/go.mod`:

```text
module rwrecv

go 1.23

require (
	buf.build/gen/go/prometheus/prometheus/protocolbuffers/go v1.31.0-20230627135113-9a12bc2590d2.1
	github.com/klauspost/compress v1.17.11
	google.golang.org/protobuf v1.35.1
)
```

`/tmp/rwrecv/main.go`:

```go
package main

import (
	"fmt"
	"io"
	"net/http"

	prompb "buf.build/gen/go/prometheus/prometheus/protocolbuffers/go"
	"github.com/klauspost/compress/snappy"
	"google.golang.org/protobuf/proto"
)

func main() {
	http.HandleFunc("/api/v1/write", func(w http.ResponseWriter, r *http.Request) {
		body, _ := io.ReadAll(r.Body)
		raw, err := snappy.Decode(nil, body) // k6 sends snappy-compressed protobuf
		if err != nil { http.Error(w, err.Error(), http.StatusBadRequest); return }
		var wr prompb.WriteRequest
		if err := proto.Unmarshal(raw, &wr); err != nil { http.Error(w, err.Error(), http.StatusBadRequest); return }
		for _, ts := range wr.GetTimeseries() {
			for _, l := range ts.GetLabels() {
				if l.GetName() == "__name__" {
					fmt.Println(l.GetValue())
				}
			}
		}
		w.WriteHeader(http.StatusNoContent)
	})
	fmt.Println("rw-receiver listening on :9090")
	_ = http.ListenAndServe(":9090", nil)
}
```

Metric-producing script `/tmp/q5.js` (covers Counter, Gauge, Rate and Trend metric types):

```javascript
import { sleep, check } from 'k6';
export const options = { vus: 2, duration: '10s' };
export default function () {
  check(1, { 'always true': (v) => v === 1 });
  sleep(0.5);
}
```

```bash
cd /tmp/rwrecv && go build -o /tmp/rwrecv/rwrecv . && /tmp/rwrecv/rwrecv > /tmp/q5_names.out 2>&1 &
sleep 2
cd "$REPO"
# default remote-write URL is http://localhost:9090/api/v1/write (config.go:L21) — matches the receiver
./k6 run -o experimental-prometheus-rw /tmp/q5.js
sort -u /tmp/q5_names.out    # the unique set of exported __name__ values
```

### Observed output

Unique exported `__name__` values from `/tmp/q5.js` (**identical across two runs**):

```text
k6_checks_rate
k6_data_received_total
k6_data_sent_total
k6_iteration_duration_p99
k6_iterations_total
k6_vus
k6_vus_max
```

One exemplar per metric type (real k6 metrics, labelled by their registered type):

| Original k6 metric | Metric type | Exported `__name__` | Suffix rule |
|--------------------|-------------|---------------------|-------------|
| `iterations`         | Counter | `k6_iterations_total`         | Counter → `_total` |
| `data_received`      | Counter | `k6_data_received_total`      | Counter → `_total` |
| `vus`                | Gauge   | `k6_vus`                      | Gauge → (no suffix) |
| `vus_max`            | Gauge   | `k6_vus_max`                  | Gauge → (no suffix) |
| `checks`             | Rate    | `k6_checks_rate`              | Rate → `_rate` |
| `iteration_duration` | Trend   | `k6_iteration_duration_p99`   | Trend → per-stat (default `p(99)`) |

Running the Q3 arrival-rate script through the same output additionally exports the named Counter
example **`k6_dropped_iterations_total`**:

```text
k6_data_received_total
k6_data_sent_total
k6_dropped_iterations_total
k6_iteration_duration_p99
k6_iterations_total
k6_vus
k6_vus_max
```

**Trend → one series per statistic.** The default only emits `p(99)`; setting
`K6_PROMETHEUS_RW_TREND_STATS="p(99),p(95),max,min,avg,med"` shows the per-statistic naming, with the
original `iteration_duration` name preserved verbatim and the statistic as the suffix (parentheses
stripped, `p(99)`→`p99`):

```text
k6_iteration_duration_avg
k6_iteration_duration_max
k6_iteration_duration_med
k6_iteration_duration_min
k6_iteration_duration_p95
k6_iteration_duration_p99
```

### Explanation & root cause (file:line)

- **CLI output id:** the generated `_builtinOutputName` string contains `experimental-prometheus-rw`
  (`cmd/builtin_output_gen.go:L10`); the output is constructed via `remotewrite.New(params)`
  (`cmd/outputs.go:L67`).
- **Prefix:** `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/config.go:L24` —
  `defaultMetricPrefix = "k6_"` (default server URL `http://localhost:9090/api/v1/write` at `:L21`;
  default push interval 5 s at `:L23`; default trend stats `[]string{"p(99)"}` at `:L28`).
- **Name assembly / integrity:** `.../remotewrite/prometheus.go` — `const namelbl = "__name__"` (`:L11`);
  `MapSeries` (`:L39`) computes `v := defaultMetricPrefix + series.Metric.Name` (`:L40`), appends
  `"_" + suffix` when the suffix is non-empty, writes the result into the `__name__` label (`:L45`), and
  sorts the label set lexicographically (`:L48`). The original `series.Metric.Name` is inserted
  **verbatim** between prefix and suffix — this is the integrity property.
- **Suffix by type:** `.../remotewrite/remotewrite.go` — `MapPrompb` (`:L316`) / `mapMonoSeries`
  (`:L319`) switch on `swm.Metric.Type` (`:L329`): Counter → `"total"` (`:L331`), Gauge → `""` (`:L336`),
  Rate → `"rate"` (`:L341`), Trend → one series per statistic (`:L347`).
- **Wire format (why the receiver is faithful):** `.../remote/client.go` `Store` (`:L78`) →
  `newWriteRequestBody` (`:L122`) does `proto.Marshal(&prompb.WriteRequest{...})` (`:L123`) then
  `snappy.Encode(...)` (`:L133`), POSTing with headers `Content-Encoding: snappy` (`:L99`) and
  `Content-Type: application/x-protobuf` (`:L100`). The receiver reverses exactly this: `snappy.Decode`
  → `proto.Unmarshal` into `prompb.WriteRequest` → read `__name__`.

### Secondary / edge conditions & stability

- **All four metric types covered:** Counter (`_total`), Gauge (none), Rate (`_rate`), Trend (per-stat).
- **Name integrity across the transformation:** for every metric the substring between `k6_` and the
  type suffix is exactly the original k6 metric name (`iterations`, `data_received`, `vus`, `checks`,
  `iteration_duration`, `dropped_iterations`, …) — never altered.
- **Stability (≥2 runs):** the unique set of exported `__name__` values was **identical** across both
  default runs. (Decode method used: the **faithful protobuf decode** via the pinned
  `prompb.WriteRequest` type, not a textual fallback.)


---

## Methodology & Reproducibility

**Environment.** All runs were performed inside the provided Docker image
`andrewparkscaleai/coding-agent:grafana__k6__ddc3b0b1d23c128e34e2792fc9075f9126e32375`
(from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_grafana_k6_1.0`), on `linux/amd64`.

**Build (done once, canonical, default configuration).**

```bash
go build -o ./k6 .          # equivalent to the Makefile `build:` target
./k6 version                # -> k6 v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)
go version                  # -> go version go1.23.12 linux/amd64
```

- Observed `./k6 version` banner: `k6 v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)`
  (version constant `const Version = "0.55.0"`, `lib/consts/consts.go:L12`).
- Observed Go toolchain: `go version go1.23.12 linux/amd64`. (`go.mod` declares `go 1.21` /
  `toolchain go1.21.13` at `go.mod:L3,L5`, but the CI/environment builds with Go 1.23.x.)
- Every test was driven through the real CLI entry point `main.go:L8-L9` → `cmd.Execute()`; no debug
  hooks, mocks, or synthetic bypasses were used.

**Per-question invocation summary.**

| Q | Script(s) | Invocation (essence) | Value(s) read from |
|---|-----------|----------------------|--------------------|
| Q1 | `/tmp/q1_ramping.js` | `./k6 run --verbose … &` then `kill -INT` (and `kill -TERM` for hard stop) | console/log stream + progress line |
| Q2 | `/tmp/q2_stream.js` + `examples/grpc_server` | `go run -mod=mod examples/grpc_server/*.go &`; `GRPC_PROTO_PATH=route_guide.proto ./k6 run --verbose … &` then `kill -INT` | interrupt log + end-of-test summary |
| Q3 | `/tmp/q3_car.js`, `/tmp/q3_si.js` | `./k6 run … &` then `curl -s http://localhost:6565/v1/metrics` | **REST API JSON:API** (primary) + summary cross-check |
| Q4 | `/tmp/q4_shared.js`, `/tmp/q4_copy.js` | `VUS=N ./k6 run …` while polling `/proc/<pid>/status` `VmRSS` | peak `VmRSS` |
| Q5 | `/tmp/q5.js` (+ `/tmp/rwrecv` receiver) | `./k6 run -o experimental-prometheus-rw …` | decoded `prompb.WriteRequest` `__name__` labels |

**Run-to-run stability (Rule 1).** Each magnitude value was confirmed across at least two runs:

- **Q1 interrupted count:** `6` interrupted iterations (== 6 active VUs) and exit code `105` in both
  runs — stable.
- **Q2 received count:** `grpc_streams_msgs_received = 98` in all three runs (rates `19.694856`,
  `19.701401`, `19.690390`/s) — stable.
- **Q3 dropped count:** *legitimately varies with sampling time* (cumulative counter). Mechanism A REST
  API @~10 s = `1970` in both runs (rate ≈197/s); summary @30 s = `5941` / `5940`. Mechanism B summary
  = `990` in both runs. The fixed element is the methodology (value read from the REST API).
- **Q4 memory peaks:** `SharedArray` `~348–363 MB` and per-VU copy `~400 MB / ~1.9 GB / ~9 GB`
  (1/10/50 VUs) across both runs — stable in shape; absolute MB are machine-dependent.
- **Q5 names:** the exported `__name__` set was identical across both runs — stable.

**Q4 memory-measurement note.** GNU `time -v` is not installed in this environment, so peak resident
memory was measured by polling `/proc/<pid>/status` `VmRSS` at 0.1 s intervals and taking the maximum.

**Repository left unchanged (MainRule).** The only file added anywhere in the repository is this
document. All observation scripts/datasets lived under `/tmp` and were deleted; the compiled `./k6`
binary (gitignored via `/k6` on line 1 of `.gitignore`) was removed; and the `examples/grpc_server`
`go.mod`/`go.sum` (which a `-mod=mod` build can rewrite) were verified byte-for-byte pristine. Final
verification:

```text
$ git status --porcelain -uall
?? blitzy/documentation/k6_ddc3b0b1d23c.md
```

(The single untracked entry above is this deliverable; no existing file was modified, added, or
deleted. `-uall` is used so git lists the individual file rather than collapsing the newly created
`blitzy/` directory to `?? blitzy/`; the two empty sibling directories `blitzy/screen_recordings`
and `blitzy/screenshots` contain no files and are therefore not tracked by git.)

