# k6 v0.55.0 Internal Orchestration Investigation

**Commit:** `ddc3b0b1d2` | **Go:** `1.21.13` | **Platform:** `linux/amd64`
**Date of Investigation:** 2026-04-13
**Binary Version:** `k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)`

---

## Table of Contents

1. [Question 1 — VU Management under SIGINT (Ramping Executor)](#question-1--vu-management-under-sigint-ramping-executor)
2. [Question 2 — gRPC Server Streaming Interruption](#question-2--grpc-server-streaming-interruption)
3. [Question 3 — Dropped Iterations via API](#question-3--dropped-iterations-via-api)
4. [Question 4 — SharedArray Data Sharing Behavior](#question-4--sharedarray-data-sharing-behavior)
5. [Question 5 — Prometheus Output Metric Name Integrity](#question-5--prometheus-output-metric-name-integrity)
6. [Summary and Conclusions](#summary-and-conclusions)

---

## Question 1 — VU Management under SIGINT (Ramping Executor)

**Question:** What exact log messages are emitted when a `ramping-vus` executor configured with at least 5 VUs receives a SIGINT signal? Are active VUs allowed to finish their current iteration or are they terminated mid-execution?

### Source Code Analysis

The complete signal handling chain from SIGINT reception to VU shutdown traverses six key components:

#### 1. Signal Trap Registration — `cmd/common.go:96-129`

The `handleTestAbortSignals()` function (line 97) is the entry point for signal handling. It creates a buffered channel `sigC` with capacity 2 (line 99) and registers listeners for `os.Interrupt`, `syscall.SIGINT`, and `syscall.SIGTERM` via `gs.SignalNotify(sigC, ...)` (line 101).

The goroutine (lines 103-122) implements a two-stage signal handler:

- **First signal** (line 105-106): Calls `gracefulStopHandler(sig)`, which allows VUs to finish their current iterations.
- **Second signal** (line 112-118): Calls `onHardStop(sig)` if non-nil, then immediately calls `gs.OSExit(int(exitcodes.ExternalAbort))` — the process exits without waiting.

```go
// cmd/common.go:96-129
func handleTestAbortSignals(gs *state.GlobalState, gracefulStopHandler, onHardStop func(os.Signal)) (stop func()) {
    gs.Logger.Debug("Trapping interrupt signals so k6 can handle them gracefully...")
    sigC := make(chan os.Signal, 2)
    done := make(chan struct{})
    gs.SignalNotify(sigC, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)

    go func() {
        select {
        case sig := <-sigC:
            gracefulStopHandler(sig)
        case <-done:
            return
        }
        select {
        case sig := <-sigC:
            if onHardStop != nil {
                onHardStop(sig)
            }
            gs.OSExit(int(exitcodes.ExternalAbort))
        case <-done:
            return
        }
    }()
    // ...
}
```

#### 2. Graceful Stop Closure — `cmd/run.go:349-357`

When the first SIGINT arrives, the `gracefulStop` closure fires:

```go
// cmd/run.go:349-357
gracefulStop := func(sig os.Signal) {
    logger.WithField("sig", sig).Debug("Stopping k6 in response to signal...")
    runAbort(errext.WithAbortReasonIfNone(
        errext.WithExitCodeIfNone(
            fmt.Errorf("test run was aborted because k6 received a '%s' signal", sig), exitcodes.ExternalAbort,
        ), errext.AbortedByUser,
    ))
    lingerCancel()
}
```

This emits `"Stopping k6 in response to signal..."` at **Debug** level, then calls `runAbort()` which propagates the cancellation through the context tree.

#### 3. Hard Stop Closure — `cmd/run.go:359-361`

If a second SIGINT arrives, the `onHardStop` closure fires:

```go
// cmd/run.go:359-361
onHardStop := func(sig os.Signal) {
    logger.WithField("sig", sig).Error("Aborting k6 in response to signal")
    globalCancel()
}
```

This emits `"Aborting k6 in response to signal"` at **Error** level, followed by immediate process exit.

#### 4. Abort Controller — `execution/abort.go:24-36`

The `testAbortController.abort()` method (line 24) acquires a lock, stores the first error reason, and calls `cancel()` on the `runCtx` context. Only the first abort reason is kept; subsequent abort attempts are logged at Debug level:

```go
// execution/abort.go:24-36
func (tac *testAbortController) abort(err error) {
    tac.lock.Lock()
    defer tac.lock.Unlock()
    if tac.reason != nil {
        tac.logger.Debugf(
            "test abort with reason '%s' was attempted when the test was already aborted due to '%s'",
            err.Error(), tac.reason.Error(),
        )
        return
    }
    tac.reason = err
    tac.cancel()
}
```

#### 5. Scheduler Interrupt Detection — `execution/scheduler.go:425-430`

The deferred function in `Run()` checks for interruption and logs the result:

```go
// execution/scheduler.go:425-430
defer func() {
    if interruptErr := GetCancelReasonIfTestAborted(runCtx); interruptErr != nil {
        logger.Debugf("The test run was interrupted, returning '%s' instead of '%s'", interruptErr, runErr)
        e.state.SetExecutionStatus(lib.ExecutionStatusInterrupted)
        runErr = interruptErr
    }
    // ...
}()
```

This emits: `"The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'"` at **Debug** level.

#### 6. VU State Machine — `lib/executor/vu_handle.go:13-264`

The `vuHandle` type implements a five-state state machine (line 16-22):

| State | Meaning |
|-------|---------|
| `stopped` | VU is idle, not running iterations |
| `starting` | VU is being initialized/activated |
| `running` | VU is actively executing iterations |
| `toGracefulStop` | VU should finish current iteration, then stop |
| `toHardStop` | VU's context is cancelled immediately |

**`gracefulStop()` (line 147-163):** Transitions `running` → `toGracefulStop`. Does **NOT** cancel the VU's context. The current iteration can complete.

```go
// lib/executor/vu_handle.go:147-163
func (vh *vuHandle) gracefulStop() {
    vh.mutex.Lock()
    defer vh.mutex.Unlock()
    switch vh.state {
    case toGracefulStop, toHardStop, stopped:
        return
    case starting:
        vh.cancel()
        vh.ctx, vh.cancel = context.WithCancel(vh.parentCtx)
        vh.changeState(stopped)
    case running:
        vh.changeState(toGracefulStop)
    }
    vh.logger.Debug("Graceful stop")
    vh.canStartIter = make(chan struct{})
}
```

**`hardStop()` (line 165-181):** Transitions to `toHardStop` AND calls `vh.cancel()` to cancel the VU's context, interrupting any in-progress iteration.

```go
// lib/executor/vu_handle.go:165-181
func (vh *vuHandle) hardStop() {
    vh.mutex.Lock()
    defer vh.mutex.Unlock()
    switch vh.state {
    case toHardStop, stopped:
        return
    case starting:
        vh.changeState(stopped)
    case running, toGracefulStop:
        vh.changeState(toHardStop)
    }
    vh.logger.Debug("Hard stop")
    vh.cancel()
    vh.ctx, vh.cancel = context.WithCancel(vh.parentCtx)
    vh.canStartIter = make(chan struct{})
}
```

**`runLoopsIfPossible()` (line 183-264):** The main VU loop. When in `running` state (line 205), it calls `runIter(ctx, vu)`. When the executor context is done (line 212), it returns immediately. When in `toGracefulStop` (line 223), it cancels the VU-level context and transitions to `stopped`.

#### 7. Iteration Accounting — `lib/executor/helpers.go:104-141`

The `getIterationRunner()` function determines whether a completed iteration is counted as "full" or "interrupted":

```go
// lib/executor/helpers.go:104-141
func getIterationRunner(
    executionState *lib.ExecutionState, logger *logrus.Entry,
) func(context.Context, lib.ActiveVU) bool {
    return func(ctx context.Context, vu lib.ActiveVU) bool {
        err := vu.RunOnce()
        select {
        case <-ctx.Done():
            executionState.AddInterruptedIterations(1)
            return false
        default:
            if err != nil {
                // ... error handling ...
            }
            executionState.AddFullIterations(1)
            return true
        }
    }
}
```

After `vu.RunOnce()` returns, if `ctx.Done()` is closed, the iteration is counted as **interrupted** (line 114-118). Otherwise, it's counted as **full** (line 137).

#### 8. Progress Reporting — `execution/scheduler.go:142-160`

The `getRunStats()` method formats the final progress bar:

```go
// execution/scheduler.go:155-158
return fmt.Sprintf(
    "%s, "+vusFmt+"/"+vusFmt+" VUs, %d complete and %d interrupted iterations",
    status, e.state.GetCurrentlyActiveVUsCount(), e.state.GetInitializedVUsCount(),
    e.state.GetFullIterationCount(), e.state.GetPartialIterationCount(),
)
```

#### 9. Handler Strategies — `lib/executor/ramping_vus.go:668-690`

Two strategies control VU transitions during ramping:

- **`maxAllowedVUsHandlerStrategy()` (line 668-677):** Calls `hardStop()` on excess VUs when the graceful step count decreases. This cancels their contexts.
- **`scheduledVUsHandlerStrategy()` (line 679-690):** Calls `start()` to ramp up, `gracefulStop()` to ramp down. Graceful stops allow the current iteration to finish.

#### 10. Waiter Function — `lib/executor/ramping_vus.go:697-712`

The `waiter()` function in `iterateSteps` detects context cancellation:

```go
// lib/executor/ramping_vus.go:697-712
func waiter(ctx context.Context, start time.Time) func(offset time.Duration) bool {
    timer := time.NewTimer(time.Hour * 24)
    return func(offset time.Duration) bool {
        diff := offset - time.Since(start)
        if diff > 0 {
            timer.Reset(diff)
            select {
            case <-ctx.Done():
                return true // exit if context is cancelled
            case <-timer.C:
            }
        }
        return false
    }
}
```

#### 11. Default Graceful Periods — `lib/executor/base_config.go:20` and `lib/executor/ramping_vus.go:52`

```go
// lib/executor/base_config.go:20
var DefaultGracefulStopValue = 30 * time.Second

// lib/executor/ramping_vus.go:52
GracefulRampDown: types.NewNullDuration(30*time.Second, false),
```

### Behavioral Answer

**On first SIGINT, active VUs are allowed to finish their current iteration (graceful behavior).** The precise sequence is:

1. The `gracefulStop` closure fires, logging `"Stopping k6 in response to signal..."` at Debug level.
2. `runAbort()` cancels the `runCtx` context, which propagates to the executor context.
3. The `waiter()` function in `iterateSteps()` detects `ctx.Done()` and exits the stepping loop.
4. The `maxAllowedVUsHandlerStrategy` calls `hardStop()` on VUs that exceed the graceful step count, cancelling their individual contexts.
5. VUs currently mid-iteration in `runLoopsIfPossible()`:
   - **If the VU's context is cancelled before `vu.RunOnce()` returns:** The iteration is counted as **interrupted** (via `getIterationRunner`, line 114-118 of helpers.go).
   - **If the VU completes `vu.RunOnce()` before context cancellation:** The iteration is counted as **full** (line 137 of helpers.go).
6. The `scheduledVUsHandlerStrategy` calls `gracefulStop()` on VUs that are ramping down, transitioning them to `toGracefulStop`. This allows the current iteration to complete but prevents new iterations from starting.
7. The scheduler's deferred function detects the interruption and logs `"The test run was interrupted, returning '...' instead of '...'"`.
8. The final summary reports both complete and interrupted iterations.

**Conclusion:** VUs that are in the middle of an iteration when SIGINT arrives are allowed to finish that iteration, with the outcome determined by a race between context cancellation and `RunOnce()` completion. VUs whose iterations finish before the context is cancelled are counted as "complete," while those whose context is cancelled before `RunOnce()` returns are counted as "interrupted."

### Runtime Evidence

**Test Script Used:**

```javascript
import { sleep } from 'k6';

export const options = {
  scenarios: {
    ramping_test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '3s', target: 5 },
        { duration: '30s', target: 5 },
      ],
      gracefulRampDown: '5s',
      gracefulStop: '5s',
    },
  },
};

export default function () {
  console.log(`VU ${__VU} iteration ${__ITER} starting`);
  sleep(2);
  console.log(`VU ${__VU} iteration ${__ITER} completed`);
}
```

**Execution Command:**

```
/tmp/k6 run --log-output=stdout --log-format=raw -v /tmp/exp1_sigint.js
# SIGINT sent via `kill -INT $PID` after 8 seconds (5 VUs at steady state)
```

**Key Log Output (verbatim):**

```
running (07.0s), 5/5 VUs, 10 complete and 0 interrupted iterations
ramping_test   [  21% ] 5/5 VUs  07.0s/33.0s
VU 1 iteration 1 completed
VU 1 iteration 2 starting
VU 3 iteration 2 completed
VU 3 iteration 3 starting
VU 5 iteration 2 completed
VU 5 iteration 3 starting
Stopping k6 in response to signal...
Metrics emission of VUs and VUsMax metrics stopped
VU 4 iteration 3 completed
VU 2 iteration 2 completed
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
     iterations...........: 13  1.631279/s
     vus..................: 5   min=1      max=5
     vus_max..............: 5   min=5      max=5


running (08.0s), 0/5 VUs, 13 complete and 5 interrupted iterations
ramping_test ✗ [  24% ] 3/5 VUs  08.0s/33.0s
```

**Key Evidence Points:**

1. `"Stopping k6 in response to signal..."` — The first log message emitted upon SIGINT reception (from `cmd/run.go:350`).
2. `"VU 4 iteration 3 completed"` and `"VU 2 iteration 2 completed"` — Two VUs completed their iterations **after** SIGINT was received, proving that active VUs are allowed to finish.
3. VUs 1, 3, and 5 had just started new iterations (`iteration 2 starting`, `iteration 3 starting`) but no corresponding "completed" messages — these were **interrupted**.
4. `"The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'"` — The scheduler's interrupt detection message (from `execution/scheduler.go:427`).
5. **`13 complete and 5 interrupted iterations`** — The final summary showing both full and interrupted iteration counts.

---

## Question 2 — gRPC Server Streaming Interruption

**Question:** What are the exact log entries produced when a gRPC server streaming test with `gracefulRampDown: '30ms'` is interrupted? What is the specific value of `grpc_streams_msgs_received` from the final metrics summary?

### Source Code Analysis

#### 1. gRPC Metric Registration — `js/modules/k6/grpc/metrics.go:12-30`

Three streaming metrics are registered:

```go
// js/modules/k6/grpc/metrics.go:17-27
if m.Streams, err = registry.NewMetric("grpc_streams", metrics.Counter); err != nil {
    return nil, err
}
if m.StreamsMessagesSent, err = registry.NewMetric("grpc_streams_msgs_sent", metrics.Counter); err != nil {
    return nil, err
}
if m.StreamsMessagesReceived, err = registry.NewMetric("grpc_streams_msgs_received", metrics.Counter); err != nil {
    return nil, err
}
```

All three are `Counter` type metrics, meaning they accumulate monotonically.

#### 2. Stream Initialization — `js/modules/k6/grpc/stream.go:81-117`

The `beginStream()` method creates the gRPC stream and increments `grpc_streams` by 1 (lines 104-112):

```go
// js/modules/k6/grpc/stream.go:104-112
metrics.PushIfNotDone(s.vu.Context(), s.vu.State().Samples, metrics.Sample{
    TimeSeries: metrics.TimeSeries{
        Metric: s.instanceMetrics.Streams,
        Tags:   s.tagsAndMeta.Tags,
    },
    Time:     time.Now(),
    Metadata: s.tagsAndMeta.Metadata,
    Value:    1,
})
```

After the metric push, `go s.loop()` is started (line 114) to monitor the stream lifecycle.

#### 3. Message Reception Counting — `js/modules/k6/grpc/stream.go:149-159`

Each received message increments `grpc_streams_msgs_received` by 1:

```go
// js/modules/k6/grpc/stream.go:149-159
func (s *stream) queueMessage(msg interface{}) {
    now := time.Now()
    metrics.PushIfNotDone(s.vu.Context(), s.vu.State().Samples, metrics.Sample{
        TimeSeries: metrics.TimeSeries{
            Metric: s.instanceMetrics.StreamsMessagesReceived,
            Tags:   s.tagsAndMeta.Tags,
        },
        Time:     now,
        Metadata: s.tagsAndMeta.Metadata,
        Value:    1,
    })
    // ... queue for event dispatch ...
}
```

#### 4. Stream Loop & Context Cancellation — `js/modules/k6/grpc/stream.go:119-147`

The `loop()` method monitors VU context cancellation:

```go
// js/modules/k6/grpc/stream.go:119-147
func (s *stream) loop() {
    ctx := s.vu.Context()
    wg := new(sync.WaitGroup)
    defer func() {
        wg.Wait()
        s.tq.Close()
    }()
    wg.Add(2)
    go s.readData(wg)
    go s.writeData(wg)

    ctxDone := ctx.Done()
    for {
        select {
        case <-ctxDone:
            // VU is shutting down during an interrupt
            // stream events will not be forwarded to the VU
            s.tq.Queue(func() error {
                return s.closeWithError(nil)
            })
            return
        case <-s.done:
            return
        }
    }
}
```

When the VU context is cancelled (line 136), the stream is closed with `nil` error. This triggers the gRPC stream to report `ErrCanceled`.

#### 5. Data Reading & Regular Closing — `js/modules/k6/grpc/stream.go:183-218`

The `readData()` method reads messages in a loop. On `io.EOF` or `grpcext.ErrCanceled` (detected by `isRegularClosing()` at line 216-218), it logs `"stream is cancelled/finished"` and queues a close:

```go
// js/modules/k6/grpc/stream.go:200-207
if isRegularClosing(err) {
    s.logger.WithError(err).Debug("stream is cancelled/finished")
    s.tq.Queue(func() error {
        return s.closeWithError(err)
    })
    return
}
```

```go
// js/modules/k6/grpc/stream.go:216-218
func isRegularClosing(err error) bool {
    return errors.Is(err, io.EOF) || errors.Is(err, grpcext.ErrCanceled)
}
```

### Behavioral Answer

With `gracefulRampDown: '30ms'`, VUs are given only 30 milliseconds to finish their current iteration before being hard-stopped. The stream lifecycle is:

1. Stream opens → `grpc_streams` incremented by 1.
2. Server sends messages → each received message increments `grpc_streams_msgs_received` by 1.
3. On VU context cancellation (via `gracefulStop` → `hardStop` sequence within 30ms), the `loop()` detects `ctx.Done()` and queues `closeWithError(nil)`.
4. The `readData()` goroutine detects `grpcext.ErrCanceled` and logs `"stream is cancelled/finished"`.
5. The stream close event fires, triggering the `error` and `end` event listeners in the JS runtime.

The exact value of `grpc_streams_msgs_received` depends on how many messages each stream receives before interruption. In the test, the server sends up to ~29 features per stream (those features within the bounding rectangle, each with a 100ms delay between sends). Interrupted streams receive fewer messages.

### Runtime Evidence

**Test Script Used:**

```javascript
import grpc from 'k6/net/grpc';
import { sleep } from 'k6';

const client = new grpc.Client();

export const options = {
  scenarios: {
    grpc_stream_test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2s', target: 3 },
        { duration: '5s', target: 3 },
      ],
      gracefulRampDown: '30ms',
      gracefulStop: '2s',
    },
  },
};

export default function () {
  client.connect('localhost:10000', { plaintext: true, reflect: true });

  const stream = new grpc.Stream(client, 'main.FeatureExplorer/ListFeatures');

  let msgCount = 0;
  stream.on('data', (data) => { msgCount++; });
  stream.on('end', () => { console.log(`Stream ended with ${msgCount} messages`); });
  stream.on('error', (err) => { console.log(`Stream error: ${err}`); });

  stream.write({
    lo: { latitude: 400000000, longitude: -750000000 },
    hi: { latitude: 420000000, longitude: -730000000 },
  });

  sleep(3);
  client.close();
}
```

**gRPC Server:** Standalone server built from the repository's own `lib/testutils/grpcservice` package, using `LoadFeatures("")` to load the 100 embedded features. The `ListFeatures` RPC sends matching features with a 100ms delay between each.

**Execution Command:**

```
/tmp/k6 run --log-output=stdout --log-format=raw -v /tmp/exp2_grpc.js
# SIGINT sent via `kill -INT $PID` after 6 seconds
```

**Key Log Output (verbatim):**

```
stream is cancelled/finished
stream /main.FeatureExplorer/ListFeatures is closing
Stream error: code: 2, message: canceled by client (k6)
Stream ended with 29 messages

Stopping k6 in response to signal...
Metrics emission of VUs and VUsMax metrics stopped
stream is cancelled/finished
stream is cancelled/finished
stream is cancelled/finished
stream /main.FeatureExplorer/ListFeatures is closing
stream /main.FeatureExplorer/ListFeatures is closing
Stream error: code: 2, message: canceled by client (k6)
Stream error: code: 2, message: canceled by client (k6)
Stream ended with 9 messages
Stream ended with 16 messages
stream /main.FeatureExplorer/ListFeatures is closing
Stream error: code: 2, message: canceled by client (k6)
Stream ended with 22 messages
Executor finished successfully
The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'
```

**Final Metrics Summary:**

```
     grpc_streams.................: 6      1.004405/s
     grpc_streams_msgs_received...: 134    22.431722/s
     grpc_streams_msgs_sent.......: 6      1.004405/s
     iteration_duration...........: avg=3s min=3s med=3s max=3s p(90)=3s p(95)=3s
     iterations...................: 3      0.502203/s
     vus..........................: 3      min=1       max=3
     vus_max......................: 3      min=3       max=3

running (6.0s), 0/3 VUs, 3 complete and 3 interrupted iterations
```

**Key Evidence Points:**

1. **`grpc_streams_msgs_received: 134`** — The total count of messages received across all 6 streams.
2. **`grpc_streams: 6`** — Six streams were opened (3 VUs × 2 iterations for completed VUs, plus partial streams for interrupted VUs).
3. **`"stream is cancelled/finished"`** — The log message from `readData()` (stream.go:201) when the stream detects `grpcext.ErrCanceled`.
4. **`"Stream error: code: 2, message: canceled by client (k6)"`** — The error event forwarded to JS, indicating gRPC status code `CANCELLED` (code 2).
5. Interrupted streams received varying numbers of messages (9, 16, 22) depending on when their context was cancelled, while completed streams received the full 29 messages.

---

## Question 3 — Dropped Iterations via API

**Question:** What is the exact value of `dropped_iterations` when a test exceeds its maximum duration capacity? Provide runtime evidence from querying the k6 REST API.

### Source Code Analysis

#### 1. Metric Definition — `metrics/builtin.go:10,84`

The `dropped_iterations` metric is defined as a built-in Counter:

```go
// metrics/builtin.go:10
DroppedIterationsName = "dropped_iterations"

// metrics/builtin.go:84
DroppedIterations: registry.MustNewMetric(DroppedIterationsName, Counter),
```

#### 2. Dropped Iteration Emission — `lib/executor/constant_arrival_rate.go:324-355`

In the `constant-arrival-rate` executor's `Run()` method, when a scheduled iteration cannot be started because all VUs are busy, a sample with value 1 is pushed:

```go
// lib/executor/constant_arrival_rate.go:324-346
droppedIterationMetric := car.executionState.Test.BuiltinMetrics.DroppedIterations
// ...
for li, gi := 0, start; ; li, gi = li+1, gi+offsets[li%len(offsets)] {
    // ...
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
        // ...
    }
}
```

Each dropped iteration increments the counter by 1. The warning `"Insufficient VUs, reached %d active VUs and cannot initialize more"` is logged once (line 352).

#### 3. REST API Route — `api/v1/routes.go:31-38`

The API route extracts the metric ID from the URL path:

```go
// api/v1/routes.go:31-38
mux.HandleFunc("/v1/metrics/", func(rw http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodGet {
        rw.WriteHeader(http.StatusMethodNotAllowed)
        return
    }
    id := r.URL.Path[len("/v1/metrics/"):]
    handleGetMetric(cs, rw, r, id)
})
```

#### 4. Metric Handler — `api/v1/metric_routes.go:27-49`

The `handleGetMetric()` function retrieves the metric by ID, locks the metrics engine, and returns a JSON:API envelope:

```go
// api/v1/metric_routes.go:27-49
func handleGetMetric(cs *ControlSurface, rw http.ResponseWriter, _ *http.Request, id string) {
    var t time.Duration
    if cs.Scheduler != nil {
        t = cs.Scheduler.GetState().GetCurrentTestRunDuration()
    }
    cs.MetricsEngine.MetricsLock.Lock()
    metric, ok := cs.MetricsEngine.ObservedMetrics[id]
    if !ok {
        cs.MetricsEngine.MetricsLock.Unlock()
        apiError(rw, "Not Found", "No metric with that ID was found", http.StatusNotFound)
        return
    }
    wrappedMetric := newMetricEnvelope(metric, t)
    cs.MetricsEngine.MetricsLock.Unlock()
    data, err := json.Marshal(wrappedMetric)
    // ...
}
```

#### 5. JSON:API Envelope — `api/v1/metric_jsonapi.go:24-28` and `api/v1/metric.go:74-82`

The response is wrapped in a JSON:API structure:

```go
// api/v1/metric_jsonapi.go:24-28
func newMetricEnvelope(m *metrics.Metric, t time.Duration) metricJSONAPI {
    return metricJSONAPI{
        Data: newMetricData(m, t),
    }
}

// api/v1/metric.go:74-82
func NewMetric(m *metrics.Metric, t time.Duration) Metric {
    return Metric{
        Name:     m.Name,
        Type:     NullMetricType{m.Type, true},
        Contains: NullValueType{m.Contains, true},
        Tainted:  m.Tainted,
        Sample:   m.Sink.Format(t),
    }
}
```

For a Counter type, `Sink.Format(t)` returns `{"count": N, "rate": R}` where `count` is the accumulated value and `rate` is `count / duration`.

### Behavioral Answer

With a `constant-arrival-rate` executor configured with `rate: 10` iterations/second, `maxVUs: 1`, and each iteration taking 1 second (`sleep(1)`), the single VU can only complete approximately 1 iteration per second. The remaining 9 scheduled iterations per second are dropped. Over the 5-second test duration, approximately 45 iterations are dropped (the exact count depends on timing precision).

The `dropped_iterations` counter is incremented by 1 for each dropped iteration (line 339-346 of `constant_arrival_rate.go`). When queried via the REST API at `GET /v1/metrics/dropped_iterations`, it returns the current accumulated count and rate at the time of the query.

### Runtime Evidence

**Test Script Used:**

```javascript
import { sleep } from 'k6';

export const options = {
  scenarios: {
    dropped_test: {
      executor: 'constant-arrival-rate',
      rate: 10,
      timeUnit: '1s',
      duration: '5s',
      preAllocatedVUs: 1,
      maxVUs: 1,
    },
  },
};

export default function () {
  sleep(1);
}
```

**Execution Command:**

```
/tmp/k6 run --address=localhost:6565 --log-output=stdout --log-format=raw -v /tmp/exp3_dropped.js
# API query sent via curl after 4 seconds
```

**API Query:**

```bash
curl -s http://localhost:6565/v1/metrics/dropped_iterations
```

**Exact API Response (at ~4 seconds into the test):**

```json
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":35,"rate":8.795577196780062}}}}
```

**Key Fields:**

| Field | Value | Meaning |
|-------|-------|---------|
| `data.type` | `"metrics"` | JSON:API resource type |
| `data.id` | `"dropped_iterations"` | Metric name from `metrics/builtin.go:10` |
| `data.attributes.type` | `"counter"` | Metric type (Counter) |
| `data.attributes.contains` | `"default"` | Value type |
| `data.attributes.sample.count` | `35` | Accumulated dropped iterations at query time |
| `data.attributes.sample.rate` | `8.796` | Dropped iterations per second |

**Final Summary Output:**

```
     dropped_iterations...: 46  8.67704/s
     iterations...........: 5   0.943157/s

running (05.3s), 0/1 VUs, 5 complete and 0 interrupted iterations
```

**Key Evidence Points:**

1. **API Response `count: 35`** — At 4 seconds into the test, 35 iterations had been dropped.
2. **Final Summary `dropped_iterations: 46`** — By the end of the 5-second test, 46 total iterations were dropped.
3. **`iterations: 5`** — Only 5 iterations completed (approximately 1 per second with the single VU).
4. **Total scheduled: ~50** (10/s × 5s). Completed: 5. Dropped: 46. This accounts for ~51 total (slight timing variance is expected).
5. The `"Insufficient VUs, reached 1 active VUs and cannot initialize more"` warning was emitted once (from `constant_arrival_rate.go:352`).

---

## Question 4 — SharedArray Data Sharing Behavior

**Question:** Does the memory footprint for files loaded via `SharedArray` remain constant with increasing VUs, or does each VU create its own copy?

### Source Code Analysis

#### 1. Module Architecture — `js/modules/k6/data/data.go:18-35`

The data module defines three types that form the sharing mechanism:

```go
// js/modules/k6/data/data.go:18-35
type (
    // RootModule is the global module instance that will create module instances for each VU.
    RootModule struct {
        shared sharedArrays
    }

    // Data represents an instance of the data module.
    Data struct {
        vu     modules.VU
        shared *sharedArrays  // POINTER to the same sharedArrays
    }

    sharedArrays struct {
        data map[string]sharedArray
        mu   sync.RWMutex
    }
)
```

**Critical observation:** `Data` (the per-VU instance) holds `shared *sharedArrays` — a **pointer** to the root module's `sharedArrays` struct. All VU instances share the same pointer.

#### 2. Single Instance Creation — `js/modules/k6/data/data.go:42-49`

```go
// js/modules/k6/data/data.go:42-49
func New() *RootModule {
    return &RootModule{
        shared: sharedArrays{
            data: make(map[string]sharedArray),
        },
    }
}
```

A single `RootModule` is created with one `sharedArrays` instance containing one map.

#### 3. Per-VU Instance — Pointer Sharing — `js/modules/k6/data/data.go:53-57`

```go
// js/modules/k6/data/data.go:53-57
func (rm *RootModule) NewModuleInstance(vu modules.VU) modules.Instance {
    return &Data{
        vu:     vu,
        shared: &rm.shared,  // ALL VUs receive a pointer to the SAME sharedArrays
    }
}
```

**This is the key mechanism.** Every VU receives `&rm.shared`, a pointer to the **same** `sharedArrays` struct. No copying occurs.

#### 4. Double-Checked Locking — `js/modules/k6/data/data.go:152-167`

The `get()` method ensures each named array is constructed exactly once:

```go
// js/modules/k6/data/data.go:152-167
func (s *sharedArrays) get(rt *sobek.Runtime, name string, call sobek.Callable) sharedArray {
    s.mu.RLock()
    array, ok := s.data[name]
    s.mu.RUnlock()
    if !ok {
        s.mu.Lock()
        defer s.mu.Unlock()
        array, ok = s.data[name]
        if !ok {
            array = getShareArrayFromCall(rt, call)
            s.data[name] = array
        }
    }
    return array
}
```

First attempt with `RLock()` (read lock, line 153-155). If not found, acquires `Lock()` (write lock, line 157), checks again (line 159), and only then initializes. This guarantees single initialization even with concurrent VUs.

#### 5. Shared Data Structure — `js/modules/k6/data/share.go:10-12`

```go
// js/modules/k6/data/share.go:10-12
type sharedArray struct {
    arr []string
}
```

The `sharedArray` holds `arr []string` — an array of JSON-serialized strings. This is the actual shared data. **The slice header (pointer, length, capacity) is copied when `sharedArray` is returned by value from `get()`, but the underlying array backing the slice is shared across all copies.**

#### 6. Per-VU Wrapper — `js/modules/k6/data/share.go:14-34`

```go
// js/modules/k6/data/share.go:14-21
type wrappedSharedArray struct {
    sharedArray                    // embeds the shared data
    rt       *sobek.Runtime       // per-VU runtime reference
    freeze   sobek.Callable       // per-VU Object.freeze reference
    isFrozen sobek.Callable       // per-VU Object.isFrozen reference
    parse    sobek.Callable       // per-VU JSON.parse reference
}

// js/modules/k6/data/share.go:23-34
func (s sharedArray) wrap(rt *sobek.Runtime) sobek.Value {
    freeze, _ := sobek.AssertFunction(rt.GlobalObject().Get("Object").ToObject(rt).Get("freeze"))
    isFrozen, _ := sobek.AssertFunction(rt.GlobalObject().Get("Object").ToObject(rt).Get("isFrozen"))
    parse, _ := sobek.AssertFunction(rt.GlobalObject().Get("JSON").ToObject(rt).Get("parse"))
    return rt.NewDynamicArray(wrappedSharedArray{
        sharedArray: s,
        rt:          rt,
        freeze:      freeze,
        isFrozen:    isFrozen,
        parse:       parse,
    })
}
```

Each VU creates a lightweight `wrappedSharedArray` that holds per-VU JS runtime references (for `JSON.parse`, `Object.freeze`, etc.) but **embeds the same `sharedArray`** — the underlying `[]string` data is NOT copied.

#### 7. Per-Access Deserialization — `js/modules/k6/data/share.go:44-58`

```go
// js/modules/k6/data/share.go:44-58
func (s wrappedSharedArray) Get(index int) sobek.Value {
    if index < 0 || index >= len(s.arr) {
        return sobek.Undefined()
    }
    val, err := s.parse(sobek.Undefined(), s.rt.ToValue(s.arr[index]))
    if err != nil {
        common.Throw(s.rt, err)
    }
    err = s.deepFreeze(s.rt, val)
    if err != nil {
        common.Throw(s.rt, err)
    }
    return val
}
```

On each element access, `JSON.parse()` is called to deserialize the string into a JS object, and `deepFreeze()` makes it immutable. This means:

- **Stored data** (the `[]string` slice): **shared** across all VUs, stored once in memory.
- **Parsed JS objects**: **temporary**, created per-access, eligible for garbage collection after use.

#### 8. Contrast: `open()` + `JSON.parse()` — `js/initcontext.go`

When using `open()` + `JSON.parse()` in the init context, each VU runs the init code independently, creating its own full copy of the parsed data in its JavaScript runtime heap. There is no sharing mechanism.

### Behavioral Answer

**SharedArray memory remains approximately constant regardless of VU count.** The root cause is the pointer-sharing design:

1. **Single initialization:** The `RootModule.New()` creates one `sharedArrays` map (line 44-48). The `get()` method (line 152-167) uses double-checked locking to ensure the array data is constructed exactly once.
2. **Pointer sharing:** `NewModuleInstance()` (line 53-57) passes `&rm.shared` — a pointer to the same struct — to every VU. No data duplication.
3. **Slice sharing:** The `sharedArray` struct (line 10-12 of share.go) holds a `[]string` slice. Go slices are reference types — copying the slice header does not copy the underlying array data.
4. **Lightweight wrappers:** Each VU gets a `wrappedSharedArray` (line 14-21 of share.go) that adds only per-VU JS runtime references (~5 pointers). The actual data array is shared.
5. **On-access parsing:** `Get()` (line 44-58 of share.go) deserializes to JS objects per-access, but these temporary objects are GC'd quickly and don't accumulate.

In contrast, `open()` + `JSON.parse()` creates a full copy of the parsed data in each VU's JavaScript heap, resulting in memory that scales linearly with VU count.

### Runtime Evidence

**Test Configuration:**
- Data: 50,000 items, each with `id`, `name`, and a 200-character `value` field (~12 MB JSON file)
- VUs: 5
- Duration: 10 seconds
- Measurement: `VmRSS` from `/proc/[pid]/status` at 5 seconds into execution

**SharedArray Script:**

```javascript
import { SharedArray } from 'k6/data';
import { sleep } from 'k6';

const data = new SharedArray('mydata', function () {
  return JSON.parse(open('/tmp/testdata.json'));
});

export const options = { vus: 5, duration: '10s' };

export default function () {
  const item = data[Math.floor(Math.random() * data.length)];
  sleep(1);
}
```

**open() Script:**

```javascript
import { sleep } from 'k6';

const data = JSON.parse(open('/tmp/testdata.json'));

export const options = { vus: 5, duration: '10s' };

export default function () {
  const item = data[Math.floor(Math.random() * data.length)];
  sleep(1);
}
```

**Memory Measurements:**

| Approach | VmRSS (kB) | VmRSS (MB) |
|----------|-----------|-----------|
| SharedArray (5 VUs) | **167,440** | **163 MB** |
| open() + JSON.parse() (5 VUs) | **395,212** | **386 MB** |

**Ratio:** `open()` uses **2.36× more memory** than `SharedArray`.

**Root Cause in Source Code:**

The memory difference is explained by:

- **SharedArray:** One copy of the data is held in the `sharedArrays.data` map (a single `[]string` slice). The 5 VUs share this via pointer (`&rm.shared` at `data.go:55`). Per-VU overhead is ~5 pointers for the `wrappedSharedArray`.
- **open() + JSON.parse():** Each of the 5 VUs runs the init code, calling `open()` which reads the file (via `js/initcontext.go`), then `JSON.parse()` which creates a full JavaScript object graph in each VU's Sobek runtime heap. Result: 5 independent copies of the parsed data.

---

## Question 5 — Prometheus Output Metric Name Integrity

**Question:** Does the `experimental-prometheus-rw` output maintain metric name integrity (specifically the `k6_` prefix convention)?

### Source Code Analysis

#### 1. Default Metric Prefix — `vendor/.../remotewrite/config.go:24`

```go
// vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/config.go:24
const defaultMetricPrefix = "k6_"
```

This package-level constant defines the prefix applied to all metric names.

#### 2. Series Mapping — `vendor/.../remotewrite/prometheus.go:11,39-52`

```go
// vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/prometheus.go:11
const namelbl = "__name__"

// vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/prometheus.go:39-52
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

The `__name__` label value is constructed as: `"k6_" + metricName + optional("_" + suffix)`. Labels are sorted lexicographically as required by the Prometheus remote write specification.

#### 3. Metric Type Suffix Mapping — `vendor/.../remotewrite/remotewrite.go:329-356`

Each metric type gets a specific suffix:

```go
// vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/remotewrite.go:329-356
switch swm.Metric.Type {
case metrics.Counter:
    ts := mapMonoSeries(swm.TimeSeries, "total", swm.Latest)    // suffix = "total"
    // ...
case metrics.Gauge:
    ts := mapMonoSeries(swm.TimeSeries, "", swm.Latest)          // suffix = "" (no suffix)
    // ...
case metrics.Rate:
    ts := mapMonoSeries(swm.TimeSeries, "rate", swm.Latest)     // suffix = "rate"
    // ...
case metrics.Trend:
    trend.MapPrompb(swm.TimeSeries, swm.Latest)                  // delegates to trend handler
    // ...
}
```

**Resulting metric name patterns:**

| k6 Metric Type | Suffix | Prometheus Name Pattern | Example |
|----------------|--------|------------------------|---------|
| Counter | `total` | `k6_<name>_total` | `k6_data_sent_total` |
| Gauge | (none) | `k6_<name>` | `k6_vus` |
| Rate | `rate` | `k6_<name>_rate` | `k6_my_success_rate_rate` |
| Trend | stat name | `k6_<name>_<stat>` | `k6_iteration_duration_p99` |

#### 4. Trend Stat Suffixes — `vendor/.../remotewrite/trend.go:78-96`

For Trend metrics, additional stat suffixes are appended:

```go
// vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/trend.go:78-96
func (tg *trendAsGauges) Append(suffix string, v float64) {
    ts := &prompb.TimeSeries{
        Labels:  make([]*prompb.Label, len(tg.labels)),
        Samples: make([]*prompb.Sample, 1),
    }
    for i := 0; i < len(tg.labels); i++ {
        ts.Labels[i] = &prompb.Label{
            Name:  tg.labels[i].Name,
            Value: tg.labels[i].Value,
        }
    }
    ts.Labels[tg.ixname].Value += "_" + suffix
    // ...
}
```

Note: `MapSeries` is called with `suffix=""` for Trends (at trend.go:48), producing `k6_<name>`. Then `Append()` adds `_<stat>` to produce `k6_<name>_<stat>`. Available stats (from config.go:28): `p(99)` becomes `p99`, plus `avg`, `min`, `max`, `med`, `count`, `sum`.

### Behavioral Answer

**Yes, the `experimental-prometheus-rw` output maintains metric name integrity.** The `k6_` prefix is hardcoded as `defaultMetricPrefix` (config.go:24) and is unconditionally prepended to every metric name in the `MapSeries()` function (prometheus.go:40). There is no configuration option to disable or change this prefix in the v0.55.0 codebase.

The complete naming convention is:

- **Counters:** `k6_<metric_name>_total` (e.g., `k6_iterations_total`, `k6_data_sent_total`, `k6_my_custom_counter_total`)
- **Gauges:** `k6_<metric_name>` (e.g., `k6_vus`, `k6_vus_max`, `k6_my_custom_gauge`)
- **Rates:** `k6_<metric_name>_rate` (e.g., `k6_my_success_rate_rate`)
- **Trends:** `k6_<metric_name>_<stat>` (e.g., `k6_iteration_duration_p99`, `k6_my_custom_trend_p99`)

### Runtime Evidence

**Test Script Used:**

```javascript
import { Counter, Trend, Gauge, Rate } from 'k6/metrics';
import { sleep } from 'k6';

const myCounter = new Counter('my_custom_counter');
const myTrend = new Trend('my_custom_trend');
const myGauge = new Gauge('my_custom_gauge');
const myRate = new Rate('my_success_rate');

export const options = { vus: 2, duration: '5s' };

export default function () {
  myCounter.add(1);
  myTrend.add(Math.random() * 100);
  myGauge.add(42);
  myRate.add(Math.random() > 0.3);
  sleep(0.5);
}
```

**Execution Command:**

```
/tmp/k6 run -o experimental-prometheus-rw=http://localhost:9090/api/v1/write /tmp/exp5_prom.js
```

**Payload Capture Method:** A Python HTTP server listening on port 9090 captured the Snappy-compressed protobuf payloads, decompressed them, and saved the raw bytes. Metric names were extracted as strings from the binary protobuf data.

**Extracted Metric Names from Remote Write Payload:**

```
k6_data_received_total
k6_data_sent_total
k6_iteration_duration_p99
k6_iterations_total
k6_my_custom_counter_total
k6_my_custom_gauge
k6_my_custom_trend_p99
k6_my_success_rate_rate
k6_vus
k6_vus_max
```

**Verification Against Source Code:**

| Extracted Name | k6 Metric | Type | Expected Suffix | Matches? |
|---|---|---|---|---|
| `k6_data_received_total` | `data_received` | Counter | `_total` | ✅ |
| `k6_data_sent_total` | `data_sent` | Counter | `_total` | ✅ |
| `k6_iteration_duration_p99` | `iteration_duration` | Trend | `_p99` | ✅ |
| `k6_iterations_total` | `iterations` | Counter | `_total` | ✅ |
| `k6_my_custom_counter_total` | `my_custom_counter` | Counter | `_total` | ✅ |
| `k6_my_custom_gauge` | `my_custom_gauge` | Gauge | (none) | ✅ |
| `k6_my_custom_trend_p99` | `my_custom_trend` | Trend | `_p99` | ✅ |
| `k6_my_success_rate_rate` | `my_success_rate` | Rate | `_rate` | ✅ |
| `k6_vus` | `vus` | Gauge | (none) | ✅ |
| `k6_vus_max` | `vus_max` | Gauge | (none) | ✅ |

All 10 captured metric names follow the `k6_` prefix convention exactly as defined in the source code.

---

## Summary and Conclusions

### Answer Summary

| # | Question | Answer |
|---|----------|--------|
| 1 | **VU Management under SIGINT** | On first SIGINT, k6 emits `"Stopping k6 in response to signal..."` and triggers graceful shutdown. Active VUs are allowed to finish their current iteration (counted as "complete"), while VUs whose context is cancelled mid-execution are counted as "interrupted". The test reported **13 complete and 5 interrupted iterations**. |
| 2 | **gRPC Server Streaming Interruption** | With `gracefulRampDown: '30ms'`, streams are cancelled via `grpcext.ErrCanceled`. Log entries include `"stream is cancelled/finished"` and `"stream /main.FeatureExplorer/ListFeatures is closing"`. The `grpc_streams_msgs_received` value was **134** across 6 streams (varying per-stream: 9, 16, 22, 29 messages depending on interruption timing). |
| 3 | **Dropped Iterations via API** | The REST API at `GET /v1/metrics/dropped_iterations` returned `{"count":35,"rate":8.796}` mid-test, with the final summary reporting **46** dropped iterations. The metric is emitted per-drop as a Counter sample with value 1 in `constant_arrival_rate.go:339-346`. |
| 4 | **SharedArray Data Sharing** | SharedArray memory remains approximately constant regardless of VU count. All VUs share a single `[]string` slice via pointer (`&rm.shared` in `data.go:55`). Measured: SharedArray=167 MB vs open()=386 MB for 5 VUs (**2.36× difference**). Root cause: `NewModuleInstance()` passes a pointer, not a copy. |
| 5 | **Prometheus Metric Name Integrity** | Yes, metric names are correctly prefixed with `k6_`. The prefix is hardcoded as `defaultMetricPrefix = "k6_"` (config.go:24) and unconditionally prepended in `MapSeries()` (prometheus.go:40). All 10 captured metric names in the remote write payload confirmed the `k6_` prefix. |

### Key Source File References

| File | Key Lines | Purpose |
|------|-----------|---------|
| `cmd/common.go` | 96-129 | Signal trap registration and two-stage handler |
| `cmd/run.go` | 349-361 | Graceful/hard stop closures with log messages |
| `execution/abort.go` | 24-36 | Context cancellation via abort controller |
| `execution/scheduler.go` | 425-430 | Interrupt detection and status logging |
| `lib/executor/vu_handle.go` | 147-264 | VU state machine (5 states), gracefulStop/hardStop |
| `lib/executor/helpers.go` | 104-141 | Iteration accounting (full vs. interrupted) |
| `lib/executor/ramping_vus.go` | 491-712 | Ramping executor: VU handle creation, stepping, waiter |
| `lib/executor/base_config.go` | 20 | Default graceful stop: 30 seconds |
| `lib/executor/constant_arrival_rate.go` | 324-355 | Dropped iteration emission and VU warning |
| `js/modules/k6/grpc/metrics.go` | 17-27 | gRPC stream metric registration |
| `js/modules/k6/grpc/stream.go` | 81-218 | Stream lifecycle: open, loop, readData, queueMessage |
| `js/modules/k6/data/data.go` | 18-167 | SharedArray module: pointer sharing, double-checked locking |
| `js/modules/k6/data/share.go` | 10-58 | SharedArray internal: shared slice, per-access parse |
| `api/v1/routes.go` | 31-38 | REST API metric route |
| `api/v1/metric_routes.go` | 27-49 | Metric handler with lock and JSON:API response |
| `api/v1/metric.go` | 74-82 | Metric model with Sink.Format() |
| `api/v1/metric_jsonapi.go` | 24-28 | JSON:API envelope construction |
| `metrics/builtin.go` | 10, 84 | `dropped_iterations` constant and registration |
| `vendor/.../remotewrite/config.go` | 24 | `defaultMetricPrefix = "k6_"` |
| `vendor/.../remotewrite/prometheus.go` | 39-52 | `MapSeries()`: prefix + name + suffix |
| `vendor/.../remotewrite/remotewrite.go` | 329-356 | Type-specific suffix mapping |
| `vendor/.../remotewrite/trend.go` | 78-96 | Trend stat suffix appending |
