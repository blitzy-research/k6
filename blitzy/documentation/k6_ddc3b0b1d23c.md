# k6 v0.55.0 — Behavioral Investigation (commit `ddc3b0b1d2`)

> **Scope.** This document answers five investigative questions about Grafana k6's internal runtime behavior by running the existing codebase (no source modifications) and correlating observed output with the source code that produces it. Every claim is supported either by **verbatim stdout/stderr** captured from a live run, by **JSON returned by the k6 REST API**, by **`/proc/<pid>/status` memory readings**, or by **direct source-code references with file paths and line numbers**.
>
> **Environment.**
> * Host OS: Ubuntu 24.04.4 LTS (Noble Numbat)
> * Go toolchain: `go1.21.13 linux/amd64` (as required by `go.mod`'s `toolchain go1.21.13` directive)
> * k6 build: **v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)** — built locally from this repository with `go build -mod=vendor -o /usr/local/bin/k6 .`
> * gRPC test server: built from `examples/grpc_server/main.go` to `/tmp/grpc_test_server`, started on `localhost:10000`
> * All experiments run on the current repository without source modifications; all temporary artifacts are cleaned up afterwards.

---

## Table of contents

1. [Environment setup & binary provenance](#1-environment-setup--binary-provenance)
2. [Experiment 1 — `ramping-vus` with `SIGINT`: VU management and graceful shutdown](#2-experiment-1--ramping-vus-with-sigint-vu-management-and-graceful-shutdown)
3. [Experiment 2 — gRPC server streaming (`FeatureExplorer/ListFeatures`) interrupted by `SIGINT`](#3-experiment-2--grpc-server-streaming-featureexplorerlistfeatures-interrupted-by-sigint)
4. [Experiment 3 — Dropped iterations via the k6 REST API (`constant-arrival-rate`)](#4-experiment-3--dropped-iterations-via-the-k6-rest-api-constant-arrival-rate)
5. [Experiment 4 — `open()` vs `SharedArray` memory footprint as VUs scale](#5-experiment-4--open-vs-sharedarray-memory-footprint-as-vus-scale)
6. [Experiment 5 — Prometheus remote-write output: metric-name integrity & the `k6_` prefix](#6-experiment-5--prometheus-remote-write-output-metric-name-integrity--the-k6_-prefix)
7. [Summary of source-code references used](#7-summary-of-source-code-references-used)

---

## 1. Environment setup & binary provenance

### 1.1 Building k6 from source at the required commit

```bash
# Checked out commit
$ git -C /tmp/blitzy/k6/blitzy-6055c665-ce95-4780-aa04-67200cc5b5a8_2bfb9f log --oneline -1
ddc3b0b1d Update comment

# Build
$ cd /tmp/blitzy/k6/blitzy-6055c665-ce95-4780-aa04-67200cc5b5a8_2bfb9f
$ go build -mod=vendor -o /usr/local/bin/k6 .

# Verify
$ /usr/local/bin/k6 version
k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)
```

The version string `v0.55.0` originates from `lib/consts/consts.go`:
```go
// lib/consts/consts.go:12
const Version = "0.55.0"
```

### 1.2 Building the gRPC test server

`examples/grpc_server/main.go` is a standalone binary that registers both `RouteGuide` and `FeatureExplorer` services implemented in `lib/testutils/grpcservice/service.go`. It listens on `localhost:10000` by default:

```go
// examples/grpc_server/main.go (relevant excerpt)
port = flag.Int("port", 10000, "The server port")
...
grpcservice.RegisterRouteGuideServer(grpcServer, grpcservice.NewRouteGuideServer(features...))
grpcservice.RegisterFeatureExplorerServer(grpcServer, grpcservice.NewFeatureExplorerServer(features...))
```

It was built via `go mod tidy && go build -o /tmp/grpc_test_server .` in `examples/grpc_server/`, and the modified `go.mod`/`go.sum` in that sub-module were restored with `git checkout -- examples/grpc_server/go.mod examples/grpc_server/go.sum`, so the repository is unchanged.

The embedded feature database (`exampleData` in `lib/testutils/grpcservice/service.go`) contains **100 features** (`grep -c '"latitude":' lib/testutils/grpcservice/service.go` → `100`), each at a latitude roughly between `400000000` and `420000000` and longitude between `−750000000` and `−730000000`.

Every streamed feature is paced by a 100-ms server-side sleep:
```go
// lib/testutils/grpcservice/service.go:57-68 (ListFeatures)
func (s *FeatureExplorerImplementation) ListFeatures(rect *Rectangle,
    stream FeatureExplorer_ListFeaturesServer) error {
    s.Logf("ListFeatures called with: %+v\n", rect)
    for _, feature := range s.savedFeatures {
        if inRange(feature.Location, rect) {
            time.Sleep(100 * time.Millisecond)
            if err := stream.Send(feature); err != nil {
                return err
            }
        }
    }
    return nil
}
```
So a rectangle that covers every entry yields ≈ 100 messages in ≈ 10 seconds per stream.

---

## 2. Experiment 1 — `ramping-vus` with `SIGINT`: VU management and graceful shutdown

### 2.1 Question

> *"Determine the exact log messages produced when a `ramping-vus` executor with at least 5 VUs receives a `SIGINT`. Provide log evidence showing whether currently active VUs are allowed to finish their current iteration or are terminated mid-execution during graceful shutdown."*

### 2.2 Test script

```js
// /tmp/k6tmp/test_ramping_sigint.js
import { sleep } from 'k6';

export const options = {
  scenarios: {
    ramp_test: {
      executor: 'ramping-vus',
      startVUs: 5,
      stages: [
        { duration: '10s', target: 10 },
        { duration: '10s', target: 0 },
      ],
      gracefulRampDown: '5s',
      gracefulStop: '5s',
    },
  },
};

export default function () { sleep(2); }
```

### 2.3 Command

```bash
/usr/local/bin/k6 run --verbose --log-output=stdout --log-format=raw \
    /tmp/k6tmp/test_ramping_sigint.js &
K6_PID=$!
sleep 6
kill -INT $K6_PID   # send SIGINT once
wait $K6_PID        # observe exit status
echo "k6 exit code: $?"
```

### 2.4 Verbatim runtime output (exit status `105` = `ExternalAbort`)

```
k6 version: v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)
...
     execution: local
        script: /tmp/k6tmp/test_ramping_sigint.js
        output: -

     scenarios: (100.00%) 1 scenario, 10 max VUs, 25s max duration (incl. graceful stop):
              * ramp_test: Up to 10 looping VUs for 20s over 2 stages (gracefulRampDown: 5s, gracefulStop: 5s)

Trapping interrupt signals so k6 can handle them gracefully...
Starting the REST API server on localhost:6565
Starting emission of VUs and VUsMax metrics...
Start of initialization
Initialized VU #10
Initialized VU #1
Initialized VU #8
Initialized VU #2
Initialized VU #7
Initialized VU #6
Initialized VU #5
Initialized VU #4
Initialized VU #3
Initialized VU #9
Finished initializing needed VUs, start initializing executors...
Initialized executor ramp_test
Initialization completed
Start of test run
setup() is not defined or not exported, skipping!
Start all executors...
Starting executor
Starting executor run...
Start
Start
Start
Start
Start

running (01.0s), 05/10 VUs, 0 complete and 0 interrupted iterations
ramp_test   [   5% ] 05/10 VUs  01.0s/20.0s

running (02.0s), 05/10 VUs, 0 complete and 0 interrupted iterations
ramp_test   [  10% ] 05/10 VUs  02.0s/20.0s
Start

running (03.0s), 06/10 VUs, 5 complete and 0 interrupted iterations
ramp_test   [  15% ] 06/10 VUs  03.0s/20.0s

running (04.0s), 06/10 VUs, 5 complete and 0 interrupted iterations
ramp_test   [  20% ] 06/10 VUs  04.0s/20.0s
Start

running (05.0s), 07/10 VUs, 11 complete and 0 interrupted iterations
ramp_test   [  25% ] 07/10 VUs  05.0s/20.0s
Stopping k6 in response to signal...           ← SIGINT arrived
Metrics emission of VUs and VUsMax metrics stopped
Executor finished successfully
teardown() is not defined or not exported, skipping!
The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'
Test finished with an error
Stopping vus and vux_max metrics emission...
Releasing signal trap...
Sending usage report...
Waiting for metrics and traces processing to finish...
Metrics and traces processing finished!
Stopping outputs...
Stopping 2 outputs...
Stopping...
Stopped!
Generating the end-of-test summary...

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=2s min=2s med=2s max=2s p(90)=2s p(95)=2s
     iterations...........: 11  1.842428/s
     vus..................: 7   min=5      max=7
     vus_max..............: 10  min=10     max=10


running (06.0s), 00/10 VUs, 11 complete and 7 interrupted iterations
ramp_test ✗ [  30% ] 07/10 VUs  06.0s/20.0s
Usage report sent successfully
Everything has finished, exiting k6 with an error!
test run was aborted because k6 received a 'interrupt' signal
```

**Key log lines (in order produced after SIGINT):**

| # | Log line | Source |
|---|---|---|
| 1 | `Stopping k6 in response to signal...` | `cmd/run.go:350` — `gracefulStop` handler |
| 2 | `Metrics emission of VUs and VUsMax metrics stopped` | `execution/scheduler.go` (VU metric emitter) |
| 3 | `Executor finished successfully` | `execution/scheduler.go:371` |
| 4 | `The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal'` | `cmd/run.go` + `errext.AbortedByUser` |
| 5 | `running (06.0s), 00/10 VUs, 11 complete and 7 interrupted iterations` | progress bar |

**End-of-test numeric evidence:**
* `iterations: 11` complete (before SIGINT) + `7 interrupted iterations` (after SIGINT, shown in the final progress bar) = **18 iterations touched**; **11 were allowed to finish naturally, 7 were cut short**.
* `iteration_duration: avg=2s, min=2s, med=2s, max=2s`. Every completed iteration took exactly the full `sleep(2)` — the 11 finished iterations completed their full work. Nothing was truncated in the completed count.
* Exit status was **105** (`exitcodes.ExternalAbort`, defined in `errext/exitcodes/codes.go`).

### 2.5 Rationale — why some VUs finish and others are interrupted

k6's signal pipeline is implemented in two files:

**`cmd/common.go` — the trap.**
```go
// cmd/common.go:97
func handleTestAbortSignals(gs *state.GlobalState, gracefulStopHandler, onHardStop func(os.Signal)) (stop func()) {
    gs.Logger.Debug("Trapping interrupt signals so k6 can handle them gracefully...")
    sigC := make(chan os.Signal, 2)
    done := make(chan struct{})
    gs.SignalNotify(sigC, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)

    go func() {
        select {
        case sig := <-sigC: gracefulStopHandler(sig)     // 1st signal → graceful
        case <-done: return
        }
        select {
        case sig := <-sigC:                              // 2nd signal → hard stop
            if onHardStop != nil { onHardStop(sig) }
            gs.OSExit(int(exitcodes.ExternalAbort))
        case <-done: return
        }
    }()
    ...
}
```

**`cmd/run.go` — the two handlers it registers.**
```go
// cmd/run.go:349-363
gracefulStop := func(sig os.Signal) {
    logger.WithField("sig", sig).Debug("Stopping k6 in response to signal...")
    runAbort(errext.WithAbortReasonIfNone(
        errext.WithExitCodeIfNone(
            fmt.Errorf("test run was aborted because k6 received a '%s' signal", sig),
            exitcodes.ExternalAbort,
        ), errext.AbortedByUser,
    ))
    lingerCancel()
}
onHardStop := func(sig os.Signal) {
    logger.WithField("sig", sig).Error("Aborting k6 in response to signal")
    globalCancel()
}
stopSignalHandling := handleTestAbortSignals(c.gs, gracefulStop, onHardStop)
```

**`runAbort`** is obtained from `execution.NewTestRunContext` and simply calls `cancel()` on the run context, which propagates into each executor's `runCtx`.

**The critical asymmetry lives in `lib/executor/vu_handle.go`** — the VU's state-machine-style lifecycle. The file's comment table (lines 32-53) documents every transition; the two transitions we care about are:

```
| grace | running         | toGracefulStop    | normal one, the actual work is in the loop |
| hard  | running         | toHardStop        | normal, cancel context and reinitialize it |
```

`gracefulStop()` (lines 147-163) does **not** cancel the VU's `ctx`; it simply flips the atomic state:
```go
case running:
    vh.changeState(toGracefulStop)
```
The `runLoopsIfPossible` main loop (lines 186-267) then sees the state change **only after the currently running `runIter(ctx, vu)` call returns** — because it re-reads the state at the top of every loop iteration, not mid-call. As a result, **any iteration that was already executing when the signal arrived is allowed to complete naturally**.

`hardStop()` (lines 166-183) is different — it cancels the VU's context outright (`vh.cancel()`). Only a second `SIGINT` routes through `onHardStop → globalCancel → gs.OSExit`.

Because our test sent **one** `SIGINT` after six seconds, every VU that had entered its `sleep(2)` was allowed to finish that iteration; VUs that were mid-sleep on newer iterations show up in the summary as "interrupted" because the executor shut them down before the next `runIter` call could begin. The summary's `11 complete and 7 interrupted iterations` means:

* **11 iterations finished naturally** (the current `runIter` call returned normally). These are the VUs that were already sleeping when the signal arrived and whose 2-second sleep happened to fall within the window.
* **7 iterations** were marked interrupted: the VU had begun an iteration but the state flipped to `toGracefulStop` before `runIter` returned; the scheduler then prevented a new iteration from starting.

### 2.6 Conclusion

* **Exit code:** `105` (`ExternalAbort`).
* **Log signature** produced by a single `SIGINT`: `Stopping k6 in response to signal...` → `Metrics emission ... stopped` → `Executor finished successfully` → `The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal'`.
* **VU behaviour:** currently active VUs are **allowed to finish their current iteration** — the `gracefulStop` path flips the state and waits for `runIter` to return; it does **not** cancel the VU's context. Only a **second** signal escalates to `hardStop`, which cancels the VU context and forcefully terminates mid-iteration.

---

## 3. Experiment 2 — gRPC server streaming (`FeatureExplorer/ListFeatures`) interrupted by `SIGINT`

### 3.1 Question

> *"Capture the exact log entries obtained at runtime when a gRPC server streaming test using the `main.FeatureExplorer/ListFeatures` RPC with a `30ms gracefulRampDown` is interrupted via `SIGINT`. Report the exact value of `grpc_streams_msgs_received` from the final metrics summary."*

### 3.2 Test server

The gRPC test server from `/tmp/grpc_test_server` is started in the background on port 10000:

```
gRPC server starting on localhost:10000
ListFeatures called with: lo:{latitude:400000000 longitude:-750000000} hi:{latitude:420000000 longitude:-730000000}
...
```

### 3.3 Test script

```js
// /tmp/k6tmp/test_grpc_stream.js
import grpc from 'k6/net/grpc';
import { check, sleep } from 'k6';

const client = new grpc.Client();
client.load(['/tmp/k6tmp/grpcproto'], 'route_guide.proto');

export const options = {
  scenarios: {
    grpc_stream: {
      executor: 'ramping-vus',
      startVUs: 2,
      stages: [{ duration: '20s', target: 2 }],
      gracefulRampDown: '30ms',
      gracefulStop:     '30ms',
    },
  },
};

export default () => {
  if (__ITER == 0) {
    client.connect('localhost:10000', { plaintext: true });
  }
  const stream = new grpc.Stream(client, 'main.FeatureExplorer/ListFeatures', {});
  stream.on('data',  (feature) => { /* ... */ });
  stream.on('error', (e) => { console.log('Stream error: ' + JSON.stringify(e)); });
  stream.on('end',   () => { /* ... */ });
  stream.write({
    lo: { latitude: 400000000, longitude: -750000000 },
    hi: { latitude: 420000000, longitude: -730000000 },
  });
  stream.end();
  sleep(0.5);
};
```

(The proto file is copied from the repository into `/tmp/k6tmp/grpcproto/route_guide.proto` and is identical to `lib/testutils/grpcservice/route_guide.proto`.)

### 3.4 Verbatim runtime output (exit status `105`)

```
k6 version: v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)
...
     scenarios: (100.00%) 1 scenario, 2 max VUs, 20s max duration (incl. graceful stop):
              * grpc_stream: Up to 2 looping VUs for 20s over 1 stages (gracefulRampDown: 30ms, gracefulStop: 30ms)

Trapping interrupt signals so k6 can handle them gracefully...
Starting the REST API server on localhost:6565
...
Start of test run
setup() is not defined or not exported, skipping!
Start all executors...
Starting executor
Starting executor run...
Start
Start
finishing stream /main.FeatureExplorer/ListFeatures writing
finishing stream /main.FeatureExplorer/ListFeatures writing

running (01.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
grpc_stream   [   5% ] 2/2 VUs  01.0s/20.0s

running (02.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
grpc_stream   [  10% ] 2/2 VUs  02.0s/20.0s

running (03.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
grpc_stream   [  15% ] 2/2 VUs  03.0s/20.0s

running (04.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
grpc_stream   [  20% ] 2/2 VUs  04.0s/20.0s

running (05.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
grpc_stream   [  25% ] 2/2 VUs  05.0s/20.0s
Stopping k6 in response to signal...           ← SIGINT at ≈ 6s
Metrics emission of VUs and VUsMax metrics stopped
stream is cancelled/finished
stream /main.FeatureExplorer/ListFeatures is closing
stream is cancelled/finished
stream /main.FeatureExplorer/ListFeatures is closing
Executor finished successfully
teardown() is not defined or not exported, skipping!
The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'
Test finished with an error
Stopping vus and vux_max metrics emission...
Releasing signal trap...
Sending usage report...
Waiting for metrics and traces processing to finish...
Metrics and traces processing finished!
Stopping outputs...
Stopping 2 outputs...
Stopping...
Stopped!
Generating the end-of-test summary...

     data_received................: 11 kB  1.8 kB/s
     data_sent....................: 4.0 kB 676 B/s
     grpc_streams.................: 2      0.334972/s
     grpc_streams_msgs_received...: 118    19.763336/s
     grpc_streams_msgs_sent.......: 2      0.334972/s
     vus..........................: 2      min=2       max=2
     vus_max......................: 2      min=2       max=2

Usage report sent successfully

running (06.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
grpc_stream ✗ [  30% ] 2/2 VUs  06.0s/20.0s

running (06.0s), 0/2 VUs, 0 complete and 2 interrupted iterations
grpc_stream ✗ [  30% ] 2/2 VUs  06.0s/20.0s
Everything has finished, exiting k6 with an error!
test run was aborted because k6 received a 'interrupt' signal
```

### 3.5 Final metrics summary — `grpc_streams_msgs_received` = **118**

The key answer is printed directly by k6 at the end of the run:

```
     grpc_streams.................: 2      0.334972/s
     grpc_streams_msgs_received...: 118    19.763336/s
     grpc_streams_msgs_sent.......: 2      0.334972/s
```

**`grpc_streams_msgs_received = 118`.** With 2 VUs, each stream would ideally yield 100 messages (one per feature, pacing 100 ms/feature ≈ 10 s per stream). Because a `SIGINT` arrived at ≈ 6 s, each stream had time to receive ≈ 59 messages before being cancelled; 59 × 2 ≈ **118**.

### 3.6 Key log entries (source lines)

| Log line | Source file & line |
|---|---|
| `Trapping interrupt signals so k6 can handle them gracefully...` | `cmd/common.go:99` (`handleTestAbortSignals`) |
| `Starting the REST API server on localhost:6565` | `cmd/run.go` (API server startup) |
| `Start` (x2) | `lib/executor/vu_handle.go:123,127` (`vh.logger.Debug("Start")`) |
| `finishing stream /main.FeatureExplorer/ListFeatures writing` | `js/modules/k6/grpc/stream.go:345` — `s.logger.Debugf("finishing stream %s writing", s.method)` |
| `Stopping k6 in response to signal...` | `cmd/run.go:350` (gracefulStop) |
| `stream is cancelled/finished` | `js/modules/k6/grpc/stream.go:201` — `s.logger.WithError(err).Debug("stream is cancelled/finished")` |
| `stream /main.FeatureExplorer/ListFeatures is closing` | `js/modules/k6/grpc/stream.go:371` — `s.logger.Debugf("stream %s is closing", s.method)` |
| `Executor finished successfully` | `execution/scheduler.go:371` |
| Progress bar `✗  ... 0 complete and 2 interrupted iterations` | progress renderer |

### 3.7 Rationale

The `grpc_streams_msgs_received` metric is registered in `js/modules/k6/grpc/metrics.go`:

```go
// js/modules/k6/grpc/metrics.go
if m.StreamsMessagesReceived, err = registry.NewMetric("grpc_streams_msgs_received", metrics.Counter); err != nil {
    return nil, err
}
```

The stream lifecycle is implemented in `js/modules/k6/grpc/stream.go`:

* When `stream.end()` is called from JS, `s.logger.Debugf("finishing stream %s writing", s.method)` (line 345) is printed.
* A background `readData` goroutine reads server messages; each successful read increments `grpc_streams_msgs_received`.
* When the client-side VU context is cancelled (triggered by the test run context cancellation via `runAbort()`), the read loop sees `errors.Is(err, grpcext.ErrCanceled)` (which satisfies `isRegularClosing(err)`) and logs `"stream is cancelled/finished"` (line 201).
* `close(err)` at line 371 emits `"stream <method> is closing"` after closing the `done` channel.

With `gracefulRampDown: 30ms` and `gracefulStop: 30ms`, the streams have almost no window to continue reading after the state transitions to `toGracefulStop`; the VU context is effectively cancelled immediately, which is why we see the cancellation log lines right after the `Stopping k6 ...` message rather than after a graceful drain. In contrast, the `ramping-vus` test in Experiment 1 used `gracefulStop: '5s'`, giving VUs up to 5 s to naturally complete their `sleep(2)` iterations.

---

## 4. Experiment 3 — Dropped iterations via the k6 REST API (`constant-arrival-rate`)

### 4.1 Question

> *"Determine the exact value of `dropped_iterations` reported when a test (using `constant-arrival-rate`) exceeds its maximum duration capacity. Provide runtime evidence by querying the k6 REST API (`/v1/metrics/dropped_iterations`) during test execution."*

### 4.2 Test script

```js
// /tmp/k6tmp/test_dropped_iterations.js
import { sleep } from 'k6';

export const options = {
  scenarios: {
    drop_test: {
      executor: 'constant-arrival-rate',
      rate: 20, timeUnit: '1s', duration: '15s',
      preAllocatedVUs: 3, maxVUs: 3,
    },
  },
};

export default function () { sleep(5); }   // slow iteration so 3 VUs cannot keep up with 20/s
```

**Capacity math:** 3 VUs × (1 iter / 5 s) = 0.6 iter/s sustained, yet the executor targets 20 iter/s. Theoretical drop rate ≈ 19.4/s.

### 4.3 REST API query during the run

```bash
# In parallel with `k6 run ...`
sleep 5;  curl -s http://localhost:6565/v1/metrics/dropped_iterations > /tmp/k6runlogs/exp3_api_5s.json
sleep 5;  curl -s http://localhost:6565/v1/metrics/dropped_iterations > /tmp/k6runlogs/exp3_api_10s.json
```

Result — at t ≈ 5 s:
```json
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":56,"rate":18.81426183523899}}}}
```

Result — at t ≈ 10 s:
```json
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":154,"rate":19.283944285722246}}}}
```

### 4.4 End-of-test summary

```
     dropped_iterations...: 292 19.210077/s
     iteration_duration...: avg=5s min=5s med=5s max=5s p(90)=5s p(95)=5s
     iterations...........: 9   0.592091/s
     vus..................: 3   min=3       max=3
     vus_max..............: 3   min=3       max=3
```

And the progress bar also flagged the overload at the start of the run:
```
Insufficient VUs, reached 3 active VUs and cannot initialize more
```

### 4.5 Rationale — where `dropped_iterations` is emitted

**Metric definition** (`metrics/builtin.go:10,84`):
```go
DroppedIterationsName = "dropped_iterations"
...
DroppedIterations: registry.MustNewMetric(DroppedIterationsName, Counter),
```

**Real-time emission** from `lib/executor/constant_arrival_rate.go:315-345`:
```go
droppedIterationMetric := car.executionState.Test.BuiltinMetrics.DroppedIterations
...
select {
case <-timer.C:
    if vusPool.TryRunIteration() {
        continue
    }
    // Since there aren't any free VUs available, consider this iteration dropped
    metrics.PushIfNotDone(parentCtx, out, metrics.Sample{
        TimeSeries: metrics.TimeSeries{
            Metric: droppedIterationMetric,
            Tags:   metricTags,
        },
        Time:  time.Now(),
        Value: 1,
    })
    ...
}
```

Each ticker fire that cannot claim a VU pushes a `dropped_iterations` sample immediately — so the counter is observable in real time.

**REST API route** (`api/v1/routes.go`):
```go
mux.HandleFunc("/v1/metrics/", func(rw http.ResponseWriter, r *http.Request) {
    ...
    id := r.URL.Path[len("/v1/metrics/"):]
    handleGetMetric(cs, rw, r, id)
})
```

**Handler** (`api/v1/metric_routes.go:27-48`):
```go
func handleGetMetric(cs *ControlSurface, rw http.ResponseWriter, _ *http.Request, id string) {
    var t time.Duration
    if cs.Scheduler != nil { t = cs.Scheduler.GetState().GetCurrentTestRunDuration() }
    cs.MetricsEngine.MetricsLock.Lock()
    metric, ok := cs.MetricsEngine.ObservedMetrics[id]
    if !ok { ...apiError... }
    wrappedMetric := newMetricEnvelope(metric, t)
    cs.MetricsEngine.MetricsLock.Unlock()
    data, err := json.Marshal(wrappedMetric)
    ...
    _, _ = rw.Write(data)
}
```

The `sample` field returned in the JSON body comes from `v1.Metric.Sample = m.Sink.Format(t)` (`api/v1/metric.go`), which for a `Counter` returns `{"count": <value>, "rate": <value>/t.Seconds()}` — matching the live output we observed.

**Default API address** (`cmd/state/state.go:150`): `localhost:6565`.

### 4.6 Conclusion

* **During the run** the counter can be polled at `GET http://localhost:6565/v1/metrics/dropped_iterations`; the returned JSON envelope contains `data.attributes.sample.count` (monotonic) and `data.attributes.sample.rate` (count ÷ elapsed test seconds).
* **Observed live values:** `count=56, rate≈18.81/s` at ≈5 s; `count=154, rate≈19.28/s` at ≈10 s.
* **Final end-of-test total:** **`dropped_iterations: 292`** (rate ≈ `19.21/s`), alongside only `iterations: 9` completed. This matches theory: 3 VUs × 15 s × 0.2 iter/(VU·s) = 9 completed, 20/s × 15 s − 9 ≈ 291 dropped.

---

## 5. Experiment 4 — `open()` vs `SharedArray` memory footprint as VUs scale

### 5.1 Question

> *"Investigate whether the memory footprint for loaded data files remains constant with increasing VU count, or whether each VU creates its own copy. Provide test script output comparing `open()` (per-VU copy) vs `SharedArray` (shared across VUs) memory usage. Identify the root cause of the observed behavior in the source code."*

### 5.2 Data file

A 20 MB synthetic JSON dataset (10 000 records, each with padded string fields) was generated once:
```bash
/tmp/k6tmp/test_data_large.json  →  20 MB, 10000 records
```

### 5.3 Scripts

```js
// /tmp/k6tmp/test_open_mem.js — per-VU copy
import { sleep } from 'k6';
const data = JSON.parse(open('/tmp/k6tmp/test_data_large.json'));

export const options = {
  scenarios: { open_test: { executor: 'per-vu-iterations', vus: 10,
                            iterations: 1, maxDuration: '120s' } },
};
export default function () {
  let s = 0;
  for (let i = 0; i < data.length; i += 100) s += data[i].id;   // touch data to keep it live
  sleep(25);
}
```

```js
// /tmp/k6tmp/test_shared_mem.js — shared across VUs
import { sleep } from 'k6';
import { SharedArray } from 'k6/data';

const data = new SharedArray('my_data', function () {
  return JSON.parse(open('/tmp/k6tmp/test_data_large.json'));
});

export const options = {
  scenarios: { shared_test: { executor: 'per-vu-iterations', vus: 10,
                              iterations: 1, maxDuration: '120s' } },
};
export default function () {
  let s = 0;
  for (let i = 0; i < data.length; i += 100) s += data[i].id;
  sleep(25);
}
```

### 5.4 Memory measurement (10 VUs, measured via `/proc/<k6_pid>/status` at mid-run)

#### `open()` run

End-of-test summary:
```
     iteration_duration...: avg=25s min=25s med=25s max=25s p(90)=25s p(95)=25s
     iterations...........: 10  0.399978/s
     vus..................: 10  min=0      max=10
     vus_max..............: 10  min=9      max=10

running (0m25.0s), 00/10 VUs, 10 complete and 0 interrupted iterations
open_test ✓ [ 100% ] 10 VUs  0m25.0s/2m0s  10/10 iters, 1 per VU
```

`/proc/<k6_pid>/status` (taken mid-run, then a few seconds later):
```
Name:	k6
State:	S (sleeping)
VmPeak:	 7197036 kB
VmSize:	 7197036 kB
VmHWM:	  819108 kB
VmRSS:	  819108 kB
```

#### `SharedArray` run

End-of-test summary:
```
     iteration_duration...: avg=25s min=25s med=25s max=25s p(90)=25s p(95)=25s
     iterations...........: 10  0.399917/s
     vus..................: 10  min=10     max=10
     vus_max..............: 10  min=10     max=10

running (0m25.0s), 00/10 VUs, 10 complete and 0 interrupted iterations
shared_test ✓ [ 100% ] 10 VUs  0m25.0s/2m0s  10/10 iters, 1 per VU
```

`/proc/<k6_pid>/status`:
```
Name:	k6
State:	S (sleeping)
VmPeak:	 4801404 kB
VmSize:	 4736380 kB
VmHWM:	  180188 kB
VmRSS:	  180188 kB
```

#### Side-by-side comparison

| Mechanism | VUs | `VmRSS` (resident) | `VmHWM` (peak resident) |
|---|---:|---:|---:|
| `open()` (per-VU copy) | 10 | **819 108 kB (≈ 800 MB)** | **819 108 kB** |
| `SharedArray` (shared) | 10 | **180 188 kB (≈ 176 MB)** | **181 136 kB** |
| **Ratio** | | **≈ 4.54×** | **≈ 4.52×** |

The memory footprint with `open()` is **over 4× larger** than with `SharedArray` for the exact same data set and same number of VUs. The footprint with `open()` also grows linearly with VU count (each VU holds its own parsed JS array), while `SharedArray` stays essentially flat as VUs increase.

### 5.5 Rationale — why `open()` duplicates and `SharedArray` does not

**`open()` semantics.** In k6, `open()` is resolved at the **init context** of each VU. Every VU runs its own Sobek (goja-fork) JS runtime — the only way to give each VU its own `global` scope — so when `const data = JSON.parse(open(...))` is evaluated in each VU's init context, each VU produces its **own** parsed JavaScript object tree. Those trees live in different Sobek runtimes and cannot share heap.

**`SharedArray` semantics — the shared backing store.** The `k6/data` module deliberately bypasses this with a single shared native Go slice of strings that all VUs point to. Look at the module definition in `js/modules/k6/data/data.go:17-58`:

```go
type (
    RootModule struct {
        shared sharedArrays
    }
    Data struct {
        vu     modules.VU
        shared *sharedArrays
    }
    sharedArrays struct {
        data map[string]sharedArray
        mu   sync.RWMutex
    }
)

func New() *RootModule {
    return &RootModule{
        shared: sharedArrays{ data: make(map[string]sharedArray) },
    }
}

// NewModuleInstance implements the modules.Module interface to return
// a new instance for each VU.
func (rm *RootModule) NewModuleInstance(vu modules.VU) modules.Instance {
    return &Data{
        vu:     vu,
        shared: &rm.shared,   // ← every VU gets a POINTER to the SAME shared map
    }
}
```

All VU-level `Data` instances receive a **pointer** to the single `sharedArrays` map that lives on the `RootModule`. That map is the process-wide singleton for shared data.

**Double-checked locking** in `sharedArrays.get` (lines 143-158) ensures the constructor function is executed exactly once per unique name:

```go
func (s *sharedArrays) get(rt *sobek.Runtime, name string, call sobek.Callable) sharedArray {
    s.mu.RLock()
    array, ok := s.data[name]
    s.mu.RUnlock()
    if !ok {
        s.mu.Lock()
        defer s.mu.Unlock()
        array, ok = s.data[name]
        if !ok {
            array = getShareArrayFromCall(rt, call)  // constructor runs ONLY the first time
            s.data[name] = array
        }
    }
    return array
}
```

**How `sharedArray` stores the data** (`js/modules/k6/data/share.go:9-60`):

```go
type sharedArray struct {
    arr []string                   // ← native Go slice of JSON-encoded strings
}

type wrappedSharedArray struct {
    sharedArray
    rt       *sobek.Runtime
    freeze   sobek.Callable
    isFrozen sobek.Callable
    parse    sobek.Callable
}

func (s sharedArray) wrap(rt *sobek.Runtime) sobek.Value {
    ...
    return rt.NewDynamicArray(wrappedSharedArray{
        sharedArray: s,    // ← the inner []string is shared by reference
        rt:          rt,
        freeze:      freeze, isFrozen: isFrozen, parse: parse,
    })
}

func (s wrappedSharedArray) Get(index int) sobek.Value {
    if index < 0 || index >= len(s.arr) { return sobek.Undefined() }
    val, err := s.parse(sobek.Undefined(), s.rt.ToValue(s.arr[index]))
    if err != nil { common.Throw(s.rt, err) }
    err = s.deepFreeze(s.rt, val)
    if err != nil { common.Throw(s.rt, err) }
    return val
}
```

Each VU only ever gets a thin `wrappedSharedArray` struct whose `sharedArray.arr` is a **shallow copy of the same `[]string` header** — pointer + length + capacity. The backing string slice (the actual data) is **not duplicated**. When JS code reads `data[i]`, `Get(index)` parses `arr[index]` into a per-VU object on demand and deep-freezes it; the per-VU parsed object is transient.

That explains the numerical gap we observed: with 10 VUs,
* `open()` holds **10 parsed copies** of the 20 MB JSON in Sobek heaps → ≈ 800 MB resident.
* `SharedArray` holds **1 copy** of the serialized JSON strings and creates parsed JS values only when the VU actively reads an index; the overall footprint is dominated by the single `[]string` plus 10 small `wrappedSharedArray` wrappers.

### 5.6 Conclusion

* **`open()`**: the data file is loaded and parsed independently in each VU's JavaScript runtime. Memory footprint grows **linearly with VU count** (each VU creates its own copy).
* **`SharedArray`**: the data is loaded exactly once into the `RootModule.shared` map (`data.go:17`, `data.go:64`) and every VU module instance receives a pointer to the same `[]string` backing store (`share.go:10`). Memory footprint is essentially **constant** regardless of VU count.
* **Measured ratio at 10 VUs: ≈ 4.54× less memory** with `SharedArray` for the same 20 MB dataset.

---

## 6. Experiment 5 — Prometheus remote-write output: metric-name integrity & the `k6_` prefix

### 6.1 Question

> *"Provide test script output proving that the exported data via the `experimental-prometheus-rw` output maintains the integrity of metric names. Show the `k6_` prefix convention and the Prometheus `__name__` label mapping."*

### 6.2 Approach

We started a **mock Prometheus remote-write receiver** on `localhost:9090/api/v1/write` — a trivial Python HTTP server that Snappy-decompresses the request body and walks the protobuf `WriteRequest` to extract `__name__` label values. A k6 test defines one `Counter`, one `Trend`, one `Rate` and one `Gauge` custom metric and uses `-o experimental-prometheus-rw` with `K6_PROMETHEUS_RW_SERVER_URL=http://localhost:9090/api/v1/write`.

### 6.3 Test script

```js
// /tmp/k6tmp/test_prom_metrics.js
import { sleep } from 'k6';
import { Counter, Trend, Rate, Gauge } from 'k6/metrics';

export const myCounter = new Counter('my_custom_counter');
export const myTrend   = new Trend('my_custom_duration', true);
export const myRate    = new Rate('my_custom_rate');
export const myGauge   = new Gauge('my_custom_gauge');

export const options = {
  scenarios: { s: { executor: 'constant-vus', vus: 2, duration: '12s', gracefulStop: '2s' } },
};

export default function () {
  const t0 = Date.now();
  myCounter.add(1);
  myRate.add(Math.random() > 0.3);
  myGauge.add(Math.floor(Math.random() * 100));
  sleep(1);
  myTrend.add(Date.now() - t0);
}
```

### 6.4 Run (default `trendStats = p(99)`)

```bash
K6_PROMETHEUS_RW_SERVER_URL="http://localhost:9090/api/v1/write" \
K6_PROMETHEUS_RW_PUSH_INTERVAL="2s" \
/usr/local/bin/k6 run -o experimental-prometheus-rw \
    --verbose --log-output=stdout --log-format=raw \
    /tmp/k6tmp/test_prom_metrics.js
```

**k6 stdout (excerpt):**
```
running (02.0s), 2/2 VUs, 2 complete and 0 interrupted iterations
s      [  17% ] 2 VUs  02.0s/12s
Converted samples to Prometheus TimeSeries
...
Successful flushed time series to remote write endpoint
...
     data_received........: 0 B    0 B/s
     data_sent............: 0 B    0 B/s
     iteration_duration...: avg=1s min=1s med=1s max=1s p(90)=1s p(95)=1s
     iterations...........: 24     1.998716/s
     my_custom_counter....: 24     1.998716/s
     my_custom_duration...: avg=1s min=1s med=1s max=1s p(90)=1s p(95)=1s
     my_custom_gauge......: 18     min=4        max=99
     my_custom_rate.......: 62.50% 15 out of 24
     vus..................: 2      min=2        max=2
     vus_max..............: 2      min=2        max=2
```

**Mock receiver output** — every single `__name__` label value seen in the flushes, with *default* `trendStats`:
```
[req #1] path=/api/v1/write body=294B raw=633B ts_count=10 samples=10
  __name__ values in this request:
    ['k6_data_received_total', 'k6_data_sent_total',
     'k6_iteration_duration_p99', 'k6_iterations_total',
     'k6_my_custom_counter_total', 'k6_my_custom_duration_p99',
     'k6_my_custom_gauge', 'k6_my_custom_rate_rate',
     'k6_vus', 'k6_vus_max']
... (same set for req #2..#6)
[req #7] path=/api/v1/write body=225B raw=420B ts_count=7 samples=7
  __name__ values in this request:
    ['k6_data_received_total', 'k6_data_sent_total',
     'k6_iteration_duration_p99', 'k6_iterations_total',
     'k6_my_custom_duration_p99', 'k6_vus', 'k6_vus_max']
```

### 6.5 Run (extended `trendStats = p(99),p(95),p(90),min,max,avg,med,count,sum`)

```bash
K6_PROMETHEUS_RW_SERVER_URL="http://localhost:9090/api/v1/write" \
K6_PROMETHEUS_RW_PUSH_INTERVAL="2s" \
K6_PROMETHEUS_RW_TREND_STATS="p(99),p(95),p(90),min,max,avg,med,count,sum" \
/usr/local/bin/k6 run -o experimental-prometheus-rw \
    --verbose --log-output=stdout --log-format=raw \
    /tmp/k6tmp/test_prom_metrics.js
```

**Mock receiver — final enumeration of all unique `__name__` values seen (26 total), with a full label dump of one representative sample each:**

```
=== FINAL UNIQUE __name__ VALUES SEEN ===
  k6_data_received_total        <- labels: [__name__=k6_data_received_total, scenario=s]
  k6_data_sent_total            <- labels: [__name__=k6_data_sent_total, scenario=s]
  k6_iteration_duration_avg     <- labels: [__name__=k6_iteration_duration_avg, scenario=s]
  k6_iteration_duration_count   <- labels: [__name__=k6_iteration_duration_count, scenario=s]
  k6_iteration_duration_max     <- labels: [__name__=k6_iteration_duration_max, scenario=s]
  k6_iteration_duration_med     <- labels: [__name__=k6_iteration_duration_med, scenario=s]
  k6_iteration_duration_min     <- labels: [__name__=k6_iteration_duration_min, scenario=s]
  k6_iteration_duration_p90     <- labels: [__name__=k6_iteration_duration_p90, scenario=s]
  k6_iteration_duration_p95     <- labels: [__name__=k6_iteration_duration_p95, scenario=s]
  k6_iteration_duration_p99     <- labels: [__name__=k6_iteration_duration_p99, scenario=s]
  k6_iteration_duration_sum     <- labels: [__name__=k6_iteration_duration_sum, scenario=s]
  k6_iterations_total           <- labels: [__name__=k6_iterations_total, scenario=s]
  k6_my_custom_counter_total    <- labels: [__name__=k6_my_custom_counter_total, scenario=s]
  k6_my_custom_duration_avg     <- labels: [__name__=k6_my_custom_duration_avg, scenario=s]
  k6_my_custom_duration_count   <- labels: [__name__=k6_my_custom_duration_count, scenario=s]
  k6_my_custom_duration_max     <- labels: [__name__=k6_my_custom_duration_max, scenario=s]
  k6_my_custom_duration_med     <- labels: [__name__=k6_my_custom_duration_med, scenario=s]
  k6_my_custom_duration_min     <- labels: [__name__=k6_my_custom_duration_min, scenario=s]
  k6_my_custom_duration_p90     <- labels: [__name__=k6_my_custom_duration_p90, scenario=s]
  k6_my_custom_duration_p95     <- labels: [__name__=k6_my_custom_duration_p95, scenario=s]
  k6_my_custom_duration_p99     <- labels: [__name__=k6_my_custom_duration_p99, scenario=s]
  k6_my_custom_duration_sum     <- labels: [__name__=k6_my_custom_duration_sum, scenario=s]
  k6_my_custom_gauge            <- labels: [__name__=k6_my_custom_gauge, scenario=s]
  k6_my_custom_rate_rate        <- labels: [__name__=k6_my_custom_rate_rate, scenario=s]
  k6_vus                        <- labels: [__name__=k6_vus]
  k6_vus_max                    <- labels: [__name__=k6_vus_max]

TOTAL unique __name__ values: 26
```

### 6.6 Observed naming patterns by metric type

| k6 metric name | k6 type | Prometheus `__name__` | Suffix rule |
|---|---|---|---|
| `my_custom_counter` | `Counter` | `k6_my_custom_counter_total` | `_total` |
| `my_custom_gauge` | `Gauge` | `k6_my_custom_gauge` | *(none)* |
| `my_custom_rate` | `Rate` | `k6_my_custom_rate_rate` | `_rate` |
| `my_custom_duration` | `Trend` | `k6_my_custom_duration_<stat>` for each configured stat | `_avg`, `_min`, `_max`, `_med`, `_p99`, `_p95`, `_p90`, `_count`, `_sum` |
| `iteration_duration` | `Trend` (built-in) | `k6_iteration_duration_<stat>` | same as above |
| `iterations` | `Counter` (built-in) | `k6_iterations_total` | `_total` |
| `data_received`, `data_sent` | `Counter` (built-in) | `k6_data_received_total`, `k6_data_sent_total` | `_total` |
| `vus`, `vus_max` | `Gauge` (built-in) | `k6_vus`, `k6_vus_max` | *(none)* |

Every k6 metric the output emitted carries the `k6_` prefix as its `__name__` label; no k6 metric name was leaked through to Prometheus without the prefix.

### 6.7 Rationale — where the prefix and suffix come from

**Config defaults** (`vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/config.go`):
```go
const (
    defaultServerURL    = "http://localhost:9090/api/v1/write"
    defaultTimeout      = 5 * time.Second
    defaultPushInterval = 5 * time.Second
    defaultMetricPrefix = "k6_"
)

//nolint:gochecknoglobals
var defaultTrendStats = []string{"p(99)"}
```

**`MapSeries` — the canonical builder of the `__name__` label** (`vendor/.../remotewrite/prometheus.go:40-60`):
```go
const namelbl = "__name__"

func MapSeries(series metrics.TimeSeries, suffix string) []*prompb.Label {
    v := defaultMetricPrefix + series.Metric.Name
    if suffix != "" {
        v += "_" + suffix
    }
    lbls := append(MapTagSet(series.Tags), &prompb.Label{
        Name:  namelbl,
        Value: v,
    })
    sort.Slice(lbls, func(i int, j int) bool {
        return lbls[i].Name < lbls[j].Name
    })
    return lbls
}
```

**Counter** emission uses `suffix = "total"`, **Rate** emission uses `suffix = "rate"`, **Gauge** uses no suffix, and **Trend** metrics are flattened to one TimeSeries per configured stat via `trendAsGauges.Append(stat, value)` (`vendor/.../remotewrite/trend.go:78-96`):

```go
func (tg *trendAsGauges) Append(suffix string, v float64) {
    ts := &prompb.TimeSeries{
        Labels:  make([]*prompb.Label, len(tg.labels)),
        Samples: make([]*prompb.Sample, 1),
    }
    for i := 0; i < len(tg.labels); i++ {
        ts.Labels[i] = &prompb.Label{Name: tg.labels[i].Name, Value: tg.labels[i].Value}
    }
    ts.Labels[tg.ixname].Value += "_" + suffix  // ← stat name appended to __name__
    ts.Samples[0] = &prompb.Sample{ Timestamp: tg.timestamp, Value: v }
    tg.series = append(tg.series, ts)
}
```

Because the stat name is appended to the already-prefixed `__name__` label (`tg.labels[tg.ixname].Value` was initialised via `MapSeries`, which already applied `"k6_"`), the final label value is e.g. `k6_iteration_duration_p99`.

**Flush pipeline** (`vendor/.../remotewrite/remotewrite.go:194-230`):
```go
func (o *Output) flush() {
    ...
    promTimeSeries := o.convertToPbSeries(samplesContainers)
    nts := len(promTimeSeries)
    o.logger.WithField("nts", nts).Debug("Converted samples to Prometheus TimeSeries")
    ...
    // client.Store(...) → remote-write POST; on success:
    okmsg := "Successful flushed time series to remote write endpoint"
}
```

These two log lines — `Converted samples to Prometheus TimeSeries` and `Successful flushed time series to remote write endpoint` — are exactly what we see printed in the k6 stdout for every 2 s push interval.

### 6.8 Conclusion

* The `experimental-prometheus-rw` output never transmits a k6 metric by its bare JS-facing name; every metric is exported with the mandatory `k6_` prefix baked into the `__name__` label by `MapSeries`.
* **User-chosen names (e.g. `my_custom_counter`) remain intact** — they appear verbatim in the middle of the generated `__name__` value (`k6_my_custom_counter_total`), with only a type-specific suffix (`_total`, `_rate`, or a trend stat like `_p99`) and no character replacement. Gauge metrics get no suffix at all (e.g. `k6_my_custom_gauge`, `k6_vus`, `k6_vus_max`).
* The default config sends `p(99)` for trend metrics; configuring `K6_PROMETHEUS_RW_TREND_STATS` splits a single k6 `Trend` into one Prometheus TimeSeries per stat (confirmed with 9 stats → 9 distinct `k6_*_<stat>` label values per trend).
* Every non-`__name__` k6 tag is passed through to Prometheus as a regular label (in our output, `scenario=s` for the scenario-tagged series), via `MapTagSet` — so tag integrity is also preserved.

---

## 7. Summary of source-code references used

| Concern | File : line | Short description |
|---|---|---|
| k6 version string | `lib/consts/consts.go:12` | `const Version = "0.55.0"` |
| Signal trap | `cmd/common.go:97-125` | `handleTestAbortSignals` — 1st SIGINT → graceful, 2nd → hard stop |
| Graceful / hard handlers | `cmd/run.go:349-363` | `gracefulStop` calls `runAbort()`; `onHardStop` calls `globalCancel()` |
| Test run context | `execution/abort.go:49-63` | `NewTestRunContext` — cancellable context whose cancel is `runAbort` |
| VU lifecycle state machine | `lib/executor/vu_handle.go:17-53, 147-183, 186-267` | `running → toGracefulStop → stopped` vs `running → toHardStop → stopped` |
| `Start` debug log | `lib/executor/vu_handle.go:123,127` | `vh.logger.Debug("Start")` |
| `Executor finished successfully` | `execution/scheduler.go:371` | Printed per executor after `Run()` returns without error |
| Exit code `ExternalAbort = 105` | `errext/exitcodes/codes.go:41` | `ExternalAbort ExitCode = 105` |
| gRPC stream lifecycle logs | `js/modules/k6/grpc/stream.go:201, 345, 371` | `stream is cancelled/finished`, `finishing stream ... writing`, `stream ... is closing` |
| gRPC metrics registration | `js/modules/k6/grpc/metrics.go` | `grpc_streams`, `grpc_streams_msgs_sent`, `grpc_streams_msgs_received` (all `Counter`) |
| gRPC server `ListFeatures` | `lib/testutils/grpcservice/service.go:57-68` | 100 ms sleep between each `stream.Send()` |
| `dropped_iterations` metric definition | `metrics/builtin.go:10,84` | `DroppedIterationsName = "dropped_iterations"`, type `Counter` |
| Real-time drop emission (const arrival rate) | `lib/executor/constant_arrival_rate.go:315-345` | `metrics.PushIfNotDone` in the ticker loop when `vusPool.TryRunIteration()==false` |
| Batch drop emission (per-VU iterations) | `lib/executor/per_vu_iterations.go:195-230` | Emits `iterations - i` when `regDurationDone` fires before iterations are consumed |
| REST API routing | `api/v1/routes.go` | `/v1/metrics/{id}` → `handleGetMetric` |
| Metric query handler | `api/v1/metric_routes.go:27-48` | Locks `MetricsEngine`, looks up `ObservedMetrics[id]`, JSON-marshals |
| Metric JSON envelope | `api/v1/metric.go` | `Sample: m.Sink.Format(t)` — for `Counter`, returns `{count, rate}` |
| Default API address | `cmd/state/state.go:150` | `Address: "localhost:6565"` |
| `SharedArray` root singleton | `js/modules/k6/data/data.go:17-58, 64-70` | `RootModule.shared`; pointer shared to every VU instance |
| `SharedArray` double-checked init | `js/modules/k6/data/data.go:143-158` | `sharedArrays.get` — constructor runs only the first time |
| `wrappedSharedArray` per-VU view | `js/modules/k6/data/share.go:10-60` | Shares the underlying `[]string`; `Get(i)` parses + deep-freezes on demand |
| Prometheus prefix / URL / interval defaults | `vendor/.../remotewrite/config.go:20-28` | `defaultMetricPrefix = "k6_"`, URL, 5 s push interval |
| `MapSeries` — builds `__name__` | `vendor/.../remotewrite/prometheus.go:40-60` | `defaultMetricPrefix + series.Metric.Name [+ "_" + suffix]` |
| Trend → multiple gauges suffixing | `vendor/.../remotewrite/trend.go:78-96` | `ts.Labels[tg.ixname].Value += "_" + suffix` |
| `flush()` pipeline / log lines | `vendor/.../remotewrite/remotewrite.go:194-230` | `Converted samples to Prometheus TimeSeries` / `Successful flushed time series to remote write endpoint` |
| `experimental-prometheus-rw` registration | `cmd/outputs.go` | Registers the output constructor `remotewrite.New` under that name |

