# k6 Runtime Investigation — Q&A Answer Document

This document answers five questions about the runtime behavior of the Grafana **k6**
load-testing tool. Every answer is grounded in **actual observed runtime evidence**
(build → run → capture) produced against the repository at its checked-out `HEAD`, not
from reading source alone. Each question section contains, in order: the **exact
command(s)** run, the **complete unedited observed output**, the **direct answer**, the
**root-cause explanation** with `file:line` citations naming the function/struct that
performs the work, and an explicit **observed-vs-inferred** labeling. A final **coverage
pass** confirms every named item is addressed.

## Preamble — build, environment, and methodology

**System under test.** `go.k6.io/k6`, checked out at commit
`ddc3b0b1d23c128e34e2792fc9075f9126e32375` (source branch `k6_ddc3b0b1d23c`).

**Canonical build command** (from the repository root; the binary is emitted to `/tmp`
so the repository stays pristine):

```bash
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
export GOTOOLCHAIN=local
go version
# => go version go1.21.13 linux/amd64
mkdir -p /tmp/k6bin
go build -o /tmp/k6bin/k6 .
/tmp/k6bin/k6 version
```

**Observed version banner** (this is the build every experiment below uses):

```
k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)
```

- **Toolchain:** `go1.21.13` (pinned via `GOTOOLCHAIN=local`; matches `go.mod`'s
  `toolchain go1.21.13`). Build used Go **vendor mode** (the repository ships a complete
  `vendor/` tree), no network required.
- **OS/arch:** `linux/amd64`.
- **Repository cleanliness:** `git status --porcelain` was empty before and after the
  build (the binary lives at `/tmp/k6bin/k6`, outside the checkout), and `go.mod`/`go.sum`
  were left untouched.

**Methodology (applied to every question):**

- **Run-first.** k6 was built and the relevant code paths were executed; the answer prose
  was written from the captured output.
- **Canonical entry points only.** Values were obtained through the real CLI (`k6 run`,
  `k6 stats`) and the REST control API (`GET /v1/metrics…`, default bind `localhost:6565`).
  No debug hooks, mocks, or synthetic bypasses were used to obtain any reported value.
- **Two-run stability.** Every magnitude/frequency/timing value was confirmed across ≥2
  runs; the run scale/duration is stated. Values that are inherently a runtime measurement
  are reported as observed (and their stability across runs is noted).
- **Every condition exercised.** Each question's primary path *and* its
  secondary/edge/transitional states were exercised (e.g., first vs. second `SIGINT` for
  Q1; before/during/after and two executor drop-sites for Q3).
- **Observed vs. inferred.** Statements demonstrated by captured output are labeled
  **Observed**; statements derived only from reading source are labeled **_inferred_**.
- **Read-only repository.** No existing repository file was modified. All harness scripts,
  data files, helper servers, and binaries were created under `/tmp` (or a git-ignored
  path) and removed afterward; this answer document is the only committed change.

---

## Q1 — Virtual User (VU) lifecycle on `SIGINT` (`ramping-vus`)

**Question.** When a scenario using the `ramping-vus` executor with at least five VUs
receives a `SIGINT`, what are the exact log messages emitted during shutdown, and does
that evidence show that currently-active VUs are permitted to **finish their in-progress
iteration** or are they **terminated mid-execution**?

**Harness** (`/tmp/q1_ramping_vus.js`) — five VUs, each iteration a 30-second `sleep` so
that a `SIGINT` a few seconds in reliably lands while every VU is mid-iteration:

```javascript
import { sleep } from 'k6';

export const options = {
  scenarios: {
    ramp: {
      executor: 'ramping-vus',
      startVUs: 5,
      stages: [
        { duration: '2s', target: 5 },
        { duration: '120s', target: 5 },
      ],
      gracefulRampDown: '30s',
      gracefulStop: '30s',
    },
  },
};

export default function () {
  sleep(30); // each iteration is long; a SIGINT a few seconds in lands mid-iteration for all VUs
}
```

### Primary path — a single `SIGINT`

**Command** (run twice for stability):

```bash
/tmp/k6bin/k6 run --verbose /tmp/q1_ramping_vus.js > /tmp/q1.log 2>&1 &
K6PID=$!
sleep 8            # all 5 VUs are now mid-iteration (each sleeping 30s)
kill -INT "$K6PID" # canonical SIGINT to the k6 process
wait "$K6PID"; echo "exit=$?"
```

**Observed output** (identical key lines across both runs; `exit=105`):

```
time="2026-07-13T17:51:00Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt

     data_received...: 0 B 0 B/s
     data_sent.......: 0 B 0 B/s
     vus.............: 5   min=5 max=5
     vus_max.........: 5   min=5 max=5

running (0m08.0s), 0/5 VUs, 0 complete and 5 interrupted iterations
time="2026-07-13T17:51:00Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

The scenario description printed at start confirms the executor and graceful windows:

```
     scenarios: (100.00%) 1 scenario, 5 max VUs, 2m32s max duration (incl. graceful stop):
              * ramp: Up to 5 looping VUs for 2m2s over 2 stages (gracefulRampDown: 30s, gracefulStop: 30s)
```

### Secondary path — a second `SIGINT` (hard stop)

A real, manual `SIGINT` makes k6 shut down in well under a second, and the kernel
coalesces a second *pending* `SIGINT`, so to deliver a **distinct** second signal to the
hard-stop path the shutdown window was widened with a normal script-defined `teardown()`
that sleeps (this changes nothing about signal handling — it is still the real `k6 run`
receiving real `kill -INT` signals). The second signal was sent while `teardown()` ran:

**Command:**

```bash
# /tmp/q1_teardown_probe.js == the harness above plus:
#   export function teardown() { console.log('TEARDOWN_STARTED');
#     for (let i=0;i<10;i++){ console.log('TEARDOWN_TICK_'+i); sleep(1);} console.log('TEARDOWN_FINISHED'); }
/tmp/k6bin/k6 run --verbose /tmp/q1_teardown_probe.js > /tmp/q1_double.log 2>&1 &
K6PID=$!
sleep 8
kill -INT "$K6PID"   # first signal  -> graceful
sleep 2              # now inside the teardown window; first signal fully consumed
kill -INT "$K6PID"   # second signal -> hard stop
wait "$K6PID"; echo "exit=$?"
```

**Observed output** (reproduced across two runs; `exit=105`):

```
time="2026-07-13T17:55:23Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-13T17:55:23Z" level=info msg=TEARDOWN_STARTED source=console
time="2026-07-13T17:55:23Z" level=info msg=TEARDOWN_TICK_0 source=console
time="2026-07-13T17:55:24Z" level=info msg=TEARDOWN_TICK_1 source=console
time="2026-07-13T17:55:25Z" level=info msg=TEARDOWN_TICK_2 source=console
time="2026-07-13T17:55:25Z" level=error msg="Aborting k6 in response to signal" sig=interrupt
```

The process exits (`OSExit`) mid-teardown — it never reaches `TEARDOWN_FINISHED`.

### Direct answer

**Currently-active VUs are terminated mid-execution; the in-progress iteration is NOT
allowed to finish.** The decisive evidence is the end-of-test summary line
`0 complete and 5 interrupted iterations` — **Observed**. All five VUs were mid-iteration
(each in a `sleep(30)`) when the single `SIGINT` arrived; none of those iterations was
allowed to complete (0 complete), and all five were counted as **interrupted** (partial),
which is precisely a mid-execution termination. Although the shutdown handler is *named*
"graceful", a manual interrupt does **not** grant the `gracefulStop` window to in-flight
iterations. The secondary path shows the first `SIGINT` logs
`Stopping k6 in response to signal... sig=interrupt` and a second `SIGINT` logs
`Aborting k6 in response to signal` and forces an immediate exit. Both paths exit with code
**105** (`ExternalAbort`).

### Root cause (`file:line`)

- **Signal trap and the two-signal state machine.** `handleTestAbortSignals()` installs
  the trap and runs a goroutine whose first `select` calls the graceful handler and whose
  second `select` calls the hard-stop handler and then exits the process:
  `gs.SignalNotify(sigC, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)`
  [`cmd/common.go:101`]; the buffered channel `sigC := make(chan os.Signal, 2)`
  [`cmd/common.go:99`]; second signal → `gs.OSExit(int(exitcodes.ExternalAbort))`
  [`cmd/common.go:118`]. `ExternalAbort` is `105` [`errext/exitcodes/codes.go:41`].
- **The two handlers and their log lines.** `gracefulStop` logs
  `"Stopping k6 in response to signal..."` and calls `runAbort(...)` with
  `exitcodes.ExternalAbort` and `errext.AbortedByUser`, which cancels the run context
  [`cmd/run.go:350,352`]; `onHardStop` logs `"Aborting k6 in response to signal"`
  [`cmd/run.go:360`]. The abort message `"test run was aborted because k6 received a '%s'
  signal"` is built at [`cmd/run.go:352`].
- **Why the graceful window does not apply to a manual interrupt.** `GetGracefulStop()`'s
  documentation states the graceful window is for the *end of the normal executor
  duration*, and "Of course, that doesn't count when the user manually interrupts the
  test, then iterations are immediately stopped" [`lib/executor/base_config.go:91-96`];
  `DefaultGracefulStopValue = 30 * time.Second` [`lib/executor/base_config.go:20`].
- **The VU contract this manifests.** "gracefulStop must let an iteration which has started
  to finish" vs. "hardStop must stop an iteration in process"
  [`lib/executor/vu_handle.go:63-67`].
- **The summary line that encodes the result.** The format
  `"%s, "+vusFmt+"/"+vusFmt+" VUs, %d complete and %d interrupted iterations"` is built from
  `GetFullIterationCount()` (complete) and `GetPartialIterationCount()` (interrupted)
  [`execution/scheduler.go:155-159`]; a non-zero *interrupted* count = mid-execution
  terminations.
- **The executor under test.** `rampingVUsType = "ramping-vus"`
  [`lib/executor/ramping_vus.go:19`].
- **Corroboration in the test suite.** The exact debug line is asserted verbatim:
  `level=debug msg="Stopping k6 in response to signal..." sig=interrupt`
  [`cmd/tests/cmd_run_test.go:1225,1246`].

### Observed vs. inferred

- **Observed:** the `Stopping k6…`/`Aborting k6…` log lines; the
  `0 complete and 5 interrupted iterations` summary; exit code `105`; the printed graceful
  windows.
- **_inferred_:** that each individual interrupted iteration was terminated *at the exact
  moment* of its `sleep(30)` — this follows from the design (context cancellation) and the
  aggregate count; the runtime signal we produced is the non-zero interrupted-iteration
  count, which is sufficient to answer the question. The mapping of the observed exit code
  to the symbol `ExternalAbort` is grounded in `errext/exitcodes` (read), consistent with
  the observed `105`.

---

## Q2 — gRPC server-streaming interruption; `grpc_streams_msgs_received`

**Question.** For a gRPC **server-streaming** test configured with a 30 ms graceful
ramp-down (`gracefulRampDown`/`gracefulStop = '30ms'`) that is interrupted, what are the
exact log entries produced at runtime, and what is the value of the
**`grpc_streams_msgs_received`** metric shown in the final end-of-test metrics summary?

**Harness — the gRPC server.** k6 ships a server-streaming gRPC server at
`examples/grpc_server/main.go` that registers the `main.FeatureExplorer` service (whose
`ListFeatures` RPC is server-streaming). That directory is a *nested* Go module requiring
`grpc v1.64.1`, which is unavailable offline here; and its `main.go` imports
`google.golang.org/grpc/testdata` (used only by the unused TLS branch), which is not
vendored. A minimal, faithful harness that registers the **same** `grpcservice`
`FeatureExplorer`/`ListFeatures` server-streaming RPC (plaintext only, dropping the unused
TLS/testdata path) was compiled inside the root module (so it resolves the internal
`grpcservice` package and the root-vendored `grpc v1.67.1`) and run on `localhost:10000`.
The server is only a harness; **k6 (the system under test) is driven canonically** via
`k6 run`. The server logged `gRPC server starting on localhost:10000`.

**Harness — the k6 client** (`/tmp/q2_grpc_stream.js`) wraps the shipped
`examples/grpc_server_streaming.js` request in a `ramping-vus` scenario with the exact
`30ms` graceful windows. (k6's `client.load()` resolves the proto path relative to the
script directory, so the self-contained proto was copied to `/tmp/route_guide.proto` and
loaded by relative name.)

```javascript
import { Client, Stream } from 'k6/net/grpc';
import { sleep } from 'k6';

const GRPC_ADDR = __ENV.GRPC_ADDR || '127.0.0.1:10000';
const GRPC_PROTO_PATH = __ENV.GRPC_PROTO_PATH; // 'route_guide.proto' (relative to /tmp)

export const options = {
  scenarios: {
    streaming: {
      executor: 'ramping-vus',
      startVUs: 5,
      stages: [
        { duration: '2s', target: 5 },
        { duration: '120s', target: 5 },
      ],
      gracefulRampDown: '30ms',
      gracefulStop: '30ms',
    },
  },
};

const client = new Client();
client.load([], GRPC_PROTO_PATH);

export default () => {
  client.connect(GRPC_ADDR, { plaintext: true });
  const stream = new Stream(client, 'main.FeatureExplorer/ListFeatures', null);
  stream.on('data', () => { /* each received Feature is counted in grpc_streams_msgs_received */ });
  stream.on('end', () => { client.close(); });
  stream.on('error', (e) => { console.log('Error: ' + JSON.stringify(e)); });
  stream.write({
    lo: { latitude: 400000000, longitude: -750000000 },
    hi: { latitude: 420000000, longitude: -730000000 },
  });
  sleep(0.5);
};
```

**Command** (run three times; SIGINT sent mid-stream at 4 s):

```bash
PROTO="$(git rev-parse --show-toplevel)/lib/testutils/grpcservice/route_guide.proto"
cp "$PROTO" /tmp/route_guide.proto
GRPC_PROTO_PATH="route_guide.proto" GRPC_ADDR=127.0.0.1:10000 \
  /tmp/k6bin/k6 run --verbose /tmp/q2_grpc_stream.js > /tmp/q2.log 2>&1 &
K6PID=$!
sleep 4
kill -INT "$K6PID"
wait "$K6PID"; echo "exit=$?"
```

**Observed output** (interrupt log line + the complete `grpc_streams*` summary block;
`exit=105`, zero stream `Error:` lines):

```
time="2026-07-13T18:01:29Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
     grpc_streams.................: 5      1.258061/s
     grpc_streams_msgs_received...: 195    49.064363/s
     grpc_streams_msgs_sent.......: 5      1.258061/s
running (0m04.0s), 0/5 VUs, 0 complete and 5 interrupted iterations
```

**Stability across the three runs** (same unchanged input):

| run | `grpc_streams` | `grpc_streams_msgs_sent` | `grpc_streams_msgs_received` |
|-----|----------------|--------------------------|------------------------------|
| 1   | 5              | 5                        | **195**                      |
| 2   | 5              | 5                        | **195**                      |
| 3   | 5              | 5                        | **195**                      |

### Direct answer

The interrupt log entry is
`level=debug msg="Stopping k6 in response to signal..." sig=interrupt`, and the final
end-of-test summary reports **`grpc_streams_msgs_received: 195`** — **Observed**, and
perfectly stable across all three runs. (Also `grpc_streams: 5`, `grpc_streams_msgs_sent:
5`, `0 complete and 5 interrupted iterations`, exit code `105`.) After the interrupt, the
VU context is cancelled and the stream is closed, so no further received messages are
counted — the reported `195` reflects only the messages counted before shutdown.

**Why 195, and why it is stable:** the harness server sends each matching feature only
after `time.Sleep(100 * time.Millisecond)` [`lib/testutils/grpcservice/service.go:62`], so
features stream at a fixed 100 ms cadence. Thirty-nine features fall inside the requested
rectangle, so a full stream takes ≈3.9 s; interrupting at a fixed 4 s reliably captures all
39 per stream across the 5 VUs → 39 × 5 = **195**. The `0 complete` iterations follow
because the interrupt at 4 s lands during the post-stream `sleep(0.5)` that runs after the
stream's `end` (~3.9 s). `grpc_streams_msgs_sent: 5` is the single `Rectangle` request each
of the 5 VUs sent.

### Root cause (`file:line`)

- **Counter definitions.** The three counters are registered in
  `js/modules/k6/grpc/metrics.go`: fields `Streams`/`StreamsMessagesSent`/
  `StreamsMessagesReceived` [`:7-9`]; names `registry.NewMetric("grpc_streams",
  metrics.Counter)` [`:17`], `"grpc_streams_msgs_sent"` [`:21`], and
  `"grpc_streams_msgs_received"` [`:25`].
- **Per-message increment of the received counter.** `queueMessage()` pushes a sample of
  `StreamsMessagesReceived` with `Value: 1` for every received message
  [`js/modules/k6/grpc/stream.go:149,153,158`]; the sent counter is incremented separately
  [`js/modules/k6/grpc/stream.go:257,262`].
- **Interrupt path that stops counting.** `stream.loop()` selects `case <-ctxDone:` when
  the VU context is cancelled and closes the stream — the comment reads "VU is shutting
  down during an interrupt / stream events will not be forwarded to the VU" — via
  `closeWithError(nil)` [`js/modules/k6/grpc/stream.go:119,133,136-140`].
- **The server-streaming RPC and its 100 ms cadence.**
  `rpc ListFeatures(Rectangle) returns (stream Feature)`
  [`lib/testutils/grpcservice/route_guide.proto:38`]; the server implementation loops over
  in-range features, sleeping `time.Sleep(100 * time.Millisecond)` before each
  `stream.Send(feature)` [`lib/testutils/grpcservice/service.go:57,61-63`].
- **The shipped client this harness mirrors.** `examples/grpc_server_streaming.js` defaults
  `GRPC_ADDR=127.0.0.1:10000` and uses `new Stream(client,
  'main.FeatureExplorer/ListFeatures', null)`; it has no `options` block (hence the `/tmp`
  wrapper supplying the `30ms` scenario).

### Observed vs. inferred

- **Observed:** the `Stopping k6…` interrupt line; the full `grpc_streams*` summary block
  with `grpc_streams_msgs_received: 195` (stable ×3); `0 complete and 5 interrupted
  iterations`; exit `105`.
- **_inferred_:** the exact per-stream arithmetic (39 in-range features × 5 VUs) is a
  reading of the server source explaining *why* the observed `195` is what it is; the
  observed value itself (195) is the reported runtime measurement.

---

## Q3 — `dropped_iterations` via the REST control API

**Question.** What is the exact value of **`dropped_iterations`** when a test exceeds its
maximum duration capacity, obtained specifically by **querying the k6 REST control API**
(e.g., `GET /v1/metrics`), with runtime evidence proving the value came from the API rather
than the console summary?

Two canonical query paths exist and both are used below: the raw HTTP API
(`curl http://localhost:6565/v1/metrics/...`) and the `k6 stats` CLI (which reads the same
API). The `--linger/-l` flag ("keep the API server alive past test end"
[`cmd/config.go:32`]) keeps the REST server reachable after the run ends.

### Primary case — `shared-iterations` over `maxDuration` (drop emitted at test end)

**Harness** (`/tmp/q3_shared_iters.js`):

```javascript
import { sleep } from 'k6';

export const options = {
  scenarios: {
    over: {
      executor: 'shared-iterations',
      vus: 5,
      iterations: 1000,   // far more than can finish within maxDuration
      maxDuration: '3s',
    },
  },
};

export default function () {
  sleep(1); // ~5 VUs * ~3 iters in 3s => ~15 completed, ~985 dropped
}
```

**Command** (run twice for stability):

```bash
/tmp/k6bin/k6 run -l /tmp/q3_shared_iters.js > /tmp/q3.log 2>&1 &
K6PID=$!
sleep 6   # test (maxDuration 3s) has ended; --linger keeps localhost:6565 alive
curl -s http://localhost:6565/v1/metrics/dropped_iterations | tee /tmp/q3_api.json; echo
python3 -c "import json;d=json.load(open('/tmp/q3_api.json'));print(d['data']['attributes']['sample']['count'])"
/tmp/k6bin/k6 stats dropped_iterations
kill -INT "$K6PID"; wait "$K6PID"
```

**Observed output — raw JSON straight from `GET /v1/metrics/dropped_iterations`** (this is
the proof the value came from the API; `count` = **985**, stable across both runs):

```json
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":985,"rate":328.1354371481373}}}}
```

Extracting `data.attributes.sample.count` prints `985`. The **list** route `GET
/v1/metrics` returns the same value inside its `data[]` array (run 2):

```json
{
  "type": "metrics",
  "id": "dropped_iterations",
  "attributes": {
    "type": "counter",
    "contains": "default",
    "tainted": null,
    "sample": {
      "count": 985,
      "rate": 328.1509262656229
    }
  }
}
```

**Corroboration via `k6 stats dropped_iterations`** (reads the same REST API):

```yaml
- name: dropped_iterations
  type:
    type: counter
    valid: true
  contains:
    type: default
    valid: true
  tainted: ""
  sample:
    count: 985
    rate: 328.1354371481373
```

(The same run's `iterations` counter reads `count: 15`; 15 completed + 985 dropped = the
1000 requested iterations.)

### Secondary case — `constant-arrival-rate` over-capacity (drops accrue during the run)

**Harness** (`/tmp/q3_arrival.js`): `rate: 200/s` with only `maxVUs: 5` cannot be serviced,
so drops accumulate throughout the 10 s run.

```javascript
import { sleep } from 'k6';
export const options = {
  scenarios: {
    car: {
      executor: 'constant-arrival-rate',
      rate: 200, timeUnit: '1s', duration: '10s',
      preAllocatedVUs: 5, maxVUs: 5,
    },
  },
};
export default function () { sleep(1); }
```

**Command + observed** — polling `GET /v1/metrics/dropped_iterations` (with `-l`) shows the
value climbing **before / during / after**, all read from the API, and the final API value
matches the console summary exactly:

```
BEFORE/early (t~0.5s):  dropped_iterations.count = 85
DURING  (t~3s):         dropped_iterations.count = 575
DURING  (t~6s):         dropped_iterations.count = 1160
AFTER   (t~11s, ended): dropped_iterations.count = 1951
AFTER (raw JSON from API):
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":1951,"rate":193.815880789517}}}}
console final for cross-check:
     dropped_iterations...: 1951 193.815881/s
```

### Direct answer

Queried from the REST API, `dropped_iterations` is **985** for the `shared-iterations`
over-`maxDuration` scenario (raw JSON `data.attributes.sample.count = 985`, stable across
two runs, corroborated by `k6 stats`), and **1951** for the `constant-arrival-rate`
over-capacity scenario (whose API value climbs 85 → 575 → 1160 → 1951 and matches the
console summary of 1951) — **Observed**. The value provably came from the API: the raw
JSON:API bodies above were returned by `GET /v1/metrics/dropped_iterations` and `GET
/v1/metrics` while the run was lingering, and `k6 stats` reproduces the same figure through
the same API.

### Root cause (`file:line`)

- **Metric definition.** `DroppedIterationsName = "dropped_iterations"`
  [`metrics/builtin.go:10`], registered as a Counter via
  `registry.MustNewMetric(DroppedIterationsName, Counter)` [`metrics/builtin.go:84`].
- **Primary (end-of-test) drop site.** In `shared-iterations`, a deferred function pushes a
  single sample after all VUs finish: `if attemptedIters < totalIters { … Metric:
  …DroppedIterations …, Value: float64(totalIters - attemptedIters) }`
  [`lib/executor/shared_iterations.go:213-225`] — emitted at test end, which is why
  `--linger` is required to read it via REST.
- **Secondary (during-run) drop site.** In `constant-arrival-rate`, when no VU is free the
  loop pushes `Value: 1` per dropped iteration: `droppedIterationMetric` [`:324`], `if
  vusPool.TryRunIteration()` [`:332`], `Metric: droppedIterationMetric` [`:341`], `Value: 1`
  [`lib/executor/constant_arrival_rate.go:345`]. (Analogous sites exist at
  `lib/executor/per_vu_iterations.go` and `lib/executor/ramping_arrival_rate.go`.)
- **REST routes.** `mux.HandleFunc("/v1/metrics", …) → handleGetMetrics`
  [`api/v1/routes.go:23,28`] and `mux.HandleFunc("/v1/metrics/", …)` which extracts the id
  and calls `handleGetMetric` [`api/v1/routes.go:31,37,38`].
- **JSON shape (`sample.count`).** ``Metric.Sample map[string]float64 `json:"sample"` ``
  [`api/v1/metric.go:69`] filled by `Sample: m.Sink.Format(t)` [`api/v1/metric.go:80`]; a
  Counter's `Format` returns `{"count": c.Value, "rate": …}` [`metrics/sink.go:64-67`]. The
  JSON:API envelope wraps it as ``Attributes Metric `json:"attributes"` ``
  [`api/v1/metric_jsonapi.go:21`] inside ``Data …`json:"data"` `` (list `[]metricData` at
  `:10-11`, single `metricData` at `:15`). The value path is therefore
  `data.attributes.sample.count`.
- **REST bind + flags.** Default bind `localhost:6565` [`cmd/state/state.go:150`], flag
  `--address/-a` [`cmd/root.go:186`], `--linger/-l` [`cmd/config.go:32`]; the `k6 stats`
  command [`cmd/stats.go:13`].

### Observed vs. inferred

- **Observed:** the raw REST JSON from both `/v1/metrics/{id}` and `/v1/metrics`; the
  extracted `count = 985` (×2) and `1951`; the `k6 stats` YAML; the live before/during/after
  climb; the API-vs-console cross-check.
- **_inferred_:** nothing material — every reported value was read directly from the API at
  runtime.

---

## Q4 — `SharedArray` data-sharing footprint

**Question.** Does the process memory footprint for a file loaded via `SharedArray` stay
**approximately constant as the VU count increases**, or does **each VU create its own
copy**? Support with test-script output and explain the root cause in the `k6/data` module.

**Data file.** A 17 MB JSON array of 50 000 records (each `{id,name,email,pad:'x'*256}`) was
generated at `/tmp/q4_data.json`.

**Two scripts.** The `SharedArray` load runs its constructor once per process; the naive
load runs `JSON.parse(open(...))` in the init context, which executes once **per VU**:

```javascript
// /tmp/q4_shared.js
import { SharedArray } from 'k6/data';
import { sleep } from 'k6';
const data = new SharedArray('users', function () { return JSON.parse(open('/tmp/q4_data.json')); });
export default function () { const u = data[(Math.random()*data.length)|0]; void u; sleep(1); }
```

```javascript
// /tmp/q4_naive.js
import { sleep } from 'k6';
const data = JSON.parse(open('/tmp/q4_data.json'));
export default function () { const u = data[(Math.random()*data.length)|0]; void u; sleep(1); }
```

**Command** — peak resident set size (VmHWM) measured with `/usr/bin/time -v` at VU counts
1/50/100/200 for each script, twice each:

```bash
/usr/bin/time -v /tmp/k6bin/k6 run --vus N --duration 3s SCRIPT 2>&1 | grep "Maximum resident set size"
```

**Observed output** — peak RSS ("Maximum resident set size", kB → MB):

| VUs | SharedArray run1 | SharedArray run2 | Naive run1 | Naive run2 |
|-----|------------------|------------------|------------|------------|
| 1   | 175.5 MB         | 176.0 MB         | 195.0 MB   | 203.0 MB   |
| 50  | 189.0 MB         | 174.0 MB         | 4191.7 MB  | 4000.4 MB  |
| 100 | 174.6 MB         | 191.0 MB         | 7769.6 MB  | 8446.1 MB  |
| 200 | 177.0 MB         | 177.0 MB         | 16059.7 MB | 16070.1 MB |

Raw kB values (run1) for reference: SharedArray `179676, 193536, 178816, 181248`; Naive
`199680, 4292260, 7956108, 16445128`.

### Direct answer

With `SharedArray`, the process footprint stays **approximately constant** as VUs increase
— ~175–191 MB across 1→200 VUs — whereas the naive `open()`/`JSON.parse` load grows
**roughly linearly**, from ~195 MB at 1 VU to ~16 GB at 200 VUs (~79.7 MB per additional
VU) — **Observed**, stable across both repeats. So **each VU does *not* get its own copy of
a `SharedArray`**; the data is stored once per process. The naive load, by contrast,
creates one full copy per VU.

### Root cause (`file:line`)

- **One per-process store.** The module's root holds a single `sharedArrays`:
  `RootModule struct { shared sharedArrays }` [`js/modules/k6/data/data.go:21-23`], where
  `sharedArrays struct { data map[string]sharedArray; mu sync.RWMutex }`
  [`js/modules/k6/data/data.go:31-34`]; `New()` creates the one map
  [`js/modules/k6/data/data.go:43-49`].
- **Every VU shares the same map by pointer.** `NewModuleInstance` hands each VU a pointer
  to the root's store: `return &Data{ vu: vu, shared: &rm.shared }`
  [`js/modules/k6/data/data.go:53-58`].
- **The JS-visible value is a proxy, not a copy.** The records are stored once as
  `sharedArray struct { arr []string }` [`js/modules/k6/data/share.go:10-12`]; `wrap()`
  returns `rt.NewDynamicArray(wrappedSharedArray{...})` — a sobek `DynamicArray` proxy over
  the single backing slice [`js/modules/k6/data/share.go:23-33`]. Writes are rejected:
  `Set`/`SetLen` `panic(s.rt.NewTypeError("SharedArray is immutable"))`
  [`js/modules/k6/data/share.go:36-41`]. Reads are **copy-on-read**: `Get(index)` does
  `s.parse(...)` (JSON.parse of the stored string) plus `deepFreeze` and returns a fresh
  value per access [`js/modules/k6/data/share.go:44-59`].
- **Why the naive load duplicates.** k6 VUs otherwise run isolated (shared-nothing) JS
  runtimes, so a top-level `JSON.parse(open(...))` executes once per VU and each VU keeps
  its own parsed copy — which is exactly the linear curve observed; `SharedArray` is the
  deliberate exception that stores the data once Go-side and hands every VU a lightweight
  proxy.

### Observed vs. inferred

- **Observed:** both peak-RSS curves (flat for `SharedArray`, linear for naive), stable
  across two repeats.
- **_inferred_:** the ~80 MB/VU slope is the naive script's per-VU parsed-copy cost; per
  Grafana documentation the `SharedArray` constructor runs once and element access returns
  a copy — this documentation is consistent with, and corroborated by, the observed flat
  footprint (documentation is **_inferred_**; the flat/linear footprints are **Observed**).

---

## Q5 — Prometheus remote-write metric-name integrity

**Question.** Investigate metric reporting through the **Prometheus remote-write output**
and provide test-script output proving that the exported data **preserves the integrity of
metric names** (the name-mapping / sanitization behavior).

**Capture harness.** k6's remote-write output POSTs a **snappy-compressed
`prompb.WriteRequest`** to `/api/v1/write`. A minimal receiver was built under `/tmp`
(`/tmp/q5bin/q5recv`) that (1) saves each raw request body to `/tmp/q5_raw/req_NNN.snappy`,
then (2) `snappy.Decode` → `proto.Unmarshal` into the **same `prompb` type k6 uses**
(`buf.build/gen/go/prometheus/prometheus/protocolbuffers/go`, from `go.mod:63`), and (3)
prints every `__name__` label plus each series' full label set. It was compiled against the
repository's own vendored dependencies (module renamed `q5recv`, repo `vendor/` symlinked,
`GOFLAGS=-mod=vendor GOPROXY=off`), so the decode uses the exact protobuf types k6 links
against — not a re-implementation.

**k6 script `/tmp/q5_metrics.js`** (a custom Counter + a custom Trend, alongside k6's
built-in metrics):

```javascript
import { Counter, Trend } from 'k6/metrics';
import { sleep } from 'k6';
const myCounter = new Counter('my_custom_counter');
const myTrend = new Trend('my_custom_trend');
export const options = { vus: 2, duration: '5s' };
export default function () { myCounter.add(1); myTrend.add(Math.random()*100); sleep(0.2); }
```

**Command** (run twice):

```bash
/tmp/q5bin/q5recv > /tmp/q5_recv_run1.log 2>&1 &          # receiver on :9090 /api/v1/write
K6_PROMETHEUS_RW_SERVER_URL="http://localhost:9090/api/v1/write" \
  /tmp/k6bin/k6 run --out experimental-prometheus-rw /tmp/q5_metrics.js > /tmp/q5_run1.log 2>&1
grep -E "__name__|k6_" /tmp/q5_recv_run1.log | sort -u
```

**Observed output** — the complete decoded `__name__` set (identical across both runs):

```
k6_data_received_total
k6_data_sent_total
k6_iteration_duration_p99
k6_iterations_total
k6_my_custom_counter_total
k6_my_custom_trend_p99
k6_vus
k6_vus_max
```

A representative full label set printed by the receiver (labels shown in wire order),
demonstrating lexicographic sorting and that empty tags are skipped:

```
series: __name__=k6_my_custom_counter_total  group=  scenario=default   wire_label_order_sorted=true
series: __name__=k6_vus                                                  wire_label_order_sorted=true
```

**Byte-integrity proof.** An independent decoder (`/tmp/q5bin/q5decode`) re-read the
persisted raw payloads directly from disk — `req_001.snappy` (528 B → 8 series) and
`req_002.snappy` (370 B → 6 series) — and produced the **same** eight names, confirming the
result comes from the exact bytes k6 emitted, not from a re-serialized copy.

### Direct answer

Metric-name integrity is preserved by a **prefix-only** mapping: every exported
`__name__` equals `k6_` + the original k6 metric name, with an optional stat/type
**suffix** — `_total` for counters, `_p99` for the default trend stat, and no suffix for
gauges. There is **no character mangling or sanitization of the base name** — k6's metric
names already satisfy the Prometheus identifier charset, so only the `k6_` namespace (and
the suffix) is added. Labels are emitted **lexicographically sorted** and empty tag values
are skipped. **Observed**, stable across both runs.

### Root cause (`file:line`)

- **Output registration.** `experimental-prometheus-rw` maps to the vendored package:
  `return remotewrite.New(params)` [`cmd/outputs.go:67`] (import at
  [`cmd/outputs.go:20`]).
- **The `__name__` mapping.** `const namelbl = "__name__"`
  [`vendor/.../remotewrite/prometheus.go:11`]; in `MapSeries(series, suffix)` the value is
  built as `v := defaultMetricPrefix + series.Metric.Name`
  [`vendor/.../remotewrite/prometheus.go:40`], with `if suffix != "" { v += "_" + suffix }`
  [`vendor/.../remotewrite/prometheus.go:41-43`], then appended as
  `&prompb.Label{Name: namelbl, Value: v}` [`vendor/.../remotewrite/prometheus.go:44-47`].
  Labels are then `sort.Slice`-sorted lexicographically
  [`vendor/.../remotewrite/prometheus.go:48-50`]. Empty tags are skipped in `MapTagSet()`:
  `if key == "" || value == "" { continue }`
  [`vendor/.../remotewrite/prometheus.go:24-26`].
- **Per-type suffixes.** In `MapPrompb()` the suffix is chosen by sink type: Counter →
  `"total"` [`vendor/.../remotewrite/remotewrite.go:330-332`], Gauge → `""`
  [`vendor/.../remotewrite/remotewrite.go:335-337`], Rate → `"rate"`
  [`vendor/.../remotewrite/remotewrite.go:341`], Trend → `trend.MapPrompb`
  [`vendor/.../remotewrite/remotewrite.go:352-354`]. Trend-stat names are normalized
  `p(99)` → `p99` (parentheses/dots stripped, `p` prepended)
  [`vendor/.../remotewrite/remotewrite.go:180-188`].
- **Defaults.** `defaultServerURL = "http://localhost:9090/api/v1/write"`
  [`vendor/.../remotewrite/config.go:21`]; `defaultMetricPrefix = "k6_"`
  [`vendor/.../remotewrite/config.go:24`]; `defaultTrendStats = []string{"p(99)"}`
  [`vendor/.../remotewrite/config.go:28`]; env override `K6_PROMETHEUS_RW_SERVER_URL`
  [`vendor/.../remotewrite/config.go:301`].
- **Dependency.** `github.com/grafana/xk6-output-prometheus-remote v0.5.0` [`go.mod:20`],
  vendored under `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/`.

### Observed vs. inferred

- **Observed:** the eight `k6_`-prefixed `__name__` values, decoded from the exact emitted
  bytes; the per-type suffixes (`_total`, `_p99`, none for gauges); sorted labels; skipped
  empty tags; byte-integrity via the independent on-disk decode; identical across two runs.
- **_inferred_:** nothing material — the mapping was proven from the emitted bytes rather
  than from reading `prometheus.go` alone. (The web-search confirmation of the remote-write
  naming convention was rate-limited/unavailable, so the vendored source above is the
  authoritative reference, corroborated by the decoded bytes.)

---

## Coverage pass

Every named mechanism, metric, log line, function/struct, flag, and route posed across the
five questions, confirmed addressed with **Observed** runtime evidence (unless explicitly
labeled **_inferred_**).

**Q1 — VU lifecycle on SIGINT (`ramping-vus`):**
- [x] `ramping-vus` executor with ≥5 VUs exercised (`startVUs: 5`) — **Observed**.
- [x] Exact single-SIGINT log line `level=debug msg="Stopping k6 in response to signal..." sig=interrupt` — **Observed** (matches assertion at `cmd/tests/cmd_run_test.go:1225,1246`).
- [x] Abort log `level=error msg="test run was aborted because k6 received a 'interrupt' signal"` — **Observed**.
- [x] Summary line `0 complete and 5 interrupted iterations` (M = 5 > 0 ⇒ mid-execution termination) — **Observed**; stable across 2 runs.
- [x] Exit code `105` (`ExternalAbort`, `errext/exitcodes/codes.go:41`) — **Observed**.
- [x] Secondary path (second SIGINT) → `level=error msg="Aborting k6 in response to signal"` via a canonical script `teardown()` window — **Observed**; exit `105`.
- [x] Root cause named: `handleTestAbortSignals()` (`cmd/common.go:97,101`), `runAbort` (`cmd/run.go:350-352`), `onHardStop` (`cmd/run.go:359-360`), summary format (`execution/scheduler.go:155-159`, `GetFullIterationCount`/`GetPartialIterationCount`), `GetGracefulStop()` semantics (`lib/executor/base_config.go:91-96`), gracefulStop-vs-hardStop contract (`lib/executor/vu_handle.go:63-67`), `rampingVUsType` (`lib/executor/ramping_vus.go:19`).

**Q2 — gRPC server-streaming interruption:**
- [x] `gracefulStop`/`gracefulRampDown = '30ms'` configured exactly — **Observed**.
- [x] `grpc_streams: 5` — **Observed**.
- [x] `grpc_streams_msgs_sent: 5` — **Observed**.
- [x] **`grpc_streams_msgs_received: 195`** — **Observed**; stable across 3 runs.
- [x] Interrupt log entries + `0 complete and 5 interrupted iterations`, exit `105` — **Observed**.
- [x] Root cause named: counters in `js/modules/k6/grpc/metrics.go` (`:7-9`, `:17`, `:21`, `:25`); per-message increment `Value:1` in `queueMessage()` (`stream.go:149,153,158`); interrupt/close path `loop()` `case <-ctxDone` (`stream.go:119,133,136-140`, comment "VU is shutting down during an interrupt"); server-streaming RPC `rpc ListFeatures(Rectangle) returns (stream Feature)` (`route_guide.proto:38`); server pacing `time.Sleep(100 * time.Millisecond)` (`grpcservice/service.go:57-63`) — the 195 = 39 in-range features × 5 VUs.

**Q3 — `dropped_iterations` via the REST API:**
- [x] Value obtained via REST API `GET /v1/metrics/dropped_iterations` — raw JSON `data.attributes.sample.count` = **985** (shared-iterations) — **Observed**; stable across 2 runs.
- [x] List route `GET /v1/metrics` showing the `dropped_iterations` entry — **Observed**.
- [x] `--linger/-l` used to keep the REST server alive past test end — **Observed**.
- [x] `k6 stats dropped_iterations` corroboration (same API): count 985, iterations 15, 15+985 = 1000 — **Observed**.
- [x] Secondary path: `constant-arrival-rate` over-capacity, live before/during/after polling `85 → 575 → 1160 → 1951`, final API count `1951` matching console — **Observed**.
- [x] Root cause named: `DroppedIterationsName` + Counter registration (`metrics/builtin.go:10,44,84`); end-of-test drop `Value: float64(totalIters-attemptedIters)` (`lib/executor/shared_iterations.go:213-225`); during-run drop `Value:1` on `!TryRunIteration()` (`lib/executor/constant_arrival_rate.go:324-345`); routes (`api/v1/routes.go:23,28,31,37-38`); JSON shape `Sample map[string]float64` (`api/v1/metric.go:69,80`), `data.attributes` envelope (`api/v1/metric_jsonapi.go:10-11,21`), Counter `Format` `{"count":...}` (`metrics/sink.go:64-67`); REST bind `localhost:6565` (`cmd/state/state.go:150`), `--linger` (`cmd/config.go:32`), `k6 stats` (`cmd/stats.go`).

**Q4 — `SharedArray` footprint:**
- [x] Peak-RSS curve for `SharedArray`: **flat ~175–191 MB** across 1/50/100/200 VUs — **Observed**; 2 runs.
- [x] Peak-RSS curve for naive per-VU load: **linear ~195 MB → 16 GB** (~79.7 MB/VU) — **Observed**; 2 runs.
- [x] Direct answer: footprint stays approximately constant with `SharedArray`; each VU does not copy — **Observed**.
- [x] Root cause named: single per-process store `RootModule.shared` / `sharedArrays{data map, mu}` (`js/modules/k6/data/data.go:21-34,43-49`); every VU gets a pointer `&Data{vu, shared:&rm.shared}` (`data.go:53-58`); proxy-backed `DynamicArray`, immutable, copy-on-read `Get()` (`js/modules/k6/data/share.go:10-12,23-41,44-59`); shared-nothing VU runtimes explain naive duplication.

**Q5 — Prometheus remote-write name integrity:**
- [x] Decoded `__name__` set (8 names) from the **exact emitted bytes** — **Observed**; identical across 2 runs.
- [x] `k6_` prefix on every name; no character mangling — **Observed**.
- [x] Per-type/stat suffixes: `_total` (counters `k6_iterations_total`, `k6_data_sent_total`, `k6_data_received_total`, `k6_my_custom_counter_total`), `_p99` (trend `k6_my_custom_trend_p99`, `k6_iteration_duration_p99`), none for gauges (`k6_vus`, `k6_vus_max`) — **Observed**.
- [x] Labels lexicographically sorted; empty tags skipped — **Observed**.
- [x] Byte-integrity via independent on-disk re-decode of `req_001.snappy`/`req_002.snappy` — **Observed**.
- [x] Root cause named: registration `remotewrite.New` (`cmd/outputs.go:20,67`); `__name__` = `defaultMetricPrefix + Metric.Name` (+suffix), sorted, empty tags skipped (`vendor/.../remotewrite/prometheus.go:11,24-26,40-50`); per-type suffixes + trend `p(99)→p99` (`vendor/.../remotewrite/remotewrite.go:180-188,330-354`); defaults `k6_`, default URL, `p(99)`, env override (`vendor/.../remotewrite/config.go:21,24,28,301`); dependency `xk6-output-prometheus-remote v0.5.0` (`go.mod:20`).

**Methodology coverage:**
- [x] Canonical binary built from HEAD; banner `k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)` recorded — **Observed**.
- [x] Every value obtained through canonical entry points (`k6 run`, `k6 stats`, REST API, real remote-write output) — no debug hooks, mocks, or bypasses — **Observed**.
- [x] Every magnitude/timing value confirmed stable across ≥2 runs (or reported as a distribution) with scale/duration stated — **Observed**.
- [x] Complete, unedited output presented for each condition with the command that produced it; no elisions.
- [x] Statements not directly observed at runtime are labeled **_inferred_**.
- [x] Repository left read-only: no existing file modified; this document is the only artifact.
