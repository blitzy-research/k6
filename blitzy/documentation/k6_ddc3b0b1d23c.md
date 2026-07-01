# How k6 consolidates its options into the single effective set the scheduler uses

**Codebase:** `go.k6.io/k6` (module declared at `go.mod:1`)
**Version under test:** `const Version = "0.55.0"` — `lib/consts/consts.go:12`
**Source commit under investigation:** `ddc3b0b1d23c128e34e2792fc9075f9126e32375` — every k6 source file cited below is read at this commit, and the run-first build was performed against it. A binary built from this exact source tree self-reports `k6 v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)`; `commit/ddc3b0b1d2` is this commit's own hash truncated to 10 characters.
**Delivered documentation branch HEAD:** `ef05fa4fbe4ca99475ea360b105f758f10650077` — the branch HEAD *after* this answer document is committed on top of the source commit (and advanced again by any later correction to this file). Every such commit adds **only** `blitzy/documentation/k6_ddc3b0b1d23c.md` and changes **no** source code, so a binary rebuilt at this HEAD self-reports the same version with a different embedded commit (`commit/ef05fa4fbe`) and byte-for-byte identical behaviour. Because the `commit/` field simply mirrors the git commit built (see [Build and version check](#build-and-version-check)), the source commit above — not the moving branch HEAD — is the authoritative anchor for the version self-report quoted throughout this document.

This document was written **run-first**: the k6 binary was built from the vendored
sources in this repository, a set of conflicting configurations was executed, and the
**verbatim** program output was captured. Every behavioural claim below is backed by
either an exact `file:line` source citation or a captured run (or both). Where the
official documentation and the observed behaviour disagree, the **observed behaviour of
this build is treated as authoritative** and the disagreement is called out explicitly
(see [§7](#7-the--e--env-reality-vs-the-docs-claim-g1g3)).

### Build and version check

The binary under test was built **from the vendored sources** with the exact command
below — this is the run-first build step referenced above. `go build` prints nothing on
success (exit `0`); `/tmp/k6bin version` then prints the self-report. The self-report
quoted throughout this document is the one produced at the **source commit under
investigation** (the state of every source file cited below); the delivered-branch build is
shown alongside it below so the two can be compared directly:

**Build, then version check:**
```
GOTOOLCHAIN=local GOFLAGS=-mod=vendor CGO_ENABLED=0 go build -o /tmp/k6bin .
/tmp/k6bin version
```

**Observed (verbatim) at the source commit `ddc3b0b1d23c…` — printed by `/tmp/k6bin version`, exit `0`:**
```
k6bin v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)
```

This is the self-report quoted throughout the rest of this document, because it is the
build of the exact source tree that every citation below refers to. Re-running the **same**
build command from the delivered documentation branch — whose HEAD sits one or more
docs-only commits ahead of the source commit — self-reports the identical version and
toolchain but a different embedded commit hash:

**Observed (verbatim) at delivered-branch HEAD `ef05fa4fbe…` — printed by `/tmp/k6bin version`, exit `0`:**
```
k6bin v0.55.0 (commit/ef05fa4fbe, go1.23.12, linux/amd64)
```

**Why these flags, and what the output means:**

- `GOTOOLCHAIN=local` pins the installed Go toolchain (no auto-download); `GOFLAGS=-mod=vendor`
  forces an **offline** build from the committed `vendor/` tree; `CGO_ENABLED=0` produces a
  static binary. The `Makefile` `build` target is itself a plain `go build` (`Makefile:7-8`)
  and every dependency is vendored, so the build needs no network access.
- The version line is produced by `FullVersion()` (`lib/consts/consts.go:16`) rendered
  through the root command's version template (`cmd/root.go:54-56`). The leading `k6bin` is
  the binary's own name — `gs.BinaryName`, set to `filepath.Base(os.Executable())`
  (`cmd/state/state.go:101`, `:112`) and consumed at `cmd/root.go:45` — which is why building
  to `-o /tmp/k6bin` self-reports as `k6bin`, not `k6`.
- The `commit/…` field is `vcs.revision` — the git commit at build time — truncated to its
  first 10 characters (`lib/consts/consts.go:30-35`). Because `vcs.revision` is simply
  whatever the build's `HEAD` points at, the field tracks the commit that was built, **not**
  any change in behaviour: building the **source commit under investigation**
  `ddc3b0b1d23c128e34e2792fc9075f9126e32375` yields `commit/ddc3b0b1d2`, while building the
  **delivered documentation branch** (HEAD `ef05fa4fbe4ca99475ea360b105f758f10650077` — the
  source commit plus docs-only commits that add only this file) yields `commit/ef05fa4fbe`.
  Run `git rev-parse HEAD` to see the current value; it is always a docs-only descendant of
  the source commit, so the two binaries are behaviourally identical. Each build here was
  clean (`vcs.modified=false`), so no `-dirty` suffix is appended (`lib/consts/consts.go:48-50`)
  — had the working tree carried uncommitted edits the hash would gain a `-dirty` suffix.
  `go1.23.12, linux/amd64` is `runtime.Version()` with `GOOS`/`GOARCH` (`lib/consts/consts.go:17`).

---

## Table of contents

1. [TL;DR](#1-tldr)
2. [How the options are consolidated (`getConsolidatedConfig`)](#2-how-the-options-are-consolidated-getconsolidatedconfig)
3. [Where each layer's values originate](#3-where-each-layers-values-originate)
4. [Shortcut → scenario derivation and defaults](#4-shortcut--scenario-derivation-and-defaults)
5. [When the decision becomes final (the freeze point)](#5-when-the-decision-becomes-final-the-freeze-point)
6. [Proof — six real conflicting runs (A–F)](#6-proof--six-real-conflicting-runs-af)
7. [The `-e`/`--env` reality vs the docs claim (G1–G3)](#7-the--e--env-reality-vs-the-docs-claim-g1g3)
8. [Multi-VU behavior via `exec.test.options`](#8-multi-vu-behavior-via-exectestoptions)
9. [Web corroboration](#9-web-corroboration)
10. [Coverage pass (R1–R4)](#10-coverage-pass-r1r4)

---

## 1. TL;DR

**The precedence ladder — highest wins:**

> **CLI flags  >  `K6_*` environment variables  >  script `export const options`  >  JSON config file (`--config`)  >  built-in defaults**

All five input layers are merged in one function, `getConsolidatedConfig`
(`cmd/config.go:189`), which applies them in a fixed order so that the highest-priority
layer is applied **last** and therefore wins. The merged result is then derived, validated,
**frozen** in `buildTestRunState` (`cmd/test_load.go:265`), and handed to
`execution.NewScheduler` (`cmd/run.go:135`). At runtime every VU reads a copy of that one
frozen `Options` value via `exec.test.options`
(`js/modules/k6/execution/execution.go:184`), so **all VUs observe identical effective
options**.

**Two counterintuitive takeaways (both reproduced below):**

1. **The script `export const options` beats the `--config` JSON file.** The config file
   is second-lowest in precedence (just above defaults), so a value in the script overrides
   the same value in `--config`. Proven by **Experiment D**: `--config cfg.json`
   (`{"vus":7,"duration":"7s"}`) plus a script declaring `{ vus: 2, duration: '3s' }`
   runs as `* default: 2 looping VUs for 3s (gracefulStop: 30s)` — the **script's**
   `2`/`3s`, not the file's `7`/`7s`.
2. **A CLI `--duration` erases a script-defined `scenarios` map** rather than merging with
   it. This is why "scenario settings win from a different place." Proven by
   **Experiment E**: a script with a named `scenarios.my_named_scenario` run with
   `--duration 1s` produces a synthetic `default` scenario and discards the named one.

The mechanical reason for both is the same single rule: **`Config.Apply(x)` gives `x`
priority** (`cmd/config.go:70`), so **whatever is applied last wins**, and when a
higher tier sets any execution field, `Options.Apply` first **zeroes** the lower tier's
execution settings (`lib/options.go:365-377`) before applying the incoming ones.

---

## 2. How the options are consolidated (`getConsolidatedConfig`)

### 2.1 The single merge point

Every option layer is combined in exactly one function:

```
func getConsolidatedConfig(gs *state.GlobalState, cliConf Config, runnerOpts lib.Options) (conf Config, err error) {
```
— `cmd/config.go:189`

Its leading doc-comment (`cmd/config.go:180-186`) states the intended order in six
bullets, verbatim:

```go
// Assemble the final consolidated configuration from all of the different sources:
// - start with the CLI-provided options to get shadowed (non-Valid) defaults in there
// - add the global file config options
// - add the Runner-provided options (they may come from Bundle too if applicable)
// - add the environment variables
// - merge the user-supplied CLI flags back in on top, to give them the greatest priority
// - set some defaults if they weren't previously specified
```
— `cmd/config.go:180-186`

### 2.2 The merge sequence *is* the mechanism

The body reads the two external file/environment sources and then applies the layers in a
fixed sequence (`cmd/config.go:190-204`):

```go
fileConf, err := readDiskConfig(gs)        // :190  read the JSON config file  (FILE tier)
...
envConf, err := readEnvConfig(gs.Env)      // :194  read K6_* env vars         (ENV tier)
...
conf = cliConf.Apply(fileConf)             // :199  start from CLI struct, apply FILE on top
conf = conf.Apply(Config{Options: runnerOpts}) // :201  apply SCRIPT tier (runner options)
conf = conf.Apply(envConf).Apply(cliConf)  // :203  apply ENV, then CLI LAST (highest priority)
conf = applyDefault(conf)                  // :204  fill built-in DEFAULTS for anything still unset
```

### 2.3 Why "last applied wins"

`Config.Apply` is documented at `cmd/config.go:70` and defined at `cmd/config.go:71`:

```go
// Apply the provided config on top of the current one, returning a new one. The provided config has priority.
func (c Config) Apply(cfg Config) Config {
```
— `cmd/config.go:70-71`

Because **"the provided config has priority,"** `a.Apply(b)` lets `b` win. Reading the
sequence above from that rule gives the ladder:

| Order applied | Line | Layer | Effect |
|---|---|---|---|
| 1 (base) | `:199` | CLI struct (as base) then **file** on top | file overrides shadowed CLI defaults |
| 2 | `:201` | **script** (`runnerOpts`) | script overrides the **file** |
| 3 | `:203` | **env** (`K6_*`) | env overrides the script |
| 4 (last) | `:203` | **CLI** (re-applied) | CLI overrides everything |
| 5 | `:204` | **defaults** | fills only still-unset fields |

- The **script beats the config file** because the script tier is applied at `:201`,
  **after** the file tier at `:199` (proven by **Experiment D**).
- The **CLI beats everything** because `cliConf` is re-applied **last** at `:203`
  (proven by **Experiment A**).

> Note on the double CLI apply: `cliConf` seeds the base at `:199` only so that
> `pflag`'s non-`Valid` (shadowed) default values are present in the struct; the
> **effective** CLI values win because `cliConf` is applied again as the final layer at
> `:203`, consistent with bullet 5 of the doc-comment ("greatest priority").

### 2.4 The per-field merge and the execution-reset block

`Config.Apply` delegates the per-field option merge to `Options.Apply`
(`lib/options.go:357`; the `Options` struct is defined at `lib/options.go:228`):

```go
func (o Options) Apply(opts Options) Options {
```
— `lib/options.go:357`

**`VUs` is merged independently** and only if the higher tier explicitly set it
(`lib/options.go:361-363`):

```go
if opts.VUs.Valid {
    o.VUs = opts.VUs
}
```
— `lib/options.go:361-363`

Immediately after, the **execution-reset block** (`lib/options.go:365-377`) fires whenever
the higher tier sets **any** of `Duration`, `Iterations`, `Stages`, or `Scenarios`:

```go
if opts.Duration.Valid || opts.Iterations.Valid || opts.Stages != nil || opts.Scenarios != nil {
    // TODO: emit a warning or a notice log message if overwrite lower tier config options?
    o.Duration = types.NewNullDuration(0, false)
    o.Iterations = null.NewInt(0, false)
    o.Stages = nil
    o.Scenarios = nil
}
```
— `lib/options.go:365-377`

**Cause → effect:** if a higher tier (e.g. the CLI `--duration`) sets *one* execution
field, this block first **zeroes all four** execution fields inherited from lower tiers —
including a script-defined `Scenarios` map — *before* the incoming value is applied. That
is precisely why a CLI `--duration` **erases** a script `scenarios` map instead of merging
with it (**Experiment E**). By contrast, `VUs` is **not** part of this reset block, so a
bare `--vus` with no `duration`/`iterations`/`stages` sets no execution field at all and is
effectively ignored (**Experiment B**).

---

## 3. Where each layer's values originate

Each of the five layers is produced by a distinct, citable piece of code:

### 3.1 CLI flags (highest precedence)

The option flag set is defined by `optionFlagSet` (`cmd/options.go:23`), and parsed flags
are mapped into a `lib.Options` by `getOptions` (`cmd/options.go:81`):

```go
func optionFlagSet() *pflag.FlagSet {          // cmd/options.go:23
func getOptions(flags *pflag.FlagSet) (lib.Options, error) {  // cmd/options.go:81
```

This is the `cliConf` that seeds the base at `cmd/config.go:199` and is re-applied last
(highest priority) at `cmd/config.go:203`.

### 3.2 `K6_*` environment variables (second-highest)

`readEnvConfig` (`cmd/config.go:170`) parses the environment map into a `Config` using
`envconfig.Process`:

```go
func readEnvConfig(envMap map[string]string) (Config, error) {   // cmd/config.go:170
    conf := Config{}
    err := envconfig.Process("", &conf, func(key string) (string, bool) {
        v, ok := envMap[key]
        return v, ok
    })
```

It is invoked as `readEnvConfig(gs.Env)` at `cmd/config.go:194`. The mapping from env var
name to option field is declared on the `Options` struct via `envconfig` struct tags —
for example `VUs` / `Duration` / `Iterations` at `lib/options.go:234-236`:

```go
VUs        null.Int           `json:"vus" envconfig:"K6_VUS"`         // lib/options.go:234
Duration   types.NullDuration `json:"duration" envconfig:"K6_DURATION"`   // :235
Iterations null.Int           `json:"iterations" envconfig:"K6_ITERATIONS"`  // :236
```

### 3.3 Script `export const options` (middle)

The script's exported `options` object is extracted from the bundled program by
`populateExports` (`js/bundle.go:188`), specifically via
`getExported(consts.Options)` (`js/bundle.go:271`):

```go
jsOptions := bi.getExported(consts.Options)   // js/bundle.go:271
```

The runner then surfaces these to the consolidation code through `GetOptions`
(`js/runner.go:340`):

```go
func (r *Runner) GetOptions() lib.Options {    // js/runner.go:340
```

In `getConsolidatedConfig` this is the `runnerOpts` parameter, applied as the script tier
at `cmd/config.go:201`.

### 3.4 JSON config file `--config` (second-lowest)

The `-c`/`--config` flag is wired in `cmd/root.go:173`:

```go
flags.StringVarP(&gs.Flags.ConfigFilePath, "config", "c", gs.Flags.ConfigFilePath, "JSON config file")
```
— `cmd/root.go:173`

When no `--config` is given, the default path is `~/loadimpact/k6/config.json`, built from
`defaultConfigFileName = "config.json"` (`cmd/state/state.go:20`) and the path join at
`cmd/state/state.go:152`:

```go
const defaultConfigFileName = "config.json"                                       // cmd/state/state.go:20
ConfigFilePath: filepath.Join(homeDir, "loadimpact", "k6", defaultConfigFileName), // :152
```

It is overridable via the `K6_CONFIG` environment variable at `cmd/state/state.go:163`:

```go
if val, ok := env["K6_CONFIG"]; ok {   // cmd/state/state.go:163
```

The file is read by `readDiskConfig` (`cmd/config.go:131`) and applied on top of the CLI
base at `cmd/config.go:199` — i.e. **before** the script tier, which is why the script
wins over it.

### 3.5 Built-in defaults (lowest)

`applyDefault(conf)` runs **last** at `cmd/config.go:204` and fills only fields that no
higher layer set. The observable default is the single-VU, single-iteration
`per-vu-iterations` scenario derived when nothing else is configured (**Experiment B**).

---

## 4. Shortcut → scenario derivation and defaults

After consolidation, the flat "shortcut" fields (`vus`/`duration`/`iterations`/`stages`)
are converted into a concrete **scenario** by `DeriveScenariosFromShortcuts`
(`lib/executor/execution_config_shortcuts.go:52`):

```go
func DeriveScenariosFromShortcuts(opts lib.Options, logger logrus.FieldLogger) (lib.Options, error) {
```
— `lib/executor/execution_config_shortcuts.go:52`

The derivation rules, each matched to a banner observed below:

| Consolidated input | Derived executor | Banner form (observed) |
|---|---|---|
| `Iterations` set | `shared-iterations` | `N iterations shared among M VUs` (Exp C, G1, G3) |
| `Duration` set | `constant-vus` | `N looping VUs for Xs` (Exp A, D, E, G2) |
| `Stages` set | `ramping-vus` | *(not exercised here)* |
| `Scenarios` set explicitly | left as-is | *(erased first if a higher tier set an execution field — Exp E)* |
| **nothing set** | **`per-vu-iterations` {1 VU, 1 iteration}** | `1 iterations for each of 1 VUs` (Exp B) |

The synthetic scenario is named by `DefaultScenarioName = "default"`
(`lib/options.go:21`) — which is why the observed banners and console JSON all use the key
`default`.

**The ignored-`vus` warning.** When `VUs` is set but equals neither the default nor
combines with an execution field, a warning is emitted. It is gated at
`lib/executor/execution_config_shortcuts.go:101` (`opts.VUs.Valid && opts.VUs.Int64 != 1`)
and the message is at `:103`:

```go
if opts.VUs.Valid && opts.VUs.Int64 != 1 {                       // :101
    logger.Warnf(
        "the `vus=%d` option will be ignored, it only works in conjunction with `iterations`, `duration`, or `stages`",  // :103
        opts.VUs.Int64,
    )
}
```
(Observed verbatim in **Experiment B**.)

**Same-tier conflict is a hard error.** Specifying a shortcut *and* `scenarios` in the same
tier raises an `ExecutionConflictError` (type + `Error()` method at
`lib/executor/execution_config_shortcuts.go:11-19`). The `duration`+`scenarios` message is
at `:76-77`:

```go
type ExecutionConflictError string      // lib/executor/execution_config_shortcuts.go:13 (block :11-19)
...
if opts.Scenarios != nil {
    return result, ExecutionConflictError(
        "using an execution configuration shortcut (`duration`) and `scenarios` simultaneously is not allowed",  // :77
    )
```
This surfaces as exit code `InvalidConfig ExitCode = 104` (`errext/exitcodes/codes.go:36`)
(observed in **Experiment F**).

**Consequence to call out:** because shortcuts become **scenarios** after consolidation, at
runtime the top-level `exec.test.options.vus` and `.duration` read **`undefined`** — the
effective values live under `exec.test.options.scenarios.default`. This is exactly what the
per-VU console lines in **Experiment A** show (`vus=undefined duration=undefined`).


---

## 5. When the decision becomes final (the freeze point)

The consolidation, derivation, validation, and **freeze** happen in a short, traceable
chain. The freeze point is **`buildTestRunState`** — where the derived options are
re-injected onto the runner and stored as the authoritative `Options` that the scheduler
and every VU will read.

### 5.1 Orchestration: consolidate → derive → validate

`consolidateDeriveAndValidateConfig` (`cmd/test_load.go:188`) drives the whole thing:

```go
func (lt *loadedTest) consolidateDeriveAndValidateConfig(   // cmd/test_load.go:188
...
consolidatedConfig, err := getConsolidatedConfig(gs, cliConfig, lt.initRunner.GetOptions())  // :203
...
derivedConfig, err := deriveAndValidateConfig(consolidatedConfig, lt.initRunner.IsExecutable, gs.Logger)  // :226
```

- `:203` passes the **script tier** into the merge via `lt.initRunner.GetOptions()`
  (→ `js/runner.go:340`).
- `:226` calls `deriveAndValidateConfig` (defined at `cmd/config.go:248`), where
  `DeriveScenariosFromShortcuts` runs and validation occurs, producing `derivedConfig`.

### 5.2 FREEZE POINT — `buildTestRunState`

`buildTestRunState` (`cmd/test_load.go:265`) re-injects the derived options onto the runner
and stores them as the run's authoritative `Options`:

```go
func (lct *loadedAndConfiguredTest) buildTestRunState(
    configToReinject lib.Options,
) (*lib.TestRunState, error) {
    // This might be the full derived or just the consodlidated options
    if err := lct.initRunner.SetOptions(configToReinject); err != nil {   // :269  re-inject onto runner
        return nil, err
    }
    ...
    return &lib.TestRunState{
        TestPreInitState: lct.preInitState,
        Runner:           lct.initRunner,                                  // :279
        Options:          lct.derivedConfig.Options, // we will always run with the derived options  // :280
        ...
    }, nil
}
```
— `cmd/test_load.go:265-284`

- `SetOptions` at `:269` re-injects the effective options onto the runner (runner
  `SetOptions` is at `js/runner.go:435`).
- The stored, frozen options are `lct.derivedConfig.Options` at **`:280`**, whose inline
  comment states *"we will always run with the derived options."* (Note the store is at
  `:280`; `:279` is `Runner: lct.initRunner,`.)

### 5.3 Consumption: the scheduler reads the frozen set

In `cmd/run.go` the frozen options flow straight into the scheduler:

```go
conf := test.derivedConfig                              // cmd/run.go:127
testRunState, err := test.buildTestRunState(conf.Options)  // :128  (freeze)
...
execScheduler, err := execution.NewScheduler(testRunState, controller)  // :135
```

The scheduler reads the frozen options directly and builds its execution plan from
`trs.Options.Scenarios`:

```go
func NewScheduler(trs *lib.TestRunState, controller Controller) (*Scheduler, error) {  // execution/scheduler.go:38
    options := trs.Options                                                             // :39
```

**Rationale:** once `buildTestRunState` returns the `TestRunState`, the option decision is
final for the run. The scheduler (`execution/scheduler.go:39`) and every VU read from this
single frozen `Options` value — there is no further merging or re-reading of the CLI,
environment, file, or script after this point.


---

## 6. Proof — six real conflicting runs (A–F)

All runs use the locally built `/tmp/k6bin` (the source-commit build,
`k6bin v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)`; built exactly as shown in
[Build and version check](#build-and-version-check) above) and are executed under a
**clean environment** so ambient `K6_*` variables cannot leak in:

```
env -i HOME=/tmp/k6home PATH=/usr/bin:/bin /tmp/k6bin run <flags> <script>
```

The exit code was captured for every run with `echo "EXIT=$?"`. Between repeat runs the
`time="…"` timestamps and the sub-second timings/rates vary, and — for the concurrent
multi-VU runs — the **relative order** in which the per-VU `VU=<n>` console lines are
emitted is **nondeterministic** (the VUs execute in parallel, so no fixed print order is
guaranteed; across repeat runs the five lines of Experiment A were observed in orders such
as `VU=3,1,5,4,2`, `VU=5,2,4,1,3`, and `VU=4,5,3,2,1`). Each per-VU line is nonetheless
identical modulo its `VU=<n>` prefix and the timestamp. Banners, warnings, errors, and exit
codes are reproduced **verbatim**.

> **Console-output note (logging artifact):** `console.log(...)` is emitted inside a
> logrus record as the `msg="..."` field with a trailing `source=console`, so the JSON
> quotes appear **backslash-escaped** on the terminal (e.g. `scenarios={\"default\":...}`).
> The real JSON keys are unescaped; the escaping is purely a logging artifact. The lines
> below are shown exactly as they appeared on screen.

### Experiment A — CLI beats script; shortcut → scenario; multi-VU identical view

**Script `a.js`** declares `export const options = { vus: 2, duration: '3s' };` and prints
`exec.test.options` per VU.

**Command:**
```
env -i HOME=/tmp/k6home PATH=/usr/bin:/bin /tmp/k6bin run --vus 5 --duration 1s a.js
```

**Observed (verbatim excerpts) — EXIT=0:**
```
     scenarios: (100.00%) 1 scenario, 5 max VUs, 31s max duration (incl. graceful stop):
              * default: 5 looping VUs for 1s (gracefulStop: 30s)

time="2026-07-01T04:18:07Z" level=info msg="VU=3 vus=undefined duration=undefined scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"1s\"}}" source=console
time="2026-07-01T04:18:07Z" level=info msg="VU=1 vus=undefined duration=undefined scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"1s\"}}" source=console
time="2026-07-01T04:18:07Z" level=info msg="VU=5 vus=undefined duration=undefined scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"1s\"}}" source=console
time="2026-07-01T04:18:07Z" level=info msg="VU=4 vus=undefined duration=undefined scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"1s\"}}" source=console
time="2026-07-01T04:18:07Z" level=info msg="VU=2 vus=undefined duration=undefined scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"1s\"}}" source=console

running (02.0s), 0/5 VUs, 5 complete and 0 interrupted iterations
default ✓ [ 100% ] 5 VUs  1s
```

**Cause → effect:** the CLI `--vus 5 --duration 1s` **beat** the script's
`{ vus: 2, duration: '3s' }` because `cliConf` is applied last at `cmd/config.go:203`.
The `--duration` shortcut was derived into a `constant-vus` scenario under
`scenarios.default` (`lib/executor/execution_config_shortcuts.go:52`), so the top-level
`vus`/`duration` read `undefined` and the effective values `"vus":5,"duration":"1s"` live
under `scenarios.default`. All five VUs printed the **identical** effective options
(feeds R1, R3, and R4).

### Experiment B — bare `--vus` ignored; per-vu-iterations default

**Script `b.js`** declares no options.

**Command:**
```
env -i HOME=/tmp/k6home PATH=/usr/bin:/bin /tmp/k6bin run --vus 5 b.js
```

**Observed (verbatim) — EXIT=0:**
```
time="2026-07-01T04:18:23Z" level=warning msg="the `vus=5` option will be ignored, it only works in conjunction with `iterations`, `duration`, or `stages`"
     execution: local
        script: b.js
        output: -

     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 1 iterations for each of 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)
```

**Cause → effect:** `VUs` is merged separately (`lib/options.go:361-363`) and is **not**
part of the execution-reset block, so a bare `--vus 5` sets no execution field. The warning
fires from `lib/executor/execution_config_shortcuts.go:103` (gated at `:101`), and with
nothing else set the default `per-vu-iterations` scenario of **1 VU / 1 iteration** is
derived (named `default`, `lib/options.go:21`).

### Experiment C — control (script `iterations`)

**Script `c.js`** declares `export const options = { iterations: 3 };`.

**Command:**
```
env -i HOME=/tmp/k6home PATH=/usr/bin:/bin /tmp/k6bin run c.js
```

**Observed (verbatim) — EXIT=0:**
```
     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 3 iterations shared among 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)
```

**Cause → effect:** a script `iterations` value derives a `shared-iterations` scenario
(`lib/executor/execution_config_shortcuts.go:52`). This is the baseline for comparison with
Experiments G1/G3 (which change the same `3` via the env tier).

### Experiment D — the script beats the JSON config file (counterintuitive)

**File `cfg.json`** = `{"vus":7,"duration":"7s"}`. **Script `d.js`** declares
`export const options = { vus: 2, duration: '3s' };`.

**Command:**
```
env -i HOME=/tmp/k6home PATH=/usr/bin:/bin /tmp/k6bin run --config cfg.json d.js
```

**Observed (verbatim) — EXIT=0:**
```
     scenarios: (100.00%) 1 scenario, 2 max VUs, 33s max duration (incl. graceful stop):
              * default: 2 looping VUs for 3s (gracefulStop: 30s)
```

**Cause → effect:** the effective values are the **script's** `2 VUs` / `3s`, **not** the
config file's `7` / `7s`. The script tier is applied at `cmd/config.go:201`, **after** the
file tier at `cmd/config.go:199`, and "the provided config has priority"
(`cmd/config.go:70`), so the later (script) layer wins. **This is exactly the confusion
the user reported** — the `--config` file is second-lowest in precedence.

### Experiment E — CLI `--duration` erases a script `scenarios` map

**Script `e.js`** defines a named
`export const options = { scenarios: { my_named_scenario: { executor: 'constant-vus', vus: 3, duration: '9s' } } };`.

**Command:**
```
env -i HOME=/tmp/k6home PATH=/usr/bin:/bin /tmp/k6bin run --duration 1s e.js
```

**Observed (verbatim) — EXIT=0:**
```
     scenarios: (100.00%) 1 scenario, 1 max VUs, 31s max duration (incl. graceful stop):
              * default: 1 looping VUs for 1s (gracefulStop: 30s)

time="2026-07-01T04:18:39Z" level=info msg="VU=1 scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":null,\"duration\":\"1s\"}}" source=console
```

**Cause → effect:** the script's named `my_named_scenario` is **gone**; the effective
scenario is a synthetic `default` (`constant-vus`, `"vus":null`, `"duration":"1s"`). The
CLI `--duration` (a higher tier setting an execution field) triggered the execution-reset
block (`lib/options.go:365-377`), which zeroed the script's `Scenarios` map; then
`DeriveScenariosFromShortcuts` (`:52`) built a fresh `constant-vus default` from the
`--duration`. This is the "scenario settings win from a different place" effect.

### Experiment F — same-tier conflict → exit code 104

**Script `f.js`** sets **both**
`export const options = { duration: '5s', scenarios: { my_scenario: { executor: 'constant-vus', vus: 1, duration: '5s' } } };`.

**Command:**
```
env -i HOME=/tmp/k6home PATH=/usr/bin:/bin /tmp/k6bin run f.js
```

**Observed (verbatim) — EXIT=104:**
```
time="2026-07-01T04:18:53Z" level=error msg="using an execution configuration shortcut (`duration`) and `scenarios` simultaneously is not allowed"
```

**Cause → effect:** combining a shortcut (`duration`) and `scenarios` in the **same tier**
is an `ExecutionConflictError` (`lib/executor/execution_config_shortcuts.go:11-19`; message
at `:76-77`), surfaced as `InvalidConfig ExitCode = 104` (`errext/exitcodes/codes.go:36`).
The observed exit code was exactly `104`.

### Summary of A–F

| Exp | Command (flags) | Winner | Effective banner / result | Exit |
|---|---|---|---|---|
| A | `--vus 5 --duration 1s` vs script `{2,3s}` | **CLI** | `* default: 5 looping VUs for 1s (gracefulStop: 30s)` | 0 |
| B | `--vus 5`, no script opts | default | `* default: 1 iterations for each of 1 VUs ...` + ignored-vus warning | 0 |
| C | script `{iterations:3}` | script | `* default: 3 iterations shared among 1 VUs ...` | 0 |
| D | `--config {7,7s}` vs script `{2,3s}` | **script** | `* default: 2 looping VUs for 3s (gracefulStop: 30s)` | 0 |
| E | `--duration 1s` vs script named `scenarios` | **CLI** | named scenario erased → `* default: 1 looping VUs for 1s ...` | 0 |
| F | script `duration` + `scenarios` | — | error, no run | **104** |


---

## 7. The `-e`/`--env` reality vs the docs claim (G1–G3)

⚠️ **A documented claim that this build contradicts.** The official Grafana k6 options
reference states that `-e K6_ITERATIONS=120` "does not configure the script iterations."
**Observed behaviour of this locally built `k6 v0.55.0` under a default `k6 run`
contradicts that claim.** Below is what actually happened; the observed runs are treated as
authoritative.

### Experiment G1 — `-e K6_ITERATIONS=7` *did* change the iterations

**Script `ge.js`** declares `export const options = { iterations: 3 };` and logs
`__ENV.K6_ITERATIONS`.

**Command:**
```
env -i HOME=/tmp/k6home PATH=/usr/bin:/bin /tmp/k6bin run -e K6_ITERATIONS=7 ge.js
```

**Observed (verbatim excerpts) — EXIT=0:**
```
     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 7 iterations shared among 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)

time="2026-07-01T04:18:53Z" level=info msg="__ENV.K6_ITERATIONS=7" source=console
```

The effective iterations **changed from `3` → `7`** — i.e. `-e K6_ITERATIONS=7` **did**
configure the option (contradicting the docs claim), while `__ENV.K6_ITERATIONS` is also
`7`.

### Experiment G2 — `-e` set both VUs and duration

**Command:**
```
env -i HOME=/tmp/k6home PATH=/usr/bin:/bin /tmp/k6bin run -e K6_VUS=4 -e K6_DURATION=2s b.js
```

**Observed (verbatim) — EXIT=0:**
```
              * default: 4 looping VUs for 2s (gracefulStop: 30s)
```

`-e K6_VUS=4 -e K6_DURATION=2s` set **both** the VUs and the duration options.

### Experiment G3 — the toggle proof (`--include-system-env-vars=false`)

**Command:**
```
env -i HOME=/tmp/k6home PATH=/usr/bin:/bin /tmp/k6bin run --include-system-env-vars=false -e K6_ITERATIONS=7 ge.js
```

**Observed (verbatim excerpts) — EXIT=0:**
```
              * default: 3 iterations shared among 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)

time="2026-07-01T04:19:07Z" level=info msg="__ENV.K6_ITERATIONS=7" source=console
```

With `--include-system-env-vars=false`, the iterations **stayed `3`** (the docs-claimed
behaviour) — yet the console **still** printed `__ENV.K6_ITERATIONS=7`. So the `-e` value
reached `__ENV` but did **not** reach the option tier.

### Mechanism (traced from code)

- A default `k6 run` builds its runtime-option flag set with `runtimeOptionFlagSet(true)`
  (`cmd/run.go:441`), so `--include-system-env-vars` defaults to **`true`**.
- In `getRuntimeOptions` (`cmd/runtime_options.go:61`), when that flag is true, `opts.Env`
  is **aliased to the process environment map** at `cmd/runtime_options.go:116-117`:

  ```go
  if opts.IncludeSystemEnvVars.Bool { // If enabled, gather the actual system environment variables
      opts.Env = environment          // cmd/runtime_options.go:117  (environment is gs.Env, by reference)
  }
  ```

- The `-e` loop then writes each key into that same map at `cmd/runtime_options.go:131`:

  ```go
  opts.Env[k] = v   // cmd/runtime_options.go:131
  ```

  Because `opts.Env` is `gs.Env` **by reference**, this **mutates `gs.Env`**.
- `getConsolidatedConfig` reads the env tier from `gs.Env` via
  `readEnvConfig(gs.Env)` (`cmd/config.go:194`), so the injected `K6_*` values are picked
  up at the **environment tier** and change the options.
- When `--include-system-env-vars=false`, the alias at `:117` never runs; `opts.Env`
  stays the **fresh** `make(map[string]string)` created at `cmd/runtime_options.go:72`
  (which only feeds `__ENV`). `gs.Env` is not mutated, so the option is unchanged — exactly
  what **G3** shows.

**State explicitly:** the belief that "`-e` only sets `__ENV` and never options" holds
**only** when `--include-system-env-vars=false`; it does **not** hold for a default
`k6 run`. (For completeness, the `archive` (`cmd/archive.go:63`), `cloud`
(`cmd/cloud.go:336`), `cloud upload` (`cmd/cloud_upload.go:66`), and `inspect`
(`cmd/inspect.go:53`) commands build their flag set with `runtimeOptionFlagSet(false)`,
where `-e` does **not** set options — in contrast to `k6 run`'s
`runtimeOptionFlagSet(true)` at `cmd/run.go:441`.)


---

## 8. Multi-VU behavior via `exec.test.options`

Options are frozen exactly once (at `buildTestRunState`, [§5](#5-when-the-decision-becomes-final-the-freeze-point)). At runtime each VU receives a
**shared-nothing copy of that same frozen `Options`** and reads it through
`exec.test.options`.

### 8.1 The runtime accessor

The `options` property is defined in the `k6/execution` module at
`js/modules/k6/execution/execution.go:184`:

```go
"options": func() interface{} {                       // js/modules/k6/execution/execution.go:184
    vuState := mi.vu.State()                          // :185
    if vuState == nil {                               // :186
        common.Throw(rt, testInfoInitContextErr)      // :187  (init-context guard)
    }
    if optionsObject == nil {
        opts, err := optionsAsObject(rt, vuState.Options)  // :190
        ...
    }
    return optionsObject
```

- It reads `vuState := mi.vu.State()` (`:185`) and, in the **init context**
  (`vuState == nil`), throws `testInfoInitContextErr` (`:186-188`). That error's message is
  defined at `js/modules/k6/execution/execution.go:164`:

  ```go
  var testInfoInitContextErr = common.NewInitContextError("getting test options in the init context is not supported")  // :164
  ```

- Otherwise it returns `optionsAsObject(rt, vuState.Options)` (`:190`; `optionsAsObject`
  is defined at `js/modules/k6/execution/execution.go:283`).

Because each VU's `vuState.Options` is a copy of the single frozen `Options`
(the one stored at `cmd/test_load.go:280` and read by the scheduler at
`execution/scheduler.go:39`), **every VU observes identical effective values**.

### 8.2 Empirical proof (reusing Experiment A)

In **Experiment A** (`--vus 5 --duration 1s a.js`), all five VUs printed the **same**
effective options. Two of the per-VU lines, byte-identical except for the `VU=<n>` id:

```
time="2026-07-01T04:18:07Z" level=info msg="VU=1 vus=undefined duration=undefined scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"1s\"}}" source=console
time="2026-07-01T04:18:07Z" level=info msg="VU=2 vus=undefined duration=undefined scenarios={\"default\":{\"executor\":\"constant-vus\",\"startTime\":null,\"gracefulStop\":null,\"env\":null,\"exec\":null,\"tags\":null,\"vus\":5,\"duration\":\"1s\"}}" source=console
```

Both show `vus=undefined duration=undefined` at the top level and the identical
`scenarios.default` object with `"vus":5,"duration":"1s"` — proving every VU sees the same
frozen effective options (the top-level `vus`/`duration` are `undefined` because the
shortcut was derived into `scenarios.default`, [§4](#4-shortcut--scenario-derivation-and-defaults)).

### 8.3 Cited limitation

Reading test options in the **init context** (module top level, before the VU has state)
is explicitly disallowed: it throws `testInfoInitContextErr` — *"getting test options in
the init context is not supported"* (`js/modules/k6/execution/execution.go:164`, thrown at
`:186-188`).

---

## 9. Web corroboration

The code and observed behaviour above are the source of truth. The official Grafana k6
documentation independently corroborates the precedence model (and highlights the one point
where this build's observed behaviour diverges from a documented claim):

- **Precedence ladder matches.** The official "How to use options" guide
  (`grafana.com/docs/k6/latest/using-k6/k6-options/how-to/`) enumerates the order
  bottom→top as: default value → config file (`--config`) → script value → environment
  variable → CLI flag, and states that command-line flags have the highest order of
  precedence — identical to the code ladder in [§2](#2-how-the-options-are-consolidated-getconsolidatedconfig).
- **Config file is second-lowest.** The same guide states the `--config` file takes "the
  second lowest order of precedence (after defaults)" and that options set anywhere else
  override it — corroborating **Experiment D** (script beats `--config`).
- **The `-e` claim this build contradicts.** The options reference
  (`grafana.com/docs/k6/latest/using-k6/k6-options/reference/`) and the environment-variables
  guide (`grafana.com/docs/k6/latest/using-k6/environment-variables/`) both state that
  "`-e K6_ITERATIONS=120` does not configure the script iterations." As shown in
  **Experiment G1**, this build's default `k6 run` **did** change iterations `3 → 7` via
  `-e K6_ITERATIONS=7`. The documented behaviour only holds with
  `--include-system-env-vars=false` (**Experiment G3**); the docs also note
  `--include-system-env-vars` is `true` for `k6 run` and `false` for `archive`/`cloud`/
  `inspect`, matching the `runtimeOptionFlagSet(true)` wiring at `cmd/run.go:441`.
- **Documented user confusion.** The `--config`-vs-options precedence is a known point of
  confusion in the community (k6-docs issue #688), which is exactly the confusion this
  document resolves.

---

## 10. Coverage pass (R1–R4)

Re-reading the original question and confirming each sub-part is explicitly answered:

| Sub-question | Answered in | Key evidence |
|---|---|---|
| **R1 — Consolidation mechanism & precedence order** | [§2](#2-how-the-options-are-consolidated-getconsolidatedconfig), [§3](#3-where-each-layers-values-originate), [§4](#4-shortcut--scenario-derivation-and-defaults) | Single merge point `getConsolidatedConfig` (`cmd/config.go:189`); apply order at `:199/:201/:203/:204`; ladder **CLI > env > script > config file > defaults**. Proven by **Experiment A** (CLI beats script) and **Experiment D** (script beats `--config`). |
| **R2 — Finalization / freeze point** | [§5](#5-when-the-decision-becomes-final-the-freeze-point) | Freeze at `buildTestRunState` (`cmd/test_load.go:265`), `SetOptions` at `:269`, stored `Options: lct.derivedConfig.Options` at `:280`; consumed by `execution.NewScheduler` (`cmd/run.go:135`) reading `trs.Options` (`execution/scheduler.go:39`). |
| **R3 — Proof via ≥2 conflicting runs (simple values AND scenarios)** | [§6](#6-proof--six-real-conflicting-runs-af), [§7](#7-the--e--env-reality-vs-the-docs-claim-g1g3) | Simple values: **A** (`--vus/--duration` beat script), **B** (bare `--vus` ignored), **C** (script iterations), **D** (script beats `--config`). Scenario settings: **E** (`--duration` erases script `scenarios`), **F** (shortcut+`scenarios` → exit `104`). Plus `-e` discrepancy **G1–G3**. |
| **R4 — Multi-VU behavior** | [§8](#8-multi-vu-behavior-via-exectestoptions) | Options frozen once; each VU reads a shared-nothing copy via `exec.test.options` (`js/modules/k6/execution/execution.go:184`, returning `optionsAsObject(rt, vuState.Options)` at `:190`). **Experiment A** shows all 5 VUs print byte-identical effective options. |

**No sub-part is left unaddressed.** The precedence order (R1), the finalization/freeze
point (R2), proof from ≥2 conflicting real runs covering both simple values and scenario
settings (R3), and multi-VU behavior with the runtime exposure mechanism (R4) are each
answered with exact `file:line` citations, verbatim captured output, and the cause→effect
rationale for every claim.

