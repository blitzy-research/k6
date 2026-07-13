# Can k6's HTTP request-timing metrics be trusted? — A run-code-first investigation

> **Scope of this document.** You are debugging HTTP timing metrics in a k6 load test and suspect "a measurement bug or race condition" in the HTTP tracer. This document answers, per timing metric, whether `res.timings` can be trusted, and adjudicates each of your five reported anomalies as *expected-by-design* or *genuine bug*. Every value cited below was **observed at runtime** through the canonical k6 entry point (the compiled `k6` binary running a real script, or a real `go test` run). Statements derived from reading code or external documentation rather than from a runtime observation here are explicitly labelled **(inferred)**.

---

## 1. Bottom line (read this first)

1. **Can `res.timings` be trusted for performance analysis? — YES, for the paths demonstrated here (HTTP/1.1 and HTTP/2 keep-alive over TLS, cold and warm, plus the error/timeout paths), with one documented caveat.** Every populated timing (`blocked`, `connecting`, `tls_handshaking`, `sending`, `waiting`, `receiving`, `duration`) is computed correctly and is trustworthy on these paths. The values you see are the *true* consequences of HTTP keep-alive connection reuse and the Go `net/http/httptrace` hook contract — not measurement errors. **The one caveat**, which the k6 maintainers themselves acknowledge, is that a few HTTP/2 corner cases (chiefly stdlib-driven request *retries*) are not perfectly handled by the current nanosecond-timestamps-with-atomics `Tracer`; see §5. This does not affect the common paths measured here.

2. **Do any of the five anomalies warrant a *new* upstream bug report? — NO.** Four of the five are unambiguously by-design behaviour; the fifth (Anomaly 4) is a known Go standard-library HTTP/2 quirk that k6 **already works around in-tree** and that the maintainers **already track** (k6 issue #866, from fix PR #862). "No new report needed" is therefore because the corner case is *already known and tracked* — **not** because "no issue exists". Two of the five are hypotheses this investigation **refutes by running the code**:
   - The **"race in the measurement code"** (Anomaly 3): running the maintainers' tracer suite — including a 200-way parallel cancelled-request stress test — under the Go race detector produced exit `0` and **zero** `WARNING: DATA RACE` (`DATA_RACE_WARNINGS=0`). The precise, honest claim is therefore: **no Go data race was observed in this Linux `-race` run** (§3.3). The all-zero rows have non-race causes.
   - The **"double counting bug"** (Anomaly 5): the tracer's own de-duplication collapses multiple `ConnectStart`/`ConnectDone` invocations to the *first* timestamp via `atomic.CompareAndSwapInt64`, so later invocations are no-ops (§3.5).

### 1.1 Environment, provenance, and canonical build (grounds every observation)

All observations were produced by a normal-user build of k6 from the checked-out source. The commands and their **complete, unelided** output:

```text
$ git rev-parse --abbrev-ref HEAD
blitzy-33959ffe-5247-4d91-b493-c741bbdb5309
$ git rev-parse HEAD
6b4e9d228606860f42af56c69822e7d6d9623597
$ git log --oneline -3
6b4e9d228 docs: add k6 HTTP tracer timing investigation (blitzy/documentation/k6_ddc3b0b1d23c.md)
ddc3b0b1d Update comment
ee913c556 Refactor back to if/else instead of switch
$ git merge-base --is-ancestor ddc3b0b1d HEAD && echo "ddc3b0b1d IS an ancestor of HEAD"
ddc3b0b1d IS an ancestor of HEAD
$ git diff --stat ddc3b0b1d..HEAD
 blitzy/documentation/k6_ddc3b0b1d23c.md | 671 ++++++++++++++++++++++++++++++++
 1 file changed, 671 insertions(+)
$ git diff --name-status ddc3b0b1d..HEAD
A	blitzy/documentation/k6_ddc3b0b1d23c.md
$ go version
go version go1.23.12 linux/amd64
$ uname -a
Linux reverse-code-generator-95718474-z9fmj 6.6.122+ #1 SMP Thu Apr  2 09:59:00 UTC 2026 x86_64 GNU/Linux
$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
$ go env CGO_ENABLED GOTOOLCHAIN
1
local
```

**Provenance, stated precisely (this corrects any impression that `ddc3b0b1d…` is the current HEAD).**

| Item | Value |
|------|-------|
| Module | `go.k6.io/k6` |
| **Source baseline** commit (all `file:line` citations below are pinned to this tree) | `ddc3b0b1d23c128e34e2792fc9075f9126e32375` — message `Update comment` |
| **Delivery HEAD** (branch tip that carries this document) | `6b4e9d228606860f42af56c69822e7d6d9623597` — message `docs: add k6 HTTP tracer timing investigation` |
| Relationship | `ddc3b0b1d` **is an ancestor of** HEAD; the only delta `ddc3b0b1d..HEAD` is **this markdown file** (`1 file changed, 671 insertions(+)`, status `A`). No `.go` file differs, so the runtime behaviour observed here is byte-for-byte the behaviour of the `ddc3b0b1d` baseline. |
| Working branch | `blitzy-33959ffe-5247-4d91-b493-c741bbdb5309` |
| Go toolchain | `go version go1.23.12 linux/amd64` |
| OS / arch | `Linux reverse-code-generator-95718474-z9fmj 6.6.122+ #1 SMP Thu Apr  2 09:59:00 UTC 2026 x86_64 GNU/Linux` |
| Race detector | `CGO_ENABLED=1` + `gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0` (environment-only; not a repo dependency) |

> **Note on the moving branch tip.** The commands above were captured during the investigation, when the branch tip was `6b4e9d228…` — the commit that introduced the *first* version of this document. Committing this *corrected* version adds one further **docs-only** commit on top, so a later `git rev-parse HEAD` will report a different hash and `git diff --stat ddc3b0b1d..HEAD` will show more than the `671` insertions of that first version. The **stable invariant** behind every observation is unaffected: `git diff --name-status ddc3b0b1d..HEAD` touches **only** `blitzy/documentation/k6_ddc3b0b1d23c.md` — no `.go`, test, configuration, or vendored file differs from the `ddc3b0b1d` baseline — so the runtime behaviour observed here is byte-for-byte the `ddc3b0b1d` tracer's behaviour regardless of the exact tip.

**Canonical build and the *actual observed* version banner.** The build was run in-place on the clean tree at HEAD:

```text
$ go build -o k6 .
$ ./k6 version
k6 v0.55.0 (commit/6b4e9d2286, go1.23.12, linux/amd64)
```

The banner shows `commit/6b4e9d2286`, **not** `commit/ddc3b0b1d2`, because Go stamps the banner from the build's VCS metadata (the current HEAD), which the following build-info confirms:

```text
$ go version -m ./k6 | grep -E "(^|[[:space:]])mod[[:space:]]|vcs\."
	mod	go.k6.io/k6	(devel)	
	build	vcs.revision=6b4e9d228606860f42af56c69822e7d6d9623597
	build	vcs.time=2026-07-13T16:45:01Z
	build	vcs.modified=false
```

This banner shape is produced by `FullVersion()`, which reads `debug.ReadBuildInfo()`, takes the first 10 characters of the `vcs.revision` setting as the commit, appends `-dirty` if `vcs.modified` is `true`, and formats `fmt.Sprintf("%s (commit/%s, %s)", Version, commit, goVersionArch)` (`lib/consts/consts.go:L52`) with `Version = "0.55.0"` (`lib/consts/consts.go:L12`). Because `vcs.modified=false`, there is no `-dirty` suffix. Since the Go source tree at HEAD is identical to the `ddc3b0b1d` baseline (the diff above is docs-only), this HEAD build is the canonical binary for observing the `ddc3b0b1d` tracer behaviour.

---

## 2. How the timing pipeline works (so the answers make sense)

Every timing value in `res.timings` and every `http_req_*` metric originates from the same object — a `Trail` produced by a per-request `Tracer`. Understanding this one-to-one flow is what makes all five anomalies obviously by-design.

**Step 1 — a *fresh* `Tracer` is created for every request** and attached to the request context. Tracers are never reused, which is why there is no cross-request state to corrupt:

```go
	ctx := req.Context()
	tracer := &Tracer{}
	reqWithTracer := req.WithContext(httptrace.WithClientTrace(ctx, tracer.Trace()))
	resp, err := t.state.Transport.RoundTrip(reqWithTracer)
```
(`lib/netext/httpext/transport.go:L203-L207`; the fresh instance is at `L205`.) The tracer's own doc comment states the rule: `// It's NOT safe to reuse Tracers between requests.` (`lib/netext/httpext/tracer.go:L145`).

**Step 2 — `Tracer.Trace()` wires all eight `httptrace` hooks** (`lib/netext/httpext/tracer.go:L162-L173`). Each hook records a Unix-nanosecond timestamp into an `int64` field of the `Tracer` struct (`lib/netext/httpext/tracer.go:L147-L159`) using `sync/atomic`.

**Step 3 — after the round-trip, `Tracer.Done()` builds the `Trail`:**

```go
func (t *transport) measureAndEmitMetrics(unfReq *unfinishedRequest) *finishedRequest {
	trail := unfReq.tracer.Done()
```
(`lib/netext/httpext/transport.go:L77-L78`.)

**Step 4 — the *same* `Trail` feeds two sinks, which is why script-visible `res.timings` and aggregated `http_req_*` metrics always agree.**

- **Sink A — `http_req_*` samples:** `trail.SaveSamples(t.state.BuiltinMetrics, &tagsAndMeta)` (`lib/netext/httpext/transport.go:L146`). `Trail.SaveSamples` builds one sample **per metric for all seven timings plus `http_reqs`** (`lib/netext/httpext/tracer.go:L43-L122`): it appends `HTTPReqs`, then `HTTPReqDuration`, `HTTPReqBlocked`, `HTTPReqConnecting`, `HTTPReqTLSHandshaking`, `HTTPReqSending`, `HTTPReqWaiting`, `HTTPReqReceiving`, each with `Value: metrics.D(tr.<Field>)`. The eight metric names are declared at `metrics/builtin.go:L16-L23` and all seven timing metrics are registered as `Trend`/`Time` metrics at `metrics/builtin.go:L91-L97`.
- **Sink B — `res.timings`:** copied field-by-field into the JS-facing struct:

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

**The user-facing struct** the user reads as `res.timings` is `ResponseTimings` (`lib/netext/httpext/response.go:L34-L44`), exposed on the response as `Timings ResponseTimings` with json tag `"timings"` (`lib/netext/httpext/response.go:L65`).

> **Structural note used later:** `ResponseTimings` declares a `LookingUp float64` field with json tag `"looking_up"` (`lib/netext/httpext/response.go:L38`), but the population site in `request.go:L98-L106` **omits it**. Therefore `res.timings.looking_up` is **always `0`** — a structural constant, not a measurement. This was confirmed at runtime (`looking_up:0` in every iteration below).

### 2.1 What actually enables connection reuse (TCP keep-alive vs HTTP keep-alive)

This distinction matters for Anomalies 1 and 2, so it is worth being precise — the base dialer's `KeepAlive` field is **not** the mechanism that makes k6 reuse connections.

- **`net.Dialer.KeepAlive` is TCP keep-alive *probes*.** k6's base dialer is configured as:

```go
		BaseDialer: net.Dialer{
			Timeout:   30 * time.Second,
			KeepAlive: 30 * time.Second,
		},
```
(`js/runner.go:L90-L93`.) This `KeepAlive: 30s` sets the interval of TCP-level keep-alive **probe packets** on an established socket (to detect dead peers); it does **not** decide whether an HTTP request reuses a pooled connection. **(inferred — Go `net` docs.)**

- **HTTP connection reuse is governed by `http.Transport`.** The transport k6 builds controls reuse via `DisableKeepAlives` and the idle-connection pool:

```go
	transport := &http.Transport{
		Proxy:               http.ProxyFromEnvironment,
		TLSClientConfig:     tlsConfig,
		DialContext:         dialer.DialContext,
		DisableCompression:  true,
		DisableKeepAlives:   r.Bundle.Options.NoConnectionReuse.Bool,
		MaxIdleConns:        int(r.Bundle.Options.Batch.Int64),
		MaxIdleConnsPerHost: int(r.Bundle.Options.BatchPerHost.Int64),
	}
```
(`js/runner.go:L193-L201`; `DisableKeepAlives` at `L198`, the idle-pool sizing at `L199-L200`.) When `DisableKeepAlives` is `false` (the default — i.e. `--no-connection-reuse` off), a completed connection is returned to the idle pool and the *next* same-host request reuses it.

- **Reuse additionally requires the response body to be fully read and closed.** k6 reads the body to EOF and closes it, which is what allows the Go client to put the connection back in the idle pool:

```go
	rc := &readCloser{resp.Body}
	// Ensure that the entire response body is read and closed, e.g. in case of decoding errors
	defer func(respBody io.ReadCloser) {
		_, _ = io.Copy(io.Discard, respBody)
		_ = respBody.Close()
	}(resp.Body)
```
(`lib/netext/httpext/compression.go:L134-L139`, inside `readResponseBody` at `L118`; the `ResponseTypeNone` branch does the same `io.Copy(io.Discard, …)`+`Close()` at `L129-L130`.) **(inferred — Go `net/http` docs:** a keep-alive connection is only reusable after its response body is drained to EOF and closed.)

The end-to-end flow, and where each anomaly is produced:

```mermaid
flowchart TD
    A["VU runs http.get in a k6 script"] --> B["transport.RoundTrip (transport.go:L200)"]
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

The complete, self-contained observation harness (server + script sources, SHA-256 hashes, and exact start/stop commands) is in the **Appendix (§7)**; each demonstration below references it.

---

## 3. Per-anomaly adjudication

Each section leads with the **direct verdict**, then the **causal mechanism** (exact function/struct + `file:line`, code quoted faithfully), then the **runtime demonstration** (exact command + complete unedited output).

### 3.1 Anomaly 1 — `connecting` and `tls_handshaking` are exactly `0` on subsequent requests

> **Your words:** *"the first request shows reasonable values for connecting time and TLS handshaking, but subsequent requests show exactly 0 for both of these metrics even though I can see network activity happening."*

**(a) Verdict: EXPECTED-BY-DESIGN.** A reused keep-alive connection performs **no new TCP dial and no new TLS handshake**, so the `ConnectStart`/`ConnectDone`/`TLSHandshake*` hooks never fire for it. k6 deliberately reports `0` for those phases, while `waiting`/`receiving` stay non-zero — that non-zero waiting/receiving **is** the "network activity" you correctly observe. This is trustworthy, not a bug.

**(b) Mechanism.** When the connection is reused, `Tracer.GotConn` is the *first* hook called, and it overwrites the connect/TLS timestamps to the same `now` instant so the (never-fired) hooks cannot leave stale values:

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

The `httptrace` contract that a reused connection does not fire these hooks is documented on the hooks themselves — e.g. `// If the connection is reused, this won't be called.` for `ConnectStart` (`lib/netext/httpext/tracer.go:L195`) and for the TLS hooks (`lib/netext/httpext/tracer.go:L228`). **(inferred — Go `net/http/httptrace` docs and HTTP/1.1 keep-alive semantics:** a reused persistent connection performs no new dial or handshake.)

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

**(c) Demonstration.** Local HTTP/1.1 keep-alive HTTPS server (`(*httptest.Server).StartTLS()`, self-signed, bound to `127.0.0.1:18443`; full source + SHA-256 in §7), single VU, 4 sequential iterations to the same host. Scale: `vus:1, iterations:4`; the reuse pattern was stable across 8 repeats (§3.2). The console line reports `res.proto`, confirming the protocol was HTTP/1.1 — so this is a direct observation of HTTP/1.1 keep-alive reuse. Complete, unedited output of the run:

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

time="2026-07-13T17:22:33Z" level=info msg="{\"iter\":0,\"proto\":\"HTTP/1.1\",\"status\":200,\"blocked\":2.334634,\"connecting\":0.188522,\"tls_handshaking\":2.059836,\"sending\":0.043956,\"waiting\":0.139227,\"receiving\":0.08754,\"duration\":0.270723,\"looking_up\":0}" source=console
time="2026-07-13T17:22:33Z" level=info msg="{\"iter\":1,\"proto\":\"HTTP/1.1\",\"status\":200,\"blocked\":0.003408,\"connecting\":0,\"tls_handshaking\":0,\"sending\":0.009444,\"waiting\":0.102632,\"receiving\":0.034596,\"duration\":0.146672,\"looking_up\":0}" source=console
time="2026-07-13T17:22:33Z" level=info msg="{\"iter\":2,\"proto\":\"HTTP/1.1\",\"status\":200,\"blocked\":0.003265,\"connecting\":0,\"tls_handshaking\":0,\"sending\":0.006067,\"waiting\":0.137621,\"receiving\":0.029304,\"duration\":0.172992,\"looking_up\":0}" source=console
time="2026-07-13T17:22:33Z" level=info msg="{\"iter\":3,\"proto\":\"HTTP/1.1\",\"status\":200,\"blocked\":0.002908,\"connecting\":0,\"tls_handshaking\":0,\"sending\":0.005778,\"waiting\":0.165527,\"receiving\":0.057033,\"duration\":0.228338,\"looking_up\":0}" source=console

     data_received..................: 2.1 kB 510 kB/s
     data_sent......................: 737 B  182 kB/s
     http_req_blocked...............: avg=586.05µs min=2.9µs    med=3.33µs   max=2.33ms   p(90)=1.63ms   p(95)=1.98ms  
     http_req_connecting............: avg=47.13µs  min=0s       med=0s       max=188.52µs p(90)=131.96µs p(95)=160.24µs
     http_req_duration..............: avg=204.68µs min=146.67µs med=200.66µs max=270.72µs p(90)=258µs    p(95)=264.36µs
       { expected_response:true }...: avg=204.68µs min=146.67µs med=200.66µs max=270.72µs p(90)=258µs    p(95)=264.36µs
     http_req_failed................: 0.00%  0 out of 4
     http_req_receiving.............: avg=52.11µs  min=29.3µs   med=45.81µs  max=87.54µs  p(90)=78.38µs  p(95)=82.96µs 
     http_req_sending...............: avg=16.31µs  min=5.77µs   med=7.75µs   max=43.95µs  p(90)=33.6µs   p(95)=38.77µs 
     http_req_tls_handshaking.......: avg=514.95µs min=0s       med=0s       max=2.05ms   p(90)=1.44ms   p(95)=1.75ms  
     http_req_waiting...............: avg=136.25µs min=102.63µs med=138.42µs max=165.52µs p(90)=157.63µs p(95)=161.58µs
     http_reqs......................: 4      984.841565/s
     iteration_duration.............: avg=977.7µs  min=266.95µs med=296.89µs max=3.05ms   p(90)=2.23ms   p(95)=2.64ms  
     iterations.....................: 4      984.841565/s


running (00m00.0s), 0/1 VUs, 4 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.0s/10m0s  4/4 shared iters
```

**Reading the output:** iteration 0 (new connection) shows `connecting=0.188522` ms and `tls_handshaking=2.059836` ms; iterations 1–3 (reused, `proto:"HTTP/1.1"`) show **exactly `0`** for both, while `waiting` stays `0.10–0.17` ms and `receiving` stays `0.03–0.06` ms — real network activity with zero connect/TLS time. The summary `http_req_connecting` (med `0s`, max `188.52µs`) and `http_req_tls_handshaking` (med `0s`, max `2.05ms`) exactly match the per-iteration `res.timings` — **proving Sink A and Sink B agree**. `looking_up` is `0` in every line. The same reuse-zero pattern also holds over **HTTP/2** (see §3.5(c), single-VU run: iter 0 cold, iters 1–3 `connecting=0`/`tls_handshaking=0`).

---

### 3.2 Anomaly 2 — `blocked` is sometimes large, sometimes near zero for "identical" requests

> **Your words:** *"sometimes the \"blocked\" timing shows massive values like 500ms when other times it's near zero for identical requests."*

**(a) Verdict: EXPECTED-BY-DESIGN.** `blocked` is the time spent *acquiring a connection*. On a **cold** request it includes the new DNS + TCP + TLS work; on a **warm** request it is just an idle-pool checkout, which is near zero. The requests are not truly identical: the first pays for the connection, the rest inherit it. The bimodal distribution is the correct measurement of two different situations.

**(b) Mechanism.** `Tracer.Done` computes `Blocked` as the gap between asking for a connection (`GetConn` → `getConn`) and getting one (`GotConn` → `gotConn`):

```go
	if t.gotConn != 0 && t.getConn != 0 && t.gotConn > t.getConn {
		trail.Blocked = time.Duration(t.gotConn - t.getConn)
	}
```
(`lib/netext/httpext/tracer.go:L323-L325`.) `getConn` is set by `GetConn` (`lib/netext/httpext/tracer.go:L187-L189`) and `gotConn` by `GotConn` (`lib/netext/httpext/tracer.go:L261`). The explicit `gotConn > getConn` guard is *why* you get a clean `0`/near-zero rather than a negative or garbage value on reuse.

**(c) Demonstration — observed distribution of the SAME unchanged input.** The **same unchanged script** from §3.1 was run **8 times** through a wrapper (`/tmp/obs/run_blocked.sh`, full source + SHA-256 in §7). Each `k6 run` is a fresh process, so iteration 0 is always cold (new connection) and iterations 1–3 are always warm (reused). **Every row below is emitted by the wrapper from k6's real `console.log` output — nothing is hand-entered.** Command and complete, unedited wrapper output:

```text
$ RUNS=8 bash /tmp/obs/run_blocked.sh
----- RUN 1 -----
  iter=0 cold(new)   blocked=2.295794   connecting=0.122556 tls_handshaking=2.114937
  iter=1 warm(reuse) blocked=0.004391   connecting=0        tls_handshaking=0       
  iter=2 warm(reuse) blocked=0.003018   connecting=0        tls_handshaking=0       
  iter=3 warm(reuse) blocked=0.005759   connecting=0        tls_handshaking=0       
----- RUN 2 -----
  iter=0 cold(new)   blocked=2.254987   connecting=0.118464 tls_handshaking=2.074127
  iter=1 warm(reuse) blocked=0.006219   connecting=0        tls_handshaking=0       
  iter=2 warm(reuse) blocked=0.003072   connecting=0        tls_handshaking=0       
  iter=3 warm(reuse) blocked=0.006649   connecting=0        tls_handshaking=0       
----- RUN 3 -----
  iter=0 cold(new)   blocked=2.276487   connecting=0.149056 tls_handshaking=2.043619
  iter=1 warm(reuse) blocked=0.005334   connecting=0        tls_handshaking=0       
  iter=2 warm(reuse) blocked=0.003214   connecting=0        tls_handshaking=0       
  iter=3 warm(reuse) blocked=0.00367    connecting=0        tls_handshaking=0       
----- RUN 4 -----
  iter=0 cold(new)   blocked=2.61411    connecting=0.141779 tls_handshaking=2.384472
  iter=1 warm(reuse) blocked=0.003635   connecting=0        tls_handshaking=0       
  iter=2 warm(reuse) blocked=0.003501   connecting=0        tls_handshaking=0       
  iter=3 warm(reuse) blocked=0.003519   connecting=0        tls_handshaking=0       
----- RUN 5 -----
  iter=0 cold(new)   blocked=2.218474   connecting=0.119442 tls_handshaking=2.031886
  iter=1 warm(reuse) blocked=0.002546   connecting=0        tls_handshaking=0       
  iter=2 warm(reuse) blocked=0.0028     connecting=0        tls_handshaking=0       
  iter=3 warm(reuse) blocked=0.001852   connecting=0        tls_handshaking=0       
----- RUN 6 -----
  iter=0 cold(new)   blocked=2.325567   connecting=0.125629 tls_handshaking=2.134099
  iter=1 warm(reuse) blocked=0.005141   connecting=0        tls_handshaking=0       
  iter=2 warm(reuse) blocked=0.00329    connecting=0        tls_handshaking=0       
  iter=3 warm(reuse) blocked=0.005737   connecting=0        tls_handshaking=0       
----- RUN 7 -----
  iter=0 cold(new)   blocked=2.267556   connecting=0.131316 tls_handshaking=2.06168 
  iter=1 warm(reuse) blocked=0.002924   connecting=0        tls_handshaking=0       
  iter=2 warm(reuse) blocked=0.002684   connecting=0        tls_handshaking=0       
  iter=3 warm(reuse) blocked=0.002457   connecting=0        tls_handshaking=0       
----- RUN 8 -----
  iter=0 cold(new)   blocked=2.850907   connecting=0.138489 tls_handshaking=2.625825
  iter=1 warm(reuse) blocked=0.003243   connecting=0        tls_handshaking=0       
  iter=2 warm(reuse) blocked=0.011873   connecting=0        tls_handshaking=0       
  iter=3 warm(reuse) blocked=0.00277    connecting=0        tls_handshaking=0       
```

Aggregate statistics **(derived from the 32 raw rows above)** — 8 runs × 4 iterations = **32 samples** (8 cold + 24 warm):

| Group | n | min (ms) | max (ms) | mean (ms) |
|-------|---|----------|----------|-----------|
| **COLD** (iter 0, new connection) | 8 | 2.218474 | 2.850907 | 2.387985 |
| **WARM** (iters 1–3, reused) | 24 | 0.001852 | 0.011873 | 0.004137 |

- `mean(cold) / mean(warm) = 577.2×`. Every cold sample `> 2.0 ms`; every warm sample `< 0.02 ms`.
- The **bimodal split is stable across all 8 runs** — cold is always in the ~2.2–2.9 ms band, warm always sub-0.02 ms. This is exactly the "sometimes big, sometimes near zero" you report.
- As a bonus, `connecting == 0` and `tls_handshaking == 0` held on **all 24/24 reused samples**, re-confirming Anomaly 1's stability.

**On the "500 ms" magnitude:** locally the cold value is a few ms because DNS is trivial and the peer is loopback. **(inferred)** Over a real network the *same* `gotConn - getConn` formula also includes real DNS resolution, the TCP three-way handshake, and the full TLS handshake to a remote host, which routinely sums to hundreds of milliseconds. The formula is identical; only the network cost differs. So a 500 ms cold `blocked` next to a near-zero warm `blocked` is the expected, correct behaviour — not a defect.

---

### 3.3 Anomaly 3 — on Windows, ALL timings occasionally return `0`; is it "a race in the measurement code"?

> **Your words:** *"occasionally ALL the timing metrics return 0 for random requests, which definitely seems like a race in the measurement code."*

**(a) Verdict: the "race" hypothesis is NOT SUPPORTED by any observation here, and an all-zero row is EXPECTED-BY-DESIGN in specific circumstances.** Precisely: **no Go data race was observed in this Linux `-race` run** (§(c) below). An **all-seven-zero** row occurs only when a request **fails before the timestamps it needs are recorded** — most clearly a *connection refusal*, where no hook fires. It does **not** occur for every error: a request that fails *after* connecting (a mid-flight timeout) keeps its partial timings (§(c) mid-flight run). The Windows-specific "random all-zero on otherwise-fine requests" is a **timer-resolution** artefact the maintainers document; it was **not reproduced on this Linux host** (labelled inferred below).

**(b) Causal mechanism.**

*Why "race" is not the right label.* The tracer is deliberately **lock-free**, using `sync/atomic` on `int64` fields *precisely because* hooks can fire after `Done()` has already returned — the safe, intended handling of late callbacks:

```go
	// It's possible for some of the methods of httptrace.ClientTrace to
	// actually be called after the http.Client or http.RoundTripper have
	// already returned our result and we've called Done(). This happens
	// mostly for cancelled requests, but we have to use atomics here as
	// well (or use global Tracer locking) so we can avoid data races.
```
(`lib/netext/httpext/tracer.go:L327-L331`.) In `Done`, every field is read with `atomic.LoadInt64` (`lib/netext/httpext/tracer.go:L332-L338`). **(inferred — Go `net/http/httptrace` docs:** hook functions "may be called concurrently from different goroutines and some may be called after the request has completed or failed" — exactly what these atomics cover.)

*Real cause (i) — a failure that precedes the needed timestamps.* When a request never establishes a connection (e.g. connection refused), the `GetConn`→`GotConn`→`ConnectStart/Done`→… hooks that would set the timestamps never fire, so `Done` returns a `Trail` whose fields are all zero and the response carries status `0`. `error_code 1212` is `tcpDialRefusedErrorCode` (`lib/netext/httpext/error_codes.go:L42`). Note carefully: the Windows connection-reset path only **maps an error code** — it does **not** clear any `Trail`:

```go
func getOSSyscallErrorCode(e *net.OpError, se *os.SyscallError) (errCode, string) {
	switch se.Unwrap() {
	case syscall.WSAECONNRESET:
		return tcpResetByPeerErrorCode, fmt.Sprintf(tcpResetByPeerErrorCodeMsg, e.Op)
	}
	return 0, ""
}
```
(`lib/netext/httpext/error_codes_syscall_windows.go:L10-L16`; `tcpResetByPeerErrorCode = 1220`, `error_codes.go:L44`.) So the all-zero row is a *consequence of which hooks fired*, not of the error-code mapping, and **not** a blanket "any error ⇒ all-zero".

*Real cause (ii) — Windows timer resolution.* The maintainers document this directly in their own test hack — on Windows two hooks can register the *same* timestamp, so a phase computes to `0`:

```go
	if runtime.GOOS == "windows" {
		// HACK: Time resolution is not as accurate on Windows, see:
		//  https://github.com/golang/go/issues/8687
		//  https://github.com/golang/go/issues/41087
		// Which seems to be causing some metrics to have a value of 0,
		// since e.g. ConnectStart and ConnectDone could register the same time.
```
(`lib/netext/httpext/tracer_test.go:L33-L40`; the test inserts a `traceDelay = 100 * time.Millisecond` sleep, `tracer_test.go:L28`, to space the hooks apart.) **(inferred — not observed here, as this run is on Linux/amd64):** on a low-resolution Windows clock this can zero a phase for a *successful* request too, which matches the "random requests" you saw on Windows. This is a platform clock-granularity artefact, not a race and not a k6 calculation error.

**(c) Demonstration.**

*(c.1) Refuting the "race" — run the tracer suite under `-race`.* The maintainers' `TestCancelledRequest` fires **200 parallel cancelled requests** — the precise concurrency that would surface a measurement race — but note it only asserts that a round-trip returned *either* a response *or* an error, **not** that any timing is correct:

```go
		resp, err := srv.Client().Transport.RoundTrip(req) //nolint:bodyclose
		_ = tracer.Done()
		if resp == nil && err == nil {
			t.Errorf("Expected either a RoundTrip response or error but got %#v and %#v", resp, err)
		}
```
(`lib/netext/httpext/tracer_test.go:L275-L279`; the 200-way parallel loop is at `L284-L291`.) Its value here is therefore as a **race-detector stress harness** (does the concurrent tracer trip `-race`?), not as a timing-correctness oracle.

Both runs were executed through a wrapper (`/tmp/obs/run_tests.sh`, full source + SHA-256 in §7) that prints the **real** `$?` of each `go test` and the **real** `grep -c "WARNING: DATA RACE"` count. The wrapper output below is an **excerpt**, not the full listing: the 200 identical `TestCancelledRequest/group/*` `--- PASS` lines (and the `TestTracer/Test_#0..#2` subtests) are collapsed to a single clearly-marked `...` line, while every non-repetitive line — the `=== RUN`/`=== PAUSE` headers, all four top-level `--- PASS` lines, `PASS`, the `ok` banner, and the wrapper's genuine `*_EXIT`/`DATA_RACE_WARNINGS` lines — is quoted verbatim. The complete listings (829 lines for the normal run, 905 for the race run) are saved at `/tmp/obs/test_normal.raw.log` and `/tmp/obs/test_race.raw.log`:

```text
$ bash /tmp/obs/run_tests.sh
### NORMAL RUN
$ go test -count=1 ./lib/netext/httpext/ -run "TestTracer|TestTracerError|TestTracerNegativeHttpSendingValues|TestCancelledRequest" -v
=== RUN   TestTracer
=== PAUSE TestTracer
=== RUN   TestTracerNegativeHttpSendingValues
=== PAUSE TestTracerNegativeHttpSendingValues
=== RUN   TestTracerError
=== PAUSE TestTracerError
=== RUN   TestCancelledRequest
=== PAUSE TestCancelledRequest
... (TestTracer/Test_#0..#2 subtests and 200 TestCancelledRequest/group/* subtests, each --- PASS) ...
--- PASS: TestTracer (0.01s)
--- PASS: TestTracerNegativeHttpSendingValues (0.01s)
--- PASS: TestTracerError (0.06s)
--- PASS: TestCancelledRequest (0.00s)
PASS
ok  	go.k6.io/k6/lib/netext/httpext	0.180s
NORMAL_EXIT=0

### RACE RUN
$ CGO_ENABLED=1 go test -race -count=1 ./lib/netext/httpext/ -run "TestTracer|TestTracerError|TestTracerNegativeHttpSendingValues|TestCancelledRequest" -v
=== RUN   TestTracer
=== PAUSE TestTracer
=== RUN   TestTracerNegativeHttpSendingValues
=== PAUSE TestTracerNegativeHttpSendingValues
=== RUN   TestTracerError
=== PAUSE TestTracerError
=== RUN   TestCancelledRequest
=== PAUSE TestCancelledRequest
... (same subtests, each --- PASS; interleaved with benign "http: TLS handshake error ... use of closed network connection" server logs from the cancelled requests) ...
--- PASS: TestTracer (0.03s)
--- PASS: TestTracerNegativeHttpSendingValues (0.04s)
--- PASS: TestTracerError (1.50s)
--- PASS: TestCancelledRequest (0.42s)
PASS
ok  	go.k6.io/k6/lib/netext/httpext	3.302s
RACE_EXIT=0
DATA_RACE_WARNINGS=0
```

**Result:** `NORMAL_EXIT=0`, `RACE_EXIT=0`, and `DATA_RACE_WARNINGS=0` — **not a single `WARNING: DATA RACE`** even under 200 concurrent cancelled requests. Precise conclusion: **no Go data race was observed in this Linux `-race` run.** (This is a strong negative result for the race hypothesis on this platform; it is not a proof that a timing value can never be affected by clock granularity on another platform — see cause (ii).)

*(c.2) The all-zero row — connection refused.* Pointing the same-shaped script at a closed port (nothing listening on `127.0.0.1:65000`) makes every request fail *before connecting*, so the emitted `Trail` is all zero (status 0). Complete, unedited output:

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

time="2026-07-13T17:23:20Z" level=warning msg="Request Failed" error="Get \"https://127.0.0.1:65000/\": dial tcp 127.0.0.1:65000: connect: connection refused"
time="2026-07-13T17:23:20Z" level=info msg="{\"iter\":0,\"proto\":\"\",\"status\":0,\"error\":\"dial: connection refused\",\"error_code\":1212,\"blocked\":0,\"connecting\":0,\"tls_handshaking\":0,\"sending\":0,\"waiting\":0,\"receiving\":0,\"duration\":0}" source=console
time="2026-07-13T17:23:20Z" level=warning msg="Request Failed" error="Get \"https://127.0.0.1:65000/\": dial tcp 127.0.0.1:65000: connect: connection refused"
time="2026-07-13T17:23:20Z" level=info msg="{\"iter\":1,\"proto\":\"\",\"status\":0,\"error\":\"dial: connection refused\",\"error_code\":1212,\"blocked\":0,\"connecting\":0,\"tls_handshaking\":0,\"sending\":0,\"waiting\":0,\"receiving\":0,\"duration\":0}" source=console
time="2026-07-13T17:23:20Z" level=warning msg="Request Failed" error="Get \"https://127.0.0.1:65000/\": dial tcp 127.0.0.1:65000: connect: connection refused"
time="2026-07-13T17:23:20Z" level=info msg="{\"iter\":2,\"proto\":\"\",\"status\":0,\"error\":\"dial: connection refused\",\"error_code\":1212,\"blocked\":0,\"connecting\":0,\"tls_handshaking\":0,\"sending\":0,\"waiting\":0,\"receiving\":0,\"duration\":0}" source=console

     data_received..............: 0 B     0 B/s
     data_sent..................: 0 B     0 B/s
     http_req_blocked...........: avg=0s       min=0s      med=0s       max=0s       p(90)=0s       p(95)=0s      
     http_req_connecting........: avg=0s       min=0s      med=0s       max=0s       p(90)=0s       p(95)=0s      
     http_req_duration..........: avg=0s       min=0s      med=0s       max=0s       p(90)=0s       p(95)=0s      
     http_req_failed............: 100.00% 3 out of 3
     http_req_receiving.........: avg=0s       min=0s      med=0s       max=0s       p(90)=0s       p(95)=0s      
     http_req_sending...........: avg=0s       min=0s      med=0s       max=0s       p(90)=0s       p(95)=0s      
     http_req_tls_handshaking...: avg=0s       min=0s      med=0s       max=0s       p(90)=0s       p(95)=0s      
     http_req_waiting...........: avg=0s       min=0s      med=0s       max=0s       p(90)=0s       p(95)=0s      
     http_reqs..................: 3       2053.092985/s
     iteration_duration.........: avg=441.18µs min=364.4µs med=400.55µs max=558.59µs p(90)=526.99µs p(95)=542.79µs
     iterations.................: 3       2053.092985/s


running (00m00.0s), 0/1 VUs, 3 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.0s/10m0s  3/3 shared iters
```

Every iteration: `proto:""`, `status:0`, `error_code:1212`, and **all seven timings `0`**; `http_req_failed` is `100.00%`. A second identical run produced byte-identical console lines (stable across ≥2 runs).

*(c.3) A failed request that is NOT all-zero — mid-flight timeout (this is the important refinement).* Here the connection **and** TLS handshake **and** request-write all succeed; the server (`/delay`) withholds the first response byte for 2 s, but the client times out after 250 ms. The request fails (`status:0`, `error_code:1050` = `requestTimeoutErrorCode`, `error_codes.go:L30`) yet keeps **non-zero** `blocked`, `connecting`, `tls_handshaking`, `sending`, `waiting`, and `duration` — only `receiving` is `0` (no body byte arrived). Complete, unedited output:

```text
$ K6_NO_USAGE_REPORT=true ./k6 run /tmp/obs/script_timeout.js

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/obs/script_timeout.js
        output: -

     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 3 iterations shared among 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)

time="2026-07-13T17:23:30Z" level=warning msg="Request Failed" error="Get \"https://127.0.0.1:18443/delay\": request timeout"
time="2026-07-13T17:23:30Z" level=info msg="{\"iter\":0,\"proto\":\"\",\"status\":0,\"error\":\"request timeout\",\"error_code\":1050,\"blocked\":2.490609,\"connecting\":0.143952,\"tls_handshaking\":2.238135,\"sending\":0.067032,\"waiting\":248.316183,\"receiving\":0,\"duration\":248.383215}" source=console
time="2026-07-13T17:23:30Z" level=warning msg="Request Failed" error="Get \"https://127.0.0.1:18443/delay\": request timeout"
time="2026-07-13T17:23:30Z" level=info msg="{\"iter\":1,\"proto\":\"\",\"status\":0,\"error\":\"request timeout\",\"error_code\":1050,\"blocked\":2.437252,\"connecting\":0.1478,\"tls_handshaking\":2.22862,\"sending\":0.046091,\"waiting\":248.689921,\"receiving\":0,\"duration\":248.736012}" source=console
time="2026-07-13T17:23:31Z" level=warning msg="Request Failed" error="Get \"https://127.0.0.1:18443/delay\": request timeout"
time="2026-07-13T17:23:31Z" level=info msg="{\"iter\":2,\"proto\":\"\",\"status\":0,\"error\":\"request timeout\",\"error_code\":1050,\"blocked\":3.851619,\"connecting\":0.191863,\"tls_handshaking\":2.86318,\"sending\":0.092055,\"waiting\":246.435945,\"receiving\":0,\"duration\":246.528}" source=console

     data_received..............: 4.2 kB  5.6 kB/s
     data_sent..................: 1.4 kB  1.8 kB/s
     http_req_blocked...........: avg=2.92ms   min=2.43ms   med=2.49ms   max=3.85ms   p(90)=3.57ms   p(95)=3.71ms  
     http_req_connecting........: avg=161.2µs  min=143.95µs med=147.79µs max=191.86µs p(90)=183.05µs p(95)=187.45µs
     http_req_duration..........: avg=247.88ms min=246.52ms med=248.38ms max=248.73ms p(90)=248.66ms p(95)=248.7ms 
     http_req_failed............: 100.00% 3 out of 3
     http_req_receiving.........: avg=0s       min=0s       med=0s       max=0s       p(90)=0s       p(95)=0s      
     http_req_sending...........: avg=68.39µs  min=46.09µs  med=67.03µs  max=92.05µs  p(90)=87.05µs  p(95)=89.55µs 
     http_req_tls_handshaking...: avg=2.44ms   min=2.22ms   med=2.23ms   max=2.86ms   p(90)=2.73ms   p(95)=2.8ms   
     http_req_waiting...........: avg=247.81ms min=246.43ms med=248.31ms max=248.68ms p(90)=248.61ms p(95)=248.65ms
     http_reqs..................: 3       3.979916/s
     iteration_duration.........: avg=251.19ms min=250.65ms med=251.46ms max=251.46ms p(90)=251.46ms p(95)=251.46ms
     iterations.................: 3       3.979916/s


running (00m00.8s), 0/1 VUs, 3 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.8s/10m0s  3/3 shared iters
```

This directly **refutes any "all errors ⇒ all-zero timings" over-generalisation**: an errored request that got far enough preserves its partial timings. (It also demonstrates the `Waiting` "no-first-byte" edge from §4: `waiting ≈ 248 ms` here is `done − wroteRequest`, and `receiving = 0`.)

*(c.4) All-zero incidence on normal input.* To quantify how often the all-zero pathology occurs on *healthy* input (which is what the Windows report was about — normal requests, not closed ports), the healthy keep-alive server was hit with `vus:4, iterations:200` and the script logged **only** all-seven-zero rows (`/tmp/obs/script_scale.js`, §7). Zero such rows were logged, and no request failed:

```text
$ K6_NO_USAGE_REPORT=true ./k6 run /tmp/obs/script_scale.js

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/obs/script_scale.js
        output: -

     scenarios: (100.00%) 1 scenario, 4 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 200 iterations shared among 4 VUs (maxDuration: 10m0s, gracefulStop: 30s)


     data_received..................: 39 kB 3.5 MB/s
     data_sent......................: 22 kB 1.9 MB/s
     http_req_blocked...............: avg=58.29µs  min=985ns   med=1.44µs  max=3.51ms   p(90)=3.03µs   p(95)=4.57µs  
     http_req_connecting............: avg=2.67µs   min=0s      med=0s      max=174.57µs p(90)=0s       p(95)=0s      
     http_req_duration..............: avg=126µs    min=51.59µs med=76.93µs max=1.32ms   p(90)=185.58µs p(95)=522.23µs
       { expected_response:true }...: avg=126µs    min=51.59µs med=76.93µs max=1.32ms   p(90)=185.58µs p(95)=522.23µs
     http_req_failed................: 0.00% 0 out of 200
     http_req_receiving.............: avg=21.1µs   min=8.42µs  med=12.2µs  max=670.27µs p(90)=25.91µs  p(95)=36.06µs 
     http_req_sending...............: avg=6.37µs   min=2.87µs  med=3.95µs  max=103.19µs p(90)=8.42µs   p(95)=11.76µs 
     http_req_tls_handshaking.......: avg=52.61µs  min=0s      med=0s      max=3.31ms   p(90)=0s       p(95)=0s      
     http_req_waiting...............: avg=98.53µs  min=35.37µs med=59.46µs max=951.19µs p(90)=144.79µs p(95)=209.47µs
     http_reqs......................: 200   17658.472429/s
     iteration_duration.............: avg=217.26µs min=73.74µs med=106.9µs max=3.97ms   p(90)=233.8µs  p(95)=572.99µs
     iterations.....................: 200   17658.472429/s


running (00m00.0s), 0/4 VUs, 200 complete and 0 interrupted iterations
default ✓ [ 100% ] 4 VUs  00m00.0s/10m0s  200/200 shared iters
```

(The console produced **no** `ALLZERO` line — the script only logs those — so the normal-input all-zero incidence is **0/200** on this Linux host; `http_req_failed 0.00% 0 out of 200`.) **Honest limitation:** the "random all-zero on normal requests" you saw is **specific to Windows timer resolution** and was **not reproduced on this Linux/amd64 host** despite running 200 healthy requests. Attempt matrix: healthy keep-alive server (0/200 all-zero); closed port (all-zero, but that is the *failure* path, not "normal"); mid-flight timeout (partial, non-zero). The Windows-only cause is therefore given as **(inferred)** from the maintainers' documented timer hack (`tracer_test.go:L33-L40`, `golang/go#8687`), not as a Linux observation. To observe it directly one would need to run on a Windows host.

---

### 3.4 Anomaly 4 — a connection flagged "not reused" yet the connect timestamps equal the got-connection timestamp

> **Your words:** *"sometimes a connection is flagged as \"not reused\" but the connect timestamps show the same value as the got connection timestamp, which should be impossible if a real TCP handshake occurred."*

**(a) Verdict: EXPECTED-BY-DESIGN — a guarded handling of a Go standard-library HTTP/2 quirk, not an impossible state.** When the Go HTTP/2 round-tripper reports a `GotConn` with `Reused=false` for a connection on which no `ConnectStart`/`ConnectDone` fired (because it was actually served from an already-established connection), the connect/TLS timestamps are still `0`. k6 detects this and back-fills them to the got-connection instant, leaving `connectStart == connectDone == gotConn` while `connReused` stays `false`. There is no handshake duration to record because no new handshake happened.

**(b) Mechanism.** The `else` (not-reused) branch of `GotConn` guards exactly this case. Because the connect/TLS hooks never fired (timestamps still `0`), it `CompareAndSwap`s them from `0` to `now`:

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
(`lib/netext/httpext/tracer.go:L278-L293`; the two back-fills are `L287-L288`.) `connReused` was set from `info.Reused` at `lib/netext/httpext/tracer.go:L262`, so it stays `false` here. The result — equal connect and got-connection timestamps with `reused=false` — is therefore the **intended** output of this guard: the guard prevents a bogus non-zero `connecting` when the stdlib mislabels a de-facto-reused connection.

**(c) Grounding and honest reproduction status.**

- **Code (observed at HEAD/baseline):** the guard branch above at `lib/netext/httpext/tracer.go:L278-L293`.
- **External authority — represented accurately.** The k6 code comment attributes this to a Go stdlib HTTP/2 behaviour. The closest public Go tracker item, `golang/go#27753` ("net/http: `Transport.MaxConnsPerHost` doesn't work well with HTTP/2"), shows that under HTTP/2 concurrency the stdlib performs **excess parallel dials**: in its reproduction, 10 concurrent requests print `TLSHandshakeStart` ~10 times while `GotConn` reports `Reused: true` for most of them (one genuinely new connection). **Note the precise shape:** #27753 documents *too many real dials with mostly `Reused: true`* — it is **not** itself a report of a *falsely-`false`* `Reused` flag. **(inferred — external, `golang/go#27753`.)** The authoritative statement that the `httptrace` reuse/handshake signals are imperfect under HTTP/2 is the k6 maintainers' own (see §5: k6 issue #866, PR #862). k6's `else` branch is the defensive response to this class of stdlib imperfection.
- **This is HTTP/2-specific and its exact internal state is not exposed to scripts.** The local HTTP/2 burst (§3.5(c)) reproduces the *upstream trigger* — excess parallel dials under a cold concurrent burst — through the canonical entry point. However, the two internal facts your report names — the `reused` boolean and the raw `connectStart`/`connectDone`/`gotConn` timestamps — are **not surfaced in `res.timings`**, so the exact "`reused=false` with equal timestamps" state **cannot be observed directly** via the canonical JS entry point. That specific internal state is therefore reported as **contract/source-derived (inferred)**, grounded in the guarded branch above and the maintainers' known-issue, not as a direct runtime observation.
- **Observed (indirect):** the `-race` suite (§3.3) passes with zero data races, so even when this branch runs concurrently it is memory-safe.

There is nothing **new** to report upstream to k6: the k6 side already handles the stdlib quirk, and the maintainers already track the residual HTTP/2 corner cases (§5).

---

### 3.5 Anomaly 5 — `ConnectStart`/`ConnectDone` sometimes called multiple times; is it "a double counting bug"?

> **Your words:** *"ConnectStart and ConnectDone sometimes get called multiple times for a single request, which looks like a double counting bug."*

**(a) Verdict: the "double counting" hypothesis is REFUTED. EXPECTED-BY-DESIGN.** Multiple invocations are part of the Go `httptrace` **contract** under dual-stack ("Happy Eyeballs") dialing, and the tracer **de-duplicates** them: only the *first* timestamp is kept, so subsequent invocations are no-ops. Nothing is counted twice.

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
(`lib/netext/httpext/tracer.go:L191-L202`; the de-dup is at `L201`.) `ConnectDone` applies the same discipline (and only on success), de-dup at `lib/netext/httpext/tracer.go:L218`. Because `Connecting = connectDone - connectStart` (§3.1) uses only these first-write values, a second or third invocation changes nothing — there is no accumulation and therefore no double counting.

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
(`lib/netext/dialer.go:L140-L149`, inside `findRemote`, `lib/netext/dialer.go:L116`; entered from `DialContext`, `lib/netext/dialer.go:L59`.) A single resolved host means one dial target, so `ConnectStart` normally fires once. The `CompareAndSwap` guard is therefore a defensive measure for the dual-stack edge case rather than something that trips on every request.

**(c) Grounding and honest reproduction status.**

- **Code (observed at HEAD/baseline):** the de-dup at `lib/netext/httpext/tracer.go:L201` and `L218`; the contract comment at `L191-L196`; the single-IP dialer at `lib/netext/dialer.go:L140-L149`.
- **(inferred — external):** the Go `net/http/httptrace` `ClientTrace` documentation states verbatim that with dual-stack enabled `ConnectStart`/`ConnectDone` "may be called multiple times" — confirming multiple calls are the contract, not a k6 defect.
- **Reproduction attempt (canonical entry point) and its honest limit.** To attempt to trigger the multi-dial regime, a local **HTTP/2** server (`/tmp/obs/h2server.go`, `httptest` with `EnableHTTP2 = true` + `(*httptest.Server).StartTLS()`; §7) was hit with a **50-VU cold concurrent burst** (`/tmp/obs/script_h2.js`, §7). All 50 requests negotiated `proto:"HTTP/2.0"` and each recorded a distinct `connecting`/`tls_handshaking` (i.e. the stdlib opened many parallel connections — the #27753 "excess parallel dials" regime), stable across two runs:

```text
$ K6_NO_USAGE_REPORT=true ./k6 run /tmp/obs/script_h2.js

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/obs/script_h2.js
        output: -

     scenarios: (100.00%) 1 scenario, 50 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 50 iterations shared among 50 VUs (maxDuration: 10m0s, gracefulStop: 30s)

time="2026-07-13T17:27:30Z" level=info msg="{\"proto\":\"HTTP/2.0\",\"status\":200,\"connecting\":0.23131,\"tls_handshaking\":6.04388,\"blocked\":8.095646,\"waiting\":7.129171}" source=console
time="2026-07-13T17:27:30Z" level=info msg="{\"proto\":\"HTTP/2.0\",\"status\":200,\"connecting\":0.163585,\"tls_handshaking\":6.307273,\"blocked\":6.614521,\"waiting\":8.60138}" source=console
time="2026-07-13T17:27:30Z" level=info msg="{\"proto\":\"HTTP/2.0\",\"status\":200,\"connecting\":2.035289,\"tls_handshaking\":6.016815,\"blocked\":8.1293,\"waiting\":8.147222}" source=console
time="2026-07-13T17:27:30Z" level=info msg="{\"proto\":\"HTTP/2.0\",\"status\":200,\"connecting\":1.509165,\"tls_handshaking\":5.837088,\"blocked\":7.457123,\"waiting\":6.965674}" source=console
time="2026-07-13T17:27:30Z" level=info msg="{\"proto\":\"HTTP/2.0\",\"status\":200,\"connecting\":3.651048,\"tls_handshaking\":6.84869,\"blocked\":10.614353,\"waiting\":6.051396}" source=console
... (45 further HTTP/2.0 console rows omitted here; all 50 are proto:"HTTP/2.0" / status:200, each with a distinct non-zero connecting and tls_handshaking — full 50-row capture: /tmp/obs/h2.log) ...

     data_received..................: 81 kB 1.2 MB/s
     data_sent......................: 26 kB 386 kB/s
     http_req_blocked...............: avg=13.86ms  min=6.36ms  med=12.16ms max=57.02ms  p(90)=17.89ms  p(95)=18.41ms 
     http_req_connecting............: avg=1.28ms   min=64.75µs med=1.41ms  max=3.96ms   p(90)=2ms      p(95)=2.97ms  
     http_req_duration..............: avg=13.03ms  min=5.68ms  med=7.57ms  max=46.06ms  p(90)=40.08ms  p(95)=40.62ms 
       { expected_response:true }...: avg=13.03ms  min=5.68ms  med=7.57ms  max=46.06ms  p(90)=40.08ms  p(95)=40.62ms 
     http_req_failed................: 0.00% 0 out of 50
     http_req_receiving.............: avg=57.91µs  min=16.04µs med=31.23µs max=863.69µs p(90)=58.16µs  p(95)=124.8µs 
     http_req_sending...............: avg=863.93µs min=44.61µs med=72.94µs max=39.2ms   p(90)=111.34µs p(95)=148.69µs
     http_req_tls_handshaking.......: avg=12.11ms  min=4.62ms  med=10.26ms max=56.82ms  p(90)=16.58ms  p(95)=17.08ms 
     http_req_waiting...............: avg=12.11ms  min=5.6ms   med=7.24ms  max=41.17ms  p(90)=39.8ms   p(95)=40.23ms 
     http_reqs......................: 50    745.889945/s
     iteration_duration.............: avg=28.19ms  min=14.73ms med=20.64ms max=66.53ms  p(90)=58.9ms   p(95)=60.3ms  
     iterations.....................: 50    745.889945/s


running (00m00.1s), 00/50 VUs, 50 complete and 0 interrupted iterations
default ✓ [ 100% ] 50 VUs  00m00.1s/10m0s  50/50 shared iters
```

  Across both burst runs: 50/50 requests measured a distinct dial and a distinct TLS handshake, `0/50` reused, all `HTTP/2.0`. **Honest limit for this anomaly:** the per-request **count** of `ConnectStart`/`ConnectDone` invocations is an internal `httptrace` fact that k6 does **not** expose to scripts (`res.timings` has no such field), so "multiple `ConnectStart`/`ConnectDone` for a single request" could **not be observed directly** through the canonical entry point. The conclusion (multiple calls are contractual and are de-duplicated to the first timestamp) is therefore retained as **contract/source-derived (inferred)** — grounded in the official `httptrace` contract, the de-dup code at `L201`/`L218`, the single-IP dialer, and the maintainers' tests — rather than claimed as a direct observation.
- **Observed (indirect):** with k6's single-IP dialer, the §3.1 run produced exactly one connect phase on the cold iteration and zero on reused ones, consistent with a single `ConnectStart`; and the `-race` suite (§3.3) shows the atomic de-dup is memory-safe under heavy concurrency.

Nothing to report upstream: multiple invocations are contractual and are correctly de-duplicated.

For completeness, the single-VU HTTP/2 reuse run confirms the Anomaly-1 reuse-zero pattern also holds over HTTP/2 (iter 0 cold; iters 1–3 `connecting=0`, `tls_handshaking=0`, `waiting` non-zero):

```text
$ K6_NO_USAGE_REPORT=true ./k6 run /tmp/obs/script_h2_reuse.js

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/obs/script_h2_reuse.js
        output: -

     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 4 iterations shared among 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)

time="2026-07-13T17:28:06Z" level=info msg="{\"proto\":\"HTTP/2.0\",\"status\":200,\"connecting\":0.137735,\"tls_handshaking\":3.241952,\"waiting\":5.527488}" source=console
time="2026-07-13T17:28:06Z" level=info msg="{\"proto\":\"HTTP/2.0\",\"status\":200,\"connecting\":0,\"tls_handshaking\":0,\"waiting\":5.371968}" source=console
time="2026-07-13T17:28:06Z" level=info msg="{\"proto\":\"HTTP/2.0\",\"status\":200,\"connecting\":0,\"tls_handshaking\":0,\"waiting\":5.327657}" source=console
time="2026-07-13T17:28:06Z" level=info msg="{\"proto\":\"HTTP/2.0\",\"status\":200,\"connecting\":0,\"tls_handshaking\":0,\"waiting\":5.331373}" source=console

     data_received..................: 1.8 kB 68 kB/s
     data_sent......................: 628 B  24 kB/s
     http_req_blocked...............: avg=884.97µs min=276ns   med=352ns   max=3.53ms   p(90)=2.47ms   p(95)=3ms     
     http_req_connecting............: avg=34.43µs  min=0s      med=0s      max=137.73µs p(90)=96.41µs  p(95)=117.07µs
     http_req_duration..............: avg=5.53ms   min=5.4ms   med=5.45ms  max=5.81ms   p(90)=5.72ms   p(95)=5.76ms  
       { expected_response:true }...: avg=5.53ms   min=5.4ms   med=5.45ms  max=5.81ms   p(90)=5.72ms   p(95)=5.76ms  
     http_req_failed................: 0.00%  0 out of 4
     http_req_receiving.............: avg=56.77µs  min=29.87µs med=51.08µs max=95.03µs  p(90)=88.14µs  p(95)=91.58µs 
     http_req_sending...............: avg=89.19µs  min=46.96µs med=58.83µs max=192.16µs p(90)=153.48µs p(95)=172.82µs
     http_req_tls_handshaking.......: avg=810.48µs min=0s      med=0s      max=3.24ms   p(90)=2.26ms   p(95)=2.75ms  
     http_req_waiting...............: avg=5.38ms   min=5.32ms  med=5.35ms  max=5.52ms   p(90)=5.48ms   p(95)=5.5ms   
     http_reqs......................: 4      150.293109/s
     iteration_duration.............: avg=6.61ms   min=5.5ms   med=5.6ms   max=9.73ms   p(90)=8.51ms   p(95)=9.12ms  
     iterations.....................: 4      150.293109/s


running (00m00.0s), 0/1 VUs, 4 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.0s/10m0s  4/4 shared iters
```

---

### 3.6 The `sending` three-way switch (for completeness)

`Tracer.Done` selects the `sending` baseline with a three-way `switch`, whose `default` arm is the same HTTP/2 `GotConn`-first / `Reused=false` case from Anomaly 4:

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
(`lib/netext/httpext/tracer.go:L346-L359`.) The totals are then `ConnDuration = Connecting + TLSHandshaking` (`lib/netext/httpext/tracer.go:L380`) and `Duration = Sending + Waiting + Receiving` (`lib/netext/httpext/tracer.go:L381`). In §3.1 the TLS arm was taken (`sending` non-zero on every iteration, e.g. `0.043956` cold and `0.005778–0.009444` warm), consistent with the HTTP/1.1-over-TLS local server.

---

## 4. Per-metric trust verdict

All seven **populated** timings are computed in `Tracer.Done` (`lib/netext/httpext/tracer.go:L315-L384`) and copied verbatim into `res.timings` via `metrics.D()` (`lib/netext/httpext/request.go:L98-L106`). Every one is **trustworthy on the paths demonstrated here**, with the specific edge behaviours noted below (so a `0` or an unusual value is interpreted correctly rather than as a defect).

| `res.timings` field | Computed at (`file:line`) | What it measures | Trustworthy? | Edge behaviour & notes (observed unless labelled) |
|---------------------|---------------------------|------------------|:------------:|---------------------------------------------------|
| `blocked` | `tracer.go:L323-L325` | `gotConn − getConn`: connection-acquisition time (DNS + TCP + TLS + pool wait) | **Yes** | **Bimodal**: cold ≈ 2.22–2.85 ms, warm ≈ 0.002–0.012 ms (§3.2). Near-zero on idle-pool checkout is correct. `0` if the guard `gotConn>getConn` is not met (e.g. dial-refused, §3.3). |
| `connecting` | `tracer.go:L340-L342` | `connectDone − connectStart`: TCP connect time | **Yes** | **Exactly `0` on reuse** (§3.1) — no new dial; 24/24 reused samples = 0. Back-filled to `0` in the HTTP/2 false-`Reused` guard (§3.4). |
| `tls_handshaking` | `tracer.go:L343-L345` | `tlsHandshakeDone − tlsHandshakeStart`: TLS handshake time | **Yes** | **Exactly `0` on reuse** (§3.1); 24/24 reused = 0. **(inferred)** can be `0` on Windows if both TLS hooks read the same low-resolution tick (§3.3 cause ii). |
| `sending` | `tracer.go:L346-L359` | request-write time; baseline chosen by the three-way switch | **Yes** | Baseline is TLS-done, else connect-done, else `gotConn` (the HTTP/2 `Reused=false` `default` arm, §3.6). Non-zero every successful iteration (e.g. `0.043956` cold, `0.005778` warm). Requires `wroteRequest != 0`. |
| `waiting` | `tracer.go:L361-L372` | time-to-first-response-byte after the request was written (TTFB) | **Yes** | If the response's first byte arrives **before** the request write completes (`gotFirstResponseByte ≤ wroteRequest`, seen in some HTTP/2 cases), `waiting` is left `0` by the guard. If **no** first byte ever arrives, `waiting = done − wroteRequest` — observed as `≈248 ms` in the mid-flight timeout (§3.3 c.3). |
| `receiving` | `tracer.go:L374-L376` | time to read the response body after first byte | **Yes** | `done − gotFirstResponseByte`; `0` when no first byte arrived (mid-flight timeout, §3.3 c.3). Non-zero on reuse (`0.03–0.06` ms, §3.1). |
| `duration` | `tracer.go:L381` | `sending + waiting + receiving`: server round-trip excluding connect | **Yes** | Matches `http_req_duration` in the summary (§3.1). On a no-first-byte timeout it is dominated by `waiting` (§3.3 c.3). |

**Two write disciplines to keep in mind (they explain the "latest vs first" edges):**

- `WroteRequest` uses `atomic.StoreInt64(&t.wroteRequest, now())` (`tracer.go:L301`) and its doc notes it **"may be called multiple times in the case of retried requests"** (`tracer.go:L297-L298`) — so `wroteRequest` holds the **latest** write-completion time. **(inferred — external:** the Go `httptrace` `WroteRequest` doc states the same.) Under stdlib HTTP/2 retries this is one of the corner cases the maintainers flag (§5).
- `GotFirstResponseByte` uses `atomic.CompareAndSwapInt64(&t.gotFirstResponseByte, 0, now())` (`tracer.go:L311`) — so it keeps the **first** byte's time and ignores later calls.

**Not populated — always `0`:** `looking_up` is declared in the struct (`response.go:L38`) but **never written** by the population site (`request.go:L98-L106`), so `res.timings.looking_up` is a **structural constant `0`**, confirmed in every observed iteration. It is not a measurement and should not be read as "DNS took 0 ms".

**Sink agreement.** Because `res.timings` (Sink B) and the `http_req_*` metrics (Sink A) are copied from the *same* `Trail`, they always agree. Observed in §3.1: `http_req_connecting` med `0s`/max `188.52µs` and `http_req_tls_handshaking` med `0s`/max `2.05ms` line up exactly with the per-iteration `res.timings`.

**Overall:** trust the values on the demonstrated paths. Interpret `0` on `connecting`/`tls_handshaking` as "connection reused" (not "instantaneous handshake"), a small `blocked` as "idle connection reused", and an all-zero row as an **errored** request (check `res.error`/`res.error_code`/`http_req_failed`) — or, on Windows only, a possible clock-granularity zero. For **HTTP/2 with stdlib retries**, treat the affected request's sub-phase split with the caveat in §5.

---

## 5. Upstream recommendation

**Do not file a *new* upstream bug report for any of the five anomalies** — but this is because the only real corner case is **already known and already tracked**, not because "no issue exists".

- Anomalies 1 & 2 are direct consequences of HTTP keep-alive connection reuse and the `httptrace` "reused connections don't fire dial/TLS hooks" contract — no defect.
- Anomaly 3's "race" was **not observed** (clean `-race` run, `DATA_RACE_WARNINGS=0`); the all-zero rows are the failure path (status 0) or Windows timer granularity — no defect.
- Anomaly 5's "double counting" is refuted by the `CompareAndSwapInt64` de-dup (`tracer.go:L201,L218`); multiple calls are the documented dual-stack contract — no defect.
- Anomaly 4 is k6's **existing in-tree workaround** (`tracer.go:L287-L288`) for a Go stdlib HTTP/2 quirk.

**The one honest caveat (disclosed rather than hidden):** the k6 maintainers themselves acknowledge that the current nanosecond-timestamps-with-atomics `Tracer` does not perfectly handle a few HTTP/2 corner cases, chiefly stdlib-driven **request retries**. This is tracked in **k6 issue #866 ("Improve HTTP request tracing")**, which arose from the fix **PR #862 ("Fix incorrect tracing of corner case HTTP requests")** whose changes are the very `GotConn`/`Done` guards cited throughout this document. **(inferred — external, k6 #866 / #862.)** In the maintainers' framing, some corner cases "aren't especially well handled by the current … atomics `Tracer` implementation, even after the fixes", and retried HTTP/2 requests may even warrant emitting more than one set of `http_req_*` samples. Consequently:

- For the **common paths** (HTTP/1.1 and non-retried HTTP/2, cold and warm) that a typical load test exercises, the timings are trustworthy — as demonstrated in §3.
- For **HTTP/2 requests that the Go stdlib silently retries**, a small sub-phase mis-attribution is a **known, tracked** limitation. Because it is already captured by k6 #866, **there is nothing new to report** — but you should be aware of it rather than assume perfection.

The single underlying stdlib matter (the HTTP/2 false-`Reused` flag) is a **Go** issue that k6 already defends against; it is not a k6 defect to file.

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
| `GotConn` | `GotConn` (`tracer.go:L256-L294`) | reused → `SwapInt64` (`L271-L277`); not-reused → `CompareAndSwap …,0,now` (`L287-L288`) | §3.1, §3.4 |
| `WroteRequest` | `WroteRequest` (`tracer.go:L299-L304`) | `StoreInt64` if `err==nil` (`L301`, keeps **latest**) | §3.6, §4 (`sending`) |
| `GotFirstResponseByte` | `GotFirstResponseByte` (`tracer.go:L310-L312`) | `CompareAndSwap …,0,now` (`L311`, keeps **first**) | §4 (`waiting`/`receiving`) |

### 6.2 Structures, functions, and disciplines

| Item | `file:line` | Addressed in |
|------|-------------|--------------|
| `Tracer` struct (8 `int64` fields + `connReused`) | `tracer.go:L147-L159` | §2 |
| Fresh `Tracer` per request | `transport.go:L205` | §2 |
| "NOT safe to reuse Tracers" | `tracer.go:L145` | §2 |
| `Tracer.Done()` builds `Trail` | `tracer.go:L315-L384` | §2, §3, §4 |
| `Trail` struct (seven timing fields + conn info) | `tracer.go:L15-L40` | §2, §4 |
| `Trail.SaveSamples` emits all seven timings + `http_reqs` | `tracer.go:L43-L122` | §2, §4 (sink agreement) |
| `Trail` → `http_req_*` (`SaveSamples` call) | `transport.go:L146` | §2, §4 |
| `Trail` → `res.timings` (`metrics.D()` ns→ms) | `request.go:L98-L106`, `units.go:L11-L13` | §2, §4 |
| `ResponseTimings` struct | `response.go:L34-L44`; `Timings` field `L65` | §2, §4 |
| Metric names + `Trend`/`Time` registration (all seven) | `builtin.go:L16-L23`, `builtin.go:L91-L97` | §2 |
| `SwapInt64` (overwrite on reuse) vs `CompareAndSwapInt64` (keep first) | `tracer.go:L271-L277` vs `L201,L218,L287-L288` | §3.1, §3.4, §3.5 |
| Zero-guards (`gotConn>getConn`; both endpoints non-zero) | `tracer.go:L323,L340,L343` | §3.1, §3.2, §4 |
| Single-IP dialer (`findRemote`/`LookupIP`/`NewHost`) | `dialer.go:L59,L116,L140-L149` | §3.5 |
| TCP keep-alive (`net.Dialer.KeepAlive`) vs HTTP reuse (`http.Transport`) | `runner.go:L90-L93` vs `runner.go:L193-L201` | §2.1 |
| Body read-to-EOF+close enabling reuse | `compression.go:L118,L129-L130,L134-L139` | §2.1 |
| Late-callback atomics comment + `atomic.LoadInt64` reads | `tracer.go:L327-L331,L332-L338` | §3.3 |
| `-race` result (exit 0, 0 data races, 200 parallel cancelled reqs) | `tracer_test.go:L257-L292` | §3.3 |
| `TestCancelledRequest` asserts only `!(resp==nil && err==nil)` | `tracer_test.go:L275-L279` | §3.3 |
| Windows reset path (`WSAECONNRESET`→`tcpResetByPeerErrorCode` 1220), maps code only | `error_codes_syscall_windows.go:L10-L16`, `error_codes.go:L44` | §3.3 |
| Dial-refused (1212), request-timeout (1050) codes | `error_codes.go:L42,L30` | §3.3 |
| Windows timer-resolution hack (golang/go#8687, #41087) | `tracer_test.go:L33-L40` | §3.3 |
| Reuse zero-asserts driven by `iterations []bool{false,true,true}` | `tracer_test.go:L115,L161-L165` | §3.1 |
| `sending` three-way switch (incl. HTTP/2 default arm) | `tracer.go:L346-L359` | §3.6 |
| `looking_up` declared but never populated (always 0) | `response.go:L38` vs `request.go:L98-L106` | §2, §4 |
| Version banner (`Version`, `FullVersion`) | `consts.go:L12,L52` | §1.1 |

### 6.3 The five preserved user examples

| # | User example (verbatim, abbreviated) | Verdict | Section |
|---|--------------------------------------|---------|---------|
| 1 | "subsequent requests show exactly 0 for … connecting … and TLS handshaking … even though I can see network activity" | Expected-by-design (keep-alive reuse) | §3.1 |
| 2 | "\"blocked\" timing shows massive values like 500ms … other times … near zero for identical requests" | Expected-by-design (`gotConn − getConn`; bimodal) | §3.2 |
| 3 | "occasionally ALL the timing metrics return 0 … definitely seems like a race" | No data race observed; failure-path / Windows timer | §3.3 |
| 4 | "connection is flagged as \"not reused\" but the connect timestamps show the same value as the got connection timestamp" | Expected-by-design (guarded HTTP/2 stdlib quirk; internal state inferred) | §3.4 |
| 5 | "ConnectStart and ConnectDone sometimes get called multiple times … looks like a double counting bug" | Double-counting refuted (dedup); multi-call count is inferred | §3.5 |

### 6.4 Methodology confirmations

- **Run-code-first:** canonical `go build -o k6 .` then `./k6 run` against local keep-alive HTTP/1.1 and HTTP/2 HTTPS servers; version banner captured from that build (§1.1).
- **Scale & stability:** every magnitude states its scale (`vus:1, iterations:4`, repeated 8×; `vus:4, iterations:200`; `vus:50, iterations:50`); the reuse-zero and bimodal-`blocked` patterns were stable across all 8 runs, the error-path all-zero across 2 runs, and the HTTP/2 burst pattern across 2 runs.
- **Distribution reporting:** Anomaly 2 (`blocked`) was reproduced by repeating the **same unchanged input** and reporting the observed 32-sample distribution (§3.2); all-zero incidence was measured on unchanged normal input as **0/200** (§3.3 c.4).
- **Canonical entry point only:** every value is an observed `res.timings`/`http_req_*` from a real `k6 run`, or a real `go test` result. No debug hook or synthetic bypass was used.
- **Inference labelled:** the 500 ms real-network scaling of `blocked`, the Windows timer-resolution zeroing (not reproduced on Linux), the HTTP/2 false-`Reused`/dual-stack external behaviours, the per-request `ConnectStart`/`ConnectDone` **count** (not JS-visible), and the exact `reused=false`+equal-timestamp internal state (not JS-visible) are marked **(inferred)**; everything else is observed.

---

## 7. Appendix — reproduction harness and cleanup

The harness lives **outside** the repository (`/tmp/obs`), so the source tree is never modified. Every file is listed below **verbatim** (byte-for-byte as it exists on disk), and the SHA-256 above is computed over that exact file — each file ends with a single trailing newline — so saving a listing back under its filename reproduces the corresponding hash, making the runs above exactly reproducible.

### 7.1 Harness sources and SHA-256

```text
$ cd /tmp/obs && sha256sum go.mod server.go h2server.go \
    script.js script_err.js script_timeout.js \
    script_scale.js script_h2.js script_h2_reuse.js \
    run_blocked.sh run_tests.sh
e781bd5a8c709997e4be4feab1da25f11a9a192382d3b9bcb8d6cc49939bd1a9  go.mod
d3e45d7d6a89aff06645e4cd9f2307f6b3e2b53ac84d99360435275a7dec2041  server.go
6f2f3a1720fcdfed7b3d0dd04842c9055b269cce579ae647b76f0588455015ae  h2server.go
10503c1ffc5ae33eb5a234144360f3e2fb2fc0ce4ec969ee6095abe093e08f50  script.js
6daadd963fbb835b90fd6adb4a82f77b27637f1d8277955e38d6a7e21ac49743  script_err.js
ed7741067ad636900ae1dcddefc3370f9f315f9d8614a8a5e025fc1006a16aaa  script_timeout.js
864e733f17b4810421385687abba388289dee2e2f0a73aea33841e93e87fa19c  script_scale.js
2bfe9ec9efa7b35199f5b52ac666c3dcf9cfc9ca695c864909f29da7a5ee0f38  script_h2.js
f84f944c53d8ca6fbba242aea67fdfedc34dee006481d9a7b7448fcdb91ce125  script_h2_reuse.js
a93d8a787db7fb6f0befefd2fcf2aa275e2284f0c60a91558635617577ced5fb  run_blocked.sh
cc31a79f94e24261346cc5ba2364eba9b2b489cf757167869b9ace2a1ea35846  run_tests.sh
```

**`go.mod`**

```go
module obs

go 1.23
```

**`server.go`** — HTTP/1.1 keep-alive HTTPS server via `(*httptest.Server).StartTLS()` (self-signed cert, keep-alive on by default). `/` responds immediately (reuse + blocked runs); `/delay` sleeps 2 s before the first byte (mid-flight timeout run).

```go
// Local keep-alive HTTPS observation server for the k6 tracer investigation.
// Uses (*httptest.Server).StartTLS() which auto-generates a self-signed cert
// and enables HTTP keep-alive (persistent connections) by default.
package main

import (
	"fmt"
	"net"
	"net/http"
	"net/http/httptest"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	mux := http.NewServeMux()
	// "/" responds immediately -> used for reuse + blocked-distribution runs.
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "hello from keep-alive server")
	})
	// "/delay" sleeps BEFORE writing any byte -> the client times out while
	// waiting for the first response byte (used for the mid-flight timeout run).
	mux.HandleFunc("/delay", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(2 * time.Second)
		fmt.Fprintln(w, "delayed hello")
	})

	ts := httptest.NewUnstartedServer(mux)
	l, err := net.Listen("tcp", "127.0.0.1:18443")
	if err != nil {
		panic(err)
	}
	ts.Listener.Close()
	ts.Listener = l
	ts.StartTLS() // (*httptest.Server).StartTLS(): self-signed cert, keep-alive on
	fmt.Printf("READY url=%s pid=%d\n", ts.URL, os.Getpid())

	c := make(chan os.Signal, 1)
	signal.Notify(c, os.Interrupt, syscall.SIGTERM)
	<-c
	ts.Close()
}
```

**`h2server.go`** — HTTP/2-over-TLS server via `httptest` with `EnableHTTP2 = true` + `(*httptest.Server).StartTLS()`.

```go
package main

import (
	"fmt"
	"net"
	"net/http"
	"net/http/httptest"
	"os"
	"os/signal"
	"syscall"
	"time"
)

// Minimal HTTP/2-over-TLS server for observation. Uses httptest.Server with
// EnableHTTP2 = true so ALPN negotiates "h2". Explicit listener pins the port.
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		// Tiny artificial delay so concurrent requests overlap in flight,
		// maximising the chance of parallel connection establishment.
		time.Sleep(5 * time.Millisecond)
		fmt.Fprintf(w, "hello h2 proto=%s", r.Proto)
	})

	ts := httptest.NewUnstartedServer(mux)
	ln, err := net.Listen("tcp", "127.0.0.1:18444")
	if err != nil {
		fmt.Fprintf(os.Stderr, "listen error: %v\n", err)
		os.Exit(1)
	}
	ts.Listener.Close()
	ts.Listener = ln
	ts.EnableHTTP2 = true
	ts.StartTLS()
	defer ts.Close()

	fmt.Printf("READY url=%s pid=%d http2=true\n", ts.URL, os.Getpid())

	sigc := make(chan os.Signal, 1)
	signal.Notify(sigc, syscall.SIGINT, syscall.SIGTERM)
	<-sigc
	fmt.Println("shutting down")
}
```

**`script.js`** — reuse + blocked-distribution driver (`vus:1, iterations:4`); logs the full `res.timings` plus `proto` and `looking_up`:

```javascript
import http from 'k6/http';

export const options = {
  vus: 1,
  iterations: 4,
  insecureSkipTLSVerify: true, // local self-signed cert
};

export default function () {
  const res = http.get('https://127.0.0.1:18443/');
  console.log(JSON.stringify({
    iter: __ITER,
    proto: res.proto,
    status: res.status,
    blocked: res.timings.blocked,
    connecting: res.timings.connecting,
    tls_handshaking: res.timings.tls_handshaking,
    sending: res.timings.sending,
    waiting: res.timings.waiting,
    receiving: res.timings.receiving,
    duration: res.timings.duration,
    looking_up: res.timings.looking_up,
  }));
}
```

**`script_err.js`** — closed-port failure path (`vus:1, iterations:3`); nothing is listening on `127.0.0.1:65000`, so every request fails before connecting:

```javascript
import http from 'k6/http';

export const options = {
  vus: 1,
  iterations: 3,
  insecureSkipTLSVerify: true,
};

export default function () {
  const res = http.get('https://127.0.0.1:65000/'); // nothing listening
  console.log(JSON.stringify({
    iter: __ITER,
    proto: res.proto,
    status: res.status,
    error: res.error,
    error_code: res.error_code,
    blocked: res.timings.blocked,
    connecting: res.timings.connecting,
    tls_handshaking: res.timings.tls_handshaking,
    sending: res.timings.sending,
    waiting: res.timings.waiting,
    receiving: res.timings.receiving,
    duration: res.timings.duration,
  }));
}
```

**`script_timeout.js`** — mid-flight timeout (`vus:1, iterations:3`); hits `/delay` (2 s server sleep) with a per-request `250ms` timeout so connect/TLS/write succeed but no first byte arrives:

```javascript
import http from 'k6/http';

export const options = {
  vus: 1,
  iterations: 3,
  insecureSkipTLSVerify: true,
};

export default function () {
  // Connection + TLS + request-write succeed; server withholds the first
  // response byte for 2s, but the client times out after 250ms.
  const res = http.get('https://127.0.0.1:18443/delay', { timeout: '250ms' });
  console.log(JSON.stringify({
    iter: __ITER,
    proto: res.proto,
    status: res.status,
    error: res.error,
    error_code: res.error_code,
    blocked: res.timings.blocked,
    connecting: res.timings.connecting,
    tls_handshaking: res.timings.tls_handshaking,
    sending: res.timings.sending,
    waiting: res.timings.waiting,
    receiving: res.timings.receiving,
    duration: res.timings.duration,
  }));
}
```

**`script_scale.js`** — all-zero incidence probe (`vus:4, iterations:200`); logs **only** all-seven-zero rows against the healthy keep-alive server:

```javascript
import http from 'k6/http';

export const options = {
  vus: 4,
  iterations: 200,
  insecureSkipTLSVerify: true,
};

export default function () {
  const res = http.get('https://127.0.0.1:18443/');
  const t = res.timings;
  const allZero = (t.blocked === 0 && t.connecting === 0 && t.tls_handshaking === 0 &&
                   t.sending === 0 && t.waiting === 0 && t.receiving === 0 && t.duration === 0);
  // Only log the pathological all-zero rows (and always the count is derived below).
  if (allZero) {
    console.log(JSON.stringify({ iter: __ITER, ALLZERO: true, status: res.status }));
  }
}
```

**`script_h2.js`** — 50-VU cold concurrent HTTP/2 burst (`vus:50, iterations:50`) against the h2 server on `:18444`:

```javascript
import http from 'k6/http';

export const options = {
  vus: 50,
  iterations: 50,
  insecureSkipTLSVerify: true,
};

export default function () {
  const res = http.get('https://127.0.0.1:18444/');
  console.log(JSON.stringify({
    proto: res.proto,
    status: res.status,
    connecting: res.timings.connecting,
    tls_handshaking: res.timings.tls_handshaking,
    blocked: res.timings.blocked,
    waiting: res.timings.waiting,
  }));
}
```

**`script_h2_reuse.js`** — single-VU HTTP/2 sequential reuse (`vus:1, iterations:4`) against the h2 server on `:18444`:

```javascript
import http from 'k6/http';
export const options = { vus: 1, iterations: 4, insecureSkipTLSVerify: true };
export default function () {
  const res = http.get('https://127.0.0.1:18444/');
  console.log(JSON.stringify({
    proto: res.proto, status: res.status,
    connecting: res.timings.connecting,
    tls_handshaking: res.timings.tls_handshaking,
    waiting: res.timings.waiting,
  }));
}
```

**`run_blocked.sh`** — emits the §3.2 rows from real k6 `console.log` output (nothing hand-entered):

```bash
#!/usr/bin/env bash
# Runs the SAME unchanged script.js N times through the compiled k6 binary and
# prints, per run, the per-iteration blocked/connecting/tls_handshaking values
# parsed from k6's console.log lines. Every printed row is emitted BY THIS
# SCRIPT from the real k6 output (nothing hand-entered).
set -u
K6="${K6:-/tmp/blitzy/k6/blitzy-33959ffe-5247-4d91-b493-c741bbdb5309_6fd566/k6}"
SCRIPT="${SCRIPT:-/tmp/obs/script.js}"
RUNS="${RUNS:-8}"
for run in $(seq 1 "$RUNS"); do
  out="$(K6_NO_USAGE_REPORT=true "$K6" run "$SCRIPT" 2>&1)"
  echo "----- RUN $run -----"
  printf '%s\n' "$out" | python3 -c '
import sys, re, json
for line in sys.stdin:
    m = re.search(r"msg=\"(\{.*\})\"\s", line)
    if not m:
        continue
    d = json.loads(m.group(1).replace("\\\"", "\""))
    tag = "cold(new)" if d["iter"] == 0 else "warm(reuse)"
    print("  iter=%d %-11s blocked=%-10s connecting=%-8s tls_handshaking=%-8s"
          % (d["iter"], tag, d["blocked"], d["connecting"], d["tls_handshaking"]))
'
done
```

**`run_tests.sh`** — emits the §3.3 `*_EXIT`/`DATA_RACE_WARNINGS` lines from the real `$?` of each `go test` and a real `grep -c`:

```bash
#!/usr/bin/env bash
# Runs the k6 tracer test suite (a) normally and (b) under the Go race detector,
# printing the REAL exit code of each `go test` invocation and the REAL count of
# "WARNING: DATA RACE" banners grepped from the race run's own output. The
# EXIT/COUNT lines below are emitted by this wrapper from $? and grep -c, so they
# are genuine, not hand-entered.
set -u
REPO="${REPO:-/tmp/blitzy/k6/blitzy-33959ffe-5247-4d91-b493-c741bbdb5309_6fd566}"
cd "$REPO"
TESTS='TestTracer|TestTracerError|TestTracerNegativeHttpSendingValues|TestCancelledRequest'

echo "### NORMAL RUN"
echo "\$ go test -count=1 ./lib/netext/httpext/ -run \"$TESTS\" -v"
go test -count=1 ./lib/netext/httpext/ -run "$TESTS" -v 2>&1 | tee /tmp/obs/test_normal.raw.log
NORMAL_EXIT=${PIPESTATUS[0]}
echo "NORMAL_EXIT=$NORMAL_EXIT"

echo ""
echo "### RACE RUN"
echo "\$ CGO_ENABLED=1 go test -race -count=1 ./lib/netext/httpext/ -run \"$TESTS\" -v"
CGO_ENABLED=1 go test -race -count=1 ./lib/netext/httpext/ -run "$TESTS" -v 2>&1 | tee /tmp/obs/test_race.raw.log
RACE_EXIT=${PIPESTATUS[0]}
DATA_RACE_WARNINGS=$(grep -c 'WARNING: DATA RACE' /tmp/obs/test_race.raw.log)
echo "RACE_EXIT=$RACE_EXIT"
echo "DATA_RACE_WARNINGS=$DATA_RACE_WARNINGS"
```

### 7.2 Server lifecycle (start, readiness, explicit-PID teardown)

```text
$ (cd /tmp/obs && go build -o server server.go)      # exit 0
$ nohup /tmp/obs/server > /tmp/obs/server.out 2>&1 &  # HTTP/1.1 keep-alive server
$ cat /tmp/obs/server.out
READY url=https://127.0.0.1:18443 pid=105589
$ curl -sk https://127.0.0.1:18443/
hello from keep-alive server

$ (cd /tmp/obs && go build -o h2server h2server.go)  # exit 0
$ nohup /tmp/obs/h2server > /tmp/obs/h2server.out 2>&1 &  # HTTP/2 server
$ cat /tmp/obs/h2server.out
READY url=https://127.0.0.1:18444 pid=109712 http2=true
$ curl -sk --http2 -w '\nHTTP_VERSION=%{http_version}\n' https://127.0.0.1:18444/
hello h2 proto=HTTP/2.0
HTTP_VERSION=2
```

Both servers were stopped by **explicit PID** (never `pkill`/`killall`), each followed by a wait loop and a port-free check:

```text
$ kill -TERM 105589   # then wait until kill -0 fails
PID 105589 terminated
$ kill -TERM 109712   # then wait until kill -0 fails
PID 109712 terminated
$ (ss -ltnp 2>/dev/null || netstat -ltnp) | grep -E ':18443|:18444' || echo "both 18443 and 18444 FREE"
both 18443 and 18444 FREE
```

### 7.3 Repository-integrity and final-cleanup evidence

Throughout the investigation the source tree stayed unmodified (only the ignored `./k6` build artifact and the external `/tmp/obs` harness existed alongside it); at finalization the build artifact and harness are removed. Captured evidence:

```text
$ git status --ignored --porcelain          # state at finalization, before cleanup
 M blitzy/documentation/k6_ddc3b0b1d23c.md
!! k6
$ ls -l k6                                   # the ignored build artifact (65,574,927 bytes)
-rwxr-xr-x 1 root root 65574927 Jul 13 17:18 k6
$ rm -f k6 && echo "removed ./k6 build artifact"
removed ./k6 build artifact
$ ls k6 2>&1 || echo "./k6 is gone"
ls: cannot access 'k6': No such file or directory
./k6 is gone
$ ls -d /tmp/obs && rm -rf /tmp/obs && echo "removed external harness /tmp/obs"
/tmp/obs
removed external harness /tmp/obs
$ ls -d /tmp/obs 2>&1 || echo "/tmp/obs is gone"
ls: cannot access '/tmp/obs': No such file or directory
/tmp/obs is gone
$ ps -eo pid,args | grep -E "obs/(server|h2server)" | grep -v grep || echo "no leftover harness server processes"
no leftover harness server processes
$ for p in 18443 18444; do (echo >/dev/tcp/127.0.0.1/$p) 2>/dev/null && echo "port $p IN USE" || echo "port $p FREE (nothing listening)"; done
port 18443 FREE (nothing listening)
port 18444 FREE (nothing listening)
$ git status --ignored --porcelain          # after cleanup: only the deliverable remains
 M blitzy/documentation/k6_ddc3b0b1d23c.md
```

The only change this task makes to the repository is the addition/*correction* of this single document, `blitzy/documentation/k6_ddc3b0b1d23c.md`; no k6 source, test, configuration, or vendored file is modified.
