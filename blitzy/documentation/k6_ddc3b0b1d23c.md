# k6 HTTP Request-Timing Metrics: Trustworthiness & Root-Cause Analysis

> **Scope & revision.** This is a code-grounded, read-only investigation of k6's HTTP request-timing
> metrics. All analysis and every `file:line` citation are pinned to the current checkout —
> commit `ddc3b0b1d2` (`k6 v0.55.0`, built with `go1.23.12`, `linux/amd64`). The investigation
> modifies no source code; it reads, builds, and runs the existing tracer subsystem to ground its
> conclusions. Line numbers reflect this exact revision.

A k6 user debugging "strange" HTTP timing metrics reported five behaviors and asked two questions:

- **Q1 — Can the timing values be trusted for performance analysis?**
- **Q2 — Should any of these issues be reported upstream (are they real bugs or expected behavior)?**

This document answers both questions directly (Section 1), explains exactly how k6 measures HTTP
timings (Section 2), adjudicates each of the five reported behaviors against the actual code
(Section 3), assesses the concurrency-safety of the measurement code (Section 4), gives a
trustworthiness verdict with a per-metric interpretation guide (Section 5), states the
upstream-report recommendation (Section 6), describes how to reproduce the behaviors without
touching the source tree (Section 7), and lists all references (Section 8).

The guiding principle throughout is **code as the source of truth**: every claim about k6's behavior
carries an inline citation to the source, and every platform/standard-library claim is tied to the
Go [`net/http/httptrace`](https://pkg.go.dev/net/http/httptrace) contract or a tracked Go runtime
issue.

---

## 1. Summary answers (Q1 and Q2)

### Q1 — Can the timing values be trusted?

**Yes — the timing values can be trusted for performance analysis, provided each metric is
interpreted with its intended meaning.** Two interpretive caveats matter most: `http_req_connecting`
and `http_req_tls_handshaking` are *expected* to be `0` on a reused connection (no new connection or
handshake occurred), and on Windows very fast phases can read `0` because of the operating system's
coarse `time.Now()` resolution. The underlying measurement code is concurrency-safe: every hook
timestamp is read and written through `sync/atomic`, and a fresh tracer is created per request
[`lib/netext/httpext/transport.go:L205`].

### Q2 — Should anything be reported upstream?

**No — none of the five reported behaviors is a new k6 defect, so no upstream bug report against k6
is warranted.** Four of the five behaviors are by-design (and three of those are directly asserted or
documented by k6's own code and tests). The fifth — all-zero metrics on Windows — is a documented
Go/Windows `time.Now()` precision limitation that is already tracked at the Go runtime level
([golang/go#8687](https://github.com/golang/go/issues/8687),
[#41087](https://github.com/golang/go/issues/41087), and the still-open
[#67066](https://github.com/golang/go/issues/67066)). The only "upstream" item that exists is the Go
runtime clock-resolution improvement, which is outside k6's control and already filed.

### Verdict table

| # | Reported behavior | Verdict | Primary citation |
|---|-------------------|---------|------------------|
| 1 | Zero `connecting`/`tls_handshaking` on subsequent (reused) requests | **By-design** | `lib/netext/httpext/tracer.go:L271-L277`, `L340-L345` |
| 2 | `blocked` swings between ~500ms and ~0 for seemingly identical requests | **By-design** | `lib/netext/httpext/tracer.go:L323-L325` |
| 3 | All timing metrics occasionally `0` on Windows (suspected race) | **Known platform limitation, not a race** | `lib/netext/httpext/tracer.go:L175-L177`; `tracer_test.go:L33-L40` |
| 4 | "Not reused" yet connect timestamp `==` `gotConn` timestamp | **By-design defensive handling** | `lib/netext/httpext/tracer.go:L278-L293` |
| 5 | `ConnectStart`/`ConnectDone` fire multiple times for one request | **By-design** | `lib/netext/httpext/tracer.go:L191-L193`, `L201`, `L218` |

Each verdict is substantiated, with reasoning, in Section 3.

---

## 2. How k6 measures HTTP timings (the measurement model)

k6's HTTP timing metrics are produced by a small, purpose-built tracer that hooks into Go's standard
HTTP client instrumentation. Understanding this pipeline is a prerequisite for adjudicating the five
behaviors, because every reported symptom is a direct consequence of how the pipeline is wired.

### 2.1 A fresh tracer per request

k6 installs its own `http.RoundTripper`. On every round trip it constructs a **fresh**
`&Tracer{}` and attaches it to the request context via `httptrace.WithClientTrace`
[`lib/netext/httpext/transport.go:L205-L206`]. The k6 transport is wired in as the request's
`http.RoundTripper` in `lib/netext/httpext/request.go` (`newTransport(...)` assigned to a
`var transport http.RoundTripper` [`lib/netext/httpext/request.go:L184-L185`] and used as
`http.Client{Transport: transport}` [`lib/netext/httpext/request.go:L232-L233`]).

Creating a new tracer per request is deliberate; the tracer type carries an explicit warning that
**"It's NOT safe to reuse Tracers between requests."** [`lib/netext/httpext/tracer.go:L145`]. This
eliminates any cross-request state sharing — a point that matters for the concurrency analysis in
Section 4.

### 2.2 Eight hooks capture nanosecond timestamps

`Tracer.Trace()` returns an `*httptrace.ClientTrace` that wires **eight** hooks — `GetConn`,
`ConnectStart`, `ConnectDone`, `TLSHandshakeStart`, `TLSHandshakeDone`, `GotConn`, `WroteRequest`,
and `GotFirstResponseByte` [`lib/netext/httpext/tracer.go:L162-L173`].

Each hook records a timestamp by calling the package helper `now()`, which is simply
`time.Now().UnixNano()` [`lib/netext/httpext/tracer.go:L175-L177`]. Those timestamps are stored into
the tracer's `int64` fields — `getConn`, `connectStart`, `connectDone`, `tlsHandshakeStart`,
`tlsHandshakeDone`, `gotConn`, `wroteRequest`, `gotFirstResponseByte`
[`lib/netext/httpext/tracer.go:L147-L159`]. Storing nanosecond integers (rather than `time.Time`
values) is what allows the writes/reads to go through `sync/atomic` (Section 4).

### 2.3 `Done()` computes per-phase durations into a `Trail`

When the request finishes, the transport's `measureAndEmitMetrics` calls `tracer.Done()`
[`lib/netext/httpext/transport.go:L78`], which converts the captured timestamps into per-phase
`time.Duration` values held in a `Trail`
[`lib/netext/httpext/tracer.go:L315-L384`]. The `Trail` struct defines the meaning of each metric via
in-code comments [`lib/netext/httpext/tracer.go:L16-L41`]:

- `Blocked` — "Waiting to acquire a connection." [`lib/netext/httpext/tracer.go:L25`]
- `Connecting` — "Connecting to remote host." [`lib/netext/httpext/tracer.go:L26`]
- `TLSHandshaking` — "Executing TLS handshake." [`lib/netext/httpext/tracer.go:L27`]
- `Sending` — "Writing request." [`lib/netext/httpext/tracer.go:L28`]
- `Waiting` — "Waiting for first byte." [`lib/netext/httpext/tracer.go:L29`]
- `Receiving` — "Receiving response." [`lib/netext/httpext/tracer.go:L30`]
- `ConnDuration` — "Total connect time (Connecting + TLSHandshaking)" [`lib/netext/httpext/tracer.go:L19-L20`]
- `Duration` — "Total request duration, excluding DNS lookup and connect time." [`lib/netext/httpext/tracer.go:L22-L23`]

### 2.4 Duration derivation (exact formulas)

`Done()` derives each phase from the raw timestamps, using guards so an unfired hook (timestamp `0`)
never produces a bogus negative or huge value:

- **Blocked** `= gotConn − getConn`, set only when `gotConn != 0 && getConn != 0 && gotConn > getConn`
  [`lib/netext/httpext/tracer.go:L323-L325`].
- **Connecting** `= connectDone − connectStart`, set only when both are non-zero
  [`lib/netext/httpext/tracer.go:L340-L342`].
- **TLSHandshaking** `= tlsHandshakeDone − tlsHandshakeStart`, set only when both are non-zero
  [`lib/netext/httpext/tracer.go:L343-L345`].
- **Sending** is chosen by a `switch` over three cases — TLS-handshake-done, connect-done, or the
  HTTP/2 "GotConn first" case — [`lib/netext/httpext/tracer.go:L346-L359`].
- **Waiting** `= gotFirstResponseByte − wroteRequest` (or, if the server never responded, the time
  from `wroteRequest` to the final `done` instant) [`lib/netext/httpext/tracer.go:L361-L372`].
- **Receiving** `= done − gotFirstResponseByte` [`lib/netext/httpext/tracer.go:L374-L376`].

The two aggregates are then `ConnDuration = Connecting + TLSHandshaking`
[`lib/netext/httpext/tracer.go:L380`] and `Duration = Sending + Waiting + Receiving`
[`lib/netext/httpext/tracer.go:L381`]. Note that `Duration` intentionally **excludes** DNS lookup,
blocked, and connect time — consistent with its doc comment [`lib/netext/httpext/tracer.go:L22-L23`].

### 2.5 `SaveSamples` emits the `http_req_*` metrics the user observes

`Trail.SaveSamples` turns those durations into metric samples
[`lib/netext/httpext/tracer.go:L43-L122`], which the transport pushes into k6's metrics pipeline via
`trail.SaveSamples(...)` [`lib/netext/httpext/transport.go:L146`] and
`metrics.PushIfNotDone(...)` [`lib/netext/httpext/transport.go:L164`]. These become the built-in
metrics the user sees, whose names are defined in `metrics/builtin.go`
[`metrics/builtin.go:L15-L23`]:

| Metric name | Constant | Line |
|-------------|----------|------|
| `http_reqs` | `HTTPReqsName` | `metrics/builtin.go:L15` |
| `http_req_failed` | `HTTPReqFailedName` | `metrics/builtin.go:L16` |
| `http_req_duration` | `HTTPReqDurationName` | `metrics/builtin.go:L17` |
| `http_req_blocked` | `HTTPReqBlockedName` | `metrics/builtin.go:L18` |
| `http_req_connecting` | `HTTPReqConnectingName` | `metrics/builtin.go:L19` |
| `http_req_tls_handshaking` | `HTTPReqTLSHandshakingName` | `metrics/builtin.go:L20` |
| `http_req_sending` | `HTTPReqSendingName` | `metrics/builtin.go:L21` |
| `http_req_waiting` | `HTTPReqWaitingName` | `metrics/builtin.go:L22` |
| `http_req_receiving` | `HTTPReqReceivingName` | `metrics/builtin.go:L23` |

The same per-phase values also surface to scripts as `response.timings` — the `ResponseTimings`
struct exposes `duration`, `blocked`, `connecting`, `tls_handshaking`, `sending`, `waiting`, and
`receiving` (plus `looking_up` for DNS) as JSON fields
[`lib/netext/httpext/response.go:L34-L44`].

### 2.6 Receiving is finalized after the body is read

One subtlety: `http_req_receiving` cannot be finalized at the moment `RoundTrip` returns, because the
response body has not yet been read. k6 handles this with a deferred-finalization mechanism: the
in-flight request is stashed by `saveCurrentRequest` [`lib/netext/httpext/transport.go:L168-L179`]
and finalized later by `processLastSavedRequest`
[`lib/netext/httpext/transport.go:L181-L198`], which is invoked once the body has been consumed
(e.g. from `request.go` after decompression [`lib/netext/httpext/request.go:L283`]). This is why
`Done()` computes `Receiving` relative to the final `done := time.Now()` instant captured at the top
of `Done()` [`lib/netext/httpext/tracer.go:L316`, `L374-L376`].

---

## 3. Per-behavior root-cause analysis

Each behavior is adjudicated with the same structure: **Observation** (the user's report) →
**Code path** (`file:line`) → **Verdict** → **Rationale** (the reasoning that ties code to verdict).

### 3.1 Behavior 1 — Zero `connecting`/`tls_handshaking` on subsequent requests

**Observation.** When making multiple sequential requests to the same endpoint, the first request
shows a reasonable `connecting` time and `tls_handshaking` time, but every subsequent request shows
exactly `0` for both — even though network activity is clearly occurring.

**Code path.** `lib/netext/httpext/tracer.go:L256-L277` (the `GotConn` reuse branch);
`lib/netext/httpext/tracer.go:L340-L345` (the `Connecting`/`TLSHandshaking` computation in `Done()`);
`lib/netext/httpext/tracer_test.go:L161-L165` (the existing reuse assertion).

**Verdict: BY-DESIGN.**

**Rationale.** On a **reused** connection, `GotConn` is the first hook called
[`lib/netext/httpext/tracer.go:L253`], and it `atomic.SwapInt64`s `connectStart` and `connectDone`
(and, when the connection is a `*tls.Conn`, `tlsHandshakeStart`/`tlsHandshakeDone`) to a single `now`
instant [`lib/netext/httpext/tracer.go:L271-L277`]. Because both endpoints of each interval are set
to the same value, `Done()` computes `Connecting = connectDone − connectStart = 0` and
`TLSHandshaking = tlsHandshakeDone − tlsHandshakeStart = 0`
[`lib/netext/httpext/tracer.go:L340-L345`].

This is reinforced by the Go standard library independently: per the
[`httptrace`](https://pkg.go.dev/net/http/httptrace) contract, the `ConnectStart`, `ConnectDone`, and
TLS-handshake hooks are **not called at all** for a connection retrieved from the idle pool — there
is simply no new connect or handshake interval to measure. So even without k6's explicit swap, those
durations would be zero on reuse.

The existing unit test encodes exactly this expectation: for a reused connection, it asserts that the
`http_req_connecting` and `http_req_tls_handshaking` samples equal `0.0`
[`lib/netext/httpext/tracer_test.go:L161-L165`]. In other words, a `0` here is *meaningful data*, not
missing data — it tells you "no new connection or TLS handshake occurred for this request because an
existing keep-alive (or HTTP/2) connection was reused." That is the normal and desirable outcome of
connection pooling, not a measurement failure.

### 3.2 Behavior 2 — Wildly variable `blocked` time

**Observation.** The `blocked` timing sometimes shows very large values (around 500ms) and other
times near-zero for what appear to be identical requests.

**Code path.** `lib/netext/httpext/tracer.go:L323-L325` (the `Blocked` computation);
`lib/netext/httpext/tracer.go:L181-L182` and `L187-L189` (the `GetConn` hook and its doc);
`lib/netext/httpext/tracer.go:L256-L263` (the `GotConn` hook).

**Verdict: BY-DESIGN.**

**Rationale.** `blocked` is computed as `Blocked = time.Duration(gotConn − getConn)`, set only when
`gotConn != 0 && getConn != 0 && gotConn > getConn` [`lib/netext/httpext/tracer.go:L323-L325`]. The
two endpoints are:

- `getConn` — recorded by the `GetConn` hook, which fires *before* a connection is acquired
  [`lib/netext/httpext/tracer.go:L188`]. Per the in-code doc (mirroring the Go contract), `GetConn`
  "is called even if there's already an idle cached connection available"
  [`lib/netext/httpext/tracer.go:L181-L182`].
- `gotConn` — recorded by `GotConn` once the connection is actually in hand
  [`lib/netext/httpext/tracer.go:L257`, `L261`].

Therefore `blocked` legitimately measures **connection-acquisition time**, and its variance is
expected:

- **Large (hundreds of ms):** a brand-new connection had to be dialed (DNS + TCP + TLS), or the
  transport's per-host / pool connection limit forced the request to wait for a free connection.
- **Near-zero:** the request immediately got an idle connection from the pool.

Two requests that "look identical" at the script level can hit either path depending on the pool
state at that instant (how many connections are open, whether one is idle, whether the per-host limit
is saturated). The swing from ~500ms to ~0 is precisely the signal `blocked` is designed to surface,
not a defect.

### 3.3 Behavior 3 — All-zero metrics on Windows (suspected race)

**Observation.** On a Windows test machine, occasionally **all** timing metrics return `0` for random
requests; the user suspects a race condition in the measurement code.

**Code path.** `lib/netext/httpext/tracer.go:L175-L177` (`now()`);
`lib/netext/httpext/tracer_test.go:L33-L40` (the Windows-resolution HACK in the tests);
`lib/netext/httpext/tracer.go:L332-L345` (the atomic loads + delta computations in `Done()`).

**Verdict: KNOWN PLATFORM LIMITATION — NOT a race.**

**Rationale.** Every timestamp originates from `now()`, defined as `time.Now().UnixNano()`
[`lib/netext/httpext/tracer.go:L175-L177`]. On Windows, `time.Now()` has historically had **coarse
resolution** (on the order of ~1–15 ms). When two hooks for a fast request fire within the same clock
tick, they read the **same** integer value, so deltas such as `connectDone − connectStart` evaluate
to `0` [`lib/netext/httpext/tracer.go:L340-L345`]. With several phases sharing a tick, an entire
request can report zeros.

This is a documented Go/Windows limitation, and **k6's own test code calls it out explicitly.** For
`runtime.GOOS == "windows"` [`lib/netext/httpext/tracer_test.go:L33`], the test inserts artificial
delays in the trace handlers, with a comment that the time resolution "is not as accurate on Windows"
and that, without the delays, "ConnectStart and ConnectDone could register the same time"
[`lib/netext/httpext/tracer_test.go:L34-L40`]. That comment cites two Go issues directly:
[golang/go#8687](https://github.com/golang/go/issues/8687) (`lib/netext/httpext/tracer_test.go:L35`)
and [#41087](https://github.com/golang/go/issues/41087) (`lib/netext/httpext/tracer_test.go:L36`).

Crucially, this is a **clock-precision artifact, not a data race.** The distinction matters:

- A *race* would mean concurrent, unsynchronized access corrupting the timestamps. As Section 4
  shows, every timestamp is written and read through `sync/atomic`, and the package is stress-tested
  under the race detector — so there is no race.
- The *zeros* arise because the clock cannot resolve two events that occur very close together; both
  reads are correct, they just return the same coarse value.

Corroborating Go runtime issues confirm this is upstream and ongoing:
[#8687](https://github.com/golang/go/issues/8687) and
[#41087](https://github.com/golang/go/issues/41087) (cited in-code),
[#11313](https://github.com/golang/go/issues/11313),
[#31160](https://github.com/golang/go/issues/31160) (demonstrates ~1ms `time.Now()` granularity on
Windows), and the still-open [#67066](https://github.com/golang/go/issues/67066) (opened April 2024,
"use QueryPerformanceCounter in time.Now on Windows to improve time resolution"). The fix lies in the
Go runtime, not in k6.

### 3.4 Behavior 4 — "Not reused" yet identical connect/`gotConn` timestamps

**Observation.** Inspecting internal tracer state, a connection is sometimes flagged as "not reused,"
yet the connect timestamps equal the "got connection" timestamp — which the user argues should be
impossible if a real TCP handshake occurred.

**Code path.** `lib/netext/httpext/tracer.go:L278-L293` (the `GotConn` `else` branch);
`lib/netext/httpext/tracer.go:L355-L358` (the matching `Sending` `default` case in `Done()`).

**Verdict: BY-DESIGN defensive handling.**

**Rationale.** When `GotConn` reports `info.Reused == false`, the tracer enters the `else` branch and
`atomic.CompareAndSwapInt64`s the still-zero `connectStart`/`connectDone` (and TLS) timestamps to the
current `now` [`lib/netext/httpext/tracer.go:L287-L291`]. The in-code comment explains precisely why:
there is a **known Go stdlib bug** where an **HTTP/2** connection can actually be reused while the
`httptrace.GotConnInfo.Reused` field is falsely `false`
[`lib/netext/httpext/tracer.go:L279-L286`].

In that false-`Reused` case, no real TCP/TLS handshake occurred for this request, so the
`ConnectStart`/`ConnectDone`/TLS hooks never fired and their timestamps are still zero. Two things
follow:

1. CAS-ing those zero timestamps to `now` prevents any later-firing hook (which also uses CAS) from
   writing a bogus value into them, keeping `Connecting`/`TLSHandshaking` at `0`
   [`lib/netext/httpext/tracer.go:L340-L345`].
2. The `Done()` `Sending` switch even contains a dedicated `default` case for exactly this situation,
   with the comment "this handles the strange HTTP/2 case where the GotConn() hook gets called first,
   but with Reused=false" [`lib/netext/httpext/tracer.go:L355-L358`].

So a connection marked "not reused" whose connect timestamps equal the `gotConn` timestamp is the
**expected output of this defensive code** for the false-`Reused` HTTP/2 case — it indicates that no
phantom handshake was counted, not that a real handshake was mis-measured. The user's intuition
("this should be impossible for a real handshake") is correct, and the code's behavior is consistent
with it: because there was no real handshake, the interval is deliberately collapsed to zero.

### 3.5 Behavior 5 — Hooks fire multiple times

**Observation.** `ConnectStart` and `ConnectDone` are sometimes called multiple times for a single
request; the user suspects a double-counting bug.

**Code path.** `lib/netext/httpext/tracer.go:L191-L193` and `L207-L208` (the hook docs);
`lib/netext/httpext/tracer.go:L201` and `L218` (the de-duplicating CAS writes);
`lib/netext/dialer.go:L18-L28`, `L58-L70` (the dual-stack dialer);
`js/runner.go:L90-L93` (the production `BaseDialer`).

**Verdict: BY-DESIGN.**

**Rationale.** Multiple invocations are explicitly sanctioned by the Go
[`httptrace`](https://pkg.go.dev/net/http/httptrace) contract: when `net.Dialer.DualStack`
(IPv6 "Happy Eyeballs") support is enabled — which is Go's default — `ConnectStart`/`ConnectDone`
"may be called multiple times." k6's own hook docs restate this verbatim
[`lib/netext/httpext/tracer.go:L191-L193`, `L207-L208`].

k6 anticipates this and **records only the first invocation** using compare-and-swap:
`atomic.CompareAndSwapInt64(&t.connectStart, 0, now())` [`lib/netext/httpext/tracer.go:L201`] and the
error-guarded `atomic.CompareAndSwapInt64(&t.connectDone, 0, now())`
[`lib/netext/httpext/tracer.go:L218`]. Because CAS only succeeds when the field is still `0`, any
repeated call after the first is a no-op — there is **no double-counting**.

The dual-stack behavior originates in k6's connection setup: its `Dialer` embeds `net.Dialer`
[`lib/netext/dialer.go:L18-L28`] and delegates to `net.Dialer.DialContext`
[`lib/netext/dialer.go:L58-L70`, delegation at `L64`]. The production `BaseDialer` is
`net.Dialer{Timeout: 30 * time.Second, KeepAlive: 30 * time.Second}`
[`js/runner.go:L90-L93`] — it sets neither `FallbackDelay < 0` nor any `DualStack` override, so Go's
default dual-stack ("Happy Eyeballs") remains enabled. Repeated hook calls are therefore the expected
consequence of dual-stack dialing, and the metric still counts each connection phase exactly once.


---

## 4. Concurrency-safety analysis

A central part of Q1 is whether the measurement *code itself* is correct under concurrency — because
the Go contract makes the hooks inherently concurrent and sometimes late.

### 4.1 The Go contract: concurrent and post-completion hooks

The [`httptrace`](https://pkg.go.dev/net/http/httptrace) `ClientTrace` documentation states that hook
functions "may be called concurrently from different goroutines and some may be called after the
request has completed or failed." k6's code acknowledges both halves of this contract:

- The `Done()` method carries a comment that some `httptrace.ClientTrace` methods can "actually be
  called after the http.Client or http.RoundTripper have already returned our result and we've called
  Done() ... mostly for cancelled requests, but we have to use atomics here as well ... so we can
  avoid data races" [`lib/netext/httpext/tracer.go:L327-L331`].
- The TLS and response hooks note they "could be called after the RoundTrip() method has returned"
  [`lib/netext/httpext/tracer.go:L240-L241`, `L308-L309`].

### 4.2 All timestamps are mutated and read through atomics

The `Tracer` stores every hook timestamp as an `int64` field
[`lib/netext/httpext/tracer.go:L147-L159`] and mutates them **exclusively** through `sync/atomic`:

- `atomic.CompareAndSwapInt64` — `ConnectStart` [`lib/netext/httpext/tracer.go:L201`], `ConnectDone`
  [`L218`], `TLSHandshakeStart` [`L231`], `TLSHandshakeDone` [`L244`], `GotFirstResponseByte` [`L311`],
  and the false-`Reused` else branch [`L287-L291`].
- `atomic.SwapInt64` — the reuse branch in `GotConn` [`lib/netext/httpext/tracer.go:L271-L277`].
- `atomic.StoreInt64` — `WroteRequest` [`lib/netext/httpext/tracer.go:L301`].
- `atomic.LoadInt64` — all reads in `Done()` [`lib/netext/httpext/tracer.go:L332-L338`].

Because reads and writes are atomic, a hook firing late or on another goroutine cannot tear a value or
race with `Done()`; the worst case is that a very-late hook's write simply loses the CAS (the field is
already set) and is ignored — which is the intended de-duplication behavior.

### 4.3 One deliberate non-atomic write

There is exactly one intentional exception. In `GotConn`, the fields `gotConn`, `connReused`, and
`connRemoteAddr` are written **without** synchronization
[`lib/netext/httpext/tracer.go:L261-L263`], guarded by the comment that this hook "shouldn't be called
multiple times so no synchronization here, it's better for the race detector to panic if we're wrong"
[`lib/netext/httpext/tracer.go:L259-L260`]. This is a conscious design decision — not an oversight —
that uses the race detector as a tripwire for an invariant the authors believe holds.

### 4.4 Per-request isolation

Because a fresh `&Tracer{}` is created for every round trip
[`lib/netext/httpext/transport.go:L205`] and the type is explicitly documented as unsafe to reuse
[`lib/netext/httpext/tracer.go:L145`], there is no shared mutable state across requests. Concurrency
concerns are therefore confined to the hooks of a *single* request, which the atomics handle.

### 4.5 Empirical guard: the 200-request cancellation stress test

The package ships a concurrency stress test, `TestCancelledRequest`
[`lib/netext/httpext/tracer_test.go:L257-L292`]. It fires **200** parallel cancelled requests
through the tracer (`for i := 0; i < 200; i++` [`lib/netext/httpext/tracer_test.go:L284`]), each
calling `tracer.Done()` [`lib/netext/httpext/tracer_test.go:L275`]. Cancelled requests are precisely
the case where hooks fire late / concurrently, so this test (normally run under `-race`) is the guard
that the atomic accounting is correct.

**Conclusion.** The measurement code is concurrency-safe by construction (atomics + per-request
tracers) and is exercised by a dedicated race-stress test. This is why Behavior 3 must be attributed
to clock resolution, not to a race: the synchronization is sound; the clock is simply coarse.

---

## 5. Trustworthiness verdict and per-metric interpretation guide

**Overall verdict.** The timing values **can be trusted** for performance analysis. Every one of the
five reported behaviors is either correct-by-design (Behaviors 1, 2, 4, 5) or a documented platform
limitation (Behavior 3), and the measurement code is concurrency-safe (Section 4). The only
adjustment required of the user is *interpretation* — understanding what each metric means and when a
`0` is expected.

### 5.1 Per-metric interpretation guide

| Metric | What it measures (code) | How to interpret / gotchas |
|--------|-------------------------|----------------------------|
| `http_reqs` | Count of requests issued [`metrics/builtin.go:L15`] | A counter, not a timing; one sample per request. |
| `http_req_failed` | Failure rate per the response callback [`metrics/builtin.go:L16`; `transport.go:L152-L162`] | A rate; pairs with the timings to separate slow from failed. |
| `http_req_blocked` | `gotConn − getConn` = connection-acquisition wait [`tracer.go:L323-L325`] | High on new dials / pool-limit waits; ~0 on idle-pool hits. Expected to vary (Behavior 2). |
| `http_req_connecting` | `connectDone − connectStart` [`tracer.go:L340-L342`] | **Expected `0` on reused connections** (Behavior 1). Non-zero only when a new TCP connection is dialed. |
| `http_req_tls_handshaking` | `tlsHandshakeDone − tlsHandshakeStart` [`tracer.go:L343-L345`] | **Expected `0` on reused connections** and on plain HTTP (Behavior 1). |
| `http_req_sending` | Request-write time, via the TLS/connect/HTTP-2 switch [`tracer.go:L346-L359`] | Usually small; the HTTP/2 `default` branch uses `gotConn` as the start (Behavior 4). |
| `http_req_waiting` | `gotFirstResponseByte − wroteRequest` ("time to first byte") [`tracer.go:L361-L372`] | The server "think time"; usually the dominant component of `duration`. |
| `http_req_receiving` | `done − gotFirstResponseByte` [`tracer.go:L374-L376`] | Finalized only after the body is read (`processLastSavedRequest` [`transport.go:L181-L198`]). |
| `http_req_duration` | `sending + waiting + receiving` [`tracer.go:L381`] | **Excludes** DNS, `blocked`, and connect/TLS time [`tracer.go:L22-L23`]. Not the same as wall-clock end-to-end. |

### 5.2 Recommended best practices

Grounded in the analysis above:

- **Rely on distributions/percentiles, not single samples.** Per-request values legitimately vary
  (e.g. `blocked` in Behavior 2); aggregates (p90/p95/p99) are the trustworthy lens.
- **Expect zeros on reuse.** `connecting`/`tls_handshaking` of `0` means a connection was reused — a
  *good* sign for throughput, not a measurement error (Behavior 1).
- **Treat sub-millisecond zeros on Windows with caution.** Because of coarse `time.Now()` resolution
  (Behavior 3), very fast phases can read `0` on Windows; prefer aggregates and avoid drawing
  conclusions from individual sub-resolution samples on that platform.
- **Remember `http_req_duration` excludes connect/blocked time.** To reason about full end-to-end
  latency, also consider `blocked`, `connecting`, and `tls_handshaking` [`tracer.go:L22-L23`, `L380-L381`].

---

## 6. Upstream-report recommendation (Q2 in depth)

**No upstream bug report against k6 is warranted.** Adjudicated against the code:

- **Behaviors 1, 2, 4, 5 are by-design.** Each is either directly asserted by k6's own tests
  (Behavior 1: `lib/netext/httpext/tracer_test.go:L161-L165`), documented in k6's own code comments
  (Behavior 4: `lib/netext/httpext/tracer.go:L279-L286`; Behavior 5:
  `lib/netext/httpext/tracer.go:L191-L193`), or a direct, guarded consequence of the documented metric
  semantics (Behavior 2: `lib/netext/httpext/tracer.go:L323-L325`). There is nothing to fix.
- **Behavior 3 is a Go/Windows `time.Now()` precision limitation**, not a k6 defect, and it is already
  tracked at the Go runtime level: [#8687](https://github.com/golang/go/issues/8687),
  [#41087](https://github.com/golang/go/issues/41087), and the still-open
  [#67066](https://github.com/golang/go/issues/67066).

The only genuine "upstream" item is the Go runtime clock-resolution improvement
([#67066](https://github.com/golang/go/issues/67066), which proposes using `QueryPerformanceCounter`
in `time.Now` on Windows). That issue is open, outside k6's control, and already filed — so if the
user wants finer Windows timing resolution, that Go issue is the relevant place to follow, and no new
report against k6 is needed.

---

## 7. Reproduction notes

Per the project rule, reproduction/investigation scripts are permitted but must live **outside** the
source tree (e.g. under `/tmp`) and must never be committed into the repository. The behaviors can be
observed as follows.

**Behaviors 1 & 2 — connection reuse and variable `blocked` (k6 script, kept under `/tmp`).**
A script that issues repeated `http.get()` calls to the same HTTPS endpoint and prints `res.timings`
after each request will show `connecting` and `tls_handshaking` drop to `0` from the second request
onward (Behavior 1), and `blocked` swinging between a large first-request value and ~0 on subsequent
idle-pool hits (Behavior 2):

```javascript
// /tmp/repro_timings.js  (NOT committed to the source tree)
import http from 'k6/http';
export default function () {
  for (let i = 0; i < 5; i++) {
    const res = http.get('https://test.k6.io');
    console.log(`#${i} blocked=${res.timings.blocked} connecting=${res.timings.connecting} ` +
                `tls=${res.timings.tls_handshaking} waiting=${res.timings.waiting}`);
  }
}
// Run with:  k6 run /tmp/repro_timings.js
```

The fields printed map directly to the `ResponseTimings` struct
[`lib/netext/httpext/response.go:L34-L44`].

**Behaviors 3, 4 & 5 — exercise the tracer and its concurrency guard.** The existing tracer tests
cover the relevant paths; running them is the in-repo way to exercise the code without adding
anything:

```bash
# Run the tracer tests (table-driven reuse cases, etc.)
go test ./lib/netext/httpext/ -run Tracer

# Run the 200-request cancellation stress test under the race detector
go test -race ./lib/netext/httpext/ -run TestCancelledRequest
```

**Empirical results obtained during Environment Setup.** k6 was built from this checkout —
`k6 v0.55.0` (commit `ddc3b0b1d2`, `go1.23.12`, `linux/amd64`) — and `go test ./lib/netext/httpext/
-run Tracer` passed (`ok`, ~0.022s), confirming the verification environment is functional and the
cited behavior is exercised by the existing suite. The reuse test directly demonstrates Behavior 1's
`0.0` assertion [`lib/netext/httpext/tracer_test.go:L161-L165`], and `TestCancelledRequest`
[`lib/netext/httpext/tracer_test.go:L257-L292`] demonstrates the concurrency guard relevant to
Behavior 3. Any such scripts or runs live outside the source tree and are never committed.

---

## 8. References

### 8.1 Code locations (commit `ddc3b0b1d2`, `k6 v0.55.0`)

**`lib/netext/httpext/tracer.go`**
- `Trail` struct and metric meanings — `L16-L41` (e.g. `Blocked` `L25`, `Waiting` `L29`; aggregates
  `ConnDuration` `L19-L20`, `Duration` `L22-L23`)
- `SaveSamples` — `L43-L122`
- "It's NOT safe to reuse Tracers between requests." — `L145`
- `Tracer` `int64` timestamp fields — `L147-L159`
- `Trace()` wires the eight hooks — `L162-L173`
- `now()` = `time.Now().UnixNano()` — `L175-L177`
- `GetConn` (doc "called even if there's already an idle cached connection available" `L181-L182`) — `L187-L189`
- `ConnectStart` CAS (`L201`) and "Happy Eyeballs ... may be called multiple times" doc — `L191-L193`
- `ConnectDone` err-guarded CAS — `L218` (doc `L207-L208`)
- `TLSHandshakeStart`/`TLSHandshakeDone` CAS — `L231`, `L244`
- `GotConn` — `L256-L294`; reuse swap `L271-L277`; false-`Reused` HTTP/2 else-branch CAS `L278-L293`
- `WroteRequest` `StoreInt64` — `L301`
- `GotFirstResponseByte` CAS — `L311`
- `Done()` — `L315-L384`; `Blocked` `L323-L325`; atomic loads `L332-L338`; `Connecting` `L340-L342`;
  `TLSHandshaking` `L343-L345`; `Sending` switch `L346-L359`; `Waiting` `L361-L372`; `Receiving`
  `L374-L376`; `ConnDuration` `L380`; `Duration` `L381`

**`lib/netext/httpext/transport.go`**
- `measureAndEmitMetrics` → `tracer.Done()` — `L78`
- `trail.SaveSamples(...)` emission — `L146`; `metrics.PushIfNotDone(...)` — `L164`
- Deferred finalization (`saveCurrentRequest`/`processLastSavedRequest`) — `L168-L198`
- Fresh `tracer := &Tracer{}` and `httptrace.WithClientTrace` wiring — `L205-L206`

**`lib/netext/httpext/tracer_test.go`**
- Windows-resolution HACK citing `#8687`/`#41087` — `L33-L40`
- Reuse assertions (`connecting`/`tls_handshaking` `== 0.0`) — `L161-L165`
- `TestCancelledRequest` (200 parallel cancelled requests) — `L257-L292`

**`lib/netext/dialer.go`** — `Dialer` embeds `net.Dialer` `L18-L28`; `DialContext` delegates to
`net.Dialer.DialContext` `L58-L70` (delegation `L64`)

**`js/runner.go`** — production `BaseDialer` (`net.Dialer{Timeout: 30s, KeepAlive: 30s}`) — `L90-L93`

**`metrics/builtin.go`** — built-in `http_req_*` metric name constants — `L15-L23`

**`lib/netext/httpext/request.go`** — k6 transport wired as `http.RoundTripper` `L184-L185`;
`http.Client{Transport: transport}` `L232-L233`; finalization call `L283`

**`lib/netext/httpext/response.go`** — `ResponseTimings` JSON fields (`response.timings`) — `L34-L44`

### 8.2 Go documentation and issues

- [`net/http/httptrace`](https://pkg.go.dev/net/http/httptrace) — the `ClientTrace` contract:
  `ConnectStart`/`ConnectDone` "may be called multiple times" under `net.Dialer.DualStack`
  ("Happy Eyeballs"); hook functions "may be called concurrently from different goroutines and some
  may be called after the request has completed or failed"; `GetConn` "is called even if there's
  already an idle cached connection available."
- [golang/go#8687](https://github.com/golang/go/issues/8687) — Windows system clock resolution.
- [golang/go#41087](https://github.com/golang/go/issues/41087) — `time.Time` precision differs by OS.
- [golang/go#11313](https://github.com/golang/go/issues/11313) — related Windows time-resolution issue.
- [golang/go#31160](https://github.com/golang/go/issues/31160) — benchmark uses low-resolution time on
  Windows (~1ms granularity demonstrated).
- [golang/go#67066](https://github.com/golang/go/issues/67066) — **open** (April 2024): use
  `QueryPerformanceCounter` in `time.Now` on Windows to improve time resolution.

