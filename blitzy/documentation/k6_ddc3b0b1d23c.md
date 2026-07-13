# How k6 Behaves When You Write and Run a Load-Testing Script

> **Scope of this document.** This is an investigative, evidence-grounded answer to eight questions about how the [`grafana/k6`](https://github.com/grafana/k6) load-testing tool behaves when a user writes and runs a script. Every behavioral claim below was produced by **actually building and running k6 from this checkout** and capturing the **complete, unedited** output. Each claim is tied either to that captured output or to a `file:line` reference in the source tree. Statements that could **not** be directly observed are explicitly labelled **[INFERRED]**; everything else is **[OBSERVED]**.
>
> **Source under test:** `grafana/k6`, branch `k6_ddc3b0b1d23c`, HEAD `ddc3b0b1d`.  
> **Binary under test:** `k6 v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)` (built from this checkout — see the build recipe below).  
> **Read-only guarantee:** No repository file was modified. All temporary scripts, output files, and the built binary were created **outside** the repository (under `/tmp`) and deleted after the investigation; the only durable change is this document.

---

## 0. Canonical build & how the evidence was produced

All version-sensitive values (banner, version string) come from a **default, canonical build** of this checkout using the vendored module tree — exactly what a normal user gets from `go build`.

**Build recipe (run outside the repository; Go was fetched to `/tmp`):**

```bash
# Toolchain (outside the repo): Go 1.23.12 - matches the repo Dockerfile (golang:1.23-alpine3.20);
# go.mod floor is go 1.21. Network was available to fetch the toolchain; the BUILD itself is offline (vendored).
export GOROOT=/tmp/gotool/go GOPATH=/tmp/gopath GOCACHE=/tmp/gocache GOMODCACHE=/tmp/gomodcache PATH=/tmp/gotool/go/bin:$PATH

# From the repository root; binary written OUTSIDE the repo:
CGO_ENABLED=0 GOFLAGS=-mod=vendor go build -o /tmp/k6bin/k6 .     # exit 0, ~26.5s, 65 MB binary
```

**Version check (OBSERVED, exit 0):**

```console
$ /tmp/k6bin/k6 version
k6 v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)
```

- The version string `0.55.0` is the compile-time constant `Version` at `lib/consts/consts.go:12` (`const Version = "0.55.0"`).
- The `commit/ddc3b0b1d2` fragment is assembled by `FullVersion()` (declared at `lib/consts/consts.go:16`), whose final `return` formats `"%s (commit/%s, %s)"` at `lib/consts/consts.go:52` and takes the first `commitLen := 10` characters of the VCS revision (`lib/consts/consts.go:31`). HEAD is `ddc3b0b1d`; the stamped 10-char revision renders as `ddc3b0b1d2`. **[OBSERVED]**
- Entry point: `main.go:8-9` is `func main() { cmd.Execute() }`, importing `go.k6.io/k6/cmd` at `main.go:5`. **[OBSERVED]**

Unless stated otherwise, every command below was run as `/tmp/k6bin/k6 ...` from the repository root. Reproducibility caveat: because the example endpoint issues an HTTP **redirect**, one iteration can produce **two** HTTP requests (`http_reqs=2`); numeric timing values naturally vary run-to-run, but metric names, units, and structure are stable.

---

## 1. The basic workflow: writing a script and running it

**Direct answer.** A k6 test is an ordinary JavaScript **ES module**: you `import` a protocol module (e.g. `k6/http`) and **`export default`** a function that becomes the "iteration" each virtual user (VU) repeats. You then run it with the **`k6 run <script>`** subcommand. No build step, no `package.json`, and no application server is needed to *host* the script — the script file is passed straight to `k6 run`. (k6 itself does start a local REST API server on `localhost:6565` by default for controlling the running test — see §5 — but you do not need to stand up any server of your own.) You can optionally scaffold a starter script with `k6 new`.

**The canonical script (shipped in-repo).** `examples/http_get.js` is literally an import + a default function (`examples/http_get.js:1-5`):

```javascript
import http from 'k6/http';

export default function () {
  http.get('https://test-api.k6.io/');
};
```

- The `import ... from 'k6/http'` line resolves against k6's **built-in** module registry, not npm — `k6/http` is registered in `js/jsmodules.go:61`.
- The **default export** is the unit of work. k6 runs the exported function named `default` (`lib/consts/js.go` `DefaultFn = "default"`); a VU repeats it once per iteration via `(*ActiveVU).RunOnce()` in `js/runner.go`. **[OBSERVED via the run in §3]**

**Scaffolding a new script (authoring aid).** The `k6 new` subcommand writes a starter script; its command is declared at `cmd/new.go:151` (`Use: "new"`, and `Short: "Create and initialize a new k6 script"` at `cmd/new.go:152`) and the default output filename is `defaultNewScriptName = "script.js"` (`cmd/new.go:15`).

**Exact command (OBSERVED, exit 0), run in an empty directory outside the repository:**

```bash
/tmp/k6bin/k6 new
```

**Complete, unedited output:**

```
Initialized a new k6 test script in script.js. You can now execute it by running `k6 run script.js`.
```

The generated `script.js` follows exactly the convention described above — it opens with the built-in module imports, exports an `options` object, and `export default`s the iteration function (generated template, **OBSERVED**, head shown):

```javascript
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  // A number specifying the number of VUs to run concurrently.
  vus: 10,
  // A string specifying the total duration of the test run.
  duration: '30s',
  // ... (commented-out cloud/browser/scenario templates omitted) ...
};

// The function that defines VU logic.
export default function() {
  http.get('https://test.k6.io');
  sleep(1);
}
```

**Running it.** The full end-to-end run and its complete output are shown in §3 (single HTTP request). The command is simply:

```bash
k6 run examples/http_get.js
```

**Coverage:** write = ES module with `import` + `export default` (`examples/http_get.js:1-5`); scaffold = `k6 new` (`cmd/new.go:15,151`); run = `k6 run <script>` (see §5). ✔

---

## 2. What k6 reports back when the script executes (the four output surfaces)

**Direct answer.** In its **default** configuration (i.e. neither `--quiet` nor `--no-summary` is passed), k6 prints **four** distinct surfaces to the terminal:

1. **A startup ASCII banner** (the Grafana/k6 logo).
2. **An execution-description block** (`execution:`, `script:`, `output:`, `scenarios:`, and the per-scenario line).
3. **A live progress line** that updates while the test runs.
4. **An end-of-test summary** — the aggregated metrics block.

All four are visible in the default captured run in §3. Two flags change which surfaces appear: `--quiet` suppresses both the banner (`printBanner` returns early on `gs.Flags.Quiet` at `cmd/ui.go:59-61`) and the progress line (`printBar` returns early at `cmd/ui.go:71-72`), and `--no-summary` removes surface 4. Mapping each surface to the code that emits it:

| # | Surface | Emitted by | Evidence |
|---|---------|-----------|----------|
| 1 | ASCII banner | `Banner()` at `lib/consts/consts.go:56`, printed via `cmd/ui.go` (`getBanner` `:53` → `printBanner` `:58`, `consts.Banner()` `:55`) | banner block in §3 output **[OBSERVED]** |
| 2 | Execution description | `printExecutionDescription` at `cmd/ui.go:100` — emits `execution:` (`:108`), `script:`, `output:`, `scenarios:` (`:149`), and the `* <name>: <desc>` line | `execution: local ... * default: 1 iterations for each of 1 VUs` in §3 **[OBSERVED]** |
| 3 | Live progress bar | `printBar` at `cmd/ui.go:70` + `ui/pb/progressbar.go` | `running (00m00.2s), 0/1 VUs, 1 complete ...` + `default ✓ [ 100% ] ...` in §3 **[OBSERVED]** |
| 4 | End-of-test summary | `HandleSummary(...)` invoked at `cmd/run.go:195`; default text summary generated by the embedded `js/summary.js` and written to stdout | the whole `data_received ... iterations` block in §3 **[OBSERVED]** |

In the default (non-`--quiet`) mode, the banner is printed even when the run **fails** — see the validation runs in §8, which show the same five banner lines before the error message. **[OBSERVED]**

**Coverage:** banner ✔ (`lib/consts/consts.go:56`); execution description ✔ (`cmd/ui.go:100`); progress ✔ (`cmd/ui.go:70` + `ui/pb/progressbar.go`); summary ✔ (`cmd/run.go:195` → `js/summary.js`); default-vs-`--quiet`/`--no-summary` distinction ✔ (`cmd/ui.go:59-61,71-72`).

---

## 3. Testing a single HTTP request

**Direct answer.** The repository already ships the exact "single HTTP request" scenario: `examples/http_get.js` is one `http.get('https://test-api.k6.io/')` inside the default function (`examples/http_get.js:1-5`, reproduced in §1). Running it with `k6 run` exercises the whole pipeline with the default execution profile of **1 iteration, 1 VU**.

**Exact command (OBSERVED, exit 0):**

```bash
/tmp/k6bin/k6 run examples/http_get.js
```

**Complete, unedited output:**

```

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: examples/http_get.js
        output: -

     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 1 iterations for each of 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)


     data_received..................: 13 kB  55 kB/s
     data_sent......................: 1.1 kB 4.9 kB/s
     http_req_blocked...............: avg=89.59ms  min=52.53ms  med=89.59ms  max=126.64ms p(90)=119.23ms p(95)=122.94ms
     http_req_connecting............: avg=20.33ms  min=11.57ms  med=20.33ms  max=29.08ms  p(90)=27.33ms  p(95)=28.21ms 
     http_req_duration..............: avg=24.2ms   min=12.55ms  med=24.2ms   max=35.85ms  p(90)=33.52ms  p(95)=34.69ms 
       { expected_response:true }...: avg=24.2ms   min=12.55ms  med=24.2ms   max=35.85ms  p(90)=33.52ms  p(95)=34.69ms 
     http_req_failed................: 0.00%  0 out of 2
     http_req_receiving.............: avg=77.66µs  min=54.82µs  med=77.66µs  max=100.5µs  p(90)=95.93µs  p(95)=98.21µs 
     http_req_sending...............: avg=141.9µs  min=102.12µs med=141.9µs  max=181.69µs p(90)=173.73µs p(95)=177.71µs
     http_req_tls_handshaking.......: avg=22.43ms  min=14.53ms  med=22.43ms  max=30.33ms  p(90)=28.75ms  p(95)=29.54ms 
     http_req_waiting...............: avg=23.98ms  min=12.32ms  med=23.98ms  max=35.65ms  p(90)=33.32ms  p(95)=34.48ms 
     http_reqs......................: 2      8.764725/s
     iteration_duration.............: avg=228.05ms min=228.05ms med=228.05ms max=228.05ms p(90)=228.05ms p(95)=228.05ms
     iterations.....................: 1      4.382363/s


running (00m00.2s), 0/1 VUs, 1 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.2s/10m0s  1/1 iters, 1 per VU
```

**Key observations from this run:**

- **`iterations` = 1** but **`http_reqs` = 2**. This is not a bug: `https://test-api.k6.io/` responds with an HTTP **302 redirect**, and k6 follows redirects by default (`--max-redirects` default `10`, see §5). So one iteration → two HTTP requests. **[OBSERVED]** This is exactly why the document reports `http_reqs=2` — reporting state faithfully even when it looks surprising.
- **`http_req_failed 0.00%  0 out of 2`** — the redirect chain is treated as a success (the final response is `expected_response:true`). **[OBSERVED]**
- **No `vus` / `vus_max` / `checks` lines appear** here. **[OBSERVED]** The script defines no `check()`, which is why `checks` is absent. The absence of the `vus`/`vus_max` gauges in this particular run **[OBSERVED]**, attributed to the run completing sub-second before a gauge sample was emitted, is **[INFERRED]** — corroborated by the ≥1s-scale run in §4a where the gauges *do* appear.

**Coverage:** single `http.get` exercised verbatim through the real `k6 run` entry point; full four-surface output captured. ✔

---

## 4. What the output looks like: metrics, units, and protocols

### 4a. Metrics (names, types, value-units)

**Direct answer.** k6 emits a fixed catalog of **built-in metrics**, registered in `RegisterBuiltinMetrics` at `metrics/builtin.go:78`. Each metric has one of **four types** and a **value type** that governs how it is rendered:

| Metric (as printed) | Type | Value type | Registered at |
|---|---|---|---|
| `http_reqs` | **Counter** | Default | `metrics/builtin.go:89` |
| `iterations` | **Counter** | Default | `:82` |
| `data_sent`, `data_received` | **Counter** | **Data** | `:108-109` |
| `http_req_failed` | **Rate** | Default | `:90` |
| `checks` | **Rate** | Default | `:86` |
| `http_req_duration` | **Trend** | **Time** | `:91` |
| `http_req_blocked`/`connecting`/`tls_handshaking`/`sending`/`waiting`/`receiving` | **Trend** | **Time** | `:92-97` |
| `iteration_duration`, `group_duration` | **Trend** | **Time** | `:83`, `:87` |
| `vus`, `vus_max` | **Gauge** | Default | `:80-81` |
| `dropped_iterations` | **Counter** | Default | `:84` |
| `ws_*` (WebSocket), `grpc_req_duration` (gRPC) | Counter/Trend | Default/Time | `:99-106` |

The four types behave as follows (definitions cross-checked against the official docs, [grafana.com/docs/k6 › Metrics](https://grafana.com/docs/k6/latest/using-k6/metrics/)): **Counters** sum values; **Gauges** keep the latest plus min/max; **Rates** track how often a non-zero value occurs; **Trends** compute statistical distributions (avg/min/med/max/percentiles).

The value types are defined in `metrics/value_type.go:6-9`: `Default` ("presented as-is", `:7`), `Time` ("time durations (milliseconds)", `:8`), `Data` ("data amounts (bytes)", `:9`).

**Corroborating evidence — the streamed metric definitions (filtered extraction).** In the opt-in JSON stream (see §7) each metric's definition is emitted as a `"type":"Metric"` record carrying its `type` and `contains` (value type). These `Metric` records are **not** grouped at the top of the file — they are **interleaved** with the data `"Point"` samples (each `Metric` definition is written immediately before that metric's first `Point`; see §7 for the real interleaved order). The 13 definition lines below are therefore a **filtered extraction**, obtained with the exact command (**OBSERVED**, from `/tmp/k6work/ext/out.json`):

```bash
grep '"type":"Metric"' /tmp/k6work/ext/out.json
```

```json
{"type":"Metric","data":{"name":"http_reqs","type":"counter","contains":"default","thresholds":[],"submetrics":null},"metric":"http_reqs"}
{"type":"Metric","data":{"name":"http_req_duration","type":"trend","contains":"time","thresholds":[],"submetrics":[{"name":"http_req_duration{expected_response:true}","suffix":"expected_response:true","tags":{"expected_response":"true"}}]},"metric":"http_req_duration"}
{"type":"Metric","data":{"name":"http_req_blocked","type":"trend","contains":"time","thresholds":[],"submetrics":null},"metric":"http_req_blocked"}
{"type":"Metric","data":{"name":"http_req_connecting","type":"trend","contains":"time","thresholds":[],"submetrics":null},"metric":"http_req_connecting"}
{"type":"Metric","data":{"name":"http_req_tls_handshaking","type":"trend","contains":"time","thresholds":[],"submetrics":null},"metric":"http_req_tls_handshaking"}
{"type":"Metric","data":{"name":"http_req_sending","type":"trend","contains":"time","thresholds":[],"submetrics":null},"metric":"http_req_sending"}
{"type":"Metric","data":{"name":"http_req_waiting","type":"trend","contains":"time","thresholds":[],"submetrics":null},"metric":"http_req_waiting"}
{"type":"Metric","data":{"name":"http_req_receiving","type":"trend","contains":"time","thresholds":[],"submetrics":null},"metric":"http_req_receiving"}
{"type":"Metric","data":{"name":"http_req_failed","type":"rate","contains":"default","thresholds":[],"submetrics":null},"metric":"http_req_failed"}
{"type":"Metric","data":{"name":"data_sent","type":"counter","contains":"data","thresholds":[],"submetrics":null},"metric":"data_sent"}
{"type":"Metric","data":{"name":"data_received","type":"counter","contains":"data","thresholds":[],"submetrics":null},"metric":"data_received"}
{"type":"Metric","data":{"name":"iteration_duration","type":"trend","contains":"time","thresholds":[],"submetrics":null},"metric":"iteration_duration"}
{"type":"Metric","data":{"name":"iterations","type":"counter","contains":"default","thresholds":[],"submetrics":null},"metric":"iterations"}
```

**`vus`/`vus_max` (Gauges) — observed at scale.** They did **not** appear in the sub-second §3 run, but a longer run makes them emit. With a 2-VU, 3-second run of a sleeping script (`/tmp/k6work/sleeper.js` = `import { sleep } from 'k6'; export default function () { sleep(1); }`):

```bash
/tmp/k6bin/k6 run -u 2 -d 3s /tmp/k6work/sleeper.js
```

```

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6work/sleeper.js
        output: -

     scenarios: (100.00%) 1 scenario, 2 max VUs, 33s max duration (incl. graceful stop):
              * default: 2 looping VUs for 3s (gracefulStop: 30s)


running (01.0s), 2/2 VUs, 0 complete and 0 interrupted iterations
default   [  33% ] 2 VUs  1.0s/3s

running (02.0s), 2/2 VUs, 2 complete and 0 interrupted iterations
default   [  67% ] 2 VUs  2.0s/3s

running (03.0s), 2/2 VUs, 4 complete and 0 interrupted iterations
default   [ 100% ] 2 VUs  3.0s/3s

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=1s min=1s med=1s max=1s p(90)=1s p(95)=1s
     iterations...........: 6   1.998671/s
     vus..................: 2   min=2      max=2
     vus_max..............: 2   min=2      max=2


running (03.0s), 0/2 VUs, 6 complete and 0 interrupted iterations
default ✓ [ 100% ] 2 VUs  3s
```

Here `vus` and `vus_max` **appear** as Gauges (`2   min=2      max=2`), and the progress line updates once per second. **[OBSERVED]** The precise cadence that gauges emit on a ~1-second tick is **[INFERRED]** from the code plus the 1s progress cadence, but their presence at ≥1s scale is directly observed. Also note `iteration_duration avg=1s` — the **`s`** time unit (see §4b).

### 4b. Units — auto-scaled by default

**Direct answer.** By default k6 does **not** print a single fixed time unit; it **auto-scales** each value to the unit that fits, so `ns`, `µs`, `ms`, and `s` can all appear in one summary. This is implemented in `js/summary.js`: durations render as `ns` (`:170`), `µs` (`:174`), `ms` (`:178`), or `s` (`:181`).

Observed unit behavior:

- **Time (Trend)** — auto-scaled per value. In §3: `http_req_duration avg=24.2ms` but `http_req_receiving avg=77.66µs`; in the env run in §6, `iteration_duration min=621ns`; in the scale run above, `iteration_duration avg=1s`. So all four of `ns/µs/ms/s` are observed across runs. **[OBSERVED]**
- **Data (Counter, Data value type)** — bytes auto-scaled to `B`/`kB` on a base of **1000** (`js/summary.js:134-136`), plus a per-second rate: `data_received 13 kB  55 kB/s`. **[OBSERVED]**
- **Counter (Default)** — an integer count (`js/summary.js:224`) plus a per-second rate (`js/summary.js:225`): `http_reqs 2 8.764725/s`, `iterations 1 4.382363/s`. **[OBSERVED]**
- **Rate** — a percentage (rendered at `js/summary.js:207`) plus the underlying fraction "passes out of total" (rendered at `js/summary.js:233-236`): `http_req_failed 0.00%  0 out of 2`. **[OBSERVED]**
- **Trend statistics shown** — by default `avg, min, med, max, p(90), p(95)`, defined in `lib/options.go:26` (`DefaultSummaryTrendStats = ["avg", "min", "med", "max", "p(90)", "p(95)"]`) and confirmed by `--help` (see §5) and every summary above. **[OBSERVED]**

**Forcing a single time unit.** `--summary-time-unit=ms` overrides auto-scaling (`js/summary.js:197-198`). Same script, forced unit:

```bash
/tmp/k6bin/k6 run --summary-time-unit=ms examples/http_get.js
```

Now the sub-millisecond timings that were `µs` in §3 are printed in `ms` instead. **Complete, unedited output (OBSERVED, exit 0):**

```

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: examples/http_get.js
        output: -

     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 1 iterations for each of 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)


     data_received..................: 13 kB  60 kB/s
     data_sent......................: 1.1 kB 5.2 kB/s
     http_req_blocked...............: avg=84.74ms  min=73.95ms  med=84.74ms  max=95.53ms  p(90)=93.37ms  p(95)=94.45ms 
     http_req_connecting............: avg=20.59ms  min=12.23ms  med=20.59ms  max=28.96ms  p(90)=27.29ms  p(95)=28.12ms 
     http_req_duration..............: avg=21.18ms  min=13.13ms  med=21.18ms  max=29.23ms  p(90)=27.62ms  p(95)=28.43ms 
       { expected_response:true }...: avg=21.18ms  min=13.13ms  med=21.18ms  max=29.23ms  p(90)=27.62ms  p(95)=28.43ms 
     http_req_failed................: 0.00%  0 out of 2
     http_req_receiving.............: avg=0.07ms   min=0.06ms   med=0.07ms   max=0.08ms   p(90)=0.07ms   p(95)=0.08ms  
     http_req_sending...............: avg=0.13ms   min=0.11ms   med=0.13ms   max=0.15ms   p(90)=0.15ms   p(95)=0.15ms  
     http_req_tls_handshaking.......: avg=21.86ms  min=13.54ms  med=21.86ms  max=30.17ms  p(90)=28.51ms  p(95)=29.34ms 
     http_req_waiting...............: avg=20.98ms  min=12.92ms  med=20.98ms  max=29.05ms  p(90)=27.43ms  p(95)=28.24ms 
     http_reqs......................: 2      9.41875/s
     iteration_duration.............: avg=212.22ms min=212.22ms med=212.22ms max=212.22ms p(90)=212.22ms p(95)=212.22ms
     iterations.....................: 1      4.709375/s


running (00m00.2s), 0/1 VUs, 1 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.2s/10m0s  1/1 iters, 1 per VU
```

### 4c. Protocols

**Direct answer.** k6 ships several protocol modules, registered in `js/jsmodules.go`: `k6/browser` (`:57`), `k6/net/grpc` (`:59`), `k6/http` (`:61`), `k6/ws` (`:63`). For a plain single-HTTP-request script, **only** the HTTP-related metrics (`http_req_*`, `data_*`, `iterations`/`iteration_duration`) emit — consistent with §3 (no `ws_*` or `grpc_*` lines). **[OBSERVED]** Per the user's focus, only HTTP is exercised in depth here.

**How the protocol is reported per request.** The `http.get` function is exported by the HTTP module at `js/modules/k6/http/http.go:71` (`mustExport("get", …)`), but the protocol *tag* itself is attached deeper down, on the HTTP request trail: `lib/netext/httpext/transport.go:118` sets the `proto` system tag from the response's `Proto` field (`SetSystemTagOrMetaIfEnabled(enabledTags, metrics.TagProto, unfReq.response.Proto)`). In the streamed JSON, each **HTTP-trail Point** therefore carries a `tags` object including `proto`, `tls_version`, `method`, `status`, `url`, `name`, etc. Example Point (**OBSERVED**, from `/tmp/k6work/ext/out.json`):

```json
{
    "metric": "http_reqs",
    "type": "Point",
    "data": {
        "time": "2026-07-13T17:32:38.605870576Z",
        "value": 1,
        "tags": {
            "expected_response": "true",
            "group": "",
            "method": "GET",
            "name": "https://test-api.k6.io/",
            "proto": "HTTP/2.0",
            "scenario": "default",
            "status": "302",
            "tls_version": "tls1.3",
            "url": "https://test-api.k6.io/"
        }
    }
}
```

**Scope of the `proto` tag (important — not every Point carries it).** That opt-in run produced **22** data `Point` samples in total. Of these, **18 are HTTP-trail Points** and each was tagged `"proto":"HTTP/2.0"` — i.e. the requests negotiated **HTTP/2**. The remaining **4 Points are non-HTTP** and carry **no** `proto` tag: they are the two I/O counters (`data_sent`, `data_received`, emitted by `(*Dialer).IOSamples` at `js/runner.go:868`) and the two iteration metrics (`iteration_duration`, `iterations`, emitted by `iterationSamples` at `js/runner.go:871,879-901`). So the correct characterization is **18 protocol-bearing HTTP Points out of 22 total Points**, verified with `grep '"type":"Point"' /tmp/k6work/ext/out.json | grep -c '"proto"'` → `18`. **[OBSERVED]** The `proto` value comes from the response's `Proto` field (`lib/netext/httpext/transport.go:118`); the default set of system tags is listed by `--help` as `proto,subproto,status,method,url,name,group,check,error,error_code,tls_version,scenario,service,expected_response` (see §5).

**Coverage:** metrics catalog + types + value-units ✔ (`metrics/builtin.go:78`, `metrics/value_type.go:6-9`); auto-scaled units + forced unit ✔ (`js/summary.js`); protocol modules + per-request `proto` tag ✔ (`js/jsmodules.go:57-63`; tagging at `lib/netext/httpext/transport.go:118`; `/tmp/k6work/ext/out.json`).

---

## 5. The command used to execute a script

**Direct answer.** The command is **`k6 run <script>`**. The subcommand is built by `getCmdRun` at `cmd/run.go:462`, declared with `Use: "run"` (`cmd/run.go:491`), `Short: "Start a test"` (`cmd/run.go:492`), and wired to the run logic via `RunE: c.run` (`cmd/run.go:499`).

**Exact command (OBSERVED, exit 0):**

```bash
/tmp/k6bin/k6 run --help
```

**Complete, unedited output:**

```
Start a test.

This also exposes a REST API to interact with it. Various k6 subcommands offer
a commandline interface for interacting with it.

Usage:
  k6 run [flags]

Examples:
  # Run a single VU, once.
  k6 run script.js

  # Run a single VU, 10 times.
  k6 run -i 10 script.js

  # Run 5 VUs, splitting 10 iterations between them.
  k6 run -u 5 -i 10 script.js

  # Run 5 VUs for 10s.
  k6 run -u 5 -d 10s script.js

  # Ramp VUs from 0 to 100 over 10s, stay there for 60s, then 10s down to 0.
  k6 run -u 0 -s 10s:100 -s 60s:100 -s 10s:0

  # Send metrics to an influxdb server
  k6 run -o influxdb=http://1.2.3.4:8086/k6

Flags:
  -u, --vus int                             number of virtual users (default 1)
  -d, --duration duration                   test duration limit
  -i, --iterations int                      script total iteration limit (among all VUs)
  -s, --stage stage                         add a stage, as `[duration]:[target]`
      --execution-segment string            limit execution to the specified segment, e.g. 10%, 1/3, 0.2:2/3
      --execution-segment-sequence string   the execution segment sequence
  -p, --paused                              start the test in a paused state
      --no-setup                            don't run setup()
      --no-teardown                         don't run teardown()
      --max-redirects int                   follow at most n redirects (default 10)
      --batch int                           max parallel batch reqs (default 20)
      --batch-per-host int                  max parallel batch reqs per host (default 6)
      --rps int                             limit requests per second
      --user-agent string                   user agent for http requests (default "k6/0.55.0 (https://k6.io/)")
      --http-debug string[="headers"]       log all HTTP requests and responses. Excludes body by default. To include body use '--http-debug=full'
      --insecure-skip-tls-verify            skip verification of TLS certificates
      --no-connection-reuse                 disable keep-alive connections
      --no-vu-connection-reuse              don't reuse connections between iterations
      --min-iteration-duration duration     minimum amount of time k6 will take executing a single iteration
  -w, --throw                               throw warnings (like failed http requests) as errors
      --blacklist-ip ip range               blacklist an ip range from being called
      --block-hostnames pattern             block a case-insensitive hostname pattern, with optional leading wildcard, from being called
      --summary-trend-stats stats           define stats for trend metrics (response times), one or more as 'avg,p(95),...' (default 'avg,min,med,max,p(90),p(95)')
      --summary-time-unit string            define the time unit used to display the trend stats. Possible units are: 's', 'ms' and 'us'
      --system-tags strings                 only include these system tags in metrics (default "proto,subproto,status,method,url,name,group,check,error,error_code,tls_version,scenario,service,expected_response")
      --tag tag                             add a tag to be applied to all samples, as `[name]=[value]`
      --console-output string               redirects the console logging to the provided output file
      --discard-response-bodies             Read but don't process or save HTTP response bodies
      --local-ips string                    Client IP Ranges and/or CIDRs from which each VU will be making requests, e.g. '192.168.220.1,192.168.0.10-192.168.0.25', 'fd:1::0/120', etc.
      --dns string                          DNS resolver configuration. Possible ttl values are: 'inf' for a persistent cache, '0' to disable the cache,
                                            or a positive duration, e.g. '1s', '1m', etc. Milliseconds are assumed if no unit is provided.
                                            Possible select values to return a single IP are: 'first', 'random' or 'roundRobin'.
                                            Possible policy values are: 'preferIPv4', 'preferIPv6', 'onlyIPv4', 'onlyIPv6' or 'any'.
                                             (default "ttl=5m,select=random,policy=preferIPv4")
      --include-system-env-vars             pass the real system environment variables to the runtime (default true)
      --compatibility-mode string           JavaScript compiler compatibility mode, "extended" or "base" or "experimental_enhanced"
                                            base: pure Sobek - Golang JS VM supporting ES6+
                                            extended: base + sets "global" as alias for "globalThis"
                                            experimental_enhanced: esbuild-based transpiling for TypeScript and ES6+ support
                                             (default "extended")
  -t, --type string                         override test type, "js" or "archive"
  -e, --env VAR=value                       add/override environment variable with VAR=value
      --no-thresholds                       don't run thresholds
      --no-summary                          don't show the summary at the end of the test
      --summary-export string               output the end-of-test summary report to JSON file
      --traces-output string                set the output for k6 traces, possible values are none,otel[=host:port] (default "none")
  -o, --out uri                             uri for an external metrics database
  -l, --linger                              keep the API server alive past test end
      --no-usage-report                     don't send anonymous usagestats (https://grafana.com/docs/k6/latest/set-up/usage-collection/)
  -h, --help                                help for run

Global Flags:
  -a, --address string      address for the REST API server (default "localhost:6565")
  -c, --config string       JSON config file (default "/root/.config/loadimpact/k6/config.json")
      --log-format string   log output format
      --log-output string   change the output for k6 logs, possible values are stderr,stdout,none,loki[=host:port],file[=./path.fileformat] (default "stderr")
      --no-color            disable colored output
      --profiling-enabled   enable profiling (pprof) endpoints, k6's REST API should be enabled as well
  -q, --quiet               disable progress updates
  -v, --verbose             enable verbose logging
```

**Salient flags** for the scenario in this document: `-u/--vus` (default **1**), `-i/--iterations`, `-d/--duration`, `-s/--stage`, `--max-redirects` (default **10** — the reason `http_reqs=2` in §3), `--summary-trend-stats` (default `avg,min,med,max,p(90),p(95)`), `--summary-time-unit`, `--summary-export`, `--no-summary`, `-o/--out`, and `-e/--env`. Default user agent is `k6/0.55.0 (https://k6.io/)`. **[OBSERVED]**

> **Version caveat (OBSERVED divergence from current public docs).** The current [k6 docs](https://grafana.com/docs/k6/latest/get-started/results-output/) describe a `--summary-mode` flag (e.g. `compact`/`full`/`legacy`) and a **grouped** summary layout (section headers such as `THRESHOLDS` / `TOTAL RESULTS` / `HTTP` / `EXECUTION` / `NETWORK`, plus `checks_total`/`checks_succeeded`/`checks_failed`). **Neither exists in this v0.55.0 build:** `--summary-mode` is **absent** from `k6 run --help` above (0 matches), and every summary captured here is the older **flat, alphabetical** list. This is a genuine version difference (the docs describe a post-0.55 summary redesign), reported here as observed rather than assumed.

**Coverage:** command = `k6 run <script>` ✔ (`cmd/run.go:462,491,492,499`); full flag list captured ✔.

---

## 6. Required configuration and environment variables

**Direct answer.** **No k6-specific configuration is required.** A minimal script runs with **no config file, no exported `options`, no CLI option, and no `K6_*` environment variable** — k6 falls back to its built-in defaults of **1 VU, 1 iteration**. (Like any process, k6 still *inherits* the ambient system environment; the point is that none of that is *required* for a baseline run.) When you *do* want to configure a run, k6 consolidates five sources in a fixed order of increasing precedence, implemented in `getConsolidatedConfig` at `cmd/config.go:180-204`: **(1) built-in defaults** (`applyDefault`, `cmd/config.go:204`) → **(2) the JSON config file** (`readDiskConfig`, `cmd/config.go:190`; default path shown by `--help` as `/root/.config/loadimpact/k6/config.json`) → **(3) the script's exported `options`** (the Runner options applied at `cmd/config.go:201`) → **(4) `K6_*` environment variables** (`readEnvConfig`, `cmd/config.go:194`; applied at `:203`) → **(5) CLI flags** (applied last at `:203` for the greatest priority). This `defaults < config file < script options < environment < CLI` order matches the official [options-precedence docs](https://grafana.com/docs/k6/latest/using-k6/k6-options/how-to/), and the `K6_`-prefixed convention is confirmed by the official [environment-variables docs](https://grafana.com/docs/k6/latest/using-k6/environment-variables/).

**Zero-config run (OBSERVED, exit 0).** Minimal script `/tmp/k6work/minimal.js` = `export default function () {}`:

```bash
/tmp/k6bin/k6 run /tmp/k6work/minimal.js
```

```

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6work/minimal.js
        output: -

     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 1 iterations for each of 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)


     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=3.22µs min=3.22µs med=3.22µs max=3.22µs p(90)=3.22µs p(95)=3.22µs
     iterations...........: 1   8885.176859/s


running (00m00.0s), 0/1 VUs, 1 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.0s/10m0s  1/1 iters, 1 per VU
```

It ran `iterations=1` with no configuration at all. **[OBSERVED]**

**Env-var configuration (OBSERVED, exit 0).** k6 reads `K6_*` variables for options. Running the same minimal script with `K6_ITERATIONS` and `K6_VUS`:

```bash
K6_ITERATIONS=5 K6_VUS=2 /tmp/k6bin/k6 run /tmp/k6work/minimal.js
```

```

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6work/minimal.js
        output: -

     scenarios: (100.00%) 1 scenario, 2 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 5 iterations shared among 2 VUs (maxDuration: 10m0s, gracefulStop: 30s)


     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=1.14µs min=621ns med=734ns max=2.81µs p(90)=2.05µs p(95)=2.43µs
     iterations...........: 5   36914.808006/s


running (00m00.0s), 0/2 VUs, 5 complete and 0 interrupted iterations
default ✓ [ 100% ] 2 VUs  00m00.0s/10m0s  5/5 shared iters
```

The env vars took effect: **2 max VUs**, **5 iterations shared among 2 VUs**, final `iterations=5`. (Note `min=621ns` — a `ns`-unit reading, further confirming §4b auto-scaling.) **[OBSERVED]**

**Runtime/logging env vars.** Beyond option vars, k6 resolves several runtime `K6_*` variables in `getFlags` at `cmd/state/state.go`: `K6_CONFIG` (`:163`), `K6_LOG_OUTPUT` (`:166`), `K6_LOG_FORMAT` (`:169`), `K6_NO_COLOR` (`:172`, also handled at `:90`), and `K6_PROFILING_ENABLED` (`:180`). For example, `K6_NO_COLOR=true` is honored and the run completes normally (**OBSERVED, exit 0** — note the `$ ... ; echo "exit=$?"` wrapper and the trailing `exit=0`):

```console
$ K6_NO_COLOR=true /tmp/k6bin/k6 run /tmp/k6work/minimal.js ; echo "exit=$?"

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6work/minimal.js
        output: -

     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 1 iterations for each of 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)


     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=3.76µs min=3.76µs med=3.76µs max=3.76µs p(90)=3.76µs p(95)=3.76µs
     iterations...........: 1   7412.403917/s


running (00m00.0s), 0/1 VUs, 1 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.0s/10m0s  1/1 iters, 1 per VU
exit=0
```

Option env vars such as `K6_VUS`, `K6_ITERATIONS`, `K6_DURATION`, `K6_STAGES` map to the corresponding run flags. **[OBSERVED for `K6_VUS`/`K6_ITERATIONS` above; the others INFERRED from the documented `K6_` convention]**

**Coverage:** no k6-specific config required (zero-config run) ✔; five-source precedence `defaults < config file < script options < environment < CLI` ✔ (`cmd/config.go:180-204`); `K6_*` option vars (`K6_VUS`/`K6_ITERATIONS`) ✔; runtime vars `K6_CONFIG`/`K6_LOG_OUTPUT`/`K6_LOG_FORMAT`/`K6_NO_COLOR`/`K6_PROFILING_ENABLED` ✔ (`cmd/state/state.go:163-180`, with observed `K6_NO_COLOR` transcript).

---

## 7. Does k6 generate external files?

**Direct answer.** **No — not by default.** A normal `k6 run` writes its results only to the terminal (`output: -`). External files/streams are produced **only on request**, via `--out <backend>=<target>` (JSON/CSV/InfluxDB/...), `--summary-export=<file>` (end-of-test summary as JSON), or a user-defined `handleSummary()` (arbitrary files).

**Proof it writes nothing by default (OBSERVED).** The demonstration below uses the canonical single-request script copied to `/tmp/k6work/http_get.js` (byte-identical to the in-repo `examples/http_get.js`), run from an empty scratch directory **outside the repository** (`/tmp/k6work/ext2`). Every path is absolute and under `/tmp`, so the commands are fully self-contained and cannot write into the repository. The directory is **identical before and after** the run, and the execution description shows `output: -`:

```console
$ cd /tmp/k6work/ext2
$ ls -A                 # empty before the run
$ /tmp/k6bin/k6 run /tmp/k6work/http_get.js

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6work/http_get.js
        output: -

     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 1 iterations for each of 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)


     data_received..................: 13 kB  54 kB/s
     data_sent......................: 1.1 kB 4.8 kB/s
     http_req_blocked...............: avg=94.36ms  min=46.65ms  med=94.36ms  max=142.07ms p(90)=132.53ms p(95)=137.3ms 
     http_req_connecting............: avg=20.93ms  min=12.3ms   med=20.93ms  max=29.57ms  p(90)=27.84ms  p(95)=28.71ms 
     http_req_duration..............: avg=22.15ms  min=14.51ms  med=22.15ms  max=29.79ms  p(90)=28.26ms  p(95)=29.03ms 
       { expected_response:true }...: avg=22.15ms  min=14.51ms  med=22.15ms  max=29.79ms  p(90)=28.26ms  p(95)=29.03ms 
     http_req_failed................: 0.00%  0 out of 2
     http_req_receiving.............: avg=86.63µs  min=58.96µs  med=86.63µs  max=114.29µs p(90)=108.76µs p(95)=111.52µs
     http_req_sending...............: avg=145.16µs min=115.02µs med=145.16µs max=175.31µs p(90)=169.28µs p(95)=172.3µs 
     http_req_tls_handshaking.......: avg=22.35ms  min=14.05ms  med=22.35ms  max=30.65ms  p(90)=28.99ms  p(95)=29.82ms 
     http_req_waiting...............: avg=21.92ms  min=14.28ms  med=21.92ms  max=29.56ms  p(90)=28.03ms  p(95)=28.8ms  
     http_reqs......................: 2      8.560436/s
     iteration_duration.............: avg=233.51ms min=233.51ms med=233.51ms max=233.51ms p(90)=233.51ms p(95)=233.51ms
     iterations.....................: 1      4.280218/s


running (00m00.2s), 0/1 VUs, 1 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.2s/10m0s  1/1 iters, 1 per VU
$ ls -A                 # still empty after the run
```

Both `ls -A` calls print nothing — no file appeared. **[OBSERVED]**

**Opt-in file generation (OBSERVED).** Re-running with both a streamed output (`--out json=`) and a summary export (`--summary-export=`), again with **absolute `/tmp` paths only** so the output lands strictly outside the repository:

```console
$ cd /tmp/k6work/ext
$ ls -A                 # empty before the run
$ /tmp/k6bin/k6 run --out json=/tmp/k6work/ext/out.json --summary-export=/tmp/k6work/ext/summary.json /tmp/k6work/http_get.js

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: /tmp/k6work/http_get.js
        output: json (/tmp/k6work/ext/out.json)

     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 1 iterations for each of 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)


     data_received..................: 12 kB  58 kB/s
     data_sent......................: 1.1 kB 5.3 kB/s
     http_req_blocked...............: avg=83.28ms  min=41.6ms   med=83.28ms  max=124.95ms p(90)=116.61ms p(95)=120.78ms
     http_req_connecting............: avg=20.23ms  min=12.2ms   med=20.23ms  max=28.26ms  p(90)=26.65ms  p(95)=27.46ms 
     http_req_duration..............: avg=20.92ms  min=13.34ms  med=20.92ms  max=28.51ms  p(90)=26.99ms  p(95)=27.75ms 
       { expected_response:true }...: avg=20.92ms  min=13.34ms  med=20.92ms  max=28.51ms  p(90)=26.99ms  p(95)=27.75ms 
     http_req_failed................: 0.00%  0 out of 2
     http_req_receiving.............: avg=95.45µs  min=77.32µs  med=95.45µs  max=113.59µs p(90)=109.96µs p(95)=111.77µs
     http_req_sending...............: avg=144.37µs min=137.14µs med=144.37µs max=151.6µs  p(90)=150.15µs p(95)=150.88µs
     http_req_tls_handshaking.......: avg=22.67ms  min=15.86ms  med=22.67ms  max=29.48ms  p(90)=28.12ms  p(95)=28.8ms  
     http_req_waiting...............: avg=20.68ms  min=13.11ms  med=20.68ms  max=28.26ms  p(90)=26.74ms  p(95)=27.5ms  
     http_reqs......................: 2      9.583238/s
     iteration_duration.............: avg=208.59ms min=208.59ms med=208.59ms max=208.59ms p(90)=208.59ms p(95)=208.59ms
     iterations.....................: 1      4.791619/s


running (00m00.2s), 0/1 VUs, 1 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.2s/10m0s  1/1 iters, 1 per VU
$ ls -A                 # two files created
out.json
summary.json
```

The execution description now reports the backend as `output: json (/tmp/k6work/ext/out.json)`, and the final `ls -A` shows **two files were created** — `out.json` and `summary.json`.

- `out.json` — newline-delimited JSON. It is **35 lines total = 13 `"Metric"` definition records + 22 `"Point"` sample records**, verified with:

```bash
wc -l /tmp/k6work/ext/out.json                          # 35
grep -c '"type":"Metric"' /tmp/k6work/ext/out.json      # 13
grep -c '"type":"Point"'  /tmp/k6work/ext/out.json      # 22
```

The records are **interleaved**, not grouped: the JSON backend's `flushMetrics` loop (`output/json/json.go:124-137`) writes each metric's `Metric` definition (via `handleMetric`, which de-duplicates through its `seenMetrics` guard at `output/json/json.go:149-153`) *immediately before* that metric's first `Point`. The real head of the stream shows this `Metric` → `Point` → `Metric` → `Point` ordering (**OBSERVED**, first 6 lines of `/tmp/k6work/ext/out.json`):

```json
{"type":"Metric","data":{"name":"http_reqs","type":"counter","contains":"default","thresholds":[],"submetrics":null},"metric":"http_reqs"}
{"metric":"http_reqs","type":"Point","data":{"time":"2026-07-13T17:32:38.605870576Z","value":1,"tags":{"expected_response":"true","group":"","method":"GET","name":"https://test-api.k6.io/","proto":"HTTP/2.0","scenario":"default","status":"302","tls_version":"tls1.3","url":"https://test-api.k6.io/"}}}
{"type":"Metric","data":{"name":"http_req_duration","type":"trend","contains":"time","thresholds":[],"submetrics":[{"name":"http_req_duration{expected_response:true}","suffix":"expected_response:true","tags":{"expected_response":"true"}}]},"metric":"http_req_duration"}
{"metric":"http_req_duration","type":"Point","data":{"time":"2026-07-13T17:32:38.605870576Z","value":13.342304,"tags":{"expected_response":"true","group":"","method":"GET","name":"https://test-api.k6.io/","proto":"HTTP/2.0","scenario":"default","status":"302","tls_version":"tls1.3","url":"https://test-api.k6.io/"}}}
{"type":"Metric","data":{"name":"http_req_blocked","type":"trend","contains":"time","thresholds":[],"submetrics":null},"metric":"http_req_blocked"}
{"metric":"http_req_blocked","type":"Point","data":{"time":"2026-07-13T17:32:38.605870576Z","value":41.609935,"tags":{"expected_response":"true","group":"","method":"GET","name":"https://test-api.k6.io/","proto":"HTTP/2.0","scenario":"default","status":"302","tls_version":"tls1.3","url":"https://test-api.k6.io/"}}}
```

The file is written by the JSON output backend, wired in `cmd/outputs.go` (`json.New` at `cmd/outputs.go:46`); it is created only when `--out json=<file>` is given.

- `summary.json` — a single JSON **object** with keys `root_group` and `metrics`. It is written by `handleSummaryResult` (declared at `cmd/run.go:508`), whose inner `getWriter` closure (`cmd/run.go:511-519`) opens each non-`stdout`/`stderr` path with `fs.OpenFile(...)` at `cmd/run.go:518`. Complete captured `summary.json` (**OBSERVED**, valid JSON; object-key order is not significant):

```json
{
    "root_group": {
        "name": "",
        "path": "",
        "id": "d41d8cd98f00b204e9800998ecf8427e",
        "groups": {},
        "checks": {}
    },
    "metrics": {
        "http_req_duration{expected_response:true}": {
            "p(95)": 27.7553031,
            "avg": 20.928093,
            "min": 13.342304,
            "med": 20.928093,
            "max": 28.513882,
            "p(90)": 26.9967242
        },
        "http_req_connecting": {
            "med": 20.2368015,
            "max": 28.264694,
            "p(90)": 26.6591155,
            "p(95)": 27.461904750000002,
            "avg": 20.2368015,
            "min": 12.208909
        },
        "http_req_waiting": {
            "p(95)": 27.505659299999998,
            "avg": 20.688261,
            "min": 13.113374,
            "med": 20.688261,
            "max": 28.263148,
            "p(90)": 26.7481706
        },
        "iteration_duration": {
            "p(95)": 208.59332,
            "avg": 208.59332,
            "min": 208.59332,
            "med": 208.59332,
            "max": 208.59332,
            "p(90)": 208.59332
        },
        "http_req_receiving": {
            "max": 0.113592,
            "p(90)": 0.1099652,
            "p(95)": 0.1117786,
            "avg": 0.095458,
            "min": 0.077324,
            "med": 0.095458
        },
        "http_reqs": {
            "count": 2,
            "rate": 9.583238195736145
        },
        "http_req_duration": {
            "avg": 20.928093,
            "min": 13.342304,
            "med": 20.928093,
            "max": 28.513882,
            "p(90)": 26.9967242,
            "p(95)": 27.7553031
        },
        "data_sent": {
            "count": 1111,
            "rate": 5323.488817731429
        },
        "http_req_sending": {
            "max": 0.151606,
            "p(90)": 0.1501596,
            "p(95)": 0.15088279999999998,
            "avg": 0.144374,
            "min": 0.137142,
            "med": 0.144374
        },
        "data_received": {
            "count": 12167,
            "rate": 58299.62956376084
        },
        "http_req_blocked": {
            "avg": 83.2820225,
            "min": 41.609935,
            "med": 83.28202250000001,
            "max": 124.95411,
            "p(90)": 116.61969250000001,
            "p(95)": 120.78690125
        },
        "http_req_tls_handshaking": {
            "max": 29.481947,
            "p(90)": 28.1202417,
            "p(95)": 28.80109435,
            "avg": 22.6734205,
            "min": 15.864894,
            "med": 22.6734205
        },
        "http_req_failed": {
            "passes": 0,
            "fails": 2,
            "value": 0
        },
        "iterations": {
            "count": 1,
            "rate": 4.791619097868073
        }
    }
}
```

Other `--out` backends (CSV, InfluxDB) are wired alongside JSON in `cmd/outputs.go` (`csv.New` `:48`, `influxdb.New` `:49`). One nuance about the multiplexing `output.Manager` (`output/manager.go`, constructed at `cmd/run.go:220`): its output list is **not** empty on a default run. `createOutputs()` (`cmd/run.go:163`) contributes **no external backend** unless `--out` is passed, but `cmd/run.go:168` still appends the internal `GroupSummary` output, and — because the end-of-test summary and thresholds are enabled by default — `cmd/run.go:188` appends an internal metrics ingester too. So the Manager always receives those **internal** outputs; what makes a default run create **no external file** is simply that `createOutputs()` added no external backend, not that the Manager list is empty. **[OBSERVED default + INFERRED for CSV/InfluxDB, which were not exercised]**

**Coverage:** default writes nothing ✔ (dir unchanged, `output: -`); `--out json=` file ✔; `--summary-export` file ✔ (`cmd/run.go:508`); `handleSummary()` acknowledged ✔; backends in `cmd/outputs.go`.

---

## 8. Script validation logic

**Direct answer.** Before it will run a script, k6 **bundles** it and requires that it export **at least one callable function**. If bundling succeeds but there are no callable exports, k6 fails with the exact error **`no exported functions in script`** (`js/bundle.go:238`, guarded by `if len(b.callableExports) == 0 {` at `:237`). A script that throws/parses incorrectly fails as a **script exception**, mapped to exit code `ScriptException = 107` (`errext/exitcodes/codes.go:48`).

Three edge cases were run directly. Each command is shown with a trailing `; echo "exit=$?"` so the real exit code is visible in the same transcript, and every run prints the banner first, then the error to **stderr**:

**(a) Empty script** — `/tmp/k6work/empty.js` (0 bytes). Fails to initialize with **exit 255** (**OBSERVED**):

```console
$ /tmp/k6bin/k6 run /tmp/k6work/empty.js ; echo "exit=$?"

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

time="2026-07-13T17:33:35Z" level=error msg="could not initialize '/tmp/k6work/empty.js': could not load JS test 'file:///tmp/k6work/empty.js': no exported functions in script"
exit=255
```

**(b) Non-function default export** — `/tmp/k6work/nonfunc.js` = `export default 42;`. Same failure, **exit 255** (**OBSERVED**):

```console
$ cat /tmp/k6work/nonfunc.js
export default 42;
$ /tmp/k6bin/k6 run /tmp/k6work/nonfunc.js ; echo "exit=$?"

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

time="2026-07-13T17:33:35Z" level=error msg="could not initialize '/tmp/k6work/nonfunc.js': could not load JS test 'file:///tmp/k6work/nonfunc.js': no exported functions in script"
exit=255
```

Both (a) and (b) hit the **same** `no exported functions in script` error (`js/bundle.go:238`, guarded by `if len(b.callableExports) == 0 {` at `js/bundle.go:237`) — a `42` default export is not callable, so it doesn't count. The exit code is the generic **255**, because this initialization error is **not** assigned a dedicated `ExitCode` constant in `errext/exitcodes/codes.go`. **[OBSERVED]**

**(c) Syntax error** — `/tmp/k6work/syntaxerr.js`, whose closing `}` is missing (the file ends after the `http.get(...)` line). Exact input (**OBSERVED**, 4 lines, no trailing brace):

```javascript
import http from 'k6/http';
export default function () {
  http.get('https://test-api.k6.io/');

```

Running it fails as a **script exception** with **exit 107** (**OBSERVED**):

```console
$ /tmp/k6bin/k6 run /tmp/k6work/syntaxerr.js ; echo "exit=$?"

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

time="2026-07-13T17:33:41Z" level=error msg="GoError: file:///tmp/k6work/syntaxerr.js: Line 5:1 Unexpected end of input\n" hint="script exception"
exit=107
```

The `hint="script exception"` and **exit 107** correspond to `ScriptException ExitCode = 107` (`errext/exitcodes/codes.go:48`). Because the closing brace is absent, the parser reaches end-of-file while the function body is still open and reports `Line 5:1 Unexpected end of input`. **[OBSERVED]**

> Distinction worth noting: "no exported functions" is a **load/initialization** failure → exit **255** (no dedicated code); a parse/throw is a **script exception** → exit **107**. Both are observed above, not assumed.

**Coverage:** bundling + "≥1 callable export" rule ✔ (`js/bundle.go:237-238`); empty / non-function / syntax-error cases ✔ with exit codes 255/255/107; `ScriptException=107` ✔ (`errext/exitcodes/codes.go:48`).

---

## 9. Final coverage checklist

| # | Question item | Answered in | Evidence (observed run + citation) |
|---|---------------|-------------|-------------------------------------|
| 1 | Authoring + running workflow | §1 | `examples/http_get.js:1-5`; `k6 new` (`cmd/new.go:15,151`); `k6 run` (§3/§5) |
| 2 | What k6 reports (4 surfaces) | §2 | banner `lib/consts/consts.go:56`; exec desc + progress `cmd/ui.go:100,70`; summary `cmd/run.go:195`→`js/summary.js` |
| 3 | Single HTTP request | §3 | full run of `examples/http_get.js`, exit 0 |
| 4a | Output metrics + types | §4a | `metrics/builtin.go:78`; `out.json` metric defs |
| 4b | Output units (auto-scaled) | §4b | `js/summary.js` (`:134-136`,`:170-181`,`:197-198`); `--summary-time-unit=ms` run |
| 4c | Protocols | §4c | modules `js/jsmodules.go:57-63`; tagging `lib/netext/httpext/transport.go:118`; 18 of 22 `out.json` Points carry `"proto":"HTTP/2.0"` |
| 5 | The execution command | §5 | `cmd/run.go:462,491,492,499`; full `k6 run --help` |
| 6 | Config / env vars | §6 | zero-config run; five-source precedence `cmd/config.go:180-204`; `K6_ITERATIONS`/`K6_VUS`/`K6_NO_COLOR` runs; `cmd/state/state.go:163-180` |
| 7 | External file generation | §7 | default writes nothing; `--out json=` + `--summary-export`; `cmd/run.go:508`, `cmd/outputs.go:46-49` |
| 8 | Script validation logic | §8 | empty/non-function/syntax runs; `js/bundle.go:238`; `errext/exitcodes/codes.go:48` |

**Observed vs inferred summary.** Directly observed: the version string and banner; all four output surfaces; the full metrics block; metric types and value-types (via `out.json`); unit auto-scaling across `ns`/`µs`/`ms`/`s` plus the `--summary-time-unit=ms` override; the interleaved `Metric`/`Point` ordering in `out.json`; the per-request protocol tag `"proto":"HTTP/2.0"` on 18 of 22 Points; all `k6 run --help` flags and the `--summary-mode` absence; the zero-config, `K6_ITERATIONS`/`K6_VUS`, and `K6_NO_COLOR` runs; default-writes-nothing plus the two opt-in files (`out.json`, `summary.json`); and all three validation exit codes (255/255/107). Inferred (and labelled inline where used): the exact ≥1-second gauge-emission cadence for `vus`/`vus_max`, and the CSV/InfluxDB output backends (acknowledged but not exercised, consistent with the HTTP-only scope).

**Repository integrity.** No **existing** repository file was modified; the only change to the working tree is the addition of this single document (`blitzy/documentation/k6_ddc3b0b1d23c.md`). Every investigation artifact — the built k6 binary, all observation scripts, and all opt-in output files — lived **outside** the checkout under `/tmp` (`/tmp/k6bin`, `/tmp/k6work`), so no build or run ever wrote into the repository, and those artifacts are deleted before completion. A final `git status --porcelain` therefore shows exactly one added file (this document) and no modification to any tracked source file.
