# How the k6 load-testing tool behaves — an evidence-backed guide (k6 v0.55.0 @ ddc3b0b1d2)

This document answers nine questions about how the [Grafana k6](https://github.com/grafana/k6) load-testing tool behaves. **k6 is a single, self-contained command-line binary that executes JavaScript test scripts on an embedded Go JavaScript virtual machine (Sobek, a goja fork).** Every behavioural claim below is backed by (a) the *real, unedited* output of a k6 binary built from source at commit `ddc3b0b1d2`, and (b) a `file:line` citation into that same source tree. Statements that are reasoned from reading the code rather than directly observed in output are explicitly marked **(inferred)**.

> **Version caveat.** This build is **k6 v0.55.0**, a *pre-v1.0.0* release. Its canonical behaviour is the **legacy end-of-test summary format** and an **in-process REST API that is ON by default** at `localhost:6565`. Newer k6 (v1.x / v2.x) changes some of this (for example the REST API is off by default and a machine-readable summary / `--summary-mode` exists). Those newer features are **not** attributed to this build — everything reported here is v0.55.0's actual behaviour.

> **Read-only investigation.** Per the governing rule, the tool was built and run *first*; the answers are derived from captured output. All test scripts, generated files, and the built binary were created **outside** the repository and deleted afterwards, leaving the repository byte-for-byte unchanged (only this document was added).

## How this evidence was produced

k6 compiles to one binary via a plain `go build` (the `Makefile` `build` target). It was built from source with the repository's vendored modules, with the output redirected **outside** the repository to honour the read-only constraint:

```bash
export PATH=$PATH:/usr/local/go/bin
export GOFLAGS=-mod=vendor
export GOCACHE=/tmp/gocache
go build -o /tmp/k6bin/k6 .
```

The toolchain is **Go 1.21.13**, matching the `go.mod` directives `go 1.21` and `toolchain go1.21.13` (`go.mod:3`, `go.mod:5`). In this containerised environment that pinned toolchain was selected explicitly (`GOTOOLCHAIN=go1.21.13`); the resulting binary's version banner is shown below and is used, verbatim, throughout this document.

Every run enters the real CLI entry point: `main.go` calls `cmd.Execute()` (`main.go:8-9`), which dispatches to the `k6 run` Cobra command built by `getCmdRun` (`cmd/run.go:462`):

```go
func main() {
	cmd.Execute()
}
```

Version banner (`k6 version`):

```text
k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)
```

**Environment.** Outbound HTTPS is available and `https://test.k6.io` is reachable. It responds with an **HTTP 302 redirect** which k6 follows by default (`--max-redirects` default `10`, `cmd/options.go:36`); this is why one `http.get(...)` produces `http_reqs=2` (see Q4/Q5).

## Q1 — How k6 behaves as a tool

k6 is a **single CLI binary** driven by subcommands (built on the Cobra framework). `main.go` is trivial — it delegates to `cmd.Execute()` (`main.go:8-9`, shown above) — and the load test is run by the `run` subcommand (`getCmdRun`, `cmd/run.go:462`). Test logic is written in JavaScript and executed on an embedded Go JS VM; the built binary needs no external runtime.

The `run` command's own long help states that it also exposes a REST API (`cmd/run.go:493-496`):

```go
		Long: `Start a test.

This also exposes a REST API to interact with it. Various k6 subcommands offer
a commandline interface for interacting with it.`,
```

**The REST API is ON by default** at `localhost:6565` for this v0.55.0 build. The default address comes from `GetDefaultFlags` (`cmd/state/state.go:150`, `Address: "localhost:6565"`) and the server is constructed by `api.GetServer` (`api/server.go:50-70`). Probing it live *while a test was running* (a background `k6 run -d 15s loop.js`, queried from a small Go client) returned:

```bash
$ curl http://localhost:6565/v1/status
{"data":{"type":"status","id":"default","attributes":{"status":7,"paused":false,"vus":1,"vus-max":1,"stopped":false,"running":true,"tainted":false}}}
```

(The value above is the real JSON body returned by the endpoint; `curl` is not installed in the test container, so it was fetched with an equivalent Go `http.Get`.) The companion endpoint `http://localhost:6565/v1/metrics` returned **16** live metric objects during the same run (observed). **Version caveat:** newer k6 (v2.x) turns this REST API off by default; this v0.55.0 build has it **on**.

## Q2 — Basic workflow of writing and running a script

The workflow has three steps: **(1)** author a `.js` file that `export default`s a function — this is the per-VU, per-iteration entry point k6 calls; **(2)** run `k6 run <script>`; **(3)** observe the live progress and the end-of-test summary. The repository's own canonical minimal script is `examples/http_get.js` (`examples/http_get.js:1-5`):

```javascript
import http from 'k6/http';

export default function () {
  http.get('https://test-api.k6.io/');
};
```

A fuller canonical pattern, with `options`, a `check`, and `sleep`, appears in the project README (`README.md:59-82`):

```javascript
import http from "k6/http";
import { check, sleep } from "k6";

// Test configuration
export const options = {
  thresholds: {
    // Assert that 99% of requests finish within 3000ms.
    http_req_duration: ["p(99) < 3000"],
  },
  // Ramp the number of virtual users up and down
  stages: [
    { duration: "30s", target: 15 },
    { duration: "1m", target: 15 },
    { duration: "20s", target: 0 },
  ],
};

// Simulated user behavior
export default function () {
  let res = http.get("https://test-api.k6.io/public/crocodiles/1/");
  // Validate response status
  check(res, { "status was 200": (r) => r.status == 200 });
  sleep(1);
}
```

Here the `default` export is the required per-iteration entry point; the optional `options` object configures thresholds/stages; `check(...)` records an assertion (surfaced as the `checks` metric); and `sleep(...)` paces each iteration.

## Q3 — What k6 reports when the script executes

During a run k6 prints a **live progress indicator**, and when the test finishes it prints an **aggregated end-of-test summary** to stdout. The full stdout of a run has this structure (observed): (1) an ASCII Grafana/k6 banner; (2) an execution block — `execution: local`, `script: <name>`, `output: -`; (3) a `scenarios:` line describing the execution plan; (4) a live progress bar; (5) `check` results; (6) the end-of-test summary of metrics; and (7) a final progress line. The complete, unedited output is shown under **Q4** below (it is not duplicated here). The end-of-test summary itself is produced by the bundled `js/summary.js` and written to stdout; its humanization logic (bytes, durations, rates) lives at `js/summary.js:134-231`.

## Q4 — Testing a single HTTP request (primary demonstration)

To understand k6's behaviour on a single HTTP request, the following temporary script was used (built on `examples/http_get.js` plus the README's `check`/`sleep` pattern). Its `default` export performs exactly one `http.get(...)`:

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
export default function () {
  const res = http.get('https://test.k6.io');
  check(res, { 'status is 200': (r) => r.status === 200 });
  sleep(1);
}
```

It was executed with the exact command (see Q6):

```bash
k6 run single_request.js
```

The **complete, unedited stdout** (run #1, exit code `0`) — banner glyphs, `µs` micro-signs, trailing spaces, and dotted-leader alignment preserved exactly:

```text
         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: single_request.js
        output: -

     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 1 iterations for each of 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)


running (00m01.0s), 1/1 VUs, 0 complete and 0 interrupted iterations
default   [   0% ] 1 VUs  00m01.0s/10m0s  0/1 iters, 1 per VU

     ✓ status is 200

     checks.........................: 100.00% 1 out of 1
     data_received..................: 12 kB   10 kB/s
     data_sent......................: 1.1 kB  916 B/s
     http_req_blocked...............: avg=82.36ms  min=60.48ms  med=82.36ms  max=104.23ms p(90)=99.86ms  p(95)=102.04ms
     http_req_connecting............: avg=17.03ms  min=12.44ms  med=17.03ms  max=21.61ms  p(90)=20.69ms  p(95)=21.15ms 
     http_req_duration..............: avg=17.27ms  min=13.08ms  med=17.27ms  max=21.45ms  p(90)=20.62ms  p(95)=21.04ms 
       { expected_response:true }...: avg=17.27ms  min=13.08ms  med=17.27ms  max=21.45ms  p(90)=20.62ms  p(95)=21.04ms 
     http_req_failed................: 0.00%   0 out of 2
     http_req_receiving.............: avg=105.03µs min=70.72µs  med=105.03µs max=139.33µs p(90)=132.47µs p(95)=135.9µs 
     http_req_sending...............: avg=161.28µs min=117.01µs med=161.28µs max=205.55µs p(90)=196.7µs  p(95)=201.12µs
     http_req_tls_handshaking.......: avg=30.2ms   min=16.52ms  med=30.2ms   max=43.88ms  p(90)=41.15ms  p(95)=42.51ms 
     http_req_waiting...............: avg=17ms     min=12.8ms   med=17ms     max=21.2ms   p(90)=20.36ms  p(95)=20.78ms 
     http_reqs......................: 2       1.666025/s
     iteration_duration.............: avg=1.2s     min=1.2s     med=1.2s     max=1.2s     p(90)=1.2s     p(95)=1.2s    
     iterations.....................: 1       0.833012/s
     vus............................: 1       min=1      max=1
     vus_max........................: 1       min=1      max=1


running (00m01.2s), 0/1 VUs, 1 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m01.2s/10m0s  1/1 iters, 1 per VU
```

**Stability across runs.** A second, identical run produced the **same metric set and the same counts** — `checks` `1 out of 1`, `http_req_failed` `0 out of 2`, `http_reqs=2`, `iterations=1`, `vus`/`vus_max=1` — with only latency magnitudes and byte totals differing slightly (run #2 reported `data_received 13 kB` and `http_req_duration avg=17.73ms`, versus run #1's `data_received 12 kB` / `avg=17.27ms`). The structural result is therefore stable; only live network timing varies, as expected.

**Why `http_reqs=2` for a single `http.get`.** `test.k6.io` answers with an HTTP `302` and k6 follows the redirect (`--max-redirects` default `10`, `cmd/options.go:36`), so one `http.get(...)` results in two HTTP requests. This is corroborated by the JSON output, where the sample tags include `status:"302"` (see Q5). It is expected behaviour, not an error — `http_req_failed` is `0 out of 2`.

## Q5 — Output shape: metrics, units, and protocols

### Metrics (16 emitted for this script)

The single-request run emitted **16 metrics** (the same 16 the REST API reported in Q1). Every built-in metric is declared and registered in `metrics/builtin.go`; each has a **metric type** — `Counter` / `Gauge` / `Trend` / `Rate` (`metrics/metric_type.go:9-14`) — and a **value type** — `Default` / `Time` (milliseconds) / `Data` (bytes) (`metrics/value_type.go:6-10`). The type/value registration happens in `RegisterBuiltinMetrics` (`metrics/builtin.go:78-111`). The "rendered" column below is quoted from the Q4 run #1 summary (trend rows show only `avg=` for brevity; the full stats are in the Q4 block):

| Metric | Metric type | Value type | Rendered (Q4 run #1) | Source: name / registration |
|---|---|---|---|---|
| `checks` | Rate | Default (rendered `%`) | `100.00% 1 out of 1` | `metrics/builtin.go:12` / `:86` |
| `data_received` | Counter | Data (bytes) | `12 kB   10 kB/s` | `metrics/builtin.go:35` / `:109` |
| `data_sent` | Counter | Data (bytes) | `1.1 kB  916 B/s` | `metrics/builtin.go:34` / `:108` |
| `http_req_blocked` | Trend | Time (ms) | `avg=82.36ms` | `metrics/builtin.go:18` / `:92` |
| `http_req_connecting` | Trend | Time (ms) | `avg=17.03ms` | `metrics/builtin.go:19` / `:93` |
| `http_req_duration` | Trend | Time (ms) | `avg=17.27ms` | `metrics/builtin.go:17` / `:91` |
| `http_req_receiving` | Trend | Time (ms) | `avg=105.03µs` | `metrics/builtin.go:23` / `:97` |
| `http_req_sending` | Trend | Time (ms) | `avg=161.28µs` | `metrics/builtin.go:21` / `:95` |
| `http_req_tls_handshaking` | Trend | Time (ms) | `avg=30.2ms` | `metrics/builtin.go:20` / `:94` |
| `http_req_waiting` | Trend | Time (ms) | `avg=17ms` | `metrics/builtin.go:22` / `:96` |
| `http_req_failed` | Rate | Default (rendered `%`) | `0.00%   0 out of 2` | `metrics/builtin.go:16` / `:90` |
| `http_reqs` | Counter | Default | `2       1.666025/s` | `metrics/builtin.go:15` / `:89` |
| `iteration_duration` | Trend | Time (ms) | `avg=1.2s` | `metrics/builtin.go:9` / `:83` |
| `iterations` | Counter | Default | `1       0.833012/s` | `metrics/builtin.go:8` / `:82` |
| `vus` | Gauge | Default | `1       min=1      max=1` | `metrics/builtin.go:6` / `:80` |
| `vus_max` | Gauge | Default | `1       min=1      max=1` | `metrics/builtin.go:7` / `:81` |

Notes: `http_req_duration` additionally shows a `{ expected_response:true }` **submetric** line (visible in the Q4 block) — it is the same Trend split by the `expected_response` tag, not a separate metric. The `checks` metric is a `Rate` registered at `metrics/builtin.go:86`; `http_req_failed` is a `Rate` at `:90`. This accounts for all 16 emitted metrics.

### Units

The internal **time base is the millisecond** (`metrics/units.go:7`):

```go
const timeUnit = time.Millisecond
```

Values are *humanized* for the summary by `js/summary.js`:

- **Bytes** render as `B` / `kB` / `MB` / … using **base 1000** — `humanizeBytes` (`js/summary.js:134-145`); e.g. `data_received` rendered `12 kB   10 kB/s`.
- **Durations** render adaptively as `ns` / `µs` / `ms` / `s` — `humanizeGenericDuration` (`js/summary.js:163-194`); note the `µs` micro-sign in e.g. `http_req_receiving` above.
- **Rates** render as a **percentage** — `humanizeValue` (`js/summary.js:204-218`); e.g. `checks` rendered `100.00% 1 out of 1` and `http_req_failed` rendered `0.00%   0 out of 2`.

Importantly, the rate branch **truncates rather than rounds** to two decimals. Quoted verbatim (`js/summary.js:205-208`; the operative line is `js/summary.js:207`):

```javascript
  if (metric.type == 'rate') {
    // Truncate instead of round when decreasing precision to 2 decimal places
    return (Math.trunc(val * 100 * 100) / 100).toFixed(2) + '%'
  }
```

Counters render as `count  rate/s` and gauges as `value  min=..  max=..` — `nonTrendMetricValueForSum` (`js/summary.js:220-232`); e.g. `http_reqs` → `2       1.666025/s`, `iterations` → `1       0.833012/s`, `vus` → `1       min=1      max=1`. The `--summary-time-unit` flag maps units via `unitMap` (`js/summary.js:147-151`):

```javascript
var unitMap = {
  s: { unit: 's', coef: 0.001 },
  ms: { unit: 'ms', coef: 1 },
  us: { unit: 'µs', coef: 1000 },
}
```

The default trend statistics shown (`avg, min, med, max, p(90), p(95)`) come from `--summary-trend-stats` (default defined at `cmd/options.go:58`; the resolved default `avg,min,med,max,p(90),p(95)` is visible in `k6 run --help`). `--summary-time-unit` accepts `s` / `ms` / `us` (`cmd/options.go:59`).

### Protocols

The single HTTPS request was served over **HTTP/2 with TLS 1.3** (observed). In the `--out json` output every HTTP metric sample is tagged; of the 25 `Point` samples, the **18** HTTP-family points all carry:

```text
"proto":"HTTP/2.0"      (all 18 HTTP samples)
"tls_version":"tls1.3"  (all 18 HTTP samples)
```

The full tag set on an `http_req_duration` `Point` sample (observed, verbatim) — note `status:"302"`, which confirms the followed redirect behind `http_reqs=2`:

```json
{"expected_response":"true","group":"","method":"GET","name":"https://test.k6.io","proto":"HTTP/2.0","scenario":"default","status":"302","tls_version":"tls1.3","url":"https://test.k6.io"}
```

The built-in metric families also cover **WebSocket** (`ws_*`, `metrics/builtin.go:25-30`) and **gRPC** (`grpc_req_duration`, `metrics/builtin.go:32`), but for this `http.get`-only script **only the HTTP family was emitted** (observed). The default system tags include `proto` and `tls_version`, confirmed in `k6 run --help`:

```text
--system-tags (default): "proto,subproto,status,method,url,name,group,check,error,error_code,tls_version,scenario,service,expected_response"
```

## Q6 — The exact command used to execute the script

The invocation is **`k6 run <script.js>`**. The subcommand is declared with `Use: "run"` (`cmd/run.go:491`) and `Short: "Start a test"` (`cmd/run.go:492`). It requires **exactly one** argument — a path to a script file, or `-` to read the script from stdin (`cmd/run.go:498`, `arg should either be "-", if reading script from stdin, or a path to a script file`).

Key flags (each cited to its definition; all also visible in `k6 run --help`):

- `-u, --vus` — number of virtual users, **default `1`** (`cmd/options.go:27`)
- `-i, --iterations` — total iteration limit among all VUs (`cmd/options.go:29`)
- `-d, --duration` — test duration limit (`cmd/options.go:28`)
- `-s, --stage [duration]:[target]` — add a ramping stage (`cmd/options.go:30`)
- `-o, --out uri` — send metrics to an external output (`cmd/config.go:31`)
- `-e, --env VAR=value` — add/override an environment variable (`cmd/runtime_options.go:32`)
- `--summary-export <file>` — write the end-of-test summary to a JSON file (`cmd/runtime_options.go:35-38`); `--no-summary` suppresses the summary (`cmd/runtime_options.go:34`)
- `--summary-trend-stats` — trend stats to show, default `avg,min,med,max,p(90),p(95)` (`cmd/options.go:58`); `--summary-time-unit` — `s`/`ms`/`us` (`cmd/options.go:59`)
- `-a, --address` — REST API address, **default `localhost:6565`** (`cmd/state/state.go:150`); `-q, --quiet` — disable progress updates

The command's built-in examples, printed verbatim by `k6 run --help` (the templates at `cmd/run.go:471-488` render `{{.}}` as `k6`):

```text
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
```

## Q7 — Configuration and environment variables

**No configuration is required for a basic run.** The defaults are 1 VU / 1 iteration, and the default config-file path (`homeDir/loadimpact/k6/config.json`, `cmd/state/state.go:152`) need not exist — every run in this document used no config file. There are, however, **two distinct environment-variable mechanisms**, and their difference is worth stating precisely:

**(a) `-e VAR=value` injects into the script's `__ENV`.** The flag is `-e, --env` (`cmd/runtime_options.go:32`). Setting `-e MY_TARGET=...` makes `__ENV.MY_TARGET` available in the script (banner elided; decision-relevant lines shown verbatim):

```text
$ k6 run -e MY_TARGET=https://example.com env_script.js
     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 1 iterations for each of 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)
time="2026-07-08T04:36:39Z" level=info msg="MY_TARGET from __ENV = https://example.com" source=console
running (00m00.2s), 0/1 VUs, 1 complete and 0 interrupted iterations
```

**(b) `K6_*` environment variables configure k6 options.** k6 binds options to `K6_*` names via `envconfig` (e.g. `Out ... envconfig:"K6_OUT"`, `cmd/config.go:45`). Passing real env vars `K6_VUS=2 K6_ITERATIONS=4` changes the execution plan to 2 VUs / 4 iterations:

```text
$ K6_VUS=2 K6_ITERATIONS=4 k6 run env_script.js
     scenarios: (100.00%) 1 scenario, 2 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 4 iterations shared among 2 VUs (maxDuration: 10m0s, gracefulStop: 30s)
running (00m00.2s), 0/2 VUs, 4 complete and 0 interrupted iterations
default ✓ [ 100% ] 2 VUs  00m00.2s/10m0s  4/4 shared iters
```

**Important nuance (observed, and contrary to a common simplification).** Because `-e` variables are merged into the same environment k6 consults for `K6_*` options (system env vars are included by default — `--include-system-env-vars` defaults to **true**, `cmd/runtime_options.go:24`), an *option-shaped* `-e` variable **is** read by the option loader. Passing `-e K6_VUS=2` alone makes k6 warn that `vus=2` will be ignored (it needs a companion `iterations`/`duration`/`stages`), and the run stays 1 VU / 1 iteration:

```text
$ k6 run -e K6_VUS=2 env_script.js
time="2026-07-08T04:36:39Z" level=warning msg="the `vus=2` option will be ignored, it only works in conjunction with `iterations`, `duration`, or `stages`"
     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 1 iterations for each of 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)
time="2026-07-08T04:36:39Z" level=info msg="MY_TARGET from __ENV = undefined" source=console
running (00m00.2s), 0/1 VUs, 1 complete and 0 interrupted iterations
```

…but supplying both via `-e` reproduces the real-env result **exactly** — 2 VUs / 4 shared iterations — proving `-e K6_*` does feed option configuration:

```text
$ k6 run -e K6_VUS=2 -e K6_ITERATIONS=4 env_script.js
     scenarios: (100.00%) 1 scenario, 2 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 4 iterations shared among 2 VUs (maxDuration: 10m0s, gracefulStop: 30s)
running (00m00.3s), 0/2 VUs, 4 complete and 0 interrupted iterations
default ✓ [ 100% ] 2 VUs  00m00.3s/10m0s  4/4 shared iters
```

So: `-e` is the mechanism for **script-visible** variables (`__ENV`), while **`K6_*`** (whether set in the real environment or via `-e`) is the mechanism for **k6 options**. The script used for these observations:

```javascript
import http from 'k6/http';
export default function () {
  console.log('MY_TARGET from __ENV = ' + __ENV.MY_TARGET);
  http.get('https://test.k6.io');
}
```

## Q8 — External files the tool generates

**By default k6 writes nothing to disk.** The banner reports `output: -`, and a clean directory was byte-identical before and after a default run (observed — the before/after listings were identical). Files are produced only when explicitly requested:

- **`--out json=out.json`** → banner shows `output: json (out.json)`. The file is newline-delimited JSON of two record kinds: `{"type":"Metric",...}` metadata (the `"Metric"` type string is set at `output/json/json.go:156`) and `{"type":"Point",...}` samples. For the single-request run it contained **16 `Metric` + 25 `Point`** records (~9112 bytes; observed).
- **`--out csv=out.csv`** → banner shows `output: csv (out.csv)` (~2968 bytes, 26 lines). Its header row and first data row, verbatim:

```text
metric_name,timestamp,metric_value,check,error,error_code,expected_response,group,method,name,proto,scenario,service,status,subproto,tls_version,url,extra_tags,metadata
http_reqs,1783485401,1.000000,,,,true,,GET,https://test.k6.io,HTTP/2.0,default,,302,,tls1.3,https://test.k6.io,,
```

- **`--summary-export=summary-export.json`** → this is **separate** from `--out`; the banner still shows `output: -`. The file's top-level keys are `{metrics, root_group}` (~3423 bytes; observed).
- **`handleSummary(data)` export** → its returned object's **keys are destinations**. Returning `{'stdout': '...', 'custom_summary.json': JSON.stringify(data)}` replaced the default stdout summary and wrote `custom_summary.json` (~2451 bytes; top-level keys `{options, state, metrics, root_group}`). Observed:

```text
time="2026-07-08T04:36:44Z" level=info msg="handleSummary called: 2 http_reqs" source=console
custom stdout summary
```

The script used:

```javascript
import http from 'k6/http';
export default function () { http.get('https://test.k6.io'); }
export function handleSummary(data) {
  console.log('handleSummary called: ' + data.metrics.http_reqs.values.count + ' http_reqs');
  return { 'stdout': 'custom stdout summary\n', 'custom_summary.json': JSON.stringify(data) };
}
```

The available `--out` backends are confirmed by the sub-packages under `output/`: **`json`, `csv`, `influxdb`, `cloud`** (`output/json/`, `output/csv/`, `output/influxdb/`, `output/cloud/`). `-o/--out` takes a URI and may be repeated (`cmd/config.go:31`). Reported byte sizes are approximate and vary with timestamps/latency **(inferred)**; the record kinds, CSV header columns, and JSON key sets are structural and were identical across runs (observed).

## Q9 — Script validation logic (with error text and exit codes)

k6 enforces three validation gates before/at execution. Each was exercised with a deliberately invalid input; all three print the ASCII banner and then fail. Exit codes are defined in `errext/exitcodes/codes.go`.

**1. The module must be resolvable.** A non-existent script fails with exit **255** (generic/unmapped). The message is `fileSchemeCouldntBeLoadedMsg` (`loader/loader.go:31-37`), surfaced by `ReadSource` (`loader/readsource.go:15-58`, returned at `loader/readsource.go:53`):

```text
         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

time="2026-07-08T04:31:17Z" level=error msg="The moduleSpecifier \"does_not_exist.js\" couldn't be found on local disk. Make sure that you've specified the right path to the file. If you're running k6 using the Docker image make sure you have mounted the local directory (-v /local/path/:/inside/docker/path) containing your script and modules so that they're accessible by k6 from inside of the container, see https://grafana.com/docs/k6/latest/using-k6/modules/#using-local-modules-with-docker."
```

**2. A callable `default` executor must exist.** A script with no `default` export fails with exit **104** (`InvalidConfig`, `errext/exitcodes/codes.go:36`). The title comes from `consolidateErrorMessage` (`cmd/config.go:271`; title literal at `cmd/config.go:268`) and the detail from `validateScenarioConfig`'s `fmt.Errorf("executor %s: function '%s' not found in exports", ...)` (`cmd/config.go:287`) — note the `executor default:` prefix (executor name = scenario name). A defensive mirror `panic` exists at `js/runner.go:752` but is normally unreachable because config validation fires first (the comment at `js/runner.go:751` reads "Shouldn't happen; this is validated in cmd.validateScenarioConfig()"):

```text
         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

time="2026-07-08T04:31:17Z" level=error msg="There were problems with the specified script configuration:\n\t- executor default: function 'default' not found in exports"
```

**3. The JavaScript must parse.** A syntax error fails with exit **107** (`ScriptException`, `errext/exitcodes/codes.go:48`). The `Line 4:7` position corresponds to the specific malformed content of the test script (line 4 is `  foo bar baz`; column 7 is the unexpected identifier `bar`), so the exact position reflects this input:

```text
         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

time="2026-07-08T04:31:17Z" level=error msg="GoError: file:///out/work/broken.js: Line 4:7 Unexpected identifier\n" hint="script exception"
```

**Validation order (observed):** (1) module resolvable via the loader → else exit **255**; then (2) parseable JavaScript → else exit **107**; then (3) a callable `default` executor → else exit **104**.

The **full exit-code taxonomy** (`errext/exitcodes/codes.go:10-56`) — `InvalidConfig=104`, `ScriptException=107`, and the rest:

```go
const (
	// CloudTestRunFailed indicates that the cloud test run failed.
	// Its value used to be 99 before k6 v0.33.0.
	CloudTestRunFailed ExitCode = 97 // This used to be 99 before k6 v0.33.0

	// CloudFailedToGetProgress indicates that k6 was unable to synchronize the
	// test progress with the cloud.
	CloudFailedToGetProgress ExitCode = 98

	// ThresholdsHaveFailed indicates that one or more thresholds have failed.
	ThresholdsHaveFailed ExitCode = 99

	// SetupTimeout indicates the execution of the test setup function timed out.
	SetupTimeout ExitCode = 100

	// TeardownTimeout indicates the execution of the test teardown function timed out.
	TeardownTimeout ExitCode = 101

	// GenericTimeout indicates a timeout with an unspecified reason.
	GenericTimeout ExitCode = 102 // TODO: remove?

	// ScriptStoppedFromRESTAPI indicates the execution has been
	// stopped by a call to the k6's REST API.
	ScriptStoppedFromRESTAPI ExitCode = 103

	// InvalidConfig indicates an invalid configuration.
	InvalidConfig ExitCode = 104

	// ExternalAbort indicates the test was aborted by an external signal
	// (e.g. SIGINT, SIGTERM, etc.) and should be considered aborted rather
	// than a failure.
	ExternalAbort ExitCode = 105

	// CannotStartRESTAPI indicates the k6's REST API server could not be started.
	CannotStartRESTAPI ExitCode = 106

	// ScriptException indicates an exception was thrown during the
	// test script's execution.
	ScriptException ExitCode = 107

	// ScriptAborted indicates the script was aborted by a call to the
	// k6 execution module's `test.abort()` function.
	ScriptAborted ExitCode = 108

	// GoPanic indicates the script was aborted by a panic in the Go runtime.
	GoPanic ExitCode = 109
)
```

## Source lineage & verified citations

All citations were verified against the source tree at commit `ddc3b0b1d`, and all behavioural claims come from the runtime output of a `k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)` binary built from that source. Read-only source files consulted and/or executed as evidence:

- `examples/http_get.js`, `README.md`
- `main.go`, `cmd/run.go`, `cmd/config.go`, `cmd/state/state.go`, `cmd/options.go`, `cmd/runtime_options.go`
- `metrics/builtin.go`, `metrics/metric_type.go`, `metrics/value_type.go`, `metrics/units.go`
- `js/summary.js`, `js/runner.go`
- `loader/readsource.go`, `loader/loader.go`, `errext/exitcodes/codes.go`, `api/server.go`
- `output/json/json.go` (for the `"Metric"` record-type string)

> **One citation correction vs. an earlier draft:** the `run` command's `Args:` constraint is at `cmd/run.go:498` (line 499 is `RunE: c.run`). This document uses the verified line `498`.
