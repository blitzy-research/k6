# How k6 Consolidates Configuration Into Effective Options (branch `k6_ddc3b0b1d23c`)

This document answers, with runtime evidence, how k6 (`grafana/k6`) consolidates its configuration inputs — the script's `export const options`, CLI flags, the `--config` JSON file, environment variables (`K6_*`), and built-in defaults — into the effective `lib.Options` that its execution scheduler uses; at what point that decision is frozen for the run; which source wins when the inputs conflict; and how this behaves when multiple VUs are running. Every behavioral claim below is backed by a real `k6 run` on a canonical build **`k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)`** and by a `file:line` citation into the source tree at HEAD `ddc3b0b1d23c`. Where the official documentation and the observed behavior diverge (the `-e`/`--env` case), the observed behavior is treated as authoritative and the discrepancy is documented.

> Note on timestamps: the `time="..."` values in the captured output below are run-specific (they record when each experiment ran) and are otherwise immaterial to the results; the behavior itself is stable across repeated runs.

## Section 1 — Direct Answer, Precedence Ladder, and the `-e`/`--env` Nuance

**How consolidation works, in one paragraph.** k6 consolidates its inputs by *layering them with successive merge (`Apply`) calls*, where each higher tier overrides the lower ones; it then derives full scenarios from the merged execution shortcuts (`vus` / `duration` / `iterations` / `stages`); and finally it **freezes** the derived result into the run state before the scheduler reads it. All of the named items — **VUs**, **duration**, **scenario settings**, the script's **`export const options`**, **CLI flags**, and the **config file** — are fields of the same `lib.Options` struct, and the winner for each field is chosen by tier.

**Effective precedence, highest → lowest:**

| Rank | Source | How it enters consolidation |
|------|--------|-----------------------------|
| 1 (highest) | **CLI flags** (e.g. `--vus`, `--duration`, `--iterations`) | applied LAST in the merge chain [cmd/config.go:L203] |
| 2 | **Environment variables** (`K6_*`) | read by `readEnvConfig(gs.Env)` [cmd/config.go:L194], applied just before the final CLI pass [cmd/config.go:L203] |
| 3 | **Script `export const options`** (the runner options) | `conf.Apply(Config{Options: runnerOpts})` [cmd/config.go:L201] |
| 4 | **Config file** (JSON via `--config`) | read by `readDiskConfig`, applied via `cliConf.Apply(fileConf)` [cmd/config.go:L199] |
| 5 (lowest) | **Built-in defaults** | filled in by `applyDefault(conf)` [cmd/config.go:L204] |

In short: **CLI flags → environment variables (`K6_*`) → script `export const options` → config file (JSON) → built-in defaults.** Sections 2–4 prove each rung of this ladder with real runs.

**The `-e`/`--env` nuance (OBSERVED — authoritative).**

- **(corroboration — official docs)** The official Grafana k6 documentation states that `-e`/`--env` only injects variables into the script's `__ENV` object and does *not* configure options — for example, it documents that `-e K6_ITERATIONS=120` does not configure the script's iterations, whereas `K6_ITERATIONS=120 k6 run script.js` does set them.
- **OBSERVED (authoritative) on this canonical build:** under a DEFAULT `k6 run`, `-e K6_ITERATIONS=120` **did** set iterations to 120 (see the nuance captures in Section 4, Case A). This is because `k6 run` defaults `--include-system-env-vars=true`, which aliases the runtime-options env map onto the process env map so that `-e` values reach the environment-variable tier. With `--include-system-env-vars=false`, `-e` only populates `__ENV` and the script value wins (iterations = 2, see Case B). A real environment variable always sets the option (Case C).
- **Conclusion (stated plainly):** the docs' statement holds only when `--include-system-env-vars=false`; under the default `k6 run`, a `-e K6_x=…` participates at the environment-variable tier. The docs are corroboration only; the code and the captured runs are authoritative.

## Section 2 — The Consolidation Mechanism (documented behavior, with citations)

Consolidation happens in `getConsolidatedConfig` [cmd/config.go:L189-L204]. It layers every source with successive `Apply` calls — where the *argument* to `Apply` wins — and then fills in defaults. The order is documented verbatim in the comment block immediately above the function [cmd/config.go:L180-L186]:

```go
// Assemble the final consolidated configuration from all of the different sources:
// - start with the CLI-provided options to get shadowed (non-Valid) defaults in there
// - add the global file config options
// - add the Runner-provided options (they may come from Bundle too if applicable)
// - add the environment variables
// - merge the user-supplied CLI flags back in on top, to give them the greatest priority
// - set some defaults if they weren't previously specified
func getConsolidatedConfig(gs *state.GlobalState, cliConf Config, runnerOpts lib.Options) (conf Config, err error) {
    fileConf, err := readDiskConfig(gs)
    ...
    envConf, err := readEnvConfig(gs.Env)
    ...
    conf = cliConf.Apply(fileConf)

    conf = conf.Apply(Config{Options: runnerOpts})

    conf = conf.Apply(envConf).Apply(cliConf)
    conf = applyDefault(conf)
    ...
    return conf, nil
}
```

**The merge chain, line by line (`cmd/config.go`):**

- `conf = cliConf.Apply(fileConf)` [L199] — the **config file** is applied over a CLI-seeded base. `cliConf` here carries only *shadowed*, non-`Valid` defaults, so the file's real values are not clobbered by empty CLI values.
- `conf = conf.Apply(Config{Options: runnerOpts})` [L201] — the **script's `export const options`** (`runnerOpts`) is applied on top of the file.
- `conf = conf.Apply(envConf).Apply(cliConf)` [L203] — **environment variables** (`envConf`) are applied next, and then the real **CLI flags** (`cliConf`) are applied **LAST, giving them the greatest priority**.
- `conf = applyDefault(conf)` [L204] — anything still unset is filled with **built-in defaults**.

**Subtlety — CLI options are applied twice.** The first application, `cliConf.Apply(fileConf)` [L199], only seeds shadowed, non-`Valid` defaults so that file / script / env values are not overwritten by empty CLI values. The last application, `.Apply(cliConf)` [L203], gives the real, user-supplied CLI flags the highest priority. This is exactly what the comment block promises: *"merge the user-supplied CLI flags back in on top, to give them the greatest priority"* [cmd/config.go:L185].

**The per-tier merge rule** lives in `Config.Apply` [cmd/config.go:L71]; its doc comment (L70) reads *"Apply the provided config on top of the current one, returning a new one. The provided config has priority."* — i.e. the argument to `Apply` wins. After consolidation, `deriveAndValidateConfig` [cmd/config.go:L248] turns the merged execution shortcuts into full scenarios (it calls `executor.DeriveScenariosFromShortcuts`) and validates the result.

### Execution group-reset — `lib/options.go:L357-L377`

```go
func (o Options) Apply(opts Options) Options {
    if opts.Paused.Valid {
        o.Paused = opts.Paused
    }
    if opts.VUs.Valid {
        o.VUs = opts.VUs
    }

    // Specifying duration, iterations, stages, or execution in a "higher" config tier
    // will overwrite all of the previous execution settings (if any) from any
    // "lower" config tiers
    ...
    if opts.Duration.Valid || opts.Iterations.Valid || opts.Stages != nil || opts.Scenarios != nil {
        o.Duration = types.NewNullDuration(0, false)
        o.Iterations = null.NewInt(0, false)
        o.Stages = nil
        o.Scenarios = nil
    }
```

- **VUs are merged independently** [lib/options.go:L361-L363]: `vus` is **not** part of the execution reset group, so a `vus` from a lower tier can carry over even when a higher tier changes the execution mode.
- **Group-reset:** if a higher tier sets **any** of `duration` / `iterations` / `stages` / `scenarios`, the merge first clears **all four** execution fields inherited from the lower tier [lib/options.go:L371-L376] before applying the higher-tier value. This is why a higher tier's execution setting replaces the lower tier's **entire** execution/scenario block (demonstrated in EXP5).

### Shortcut → scenario derivation — `lib/executor/execution_config_shortcuts.go:L52-L120`

```go
func DeriveScenariosFromShortcuts(opts lib.Options, logger logrus.FieldLogger) (lib.Options, error) {
    result := opts
    switch {
    case opts.Iterations.Valid:
        ...
        result.Scenarios = getSharedIterationsScenario(opts.Iterations, opts.Duration, opts.VUs)   // -> "shared-iterations"
    case opts.Duration.Valid:
        ...
        result.Scenarios = getConstantVUsScenario(opts.Duration, opts.VUs)                          // -> "constant-vus"
    case len(opts.Stages) > 0:
        ...
        result.Scenarios = getRampingVUsScenario(opts.Stages, opts.VUs)                             // -> "ramping-vus"
    case len(opts.Scenarios) > 0:
        // Do nothing, scenarios was explicitly specified
    default:
        if opts.VUs.Valid && opts.VUs.Int64 != 1 {
            logger.Warnf(
                "the `vus=%d` option will be ignored, it only works in conjunction with `iterations`, `duration`, or `stages`",
                opts.VUs.Int64,
            )
        }
        ...
        // per-VU iterations config with 1 VU and 1 iteration
        result.Scenarios = lib.ScenarioConfigs{
            lib.DefaultScenarioName: NewPerVUIterationsConfig(lib.DefaultScenarioName),
        }
    }
```

The shortcut-to-scenario mapping, with citations:

- `iterations` → **`shared-iterations`** [lib/executor/execution_config_shortcuts.go:L56-L67]
- `duration` → **`constant-vus`** [lib/executor/execution_config_shortcuts.go:L69-L86]
- `stages` → **`ramping-vus`** [lib/executor/execution_config_shortcuts.go:L88-L94]
- explicit `scenarios` → **used as-is** [lib/executor/execution_config_shortcuts.go:L96-L97]
- otherwise (no execution shortcut) → a per-VU-iterations default of **1 VU / 1 iteration** [lib/executor/execution_config_shortcuts.go:L99, L115-L119]
- a lone `vus` (≠ 1, without `iterations` / `duration` / `stages`) is **ignored with a warning** [lib/executor/execution_config_shortcuts.go:L101-L106]; the exact warning text is emitted at L103 (observed in EXP6): `` the `vus=%d` option will be ignored, it only works in conjunction with `iterations`, `duration`, or `stages` ``.

## Section 3 — Finalization / the "Freeze" Point

Consolidation becomes **final** during test loading:

- The finalization entry point is `loadAndConfigureLocalTest` [cmd/test_load.go:L246], which calls `consolidateDeriveAndValidateConfig` — that helper runs `getConsolidatedConfig` (Section 2) and then `deriveAndValidateConfig` to produce `derivedConfig`.
- The **freeze** happens in `buildTestRunState`, which writes the derived options into the run state: `Options: lct.derivedConfig.Options` [cmd/test_load.go:L280], annotated with the comment **`// we will always run with the derived options`**.
- From `cmd/run.go`, `conf := test.derivedConfig` [cmd/run.go:L127] is handed to the scheduler via `execution.NewScheduler(testRunState, controller)` [cmd/run.go:L135]; the immediately preceding debug line `"Initializing the execution scheduler..."` is emitted at [cmd/run.go:L134].
- The scheduler reads the frozen options: `options := trs.Options` [execution/scheduler.go:L39] inside `NewScheduler` [execution/scheduler.go:L38], and it builds the entire execution plan from them via `options.Scenarios.GetFullExecutionRequirements(et)` [execution/scheduler.go:L44].

**Stated plainly:** after the `TestRunState.Options` assignment [cmd/test_load.go:L280], **no source (script / CLI / config file / env) is re-read** — the options are frozen for the run.

**Runtime observability instrument.** The canonical, non-bypassing way for a script to read these frozen options is `exec.test.options`. Its accessor is at [js/modules/k6/execution/execution.go:L184-L196]; it builds the JS object via `optionsAsObject` [js/modules/k6/execution/execution.go:L283], which `json.Marshal`s the consolidated `lib.Options` and then `JSON.parse`s it into a deeply-frozen JS object. Every experiment below reads the effective, frozen options through this instrument (`exec.test.options.scenarios`).

**The `--verbose` freeze trace (EXP7).** With `--verbose`, k6 emits its lifecycle as debug lines. The four relevant lines appear in this order, proving the scheduler only begins reading options **after** consolidation, derivation, and validation are complete:

```text
Parsing CLI flags...
Consolidating config layers...
Parsing thresholds and validating config...
Initializing the execution scheduler...
```

These are emitted at [cmd/test_load.go:L194] ("Parsing CLI flags..."), [cmd/test_load.go:L202] ("Consolidating config layers..."), [cmd/test_load.go:L208] ("Parsing thresholds and validating config..."), and [cmd/run.go:L134] ("Initializing the execution scheduler..."). The full, unedited EXP7 capture is in Section 4.

## Section 4 — Conflicting-Setup Demonstrations (EXP1–EXP7)

All runs used the canonical build **`k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)`**. Each observation script logs `exec.vu.idInTest` and `JSON.stringify(exec.test.options.scenarios)` from inside the VU, revealing the frozen, derived options actually in effect. The temporary observation scripts lived **outside** the repository under `/tmp/k6obs/` and were deleted afterward, so the repository is left unchanged.

Summary of results:

| Exp | Conflict setup | Command | Observed effective result | Winner |
|-----|----------------|---------|---------------------------|--------|
| EXP1 | script `{vus:2,duration:'1s'}` vs CLI `--vus 5 --duration 2s` | `k6 run --vus 5 --duration 2s --quiet exp1.js` | `constant-vus`, `vus:5`, `duration:"2s"`; summary `vus: 5, vus_max: 5` | **CLI** |
| EXP2 | config `{vus:7,iterations:7}` vs script `{vus:3,iterations:3}` | `k6 run --config cfg.json --quiet exp_cfg_script.js` | `shared-iterations`, `vus:3`, `iterations:3` | **Script** |
| EXP3a | config `{vus:7,iterations:7}` vs script with no options | `k6 run --config cfg.json --quiet exp_cfg_only.js` | `shared-iterations`, `vus:7`, `iterations:7` | **Config file** (over defaults) |
| EXP3b | config `{vus:7,iterations:7}` vs CLI `--vus 2 --iterations 2` | `k6 run --config cfg.json --vus 2 --iterations 2 --quiet exp_cfg_only.js` | `shared-iterations`, `vus:2`, `iterations:2` | **CLI** |
| EXP4a | env `K6_VUS=8 K6_ITERATIONS=8` vs script `{vus:2,iterations:2}` | `K6_VUS=8 K6_ITERATIONS=8 k6 run --quiet exp_env.js` | `shared-iterations`, `vus:8`, `iterations:8` | **Env** |
| EXP4b | env `K6_VUS=8 K6_ITERATIONS=8` vs CLI `--vus 2 --iterations 2` | `K6_VUS=8 K6_ITERATIONS=8 k6 run --vus 2 --iterations 2 --quiet exp_env.js` | `shared-iterations`, `vus:2`, `iterations:2` | **CLI** |
| EXP5 | script `scenarios{my_named: shared-iterations, vus:3, iterations:6}` vs CLI `--vus 4 --duration 1s` | `k6 run --vus 4 --duration 1s --quiet exp5.js` | single `default` `constant-vus`, `vus:4`, `duration:"1s"`; `my_named` replaced | **CLI** (group-reset) |
| EXP6 | script `{vus:5}` alone | `k6 run exp6.js` | warning emitted; `1 iterations for each of 1 VUs`; `iterations: 1` | `vus` ignored → default |
| EXP7 | `--verbose` freeze trace | `k6 run --verbose --quiet expmin.js` | debug lines ordered: Parsing CLI flags → Consolidating config layers → Parsing thresholds and validating config → Initializing the execution scheduler | freeze proof |

The config file `cfg.json` (used by EXP2, EXP3a, EXP3b):

```json
{
  "vus": 7,
  "iterations": 7
}
```

### EXP1 — script `{vus:2,duration:'1s'}` vs CLI `--vus 5 --duration 2s` (also the multi-VU evidence)

Script `exp1.js`:

```javascript
import exec from 'k6/execution';
import { sleep } from 'k6';

export const options = { vus: 2, duration: '1s' };

let logged = false;
export default function () {
  if (!logged) {
    console.log(`VU=${exec.vu.idInTest} scenarios=${JSON.stringify(exec.test.options.scenarios)}`);
    logged = true;
  }
  sleep(1);
}
```

Command:

```bash
k6 run --vus 5 --duration 2s --quiet exp1.js
```

Output (COMPLETE, UNEDITED — the five VU lines may appear in any order across runs, which is expected):

```text
time="2026-07-10T06:37:04Z" level=info msg="VU=5 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"2s\"}}" source=console
time="2026-07-10T06:37:04Z" level=info msg="VU=3 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"2s\"}}" source=console
time="2026-07-10T06:37:04Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"2s\"}}" source=console
time="2026-07-10T06:37:04Z" level=info msg="VU=2 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"2s\"}}" source=console
time="2026-07-10T06:37:04Z" level=info msg="VU=4 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"2s\"}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=1s min=1s med=1s max=1s p(90)=1s p(95)=1s
     iterations...........: 10  4.99324/s
     vus..................: 5   min=5     max=5
     vus_max..............: 5   min=5     max=5
```

Explanation: the script asked for `vus:2, duration:'1s'`, but the effective frozen scenario is `constant-vus` with `vus:5, duration:"2s"` — **CLI wins**. All five VUs logged the identical scenario (multi-VU consistency, see Section 5). This is the CLI-applied-last rule [cmd/config.go:L203] combined with the `duration` → `constant-vus` derivation [lib/executor/execution_config_shortcuts.go:L69-L86].

### EXP2 — config `{vus:7,iterations:7}` vs script `{vus:3,iterations:3}`

Script `exp_cfg_script.js`:

```javascript
import exec from 'k6/execution';

export const options = { vus: 3, iterations: 3 };

export default function () {
  console.log(`VU=${exec.vu.idInTest} scenarios=${JSON.stringify(exec.test.options.scenarios)}`);
}
```

Command:

```bash
k6 run --config cfg.json --quiet exp_cfg_script.js
```

Output (COMPLETE, UNEDITED):

```text
time="2026-07-10T06:37:17Z" level=info msg="VU=2 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":3,\"iterations\":3,\"maxDuration\":null}}" source=console
time="2026-07-10T06:37:17Z" level=info msg="VU=3 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":3,\"iterations\":3,\"maxDuration\":null}}" source=console
time="2026-07-10T06:37:17Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":3,\"iterations\":3,\"maxDuration\":null}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=548.13µs min=538.56µs med=549.84µs max=556µs p(90)=554.77µs p(95)=555.39µs
     iterations...........: 3   4502.672336/s
```

Explanation: the script `{vus:3,iterations:3}` beats the config file `{vus:7,iterations:7}` — **script wins** over the config file. The script (runner) options are applied above the file [cmd/config.go:L201], and `iterations` derives `shared-iterations` [lib/executor/execution_config_shortcuts.go:L56-L67].

### EXP3a — config `{vus:7,iterations:7}` vs script with no options

Script `exp_cfg_only.js`:

```javascript
import exec from 'k6/execution';

export default function () {
  console.log(`VU=${exec.vu.idInTest} scenarios=${JSON.stringify(exec.test.options.scenarios)}`);
}
```

Command:

```bash
k6 run --config cfg.json --quiet exp_cfg_only.js
```

Output (COMPLETE, UNEDITED):

```text
time="2026-07-10T06:37:17Z" level=info msg="VU=7 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console
time="2026-07-10T06:37:17Z" level=info msg="VU=4 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console
time="2026-07-10T06:37:17Z" level=info msg="VU=2 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console
time="2026-07-10T06:37:17Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console
time="2026-07-10T06:37:17Z" level=info msg="VU=5 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console
time="2026-07-10T06:37:17Z" level=info msg="VU=3 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console
time="2026-07-10T06:37:17Z" level=info msg="VU=6 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=2.23ms min=2.17ms med=2.22ms max=2.32ms p(90)=2.3ms p(95)=2.31ms
     iterations...........: 7   2809.992333/s
```

Explanation: with no script options, the config file `{vus:7,iterations:7}` takes effect — **config file wins** over the built-in defaults. The file is applied over the CLI-seeded base [cmd/config.go:L199], and `applyDefault` [cmd/config.go:L204] only fills what remains unset.

### EXP3b — config `{vus:7,iterations:7}` vs CLI `--vus 2 --iterations 2`

(Uses the same `exp_cfg_only.js`.)

Command:

```bash
k6 run --config cfg.json --vus 2 --iterations 2 --quiet exp_cfg_only.js
```

Output (COMPLETE, UNEDITED):

```text
time="2026-07-10T06:37:17Z" level=info msg="VU=2 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":2,\"iterations\":2,\"maxDuration\":null}}" source=console
time="2026-07-10T06:37:17Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":2,\"iterations\":2,\"maxDuration\":null}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=566.54µs min=555.31µs med=566.54µs max=577.77µs p(90)=575.52µs p(95)=576.65µs
     iterations...........: 2   2936.163403/s
```

Explanation: CLI `--vus 2 --iterations 2` beats the config file — **CLI wins**, because CLI flags are applied last [cmd/config.go:L203].

### EXP4a — env `K6_VUS=8 K6_ITERATIONS=8` vs script `{vus:2,iterations:2}`

Script `exp_env.js`:

```javascript
import exec from 'k6/execution';

export const options = { vus: 2, iterations: 2 };

export default function () {
  console.log(`VU=${exec.vu.idInTest} scenarios=${JSON.stringify(exec.test.options.scenarios)}`);
}
```

Command:

```bash
K6_VUS=8 K6_ITERATIONS=8 k6 run --quiet exp_env.js
```

Output (COMPLETE, UNEDITED):

```text
time="2026-07-10T06:37:32Z" level=info msg="VU=7 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":8,\"iterations\":8,\"maxDuration\":null}}" source=console
time="2026-07-10T06:37:32Z" level=info msg="VU=5 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":8,\"iterations\":8,\"maxDuration\":null}}" source=console
time="2026-07-10T06:37:32Z" level=info msg="VU=4 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":8,\"iterations\":8,\"maxDuration\":null}}" source=console
time="2026-07-10T06:37:32Z" level=info msg="VU=3 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":8,\"iterations\":8,\"maxDuration\":null}}" source=console
time="2026-07-10T06:37:32Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":8,\"iterations\":8,\"maxDuration\":null}}" source=console
time="2026-07-10T06:37:32Z" level=info msg="VU=8 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":8,\"iterations\":8,\"maxDuration\":null}}" source=console
time="2026-07-10T06:37:32Z" level=info msg="VU=6 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":8,\"iterations\":8,\"maxDuration\":null}}" source=console
time="2026-07-10T06:37:32Z" level=info msg="VU=2 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":8,\"iterations\":8,\"maxDuration\":null}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=661.58µs min=584.32µs med=656.53µs max=742.93µs p(90)=727.17µs p(95)=735.05µs
     iterations...........: 8   9536.377106/s
```

Explanation: the real environment variables `K6_VUS=8 K6_ITERATIONS=8` beat the script `{vus:2,iterations:2}` — **env wins** over the script. `readEnvConfig(gs.Env)` [cmd/config.go:L194] produces `envConf`, which is applied above the runner/script options [cmd/config.go:L203].

### EXP4b — env `K6_VUS=8 K6_ITERATIONS=8` vs CLI `--vus 2 --iterations 2`

(Uses the same `exp_env.js`.)

Command:

```bash
K6_VUS=8 K6_ITERATIONS=8 k6 run --vus 2 --iterations 2 --quiet exp_env.js
```

Output (COMPLETE, UNEDITED):

```text
time="2026-07-10T06:37:32Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":2,\"iterations\":2,\"maxDuration\":null}}" source=console
time="2026-07-10T06:37:32Z" level=info msg="VU=2 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":2,\"iterations\":2,\"maxDuration\":null}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=438.9µs min=433.89µs med=438.9µs max=443.9µs p(90)=442.9µs p(95)=443.4µs
     iterations...........: 2   3659.65903/s
```

Explanation: CLI `--vus 2 --iterations 2` beats the env vars `K6_VUS=8 K6_ITERATIONS=8` — **CLI wins** (top of the ladder), because `.Apply(cliConf)` runs last [cmd/config.go:L203].

### EXP5 — script `scenarios{my_named: shared-iterations, vus:3, iterations:6}` vs CLI `--vus 4 --duration 1s` (group-reset)

Script `exp5.js`:

```javascript
import exec from 'k6/execution';
import { sleep } from 'k6';

export const options = {
  scenarios: {
    my_named: { executor: 'shared-iterations', vus: 3, iterations: 6 },
  },
};

let logged = false;
export default function () {
  if (!logged) {
    console.log(`VU=${exec.vu.idInTest} scenarios=${JSON.stringify(exec.test.options.scenarios)}`);
    logged = true;
  }
  sleep(1);
}
```

Command:

```bash
k6 run --vus 4 --duration 1s --quiet exp5.js
```

Output (COMPLETE, UNEDITED):

```text
time="2026-07-10T06:40:59Z" level=info msg="VU=3 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":4,\"duration\":\"1s\"}}" source=console
time="2026-07-10T06:40:59Z" level=info msg="VU=2 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":4,\"duration\":\"1s\"}}" source=console
time="2026-07-10T06:40:59Z" level=info msg="VU=4 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":4,\"duration\":\"1s\"}}" source=console
time="2026-07-10T06:40:59Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":4,\"duration\":\"1s\"}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=1s min=1s med=1s max=1s p(90)=1s p(95)=1s
     iterations...........: 7   3.496957/s
     vus..................: 3   min=3      max=4
     vus_max..............: 4   min=4      max=4
```

Explanation: the script defined a **named** scenario `my_named`, but the CLI set `duration` (an execution field). Per the group-reset [lib/options.go:L371-L376], the higher (CLI) tier first cleared the lower tier's `scenarios`, so `my_named` was **replaced** by a single derived `default` `constant-vus` scenario with `vus:4, duration:"1s"` — **CLI wins via group-reset**. All four VUs are identical. See also `duration` → `constant-vus` [lib/executor/execution_config_shortcuts.go:L69-L86].

### EXP6 — script `{vus:5}` alone (a lone `vus` is ignored)

Script `exp6.js`:

```javascript
import exec from 'k6/execution';

export const options = { vus: 5 };

export default function () {
  console.log(`VU=${exec.vu.idInTest} scenarios=${JSON.stringify(exec.test.options.scenarios)}`);
}
```

Command:

```bash
k6 run exp6.js
```

Output (COMPLETE, UNEDITED — the ASCII banner and warning are included exactly as printed; this run was not `--quiet`):

```text

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

time="2026-07-10T06:41:01Z" level=warning msg="the `vus=5` option will be ignored, it only works in conjunction with `iterations`, `duration`, or `stages`"
     execution: local
        script: exp6.js
        output: -

     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 1 iterations for each of 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)

time="2026-07-10T06:41:01Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"per-vu-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":null,\"iterations\":null,\"maxDuration\":null}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=366.82µs min=366.82µs med=366.82µs max=366.82µs p(90)=366.82µs p(95)=366.82µs
     iterations...........: 1   2137.743355/s


running (00m00.0s), 0/1 VUs, 1 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.0s/10m0s  1/1 iters, 1 per VU
```

Explanation: a lone `vus:5` (without `iterations` / `duration` / `stages`) triggers the warning and is **ignored**; k6 falls back to the per-VU-iterations default of **1 VU / 1 iteration** — the scenario summary shows `* default: 1 iterations for each of 1 VUs`, the effective executor is `per-vu-iterations`, and the summary reports `iterations: 1`. See [lib/executor/execution_config_shortcuts.go:L99-L106] (the default branch and lone-`vus` warning) and [lib/executor/execution_config_shortcuts.go:L115-L119] (the per-VU default). This run was not `--quiet`, so the banner and scenario summary are shown, unedited.

### EXP7 — the `--verbose` freeze trace

Script `expmin.js`:

```javascript
export const options = { vus: 1, iterations: 1 };
export default function () {}
```

Command:

```bash
k6 run --verbose --quiet expmin.js
```

Output (COMPLETE, UNEDITED — the full debug trace):

```text
time="2026-07-10T06:41:17Z" level=debug msg="Logger format: TEXT"
time="2026-07-10T06:41:17Z" level=debug msg="k6 version: v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)"
time="2026-07-10T06:41:17Z" level=debug msg="Resolving and reading test 'expmin.js'..."
time="2026-07-10T06:41:17Z" level=debug msg=Loading... moduleSpecifier="file:///tmp/k6obs/expmin.js" originalModuleSpecifier=expmin.js
time="2026-07-10T06:41:17Z" level=debug msg="'expmin.js' resolved to 'file:///tmp/k6obs/expmin.js' and successfully loaded 80 bytes!"
time="2026-07-10T06:41:17Z" level=debug msg="Gathering k6 runtime options..."
time="2026-07-10T06:41:17Z" level=debug msg="Initializing k6 runner for 'expmin.js' (file:///tmp/k6obs/expmin.js)..."
time="2026-07-10T06:41:17Z" level=debug msg="Detecting test type for..." test_path="file:///tmp/k6obs/expmin.js"
time="2026-07-10T06:41:17Z" level=debug msg="Trying to load as a JS test..." test_path="file:///tmp/k6obs/expmin.js"
time="2026-07-10T06:41:17Z" level=debug msg="Runner successfully initialized!"
time="2026-07-10T06:41:17Z" level=debug msg="Parsing CLI flags..."
time="2026-07-10T06:41:17Z" level=debug msg="Consolidating config layers..."
time="2026-07-10T06:41:17Z" level=debug msg="Parsing thresholds and validating config..."
time="2026-07-10T06:41:17Z" level=debug msg="Initializing the execution scheduler..."
time="2026-07-10T06:41:17Z" level=debug msg="Starting 2 outputs..." component=output-manager
time="2026-07-10T06:41:17Z" level=debug msg=Starting... component=metrics-engine-ingester
time="2026-07-10T06:41:17Z" level=debug msg="Started!" component=metrics-engine-ingester
time="2026-07-10T06:41:17Z" level=debug msg="     execution: local\n        script: expmin.js\n        output: -\n\n     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):\n              * default: 1 iterations shared among 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)\n\n"
time="2026-07-10T06:41:17Z" level=debug msg="Trapping interrupt signals so k6 can handle them gracefully..."
time="2026-07-10T06:41:17Z" level=debug msg="Starting the REST API server on localhost:6565"
time="2026-07-10T06:41:17Z" level=debug msg="Starting emission of VUs and VUsMax metrics..."
time="2026-07-10T06:41:17Z" level=debug msg="Start of initialization" executorsCount=1 neededVUs=1 phase=execution-scheduler-init
time="2026-07-10T06:41:17Z" level=debug msg="Initialized VU #1" phase=execution-scheduler-init
time="2026-07-10T06:41:17Z" level=debug msg="Finished initializing needed VUs, start initializing executors..." phase=execution-scheduler-init
time="2026-07-10T06:41:17Z" level=debug msg="Initialized executor default" phase=execution-scheduler-init
time="2026-07-10T06:41:17Z" level=debug msg="Initialization completed" phase=execution-scheduler-init
time="2026-07-10T06:41:17Z" level=debug msg="Start of test run" executorsCount=1 phase=execution-scheduler-run
time="2026-07-10T06:41:17Z" level=debug msg="setup() is not defined or not exported, skipping!"
time="2026-07-10T06:41:17Z" level=debug msg="Start all executors..." phase=execution-scheduler-run
time="2026-07-10T06:41:17Z" level=debug msg="Starting executor" executor=default startTime=0s type=shared-iterations
time="2026-07-10T06:41:17Z" level=debug msg="Starting executor run..." executor=shared-iterations iterations=1 maxDuration=10m0s scenario=default type=shared-iterations vus=1
time="2026-07-10T06:41:17Z" level=debug msg="Regular duration is done, waiting for iterations to gracefully finish" executor=shared-iterations gracefulStop=30s scenario=default
time="2026-07-10T06:41:17Z" level=debug msg="Executor finished successfully" executor=default startTime=0s type=shared-iterations
time="2026-07-10T06:41:17Z" level=debug msg="teardown() is not defined or not exported, skipping!"
time="2026-07-10T06:41:17Z" level=debug msg="Test finished cleanly"
time="2026-07-10T06:41:17Z" level=debug msg="Stopping vus and vux_max metrics emission..." phase=execution-scheduler-init
time="2026-07-10T06:41:17Z" level=debug msg="Metrics emission of VUs and VUsMax metrics stopped"
time="2026-07-10T06:41:17Z" level=debug msg="Releasing signal trap..."
time="2026-07-10T06:41:17Z" level=debug msg="Sending usage report..."
time="2026-07-10T06:41:17Z" level=debug msg="Waiting for metrics and traces processing to finish..."
time="2026-07-10T06:41:17Z" level=debug msg="Metrics and traces processing finished!"
time="2026-07-10T06:41:17Z" level=debug msg="Stopping outputs..."
time="2026-07-10T06:41:17Z" level=debug msg="Stopping 2 outputs..." component=output-manager
time="2026-07-10T06:41:17Z" level=debug msg=Stopping... component=metrics-engine-ingester
time="2026-07-10T06:41:17Z" level=debug msg="Stopped!" component=metrics-engine-ingester
time="2026-07-10T06:41:17Z" level=debug msg="Generating the end-of-test summary..."

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=2.73µs min=2.73µs med=2.73µs max=2.73µs p(90)=2.73µs p(95)=2.73µs
     iterations...........: 1   6911.614277/s

time="2026-07-10T06:41:17Z" level=debug msg="Usage report sent successfully"
time="2026-07-10T06:41:17Z" level=debug msg="Everything has finished, exiting k6 normally!"
```

Explanation: the debug lines appear in the order **Parsing CLI flags… → Consolidating config layers… → Parsing thresholds and validating config… → Initializing the execution scheduler…** (emitted at [cmd/test_load.go:L194], [cmd/test_load.go:L202], [cmd/test_load.go:L208], and [cmd/run.go:L134]). This proves the scheduler only begins reading options **after** consolidation, derivation, and validation are complete — the "before vs. after" transition of the freeze point.

### The `-e`/`--env` nuance — the OBSERVED correction to the official docs

The same observation script is run three ways. It logs both the raw `__ENV.K6_ITERATIONS` (what the script sees) and `exec.test.options.scenarios.default.iterations` (the frozen effective value).

Script `nuance.js`:

```javascript
import exec from 'k6/execution';
import { sleep } from 'k6';

export const options = { vus: 2, iterations: 2 };

let logged = false;
export default function () {
  if (!logged) {
    console.log(`VU=${exec.vu.idInTest} __ENV.K6_ITERATIONS=${__ENV.K6_ITERATIONS} effective.iterations=${exec.test.options.scenarios.default.iterations}`);
    logged = true;
  }
  sleep(0.05);
}
```

Case A — `-e` flag, default `k6 run`:

```bash
k6 run -e K6_ITERATIONS=120 --quiet nuance.js
```

```text
time="2026-07-10T06:42:12Z" level=info msg="VU=1 __ENV.K6_ITERATIONS=120 effective.iterations=120" source=console
time="2026-07-10T06:42:12Z" level=info msg="VU=2 __ENV.K6_ITERATIONS=120 effective.iterations=120" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=50.45ms min=50.17ms med=50.43ms max=51.34ms p(90)=50.55ms p(95)=50.59ms
     iterations...........: 120 39.62334/s
     vus..................: 2   min=2      max=2
     vus_max..............: 2   min=2      max=2
```

Case B — `-e` flag WITH `--include-system-env-vars=false`:

```bash
k6 run -e K6_ITERATIONS=120 --include-system-env-vars=false --quiet nuance.js
```

```text
time="2026-07-10T06:42:16Z" level=info msg="VU=2 __ENV.K6_ITERATIONS=120 effective.iterations=2" source=console
time="2026-07-10T06:42:16Z" level=info msg="VU=1 __ENV.K6_ITERATIONS=120 effective.iterations=2" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=51.19ms min=51.17ms med=51.19ms max=51.22ms p(90)=51.21ms p(95)=51.22ms
     iterations...........: 2   38.911509/s
```

Case C — real environment variable:

```bash
K6_ITERATIONS=120 k6 run --quiet nuance.js
```

```text
time="2026-07-10T06:42:16Z" level=info msg="VU=1 __ENV.K6_ITERATIONS=120 effective.iterations=120" source=console
time="2026-07-10T06:42:16Z" level=info msg="VU=2 __ENV.K6_ITERATIONS=120 effective.iterations=120" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=50.39ms min=50.11ms med=50.37ms max=50.93ms p(90)=50.47ms p(95)=50.52ms
     iterations...........: 120 39.677785/s
```

Explanation (OBSERVED, authoritative; docs corroboration-only): in `__ENV`, `-e K6_ITERATIONS=120` **always** appears (all three cases show `__ENV.K6_ITERATIONS=120`). Whether it **configures** the option, however, depends on `--include-system-env-vars`:

- Case A (default, `true` for `k6 run`) → the option **is** set: `effective.iterations=120`, summary `iterations: 120`.
- Case B (`--include-system-env-vars=false`) → the option is **not** set and the **script** value wins: `effective.iterations=2`, summary `iterations: 2`.
- Case C (real env var) → always set: `effective.iterations=120`, summary `iterations: 120`.

**Root cause (code-grounded).** `k6 run` registers its runtime-option flag set with system env vars enabled by default — `flags.AddFlagSet(runtimeOptionFlagSet(true))` [cmd/run.go:L441]. In `cmd/runtime_options.go`, when `IncludeSystemEnvVars` is true the runtime options' `Env` map is aliased to the process env map (`if opts.IncludeSystemEnvVars.Bool` [cmd/runtime_options.go:L116] → `opts.Env = environment` [cmd/runtime_options.go:L117]), and each `-e VAR=value` is then written into that shared map (`opts.Env[k] = v` [cmd/runtime_options.go:L131]); so the value ends up in `gs.Env`. `getConsolidatedConfig` reads it via `readEnvConfig(gs.Env)` [cmd/config.go:L194] and applies it at the environment-variable tier [cmd/config.go:L203]. With `--include-system-env-vars=false`, `opts.Env` is a fresh map, so `-e` never reaches `gs.Env` — it only populates `__ENV` for the script and does not configure options. (The precise runtime-options line numbers were re-verified against HEAD; the exact internal reason the official docs describe only the `false` behavior is **(inferred)**.)

## Section 5 — Multi-VU Behavior

Because the scheduler builds the execution plan **once** from the single frozen `TestRunState.Options` [execution/scheduler.go:L38-L44], every VU observes an identical, globally consolidated view — the options are global to the run, not resolved per-VU.

EXP1 is the direct evidence. With `--vus 5`, all five distinct VUs logged the **identical** `constant-vus` scenario with `vus:5, duration:"2s"`. The key repeated line (one per VU, VU=1 … VU=5) is:

```text
scenarios={\"default\":{\"executor\":\"constant-vus\",...,\"vus\":5,\"duration\":\"2s\"}}
```

and the end-of-test summary confirms the run-wide values:

```text
     vus..................: 5   min=5     max=5
     vus_max..............: 5   min=5     max=5
```

EXP5 corroborates this: all four VUs reported the same single derived `default` `constant-vus` scenario. In both cases the consolidated options are **invariant across VUs and across iterations** for a given run — there is no per-VU re-resolution of configuration.

## Section 6 — Grounding in the Repository's Own Tests

The observed precedence and group-reset are independently confirmed by k6's own consolidation contract in `cmd/config_consolidation_test.go`. That test encodes the same contract as a table of `cli` / `env` / `runner` (script) / `fs` (config file) inputs with expected derived scenarios. The per-case input struct carries exactly those tiers:

```go
type opts struct {
    cli    []string
    env    []string
    runner *lib.Options
    fs     fsext.Fs
    cmds   []string
}
```

(fields `cli` / `env` / `runner` / `fs` at [cmd/config_consolidation_test.go:L121-L124]). Each case is executed through the same consolidation code path by `runTestCase` [cmd/config_consolidation_test.go:L503], and the whole table is driven by `TestConfigConsolidation` [cmd/config_consolidation_test.go:L577]. This is an independent, in-repo confirmation of the observed **CLI > env > script > config-file** precedence and the execution group-reset behavior. (This is a code-grounded reference; any claim about the test's behavior beyond its structure is **(inferred)**, as the table itself was not re-executed for this document.)

## Section 7 — Exact Build & Invocation Commands + Methodology Guardrails

**Toolchain.** The module declares `go 1.21` [go.mod:L3] and `toolchain go1.21.13` [go.mod:L5]. The canonical build used Go 1.21.13.

**Canonical build.** A normal user builds from the repository root with `go build` (producing `./k6`). The exact command used in this investigation (dependencies are vendored) was:

```bash
GOFLAGS=-mod=vendor GOTOOLCHAIN=local go build -o /tmp/k6bin/k6 .
```

**Version banner** (its commit matches HEAD `ddc3b0b1d23c…`):

```text
k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)
```

**Every `k6 run …` command used** (see Section 4 for each capture):

```bash
k6 run --vus 5 --duration 2s --quiet exp1.js                                      # EXP1
k6 run --config cfg.json --quiet exp_cfg_script.js                                # EXP2
k6 run --config cfg.json --quiet exp_cfg_only.js                                  # EXP3a
k6 run --config cfg.json --vus 2 --iterations 2 --quiet exp_cfg_only.js           # EXP3b
K6_VUS=8 K6_ITERATIONS=8 k6 run --quiet exp_env.js                                # EXP4a
K6_VUS=8 K6_ITERATIONS=8 k6 run --vus 2 --iterations 2 --quiet exp_env.js         # EXP4b
k6 run --vus 4 --duration 1s --quiet exp5.js                                      # EXP5
k6 run exp6.js                                                                    # EXP6
k6 run --verbose --quiet expmin.js                                                # EXP7
k6 run -e K6_ITERATIONS=120 --quiet nuance.js                                     # nuance A
k6 run -e K6_ITERATIONS=120 --include-system-env-vars=false --quiet nuance.js     # nuance B
K6_ITERATIONS=120 k6 run --quiet nuance.js                                        # nuance C
```

**Methodology guardrails.**

- Default/canonical build; no non-default build flags that would alter the reported values.
- Every run went through the real `k6 run` entry point — no debug hooks, mocks, fallbacks, or synthetic stand-ins.
- The in-VU reading uses the canonical `exec.test.options` instrument [js/modules/k6/execution/execution.go:L184-L196], which exposes the frozen, derived `lib.Options`.
- Complete, unedited output was captured for every condition (banner, warnings, per-VU `console.log` lines, and end-of-test summary).
- Results were confirmed stable across ≥2 repeated runs (EXP1 → `vus:5` / `vus_max:5`; EXP3a → `iterations: 7`; EXP6 → warning + `iterations: 1` reproduced identically).
- Temporary observation scripts were kept **outside** the repository under `/tmp/k6obs/` and deleted afterward, so the repository is left unchanged.
- The `time="..."` timestamps in the captures are run-specific and otherwise immaterial; anything not directly observed at runtime is labeled **(inferred)**.

## Section 8 — Coverage Pass (every named item and sub-question)

Named items:

- **VUs** — merged independently of the execution group-reset [lib/options.go:L361-L363]; observed carrying/overriding across tiers (EXP1 `vus:5`, EXP3a `vus:7`, EXP4a `vus:8`). ✅
- **duration** — `duration` → `constant-vus` derivation [lib/executor/execution_config_shortcuts.go:L69-L86]; observed CLI `duration:"2s"` winning (EXP1) and triggering the group-reset (EXP5). ✅
- **scenario settings** — explicit `scenarios` used as-is [lib/executor/execution_config_shortcuts.go:L96-L97]; observed replaced by a higher-tier execution field via the group-reset (EXP5). ✅
- **`export const options`** (script) — applied above the config file [cmd/config.go:L201]; observed beating the config (EXP2) and losing to env (EXP4a) and CLI (EXP1 / EXP3b / EXP5). ✅
- **CLI flags** — applied last / highest [cmd/config.go:L203]; observed winning over script (EXP1), config (EXP3b), and env (EXP4b). ✅
- **config file** (`--config` JSON) — applied above CLI-seeded defaults [cmd/config.go:L199]; observed winning over defaults (EXP3a) and losing to script/CLI (EXP2 / EXP3b). ✅

Sub-questions:

- **How consolidation works** — Section 2 (the `Apply` chain, group-reset, and shortcut derivation). ✅
- **When it is frozen** — Section 3 (the `TestRunState.Options` assignment [cmd/test_load.go:L280]) + the EXP7 `--verbose` trace. ✅
- **Which source wins (conflicting real runs)** — Section 4, EXP1–EXP6, confirming **CLI > env > script > config file > defaults**. ✅
- **Multi-VU behavior** — Section 5 (EXP1 VU=1 … VU=5 identical; EXP5 corroborates). ✅
- **The OBSERVED `-e`/`--env` correction vs the official docs** — Sections 1 and 4 (Cases A/B/C). ✅
