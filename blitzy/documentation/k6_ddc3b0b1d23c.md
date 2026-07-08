# k6 HTTP request‑timing metrics: are the `http_req_*` values trustworthy, or is there a measurement bug?

> Investigative Q&A — run‑first‑then‑write. Every runtime claim below is tagged **OBSERVED** and sits next to the exact
> command that produced it and that command's complete, unedited output. Every claim drawn purely from reading the source
> is tagged **INFERRED**. All observations come from the canonical path: the compiled default `k6` binary (the `k6/http`
> JavaScript module) and the production `httpext.Tracer` exercised through `transport.RoundTrip` — no debug shim or
> synthetic stand‑in.

---

## TL;DR (the verdict)

**Yes — you can trust the timing values, once you understand what each one measures. None of the five behaviors you
reported is a bug in k6.** Four of them (Q1, Q2, Q4, Q5) are the *correct, intended* behavior of k6's tracer and of Go's
`net/http/httptrace` connection‑lifecycle contract; they are `0`/near‑zero exactly when there is genuinely no new‑connection
work to measure. The fifth (Q3, the Windows all‑zeros) is **not a race in k6's measurement code** — it is a well‑known
**Windows platform timer‑resolution limitation** in Go's `time.Now()`, which k6 already documents and works around in its
own test suite and which is tracked upstream in Go (issues #8687 and #41087). The only item worth escalating is Q3, and it
is already an open Go issue, not a k6 defect. Concretely: use `http_req_duration` for per‑request latency (it deliberately
excludes connection setup); read `http_req_blocked` / `http_req_connecting` / `http_req_tls_handshaking` as *new‑connection*
cost that shows up mainly on the first request per connection (look at max/percentiles, not the median).

| Question | Behavior | Verdict | Evidence |
|---|---|---|---|
| **Q1** | `connecting` & `tls_handshaking` = exactly `0` on 2nd+ requests | Correct — reused keep‑alive connection, hooks not fired | OBSERVED |
| **Q2** | `blocked` bimodal (hundreds of ms vs near‑zero) | Correct — new‑connection setup folds into `blocked`; reuse ≈ 0 | OBSERVED |
| **Q3** | Windows: ALL timings occasionally `0` | Not a race — Windows `time.Now()` resolution artifact | INFERRED |
| **Q4** | `connReused == false` yet connect stamps == got‑conn stamp | Correct — CAS fallback fills never‑fired connect stamps | OBSERVED |
| **Q5** | `ConnectStart`/`ConnectDone` fire multiple times | Correct — Happy‑Eyeballs dual dial; atomic CAS de‑duplicates | OBSERVED |
| **Ultimate (a)** | Can I trust the values? | **Yes**, with the semantics understood | OBSERVED + code |
| **Ultimate (b)** | Escalate anything? | Only Q3 — already an open Go issue, no k6 change warranted | OBSERVED + INFERRED |

---

## 1. Build & environment

The investigation was performed against the k6 source at the branch this document is named for, **`k6_ddc3b0b1d23c`**, i.e.
commit **`ddc3b0b1d23c128e34e2792fc9075f9126e32375`** (`git rev-parse HEAD`). The working tree was clean
(`git status --porcelain` empty) before and after the investigation; the answer document is the only net change.

`go.mod` (L1–L5) pins the toolchain and the module is fully **vendored**, so the binary and tests build offline:

```text
module go.k6.io/k6

go 1.21

toolchain go1.21.13
```

Toolchain used: `go version go1.21.13 linux/amd64` (matches the `toolchain go1.21.13` directive). Dependencies are in
`vendor/`, so builds run with `GOFLAGS=-mod=vendor`.

### Canonical default build + banner (OBSERVED)

```bash
export PATH=/usr/local/go/bin:$PATH
export GOFLAGS=-mod=vendor GOTOOLCHAIN=local GOPATH=/tmp/gopath GOCACHE=/tmp/gocache
cd <repo>                      # the k6_ddc3b0b1d23c checkout
go build -o /tmp/k6 .          # canonical binary, named "k6"
/tmp/k6 version
```

Output (**OBSERVED**):

```text
k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)
```

**Banner‑basename nuance (OBSERVED).** The version‑details string `v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)` is
produced by `consts.FullVersion()` (`lib/consts/consts.go:16-52`; `Version = "0.55.0"` at `lib/consts/consts.go:12`) and is
independent of the binary's file name — the **prefix** is the executable's basename at runtime. Copying the exact same
binary to a different name proves it:

```bash
cp /tmp/k6 /tmp/k6bin
/tmp/k6 version
/tmp/k6bin version
```

```text
k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)
k6bin v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)
```

A normal user's binary is named `k6`, so the canonical banner is the `k6 …` form. The commit is stamped from VCS:
`FullVersion` reads `vcs.revision` (first 10 chars) and `vcs.modified` from `debug.ReadBuildInfo()`; a clean tree yields no
`-dirty` suffix (`lib/consts/consts.go:30-49`).

### Observation channels used (all canonical)

1. **`response.timings`** in a k6 JavaScript script run by the compiled `/tmp/k6` binary (the `k6/http` module path).
2. **`--http-debug=full`** — the built‑in wire‑dump transport (`lib/netext/httpext/httpdebug_transport.go`).
3. **`--out json`** — machine‑readable per‑request samples.
4. An **in‑package Go harness** exercising the production `httpext.Tracer` through the real `transport.RoundTrip`, mirroring
   `lib/netext/httpext/tracer_test.go`, used to read the unexported nanosecond timestamps (`getConn`, `connectStart`,
   `connectDone`, `tlsHandshakeStart`, `tlsHandshakeDone`, `gotConn`, `connReused`). It was written to
   `lib/netext/httpext/zz_investig_test.go`, run, then deleted; the repository was left clean.

Aside from that single temporary in‑package test file, every *other* artifact — the binary `/tmp/k6`, the local
server, and the JavaScript scripts — lived in `/tmp`, outside the repository. All of them (the in‑package harness
included) were removed afterward, leaving the working tree clean — `git status --porcelain` reports only
`blitzy/documentation/k6_ddc3b0b1d23c.md`. See the **Methods & reproducibility** appendix for the full, reproducible method.

---

## 2. How k6 measures a single request (background)

k6 attaches a fresh `httpext.Tracer` to **every** request and derives the timings from Go's `net/http/httptrace`
connection‑lifecycle hooks. The relevant lifecycle:

- **Per‑request tracer.** `transport.RoundTrip` creates a brand‑new `Tracer{}` for each request and attaches its hooks via
  `httptrace.WithClientTrace`:
  - `tracer := &Tracer{}` — `lib/netext/httpext/transport.go:205`
  - `req.WithContext(httptrace.WithClientTrace(ctx, tracer.Trace()))` — `lib/netext/httpext/transport.go:206`
  - `t.state.Transport.RoundTrip(reqWithTracer)` — `lib/netext/httpext/transport.go:207`
  - The doc‑comment on the struct is explicit: *"It's NOT safe to reuse Tracers between requests."* — `tracer.go:145`.
- **The eight hooks.** `Tracer.Trace()` (`tracer.go:161-173`) wires exactly eight `httptrace.ClientTrace` hooks — and
  **no** `DNSStart`/`DNSDone`, which is why DNS time is not a separate metric (it is absorbed into `blocked`):
  `GetConn`, `ConnectStart`, `ConnectDone`, `TLSHandshakeStart`, `TLSHandshakeDone`, `GotConn`, `WroteRequest`,
  `GotFirstResponseByte`.
- **Time source.** Every hook stamps `now()`, which is `return time.Now().UnixNano()` — `tracer.go:175-177`.
- **Derivation.** `Tracer.Done()` (`tracer.go:315`) turns the raw nanosecond timestamps into the `Trail` durations.
- **Emission.** `measureAndEmitMetrics` calls `trail := unfReq.tracer.Done()` (`transport.go:78`), then
  `trail.SaveSamples(...)` (`transport.go:146`) and `metrics.PushIfNotDone(...)` (`transport.go:164`). `SaveSamples`
  (`tracer.go:44`) emits eight samples — `http_reqs` (counter, value `1`) plus the seven timing Trends — reserving a ninth
  slot for `http_req_failed` (`tracer.go:47` comment "this is with 1 more for a possible HTTPReqFailed"), which
  `measureAndEmitMetrics` appends separately.
- **User‑facing path.** In a script, `http.get(url)` → `Client.Request` (`js/modules/k6/http/request.go:39`) →
  `httpext.MakeRequest` (`js/modules/k6/http/request.go:51`) → the `Trail` surfaces as `response.timings`, whose JSON field
  names are defined by `ResponseTimings` (`lib/netext/httpext/response.go:34-43`: `connecting` at L39,
  `tls_handshaking` at L40).

The `Trail` fields carry their own documentation of intent (`tracer.go:20-30`):

```text
ConnDuration                     // Connecting + TLSHandshaking
Duration        // Total request duration, EXCLUDING DNS lookup and connect time.
Blocked         // Waiting to acquire a connection.
Connecting      // Connecting to remote host.
TLSHandshaking  // Executing TLS handshake.
Sending         // Writing request.
Waiting         // Waiting for first byte.
Receiving       // Receiving response.
```

### Hook‑flow diagram

```mermaid
graph LR
    A[GetConn: getConn = now] --> B[GotConn: gotConn = now, connReused]
    B -->|Reused=true| C[Swap connectStart=connectDone=now, TLS=now]
    B -->|Reused=false| D[CAS connectStart/connectDone if never fired]
    A -.new conn.-> E[ConnectStart CAS] --> F[ConnectDone CAS] --> G[TLS Start/Done CAS]
    C --> H[Done: Blocked, Connecting, TLS, Sending, Waiting, Receiving]
    D --> H
    G --> B
    H --> I[SaveSamples: emit http_reqs + 7 http_req_* timing metrics]
```

The seven timing-metric names are declared in `metrics/builtin.go` (`http_req_blocked` L18, `http_req_connecting` L19,
`http_req_tls_handshaking` L20, plus `http_req_duration` L17, `http_req_sending`/`waiting`/`receiving` L21‑23) and
registered as `(Trend, Time)` metrics in `RegisterBuiltinMetrics` (`metrics/builtin.go:91-97`). Emitted values are
milliseconds: `SaveSamples` converts each duration with `metrics.D(...)`, and `metrics.D` divides by
`timeUnit = time.Millisecond` (`metrics/units.go:7`, `metrics/units.go:11-13`).

---

## Q1 — Why do `http_req_connecting` and `http_req_tls_handshaking` read exactly `0` on the 2nd+ sequential requests? — **OBSERVED**

**Direct answer: this is correct, expected behavior — not a bug.** Requests 2, 3, … reuse the keep‑alive connection opened
by request 1. For a reused connection the Go standard library does **not** invoke the `ConnectStart` / `ConnectDone` /
`TLSHandshakeStart` / `TLSHandshakeDone` trace hooks at all — there is no new TCP connect and no new TLS handshake to time.
k6's tracer therefore has nothing to measure for those phases, and `Done()` reports exactly `0`. (The network activity you
still see is the request/response bytes themselves flowing over the already‑open socket — see the `--http-debug` evidence in
the **Observation channels** appendix, which shows the reused request still sends a full GET and receives a full 200.)

**Command** (canonical binary, `seq.js`, TLS HTTP/1.1 keep‑alive, 4 sequential GETs):

```bash
URL="$TLS_H1_URL" N=4 /tmp/k6 run --quiet --no-summary seq.js
```

**Complete captured output — RUN #1** (**OBSERVED**, complete unedited k6 output; each line is the `console.log`
payload wrapped by k6's structured logger as `time=… level=info msg="…" source=console`; timings in ms):

```text
time="2026-07-08T05:10:13Z" level=info msg="req#0 proto=HTTP/1.1 status=200 remote=127.0.0.1:42367 blocked=2.551 connecting=0.134 tls=2.340 sending=0.039 waiting=30.455 receiving=0.097 duration=30.591" source=console
time="2026-07-08T05:10:13Z" level=info msg="req#1 proto=HTTP/1.1 status=200 remote=127.0.0.1:42367 blocked=0.003 connecting=0.000 tls=0.000 sending=0.018 waiting=30.428 receiving=0.071 duration=30.517" source=console
time="2026-07-08T05:10:14Z" level=info msg="req#2 proto=HTTP/1.1 status=200 remote=127.0.0.1:42367 blocked=0.003 connecting=0.000 tls=0.000 sending=0.016 waiting=30.468 receiving=0.097 duration=30.580" source=console
time="2026-07-08T05:10:14Z" level=info msg="req#3 proto=HTTP/1.1 status=200 remote=127.0.0.1:42367 blocked=0.002 connecting=0.000 tls=0.000 sending=0.013 waiting=30.390 receiving=0.043 duration=30.446" source=console
```

**Stability — RUN #2** (**OBSERVED**, same unchanged input; identical pattern; complete unedited output):

```text
time="2026-07-08T05:10:14Z" level=info msg="req#0 proto=HTTP/1.1 status=200 remote=127.0.0.1:42367 blocked=2.432 connecting=0.135 tls=2.204 sending=0.077 waiting=30.350 receiving=0.108 duration=30.534" source=console
time="2026-07-08T05:10:14Z" level=info msg="req#1 proto=HTTP/1.1 status=200 remote=127.0.0.1:42367 blocked=0.003 connecting=0.000 tls=0.000 sending=0.019 waiting=30.375 receiving=0.041 duration=30.436" source=console
time="2026-07-08T05:10:14Z" level=info msg="req#2 proto=HTTP/1.1 status=200 remote=127.0.0.1:42367 blocked=0.007 connecting=0.000 tls=0.000 sending=0.010 waiting=30.358 receiving=0.066 duration=30.435" source=console
time="2026-07-08T05:10:14Z" level=info msg="req#3 proto=HTTP/1.1 status=200 remote=127.0.0.1:42367 blocked=0.002 connecting=0.000 tls=0.000 sending=0.018 waiting=30.387 receiving=0.094 duration=30.499" source=console
```

Across both runs, `connecting` and `tls` are the **first request's** measured values and **exactly `0.000`** on every
subsequent (reused) request. Stable across ≥2 runs.

**HTTP/2 variant** (**OBSERVED**, `URL="$TLS_H2_URL"`) — the same invariant holds, and reused‑request `blocked` is even
`0.000`:

```text
time="2026-07-08T05:10:24Z" level=info msg="req#0 proto=HTTP/2.0 status=200 remote=127.0.0.1:42919 blocked=2.576 connecting=0.127 tls=2.284 sending=0.168 waiting=30.737 receiving=0.108 duration=31.014" source=console
time="2026-07-08T05:10:24Z" level=info msg="req#1 proto=HTTP/2.0 status=200 remote=127.0.0.1:42919 blocked=0.000 connecting=0.000 tls=0.000 sending=0.058 waiting=30.506 receiving=0.071 duration=30.635" source=console
time="2026-07-08T05:10:24Z" level=info msg="req#2 proto=HTTP/2.0 status=200 remote=127.0.0.1:42919 blocked=0.000 connecting=0.000 tls=0.000 sending=0.067 waiting=30.448 receiving=0.075 duration=30.589" source=console
time="2026-07-08T05:10:25Z" level=info msg="req#3 proto=HTTP/2.0 status=200 remote=127.0.0.1:42919 blocked=0.000 connecting=0.000 tls=0.000 sending=0.061 waiting=30.490 receiving=0.070 duration=30.621" source=console
```

**Plaintext HTTP/1.1 variant** (**OBSERVED**, `URL="$PLAIN_URL"`) — note `tls=0.000` even on **req#0**:

```text
time="2026-07-08T05:10:25Z" level=info msg="req#0 proto=HTTP/1.1 status=200 remote=127.0.0.1:37481 blocked=0.192 connecting=0.132 tls=0.000 sending=0.073 waiting=30.412 receiving=0.101 duration=30.586" source=console
time="2026-07-08T05:10:25Z" level=info msg="req#1 proto=HTTP/1.1 status=200 remote=127.0.0.1:37481 blocked=0.003 connecting=0.000 tls=0.000 sending=0.016 waiting=30.465 receiving=0.084 duration=30.566" source=console
time="2026-07-08T05:10:25Z" level=info msg="req#2 proto=HTTP/1.1 status=200 remote=127.0.0.1:37481 blocked=0.003 connecting=0.000 tls=0.000 sending=0.015 waiting=30.383 receiving=0.082 duration=30.480" source=console
time="2026-07-08T05:10:25Z" level=info msg="req#3 proto=HTTP/1.1 status=200 remote=127.0.0.1:37481 blocked=0.003 connecting=0.000 tls=0.000 sending=0.014 waiting=30.379 receiving=0.050 duration=30.443" source=console
```

> **Distinct nuance:** `tls_handshaking == 0` has **two** independent causes — (a) a *reused* connection (no handshake
> happens), **or** (b) a *plaintext* (non‑TLS) request, where there is no handshake at all even on the first request. Both
> are visible above.

**Internal‑timestamp confirmation** (**OBSERVED**, in‑package Go harness reading the unexported `int64` nanosecond fields
of the production `Tracer`). Command:

```bash
go test -run 'TestZZInvestigCanonical' -count=1 -v ./lib/netext/httpext/
```

Complete, unedited output:

```text
=== RUN   TestZZInvestigCanonical
===== CANONICAL request #0 (expected Reused=false) =====
[req#0] getConn=1783487617111983700 connectStart=1783487617112038743 connectDone=1783487617112160203 tlsStart=1783487617112180043 tlsDone=1783487617114302656 gotConn=1783487617114309634 connReused=false
    -> connectStart==connectDone? false ; connectStart==gotConn? false ; connectDone==gotConn? false
    Trail: ConnReused=false Blocked=2.325934ms Connecting=121.46µs TLSHandshaking=2.122613ms Sending=50.627µs Waiting=242.681µs Receiving=55.249µs ConnDuration=2.244073ms Duration=348.557µs
    metric http_req_connecting          = 0.12146
    metric http_req_tls_handshaking     = 2.122613
===== CANONICAL request #1 (expected Reused=true) =====
[req#1] getConn=1783487617114684919 connectStart=1783487617114686599 connectDone=1783487617114686599 tlsStart=1783487617114686599 tlsDone=1783487617114686599 gotConn=1783487617114686599 connReused=true
    -> connectStart==connectDone? true ; connectStart==gotConn? true ; connectDone==gotConn? true
    Trail: ConnReused=true Blocked=1.68µs Connecting=0s TLSHandshaking=0s Sending=13.258µs Waiting=93.581µs Receiving=38.468µs ConnDuration=0s Duration=145.307µs
    metric http_req_connecting          = 0
    metric http_req_tls_handshaking     = 0
--- PASS: TestZZInvestigCanonical (0.00s)
PASS
ok  	go.k6.io/k6/lib/netext/httpext	0.010s
```

On the first request the connect/TLS timestamps are all **distinct**; on the reused request they are all **identical**
(`connectStart == connectDone == gotConn`), which is what forces the durations to `0`.

**Responsible code (cause → effect):**

- `GotConn` (`lib/netext/httpext/tracer.go:256`) sets `t.connReused = info.Reused` (`:262`). When `info.Reused == true`
  (`:271`) it uses `atomic.SwapInt64` to force `connectStart = connectDone = now` and, when the connection is a `*tls.Conn`,
  `tlsHandshakeStart = tlsHandshakeDone = now` (`:271-277`). Because start and done are set to the *same* `now`, the spans
  are zero.
- `Done()` (`:315`) computes `trail.Connecting = connectDone - connectStart` only when both are non‑zero (`:340-341`) → `0`,
  and `trail.TLSHandshaking = tlsHandshakeDone - tlsHandshakeStart` (`:343-344`) → `0`.
- Why the stdlib doesn't fire the hooks on reuse is documented on the hooks themselves:
  - `ConnectStart` — *"If the connection is reused, this won't be called."* (`:191-196`), method at `:197`.
  - `ConnectDone` — same note (`:204-212`), method at `:213`.
  - `TLSHandshakeStart` (`:224-229`, method `:230`) and `TLSHandshakeDone` (`:234-241`, method `:242`) — same note.
- Metric identity: `http_req_connecting` / `http_req_tls_handshaking` names at `metrics/builtin.go:19-20`, registered as
  `(Trend, Time)` at `metrics/builtin.go:93-94`; JS field names `connecting` / `tls_handshaking` at
  `lib/netext/httpext/response.go:39-40`; ms conversion via `metrics.D` (`metrics/units.go:11-13`).

---

## Q2 — Why is `http_req_blocked` sometimes hundreds of ms and other times near‑zero for "identical" requests? — **OBSERVED** (reproduced via repeated identical runs)

> Your example, preserved verbatim: *"sometimes the 'blocked' timing shows massive values like 500ms when other times it's
> near zero for identical requests."*

**Direct answer: also expected, not a bug.** `http_req_blocked` is the time from *"we asked for a connection"* to *"we got
one."* When a request opens a **new** connection, the entire TCP dial + (optional) TLS handshake — plus any wait for a free
connection‑pool slot, plus DNS (k6 wires no DNS hooks, so DNS folds into `blocked`) — is counted here, so `blocked` is
large. When a request reuses an **idle pooled** connection there is essentially nothing to wait for, so `blocked` is
near‑zero. "Identical" requests differ only in whether they land on a warm connection. The magnitude of the large case
scales with network RTT, the number of TLS round‑trips, and connection‑pool contention — which is why against a real WAN
target you see hundreds of ms.

The behavior is inherently **bimodal**, so per the run‑to‑run rule it is characterized by running the *same unchanged input*
repeatedly and reporting the distribution (not by stabilizing it).

**Command** (repeated identical load test; the summary reports the distribution):

```bash
URL="$TLS_H1_URL" /tmp/k6 run --vus 50 --duration 5s scale.js
```

**Complete captured output — LOCAL RUN #1** (**OBSERVED**, complete unedited k6 run output — banner, progress, and full end‑of‑test summary):

```text
         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: scale.js
        output: -

     scenarios: (100.00%) 1 scenario, 50 max VUs, 35s max duration (incl. graceful stop):
              * default: 50 looping VUs for 5s (gracefulStop: 30s)


running (01.0s), 50/50 VUs, 1550 complete and 0 interrupted iterations
default   [  20% ] 50 VUs  1.0s/5s

running (02.0s), 50/50 VUs, 3200 complete and 0 interrupted iterations
default   [  40% ] 50 VUs  2.0s/5s

running (03.0s), 50/50 VUs, 4801 complete and 0 interrupted iterations
default   [  60% ] 50 VUs  3.0s/5s

running (04.0s), 50/50 VUs, 6437 complete and 0 interrupted iterations
default   [  80% ] 50 VUs  4.0s/5s

running (05.0s), 50/50 VUs, 8050 complete and 0 interrupted iterations
default   [ 100% ] 50 VUs  5.0s/5s

     data_received..................: 1.4 MB 282 kB/s
     data_sent......................: 854 kB 170 kB/s
     http_req_blocked...............: avg=78.95µs min=667ns   med=1.34µs  max=18.29ms p(90)=2.02µs  p(95)=2.48µs 
     http_req_connecting............: avg=4.78µs  min=0s      med=0s      max=2.69ms  p(90)=0s      p(95)=0s     
     http_req_duration..............: avg=30.72ms min=30.1ms  med=30.7ms  max=32.59ms p(90)=31.09ms p(95)=31.18ms
       { expected_response:true }...: avg=30.72ms min=30.1ms  med=30.7ms  max=32.59ms p(90)=31.09ms p(95)=31.18ms
     http_req_failed................: 0.00%  0 out of 8130
     http_req_receiving.............: avg=27.11µs min=9.86µs  med=22.18µs max=1.65ms  p(90)=33.19µs p(95)=39.6µs 
     http_req_sending...............: avg=9.17µs  min=3.44µs  med=6.21µs  max=1.37ms  p(90)=9.19µs  p(95)=11.32µs
     http_req_tls_handshaking.......: avg=71.16µs min=0s      med=0s      max=17.79ms p(90)=0s      p(95)=0s     
     http_req_waiting...............: avg=30.68ms min=30.07ms med=30.67ms max=32.45ms p(90)=31.06ms p(95)=31.14ms
     http_reqs......................: 8130   1616.440751/s
     iteration_duration.............: avg=30.84ms min=30.13ms med=30.74ms max=48.93ms p(90)=31.15ms p(95)=31.24ms
     iterations.....................: 8130   1616.440751/s
     vus............................: 50     min=50        max=50
     vus_max........................: 50     min=50        max=50


running (05.0s), 00/50 VUs, 8130 complete and 0 interrupted iterations
default ✓ [ 100% ] 50 VUs  5s
```

**LOCAL RUN #2** (**OBSERVED**, same unchanged input — stable *shape*; complete unedited k6 run output):

```text
         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: scale.js
        output: -

     scenarios: (100.00%) 1 scenario, 50 max VUs, 35s max duration (incl. graceful stop):
              * default: 50 looping VUs for 5s (gracefulStop: 30s)


running (01.0s), 50/50 VUs, 1552 complete and 0 interrupted iterations
default   [  20% ] 50 VUs  1.0s/5s

running (02.0s), 50/50 VUs, 3200 complete and 0 interrupted iterations
default   [  40% ] 50 VUs  2.0s/5s

running (03.0s), 50/50 VUs, 4802 complete and 0 interrupted iterations
default   [  60% ] 50 VUs  3.0s/5s

running (04.0s), 50/50 VUs, 6450 complete and 0 interrupted iterations
default   [  80% ] 50 VUs  4.0s/5s

running (05.0s), 50/50 VUs, 8051 complete and 0 interrupted iterations
default   [ 100% ] 50 VUs  5.0s/5s

     data_received..................: 1.4 MB 281 kB/s
     data_sent......................: 851 kB 169 kB/s
     http_req_blocked...............: avg=86.92µs min=603ns   med=1.4µs   max=20.75ms p(90)=2.22µs  p(95)=2.88µs 
     http_req_connecting............: avg=4.82µs  min=0s      med=0s      max=2.13ms  p(90)=0s      p(95)=0s     
     http_req_duration..............: avg=30.75ms min=30.07ms med=30.7ms  max=37.25ms p(90)=31.21ms p(95)=31.43ms
       { expected_response:true }...: avg=30.75ms min=30.07ms med=30.7ms  max=37.25ms p(90)=31.21ms p(95)=31.43ms
     http_req_failed................: 0.00%  0 out of 8102
     http_req_receiving.............: avg=33.38µs min=9.67µs  med=22.86µs max=1.92ms  p(90)=38.43µs p(95)=49.98µs
     http_req_sending...............: avg=11.75µs min=3.27µs  med=6.44µs  max=1.87ms  p(90)=9.94µs  p(95)=13.46µs
     http_req_tls_handshaking.......: avg=78.34µs min=0s      med=0s      max=20.5ms  p(90)=0s      p(95)=0s     
     http_req_waiting...............: avg=30.7ms  min=30.06ms med=30.66ms max=37.2ms  p(90)=31.16ms p(95)=31.31ms
     http_reqs......................: 8102   1612.167423/s
     iteration_duration.............: avg=30.88ms min=30.1ms  med=30.75ms max=51.79ms p(90)=31.27ms p(95)=31.59ms
     iterations.....................: 8102   1612.167423/s
     vus............................: 50     min=50        max=50
     vus_max........................: 50     min=50        max=50


running (05.0s), 00/50 VUs, 8102 complete and 0 interrupted iterations
default ✓ [ 100% ] 50 VUs  5s
```

> **Interpretation:** the distribution is **bimodal**. The median is ≈ 1.34 µs (RUN #1) / 1.4 µs (RUN #2) — the ~99.4 % of
> the ~8100 requests per run that reuse a warm connection. The max is in the tens of ms — the ~50 *new* connections (one per
> VU) that fold their TLS handshake into `blocked`. Note how the `blocked` max (`18.29ms` / `20.75ms`) tracks the
> `tls_handshaking` max (`17.79ms` / `20.5ms`): the new‑connection TLS handshake is exactly what makes `blocked` large.
> `connecting`/`tls_handshaking` have a median of `0s` (all the reused requests) with a nonzero max (the new connections) —
> the same reuse effect as Q1, now at scale.

**Remote WAN magnitude** (**OBSERVED** — substantiates the "hundreds of ms" you saw against a real target). Baseline with
`curl` first, then k6:

```bash
curl -sS -o /dev/null -w "connect=%{time_connect}s appconnect=%{time_appconnect}s total=%{time_total}s\n" https://quickpizza.grafana.com/
/tmp/k6 run --vus 30 --duration 5s remote.js     # target https://quickpizza.grafana.com/
```

```text
# curl baseline (one new connection):
connect=0.108596s appconnect=0.154064s total=0.196787s

# k6 RUN #1 (complete unedited output):
         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: remote.js
        output: -

     scenarios: (100.00%) 1 scenario, 30 max VUs, 35s max duration (incl. graceful stop):
              * default: 30 looping VUs for 5s (gracefulStop: 30s)


running (01.0s), 30/30 VUs, 898 complete and 0 interrupted iterations
default   [  20% ] 30 VUs  1.0s/5s

running (02.0s), 30/30 VUs, 1951 complete and 0 interrupted iterations
default   [  40% ] 30 VUs  2.0s/5s

running (03.0s), 30/30 VUs, 3021 complete and 0 interrupted iterations
default   [  60% ] 30 VUs  3.0s/5s

running (04.0s), 30/30 VUs, 4103 complete and 0 interrupted iterations
default   [  80% ] 30 VUs  4.0s/5s

running (05.0s), 30/30 VUs, 5156 complete and 0 interrupted iterations
default   [ 100% ] 30 VUs  5.0s/5s

     data_received..................: 18 MB  3.4 MB/s
     data_sent......................: 293 kB 56 kB/s
     http_req_blocked...............: avg=758.39µs min=154ns   med=282ns   max=159.24ms p(90)=411ns   p(95)=472ns   
     http_req_connecting............: avg=173.2µs  min=0s      med=0s      max=43.99ms  p(90)=0s      p(95)=0s      
     http_req_duration..............: avg=28.25ms  min=20.97ms med=22.64ms max=254.09ms p(90)=34.08ms p(95)=43.4ms  
       { expected_response:true }...: avg=28.25ms  min=20.97ms med=22.64ms max=254.09ms p(90)=34.08ms p(95)=43.4ms  
     http_req_failed................: 0.00%  0 out of 5191
     http_req_receiving.............: avg=81.47µs  min=12.73µs med=49.13µs max=32.33ms  p(90)=86.21µs p(95)=100.87µs
     http_req_sending...............: avg=28.33µs  min=13.14µs med=21.97µs max=1.43ms   p(90)=36.44µs p(95)=44.29µs 
     http_req_tls_handshaking.......: avg=183.06µs min=0s      med=0s      max=45.63ms  p(90)=0s      p(95)=0s      
     http_req_waiting...............: avg=28.14ms  min=20.62ms med=22.43ms max=253.99ms p(90)=33.85ms p(95)=43.32ms 
     http_reqs......................: 5191   998.58366/s
     iteration_duration.............: avg=29.06ms  min=21.02ms med=22.96ms max=254.17ms p(90)=35.3ms  p(95)=43.56ms 
     iterations.....................: 5191   998.58366/s
     vus............................: 30     min=30        max=30
     vus_max........................: 30     min=30        max=30


running (05.2s), 00/30 VUs, 5191 complete and 0 interrupted iterations
default ✓ [ 100% ] 30 VUs  5s

# k6 RUN #2 (complete unedited output):
         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: remote.js
        output: -

     scenarios: (100.00%) 1 scenario, 30 max VUs, 35s max duration (incl. graceful stop):
              * default: 30 looping VUs for 5s (gracefulStop: 30s)


running (01.0s), 30/30 VUs, 883 complete and 0 interrupted iterations
default   [  20% ] 30 VUs  1.0s/5s

running (02.0s), 30/30 VUs, 1890 complete and 0 interrupted iterations
default   [  40% ] 30 VUs  2.0s/5s

running (03.0s), 30/30 VUs, 2870 complete and 0 interrupted iterations
default   [  60% ] 30 VUs  3.0s/5s

running (04.0s), 30/30 VUs, 3815 complete and 0 interrupted iterations
default   [  80% ] 30 VUs  4.0s/5s

running (05.0s), 30/30 VUs, 4783 complete and 0 interrupted iterations
default   [ 100% ] 30 VUs  5.0s/5s

     data_received..................: 16 MB  3.2 MB/s
     data_sent......................: 273 kB 54 kB/s
     http_req_blocked...............: avg=483.37µs min=147ns   med=272ns   max=103.79ms p(90)=387ns   p(95)=455ns   
     http_req_connecting............: avg=196.39µs min=0s      med=0s      max=44.81ms  p(90)=0s      p(95)=0s      
     http_req_duration..............: avg=30.69ms  min=20.83ms med=32.09ms max=308.35ms p(90)=43.96ms p(95)=44.28ms 
       { expected_response:true }...: avg=30.69ms  min=20.83ms med=32.09ms max=308.35ms p(90)=43.96ms p(95)=44.28ms 
     http_req_failed................: 0.00%  0 out of 4817
     http_req_receiving.............: avg=137.6µs  min=13.98µs med=47.74µs max=275.59ms p(90)=84.75µs p(95)=104.78µs
     http_req_sending...............: avg=26.84µs  min=14.03µs med=21.69µs max=1.22ms   p(90)=34.86µs p(95)=42.17µs 
     http_req_tls_handshaking.......: avg=205.88µs min=0s      med=0s      max=46.07ms  p(90)=0s      p(95)=0s      
     http_req_waiting...............: avg=30.53ms  min=20.55ms med=32.02ms max=252.6ms  p(90)=43.87ms p(95)=44.2ms  
     http_reqs......................: 4817   955.595917/s
     iteration_duration.............: avg=31.23ms  min=20.89ms med=32.15ms max=308.41ms p(90)=44.04ms p(95)=44.37ms 
     iterations.....................: 4817   955.595917/s
     vus............................: 30     min=30        max=30
     vus_max........................: 30     min=30        max=30


running (05.0s), 00/30 VUs, 4817 complete and 0 interrupted iterations
default ✓ [ 100% ] 30 VUs  5s
```

> On the WAN target the `blocked` **max is 104–159 ms on new connections** versus a **median of ~272–282 ns on reuse** — a
> ~380,000–565,000× spread between the two modes, stable across two runs. This host is fast (~20 ms RTT, `duration` median
> ≈ 22.6 ms / 32.1 ms); the exact `blocked` magnitude depends on RTT, the number of TLS round‑trips, and pool contention.
> Your reported ~500 ms simply reflects a higher‑latency target and/or more new‑connection acquisitions (e.g. a larger VU
> ramp hitting a cold pool, or a target that negotiates TLS more slowly). The mechanism is identical to what is observed
> here; only the magnitude of the "new connection" mode scales up. *(Honest note: the maximum I could observe on this
> particular target was ~159 ms, not 500 ms — the 500 ms figure is consistent with the same mechanism against a slower
> endpoint, but I report the ~159 ms I actually measured rather than forcing a 500 ms result.)*

**Responsible code (cause → effect):** `Done()` computes

```go
if t.gotConn != 0 && t.getConn != 0 && t.gotConn > t.getConn {
    trail.Blocked = time.Duration(t.gotConn - t.getConn)
}
```

only when `gotConn > getConn` (`lib/netext/httpext/tracer.go:323-325`). `getConn` is stamped by `GetConn`
(`tracer.go:187-188`) — and per its doc‑comment `GetConn` *"is called even if there's already an idle cached connection
available"* (`:180-186`), which is why even a pooled hit still produces a tiny positive `blocked`. `gotConn` is stamped by
`GotConn` (`:256-262`). For a **new** connection the `getConn → gotConn` interval spans dial + connect + TLS (large); for a
**pooled** hit it is a few hundred nanoseconds. Because `Tracer.Trace()` wires **no** `DNSStart`/`DNSDone` hooks
(`:161-173`), DNS time is not a separate metric — it is absorbed into `blocked`. Metric name `http_req_blocked` at
`metrics/builtin.go:18`, registered `(Trend, Time)` at `:92`.

---

## Q3 — On Windows, why do ALL timing metrics occasionally read `0` for random requests? Is it a race in the measurement code? — **INFERRED** (not reproducible on the Linux investigation container)

**Direct answer (INFERRED): it is *not* a data race in k6. It is a Windows platform timer‑resolution artifact.** k6's
tracer timestamps everything with `time.Now()`, and Windows' `time.Now()` has historically had coarse resolution (roughly
1–15 ms). When two hook events for a fast request occur within a single timer tick, they receive the *same* nanosecond
value, so the corresponding duration computes as `end − start = 0`. When several phases of a very fast request all fall in
the same tick, *all* of them can read `0`. This is a known Go‑platform limitation, already tracked upstream in Go — not a
k6 defect.

**Why this cannot be reproduced here (stated plainly):** the investigation container is Linux (`go1.21.13 linux/amd64`),
whose `time.Now()` is high‑resolution (the nanosecond timestamps in every harness above are strictly increasing and
distinct), so the zeros do not occur. Fabricating a Windows run would violate the run‑first rule, so this answer is grounded
in code reading plus documented upstream evidence and is labeled **INFERRED** throughout.

**Cross‑platform time source (INFERRED from code).** `now()` returns `time.Now().UnixNano()` at
`lib/netext/httpext/tracer.go:175-177`. `grep` confirms `tracer.go` calls `time.Now()` at exactly two sites — the `now()`
helper and the final `done` stamp in `Done()` — and there is **no** OS branch, **no** `QueryPerformanceCounter`, and **no**
alternative clock anywhere:

```bash
grep -n "time.Now" lib/netext/httpext/tracer.go
grep -n "runtime.GOOS" lib/netext/httpext/tracer.go || echo "(no match)"
grep -n "QueryPerformanceCounter" lib/netext/httpext/tracer.go || echo "(no match)"
grep -c "atomic\." lib/netext/httpext/tracer.go
```

```text
176:	return time.Now().UnixNano()
316:	done := time.Now()
(no match)
(no match)
21
```

So `now()` is genuinely cross‑platform — Windows uses the very same `time.Now()`. The only Windows‑specific file in the
package, `lib/netext/httpext/error_codes_syscall_windows.go`, contains **no** clock override (it only maps
`syscall.WSAECONNRESET` to an error code), corroborating that nothing in k6 special‑cases the Windows clock.

To confirm the mechanism is race‑free in practice — not merely by reading it — the production package's own test
suite was run under the Go race detector (**OBSERVED**):

```bash
go test -race -count=1 ./lib/netext/httpext/
```

```text
ok  	go.k6.io/k6/lib/netext/httpext	4.039s
```

**Not a race (mechanism read from the code; race‑freedom OBSERVED via `-race` above).** The correct statement is *not*
that every shared field is guarded by `sync/atomic` — it is that the design is deliberately race‑free, and the two fields
the zero‑duration reasoning depends on are in fact written and read with **plain, non‑atomic assignments** on purpose.
`GetConn` sets `t.getConn = now()` (`lib/netext/httpext/tracer.go:188`) and `GotConn` sets `t.gotConn = now` and
`t.connReused = info.Reused` (`:261-262`), directly beneath the comment that explains why (`:259-260`): *"This shouldn't be
called multiple times so no synchronization here, it's better for the race detector to panic if we're wrong."* `Done()` then
derives `Blocked` from **direct** (non‑atomic) reads of exactly those two fields —
`if t.gotConn != 0 && t.getConn != 0 && t.gotConn > t.getConn { trail.Blocked = time.Duration(t.gotConn - t.getConn) }`
(`:323-324`). What the **21** `atomic.` call sites the `grep -c` above counts actually guard is the *other* group of
timestamps — `connectStart`, `connectDone`, `tlsHandshakeStart`, `tlsHandshakeDone`, `wroteRequest`, `gotFirstResponseByte` —
which legitimately can be written more than once (from the parallel dial goroutines during dual‑stack "Happy Eyeballs"
setup) or *after* `Done()` has already returned (for a cancelled request): `CompareAndSwapInt64` keeps only the first write
in the hooks (`:201`, `:218`, `:231`, `:244`, `:311`), `atomic.StoreInt64` records `WroteRequest` (`:301`), `SwapInt64`
forces the reuse values in the `Reused==true` branch (`:272-276`), `CompareAndSwapInt64` fills any never‑fired stamp in
the `Reused==false` else‑branch (`:287-291`), and `LoadInt64` reads them all back in `Done()` (`:332-338`). `Done()`
documents that intent (`:327-331`): *"we have to use atomics here as well (or use global Tracer locking) so we can avoid
data races."* So the zeros are a **value** artifact — two events legitimately share one coarse timestamp — **not** a torn or
racy read, and the race detector agrees.

**The maintainers' own corroboration.** `lib/netext/httpext/tracer_test.go` defines `const traceDelay = 100 * time.Millisecond`
(`:28`) and, in `getTestTracer`, when `runtime.GOOS == "windows"` (`:33`) it wraps every hook to `time.Sleep(traceDelay)`
before recording. Its comment (`:34-40`) names the exact cause and the upstream issues:

```go
// HACK: Time resolution is not as accurate on Windows, see:
//  https://github.com/golang/go/issues/8687
//  https://github.com/golang/go/issues/41087
// Which seems to be causing some metrics to have a value of 0,
// since e.g. ConnectStart and ConnectDone could register the same time.
// So we force delays in the ClientTrace event handlers
// to hopefully reduce the chances of this happening.
```

This is the decisive evidence that the k6 team already attributes the Windows zeros to timer resolution (specifically the
case where "ConnectStart and ConnectDone could register the same time"), not to a race — and that they mitigate it *in the
tests* by inserting delays, without altering the production time source. *(Note: this document cites the actual working‑copy
line for the OS check, `:33`; some earlier notes referenced `:31`.)*

**Upstream issues to cite:** Go **#8687** ("windows: time.Now() resolution is very low" / system clock resolution) and Go
**#41087** ("time: `Time` has different precision depending on the operating system") — the two named in the k6 test —
plus the related **#11313**, **#31160**, **#51530**, and **#67066** (which proposes using `QueryPerformanceCounter` for a
high‑resolution clock on Windows). The root cause lives entirely in the Go runtime's Windows clock, not in k6.

---

## Q4 — Why can a connection be flagged "not reused" while the connect timestamps equal the got‑connection timestamp (zero connect duration)? — **OBSERVED** (via the production `Tracer` hook methods)

> Your example, preserved verbatim: *"sometimes a connection is flagged as 'not reused' but the connect timestamps show the
> same value as the got connection timestamp."*

**Direct answer: not a bug.** When `GotConn` fires with `Reused == false` but the `ConnectStart` / `ConnectDone` hooks
never fired for this request's tracer, the `else` branch fills the never‑set connect timestamps with `now` via
compare‑and‑swap. That makes `connectStart == connectDone == gotConn`, so the connect duration is `0`, while `connReused`
stays `false`. This is the documented Go stdlib situation where either (a) an HTTP/2 connection is effectively reused yet
reported with `Reused == false`, or (b) the stdlib abandons a connection mid‑dial and hands over a just‑freed,
already‑established connection instead. In both cases there was genuinely no new dial *for this tracer* to time, so `0` is
the correct connect duration.

**Command and complete, unedited output** (**OBSERVED**, in‑package harness: a real `*tls.Conn` is dialed, then
`GotConn(Reused=false)` is invoked on the production `Tracer` with **no** prior `ConnectStart`/`ConnectDone`):

```bash
go test -run 'TestZZInvestigQ4NotReusedZeroConnect' -count=1 -v ./lib/netext/httpext/
```

```text
=== RUN   TestZZInvestigQ4NotReusedZeroConnect
===== Q4: Reused==false, connect hooks never fired (conn is *tls.Conn=true) =====
[after GetConn] getConn=1783487631597816973 connectStart=0 connectDone=0 tlsStart=0 tlsDone=0 gotConn=0 connReused=false
    -> connectStart==connectDone? true ; connectStart==gotConn? true ; connectDone==gotConn? true
[after GotConn(Reused=false)] getConn=1783487631597816973 connectStart=1783487631599966403 connectDone=1783487631599966403 tlsStart=1783487631599966403 tlsDone=1783487631599966403 gotConn=1783487631599966403 connReused=false
    -> connectStart==connectDone? true ; connectStart==gotConn? true ; connectDone==gotConn? true
    Trail: ConnReused=false Connecting=0s TLSHandshaking=0s Blocked=2.14943ms
--- PASS: TestZZInvestigQ4NotReusedZeroConnect (0.01s)
PASS
ok  	go.k6.io/k6/lib/netext/httpext	0.036s
```

Before `GotConn`, the connect stamps are `0`. After `GotConn(Reused=false)` with no prior connect hooks, all of
`connectStart`, `connectDone`, `tlsStart`, `tlsDone`, `gotConn` equal the *same* value `1783487631599966403` — exactly your
report: `connReused == false`, connect‑start/connect‑done == got‑connection, so `Connecting = 0s` and `TLSHandshaking = 0s`.
(`Blocked = 2.14943ms` here is just the `getConn → gotConn` gap created by the deliberate 2 ms sleep in the harness.)

> **Evidence labeling:** this is a **hook‑level** reproduction driving the *real* `Tracer.GotConn` / `Tracer.GetConn` /
> `Tracer.Done` methods (canonical code). The nondeterministic HTTP/2 stdlib race that spontaneously produces a false
> `Reused=false` in the wild cannot be forced on demand, so the triggering *condition* (a `Reused=false` `GotConn` with no
> preceding connect hooks) is injected while every timestamp is computed by the production tracer. The resulting state is
> identical to the in‑the‑wild case.

**Responsible code (cause → effect):** the `else` (`info.Reused == false`) branch of `GotConn` only fills the connect
stamps if they are still `0`:

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

at `lib/netext/httpext/tracer.go:287-291` (the enclosing `else` opens at `:278`). The comment directly above (`:279-286`)
documents the HTTP/2 false‑`Reused` case (*"a bug in the Go stdlib where an HTTP/2 connection can be reused but the
`httptrace.GotConnInfo` struct will contain a false `Reused` property"*), and the comment at `:265-269` documents the
abandon‑then‑reuse‑a‑freed‑connection case. Because start and done receive the same `now`, `Done()` computes
`trail.Connecting = 0` (`:340-341`) and `trail.TLSHandshaking = 0` (`:343-344`). If, instead, the real `ConnectStart`/
`ConnectDone` had fired first, they would already hold nonzero values and the `CompareAndSwap(…, 0, …)` would be a no‑op —
preserving the real dial span.

---

## Q5 — Why do `ConnectStart`/`ConnectDone` fire more than once per request, and does that double‑count? — **OBSERVED** (via the production `Tracer` hook methods)

> Your example, paraphrased: *"ConnectStart and ConnectDone sometimes get called multiple times for a single request, which
> looks like a double counting bug."*

**Direct answer: not double counting — it is deliberate de‑duplication.** With dual‑stack "Happy Eyeballs" (RFC 6555)
dialing, which Go's dialer enables by default, the dialer may start several parallel dials for one request (e.g. an IPv6 and
an IPv4 attempt), so `ConnectStart`/`ConnectDone` legitimately fire multiple times. k6 records only the **first**
`ConnectStart` via an atomic compare‑and‑swap, ignores **failed** dials in `ConnectDone`, and records only the first
**successful** `ConnectDone`. The result is a connect duration that reflects a single dial span, not a sum.

**Command and complete, unedited output** (**OBSERVED**, in‑package harness driving the real `Tracer.ConnectStart` /
`Tracer.ConnectDone` methods with the multi‑dial sequence the stdlib produces — IPv6 attempt fails, IPv4 attempt succeeds,
plus a spurious extra `ConnectDone`):

```bash
go test -run 'TestZZInvestigQ5DualStackDedup' -count=1 -v ./lib/netext/httpext/
```

```text
=== RUN   TestZZInvestigQ5DualStackDedup
===== Q5: repeated ConnectStart/ConnectDone (dual-stack Happy Eyeballs) =====
initial: connectStart=0 connectDone=0
after ConnectStart #1 (IPv6):  connectStart=1783487632707415843
after ConnectStart #2 (IPv4):  connectStart=1783487632707415843  (unchanged==first? true)
after ConnectDone #1 (IPv6,err!=nil): connectDone=0  (still 0? true)
after ConnectDone #2 (IPv4,err==nil): connectDone=1783487632711606025  (recorded now)
after ConnectDone #3 (extra):         connectDone=1783487632711606025  (unchanged? true)
Connecting (connectDone-connectStart) = 4.190182ms  -> single dial only, no double count
--- PASS: TestZZInvestigQ5DualStackDedup (0.00s)
PASS
ok  	go.k6.io/k6/lib/netext/httpext	0.010s
```

Reading the transitions: the second `ConnectStart` leaves `connectStart` **unchanged** (`unchanged==first? true`); the
**failed** IPv6 `ConnectDone` records **nothing** (`connectDone` stays `0`); the successful IPv4 `ConnectDone` records once;
and the third, spurious `ConnectDone` is **ignored** (`unchanged? true`). The final `Connecting` is a single
`connectDone − connectStart` span (here `4.190182ms`), never a sum of the two dials — so there is no double count.

**Responsible code (cause → effect):**

- `ConnectStart` records only the first dial:
  ```go
  // If using dual-stack dialing, it's possible to get this
  // multiple times, so the atomic compareAndSwap ensures
  // that only the first call's time is recorded
  atomic.CompareAndSwapInt64(&t.connectStart, 0, now())
  ```
  method at `lib/netext/httpext/tracer.go:197`, CAS at `:201`.
- `ConnectDone` mirrors it but is guarded by `err == nil`, so a failed Happy‑Eyeballs dial records nothing:
  ```go
  if err == nil {
      atomic.CompareAndSwapInt64(&t.connectDone, 0, now())
  }
  // if there is an error it either is happy eyeballs related and doesn't matter or it will be
  // returned by the http call
  ```
  method at `:213`, CAS at `:218`, error note at `:220-221`.
- The doc‑comments at `:191-196` (`ConnectStart`) and `:204-212` (`ConnectDone`) explicitly state that with
  `net.Dialer.DualStack` (IPv6 "Happy Eyeballs") support enabled — *the default* — these hooks "may be called multiple
  times." The atomic `CompareAndSwapInt64(…, 0, …)` idiom is what turns "called multiple times" into "recorded once."

---

## Ultimate (a) — Can the timing values be trusted for performance analysis? — **OBSERVED + code**

**Direct answer: YES — the values are trustworthy, provided their semantics are understood.** `connecting`,
`tls_handshaking`, and `blocked` deliberately reflect **only real new‑connection work**; they are `0`/near‑zero on reuse by
design (Q1, Q2), and the "impossible" cases you flagged (Q4, Q5) are correct handling of documented HTTP edge cases. The
one number to trust for **per‑request latency** is `http_req_duration`, and it **excludes** the connection phases entirely:

```go
trail.ConnDuration = trail.Connecting + trail.TLSHandshaking   // lib/netext/httpext/tracer.go:380
trail.Duration     = trail.Sending + trail.Waiting + trail.Receiving  // :381
```

`Duration` is even documented as *"Total request duration, excluding DNS lookup and connect time."* (`tracer.go:22-23`).
**External corroboration** — an upstream project record, neither runtime‑observed nor code‑inferred. grafana/k6 issue
**#2692**, *"HTTP metric including connection time"* (opened 2022‑09‑26; closed as *not planned*), documents this exact
semantics: `http_req_duration` measures only `http_req_sending` + `http_req_waiting` + `http_req_receiving`,
*"excluding the time spent during the connection"*. The issue *proposes* a separate `http_req_total_duration` that would
add the connection phases back in; because it was closed as *not planned*, `http_req_duration` still excludes them —
exactly what the code above computes — confirming the exclusion is intended design, not a measurement bug.

The three components it sums are derived from the last two hooks:

- **`WroteRequest`** (`tracer.go:299`, records `wroteRequest` via `atomic.StoreInt64` at `:301`, only when `info.Err == nil`)
  ends the *sending* phase. `Sending` is computed in the `switch` at `tracer.go:346-360` — from the TLS‑done time for TLS
  requests, else the connect‑done time, else (the HTTP/2 `Reused=false` corner) the `gotConn` time.
- **`GotFirstResponseByte`** (`tracer.go:310`, records `gotFirstResponseByte` via `atomic.CompareAndSwapInt64` at `:311`)
  ends the *waiting* phase (`Waiting`, `:361-372`) and starts the *receiving* phase (`Receiving = done − gotFirstResponseByte`,
  `:374`).

**Observed confirmation** (**OBSERVED**, from the in‑package harness above): the arithmetic closes exactly, and `Duration`
excludes `ConnDuration`:

```text
# req#0 (new connection):
Connecting=121.46µs  TLSHandshaking=2.122613ms  -> ConnDuration=2.244073ms
Sending=50.627µs + Waiting=242.681µs + Receiving=55.249µs = Duration=348.557µs     (ConnDuration NOT included)

# req#1 (reused connection):
Connecting=0s  TLSHandshaking=0s  -> ConnDuration=0s
Sending=13.258µs + Waiting=93.581µs + Receiving=38.468µs = Duration=145.307µs
```

(These numbers are the derived arithmetic from the complete `TestZZInvestigCanonical` output shown under Q1 above.) For
req#0, `ConnDuration` (2.244 ms) is more than 6× `Duration` (349 µs), yet the two are kept separate — so a new connection's
setup cost does **not** inflate the latency you read from `http_req_duration`. That is precisely why the values are
trustworthy.

**Practical guidance to include in your analysis:**

- Use **`http_req_duration`** for request latency. It is send + wait (server think time / TTFB) + receive, and nothing
  else.
- Read **`http_req_blocked`**, **`http_req_connecting`**, **`http_req_tls_handshaking`** as **new‑connection cost**. They
  are `0`/near‑zero on reused connections *by design*, so look at their **max / p95 / p99**, not their median — the median
  will (correctly) be `0` under keep‑alive.
- A rising `blocked`/`connecting`/`tls_handshaking` **max** over a run indicates connection churn or pool pressure (many new
  connections), not slow requests per se.
- On **Windows**, treat occasional per‑request zeros as the known timer‑resolution artifact (Q3): prefer aggregates and
  percentiles over individual per‑request timings, or measure fine‑grained per‑request timings on Linux/macOS.

---

## Ultimate (b) — Should anything be reported upstream? — **OBSERVED + INFERRED**

**Direct answer: of the five, only the Windows all‑zeros behavior (Q3) is genuinely upstream‑worthy — and it is already an
open Go platform issue, not a k6 defect.** Nothing should be filed against k6.

- **Q1, Q2, Q4, Q5 — nothing to file (OBSERVED).** These are the correct, intended behavior of the k6 tracer and of Go's
  `net/http/httptrace` contract:
  - Q1/Q2: reused connections skip the connect/TLS hooks, so `connecting`/`tls_handshaking` are `0` and `blocked` collapses
    to near‑zero — confirmed by every observation channel (see the appendix).
  - Q4: the `Reused=false`‑with‑zero‑connect case is explicitly anticipated and handled by the `else`‑branch CAS
    (`tracer.go:287-291`) for a documented Go stdlib HTTP/2 quirk.
  - Q5: multiple `ConnectStart`/`ConnectDone` calls are the *expected* consequence of default dual‑stack dialing, and k6
    de‑duplicates them with atomics (`tracer.go:201`, `:218`).
- **Q3 — already upstream in Go, no k6 change warranted (INFERRED).** k6 already documents and works around the Windows
  timer resolution in its own test suite (`lib/netext/httpext/tracer_test.go:33-40`), and the root cause is in the Go
  runtime's Windows clock (issues **#8687**, **#41087**, and related). Recommendation:
  - **Do not** file a k6 bug for Q3, and **do not** add a `QueryPerformanceCounter`/high‑resolution clock override to k6 —
    the fix belongs in Go, where it is tracked (notably #67066).
  - A k6 user on Windows should treat occasional per‑request zero timings as a **known platform measurement‑resolution
    limitation**: rely on aggregates/percentiles, or run fine‑grained per‑request timing measurements on Linux/macOS.

In short: **no k6 code change is justified by any of Q1–Q5.** The single upstream item (Q3) is a Go‑runtime clock
limitation that is already being tracked and, if anything, only warrants keeping an eye on the Go issues above.

---

## Observation channels appendix — all four channels agree

The same request timings were observed through four independent, canonical channels; they agree on every invariant
(connect/TLS exactly `0` on reuse; `blocked` large only on new connections).

### Channel 1 — JS `response.timings`

The per‑request `console.log` output shown under Q1/Q2. This is the object a k6 script reads via `r.timings`
(`ResponseTimings`, `lib/netext/httpext/response.go:34-43`).

### Channel 2 — `--http-debug=full` (proves the reused request still hits the wire)

This channel directly answers *"0 connect but I see network activity."* With `N=2`, k6 emits **two** request dumps and
**two** response dumps — the reused second request sends and receives over the wire exactly like the first; only its
`connecting`/`tls` are `0`.

```bash
URL="$TLS_H1_URL" N=2 /tmp/k6 run --http-debug=full --quiet --no-summary seq.js
```

Complete captured output (**OBSERVED**; the full k6 log line is shown verbatim — `time=…`/`level=`/`msg=` plus the
`group=/iter=/request_id=/scenario=/source=/vu=` fields — so the two distinct `request_id`s are visible):

```text
time="2026-07-08T05:15:21Z" level=info msg="Request:\nGET / HTTP/1.1\nHost: 127.0.0.1:42367\nUser-Agent: k6/0.55.0 (https://k6.io/)\nAccept-Encoding: gzip\n\n\n" group= iter=0 request_id=454ba041-6ca6-40e2-58ee-88d9930ed3a0 scenario=default source=http-debug vu=1
time="2026-07-08T05:15:21Z" level=info msg="Response:\nHTTP/1.1 200 OK\nContent-Length: 42\nContent-Type: text/plain\nDate: Wed, 08 Jul 2026 05:15:21 GMT\n\nhello from investig server proto=HTTP/1.1\n\n" group= iter=0 request_id=454ba041-6ca6-40e2-58ee-88d9930ed3a0 scenario=default source=http-debug vu=1
time="2026-07-08T05:15:21Z" level=info msg="req#0 proto=HTTP/1.1 status=200 remote=127.0.0.1:42367 blocked=2.399 connecting=0.145 tls=2.190 sending=0.032 waiting=30.408 receiving=0.153 duration=30.593" source=console
time="2026-07-08T05:15:21Z" level=info msg="Request:\nGET / HTTP/1.1\nHost: 127.0.0.1:42367\nUser-Agent: k6/0.55.0 (https://k6.io/)\nAccept-Encoding: gzip\n\n\n" group= iter=0 request_id=46e04c83-d387-46da-611d-284e3b27be01 scenario=default source=http-debug vu=1
time="2026-07-08T05:15:21Z" level=info msg="Response:\nHTTP/1.1 200 OK\nContent-Length: 42\nContent-Type: text/plain\nDate: Wed, 08 Jul 2026 05:15:21 GMT\n\nhello from investig server proto=HTTP/1.1\n\n" group= iter=0 request_id=46e04c83-d387-46da-611d-284e3b27be01 scenario=default source=http-debug vu=1
time="2026-07-08T05:15:21Z" level=info msg="req#1 proto=HTTP/1.1 status=200 remote=127.0.0.1:42367 blocked=0.004 connecting=0.000 tls=0.000 sending=0.016 waiting=30.358 receiving=0.079 duration=30.453" source=console
```

Two full request/response exchanges over the wire (distinct `request_id`s `454ba041…` and `46e04c83…`), yet req#1 (reused)
reports `connecting=0.000 tls=0.000`. The wire‑dump transport is `lib/netext/httpext/httpdebug_transport.go` (its
`RoundTrip` dumps request and response via `httputil.DumpRequestOut`).

### Channel 3 — `--out json` (machine‑readable per‑request samples)

```bash
URL="$TLS_H1_URL" N=3 /tmp/k6 run --out json=/tmp/investig/out.json --quiet --no-summary seq.js
# then extract the per-request Point values for each metric from out.json
```

Raw per‑request `Point` samples for the three metrics (**OBSERVED**, the complete unedited JSON lines filtered from
`out.json` with `grep '"type":"Point"'`; grouped by request — req#0 new, req#1/req#2 reused):

```text
{"metric":"http_req_blocked","type":"Point","data":{"time":"2026-07-08T05:15:32.863235797Z","value":2.324903,"tags":{"expected_response":"true","group":"","method":"GET","name":"https://127.0.0.1:42367","proto":"HTTP/1.1","scenario":"default","status":"200","tls_version":"tls1.3","url":"https://127.0.0.1:42367"}}}
{"metric":"http_req_connecting","type":"Point","data":{"time":"2026-07-08T05:15:32.863235797Z","value":0.134932,"tags":{"expected_response":"true","group":"","method":"GET","name":"https://127.0.0.1:42367","proto":"HTTP/1.1","scenario":"default","status":"200","tls_version":"tls1.3","url":"https://127.0.0.1:42367"}}}
{"metric":"http_req_tls_handshaking","type":"Point","data":{"time":"2026-07-08T05:15:32.863235797Z","value":2.117265,"tags":{"expected_response":"true","group":"","method":"GET","name":"https://127.0.0.1:42367","proto":"HTTP/1.1","scenario":"default","status":"200","tls_version":"tls1.3","url":"https://127.0.0.1:42367"}}}
{"metric":"http_req_blocked","type":"Point","data":{"time":"2026-07-08T05:15:32.893906257Z","value":0.002121,"tags":{"expected_response":"true","group":"","method":"GET","name":"https://127.0.0.1:42367","proto":"HTTP/1.1","scenario":"default","status":"200","tls_version":"tls1.3","url":"https://127.0.0.1:42367"}}}
{"metric":"http_req_connecting","type":"Point","data":{"time":"2026-07-08T05:15:32.893906257Z","value":0,"tags":{"expected_response":"true","group":"","method":"GET","name":"https://127.0.0.1:42367","proto":"HTTP/1.1","scenario":"default","status":"200","tls_version":"tls1.3","url":"https://127.0.0.1:42367"}}}
{"metric":"http_req_tls_handshaking","type":"Point","data":{"time":"2026-07-08T05:15:32.893906257Z","value":0,"tags":{"expected_response":"true","group":"","method":"GET","name":"https://127.0.0.1:42367","proto":"HTTP/1.1","scenario":"default","status":"200","tls_version":"tls1.3","url":"https://127.0.0.1:42367"}}}
{"metric":"http_req_blocked","type":"Point","data":{"time":"2026-07-08T05:15:32.924480055Z","value":0.001842,"tags":{"expected_response":"true","group":"","method":"GET","name":"https://127.0.0.1:42367","proto":"HTTP/1.1","scenario":"default","status":"200","tls_version":"tls1.3","url":"https://127.0.0.1:42367"}}}
{"metric":"http_req_connecting","type":"Point","data":{"time":"2026-07-08T05:15:32.924480055Z","value":0,"tags":{"expected_response":"true","group":"","method":"GET","name":"https://127.0.0.1:42367","proto":"HTTP/1.1","scenario":"default","status":"200","tls_version":"tls1.3","url":"https://127.0.0.1:42367"}}}
{"metric":"http_req_tls_handshaking","type":"Point","data":{"time":"2026-07-08T05:15:32.924480055Z","value":0,"tags":{"expected_response":"true","group":"","method":"GET","name":"https://127.0.0.1:42367","proto":"HTTP/1.1","scenario":"default","status":"200","tls_version":"tls1.3","url":"https://127.0.0.1:42367"}}}
```

The per‑request values extracted from the `"value"` field of those `Point` records (first value = new connection, rest =
reused):

```text
http_req_connecting      = 0.134932, 0, 0
http_req_tls_handshaking = 2.117265, 0, 0
http_req_blocked         = 2.324903, 0.002121, 0.001842
```

Exactly the Q1/Q2 pattern in machine‑readable form: the first request carries the connect/TLS cost; the reused requests are
`0`, with a tiny residual `blocked`.

### Channel 4 — in‑package `Tracer` Go harness

The internal nanosecond timestamps and derived `Trail`/metric values shown under Q1 (canonical), Q4, and Q5. This is the
production `httpext.Tracer` exercised through the real `transport.RoundTrip` and the real hook methods, mirroring
`lib/netext/httpext/tracer_test.go`.

---

## Methods & reproducibility

All harnesses ran through the canonical path; all temporary artifacts lived in `/tmp` (outside the repository) except the
in‑package Go harness, which was written into the package, run, and then deleted. The working tree was verified clean
(`git status --porcelain` shows only the added `blitzy/documentation/k6_ddc3b0b1d23c.md`).

### Build

```bash
export PATH=/usr/local/go/bin:$PATH
export GOFLAGS=-mod=vendor GOTOOLCHAIN=local GOPATH=/tmp/gopath GOCACHE=/tmp/gocache
cd <repo>                       # the k6_ddc3b0b1d23c checkout (commit ddc3b0b1d23c…)
go build -o /tmp/k6 .
/tmp/k6 version                 # -> k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)
```

### Local test server — `/tmp/investig/server.go` (stdlib only, outside the k6 module)

A tiny `httptest`‑based server exposing three endpoints — TLS+HTTP/2, TLS+HTTP/1.1‑only, and plaintext — with a 30 ms
handler sleep so `waiting` is measurably ~30 ms:

```go
package main

import (
	"fmt"; "io"; "net"; "net/http"; "net/http/httptest"; "os"; "time"
)
func handler(w http.ResponseWriter, r *http.Request) {
	time.Sleep(30 * time.Millisecond)
	w.Header().Set("Content-Type", "text/plain")
	io.WriteString(w, "hello from investig server proto="+r.Proto+"\n")
}
func main() {
	mux := http.NewServeMux(); mux.HandleFunc("/", handler)
	tlsSrv := httptest.NewUnstartedServer(mux); tlsSrv.EnableHTTP2 = true; tlsSrv.StartTLS()
	h1Srv := httptest.NewTLSServer(mux)
	plain := httptest.NewServer(mux)
	fmt.Printf("TLS_H2_URL=%s\nTLS_H1_URL=%s\nPLAIN_URL=%s\n", tlsSrv.URL, h1Srv.URL, plain.URL)
	f, _ := os.Create("/tmp/investig/urls.env")
	fmt.Fprintf(f, "TLS_H2_URL=%s\nTLS_H1_URL=%s\nPLAIN_URL=%s\n", tlsSrv.URL, h1Srv.URL, plain.URL); f.Close()
	_ = net.IPv4zero; select {}
}
```

```bash
cd /tmp/investig && go build -o /tmp/investig/server server.go \
  && nohup /tmp/investig/server > /tmp/investig/server.out 2>&1 & sleep 2 && cat /tmp/investig/urls.env
```

### k6 scripts (outside the repo)

`/tmp/investig/seq.js` — sequential same‑endpoint GETs, printing per‑request `response.timings`:

```javascript
import http from 'k6/http';
export const options = { vus: 1, iterations: 1, insecureSkipTLSVerify: true };
const URL = __ENV.URL;
const N = parseInt(__ENV.N || '4');
export default function () {
  for (let i = 0; i < N; i++) {
    const r = http.get(URL);
    const t = r.timings;
    console.log(
      `req#${i} proto=${r.proto} status=${r.status} remote=${r.remote_ip}:${r.remote_port}` +
      ` blocked=${t.blocked.toFixed(3)} connecting=${t.connecting.toFixed(3)} tls=${t.tls_handshaking.toFixed(3)}` +
      ` sending=${t.sending.toFixed(3)} waiting=${t.waiting.toFixed(3)} receiving=${t.receiving.toFixed(3)} duration=${t.duration.toFixed(3)}`);
  }
}
```

`/tmp/investig/scale.js` — one GET per iteration (distribution comes from the run summary):

```javascript
import http from 'k6/http';
export const options = { insecureSkipTLSVerify: true };
const URL = __ENV.URL;
export default function () { http.get(URL); }
```

`/tmp/investig/remote.js` — remote WAN magnitude for Q2:

```javascript
import http from 'k6/http';
export const options = {};
export default function () { http.get('https://quickpizza.grafana.com/'); }
```

### In‑package `Tracer` harness — `lib/netext/httpext/zz_investig_test.go` (temporary; deleted after use)

Written into the package (so it can read the unexported timestamp fields), it exercises the **production** `Tracer` through
the real `transport.RoundTrip` and the real hook methods (`GetConn`, `GotConn`, `ConnectStart`, `ConnectDone`, `Done`). It
was run with `go test -run 'TestZZInvestig' -count=1 -v ./lib/netext/httpext/` (result `ok`), then removed:

```bash
rm -f lib/netext/httpext/zz_investig_test.go
git status --porcelain   # -> only blitzy/documentation/k6_ddc3b0b1d23c.md
```

The harness contains three tests: `TestZZInvestigCanonical` (a fresh `Tracer` per request against an `httptest` TLS
`httpbin` server with the real k6 dialer/resolver and the registered built‑in metrics — the Q1‑internal + Ultimate‑(a)
evidence), `TestZZInvestigQ4NotReusedZeroConnect` (Q4), and `TestZZInvestigQ5DualStackDedup` (Q5). Its structure mirrors the
project's own `lib/netext/httpext/tracer_test.go`.

### Cleanup

The compiled binary (`/tmp/k6`), the local server, the JS scripts, and the Go harness all reside in `/tmp` or were deleted
from the package; the repository's only net change is this document. `git status --porcelain` confirms it.

Captured verification (post‑commit steady state — the exact bytes emitted):

```text
$ git status --porcelain
                                            # (no output — clean working tree)

$ git diff --name-status ddc3b0b1d23c..HEAD
A	blitzy/documentation/k6_ddc3b0b1d23c.md
```

---

### Coverage — every named hook, by `file:line`

| Hook / function | `file:line` | Role |
|---|---|---|
| `now` | `tracer.go:175-177` | `time.Now().UnixNano()` — the single, cross‑platform time source (Q3) |
| `GetConn` | `tracer.go:187` | stamps `getConn`; called even for an idle cached connection (Q2 `blocked`) |
| `ConnectStart` | `tracer.go:197` | CAS `connectStart` (`:201`); records only first dial (Q5) |
| `ConnectDone` | `tracer.go:213` | CAS `connectDone` (`:218`) only if `err == nil`; ignores failed dials (Q5) |
| `TLSHandshakeStart` | `tracer.go:230` | CAS `tlsHandshakeStart` (`:231`); skipped on reuse (Q1) |
| `TLSHandshakeDone` | `tracer.go:242` | CAS `tlsHandshakeDone` (`:244`); skipped on reuse (Q1) |
| `GotConn` | `tracer.go:256` | sets `connReused` (`:262`); reuse branch swaps stamps to `now` (`:271-277`); else‑branch CAS (`:287-291`) (Q1, Q4) |
| `WroteRequest` | `tracer.go:299` | `StoreInt64 wroteRequest` (`:301`); ends *sending* (Ultimate a) |
| `GotFirstResponseByte` | `tracer.go:310` | CAS `gotFirstResponseByte` (`:311`); ends *waiting*, starts *receiving* (Ultimate a) |
| `Done` | `tracer.go:315` | derives every `Trail` duration; `Blocked` `:323-325`, `Connecting` `:340-341`, `TLSHandshaking` `:343-344`, `ConnDuration` `:380`, `Duration` `:381` |
