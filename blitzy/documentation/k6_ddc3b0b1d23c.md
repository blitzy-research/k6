# Can k6's HTTP request-timing metrics be trusted? — A run-code-first investigation

> **Scope of this document.** You are debugging HTTP timing metrics in a k6 load test and suspect "a measurement bug or race condition" in the HTTP tracer. This document answers, per timing metric, whether `res.timings` can be trusted, and adjudicates each of your five reported anomalies as *expected‑by‑design* or *genuine bug*. Every value cited below was **observed at runtime** through the canonical k6 entry point (the compiled `k6` binary running a real script, or a real `go test` run). Statements derived from reading code rather than from a runtime observation are explicitly labelled **(inferred)**.

---

## 1. Bottom line (read this first)

1. **Can `res.timings` be trusted for performance analysis? — YES.** Every populated timing (`blocked`, `connecting`, `tls_handshaking`, `sending`, `waiting`, `receiving`, `duration`) is computed correctly and is trustworthy. The values you see are the *true* consequences of HTTP keep‑alive connection reuse and the Go `net/http/httptrace` hook contract — not measurement errors.

2. **Do any of the five anomalies warrant an upstream bug report? — NO.** All five are by‑design behaviour. Two of them are hypotheses that this investigation **refutes by running the code**:
   - The **"race in the measurement code"** (Anomaly 3) is refuted by running the maintainers' tracer suite — including a 200‑way parallel cancelled‑request stress test — **under the Go race detector**: exit `0`, **zero** `WARNING: DATA RACE`.
   - The **"double counting bug"** (Anomaly 5) is refuted by the tracer's own de‑duplication: multiple `ConnectStart`/`ConnectDone` invocations collapse to the *first* timestamp via `atomic.CompareAndSwapInt64`, so later invocations are no‑ops.

**The only nuance** is the Go standard library's HTTP/2 false‑`Reused` quirk (Anomaly 4). k6 **already works around it in‑tree** (`lib/netext/httpext/tracer.go:L287-L288`), so there is still nothing to report upstream.

### Environment and canonical build (grounds every observation)

All observations were produced by a normal‑user build of k6 from the checked‑out source.

| Item | Value |
|------|-------|
| Module / commit | `go.k6.io/k6` @ HEAD `ddc3b0b1d23c128e34e2792fc9075f9126e32375` |
| Build command | `go build -o k6 .` (uses the vendored module set under `vendor/`) |
| Version banner | `k6 v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)` |
| Go toolchain | `go version go1.23.12 linux/amd64` |
| OS / arch | `Linux … 6.6.122+ x86_64 GNU/Linux` |
| Race detector | `CGO_ENABLED=1` + `gcc 15.2.0` (environment‑only; not a repo dependency) |

The banner shape is produced by `FullVersion()` — `fmt.Sprintf("%s (commit/%s, %s)", Version, commit, goVersionArch)` (`lib/consts/consts.go:L52`) with `Version = "0.55.0"` (`lib/consts/consts.go:L12`).

```text
$ go build -o k6 .
$ ./k6 version
k6 v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)
```

---

## 2. How the timing pipeline works (so the answers make sense)

Every timing value in `res.timings` and every `http_req_*` metric originates from the same object — a `Trail` produced by a per‑request `Tracer`. Understanding this one‑to‑one flow is what makes all five anomalies obviously by‑design.

**Step 1 — a *fresh* `Tracer` is created for every request** and attached to the request context. Tracers are never reused, which is why there is no cross‑request state to corrupt:

```go
// RoundTrip is the implementation of http.RoundTripper
func (t *transport) RoundTrip(req *http.Request) (*http.Response, error) {
	t.processLastSavedRequest(nil)

	ctx := req.Context()
	tracer := &Tracer{}
	reqWithTracer := req.WithContext(httptrace.WithClientTrace(ctx, tracer.Trace()))
	resp, err := t.state.Transport.RoundTrip(reqWithTracer)
```
(`lib/netext/httpext/transport.go:L200-L207`; the fresh instance is at `L205`.) The tracer's own doc comment states the rule: <br>`// It's NOT safe to reuse Tracers between requests.` (`lib/netext/httpext/tracer.go:L145`).

**Step 2 — `Tracer.Trace()` wires all eight `httptrace` hooks:**

```go
func (t *Tracer) Trace() *httptrace.ClientTrace {
	return &httptrace.ClientTrace{
		GetConn:              t.GetConn,
		ConnectStart:         t.ConnectStart,
		ConnectDone:          t.ConnectDone,
		TLSHandshakeStart:    t.TLSHandshakeStart,
		TLSHandshakeDone:     t.TLSHandshakeDone,
		GotConn:              t.GotConn,
		WroteRequest:         t.WroteRequest,
		GotFirstResponseByte: t.GotFirstResponseByte,
	}
}
```
(`lib/netext/httpext/tracer.go:L162-L173`.) Each hook records a Unix‑nanosecond timestamp into an `int64` field of the `Tracer` struct (`lib/netext/httpext/tracer.go:L147-L159`) using `sync/atomic`.

**Step 3 — after the round‑trip, `Tracer.Done()` builds the `Trail`:**

```go
func (t *transport) measureAndEmitMetrics(unfReq *unfinishedRequest) *finishedRequest {
	trail := unfReq.tracer.Done()
```
(`lib/netext/httpext/transport.go:L77-L78`.)

**Step 4 — the *same* `Trail` feeds two sinks, which is why script‑visible `res.timings` and aggregated `http_req_*` metrics always agree:**

- **Sink A — `http_req_*` samples:** `trail.SaveSamples(t.state.BuiltinMetrics, &tagsAndMeta)` (`lib/netext/httpext/transport.go:L146`). The metric names and their registration as `Trend`/`Time` metrics are in `metrics/builtin.go:L18-L20` and `metrics/builtin.go:L92-L94`.
- **Sink B — `res.timings`:** copied field‑by‑field into the JS‑facing struct:

```go
	k6Response.Timings = ResponseTimings{
		Duration:       metrics.D(trail.Duration),
		Blocked:        metrics.D(trail.Blocked),
		Connecting:     metrics.D(trail.Connecting),
		TLSHandshaking: metrics.D(trail.TLSHandshaking),
		Sending:        metrics.D(trail.Sending),
		Waiting:        metrics.D(trail.Waiting),
		Receiving:      metrics.D(trail.Receiving),
	}
```
(`lib/netext/httpext/request.go:L98-L106`.)

`metrics.D()` converts nanoseconds → **milliseconds**, which is why `res.timings` values are in ms:

```go
const timeUnit = time.Millisecond

// D formats a duration for emission.
// The reverse of D() is ToD().
func D(d time.Duration) float64 {
	return float64(d) / float64(timeUnit)
}
```
(`metrics/units.go:L7,L11-L13`.)

**The user‑facing struct** the user reads as `res.timings` is `ResponseTimings` (`lib/netext/httpext/response.go:L34-L44`), exposed on the response as `Timings ResponseTimings \`json:"timings"\`` (`lib/netext/httpext/response.go:L65`).

> **Structural note used later:** `ResponseTimings` declares a `LookingUp float64 \`json:"looking_up"\`` field (`lib/netext/httpext/response.go:L38`), but the population site in `request.go:L98-L106` **omits it**. Therefore `res.timings.looking_up` is **always `0`** — a structural constant, not a measurement. This was confirmed at runtime (`looking_up:0` in every iteration below).

**Base dialer configuration** (relevant to Anomalies 1 & 2 — this is the keep‑alive that enables reuse):

```go
		BaseDialer: net.Dialer{
			Timeout:   30 * time.Second,
			KeepAlive: 30 * time.Second,
		},
```
(`js/runner.go:L90-L93`.)

The end‑to‑end flow, and where each anomaly is produced:

```mermaid
flowchart TD
    A["VU runs http.get in a k6 script"] --> B["transport.RoundTrip (transport.go:L201)"]
    B --> C["fresh Tracer created per request (transport.go:L205)"]
    C --> D["httptrace hooks fire: GetConn, ConnectStart/Done, TLSHandshakeStart/Done, GotConn, WroteRequest, GotFirstResponseByte"]
    D --> E{"Connection reused?"}
    E -->|"Reused=true"| F["GotConn SwapInt64 sets connectStart==connectDone==now (tracer.go:L271-277)  --> Anomaly 1 & 2"]
    E -->|"Reused=false"| G["hooks record FIRST timestamp via CompareAndSwap; HTTP/2 false-Reused guarded (tracer.go:L287-288) --> Anomaly 4 & 5"]
    F --> H["Tracer.Done builds Trail (tracer.go:L315-384)"]
    G --> H
    H --> I["http_req_* samples (transport.go:L146)"]
    H --> J["res.timings via metrics.D() ns->ms (request.go:L98-106)"]
```

---

## 3. Per‑anomaly adjudication

Each section leads with the **direct verdict**, then the **causal mechanism** (exact function/struct + `file:line`, code quoted faithfully), then the **runtime demonstration** (exact command + complete unedited output).

### 3.1 Anomaly 1 — `connecting` and `tls_handshaking` are exactly `0` on subsequent requests

> **Your words:** *"the first request shows reasonable values for connecting time and TLS handshaking, but subsequent requests show exactly 0 for both of these metrics even though I can see network activity happening."*

**(a) Verdict: EXPECTED‑BY‑DESIGN.** A reused keep‑alive connection performs **no new TCP dial and no new TLS handshake**, so the `ConnectStart`/`ConnectDone`/`TLSHandshake*` hooks never fire for it. k6 deliberately reports `0` for those phases, while `waiting`/`receiving` stay non‑zero — that non‑zero waiting/receiving **is** the "network activity" you correctly observe. This is trustworthy, not a bug.

**(b) Mechanism.** When the connection is reused, `Tracer.GotConn` is the *first* hook called, and it overwrites the connect/TLS timestamps to the same `now` instant so the (never‑fired) hooks cannot leave stale values:

```go
	_, isConnTLS := info.Conn.(*tls.Conn)
	if info.Reused {
		atomic.SwapInt64(&t.connectStart, now)
		atomic.SwapInt64(&t.connectDone, now)
		if isConnTLS {
			atomic.SwapInt64(&t.tlsHandshakeStart, now)
			atomic.SwapInt64(&t.tlsHandshakeDone, now)
		}
	} else {
```
(`lib/netext/httpext/tracer.go:L270-L278`.) Because `connectStart == connectDone` and `tlsHandshakeStart == tlsHandshakeDone`, `Tracer.Done` computes both phases as exactly zero:

```go
	if connectDone != 0 && connectStart != 0 {
		trail.Connecting = time.Duration(connectDone - connectStart)
	}
	if tlsHandshakeDone != 0 && tlsHandshakeStart != 0 {
		trail.TLSHandshaking = time.Duration(tlsHandshakeDone - tlsHandshakeStart)
	}
```
(`lib/netext/httpext/tracer.go:L340-L345`: `Connecting = connectDone - connectStart = 0`, `TLSHandshaking = 0`.)

The `httptrace` contract that a reused connection does not fire these hooks is documented on the hooks themselves — e.g. `// If the connection is reused, this won't be called.` for `ConnectStart` (`lib/netext/httpext/tracer.go:L195`) and for the TLS hooks (`lib/netext/httpext/tracer.go:L228`). Externally, the Go `net/http/httptrace` docs and HTTP/1.1 keep‑alive semantics confirm a reused persistent connection performs no new dial or handshake **(inferred from Go docs)**.

**Maintainers corroborate this exact behaviour.** `TestTracer` drives three sequential requests with `iterations []bool{false, true, true}` (`lib/netext/httpext/tracer_test.go:L115`) and asserts that on a reused connection `http_req_connecting` and `http_req_tls_handshaking` are `0`:

```go
				case builtinMetrics.HTTPReqConnecting, builtinMetrics.HTTPReqTLSHandshaking:
					if isReuse {
						assert.Equal(t, 0.0, s.Value)
						break
					}
					fallthrough
```
(`lib/netext/httpext/tracer_test.go:L161-L165`.)

**(c) Demonstration.** Local HTTP/1.1 keep‑alive HTTPS server (`httptest.StartTLS`, self‑signed, bound to `127.0.0.1:8443`), single VU, 4 sequential iterations to the same host. Scale: `vus:1, iterations:4`; the reuse pattern was stable across 8 repeats (see §3.2). The protocol was confirmed to be HTTP/1.1 (`res.proto:"HTTP/1.1"`), so this is a direct observation of HTTP/1.1 keep‑alive reuse.

Complete, unedited output of the run:

```text
$ K6_NO_USAGE_REPORT=true ./k6 run /tmp/obs/script.js

         /\      Grafana   /‾‾/
    /\  /  \     |\  __   /  /
   /  \/    \    | |/ /  /   ‾‾\
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/

     execution: local
        script: /tmp/obs/script.js
        output: -

     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 4 iterations shared among 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)

time="2026-07-13T16:26:35Z" level=info msg="{\"iter\":0,\"reused_hint\":\"new?\",\"blocked\":4.139314,\"connecting\":0.27822,\"tls_handshaking\":3.72873,\"sending\":0.130117,\"waiting\":0.358534,\"receiving\":0.170281,\"duration\":0.658932,\"looking_up\":0}" source=console
time="2026-07-13T16:26:35Z" level=info msg="{\"iter\":1,\"reused_hint\":\"reused?\",\"blocked\":0.008168,\"connecting\":0,\"tls_handshaking\":0,\"sending\":0.025388,\"waiting\":0.286007,\"receiving\":0.082686,\"duration\":0.394081,\"looking_up\":0}" source=console
time="2026-07-13T16:26:35Z" level=info msg="{\"iter\":2,\"reused_hint\":\"reused?\",\"blocked\":0.005335,\"connecting\":0,\"tls_handshaking\":0,\"sending\":0.014653,\"waiting\":0.190191,\"receiving\":0.069479,\"duration\":0.274323,\"looking_up\":0}" source=console
time="2026-07-13T16:26:35Z" level=info msg="{\"iter\":3,\"reused_hint\":\"reused?\",\"blocked\":0.004537,\"connecting\":0,\"tls_handshaking\":0,\"sending\":0.019466,\"waiting\":0.274791,\"receiving\":0.040969,\"duration\":0.335226,\"looking_up\":0}" source=console

     data_received..................: 2.1 kB 263 kB/s
     data_sent......................: 733 B  93 kB/s
     http_req_blocked...............: avg=1.03ms   min=4.53µs   med=6.75µs   max=4.13ms   p(90)=2.89ms   p(95)=3.51ms
     http_req_connecting............: avg=69.55µs  min=0s       med=0s       max=278.22µs p(90)=194.75µs p(95)=236.48µs
     http_req_duration..............: avg=415.64µs min=274.32µs med=364.65µs max=658.93µs p(90)=579.47µs p(95)=619.2µs
       { expected_response:true }...: avg=415.64µs min=274.32µs med=364.65µs max=658.93µs p(90)=579.47µs p(95)=619.2µs
     http_req_failed................: 0.00%  0 out of 4
     http_req_receiving.............: avg=90.85µs  min=40.96µs  med=76.08µs  max=170.28µs p(90)=144µs    p(95)=157.14µs
     http_req_sending...............: avg=47.4µs   min=14.65µs  med=22.42µs  max=130.11µs p(90)=98.69µs  p(95)=114.4µs
     http_req_tls_handshaking.......: avg=932.18µs min=0s       med=0s       max=3.72ms   p(90)=2.61ms   p(95)=3.16ms
     http_req_waiting...............: avg=277.38µs min=190.19µs med=280.39µs max=358.53µs p(90)=336.77µs p(95)=347.65µs
     http_reqs......................: 4      507.94857/s
     iteration_duration.............: avg=1.9ms    min=455.74µs med=643.51µs max=5.85ms   p(90)=4.34ms   p(95)=5.1ms
     iterations.....................: 4      507.94857/s


running (00m00.0s), 0/1 VUs, 4 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.0s/10m0s  4/4 shared iters
```

**Reading the output:** iteration 0 (new connection) shows `connecting=0.27822` ms and `tls_handshaking=3.72873` ms; iterations 1–3 (reused) show **exactly `0`** for both, while `waiting` stays `0.19–0.29` ms and `receiving` stays `0.04–0.08` ms — real network activity with zero connect/TLS time. Note also that the summary `http_req_connecting` (med `0s`, max `278.22µs`) and `http_req_tls_handshaking` (med `0s`, max `3.72ms`) exactly match the per‑iteration `res.timings` — **proving Sink A and Sink B agree**. `looking_up` is `0` in every line.

---

### 3.2 Anomaly 2 — `blocked` is sometimes large, sometimes near zero for "identical" requests

> **Your words:** *"sometimes the \"blocked\" timing shows massive values like 500ms when other times it's near zero for identical requests."*

**(a) Verdict: EXPECTED‑BY‑DESIGN.** `blocked` is the time spent *acquiring a connection*. On a **cold** request it includes the new DNS + TCP + TLS work; on a **warm** request it is just an idle‑pool checkout, which is near zero. The requests are not truly identical: the first pays for the connection, the rest inherit it. The bimodal distribution is the correct measurement of two different situations.

**(b) Mechanism.** `Tracer.Done` computes `Blocked` as the gap between asking for a connection (`GetConn` → `getConn`) and getting one (`GotConn` → `gotConn`):

```go
	if t.gotConn != 0 && t.getConn != 0 && t.gotConn > t.getConn {
		trail.Blocked = time.Duration(t.gotConn - t.getConn)
	}
```
(`lib/netext/httpext/tracer.go:L323-L325`.) `getConn` is set by `GetConn` (`lib/netext/httpext/tracer.go:L187-L189`) and `gotConn` by `GotConn` (`lib/netext/httpext/tracer.go:L261`). The explicit `gotConn > getConn` guard is *why* you get a clean `0`/near‑zero rather than a negative or garbage value on reuse.

**(c) Demonstration — observed distribution.** The **same unchanged script** from §3.1 was run **8 times**. Each `k6 run` is a fresh process, so iteration 0 is always cold (new connection) and iterations 1–3 are always warm (reused). Command:

```text
$ for i in $(seq 1 8); do K6_NO_USAGE_REPORT=true ./k6 run /tmp/obs/script.js; done
```

Complete per‑iteration `blocked` (ms) captured from the console line of each run:

```text
----- RUN 1 -----   iter0(new)=2.624108   iter1=0.003531  iter2=0.003949  iter3=0.003347
----- RUN 2 -----   iter0(new)=3.164153   iter1=0.005399  iter2=0.003524  iter3=0.004233
----- RUN 3 -----   iter0(new)=2.385232   iter1=0.008528  iter2=0.002912  iter3=0.006019
----- RUN 4 -----   iter0(new)=2.639064   iter1=0.004310  iter2=0.005127  iter3=0.003171
----- RUN 5 -----   iter0(new)=2.283921   iter1=0.003026  iter2=0.002327  iter3=0.006104
----- RUN 6 -----   iter0(new)=2.456022   iter1=0.003846  iter2=0.002693  iter3=0.002679
----- RUN 7 -----   iter0(new)=2.574624   iter1=0.004169  iter2=0.006310  iter3=0.005153
----- RUN 8 -----   iter0(new)=2.980089   iter1=0.004246  iter2=0.003475  iter3=0.003470
```

Aggregate statistics over the 8 runs (36 samples: 8 cold + 24 warm):

| Group | n | min (ms) | max (ms) | mean (ms) |
|-------|---|----------|----------|-----------|
| **COLD** (iter 0, new connection) | 8 | 2.283921 | 3.164153 | 2.638402 |
| **WARM** (iters 1–3, reused) | 24 | 0.002327 | 0.008528 | 0.004231 |

- `mean(cold) / mean(warm) = 623.6×`. Every cold sample `> 2.0 ms`; every warm sample `< 0.01 ms`.
- The **bimodal split is stable across all 8 runs** — cold is always in the ~2.3–3.2 ms band, warm always sub‑0.01 ms. This is exactly the "sometimes big, sometimes near zero" you report.
- As a bonus, `connecting == 0` and `tls_handshaking == 0` held on **all 24/24 reused samples**, re‑confirming Anomaly 1's stability.

**On the "500 ms" magnitude:** locally the cold value is a few ms because DNS is trivial and the peer is loopback. **(inferred)** Over a real network the *same* `gotConn - getConn` formula also includes real DNS resolution, the TCP three‑way handshake, and the full TLS handshake to a remote host, which routinely sums to hundreds of milliseconds. The formula is identical; only the network cost differs. So a 500 ms cold `blocked` next to a near‑zero warm `blocked` is the expected, correct behaviour — not a defect.

---


### 3.3 Anomaly 3 — on Windows, ALL timings occasionally return `0`; is it "a race in the measurement code"?

> **Your words:** *"occasionally ALL the timing metrics return 0 for random requests, which definitely seems like a race in the measurement code."*

**(a) Verdict: the "race" hypothesis is REFUTED. The all‑zero result is EXPECTED‑BY‑DESIGN,** arising from two non‑race causes: **(i)** the **request‑error path**, which emits an all‑zero `Trail` (status 0) when the request never completes; and **(ii)** **Windows low timer resolution**, where two hooks can read the *same* clock tick so a phase computes to `0`. Neither is a data race.

**(b) Causal mechanism — why the all‑zero is not a race, plus its two real causes.**

*Why it is not a race.* The tracer is deliberately **lock‑free**, using `sync/atomic` on `int64` fields *precisely because* hooks can fire after `Done()` has already returned — the safe, intended handling of late callbacks, not a bug:

```go
	// It's possible for some of the methods of httptrace.ClientTrace to
	// actually be called after the http.Client or http.RoundTripper have
	// already returned our result and we've called Done(). This happens
	// mostly for cancelled requests, but we have to use atomics here as
	// well (or use global Tracer locking) so we can avoid data races.
```
(`lib/netext/httpext/tracer.go:L327-L331`.) In `Done`, every field is read with `atomic.LoadInt64` (`lib/netext/httpext/tracer.go:L332-L338`). The Go `net/http/httptrace` docs themselves state that hook functions "may be called concurrently … and some may be called after the request has completed" **(inferred from Go docs)** — which is the exact scenario these atomics cover.

*Real cause (i) — the request‑error path.* When a request never completes (connection refused, reset, cancelled, timeout), the hooks that set the timestamps never fire, so `Done` returns a `Trail` whose fields are all zero and the response carries status `0`. `error_code 1212` is `tcpDialRefusedErrorCode` (`lib/netext/httpext/error_codes.go:L42`). The **Windows** reset variant maps `syscall.WSAECONNRESET` → `tcpResetByPeerErrorCode` (`= 1220`, `lib/netext/httpext/error_codes.go:L44`):

```go
func getOSSyscallErrorCode(e *net.OpError, se *os.SyscallError) (errCode, string) {
	switch se.Unwrap() {
	case syscall.WSAECONNRESET:
		return tcpResetByPeerErrorCode, fmt.Sprintf(tcpResetByPeerErrorCodeMsg, e.Op)
	}
	return 0, ""
}
```
(`lib/netext/httpext/error_codes_syscall_windows.go:L10-L16`.) So on Windows a mid‑flight reset surfaces as an *errored* request with zero timings — an error signal, not a measurement fault.

*Real cause (ii) — Windows timer resolution.* The maintainers document this directly in their own test hack — on Windows, two hooks can register the *same* timestamp, making a phase compute to `0`:

```go
	if runtime.GOOS == "windows" {
		// HACK: Time resolution is not as accurate on Windows, see:
		//  https://github.com/golang/go/issues/8687
		//  https://github.com/golang/go/issues/41087
		// Which seems to be causing some metrics to have a value of 0,
		// since e.g. ConnectStart and ConnectDone could register the same time.
```
(`lib/netext/httpext/tracer_test.go:L33-L40`; the test inserts a `traceDelay = 100 * time.Millisecond` sleep, `tracer_test.go:L28`, to space the hooks apart.) **(inferred — not observed here, as this run is on Linux/amd64):** on a low‑resolution Windows clock the same effect can zero a phase for a *successful* request too, which explains the "random requests" you saw on your Windows machine. This is a platform clock‑granularity artefact, again not a race and not a k6 calculation error.

**(c) Demonstration.**

*Refuting the race — run under the race detector.* The maintainers' `TestCancelledRequest` fires **200 parallel cancelled requests** — the precise concurrency that would surface a measurement race:

```go
	// This Run will not return until the parallel subtests complete.
	t.Run("group", func(t *testing.T) {
		t.Parallel()
		for i := 0; i < 200; i++ {
			t.Run(fmt.Sprintf("TestCancelledRequest_%d", i),
				func(t *testing.T) {
					t.Parallel()
					cancelTest(t)
				})
		}
	})
```
(`lib/netext/httpext/tracer_test.go:L282-L292`; each `cancelTest` cancels mid‑flight and then calls `tracer.Done()` at `L275`.)

Normal run — complete top‑level output:

```text
$ go test -count=1 ./lib/netext/httpext/ -run "TestTracer|TestTracerError|TestTracerNegativeHttpSendingValues|TestCancelledRequest" -v
=== RUN   TestTracer
=== RUN   TestTracerNegativeHttpSendingValues
=== RUN   TestTracerError
=== RUN   TestCancelledRequest
=== RUN   TestTracer/Test_#0
=== RUN   TestTracer/Test_#1
=== RUN   TestTracer/Test_#2
--- PASS: TestTracer (0.01s)
--- PASS: TestTracerNegativeHttpSendingValues (0.01s)
--- PASS: TestTracerError (0.06s)
--- PASS: TestCancelledRequest (0.00s)
PASS
ok  	go.k6.io/k6/lib/netext/httpext	0.320s
NORMAL_EXIT=0
```

Race‑detector run — complete top‑level output, exit code, and data‑race count:

```text
$ CGO_ENABLED=1 go test -race -count=1 ./lib/netext/httpext/ -run "TestTracer|TestCancelledRequest" -v
--- PASS: TestTracer (0.03s)
--- PASS: TestTracerNegativeHttpSendingValues (0.05s)
--- PASS: TestCancelledRequest (0.56s)
--- PASS: TestTracerError (2.04s)
PASS
ok  	go.k6.io/k6/lib/netext/httpext	3.083s
RACE_EXIT=0
DATA_RACE_WARNINGS=0
```

**Result:** exit `0`, and `DATA_RACE_WARNINGS=0` — **not a single `WARNING: DATA RACE`** even under 200 concurrent cancelled requests. The "race in the measurement code" hypothesis is refuted by execution.

*The all‑zero result — run against a closed port.* Pointing the same script at a closed port (nothing listening on `127.0.0.1:65000`) makes every request error, so the emitted `Trail` is all zero (status 0). Complete, unedited output:

```text
$ K6_NO_USAGE_REPORT=true ./k6 run /tmp/obs/script_err.js

         /\      Grafana   /‾‾/
    /\  /  \     |\  __   /  /
   /  \/    \    | |/ /  /   ‾‾\
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/

     execution: local
        script: /tmp/obs/script_err.js
        output: -

     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 3 iterations shared among 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)

time="2026-07-13T16:27:32Z" level=warning msg="Request Failed" error="Get \"https://127.0.0.1:65000/\": dial tcp 127.0.0.1:65000: connect: connection refused"
time="2026-07-13T16:27:32Z" level=info msg="{\"iter\":0,\"status\":0,\"error\":\"dial: connection refused\",\"error_code\":1212,\"blocked\":0,\"connecting\":0,\"tls_handshaking\":0,\"sending\":0,\"waiting\":0,\"receiving\":0,\"duration\":0}" source=console
time="2026-07-13T16:27:32Z" level=warning msg="Request Failed" error="Get \"https://127.0.0.1:65000/\": dial tcp 127.0.0.1:65000: connect: connection refused"
time="2026-07-13T16:27:32Z" level=info msg="{\"iter\":1,\"status\":0,\"error\":\"dial: connection refused\",\"error_code\":1212,\"blocked\":0,\"connecting\":0,\"tls_handshaking\":0,\"sending\":0,\"waiting\":0,\"receiving\":0,\"duration\":0}" source=console
time="2026-07-13T16:27:32Z" level=warning msg="Request Failed" error="Get \"https://127.0.0.1:65000/\": dial tcp 127.0.0.1:65000: connect: connection refused"
time="2026-07-13T16:27:32Z" level=info msg="{\"iter\":2,\"status\":0,\"error\":\"dial: connection refused\",\"error_code\":1212,\"blocked\":0,\"connecting\":0,\"tls_handshaking\":0,\"sending\":0,\"waiting\":0,\"receiving\":0,\"duration\":0}" source=console

     data_received..............: 0 B     0 B/s
     data_sent..................: 0 B     0 B/s
     http_req_blocked...........: avg=0s       min=0s       med=0s       max=0s       p(90)=0s       p(95)=0s
     http_req_connecting........: avg=0s       min=0s       med=0s       max=0s       p(90)=0s       p(95)=0s
     http_req_duration..........: avg=0s       min=0s       med=0s       max=0s       p(90)=0s       p(95)=0s
     http_req_failed............: 100.00% 3 out of 3
     http_req_receiving.........: avg=0s       min=0s       med=0s       max=0s       p(90)=0s       p(95)=0s
     http_req_sending...........: avg=0s       min=0s       med=0s       max=0s       p(90)=0s       p(95)=0s
     http_req_tls_handshaking...: avg=0s       min=0s       med=0s       max=0s       p(90)=0s       p(95)=0s
     http_req_waiting...........: avg=0s       min=0s       med=0s       max=0s       p(90)=0s       p(95)=0s
     http_reqs..................: 3       1816.797749/s
     iteration_duration.........: avg=497.25µs min=414.03µs med=484.61µs max=593.13µs p(90)=571.43µs p(95)=582.28µs
     iterations.................: 3       1816.797749/s


running (00m00.0s), 0/1 VUs, 3 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.0s/10m0s  3/3 shared iters
```

Every iteration: `status:0`, `error_code:1212`, and **all seven timings `0`**; `http_req_failed` is `100.00%`. A second identical run produced byte‑identical console lines (stable across ≥2 runs):

```text
$ K6_NO_USAGE_REPORT=true ./k6 run /tmp/obs/script_err.js   # second run, console lines only
{"iter":0,"status":0,"error":"dial: connection refused","error_code":1212,"blocked":0,"connecting":0,"tls_handshaking":0,"sending":0,"waiting":0,"receiving":0,"duration":0}
{"iter":1,"status":0,"error":"dial: connection refused","error_code":1212,"blocked":0,"connecting":0,"tls_handshaking":0,"sending":0,"waiting":0,"receiving":0,"duration":0}
{"iter":2,"status":0,"error":"dial: connection refused","error_code":1212,"blocked":0,"connecting":0,"tls_handshaking":0,"sending":0,"waiting":0,"receiving":0,"duration":0}
```

---

### 3.4 Anomaly 4 — a connection flagged "not reused" yet the connect timestamps equal the got‑connection timestamp

> **Your words:** *"sometimes a connection is flagged as \"not reused\" but the connect timestamps show the same value as the got connection timestamp, which should be impossible if a real TCP handshake occurred."*

**(a) Verdict: EXPECTED‑BY‑DESIGN — a guarded handling of a Go standard‑library quirk, not an impossible state.** It arises from a documented Go HTTP/2 bug where `httptrace.GotConnInfo.Reused` is *falsely* `false` for a connection that was actually reused. k6 detects the situation (no `ConnectStart`/`ConnectDone` fired, yet `GotConn` says not‑reused) and back‑fills the still‑zero timestamps to the got‑connection instant.

**(b) Mechanism.** The `else` (not‑reused) branch of `GotConn` guards exactly this case. Because the connect/TLS hooks never fired (the timestamps are still `0`), it `CompareAndSwap`s them from `0` to `now`, leaving `connectStart == connectDone == gotConn` while `connReused` remains `false`:

```go
	} else {
		// There's a bug in the Go stdlib where an HTTP/2 connection can be reused
		// but the httptrace.GotConnInfo struct will contain a false Reused property...
		// That's probably from a previously made connection that was abandoned and
		// directly put in the connection pool in favor of a just-freed already
		// established connection...
		//
		// Using CompareAndSwap here because the HTTP/2 roundtripper has retries and
		// it's possible this isn't actually the first request attempt...
		atomic.CompareAndSwapInt64(&t.connectStart, 0, now)
		atomic.CompareAndSwapInt64(&t.connectDone, 0, now)
		if isConnTLS {
			atomic.CompareAndSwapInt64(&t.tlsHandshakeStart, 0, now)
			atomic.CompareAndSwapInt64(&t.tlsHandshakeDone, 0, now)
		}
	}
```
(`lib/netext/httpext/tracer.go:L278-L293`; the two back‑fills are `L287-L288`.) `connReused` was set from `info.Reused` at `lib/netext/httpext/tracer.go:L262`, so it stays `false` here. The result — equal connect and got‑connection timestamps with `reused=false` — is therefore the **intended** output of this guard, not a contradiction: no real TCP handshake occurred (the stdlib mislabeled a reuse), so there is no handshake duration to record, and the guard prevents a bogus non‑zero `connecting`.

**(c) Demonstration and grounding.** This is an **HTTP/2‑specific** stdlib interaction. The local server negotiated **HTTP/1.1** (confirmed at runtime: `res.proto:"HTTP/1.1"`), so the branch is not exercised on the default local run; the following are the grounds, labelled accordingly:

- **Code (observed at HEAD):** the guard branch above at `lib/netext/httpext/tracer.go:L278-L293`.
- **(inferred — external):** the underlying stdlib behaviour is the class of Go issue `golang/go#27753`, where a reproduction with 10 concurrent HTTP/2 requests prints `TLSHandshakeStart` 10 times while `GotConn` reports `Reused: true` for 9 of 10 (the first is `Reused: false`) — i.e. the `Reused` flag is unreliable under HTTP/2 concurrency. k6's branch is the defensive response to exactly that.
- **Observed (indirect):** the `-race` suite in §3.3 passes with zero races, so even when this branch runs concurrently it is memory‑safe.

There is nothing to report upstream to k6: the k6 side already handles the stdlib quirk. (The stdlib issue itself is a Go matter, not a k6 defect.)

---

### 3.5 Anomaly 5 — `ConnectStart`/`ConnectDone` sometimes called multiple times; is it "a double counting bug"?

> **Your words:** *"ConnectStart and ConnectDone sometimes get called multiple times for a single request, which looks like a double counting bug."*

**(a) Verdict: the "double counting" hypothesis is REFUTED. EXPECTED‑BY‑DESIGN.** Multiple invocations are part of the Go `httptrace` **contract** under dual‑stack ("Happy Eyeballs") dialing, and the tracer **de‑duplicates** them: only the *first* timestamp is kept, so subsequent invocations are no‑ops. Nothing is counted twice.

**(b) Mechanism.** The hook doc records the contract, and the body uses `atomic.CompareAndSwapInt64(&t.connectStart, 0, now())` — which writes only while the field is still `0`, i.e. only on the first call:

```go
// ConnectStart is called when a new connection's Dial begins.
// If net.Dialer.DualStack (IPv6 "Happy Eyeballs") support is
// enabled (default), this may be called multiple times.
//
// If the connection is reused, this won't be called. Otherwise,
// it will be called after GetConn() and before ConnectDone().
func (t *Tracer) ConnectStart(_, _ string) {
	// If using dual-stack dialing, it's possible to get this
	// multiple times, so the atomic compareAndSwap ensures
	// that only the first call's time is recorded
	atomic.CompareAndSwapInt64(&t.connectStart, 0, now())
}
```
(`lib/netext/httpext/tracer.go:L191-L202`; the de‑dup is at `L201`.) `ConnectDone` applies the same discipline (and only on success):

```go
func (t *Tracer) ConnectDone(_, _ string, err error) {
	// If using dual-stack dialing, it's possible to get this
	// multiple times, so the atomic compareAndSwap ensures
	// that only the first call's time is recorded
	if err == nil {
		atomic.CompareAndSwapInt64(&t.connectDone, 0, now())
	}
	// if there is an error it either is happy eyeballs related and doesn't matter or it will be
	// returned by the http call
}
```
(`lib/netext/httpext/tracer.go:L213-L222`; the de‑dup is at `L218`.) Because `Connecting = connectDone - connectStart` (§3.1) uses only these first‑write values, a second or third invocation changes nothing — there is no accumulation and therefore no double counting.

**k6 makes multiple invocations rare in the default path.** k6's custom dialer resolves each host to a **single IP** before handing the address to `net/http`, so the standard "try IPv6 then IPv4" racing that triggers repeated `ConnectStart` does not normally occur:

```go
	ip, err = d.Resolver.LookupIP(host)
	if err != nil {
		return nil, err
	}

	if ip == nil {
		return nil, fmt.Errorf("lookup %s: no such host", host)
	}

	return types.NewHost(ip, port)
```
(`lib/netext/dialer.go:L140-L149`, inside `findRemote`, `lib/netext/dialer.go:L116`; entered from `DialContext`, `lib/netext/dialer.go:L59`.) A single resolved host means one dial target, so `ConnectStart` normally fires once. The `CompareAndSwap` guard is therefore a defensive measure for the dual‑stack edge case rather than something that trips on every request.

**(c) Demonstration and grounding.**

- **Code (observed at HEAD):** the de‑dup at `lib/netext/httpext/tracer.go:L201` and `L218`; the contract comment at `L191-L196`; the single‑IP dialer at `lib/netext/dialer.go:L140-L149`.
- **(inferred — external):** the Go `net/http/httptrace` `ClientTrace` documentation states verbatim that with dual‑stack enabled `ConnectStart`/`ConnectDone` "may be called multiple times" — confirming multiple calls are the contract, not a k6 defect.
- **Observed (indirect):** with k6's single‑IP dialer, the §3.1 run produced exactly one connect phase on the cold iteration and zero on reused ones, consistent with a single `ConnectStart`; and the `-race` suite (§3.3) shows the atomic de‑dup is memory‑safe under heavy concurrency.

Nothing to report upstream: multiple invocations are contractual and are correctly de‑duplicated.

---

### 3.6 The `sending` three‑way switch (for completeness)

`Tracer.Done` selects the `sending` baseline with a three‑way `switch`, whose `default` arm is the same HTTP/2 `GotConn`‑first / `Reused=false` case from Anomaly 4:

```go
	if wroteRequest != 0 {
		switch {
		case tlsHandshakeDone != 0:
			// If the request was sent over TLS, we need to use
			// TLS Handshake Done time to calculate sending duration
			trail.Sending = time.Duration(wroteRequest - tlsHandshakeDone)
		case connectDone != 0:
			// Otherwise, use the end of the normal connection
			trail.Sending = time.Duration(wroteRequest - connectDone)
		default:
			// Finally, this handles the strange HTTP/2 case where the GotConn() hook
			// gets called first, but with Reused=false
			trail.Sending = time.Duration(wroteRequest - gotConn)
		}
```
(`lib/netext/httpext/tracer.go:L346-L359`.) The totals are then `ConnDuration = Connecting + TLSHandshaking` (`lib/netext/httpext/tracer.go:L380`) and `Duration = Sending + Waiting + Receiving` (`lib/netext/httpext/tracer.go:L381`). In §3.1 the TLS arm was taken (`sending` non‑zero on every iteration, e.g. `0.130117` cold and `0.014653–0.025388` warm), consistent with the HTTP/1.1‑over‑TLS local server.

---


## 4. Per‑metric trust verdict

All seven **populated** timings are computed in `Tracer.Done` (`lib/netext/httpext/tracer.go:L315-L384`) and copied verbatim into `res.timings` via `metrics.D()` (`lib/netext/httpext/request.go:L98-L106`). Every one is **trustworthy**.

| `res.timings` field | Computed at (`file:line`) | What it measures | Trustworthy? | Notes (observed) |
|---------------------|---------------------------|------------------|:------------:|------------------|
| `blocked` | `tracer.go:L323-L325` | `gotConn − getConn`: connection‑acquisition time (DNS + TCP + TLS + pool wait) | **Yes** | **Bimodal**: cold ≈ 2.28–3.16 ms, warm ≈ 0.002–0.009 ms (§3.2). Near‑zero on idle‑pool checkout is correct. |
| `connecting` | `tracer.go:L340-L342` | `connectDone − connectStart`: TCP connect time | **Yes** | **Exactly `0` on reuse** (§3.1) because no new dial occurs; 24/24 reused samples = 0. |
| `tls_handshaking` | `tracer.go:L343-L345` | `tlsHandshakeDone − tlsHandshakeStart`: TLS handshake time | **Yes** | **Exactly `0` on reuse** (§3.1) because no new handshake occurs; 24/24 reused samples = 0. |
| `sending` | `tracer.go:L346-L359` | request‑write time (baseline chosen by the three‑way switch) | **Yes** | Non‑zero every iteration (e.g. `0.130117` cold, `0.014653` warm). `default` arm covers HTTP/2 `Reused=false`. |
| `waiting` | `tracer.go:L361-L372` | time‑to‑first‑response‑byte after the request was written (TTFB) | **Yes** | Non‑zero on reuse (`0.19–0.29` ms) — this is the "network activity" seen during Anomaly 1. |
| `receiving` | `tracer.go:L374-L376` | time to read the response body after first byte | **Yes** | Non‑zero on reuse (`0.04–0.08` ms). |
| `duration` | `tracer.go:L381` | `sending + waiting + receiving`: server round‑trip excluding connect | **Yes** | Matches `http_req_duration` in the summary (§3.1). |

**Not populated — always `0`:** `looking_up` is declared in the struct (`response.go:L38`) but **never written** by the population site (`request.go:L98-L106`), so `res.timings.looking_up` is a **structural constant `0`**, confirmed in every observed iteration. It is not a measurement and should not be interpreted as "DNS took 0 ms".

**Sink agreement.** Because `res.timings` (Sink B) and the `http_req_*` metrics (Sink A) are copied from the *same* `Trail`, they always agree. Observed in §3.1: `http_req_connecting` med `0s`/max `278.22µs` and `http_req_tls_handshaking` med `0s`/max `3.72ms` line up exactly with the per‑iteration `res.timings`.

**Overall:** trust the values. Interpret `0` on `connecting`/`tls_handshaking` as "connection reused" (not "instantaneous handshake"), interpret a small `blocked` as "idle connection reused", and interpret an all‑zero row as an **errored** request (check `res.error`/`res.error_code`/`http_req_failed`), or — on Windows only — a possible clock‑granularity zero.

---

## 5. Upstream recommendation

**Do not file an upstream bug report for any of the five anomalies.** All are by‑design:

- Anomalies 1 & 2 are direct consequences of HTTP keep‑alive connection reuse and the `httptrace` "reused connections don't fire dial/TLS hooks" contract.
- Anomaly 3's "race" is refuted by a clean `-race` run; the all‑zero rows are the error path (status 0) or Windows timer granularity.
- Anomaly 4 is k6's **existing workaround** for a Go stdlib HTTP/2 quirk (`tracer.go:L287-L288`) — already handled.
- Anomaly 5's "double counting" is refuted by the `CompareAndSwapInt64` de‑dup (`tracer.go:L201,L218`); multiple calls are the documented dual‑stack contract.

The single nuance (the Go HTTP/2 false‑`Reused` flag) is a **Go standard‑library** matter that k6 already defends against; there is no k6 defect to report.

---

## 6. Coverage pass

Confirming every named item, hook, condition, flag, and user example is addressed by name.

### 6.1 The eight `httptrace` hooks (`Tracer.Trace()`, `tracer.go:L162-L173`)

| Hook | Tracer method (`file:line`) | Write discipline | Addressed in |
|------|-----------------------------|------------------|--------------|
| `GetConn` | `GetConn` (`tracer.go:L187-L189`) | plain store `t.getConn` | §2, §3.2 (`blocked` start) |
| `ConnectStart` | `ConnectStart` (`tracer.go:L197-L202`) | `CompareAndSwap …,0,now` (dedup, `L201`) | §3.5 |
| `ConnectDone` | `ConnectDone` (`tracer.go:L213-L222`) | `CompareAndSwap …,0,now` if `err==nil` (dedup, `L218`) | §3.5 |
| `TLSHandshakeStart` | `TLSHandshakeStart` (`tracer.go:L230-L232`) | `CompareAndSwap …,0,now` (`L231`) | §3.1 |
| `TLSHandshakeDone` | `TLSHandshakeDone` (`tracer.go:L242-L247`) | `CompareAndSwap …,0,now` if `err==nil` (`L244`) | §3.1 |
| `GotConn` | `GotConn` (`tracer.go:L256-L294`) | reused → `SwapInt64`; not‑reused → `CompareAndSwap …,0,now` | §3.1, §3.4 |
| `WroteRequest` | `WroteRequest` (`tracer.go:L299-L304`) | `StoreInt64` if `err==nil` (`L301`) | §3.6 (`sending`) |
| `GotFirstResponseByte` | `GotFirstResponseByte` (`tracer.go:L310-L312`) | `CompareAndSwap …,0,now` (`L311`) | §4 (`waiting`/`receiving`) |

### 6.2 Structures, functions, and disciplines

| Item | `file:line` | Addressed in |
|------|-------------|--------------|
| `Tracer` struct (8 `int64` fields + `connReused`) | `tracer.go:L147-L159` | §2 |
| Fresh `Tracer` per request | `transport.go:L205` | §2 |
| "NOT safe to reuse Tracers" | `tracer.go:L145` | §2 |
| `Tracer.Done()` builds `Trail` | `tracer.go:L315-L384` | §2, §3, §4 |
| `Trail` → `http_req_*` (`SaveSamples`) | `transport.go:L146` | §2, §4 (sink agreement) |
| `Trail` → `res.timings` (`metrics.D()` ns→ms) | `request.go:L98-L106`, `units.go:L11-L13` | §2, §4 |
| `ResponseTimings` struct | `response.go:L34-L44`; `Timings` field `L65` | §2, §4 |
| Metric names + `Trend`/`Time` registration | `builtin.go:L18-L20,L92-L94` | §2 |
| `SwapInt64` (overwrite on reuse) vs `CompareAndSwapInt64` (keep first) | `tracer.go:L271-L277` vs `L201,L218,L287-L288` | §3.1, §3.4, §3.5 |
| Zero‑guards (`gotConn>getConn`; both endpoints non‑zero) | `tracer.go:L323,L340,L343` | §3.1, §3.2, §4 |
| Single‑IP dialer (`findRemote`/`LookupIP`/`NewHost`) | `dialer.go:L59,L116,L140-L149` | §3.5 |
| Base `net.Dialer` (`Timeout`/`KeepAlive` 30s) | `runner.go:L90-L93` | §2 |
| Late‑callback atomics comment | `tracer.go:L327-L331` | §3.3 |
| `-race` result (exit 0, 0 data races, 200 parallel cancelled reqs) | `tracer_test.go:L257-L292` | §3.3 |
| Windows error path (`WSAECONNRESET`→`tcpResetByPeerErrorCode` 1220) | `error_codes_syscall_windows.go:L10-L16`, `error_codes.go:L44` | §3.3 |
| Dial‑refused code (`tcpDialRefusedErrorCode` 1212) | `error_codes.go:L42` | §3.3 |
| Windows timer‑resolution hack (golang/go#8687, #41087) | `tracer_test.go:L33-L40` | §3.3 |
| Reuse zero‑asserts driven by `iterations []bool{false,true,true}` | `tracer_test.go:L115,L161-L165` | §3.1 |
| `sending` three‑way switch (incl. HTTP/2 default arm) | `tracer.go:L346-L359` | §3.6 |
| `looking_up` declared but never populated (always 0) | `response.go:L38` vs `request.go:L98-L106` | §2, §4 |
| Version banner (`Version`, `FullVersion`) | `consts.go:L12,L52` | §1 |

### 6.3 The five preserved user examples

| # | User example (verbatim, abbreviated) | Verdict | Section |
|---|--------------------------------------|---------|---------|
| 1 | "subsequent requests show exactly 0 for … connecting … and TLS handshaking … even though I can see network activity" | Expected‑by‑design (keep‑alive reuse) | §3.1 |
| 2 | "\"blocked\" timing shows massive values like 500ms … other times … near zero for identical requests" | Expected‑by‑design (`gotConn − getConn`; bimodal) | §3.2 |
| 3 | "occasionally ALL the timing metrics return 0 … definitely seems like a race" | **Race hypothesis refuted**; error‑path / Windows timer | §3.3 |
| 4 | "connection is flagged as \"not reused\" but the connect timestamps show the same value as the got connection timestamp" | Expected‑by‑design (guarded HTTP/2 stdlib quirk) | §3.4 |
| 5 | "ConnectStart and ConnectDone sometimes get called multiple times … looks like a double counting bug" | **Double‑counting hypothesis refuted** (dedup) | §3.5 |

### 6.4 Methodology confirmations

- **Run‑code‑first:** canonical `go build -o k6 .` then `./k6 run` against a local keep‑alive HTTPS server; version banner captured from that build (§1).
- **Scale & stability:** every magnitude states its scale (`vus:1, iterations:4`, repeated 8×); the reuse‑zero and bimodal‑`blocked` patterns were stable across all 8 runs, and the error‑path all‑zero across 2 runs.
- **Distribution reporting:** Anomaly 2 (`blocked`) and Anomaly 3 (all‑zero) were reproduced by repeating the **same unchanged input** and reporting the observed distribution (§3.2, §3.3).
- **Canonical entry point only:** every value is an observed `res.timings`/`http_req_*` from a real `k6 run`, or a real `go test` result. No debug hook or synthetic bypass was used.
- **Inference labelled:** the 500 ms real‑network scaling of `blocked`, the Windows timer‑resolution zeroing, and the HTTP/2 false‑`Reused`/dual‑stack external behaviours are marked **(inferred)**; everything else is observed.

