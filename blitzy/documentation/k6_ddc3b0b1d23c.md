# How k6 Consolidates Configuration Into Effective Options (branch `k6_ddc3b0b1d23c`)

This document answers, for the k6 load-testing tool (`grafana/k6`): **how does `k6 run` consolidate the script's `export const options`, CLI flags, the `--config` file, environment variables, and built-in defaults into the effective `lib.Options` the execution scheduler uses; at what point is that decision frozen; which source wins when they conflict (shown with real runs); and how does this behave with multiple VUs.** Every behavioral claim below is backed by a real `k6 run` on a canonical build **`k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)`** together with a `file:line` citation into the repository at HEAD `ddc3b0b1d23c…`. Anything not directly observed at runtime is explicitly labeled **(inferred)**.

## How to read this document

- **Observed** statements are backed by the complete, unedited command output reproduced in the fenced `text` blocks below. **(inferred)** statements are conclusions drawn from reading the source at the cited `file:line` (for example, "no source is re-read after the freeze" is a property of the traced code path, not something a single log line prints).
- **Output fidelity.** Every `text` block is the complete, unedited output of the command shown immediately above it — nothing is trimmed before or after the relevant lines. The only text removed is a leading `### CMD: …` shell annotation that the capture harness wrote (it is *not* emitted by k6). Timestamps are run-specific and harmless; VU log lines may appear in any order across runs (k6 starts VUs concurrently), which is expected and called out where relevant.
- **Commands.** Commands are shown as `k6 run …`, where `k6` is the canonical binary built in §7 and placed on `PATH` (the absolute-path form works identically). The observation scripts referenced (e.g. `exp1.js`) lived **outside** the repository in a private `mktemp -d` directory and were deleted afterward; the repository is left unchanged except for this one file.
- **Authoritative source of truth = the code plus the captured runs.** Official Grafana k6 web documentation is used only as corroboration; where it differs from observed behavior (the `-e`/`--env` case, §1.3 and §4), the observed behavior is authoritative and the discrepancy is documented.

## 1. Direct answer, precedence ladder, and the `-e`/`--env` nuance

### 1.1 Direct answer

k6 consolidates its inputs by **layering them with successive merge (`Apply`) calls**, where each higher tier overrides the lower ones; it then **derives** full scenarios from the merged execution shortcuts, and finally **freezes** the result into the run state before the scheduler reads it. The merge lives in `getConsolidatedConfig` [cmd/config.go:L189-L205], the per-field merge rule in `Options.Apply` [lib/options.go:L357-L399], the shortcut→scenario derivation in `DeriveScenariosFromShortcuts` [lib/executor/execution_config_shortcuts.go:L52-L120], and the freeze in `buildTestRunState` [cmd/test_load.go:L280].

**Effective precedence, highest → lowest:**

| Rank | Source | Where it enters the merge |
|------|--------|---------------------------|
| 1 (highest) | **CLI flags** (`--vus`, `--duration`, `--iterations`, …) | applied **last**: `.Apply(cliConf)` [cmd/config.go:L203] |
| 2 | **Environment variables** (`K6_*`) | `.Apply(envConf)` [cmd/config.go:L203], read by `readEnvConfig(gs.Env)` [cmd/config.go:L194] |
| 3 | **Script `export const options`** (the "runner" options) | `conf.Apply(Config{Options: runnerOpts})` [cmd/config.go:L201] |
| 4 | **Config file** (JSON via `--config`) | `cliConf.Apply(fileConf)` [cmd/config.go:L199] |
| 5 (lowest) | **Built-in defaults** | seeded shadow defaults from the CLI layer + `applyDefault(conf)` [cmd/config.go:L204] |

Each named item the question asks about resolves through this **same** ladder: **VUs**, **duration**, **scenario settings**, the script's **`export const options`**, **CLI flags**, and the **config file** are all merged into fields of a single `lib.Options` value, and the winner for each field is chosen by tier. (The five sources are *input tiers*; **VUs / duration / scenario settings are fields** of `lib.Options` that those tiers populate — they are not themselves tiers.) This ladder is proven end-to-end by the conflicting runs in §4.

### 1.2 Two subtleties that shape every result

- **VUs merge independently; the other execution fields reset as a group.** In `Options.Apply`, `vus` is copied on its own [lib/options.go:L361-L363], but if a higher tier sets **any** of `duration` / `iterations` / `stages` / `scenarios`, the merge first **clears all four** of those execution fields from the lower tier [lib/options.go:L371-L377] before applying the higher tier's values [lib/options.go:L379-L399]. This is why a higher tier's execution setting replaces the lower tier's *entire* execution/scenario block, while a lower tier's `vus` can still carry over (observed in EXP5 and EXP-carryover).
- **Shortcuts become scenarios.** After merging, `DeriveScenariosFromShortcuts` turns the merged shortcut fields into a full scenario: `iterations`→`shared-iterations` [L56-L67], `duration`→`constant-vus` [L69-L86], `stages`→`ramping-vus` [L88-L94], an explicit `scenarios` map is used as-is [L96-L97], and if none is set it builds a per-VU default of **1 VU / 1 iteration** [L115-L119]. A lone `vus` (≠1, with no `iterations`/`duration`/`stages`) is **ignored with a warning** [L101-L106] (observed in EXP6).

### 1.3 The `-e`/`--env` nuance (observed correction to the docs)

- **Corroboration — official docs.** The Grafana k6 documentation states that `-e`/`--env` only injects variables into the script's `__ENV` object and does **not** configure options — e.g. it documents that `-e K6_ITERATIONS=120` "does not configure the script iterations", whereas `K6_ITERATIONS=120 k6 run script.js` does set iterations. See "Environment variables" (<https://grafana.com/docs/k6/latest/using-k6/environment-variables/>) and the `-e` note in "Options reference" (<https://grafana.com/docs/k6/latest/using-k6/k6-options/reference/>). *(These pages track `latest`; the qualification below reflects the observed behavior of the pinned build `commit/ddc3b0b1d2`.)*
- **Observed (authoritative) on this canonical build.** In a **default** `k6 run`, `-e K6_ITERATIONS=120` **did** set iterations to 120 (Case A). This is because `k6 run` defaults `--include-system-env-vars=true`, which aliases the runtime-options env map onto the process env map so `-e` values reach the environment tier of the merge. With `--include-system-env-vars=false`, `-e` only populates `__ENV` and the **script** value wins (iterations = 2, Case B). A real environment variable always sets it (Case C). Root cause is traced in §4 to `cmd/run.go:L441` and `cmd/runtime_options.go:L116-L117,L131`.
- **Conclusion:** the docs' statement holds only when `--include-system-env-vars=false`; under the default `k6 run` a `-e K6_x=…` participates at the environment-variable tier. Docs are corroboration-only; the runs and code are authoritative.

### 1.4 A note on secrets (safety guidance)

All `-e`/`K6_*`/CLI examples in this document use **non-sensitive numeric configuration** (VU counts, iteration counts, durations). Values passed on the command line or through environment variables can be exposed through process listings (e.g. `ps`, `/proc/<pid>/environ`), shell history, CI job logs, and k6's own log output — k6 does not mask them. Do **not** treat CLI flags or environment variables as a general secret-delivery mechanism; deliver credentials through a dedicated secret-management facility instead. This caution applies equally to the environment demonstrations in §4.

## 2. The consolidation mechanism (documented behavior, with citations)

### 2.1 The merge pipeline — `getConsolidatedConfig` [cmd/config.go:L189-L205]

The order is documented verbatim in the comment block immediately above the function [cmd/config.go:L180-L186] and implemented by the `Apply` chain. The following is a faithful **excerpt** (Go source with error-handling and trailing validation elided as Go comments, so the block remains valid Go rather than containing bare `...` placeholders):

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
	// (error handling omitted for brevity)
	envConf, err := readEnvConfig(gs.Env)
	// (error handling omitted for brevity)

	conf = cliConf.Apply(fileConf)                 // L199: config file over CLI-seeded shadow defaults

	conf = conf.Apply(Config{Options: runnerOpts}) // L201: script `export const options` on top

	conf = conf.Apply(envConf).Apply(cliConf)      // L203: env vars, then CLI flags LAST (highest)
	conf = applyDefault(conf)                      // L204: fill a subset of still-unset defaults
	// (summary trend-stats validation omitted for brevity)
	return conf, nil
}
```

Reading the chain (each step's argument wins, per `Config.Apply` below):

1. `conf = cliConf.Apply(fileConf)` [L199] — the config file is layered over a **CLI-seeded shadow config**. The CLI layer is applied here *first* only to seed shadowed, non-`Valid` default values, so that empty CLI values do not clobber file/script/env values later.
2. `conf = conf.Apply(Config{Options: runnerOpts})` [L201] — the script's `export const options` (the "runner" options) are applied above the file.
3. `conf = conf.Apply(envConf).Apply(cliConf)` [L203] — environment variables are applied, then **the real CLI flags are applied last, giving them the greatest priority** (comment L185: "merge the user-supplied CLI flags back in on top, to give them the greatest priority"). Note the CLI config is therefore applied **twice** — once (step 1) only to seed shadow defaults, and once (here) to win.
4. `conf = applyDefault(conf)` [L204] — fills still-unset values (see §2.3).

The per-tier merge rule lives in `Config.Apply` [cmd/config.go:L71]; its doc comment [L70] reads *"The provided config has priority."* (the argument to `Apply` wins):

```go
// Apply the provided config on top of the current one, returning a new one. The provided config has priority.
func (c Config) Apply(cfg Config) Config {
	c.Options = c.Options.Apply(cfg.Options)
	// (remaining Config-level fields merged similarly)
	return c
}
```

After consolidation, `deriveAndValidateConfig` [cmd/config.go:L248] derives full scenarios (it calls `executor.DeriveScenariosFromShortcuts`, §2.4) and validates the result.

### 2.2 Per-field merge and the execution group-reset — `Options.Apply` [lib/options.go:L357-L399]

This is the exact, complete function body from the merge start through the scenarios application (verbatim — no elisions):

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
	// Still, if more than one of those options is simultaneously specified in the same
	// config tier, they will be preserved, so the validation after we've consolidated
	// all of the options can return an error.
	if opts.Duration.Valid || opts.Iterations.Valid || opts.Stages != nil || opts.Scenarios != nil {
		// TODO: emit a warning or a notice log message if overwrite lower tier config options?
		o.Duration = types.NewNullDuration(0, false)
		o.Iterations = null.NewInt(0, false)
		o.Stages = nil
		o.Scenarios = nil
	}

	if opts.Duration.Valid {
		o.Duration = opts.Duration
	}
	if opts.Iterations.Valid {
		o.Iterations = opts.Iterations
	}
	if opts.Stages != nil {
		o.Stages = []Stage{}
		for _, s := range opts.Stages {
			if s.Duration.Valid {
				o.Stages = append(o.Stages, s)
			}
		}
	}
	// o.Execution can also be populated by the duration/iterations/stages config shortcuts, but
	// that happens after the configuration from the different sources is consolidated. It can't
	// happen here, because something like `K6_ITERATIONS=10 k6 run --vus 5 script.js` wont't
	// work correctly at this level.
	if opts.Scenarios != nil {
		o.Scenarios = opts.Scenarios
	}
	// (remaining fields — ExecutionSegment, RPS, tags, TLS, DNS, etc. — merged similarly)
```

Two points, each essential to every result below:

- **VUs merge independently** [L361-L363]: `vus` is copied by itself and is **not** part of the execution reset group, so a `vus` from a lower tier can carry over even when a higher tier changes the execution mode.
- **Group-reset then application** [L371-L399]: if the higher tier sets any of `duration`/`iterations`/`stages`/`scenarios`, the merge first **clears all four** [L373-L376], and *then* applies whichever of those the higher tier actually set — `Duration` [L379-L381], `Iterations` [L382-L384], `Stages` [L385-L392], `Scenarios` [L397-L399]. The clear-then-apply pair is why a higher tier's single execution field replaces the lower tier's entire execution/scenario block (EXP5), while a lower-tier `vus` survives (EXP-carryover).

### 2.3 Where built-in defaults actually come from (terminology)

Built-in defaults enter along **two** paths, not one:

- **`applyDefault` [cmd/config.go:L222]** fills only a **subset** of fields when still unset: `SystemTags` [L223-L225], `SummaryTrendStats` [L226-L228], the `DNS` sub-fields (`TTL`/`Select`/`Policy`) [L229+], and `SetupTimeout`/`TeardownTimeout`. It does **not** set the execution defaults (`vus`/`iterations`).
- **The execution default of 1 VU / 1 iteration** comes instead from (a) the CLI-seeded shadow config in step 1 of the merge, and (b) the `default:` branch of `DeriveScenariosFromShortcuts` [lib/executor/execution_config_shortcuts.go:L115-L119], which builds a `per-vu-iterations` scenario when no execution shortcut was supplied.

(Terminology note: "CLI flags", "the config file", and "`export const options`" are configuration **input sources/tiers**; `vus`, `duration`, and the scenario settings are **fields** of the resulting `lib.Options`. The tiers populate the fields.)

### 2.4 Shortcut → scenario derivation — `DeriveScenariosFromShortcuts` [lib/executor/execution_config_shortcuts.go:L52-L120]

Faithful **excerpt** (conflict-check and warning bodies elided as Go comments to keep the block valid Go):

```go
func DeriveScenariosFromShortcuts(opts lib.Options, logger logrus.FieldLogger) (lib.Options, error) {
	result := opts

	switch {
	case opts.Iterations.Valid:
		// (conflict checks vs stages/scenarios omitted)
		result.Scenarios = getSharedIterationsScenario(opts.Iterations, opts.Duration, opts.VUs) // -> "shared-iterations"

	case opts.Duration.Valid:
		// (conflict checks vs stages/scenarios and duration>0 check omitted)
		result.Scenarios = getConstantVUsScenario(opts.Duration, opts.VUs)                        // -> "constant-vus"

	case len(opts.Stages) > 0:
		// (conflict check vs scenarios omitted)
		result.Scenarios = getRampingVUsScenario(opts.Stages, opts.VUs)                           // -> "ramping-vus"

	case len(opts.Scenarios) > 0:
		// Do nothing, scenarios was explicitly specified

	default:
		// Check if we should emit some warnings
		if opts.VUs.Valid && opts.VUs.Int64 != 1 {
			logger.Warnf(
				"the `vus=%d` option will be ignored, it only works in conjunction with `iterations`, `duration`, or `stages`",
				opts.VUs.Int64,
			)
		}
		// (empty-stages / empty-scenarios warnings omitted)
		// No execution parameters whatsoever were specified, so we'll create a per-VU iterations config
		// with 1 VU and 1 iteration.
		result.Scenarios = lib.ScenarioConfigs{
			lib.DefaultScenarioName: NewPerVUIterationsConfig(lib.DefaultScenarioName),
		}
	}
	// (validation of the derived scenarios omitted)
	return result, nil
}
```

Mapping, with citations:

- `iterations` → `shared-iterations` [L56-L67]
- `duration` → `constant-vus` [L69-L86]
- `stages` → `ramping-vus` [L88-L94]
- explicit `scenarios` → used as-is [L96-L97]
- otherwise → per-VU-iterations default of **1 VU / 1 iteration** [L115-L119]
- a lone `vus` (≠1, without `iterations`/`duration`/`stages`) is **ignored with a warning** [L101-L106]; the exact warning text is at [L103] (observed verbatim in EXP6).

## 3. Finalization: where consolidation is "frozen"

### 3.1 The finalization chain (source trace)

- The finalization entry point is `loadAndConfigureLocalTest` [cmd/test_load.go:L246], which calls `consolidateDeriveAndValidateConfig` [cmd/test_load.go:L255] — this runs `getConsolidatedConfig` (§2.1) and then `deriveAndValidateConfig` (§2.4) to produce `derivedConfig`.
- The derived options are then **reinjected into the Runner** and **written into the run state** in `buildTestRunState` [cmd/test_load.go:L265]:
  - `lct.initRunner.SetOptions(configToReinject)` [cmd/test_load.go:L268-L270] pushes the derived options back into the JS runner/bundle (this is the "Runner reinjection" step; it is what makes the frozen options visible to every VU — see §5).
  - `Options: lct.derivedConfig.Options,` [cmd/test_load.go:L280] assigns them into the `TestRunState`, annotated verbatim `// we will always run with the derived options`.
- From `cmd/run.go`: `conf := test.derivedConfig` [L127] is passed via `test.buildTestRunState(conf.Options)` [L128] and then to `execution.NewScheduler(testRunState, controller)` [L135] (the immediately preceding debug line `"Initializing the execution scheduler..."` is emitted at [cmd/run.go:L134]).
- The scheduler reads the frozen options: `options := trs.Options` [execution/scheduler.go:L39] inside `NewScheduler` [execution/scheduler.go:L38], and builds the entire execution plan from them via `options.Scenarios.GetFullExecutionRequirements(et)` [execution/scheduler.go:L44].

### 3.2 What is observed vs. inferred about the freeze

- **(inferred, from the traced code path)** After the `TestRunState.Options = lct.derivedConfig.Options` assignment [cmd/test_load.go:L280], **no source (script / CLI / config file / env) is re-read** for options — the scheduler and all VUs read the single frozen `trs.Options`. This is a property of the code path (`cmd/run.go:L127-L135` → `execution/scheduler.go:L38-L44`), not something a single log line prints; it is therefore labeled inferred. The `--verbose` trace below corroborates the *ordering* of the phases but does not by itself prove the absence of a re-read.
- **(observed)** The lifecycle **log ordering** is captured directly in EXP7: the debug lines appear in the order **`Parsing CLI flags…` → `Consolidating config layers…` → `Parsing thresholds and validating config…` → `Initializing the execution scheduler…`** (emit sites `cmd/test_load.go:L194,L202,L208` and `cmd/run.go:L134`). This shows the scheduler is initialized only **after** consolidation, derivation, and validation complete — the observable "before vs. after" of the freeze.

### 3.3 The runtime observability instrument (how the experiments read the frozen options)

Every experiment below reads the frozen options from **inside a running VU** via `exec.test.options`, the canonical, non-bypassing accessor at `js/modules/k6/execution/execution.go:L184-L196`. It builds the JS object through `optionsAsObject` [js/modules/k6/execution/execution.go:L283] by `json.Marshal`-ing the consolidated `lib.Options` [L284] and `JSON.parse`-ing it into a deep-frozen JS object [L289-L292]. Using this accessor guarantees we observe exactly the options the scheduler froze, through the real `k6 run` path (no debug hooks, mocks, or fallbacks).

The essential four ordered debug lines (full capture in EXP7):

```text
Parsing CLI flags...
Consolidating config layers...
Parsing thresholds and validating config...
Initializing the execution scheduler...
```

## 4. Conflicting-setup demonstrations (EXP1–EXP7)

All runs used the canonical build `k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)`. Each observation script logs `exec.vu.idInTest` and `JSON.stringify(exec.test.options.scenarios)` from inside the VU, revealing the frozen derived options actually in effect. Scripts lived outside the repository under a private `mktemp -d` directory (see §7) and were deleted afterward. Every output block below is complete and unedited (only the leading `### CMD:` harness annotation is removed). Where a claimed condition required stability across at least two unchanged runs, both complete captures are included (EXP1, EXP3a, EXP6).

**Summary of conflicting setups and observed winners:**

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
| EXP7 | `--verbose` freeze trace | `k6 run --verbose --quiet expmin.js` | debug order: Parsing CLI flags → Consolidating config layers → Parsing thresholds and validating config → Initializing the execution scheduler | freeze proof |
| EXP-carryover | script `{vus:6}` vs CLI `--duration 2s` (only) | `k6 run --duration 2s --quiet exp_carryover.js` | `constant-vus`, `vus:6`, `duration:"2s"`; summary `vus: 6, vus_max: 6` | duration=**CLI**, vus **carries over** from script |
| EXP-nuance | `-e` vs real env vs `--include-system-env-vars=false` | see §4 EXP-nuance | Case A/C set iterations=120; Case B keeps script's 2 | see §1.3 |

The config file `cfg.json` used by EXP2/EXP3a/EXP3b:

```json
{
  "vus": 7,
  "iterations": 7
}
```

### EXP1 — script vs CLI (also the multi-VU evidence)

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

Run 1 output (complete, unedited; the five VU lines may appear in any order across runs, which is expected):

```text
time="2026-07-10T08:27:41Z" level=info msg="VU=4 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"2s\"}}" source=console
time="2026-07-10T08:27:41Z" level=info msg="VU=2 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"2s\"}}" source=console
time="2026-07-10T08:27:41Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"2s\"}}" source=console
time="2026-07-10T08:27:41Z" level=info msg="VU=5 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"2s\"}}" source=console
time="2026-07-10T08:27:41Z" level=info msg="VU=3 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"2s\"}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=1s min=1s med=1s max=1s p(90)=1s p(95)=1s
     iterations...........: 10  4.996553/s
     vus..................: 5   min=5      max=5
     vus_max..............: 5   min=5      max=5
```

Run 2 output (same command, complete and unedited — stability confirmation; note the different VU ordering and the slightly different iteration rate, both natural run-to-run variation, while the frozen options are identical):

```text
time="2026-07-10T08:27:43Z" level=info msg="VU=5 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"2s\"}}" source=console
time="2026-07-10T08:27:43Z" level=info msg="VU=3 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"2s\"}}" source=console
time="2026-07-10T08:27:43Z" level=info msg="VU=2 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"2s\"}}" source=console
time="2026-07-10T08:27:43Z" level=info msg="VU=4 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"2s\"}}" source=console
time="2026-07-10T08:27:43Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"2s\"}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=1s min=1s med=1s max=1s p(90)=1s p(95)=1s
     iterations...........: 10  4.990503/s
     vus..................: 5   min=5      max=5
     vus_max..............: 5   min=5      max=5
```

**Winner: CLI.** The script asked for `vus:2, duration:'1s'`, but the effective frozen scenario is `constant-vus` with `vus:5, duration:"2s"` — the CLI flags won. All five distinct VUs (VU=1 … VU=5) logged the identical scenario, and the summary reports `vus: 5, vus_max: 5`. Cite the precedence chain [cmd/config.go:L203] and the `duration`→`constant-vus` derivation [lib/executor/execution_config_shortcuts.go:L69-L86]. (This run is the direct multi-VU evidence used in §5.)

### EXP2 — config file vs script

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

Output (complete, unedited):

```text
time="2026-07-10T08:27:45Z" level=info msg="VU=3 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":3,\"iterations\":3,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=2 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":3,\"iterations\":3,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":3,\"iterations\":3,\"maxDuration\":null}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=501.51µs min=493.06µs med=504.25µs max=507.22µs p(90)=506.62µs p(95)=506.92µs
     iterations...........: 3   4809.064124/s
```

**Winner: Script.** The script's `{vus:3, iterations:3}` beats the config file's `{vus:7, iterations:7}`, because the script (runner) options are applied above the file [cmd/config.go:L201]. The `iterations` shortcut derives a `shared-iterations` scenario [lib/executor/execution_config_shortcuts.go:L56-L67].

### EXP3a — config file vs built-in defaults (script has no options)

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

Run 1 output (complete, unedited):

```text
time="2026-07-10T08:27:45Z" level=info msg="VU=4 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=2 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=5 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=3 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=6 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=7 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=606.76µs min=539.63µs med=589.35µs max=681.94µs p(90)=679.3µs p(95)=680.62µs
     iterations...........: 7   8745.550701/s
```

Run 2 output (same command, complete and unedited — stability confirmation; identical frozen options, natural variation only in VU ordering and iteration rate):

```text
time="2026-07-10T08:27:45Z" level=info msg="VU=7 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=6 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=4 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=2 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=5 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=3 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":7,\"iterations\":7,\"maxDuration\":null}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=1.87ms min=1.52ms med=1.9ms max=1.98ms p(90)=1.97ms p(95)=1.97ms
     iterations...........: 7   3292.662678/s
```

**Winner: Config file (over defaults).** With no script options, the config file's `{vus:7, iterations:7}` takes effect over the built-in defaults, because the file is layered over the CLI-seeded shadow defaults [cmd/config.go:L199] and `applyDefault` [cmd/config.go:L204] only fills the subset described in §2.3 (not `vus`/`iterations`).

### EXP3b — config file vs CLI

Uses the same `exp_cfg_only.js`.

Command:

```bash
k6 run --config cfg.json --vus 2 --iterations 2 --quiet exp_cfg_only.js
```

Output (complete, unedited):

```text
time="2026-07-10T08:27:45Z" level=info msg="VU=2 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":2,\"iterations\":2,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":2,\"iterations\":2,\"maxDuration\":null}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=513.72µs min=511.98µs med=513.72µs max=515.46µs p(90)=515.11µs p(95)=515.28µs
     iterations...........: 2   2964.174981/s
```

**Winner: CLI.** The CLI `--vus 2 --iterations 2` beats the config file, because the CLI flags are applied last [cmd/config.go:L203].

### EXP4a — environment vs script

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

Output (complete, unedited):

```text
time="2026-07-10T08:27:45Z" level=info msg="VU=3 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":8,\"iterations\":8,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=8 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":8,\"iterations\":8,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=4 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":8,\"iterations\":8,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":8,\"iterations\":8,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=2 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":8,\"iterations\":8,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=5 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":8,\"iterations\":8,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=7 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":8,\"iterations\":8,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=6 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":8,\"iterations\":8,\"maxDuration\":null}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=2.1ms min=1.97ms med=2.12ms max=2.22ms p(90)=2.17ms p(95)=2.2ms
     iterations...........: 8   3405.835047/s
```

**Winner: Env.** Real environment variables `K6_VUS=8 K6_ITERATIONS=8` beat the script's `{vus:2, iterations:2}`, because `envConf` is applied above the runner/script [cmd/config.go:L203], having been read by `readEnvConfig(gs.Env)` [cmd/config.go:L194]. (Safety: as noted in §1.4, environment values are visible to process inspection and logs — use non-sensitive values here, as done.)

### EXP4b — environment vs CLI

Uses the same `exp_env.js`.

Command:

```bash
K6_VUS=8 K6_ITERATIONS=8 k6 run --vus 2 --iterations 2 --quiet exp_env.js
```

Output (complete, unedited):

```text
time="2026-07-10T08:27:45Z" level=info msg="VU=2 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":2,\"iterations\":2,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":2,\"iterations\":2,\"maxDuration\":null}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=445.63µs min=427.8µs med=445.63µs max=463.46µs p(90)=459.89µs p(95)=461.67µs
     iterations...........: 2   3598.267794/s
```

**Winner: CLI.** The CLI `--vus 2 --iterations 2` beats the environment variables — CLI sits at the top of the ladder via the final `.Apply(cliConf)` [cmd/config.go:L203].

### EXP5 — group-reset: named script scenario vs CLI shortcut

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

Output (complete, unedited):

```text
time="2026-07-10T08:27:45Z" level=info msg="VU=3 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":4,\"duration\":\"1s\"}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=2 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":4,\"duration\":\"1s\"}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=4 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":4,\"duration\":\"1s\"}}" source=console
time="2026-07-10T08:27:45Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":4,\"duration\":\"1s\"}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=1s min=1s med=1s max=1s p(90)=1s p(95)=1s
     iterations...........: 8   3.99644/s
     vus..................: 4   min=4     max=4
     vus_max..............: 4   min=4     max=4
```

**Winner: CLI, via the group-reset.** The script defined a *named* scenario `my_named`, but the CLI set `duration` (an execution field). Per the group-reset [lib/options.go:L371-L377], the higher CLI tier first cleared the lower tier's `scenarios`, then applied its own execution field [lib/options.go:L379-L399]; the derivation then produced a single `default` `constant-vus` scenario with `vus:4, duration:"1s"` [lib/executor/execution_config_shortcuts.go:L69-L86]. The named `my_named` scenario is gone — replaced wholesale. All four VUs observe the identical scenario.

### EXP6 — lone `vus` ignored with a warning

Script `exp6.js`:

```javascript
import exec from 'k6/execution';

export const options = { vus: 5 };

export default function () {
  console.log(`VU=${exec.vu.idInTest} scenarios=${JSON.stringify(exec.test.options.scenarios)}`);
}
```

Command (intentionally **without** `--quiet`, so the banner, scenario summary, and progress line are included unedited):

```bash
k6 run exp6.js
```

Run 1 output (complete, unedited):

```text

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

time="2026-07-10T08:27:47Z" level=warning msg="the `vus=5` option will be ignored, it only works in conjunction with `iterations`, `duration`, or `stages`"
     execution: local
        script: exp6.js
        output: -

     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 1 iterations for each of 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)

time="2026-07-10T08:27:47Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"per-vu-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":null,\"iterations\":null,\"maxDuration\":null}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=384.89µs min=384.89µs med=384.89µs max=384.89µs p(90)=384.89µs p(95)=384.89µs
     iterations...........: 1   2081.954039/s


running (00m00.0s), 0/1 VUs, 1 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.0s/10m0s  1/1 iters, 1 per VU
```

Run 2 output (same command, complete and unedited — stability confirmation; the warning, scenario summary, and `iterations: 1` reproduce identically, with only the timestamp and iteration-duration differing):

```text

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

time="2026-07-10T08:27:47Z" level=warning msg="the `vus=5` option will be ignored, it only works in conjunction with `iterations`, `duration`, or `stages`"
     execution: local
        script: exp6.js
        output: -

     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 1 iterations for each of 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)

time="2026-07-10T08:27:47Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"per-vu-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":null,\"iterations\":null,\"maxDuration\":null}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=395.21µs min=395.21µs med=395.21µs max=395.21µs p(90)=395.21µs p(95)=395.21µs
     iterations...........: 1   2052.250292/s


running (00m00.0s), 0/1 VUs, 1 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.0s/10m0s  1/1 iters, 1 per VU
```

**Result: `vus` ignored → per-VU default.** A lone `vus:5` (without `iterations`/`duration`/`stages`) triggers the warning `` the `vus=5` option will be ignored, it only works in conjunction with `iterations`, `duration`, or `stages` `` [lib/executor/execution_config_shortcuts.go:L101-L106], and k6 falls back to the per-VU-iterations default of **1 VU / 1 iteration** [L115-L119]. The scenario summary shows `* default: 1 iterations for each of 1 VUs`, the effective executor is `per-vu-iterations`, and the end summary shows `iterations: 1`.

**Null vs. effective default (why the JSON shows `vus:null, iterations:null` yet the run uses 1 and 1).** The per-VU default is created by `NewPerVUIterationsConfig` [lib/executor/per_vu_iterations.go:L38-L44], which sets `VUs: null.NewInt(1, false)` [L41] and `Iterations: null.NewInt(1, false)` [L42] — i.e. the underlying integer is **1** but the `Valid` flag is **false**. Because the value is not `Valid`, JSON marshaling serializes it as `null`, which is why `exec.test.options.scenarios.default` shows `"vus":null,"iterations":null` in the console line. The **effective** values the executor actually uses come from the accessors `GetVUs()` [lib/executor/per_vu_iterations.go:L51-L53], which returns `et.ScaleInt64(pvic.VUs.Int64)` (= 1 for a single local instance, where `ScaleInt64` is an identity — it returns the value unchanged when the execution-segment sequence has length 1 [lib/execution_segment.go:L734-L739]), and `GetIterations()` [L59-L60], which returns the underlying `.Int64` (= 1) — both ignoring the `Valid` flag, hence the run really executes 1 VU / 1 iteration. This is exactly the situation the source comment describes at `js/modules/k6/execution/execution.go:L281-L282`: *"When values are not set then the default value returned from JSON is used. Most of the lib.Options are Nullable types so they will be null on default."* So the `null` in the serialized object and the observed `1 VUs / 1 iteration` in the summary are consistent, not contradictory.

### EXP7 — `--verbose` freeze trace

Script `expmin.js`:

```javascript
export const options = { vus: 1, iterations: 1 };
export default function () {}
```

Command:

```bash
k6 run --verbose --quiet expmin.js
```

Output (complete, unedited — full debug trace; note the version line reports `commit/ddc3b0b1d2`, and the module paths show the private `mktemp -d` observation directory used in this session, `/tmp/k6obs.jGiEbJ/`):

```text
time="2026-07-10T08:27:47Z" level=debug msg="Logger format: TEXT"
time="2026-07-10T08:27:47Z" level=debug msg="k6 version: v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)"
time="2026-07-10T08:27:47Z" level=debug msg="Resolving and reading test 'expmin.js'..."
time="2026-07-10T08:27:47Z" level=debug msg=Loading... moduleSpecifier="file:///tmp/k6obs.jGiEbJ/expmin.js" originalModuleSpecifier=expmin.js
time="2026-07-10T08:27:47Z" level=debug msg="'expmin.js' resolved to 'file:///tmp/k6obs.jGiEbJ/expmin.js' and successfully loaded 80 bytes!"
time="2026-07-10T08:27:47Z" level=debug msg="Gathering k6 runtime options..."
time="2026-07-10T08:27:47Z" level=debug msg="Initializing k6 runner for 'expmin.js' (file:///tmp/k6obs.jGiEbJ/expmin.js)..."
time="2026-07-10T08:27:47Z" level=debug msg="Detecting test type for..." test_path="file:///tmp/k6obs.jGiEbJ/expmin.js"
time="2026-07-10T08:27:47Z" level=debug msg="Trying to load as a JS test..." test_path="file:///tmp/k6obs.jGiEbJ/expmin.js"
time="2026-07-10T08:27:47Z" level=debug msg="Runner successfully initialized!"
time="2026-07-10T08:27:47Z" level=debug msg="Parsing CLI flags..."
time="2026-07-10T08:27:47Z" level=debug msg="Consolidating config layers..."
time="2026-07-10T08:27:47Z" level=debug msg="Parsing thresholds and validating config..."
time="2026-07-10T08:27:47Z" level=debug msg="Initializing the execution scheduler..."
time="2026-07-10T08:27:47Z" level=debug msg="Starting 2 outputs..." component=output-manager
time="2026-07-10T08:27:47Z" level=debug msg=Starting... component=metrics-engine-ingester
time="2026-07-10T08:27:47Z" level=debug msg="Started!" component=metrics-engine-ingester
time="2026-07-10T08:27:47Z" level=debug msg="     execution: local\n        script: expmin.js\n        output: -\n\n     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):\n              * default: 1 iterations shared among 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)\n\n"
time="2026-07-10T08:27:47Z" level=debug msg="Trapping interrupt signals so k6 can handle them gracefully..."
time="2026-07-10T08:27:47Z" level=debug msg="Starting the REST API server on localhost:6565"
time="2026-07-10T08:27:47Z" level=debug msg="Starting emission of VUs and VUsMax metrics..."
time="2026-07-10T08:27:47Z" level=debug msg="Start of initialization" executorsCount=1 neededVUs=1 phase=execution-scheduler-init
time="2026-07-10T08:27:47Z" level=debug msg="Initialized VU #1" phase=execution-scheduler-init
time="2026-07-10T08:27:47Z" level=debug msg="Finished initializing needed VUs, start initializing executors..." phase=execution-scheduler-init
time="2026-07-10T08:27:47Z" level=debug msg="Initialized executor default" phase=execution-scheduler-init
time="2026-07-10T08:27:47Z" level=debug msg="Initialization completed" phase=execution-scheduler-init
time="2026-07-10T08:27:47Z" level=debug msg="Start of test run" executorsCount=1 phase=execution-scheduler-run
time="2026-07-10T08:27:47Z" level=debug msg="setup() is not defined or not exported, skipping!"
time="2026-07-10T08:27:47Z" level=debug msg="Start all executors..." phase=execution-scheduler-run
time="2026-07-10T08:27:47Z" level=debug msg="Starting executor" executor=default startTime=0s type=shared-iterations
time="2026-07-10T08:27:47Z" level=debug msg="Starting executor run..." executor=shared-iterations iterations=1 maxDuration=10m0s scenario=default type=shared-iterations vus=1
time="2026-07-10T08:27:47Z" level=debug msg="Regular duration is done, waiting for iterations to gracefully finish" executor=shared-iterations gracefulStop=30s scenario=default
time="2026-07-10T08:27:47Z" level=debug msg="Executor finished successfully" executor=default startTime=0s type=shared-iterations
time="2026-07-10T08:27:47Z" level=debug msg="teardown() is not defined or not exported, skipping!"
time="2026-07-10T08:27:47Z" level=debug msg="Test finished cleanly"
time="2026-07-10T08:27:47Z" level=debug msg="Stopping vus and vux_max metrics emission..." phase=execution-scheduler-init
time="2026-07-10T08:27:47Z" level=debug msg="Metrics emission of VUs and VUsMax metrics stopped"
time="2026-07-10T08:27:47Z" level=debug msg="Releasing signal trap..."
time="2026-07-10T08:27:47Z" level=debug msg="Sending usage report..."
time="2026-07-10T08:27:47Z" level=debug msg="Waiting for metrics and traces processing to finish..."
time="2026-07-10T08:27:47Z" level=debug msg="Metrics and traces processing finished!"
time="2026-07-10T08:27:47Z" level=debug msg="Stopping outputs..."
time="2026-07-10T08:27:47Z" level=debug msg="Stopping 2 outputs..." component=output-manager
time="2026-07-10T08:27:47Z" level=debug msg=Stopping... component=metrics-engine-ingester
time="2026-07-10T08:27:47Z" level=debug msg="Stopped!" component=metrics-engine-ingester
time="2026-07-10T08:27:47Z" level=debug msg="Generating the end-of-test summary..."

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=2.14µs min=2.14µs med=2.14µs max=2.14µs p(90)=2.14µs p(95)=2.14µs
     iterations...........: 1   6846.595188/s

time="2026-07-10T08:27:47Z" level=debug msg="Usage report sent successfully"
time="2026-07-10T08:27:47Z" level=debug msg="Everything has finished, exiting k6 normally!"
```

**(observed)** The four consolidation/scheduler debug lines appear in the order `Parsing CLI flags…` → `Consolidating config layers…` → `Parsing thresholds and validating config…` → `Initializing the execution scheduler…` (emit sites `cmd/test_load.go:L194,L202,L208` and `cmd/run.go:L134`). **(inferred)** Combined with the source trace in §3.1, this shows the scheduler begins reading options only after consolidation, derivation, and validation are complete; the absence of any later re-read is a property of the code path (`cmd/run.go:L127-L135` → `execution/scheduler.go:L38-L44`), not of any single log line.

### EXP-carryover — lower-tier `vus` carries over under a higher-tier execution field

This isolates the independent-VU merge: the **lower tier (script) sets only `vus`** (no `duration`/`iterations`/`stages`/`scenarios`), while the **higher tier (CLI) sets only `duration`**. If the group-reset also cleared `vus`, the script's `vus` would vanish; because `vus` merges independently [lib/options.go:L361-L363], it must carry over.

Script `exp_carryover.js`:

```javascript
import exec from 'k6/execution';
import { sleep } from 'k6';

// Lower tier (script) sets ONLY vus; no duration/iterations/stages/scenarios.
export const options = { vus: 6 };

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
k6 run --duration 2s --quiet exp_carryover.js
```

Output (complete, unedited):

```text
time="2026-07-10T08:27:54Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":6,\"duration\":\"2s\"}}" source=console
time="2026-07-10T08:27:54Z" level=info msg="VU=3 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":6,\"duration\":\"2s\"}}" source=console
time="2026-07-10T08:27:54Z" level=info msg="VU=5 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":6,\"duration\":\"2s\"}}" source=console
time="2026-07-10T08:27:54Z" level=info msg="VU=2 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":6,\"duration\":\"2s\"}}" source=console
time="2026-07-10T08:27:54Z" level=info msg="VU=6 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":6,\"duration\":\"2s\"}}" source=console
time="2026-07-10T08:27:54Z" level=info msg="VU=4 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":6,\"duration\":\"2s\"}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=1s min=1s med=1s max=1s p(90)=1s p(95)=1s
     iterations...........: 12  5.995007/s
     vus..................: 6   min=6      max=6
     vus_max..............: 6   min=6      max=6
```

**Result: `duration` from CLI, `vus` carried over from the script.** The effective scenario is `constant-vus` with `duration:"2s"` (from the CLI's `--duration`, which set the execution mode and triggered the group-reset of the execution block) **and** `vus:6` (which came from the script's lower tier and survived because `vus` is merged independently [lib/options.go:L361-L363], outside the reset group). The summary confirms `vus: 6, vus_max: 6`, and all six VUs observe the identical scenario. This is the runtime proof of the carry-over behavior described in §1.2 and §2.2. Note the contrast with EXP6: a lone `vus` with *no* execution field anywhere is ignored, but here the CLI supplies `duration`, so a `constant-vus` scenario exists for the carried-over `vus:6` to populate.

### EXP-nuance — the `-e`/`--env` nuance (Case A vs B vs C)

This demonstrates the observed correction to the official docs (§1.3). Script `nuance.js` logs both `__ENV.K6_ITERATIONS` (what the script sees) and `exec.test.options.scenarios.default.iterations` (the frozen effective option):

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

**Case A — `-e` flag, default `k6 run`:**

```bash
k6 run -e K6_ITERATIONS=120 --quiet nuance.js
```

```text
time="2026-07-10T08:27:47Z" level=info msg="VU=1 __ENV.K6_ITERATIONS=120 effective.iterations=120" source=console
time="2026-07-10T08:27:47Z" level=info msg="VU=2 __ENV.K6_ITERATIONS=120 effective.iterations=120" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=50.36ms min=50.14ms med=50.36ms max=50.68ms p(90)=50.44ms p(95)=50.48ms
     iterations...........: 120 39.695744/s
     vus..................: 2   min=2       max=2
     vus_max..............: 2   min=2       max=2
```

**Case B — `-e` flag WITH `--include-system-env-vars=false`:**

```bash
k6 run -e K6_ITERATIONS=120 --include-system-env-vars=false --quiet nuance.js
```

```text
time="2026-07-10T08:27:50Z" level=info msg="VU=2 __ENV.K6_ITERATIONS=120 effective.iterations=2" source=console
time="2026-07-10T08:27:50Z" level=info msg="VU=1 __ENV.K6_ITERATIONS=120 effective.iterations=2" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=51.11ms min=51.1ms med=51.11ms max=51.12ms p(90)=51.11ms p(95)=51.12ms
     iterations...........: 2   39.062038/s
```

**Case C — a real environment variable (complete, unedited — including the `vus`/`vus_max` summary lines):**

```bash
K6_ITERATIONS=120 k6 run --quiet nuance.js
```

```text
time="2026-07-10T08:27:50Z" level=info msg="VU=1 __ENV.K6_ITERATIONS=120 effective.iterations=120" source=console
time="2026-07-10T08:27:50Z" level=info msg="VU=2 __ENV.K6_ITERATIONS=120 effective.iterations=120" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=50.39ms min=50.26ms med=50.37ms max=50.94ms p(90)=50.45ms p(95)=50.46ms
     iterations...........: 120 39.680234/s
     vus..................: 2   min=2       max=2
     vus_max..............: 2   min=2       max=2
```

**Reading the three cases (observed; docs corroboration-only).** In `__ENV`, `-e K6_ITERATIONS=120` always appears (all three cases show `__ENV.K6_ITERATIONS=120`). But whether it **configures the option** depends on `--include-system-env-vars`:

- **Case A** (default `k6 run`, so `--include-system-env-vars=true`) → the option **is** set: `effective.iterations=120`, summary `iterations: 120`.
- **Case B** (`--include-system-env-vars=false`) → the option is **not** set; the **script** value wins: `effective.iterations=2`, summary `iterations: 2`.
- **Case C** (a real environment variable) → always sets it: `effective.iterations=120`, summary `iterations: 120`.

**Root cause (code-grounded).** `k6 run` registers its runtime-option flag set with system env vars enabled by default — `flags.AddFlagSet(runtimeOptionFlagSet(true))` [cmd/run.go:L441]. In `cmd/runtime_options.go`, when `IncludeSystemEnvVars` is true the runtime options' `Env` map is aliased to the process environment map — `if opts.IncludeSystemEnvVars.Bool { opts.Env = environment }` [cmd/runtime_options.go:L116-L117] — and each `-e VAR=value` is then written into that shared map — `opts.Env[k] = v` [cmd/runtime_options.go:L131]. So the `-e` value ends up in `gs.Env`, which `getConsolidatedConfig` reads via `readEnvConfig(gs.Env)` [cmd/config.go:L194] and applies at the env tier [cmd/config.go:L203]. With `--include-system-env-vars=false`, `opts.Env` is a fresh map (not aliased to the process env), so `-e` only populates `__ENV` for the script and never reaches the option merge. This is why Case A behaves like the environment tier while Case B leaves the script value to win.

Comparison with the official documentation: the docs say `-e K6_ITERATIONS=120` "does not configure the script iterations" (see §1.3 links). That is accurate **only** under `--include-system-env-vars=false` (Case B); under the default `k6 run` (Case A) the `-e` value *does* configure it. The observed runs are authoritative; the docs are corroboration and describe the `false` case.

## 5. Multi-VU behavior

**(inferred, from the traced code path)** The scheduler builds the execution plan **once** from the single frozen `TestRunState.Options` — `options := trs.Options` [execution/scheduler.go:L39] inside `NewScheduler` [execution/scheduler.go:L38], with the plan derived at `options.Scenarios.GetFullExecutionRequirements(et)` [execution/scheduler.go:L44]. Each VU's runtime state is populated with the same reinjected options — `vu.state = &lib.State{ … Options: vu.Runner.Bundle.Options … }` [js/runner.go:L230-L232] — and the in-VU accessor returns that same object [js/modules/k6/execution/execution.go:L184-L196]. So options are **global to the run, not resolved per VU**; there is no per-VU re-consolidation.

**(observed) Identical across VUs.** In EXP1 (`--vus 5`), all five distinct VUs logged byte-identical frozen scenarios and the summary reported `vus: 5, vus_max: 5`. The exact VU=1 line from EXP1 run 1 (one complete line, quoted verbatim — the other four VU lines differ only in the `VU=<n>` label):

```text
time="2026-07-10T08:27:41Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"2s\"}}" source=console
```

**(observed) Identical across iterations *and* VUs.** To exercise invariance across multiple iterations (not just the first), `exp_iter.js` logs on **every** iteration, tagging each line with the VU id and the in-test iteration index `exec.scenario.iterationInTest`:

```javascript
import exec from 'k6/execution';

export const options = { vus: 3, iterations: 9 };

export default function () {
  console.log(`VU=${exec.vu.idInTest} iterInTest=${exec.scenario.iterationInTest} scenarios=${JSON.stringify(exec.test.options.scenarios)}`);
}
```

Command:

```bash
k6 run --quiet exp_iter.js
```

Output (complete, unedited — nine lines, one per iteration, spanning all three VUs and iteration indices 0–8):

```text
time="2026-07-10T08:27:56Z" level=info msg="VU=1 iterInTest=2 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":3,\"iterations\":9,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:56Z" level=info msg="VU=2 iterInTest=1 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":3,\"iterations\":9,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:56Z" level=info msg="VU=3 iterInTest=0 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":3,\"iterations\":9,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:56Z" level=info msg="VU=1 iterInTest=3 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":3,\"iterations\":9,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:56Z" level=info msg="VU=2 iterInTest=4 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":3,\"iterations\":9,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:56Z" level=info msg="VU=3 iterInTest=5 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":3,\"iterations\":9,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:56Z" level=info msg="VU=1 iterInTest=6 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":3,\"iterations\":9,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:56Z" level=info msg="VU=2 iterInTest=7 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":3,\"iterations\":9,\"maxDuration\":null}}" source=console
time="2026-07-10T08:27:56Z" level=info msg="VU=3 iterInTest=8 scenarios={\"default\":{\"executor\":\"shared-iterations\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":3,\"iterations\":9,\"maxDuration\":null}}" source=console

     data_received........: 0 B 0 B/s
     data_sent............: 0 B 0 B/s
     iteration_duration...: avg=759.67µs min=240.14µs med=480.54µs max=1.54ms p(90)=1.53ms p(95)=1.54ms
     iterations...........: 9   3614.810602/s
```

Every one of the nine `(VU, iterInTest)` observations reports the identical frozen scenario `shared-iterations` with `vus:3, iterations:9`. This directly demonstrates that the consolidated options are invariant across both VUs and iterations for a given run, corroborating the source-level conclusion above. EXP5 (all four VUs identical) and EXP-carryover (all six VUs identical) further corroborate the cross-VU invariance.

## 6. Grounding in the repository's own consolidation test

The observed precedence and group-reset are cross-checked against the repository's own consolidation contract in `cmd/config_consolidation_test.go`. That test encodes the **same** contract as a table of `cli` / `env` / `runner` (script) / `fs` (config file) inputs with expected derived scenarios. The per-case input struct `opts` declares those consolidation-input fields — `cli []string` [cmd/config_consolidation_test.go:L121], `env []string` [L122], `runner *lib.Options` [L123], `fs fsext.Fs` [L124] (plus a `cmds []string` field [L125] for the subcommand, which is not a consolidation input); `runTestCase` [cmd/config_consolidation_test.go:L503] executes each case through the same consolidation code path exercised by `k6 run`; and `TestConfigConsolidation` [cmd/config_consolidation_test.go:L577] runs the whole table. This is an independent, in-repository confirmation of the CLI > env > script > config-file precedence and the execution group-reset that the runs above demonstrate. **(inferred)** Beyond the struct/function structure cited here (verified by reading), the specific per-case expectations are not re-executed in this document; they are referenced as a code-grounded corroboration.

## 7. Exact build and invocation commands + methodology guardrails

### 7.1 Toolchain

The module declares `go 1.21` [go.mod:L3] and `toolchain go1.21.13` [go.mod:L5]. The canonical binary was built with Go 1.21.13.

### 7.2 Canonical, reproducible build (pinned commit)

A normal user building from a checkout of this commit runs `go build` from the repository root, producing `./k6`. **Important reproducibility note:** the version banner embeds the VCS revision via `FullVersion()` [lib/consts/consts.go:L16-L52], which reads `vcs.revision` from `debug.ReadBuildInfo()` and uses its first 10 characters [lib/consts/consts.go:L30-L35], appending `-dirty` if `vcs.modified` is `true` [L36-L50]. Go's VCS stamping requires a real `.git` **directory** at the module root; it does not stamp when building from a linked worktree (where `.git` is a file) or from an unstamped tree. To reproduce the exact `commit/ddc3b0b1d2` banner deterministically, build from a **clean detached checkout at the full commit** in a private temporary directory (dependencies are vendored, so the build is fully offline):

```bash
# 1. Private temp dirs (never inside the repository)
K6SRC="$(mktemp -d)"          # clean source checkout
K6BIN="$(mktemp -d)"          # built binary
K6OBS="$(mktemp -d)"          # observation scripts (see 7.3)

# 2. Clean checkout pinned to the exact commit, then build canonically
git clone --quiet /path/to/k6 "$K6SRC/k6"
cd "$K6SRC/k6"
git checkout --quiet --detach ddc3b0b1d23c128e34e2792fc9075f9126e32375
GOFLAGS=-mod=vendor GOTOOLCHAIN=local CGO_ENABLED=1 go build -o "$K6BIN/k6" .

# 3. Put the built binary on PATH so `k6` resolves to it (absolute path works too)
export PATH="$K6BIN:$PATH"
```

Produced version banner (confirm the commit matches HEAD `ddc3b0b1d23c…`):

```text
k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)
```

### 7.3 Running the observation scripts

The observation scripts live in the private `mktemp -d` directory `"$K6OBS"` (outside the repository). Run each experiment from that directory so the relative script names resolve:

```bash
cd "$K6OBS"
# ... write exp1.js, exp_cfg_only.js, cfg.json, etc. here ...
k6 run --vus 5 --duration 2s --quiet exp1.js
```

Every `k6 run …` command shown in §4 and §5 was executed this way (with `k6` resolving to `"$K6BIN/k6"` via the exported `PATH`). In the captured `--verbose` trace (EXP7) the module path therefore appears as `file:///tmp/k6obs.<random>/…` — the `mktemp -d` directory from this session — which is expected.

### 7.4 Cleanup (repository left unchanged)

After capturing the output, remove the temporary directories so nothing is left behind, and confirm the repository is unmodified:

```bash
rm -rf "$K6SRC" "$K6BIN" "$K6OBS"
# also remove any ./k6 that a repo-root `go build` may have produced (it is .gitignored but should not linger)
```

### 7.5 Methodology guardrails

- **Canonical/default build**: default `go build` with the pinned Go 1.21.13 toolchain and vendored dependencies; no non-default build flags that would alter reported values.
- **Real entry point only**: every run goes through the real `k6 run` command; the in-VU reads use the canonical `exec.test.options` instrument [js/modules/k6/execution/execution.go:L184-L196]. No debug hooks, mocks, fallbacks, or synthetic stand-ins were used.
- **Complete, unedited output** is included for every condition (only the leading `### CMD:` harness annotation was stripped; it is not k6 output).
- **Stability across ≥2 runs**: EXP1 (`vus:5`/`vus_max:5`), EXP3a (`iterations: 7`), and EXP6 (warning + `iterations: 1`) were each captured over two unchanged runs with identical frozen options; natural run-to-run variation (timestamps, VU start ordering, iteration rate) is expected and shown as captured.
- **(inferred) labels**: statements not directly observed at runtime (no re-read after the freeze; scheduler read timing; the internal reason the docs differ) are labeled **(inferred)** and tied to the cited `file:line` code path.
- **Read-only repository**: this document is the only file added; observation scripts stayed outside the repository and were deleted.

## 8. Coverage pass — every named item and sub-question

**Named items:**

- **VUs** — merged independently of the execution reset group [lib/options.go:L361-L363]; observed overriding across tiers (EXP1 `vus:5`, EXP3a `vus:7`, EXP4a `vus:8`) and **carrying over** under a higher-tier execution field (EXP-carryover `vus:6`). ✅
- **duration** — `duration`→`constant-vus` derivation [lib/executor/execution_config_shortcuts.go:L69-L86]; observed CLI `duration:"2s"` winning (EXP1) and triggering the group-reset (EXP5, EXP-carryover). ✅
- **scenario settings** — explicit `scenarios` used as-is [lib/executor/execution_config_shortcuts.go:L96-L97]; observed replaced wholesale by a higher-tier execution field via the group-reset (EXP5). ✅
- **`export const options`** (script) — applied above the config file [cmd/config.go:L201]; observed beating the config file (EXP2), losing to env (EXP4a) and CLI (EXP1/EXP3b/EXP5), and contributing a carried-over `vus` (EXP-carryover). ✅
- **CLI flags** — applied last/highest [cmd/config.go:L203]; observed winning over script (EXP1), config file (EXP3b), and env (EXP4b). ✅
- **config file** (`--config` JSON) — applied above the CLI-seeded shadow defaults [cmd/config.go:L199]; observed winning over defaults (EXP3a) and losing to script/CLI (EXP2/EXP3b). ✅

**Sub-questions:**

- **How does consolidation work?** Layered `Apply` merges with the argument winning, then shortcut→scenario derivation — §2, proven by §4. ✅
- **When is it frozen?** At `TestRunState.Options = derivedConfig.Options` [cmd/test_load.go:L280], consumed by the scheduler [execution/scheduler.go:L38-L44]; the `--verbose` ordering is observed (EXP7) and the no-re-read property is inferred from the traced path — §3. ✅
- **Which source wins (conflicting real runs)?** CLI > env > script > config file > defaults, demonstrated by EXP1–EXP6, EXP-carryover, and EXP-nuance — §4. ✅
- **Multiple VUs?** All VUs (and all iterations) observe the identical, globally consolidated options — EXP1 (5 VUs), `exp_iter.js` (3 VUs × 9 iterations), EXP5 (4 VUs), EXP-carryover (6 VUs) — §5. ✅
- **`-e`/`--env` correction vs official docs** — observed authoritative behavior documented against the corroborating docs — §1.3, §4 EXP-nuance. ✅

**Epistemic summary:** the precedence ladder, the winners in every conflict, the group-reset, the lone-`vus` warning, the null-vs-effective-default reconciliation, the multi-VU/iteration invariance, and the `-e`/`--env` behavior are all **observed** with complete captured output. The "no source is re-read after the freeze", the scheduler read-timing conclusion, and the internal reason the docs differ are **(inferred)** from the cited code path and labeled as such.
