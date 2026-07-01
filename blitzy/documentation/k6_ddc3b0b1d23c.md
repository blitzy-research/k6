# k6 — Behavior When Exercising a Single HTTP Request

This document answers, in a grounded and reproducible way, **how the [k6](https://grafana.com/docs/k6/latest/) load-testing tool behaves when it exercises a single HTTP request**. The system under study is the **k6 source at commit `ddc3b0b1d2`** (semantic version **k6 v0.55.0**), built from this repository (`go.k6.io/k6`) with **Go 1.23.12**. Built from that source commit, the binary reports its identity verbatim as:

```text
k6 v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)
```

> **Build-identity note — the `commit/…` label vs. the source commit.** The `commit/…` field is **build metadata that Go stamps from the git `HEAD` at build time**, not a property of the source snapshot: `FullVersion()` copies the first ten characters of the `vcs.revision` build setting [`lib/consts/consts.go:30-35`] into the `commit/<hash>` string [`lib/consts/consts.go:52`], and appends `-dirty` when the working tree has uncommitted changes [`lib/consts/consts.go:48-49`]. The verbatim identity above is what the binary prints when built from the **source commit `ddc3b0b1d2`** under study; the exact offline clone-and-build commands are in the [Reproduction appendix](#reproduction-appendix). Building instead from the delivered documentation branch's working copy stamps *that branch's* current `HEAD` — a different, evolving hash (for example `commit/32396bd430`, or with a `-dirty` suffix while this file is uncommitted) — because the branch adds only *this documentation file* on top of the source commit: `git diff --name-status ddc3b0b1d2..HEAD` returns only `A blitzy/documentation/k6_ddc3b0b1d23c.md`. The k6 **runtime source is therefore byte-for-byte identical** to `ddc3b0b1d2`, and every behavior documented below is unchanged regardless of which of the two builds produced the binary.

Every answer below is grounded in **two** kinds of evidence: (1) *observed output* captured from a real build-and-run of that binary against a **local** HTTP server (no internet dependency), quoted verbatim; and (2) *exact source citations* given as `` `file:line` `` references verified against the repository at commit `ddc3b0b1d2`. Where a value is something the question asks for (a metric name, a unit, a status code, an exit code, a config key), it is quoted literally rather than paraphrased.

## Question decomposition

The prompt decomposes into six distinct sub-questions, each answered in its own section:

- **O1** — How to test a single HTTP request (the minimal write→run workflow).
- **O2** — What the output looks like during the run (the **METRICS**, **UNITS**, and **PROTOCOLS** it reports).
- **O3** — The exact command used to execute the script.
- **O4** — Whether any specific configuration values or environment variables are required.
- **O5** — Whether the tool generates any external files.
- **O6** — Whether k6 enforces specific validation logic on the script.

A **Reproduction appendix**, a **Version-fidelity note**, and a final **Coverage pass** table follow the six answers.

---

## O1 — Testing a single HTTP request (minimal workflow)

The minimal workflow to test a single HTTP request with k6 is:

1. **Write** a JavaScript file that `import`s the `k6/http` module and `export`s a `default` function which performs one `http.get(...)` call.
2. **Run** it with `k6 run <script.js>` (see [O3](#o3--the-run-command)).

That is the entire "hello world" of k6: a default-exported function is one *iteration*, and by default k6 runs exactly **1 iteration** with **1 virtual user (VU)** (see [O4](#o4--configuration--environment-variables)).

### Canonical repository example

The repository ships this exact minimal script. Quoted verbatim from `examples/http_get.js` (lines 1–5):

```javascript
import http from 'k6/http';

export default function () {
  http.get('https://test-api.k6.io/');
};
```

- Source: `examples/http_get.js:1-5`.
- `http.get` maps to an HTTP `GET`. The `get` method is registered at `js/modules/k6/http/http.go:71` as:

  ```go
  mustExport("get", func(url sobek.Value, args ...sobek.Value) (*Response, error) {
  ```

  It issues the request through the module's default client via `mi.defaultClient.Request(http.MethodGet, url, args...)` (same function body). Source: `js/modules/k6/http/http.go:71`.
- The canonical write→run workflow and the threshold syntax are also documented in the repository `README.md` "Example script" section (`README.md:55-83`), which includes an `options.thresholds` entry `http_req_duration: ["p(99) < 3000"]` (`README.md:66`) and a `default` function calling `http.get(...)` (`README.md:77-78`).

### Offline-safe demonstration variant (temporary — NOT committed to the repo)

Because the repository example targets an external URL (`https://test-api.k6.io/`, `examples/http_get.js:4`) and no internet access is assumed, the live demonstration used an equivalent script pointed at a **local** server. It preserves the essential shape (`import http from 'k6/http'` + a `default` function calling `http.get(...)`) and adds a `check` to assert the status:

```javascript
import http from 'k6/http';
import { check } from 'k6';
const target = __ENV.TARGET || 'http://127.0.0.1:8099/';
export default function () {
  const res = http.get(target);
  check(res, { 'status is 200': (r) => r.status === 200 });
}
```

The **repository example is the canonical reference**; the **local-target variant is the reproducible demonstration** whose output is quoted verbatim in [O2](#o2--output-anatomy-metrics-units-protocols). This script (`single_get.js`) and its variants live outside the repository and were removed after the investigation (see the [Reproduction appendix](#reproduction-appendix)).

---

## O2 — Output anatomy: METRICS, UNITS, PROTOCOLS

By default k6 prints, to **stdout**: an ASCII banner, an execution-description block (`execution`/`script`/`output`/`scenarios`), a live progress bar, and — at the end — an aggregated **end-of-test summary**. The full verbatim output of the single-request run is below.

**Command:**

```bash
K6_NO_COLOR=true /tmp/k6bin/k6 run single_get.js
```

**Verbatim end-of-test summary (exit code `0`):**

```text
         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

     execution: local
        script: single_get.js
        output: -

     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 1 iterations for each of 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)


     ✓ status is 200

     checks.........................: 100.00% 1 out of 1
     data_received..................: 193 B   158 kB/s
     data_sent......................: 80 B    66 kB/s
     http_req_blocked...............: avg=216.97µs min=216.97µs med=216.97µs max=216.97µs p(90)=216.97µs p(95)=216.97µs
     http_req_connecting............: avg=149.58µs min=149.58µs med=149.58µs max=149.58µs p(90)=149.58µs p(95)=149.58µs
     http_req_duration..............: avg=565.62µs min=565.62µs med=565.62µs max=565.62µs p(90)=565.62µs p(95)=565.62µs
       { expected_response:true }...: avg=565.62µs min=565.62µs med=565.62µs max=565.62µs p(90)=565.62µs p(95)=565.62µs
     http_req_failed................: 0.00%   0 out of 1
     http_req_receiving.............: avg=115.73µs min=115.73µs med=115.73µs max=115.73µs p(90)=115.73µs p(95)=115.73µs
     http_req_sending...............: avg=82.33µs  min=82.33µs  med=82.33µs  max=82.33µs  p(90)=82.33µs  p(95)=82.33µs 
     http_req_tls_handshaking.......: avg=0s       min=0s       med=0s       max=0s       p(90)=0s       p(95)=0s      
     http_req_waiting...............: avg=367.56µs min=367.56µs med=367.56µs max=367.56µs p(90)=367.56µs p(95)=367.56µs
     http_reqs......................: 1       818.294445/s
     iteration_duration.............: avg=1.1ms    min=1.1ms    med=1.1ms    max=1.1ms    p(90)=1.1ms    p(95)=1.1ms   
     iterations.....................: 1       818.294445/s


running (00m00.0s), 0/1 VUs, 1 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.0s/10m0s  1/1 iters, 1 per VU
```

> **Version-fidelity note.** The output above is the **observed k6 v0.55.0 flat summary** — metrics are listed alphabetically with no section banners. The *latest* published Grafana k6 docs describe a newer **grouped** end-of-test summary (with `█ THRESHOLDS` and `█ TOTAL RESULTS` blocks; `HTTP` / `EXECUTION` / `NETWORK` / `CUSTOM` sections; and a `--summary-mode` option: `compact` / `full` / `legacy`). That grouped layout is a later-version feature and is **out of scope** for this v0.55.0 answer; do not read the quoted output as the grouped format.

### O2a — METRICS (names + types)

The built-in metrics are registered in `metrics/builtin.go` by `RegisterBuiltinMetrics`. The metrics visible in the summary above, with their type (and value-type where non-default), are:

| Metric | Type | Value type | Source |
|--------|------|-----------|--------|
| `checks` | Rate | — | `metrics/builtin.go:86` |
| `data_received` | Counter | Data | `metrics/builtin.go:109` |
| `data_sent` | Counter | Data | `metrics/builtin.go:108` |
| `http_req_blocked` | Trend | Time | `metrics/builtin.go:92` |
| `http_req_connecting` | Trend | Time | `metrics/builtin.go:93` |
| `http_req_duration` | Trend | Time | `metrics/builtin.go:91` |
| `http_req_failed` | Rate | — | `metrics/builtin.go:90` |
| `http_req_receiving` | Trend | Time | `metrics/builtin.go:97` |
| `http_req_sending` | Trend | Time | `metrics/builtin.go:95` |
| `http_req_tls_handshaking` | Trend | Time | `metrics/builtin.go:94` |
| `http_req_waiting` | Trend | Time | `metrics/builtin.go:96` |
| `http_reqs` | Counter | — | `metrics/builtin.go:89` |
| `iteration_duration` | Trend | Time | `metrics/builtin.go:83` |
| `iterations` | Counter | — | `metrics/builtin.go:82` |

Two related built-ins **do not appear** in this 1-VU / 1-iteration flat summary but are worth naming explicitly:

- `vus` — Gauge (`metrics/builtin.go:80`) and `vus_max` — Gauge (`metrics/builtin.go:81`). These are **runtime gauges** sampled during execution; the flat summary omits gauges for a completed 1-iteration run, so their absence from the quoted output is expected.
- `dropped_iterations` — Counter (`metrics/builtin.go:84`); emitted only when the scheduler has to drop iterations (never here).

**Composition relationship.** `http_req_duration` is the end-to-end request latency and is composed of `http_req_sending` + `http_req_waiting` (time-to-first-byte) + `http_req_receiving`; `http_req_blocked`, `http_req_connecting`, and `http_req_tls_handshaking` measure the connection-establishment phases. Every `Trend` metric is reported as `avg` / `min` / `med` / `max` / `p(90)` / `p(95)`, exactly as shown in the summary (e.g. `http_req_duration..............: avg=565.62µs min=565.62µs med=565.62µs max=565.62µs p(90)=565.62µs p(95)=565.62µs`). This matches the [Grafana k6 Metrics reference](https://grafana.com/docs/k6/latest/using-k6/metrics/).

### O2b — UNITS

k6 assigns each metric a **value type** that governs how its numbers are humanized for display. The value types are defined in `metrics/value_type.go:6-9`:

```go
const (
	Default = ValueType(iota) // Values are presented as-is
	Time                      // Values are time durations (milliseconds)
	Data                      // Values are data amounts (bytes)
)
```

The humanized rendering happens in the embedded summary script `js/summary.js`:

- **Data** → `humanizeBytes` (function at `js/summary.js:134`) uses `units = ['B', 'kB', 'MB', 'GB', 'TB', 'PB', 'EB', 'ZB', 'YB']` (`js/summary.js:135`) with `base = 1000` (`js/summary.js:136`).
- **Time** → durations are scaled through the `unitMap` at `js/summary.js:147-150`:

  ```javascript
  var unitMap = {
    s: { unit: 's', coef: 0.001 },
    ms: { unit: 'ms', coef: 1 },
    us: { unit: 'µs', coef: 1000 },
  }
  ```

- **Percentage / Rate** and dispatch by type is handled by `humanizeValue` (`js/summary.js:204`); a `rate` metric is formatted as `(Math.trunc(val * 100 * 100) / 100).toFixed(2) + '%'` (`js/summary.js:207`), `Data` goes to `humanizeBytes`, and `Time` goes to the duration humanizer.

Tying each unit to the **observed** literals in the summary:

- **Durations auto-scale** by magnitude: `565.62µs` (microseconds) for `http_req_duration` vs `1.1ms` (milliseconds) for `iteration_duration`; `http_req_tls_handshaking` shows `0s` because there is no TLS on plain HTTP.
- **Data** shows a byte count plus a throughput rate: `data_received..................: 193 B   158 kB/s` and `data_sent......................: 80 B    66 kB/s`.
- **Counters** show a count plus a per-second rate: `http_reqs......................: 1       818.294445/s` (and `iterations` likewise `1       818.294445/s`).
- **Rates** show a percentage plus `N out of M`: `checks.........................: 100.00% 1 out of 1` and `http_req_failed................: 0.00%   0 out of 1`.

### O2c — PROTOCOLS (and tags)

The `proto` value shown for HTTP traffic is one of k6's **system tags**. The default system tag set is defined at `metrics/system_tag.go:47-49`:

```go
var DefaultSystemTagSet = SystemTagSet(
	TagProto | TagSubproto | TagStatus | TagMethod | TagURL | TagName | TagGroup |
		TagCheck | TagError | TagErrorCode | TagTLSVersion | TagScenario | TagService | TagExpectedResponse)
```

So the tags enabled by default include `proto`, `subproto`, `status`, `method`, `url`, `name`, `group`, `check`, `error`, `error_code`, `tls_version`, `scenario`, `service`, and `expected_response`. A code comment just above the definition (`metrics/system_tag.go:44`) notes that `iter`, `vu`, `ocsp_status`, and `ip` are **not** enabled by default.

- **`proto`** — its value is taken from the actual HTTP response protocol. At `lib/netext/httpext/transport.go:118`:

  ```go
  tagsAndMeta.SetSystemTagOrMetaIfEnabled(enabledTags, metrics.TagProto, unfReq.response.Proto)
  ```

  where `Proto` is the response struct field declared at `lib/netext/httpext/response.go:61`:

  ```go
  Proto          string                   `json:"proto"`
  ```

  Against the local Python server, the observed `proto` value was **`HTTP/1.0`** (the protocol that server negotiates; see the [Reproduction appendix](#reproduction-appendix)).

- **`expected_response`** — the `{ expected_response:true }` sub-row on `http_req_duration` derives from `lib/netext/httpext/transport.go:143`:

  ```go
  tagsAndMeta.SetSystemTagOrMetaIfEnabled(enabledTags, metrics.TagExpectedResponse, strconv.FormatBool(expected))
  ```

  `expected` is computed against the default success range `{{200, 399}}` defined at `js/modules/k6/http/response_callback.go:12-13`:

  ```go
  var defaultExpectedStatuses = expectedStatuses{
      minmax: [][2]int{{200, 399}},
  }
  ```

  Because the local server returned `200`, `expected` was `true`, producing the `{ expected_response:true }` sub-metric on `http_req_duration`. By default, therefore, HTTP `4xx`/`5xx` count as **failures** (only `200–399` are "expected"); this is changeable in-script via `setResponseCallback`. This matches the failure semantics described in the [Grafana k6 Metrics reference](https://grafana.com/docs/k6/latest/using-k6/metrics/).

- **Observed granular tag values** (from the `--out json` capture in [O5](#o5--external-files)): `proto` = `HTTP/1.0`, `method` = `GET`, `status` = `200`, `expected_response` = `true`, `scenario` = `default`, and `url` / `name` = `http://127.0.0.1:8099/`.

---


## O3 — The run command

The exact command to execute a script is:

```bash
k6 run <script.js>
```

The observed invocation for the demonstration was `k6 run single_get.js`.

- The `run` subcommand is constructed by `getCmdRun` at `cmd/run.go:462`. Its Cobra command is declared with `Use: "run"` at `cmd/run.go:491` and `Short: "Start a test"` at `cmd/run.go:492`, and it accepts **exactly one** positional argument — the path to the script — enforced by the Cobra `Args` validator at `cmd/run.go:498`:

  ```go
  func getCmdRun(gs *state.GlobalState) *cobra.Command {
  ```
  ```go
  runCmd := &cobra.Command{
      Use:   "run",
      Short: "Start a test",
  ```

  The one-positional-argument rule is the `Args` field, quoted verbatim from `cmd/run.go:498`:

  ```go
  Args:    exactArgsWithMsg(1, "arg should either be \"-\", if reading script from stdin, or a path to a script file"),
  ```

- The program entrypoint is `main.go`, whose `main()` calls `cmd.Execute()` (`main.go:8-9`), which wires up the command tree that includes `run`.

**Common flags** relevant to a single-request run (each detailed in its linked section):

- `--out <backend>=<arg>` — stream results to an output backend such as `json`/`csv` (see [O5](#o5--external-files)).
- `-e` / `--env VAR=value` — inject a user variable readable via `__ENV` (see [O4](#o4--configuration--environment-variables)); defined at `cmd/runtime_options.go:32`.
- `--no-summary` — suppress the end-of-test summary (flag registered at `cmd/runtime_options.go:34`).

---

## O4 — Configuration / environment variables

**No configuration or environment variable is required for a basic single-request run.** An unconfigured run defaults to **1 VU / 1 iteration**, as the observed scenario line confirms verbatim:

```text
              * default: 1 iterations for each of 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)
```

Two mechanisms exist when you *do* want to configure a run:

- **User variables** are supplied with `-e` / `--env VAR=value` and read in-script through the `__ENV` object. The flag is defined at `cmd/runtime_options.go:32`:

  ```go
  flags.StringArrayP("env", "e", nil, "add/override environment variable with `VAR=value`")
  ```

- **`K6_*` options** bind k6's own configuration through envconfig. Examples declared at `cmd/config.go:45-48` are `K6_OUT`, `K6_LINGER`, `K6_NO_USAGE_REPORT`, and `K6_WEB_DASHBOARD`:

  ```go
  Out           []string  `json:"out" envconfig:"K6_OUT"`
  Linger        null.Bool `json:"linger" envconfig:"K6_LINGER"`
  NoUsageReport null.Bool `json:"noUsageReport" envconfig:"K6_NO_USAGE_REPORT"`
  WebDashboard  null.Bool `json:"webDashboard" envconfig:"K6_WEB_DASHBOARD"`
  ```

  Other commonly used `K6_*` variables — each grounded in source — include `K6_NO_SUMMARY` (read via `saveBoolFromEnv(environment, "K6_NO_SUMMARY", &opts.NoSummary)` at `cmd/runtime_options.go:94`), and `K6_VUS`, `K6_DURATION`, and `K6_ITERATIONS` (declared as envconfig-bound option fields at `lib/options.go:234-236`):

  ```go
  VUs        null.Int           `json:"vus" envconfig:"K6_VUS"`
  Duration   types.NullDuration `json:"duration" envconfig:"K6_DURATION"`
  Iterations null.Int           `json:"iterations" envconfig:"K6_ITERATIONS"`
  ```

**Confirming `-e` works.** The demonstration injected a variable and had the script log it back:

```bash
/tmp/k6bin/k6 run -e TARGET=http://127.0.0.1:8099/ env_test.js
```

Observed console line (verbatim, exit code `0`):

```text
time="2026-07-01T02:45:44Z" level=info msg="injected TARGET=http://127.0.0.1:8099/" source=console
```

The script read `__ENV.TARGET` and logged the injected value, confirming the `-e`/`__ENV` mechanism end-to-end.

---


## O5 — External files

**A plain run writes NO external files.** The summary header line reports the output destination as a dash:

```text
        output: -
```

`output: -` means results were sent only to **stdout** (the end-of-test summary); nothing was written to disk.

**Files are produced only when explicitly requested**, via `--out <backend>=<file>`. The demonstration ran:

```bash
/tmp/k6bin/k6 run --out json=result.json --out csv=result.csv single_get.js
```

which changed the header line to (verbatim):

```text
        output: json (result.json), csv (result.csv)
```

and created two files: `result.json` and `result.csv`.

### `result.json` — JSON-lines

`result.json` is written as **JSON-lines**: alternating `Metric` *definition* objects and `Point` *data* objects, one JSON document per line. Verbatim captured lines:

```json
{"type":"Metric","data":{"name":"http_reqs","type":"counter","contains":"default","thresholds":[],"submetrics":null},"metric":"http_reqs"}
{"metric":"http_reqs","type":"Point","data":{"time":"2026-07-01T02:45:10.321870603Z","value":1,"tags":{"expected_response":"true","group":"","method":"GET","name":"http://127.0.0.1:8099/","proto":"HTTP/1.0","scenario":"default","status":"200","url":"http://127.0.0.1:8099/"}}}
{"type":"Metric","data":{"name":"http_req_duration","type":"trend","contains":"time","thresholds":[],"submetrics":[{"name":"http_req_duration{expected_response:true}","suffix":"expected_response:true","tags":{"expected_response":"true"}}]},"metric":"http_req_duration"}
{"metric":"http_req_duration","type":"Point","data":{"time":"2026-07-01T02:45:10.321870603Z","value":0.432926,"tags":{"expected_response":"true","group":"","method":"GET","name":"http://127.0.0.1:8099/","proto":"HTTP/1.0","scenario":"default","status":"200","url":"http://127.0.0.1:8099/"}}}
```

- The `"type":"Metric"` envelope is written from `output/json/json.go:156` (`Type: "Metric"`).
- The `"type":"Point"` envelope is written from `output/json/wrapper.go:27` (`Type: "Point"`).
- The `proto` tag value `HTTP/1.0` appears in each point's `tags`, and the `http_req_duration` submetric `http_req_duration{expected_response:true}` is declared in that metric's definition object — corroborating the tags discussed in [O2c](#o2c--protocols-and-tags).

### `result.csv` — 19-column schema

`result.csv` has a **19-column header**. Verbatim header plus a sample data row:

```text
metric_name,timestamp,metric_value,check,error,error_code,expected_response,group,method,name,proto,scenario,service,status,subproto,tls_version,url,extra_tags,metadata
http_reqs,1782873910,1.000000,,,,true,,GET,http://127.0.0.1:8099/,HTTP/1.0,default,,200,,,http://127.0.0.1:8099/,,
```

The header begins with the fixed triple `metric_name,timestamp,metric_value` followed by the tag columns. That leading triple is built by `MakeHeader` (`output/csv/output.go:208-212`), specifically at `output/csv/output.go:212`:

```go
return append([]string{"metric_name", "timestamp", "metric_value"}, tags...)
```

(The tag list is extended with `extra_tags` and `metadata` at `output/csv/output.go:210-211` before that append, which is why they appear as the final two columns.)

### Arbitrary files via `handleSummary()`

A script may also write **arbitrary files** by exporting a `handleSummary(data)` function that returns a `{ filename: content }` map. k6 consumes that map in `handleSummaryResult` at `cmd/run.go:508`; for any key other than `stdout`/`stderr` it opens the path for writing (around `cmd/run.go:518`):

```go
return fs.OpenFile(path, syscall.O_WRONLY|syscall.O_CREAT|syscall.O_TRUNC, 0o666)
```

and copies the associated content into it. The end-of-test summary pipeline itself lives in `js/summary.go` (which embeds `js/summary.js` and `js/summary-wrapper.js`), and the streaming output backends live under `output/` (`json`, `csv`, `influxdb`, `cloud`, …).

**In short (O5):** no files by default (`output: -`); `result.json` / `result.csv` only with `--out`; arbitrary files only via a `handleSummary()` export.

---


## O6 — Script validation logic

Yes — k6 validates the script before and at initialization, and returns **distinct non-zero exit codes** depending on the failure. Each case below is shown with the command, the **verbatim** error message, and the observed exit code.

### (a) No callable `default` export → exit `104` (`InvalidConfig`)

Script `no_default.js` containing only `export function foo() {}` (no `default`). Command (run from `/tmp/k6work/`, with exit-code capture):

```bash
/tmp/k6bin/k6 run no_default.js; echo "exit=$?"
```

Verbatim stderr, followed by the captured exit code:

```text
time="2026-07-01T02:45:44Z" level=error msg="There were problems with the specified script configuration:\n\t- executor default: function 'default' not found in exports"
exit=104
```

Source: `cmd/config.go:287`:

```go
return fmt.Errorf("executor %s: function '%s' not found in exports", conf.GetName(), execFn)
```

### (b) Empty script (or non-function `default`) → exit `255`

Empty script `empty.js`. Command (run from `/tmp/k6work/`, with exit-code capture):

```bash
/tmp/k6bin/k6 run empty.js; echo "exit=$?"
```

Verbatim stderr, followed by the captured exit code:

```text
time="2026-07-01T02:45:44Z" level=error msg="could not initialize 'empty.js': could not load JS test 'file:///tmp/k6work/empty.js': no exported functions in script"
exit=255
```

Source: `js/bundle.go:237-238`:

```go
if len(b.callableExports) == 0 {
    return errors.New("no exported functions in script")
}
```

### (c) JavaScript syntax error → exit `107` (`ScriptException`)

Script `syntax.js` with a deliberate syntax error. Command (run from `/tmp/k6work/`, with exit-code capture):

```bash
/tmp/k6bin/k6 run syntax.js; echo "exit=$?"
```

Verbatim stderr, followed by the captured exit code:

```text
time="2026-07-01T02:45:44Z" level=error msg="GoError: file:///tmp/k6work/syntax.js: Line 1:34 Unexpected identifier (and 5 more errors)\n" hint="script exception"
exit=107
```

(The specific `Line 1:34` column and token reflect the exact contents of the deliberately-broken `syntax.js`; the deterministic, question-relevant fact is that any JavaScript parse error surfaces as a `GoError … script exception` and exit `107`.)

### (d) Missing file → exit `255`

Running a path that does not exist. Command (run from `/tmp/k6work/`, with exit-code capture):

```bash
/tmp/k6bin/k6 run does_not_exist.js; echo "exit=$?"
```

Verbatim stderr, followed by the captured exit code:

```text
time="2026-07-01T02:45:44Z" level=error msg="The moduleSpecifier \"does_not_exist.js\" couldn't be found on local disk. Make sure that you've specified the right path to the file. If you're running k6 using the Docker image make sure you have mounted the local directory (-v /local/path/:/inside/docker/path) containing your script and modules so that they're accessible by k6 from inside of the container, see https://grafana.com/docs/k6/latest/using-k6/modules/#using-local-modules-with-docker."
exit=255
```

### (e) Threshold breach → exit `99` (`ThresholdsHaveFailed`)

Adding an impossible threshold to the script's options, e.g.:

```javascript
export const options = { thresholds: { http_req_duration: ['p(95)<0.0001'] } };
```

makes the summary mark the metric with a `✗`, and k6 exits `99`. The demonstration ran a `threshold_fail.js` (a single `http.get(...)` plus the impossible threshold shown above). Command (run from `/tmp/k6work/`, with exit-code capture):

```bash
/tmp/k6bin/k6 run threshold_fail.js; echo "exit=$?"
```

Verbatim stderr, followed by the captured exit code:

```text
level=error msg="thresholds on metrics 'http_req_duration' have been crossed"
exit=99
```

### Exit-code enumeration

The exit codes are enumerated in `errext/exitcodes/codes.go`:

| Constant | Value | Source |
|----------|-------|--------|
| `ThresholdsHaveFailed` | `99` | `errext/exitcodes/codes.go:20` |
| `SetupTimeout` | `100` | `errext/exitcodes/codes.go:23` |
| `TeardownTimeout` | `101` | `errext/exitcodes/codes.go:26` |
| `InvalidConfig` | `104` | `errext/exitcodes/codes.go:36` |
| `ExternalAbort` | `105` | `errext/exitcodes/codes.go:41` |
| `ScriptException` | `107` | `errext/exitcodes/codes.go:48` |
| `ScriptAborted` | `108` | `errext/exitcodes/codes.go:52` |
| `GoPanic` | `109` | `errext/exitcodes/codes.go:55` |

**The distinction matters.** *Configuration* failures (like a missing `default` export that no scenario can execute) map to `104` (`InvalidConfig`, `cmd/config.go:287`). *Init-time load* failures (an empty script with no exported functions, or a missing file) surface as the generic `255`. *Runtime* JS exceptions map to `107` (`ScriptException`). A crossed *threshold* forces `99` (`ThresholdsHaveFailed`) at the end of an otherwise-successful run.

---


## Reproduction appendix

The observations above can be reproduced end-to-end as follows.

### Build

The repository was compiled with **Go 1.23.12**. (`go.mod` declares `go 1.21` at `go.mod:3` and `toolchain go1.21.13` at `go.mod:5`; the project's CI and Docker images standardize on Go 1.23.x.) Dependencies are **vendored** (`vendor/` is present), so the build needs no network.

The `commit/…` field in the version string is **Go's VCS stamp of the git `HEAD` at build time**, not a property of the source snapshot: `FullVersion()` copies the first ten characters of the `vcs.revision` build setting [`lib/consts/consts.go:30-35`] into the `commit/<hash>` string [`lib/consts/consts.go:52`], and appends `-dirty` when the working tree has uncommitted changes [`lib/consts/consts.go:48-49`]. To reproduce the **source-commit identity `commit/ddc3b0b1d2`** quoted at the top of this document — the identity of the k6 tree under study — build from a checkout whose `HEAD` *is* that commit. A standalone (non-worktree) clone does this offline:

```bash
# Reproduce the quoted source-commit identity (offline; hardlinked objects; real .git directory):
git clone --local . /tmp/k6src && cd /tmp/k6src
git checkout --detach ddc3b0b1d2
GOFLAGS=-mod=vendor go build -o /tmp/k6src/k6 .
/tmp/k6src/k6 version
# => k6 v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)
```

Building instead from the **delivered documentation branch's working copy** (i.e. `go build` at the repository root) stamps that branch's current `HEAD`. That hash differs from the source commit and evolves with each commit on the branch (and gains a `-dirty` suffix while this file is uncommitted), so it is shown below as a placeholder rather than a fixed value. The branch adds only this documentation file on top of the source commit — `git diff --name-status ddc3b0b1d2..HEAD` returns only `A blitzy/documentation/k6_ddc3b0b1d23c.md` — so the runtime source, and every behavior documented here, is identical; only the embedded label differs:

```bash
# Build from the delivered branch working copy (run at the repository root):
GOFLAGS=-mod=vendor go build -o /tmp/k6bin/k6 .
/tmp/k6bin/k6 version
# => k6 v0.55.0 (commit/<HEAD>, go1.23.12, linux/amd64)   # <HEAD> = first 10 chars of `git rev-parse HEAD` (e.g. commit/32396bd430), plus "-dirty" if the tree is modified
```

A `git worktree`-based checkout of the source commit is **not** stamped by Go — a linked worktree's top-level `.git` is a gitdir-pointer file rather than a real directory — so it prints `k6 v0.55.0 (go1.23.12, linux/amd64)` with no `commit/…`; use a real clone as shown above.

### Local target (offline-safe)

Because the repository examples target external URLs and no internet access is assumed, a **local** endpoint was used: a Python `http.server`-based handler bound to `127.0.0.1:8099` that answers every request with `200` and a small JSON body (`Content-Type: application/json`). A minimal handler of this shape reproduces the observed behavior:

```python
from http.server import BaseHTTPRequestHandler, HTTPServer

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        body = b'{"ok":true}'
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

HTTPServer(("127.0.0.1", 8099), Handler).serve_forever()
```

`curl` confirmed the response line and headers `HTTP/1.0 200 OK`, `Server: BaseHTTP/0.6 Python/3.12.3`, `Content-Type: application/json`. The `HTTP/1.0` in that status line comes from `BaseHTTPRequestHandler` (its default `protocol_version` is `HTTP/1.0`), and is exactly why the `proto` tag was observed as `HTTP/1.0` in [O2c](#o2c--protocols-and-tags) and in the `result.json` / `result.csv` captures in [O5](#o5--external-files).

### Exact commands run

- Plain run: `K6_NO_COLOR=true /tmp/k6bin/k6 run single_get.js` → the end-of-test summary in [O2](#o2--output-anatomy-metrics-units-protocols).
- File outputs: `/tmp/k6bin/k6 run --out json=result.json --out csv=result.csv single_get.js` → `result.json` / `result.csv` in [O5](#o5--external-files).
- Env variable: `/tmp/k6bin/k6 run -e TARGET=http://127.0.0.1:8099/ env_test.js` → the injected-`__ENV` log line in [O4](#o4--configuration--environment-variables).
- Invalid-script runs: `no_default.js`, `empty.js`, `syntax.js`, a missing path, and an impossible-threshold script → the five validation errors and exit codes in [O6](#o6--script-validation-logic).

### Captured outputs

The verbatim end-of-test summary, the `result.json` JSON-lines and `result.csv` 19-column schema, and the five verbatim validation errors with their exit codes (`104`, `255`, `107`, `255`, `99`) are all quoted in the sections above.

### Cleanup / repository integrity

The temporary scripts (`single_get.js`, `env_test.js`, `no_default.js`, `empty.js`, `syntax.js`, …) and the generated `result.json` / `result.csv` were created **outside** the repository (under `/tmp`) and removed after the investigation. The built `k6` binary at `/tmp/k6bin/k6` also lives outside the repository. Consequently `git status --porcelain` on the repository is empty apart from this new document — no existing repository file was modified.

### Version-fidelity note (reiterated)

As noted under [O2](#o2--output-anatomy-metrics-units-protocols): the output quoted here is the **observed k6 v0.55.0 flat summary** (alphabetical metric list, no section banners). The *latest* published Grafana k6 docs describe a newer **grouped** summary (`█ THRESHOLDS` / `█ TOTAL RESULTS`; `HTTP` / `EXECUTION` / `NETWORK` / `CUSTOM` sections; a `--summary-mode` option with `compact` / `full` / `legacy`). That grouped layout is a later-version feature and is **out of scope** for this v0.55.0 answer.

### Documentation references

- Results output / end-of-test summary: [https://grafana.com/docs/k6/latest/get-started/results-output/](https://grafana.com/docs/k6/latest/get-started/results-output/)
- Metrics reference: [https://grafana.com/docs/k6/latest/using-k6/metrics/](https://grafana.com/docs/k6/latest/using-k6/metrics/)

---

## Coverage pass

Every sub-question is answered above:

| Sub-question | Answered in section | Key grounded evidence |
|--------------|---------------------|-----------------------|
| **O1** — How to test a single HTTP request | [O1](#o1--testing-a-single-http-request-minimal-workflow) | `examples/http_get.js:1-5`; `http.get` → `GET` at `js/modules/k6/http/http.go:71`; `README.md:55-83` |
| **O2** — Output (METRICS, UNITS, PROTOCOLS) | [O2](#o2--output-anatomy-metrics-units-protocols) | Verbatim end-of-test summary; metric types `metrics/builtin.go:80-109`; units `metrics/value_type.go:6-9` + `js/summary.js:134-150,204-207`; tags `metrics/system_tag.go:47-49`, `lib/netext/httpext/transport.go:118/143`, `lib/netext/httpext/response.go:61`, `js/modules/k6/http/response_callback.go:12-13`; observed `proto=HTTP/1.0`, `565.62µs`, `1.1ms`, `193 B`, `818.294445/s`, `100.00%` |
| **O3** — The run command | [O3](#o3--the-run-command) | `k6 run <script.js>`; `cmd/run.go:462/491/492`; `main.go:8-9` |
| **O4** — Configuration / env vars | [O4](#o4--configuration--environment-variables) | None required (defaults 1 VU / 1 iteration); `-e`/`--env` at `cmd/runtime_options.go:32`; `K6_*` at `cmd/config.go:45-48`; verbatim `injected TARGET=...` log line |
| **O5** — External files | [O5](#o5--external-files) | `output: -` (none by default); `--out json/csv` → verbatim `result.json` JSON-lines + 19-column `result.csv`; `output/json/json.go:156`, `output/json/wrapper.go:27`, `output/csv/output.go:212`; `handleSummary()` via `cmd/run.go:508/518` |
| **O6** — Script validation logic | [O6](#o6--script-validation-logic) | Five verbatim errors + exit codes `104`/`255`/`107`/`255`/`99`; `cmd/config.go:287`; `js/bundle.go:237-238`; `errext/exitcodes/codes.go:20-55` |

All six sub-questions (O1–O6) are addressed, each grounded in verbatim observed output and/or an exact `file:line` citation.

