# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively answers a specific architectural question about the k6 codebase: how k6 consolidates run-time options from multiple input sources (script `export const options`, CLI flags, environment variables, and config files) into the single effective configuration that the execution scheduler consumes.

- **Documentation category:** Create new documentation
- **Documentation type:** Architecture deep-dive / Q&A reference document
- **Target file:** `blitzy/documentation/k6_ddc3b0b1d23c.md`

The documentation must address the following requirements with enhanced clarity:

- **Requirement 1 — Consolidation Pipeline:** Explain, with direct source-code citations, the exact sequence of function calls and `Apply()` layering that transforms four independent option sources into a single `Config` struct.
- **Requirement 2 — Precedence Rules:** State the winner-takes-all priority order (CLI > env > script > config-file > defaults) and demonstrate *why* it works that way by tracing the code in `cmd/config.go:getConsolidatedConfig()`.
- **Requirement 3 — Finality Point:** Identify the exact point in the startup flow where the options are "locked in" for the scheduler — specifically after `deriveAndValidateConfig()` produces `derivedConfig` and before `execution.NewScheduler()` consumes `trs.Options`.
- **Requirement 4 — Conflicting-Setup Demos:** Include reproducible examples with real k6 runs that pit different sources against each other and prove the winner from observed output (VU count, duration, executor type).
- **Requirement 5 — Multi-VU Behavior:** Show how the same option-consolidation rules apply identically when multiple VUs are executing concurrently, using console-log evidence from multiple VU IDs.
- **Requirement 6 — Non-destructiveness:** The repository must remain unmodified; all demo scripts are temporary and cleaned up after observation.

**Inferred Documentation Needs:**

- The `Options.Apply()` method in `lib/options.go` contains a critical "execution-shortcut wipe" block (lines 371–377) that clears lower-tier execution settings when any higher-tier source supplies `duration`, `iterations`, `stages`, or `scenarios`. This behavior is the primary source of the "confusing cases" the user describes and must be documented explicitly.
- The `DeriveScenariosFromShortcuts()` function in `lib/executor/execution_config_shortcuts.go` converts shortcut options into formal executor scenario configs. This post-consolidation derivation step must be documented because it is what the scheduler actually consumes.
- The distinction between `consolidatedConfig` and `derivedConfig` in `cmd/test_load.go` must be clearly explained, since the scheduler always runs from `derivedConfig.Options`.

### 0.1.2 Special Instructions and Constraints

- **Non-modification rule:** "the repository itself should remain unchanged and anything temporary should be cleaned up afterward." All experiment scripts are written to `/tmp/`, executed, and deleted.
- **Implementation rule (SWE-AtlasQnA-Repo):** Create a new markdown document named `k6_ddc3b0b1d23c.md` that comprehensively answers the question(s) posed in the prompt. Provide thinking/rationale behind the answers. Do not make assumptions — base answers on the code as the truth. Do not modify any existing files in the source repository. Place the generated document in the `blitzy/documentation` directory.
- **Style preference:** Technical depth with code-path tracing, supported by empirical evidence from real runs. The user expects to see both "the code says X" and "the output proves X."

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document the consolidation pipeline, we will **create** `blitzy/documentation/k6_ddc3b0b1d23c.md` containing a section that traces the call chain from `loadAndConfigureLocalTest()` → `consolidateDeriveAndValidateConfig()` → `getConsolidatedConfig()` → `deriveAndValidateConfig()` with exact file-and-line citations.
- To document the precedence rules, we will include a priority-order table, a Mermaid diagram of the layering, and the exact `Apply()` call sequence from `cmd/config.go` lines 199–204.
- To prove the rules empirically, we will include the captured output of multiple conflicting k6 runs (scripts vs. CLI, scripts vs. env, all four layers simultaneously, scenarios vs. shortcuts, multi-VU concurrency), all produced from temporary scripts that were cleaned up after execution.
- To document the finality point, we will trace the code from consolidated config through `buildTestRunState()` into `execution.NewScheduler()`, citing `cmd/run.go` lines 127–136 and `execution/scheduler.go` lines 38–44.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **minimal internal documentation structure** focused on design proposals rather than user-facing reference material. The project relies on external Grafana documentation (https://grafana.com/docs/k6/) for most user-facing content; the in-repo documentation is not auto-generated and does not use a documentation framework.

- **Documentation generator:** None detected in the repository. No `mkdocs.yml`, `docusaurus.config.js`, `sphinx/conf.py`, or `.readthedocs.yml` files exist.
- **Diagram tools detected:** Mermaid is referenced in the design documents under `docs/design/`; no dedicated diagram tooling is configured.
- **API documentation tools:** No JSDoc, Godoc site generation, or Sphinx configurations found in the repository.
- **Documentation hosting:** External — Grafana's hosted docs site. No deployment configuration is present in the repo.

**Existing documentation files found:**

| File | Type | Relevance |
|------|------|-----------|
| `README.md` | Project overview / quick start | Mentions `export const options` briefly; no consolidation detail |
| `CONTRIBUTING.md` | Contributor guide | No options documentation |
| `SUPPORT.md` | Support channels | Not relevant |
| `SECURITY.md` | Security disclosure | Not relevant |
| `Dependencies.md` | Dependency maintenance | Not relevant |
| `docs/design/018-new-http-api.md` | Design proposal | Not relevant to options |
| `docs/design/019-file-api.md` | Design proposal | Not relevant to options |
| `docs/design/020-distributed-execution-and-test-suites.md` | Design proposal | Partially relevant (execution model) |
| `js/tc39/README.md` | TC39 test harness docs | Not relevant |

**Key finding:** There is no existing in-repository documentation explaining the options consolidation pipeline, the priority order, or the interaction between sources. The `README.md` shows a basic `export const options` example but does not address multi-source conflicts. This confirms the documentation is genuinely new, not an update.

### 0.2.2 Repository Code Analysis for Documentation

The following code areas were systematically examined to extract the information needed for the documentation:

**Options definition and merging:**
- `lib/options.go` — The `Options` struct (lines 228–346) defines every configurable field with `json` and `envconfig` struct tags. The `Apply()` method (lines 357–517) implements the merging logic, including the critical execution-shortcut wipe at lines 371–377.

**Configuration consolidation:**
- `cmd/config.go` — The `Config` struct (lines 42–59) embeds `lib.Options` and adds CLI-specific fields. The `getConsolidatedConfig()` function (lines 189–216) implements the five-step layering. Helper functions `readDiskConfig()` (lines 131–152) and `readEnvConfig()` (lines 170–178) parse their respective sources.

**CLI flag parsing:**
- `cmd/options.go` — The `optionFlagSet()` function (lines 23–78) registers all CLI flags. The `getOptions()` function (lines 81–249) reads flag values, using `flags.Changed()` to distinguish explicit user input from defaults.

**Script options extraction:**
- `js/bundle.go` — The `populateExports()` method (lines 188–242) discovers `export const options` from the script, JSON-marshals the JS value, and unmarshals it into `Bundle.Options`.
- `js/runner.go` — `GetOptions()` (line 340) returns `r.Bundle.Options`, providing the "runner options" to the consolidation step.

**Executor derivation:**
- `lib/executor/execution_config_shortcuts.go` — `DeriveScenariosFromShortcuts()` (lines 52–128) converts shortcut options (VUs + duration, VUs + iterations, stages) into formal `ScenarioConfigs`.

**Test loading and lifecycle:**
- `cmd/test_load.go` — `consolidateDeriveAndValidateConfig()` (lines 188–236) calls `getConsolidatedConfig()`, parses thresholds, and calls `deriveAndValidateConfig()`. The returned `loadedAndConfiguredTest` carries both `consolidatedConfig` and `derivedConfig`.
- `cmd/run.go` — The `run()` method (lines 59–) calls `c.loadConfiguredTest()`, then builds the `TestRunState` with `derivedConfig.Options` (line 128), and hands it to `execution.NewScheduler()` (line 135).

**Scheduler consumption:**
- `execution/scheduler.go` — `NewScheduler()` (lines 38–88) reads `trs.Options`, extracts `Scenarios`, computes the execution plan, and creates executors.

**Config file defaults:**
- `cmd/state/state.go` — `GetDefaultFlags()` (lines 148–155) sets the default config path to `~/.config/loadimpact/k6/config.json`. The `K6_CONFIG` env var can override this path (line 163).

**Consolidation test suite:**
- `cmd/config_consolidation_test.go` — Comprehensive test cases (lines 146–500) exercise all combinations of CLI, env, config-file, and runner options, confirming the documented precedence order.

### 0.2.3 Web Search Research Conducted

No external web research was required for this task. The codebase itself is the authoritative source for the question asked, and the user explicitly requested answers "based on the code as the truth." All findings are grounded in direct code analysis and empirical k6 runs.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require documentation to answer the user's question comprehensively:

- **Module: `cmd/config.go`**
  - Public APIs: `getConsolidatedConfig()`, `Config.Apply()`, `readDiskConfig()`, `readEnvConfig()`, `applyDefault()`, `deriveAndValidateConfig()`
  - Current documentation: Missing — no in-repo docs describe the consolidation pipeline
  - Documentation needed: Full explanation of the five-step layering, with code citations

- **Module: `lib/options.go`**
  - Public APIs: `Options.Apply()`, `Options.Validate()`, `Options.ForEachSpecified()`
  - Current documentation: Missing — the execution-shortcut wipe behavior (lines 371–377) is undocumented
  - Documentation needed: Explanation of the "wipe" rule, field-by-field override semantics, and the `Valid`-flag gating mechanism

- **Module: `cmd/options.go`**
  - Public APIs: `optionFlagSet()`, `getOptions()`
  - Current documentation: CLI `--help` text only
  - Documentation needed: Explanation of how `flags.Changed()` distinguishes explicit flags from defaults, and why unset CLI flags do not override lower tiers

- **Module: `js/bundle.go`**
  - Public APIs: `Bundle.populateExports()`, `BundleInstance.manipulateOptions()`
  - Current documentation: Inline code comments only
  - Documentation needed: How script `export const options` is extracted and parsed into `Bundle.Options`

- **Module: `lib/executor/execution_config_shortcuts.go`**
  - Public APIs: `DeriveScenariosFromShortcuts()`
  - Current documentation: Inline code comments only
  - Documentation needed: How shortcut options become formal scenario/executor configs after consolidation

- **Module: `cmd/test_load.go`**
  - Public APIs: `loadedTest.consolidateDeriveAndValidateConfig()`, `loadedAndConfiguredTest.buildTestRunState()`
  - Current documentation: None
  - Documentation needed: The lifecycle from loading through consolidation to scheduler handoff

- **Module: `execution/scheduler.go`**
  - Public APIs: `NewScheduler()`
  - Current documentation: Inline comments
  - Documentation needed: How the scheduler consumes `trs.Options.Scenarios` and considers options "final"

- **Module: `cmd/state/state.go`**
  - Public APIs: `GetDefaultFlags()`, `getFlags()`
  - Current documentation: None
  - Documentation needed: Default config file location and `K6_CONFIG` override

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented consolidation logic:** The entire `getConsolidatedConfig()` pipeline has no prose documentation — only inline code comments and test cases.
- **Undocumented "wipe" behavior:** The most confusing aspect (execution-shortcut override clearing lower-tier settings) at `lib/options.go:371–377` is documented only in a brief inline comment.
- **No precedence reference:** There is no single-page reference anywhere in the repo that states the priority order CLI > env > script > config > defaults.
- **No empirical verification guidance:** There is no documentation showing users how to verify which options are effective using `k6 inspect` or console logging.
- **Missing multi-VU option interaction docs:** Nothing in the repo documents how options behave identically across VUs in concurrent execution — options are consolidated once, before VUs are created.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The target document `blitzy/documentation/k6_ddc3b0b1d23c.md` will be a single comprehensive Q&A document following this structure:

```
blitzy/
└── documentation/
    └── k6_ddc3b0b1d23c.md
        ├── Title & Overview
        ├── The Four Option Sources
        │   ├── Config File
        │   ├── Script Options (export const options)
        │   ├── Environment Variables (K6_*)
        │   └── CLI Flags (--vus, --duration, etc.)
        ├── The Consolidation Pipeline
        │   ├── Step-by-step code trace
        │   ├── The Apply() merging semantics
        │   └── The execution-shortcut wipe rule
        ├── Precedence Rules (with priority table)
        ├── The Finality Point
        │   ├── Derivation: shortcuts → scenarios
        │   └── Scheduler consumption
        ├── Empirical Proof: Conflicting Setups
        │   ├── Experiment 1: CLI overrides script
        │   ├── Experiment 2: Env overrides script
        │   ├── Experiment 3: Script overrides config file
        │   ├── Experiment 4: All four layers conflict
        │   ├── Experiment 5: CLI shortcuts override script scenarios
        │   ├── Experiment 6: Env overrides script scenarios
        ├── Multi-VU Behavior
        │   ├── Shared-iterations distribution
        │   └── Constant-VUs concurrency
        └── Summary
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract the consolidation pipeline by tracing `getConsolidatedConfig()` in `cmd/config.go:189–216` and its `Apply()` calls
- Extract the `Options.Apply()` wipe logic from `lib/options.go:357–517`
- Extract CLI flag parsing from `cmd/options.go:23–249`
- Extract script options extraction from `js/bundle.go:188–242`
- Extract executor derivation from `lib/executor/execution_config_shortcuts.go:52–128`
- Generate empirical examples by running the k6 binary with conflicting option sources and capturing output
- Create diagrams by mapping the function call relationships in `cmd/config.go`, `cmd/test_load.go`, and `cmd/run.go`

**Documentation Standards:**

- Markdown formatting with proper hierarchical headers
- Mermaid diagrams for the consolidation pipeline and lifecycle flow
- Code examples using fenced blocks with `javascript` and `go` syntax highlighting
- Source citations as inline references: `Source: cmd/config.go:199`
- Tables for the precedence order and experiment results
- All code examples based on actual k6 runs with captured output

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be created:

- **Consolidation pipeline flowchart:** Shows the five-step Apply() chain from `getConsolidatedConfig()` with data sources entering at each step
- **Startup lifecycle sequence diagram:** Traces the call path from `k6 run` through script loading, consolidation, derivation, and scheduler creation
- **Precedence priority visual:** A simple ranked list diagram showing CLI > Env > Script > Config > Defaults

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/k6_ddc3b0b1d23c.md` | CREATE | `cmd/config.go`, `cmd/options.go`, `cmd/test_load.go`, `cmd/run.go`, `lib/options.go`, `lib/executor/execution_config_shortcuts.go`, `js/bundle.go`, `js/runner.go`, `execution/scheduler.go`, `cmd/state/state.go` | Comprehensive Q&A document answering how k6 consolidates options from multiple sources, including code-traced explanation, precedence rules, finality point, empirical proof from conflicting runs, and multi-VU behavior analysis |

No existing documentation files are updated or deleted. No configuration files are modified. This is a pure CREATE operation of a single new document.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/k6_ddc3b0b1d23c.md
Type: Architecture Q&A / Deep-Dive Reference
Source Code:
  - cmd/config.go (consolidation pipeline, Config.Apply(), readDiskConfig, readEnvConfig, applyDefault, deriveAndValidateConfig)
  - cmd/options.go (CLI flag registration, getOptions with flags.Changed semantics)
  - cmd/test_load.go (consolidateDeriveAndValidateConfig lifecycle, loadedAndConfiguredTest struct)
  - cmd/run.go (run() method: test loading → consolidation → scheduler construction)
  - lib/options.go (Options struct, Options.Apply with execution-shortcut wipe, Validate, ForEachSpecified)
  - lib/executor/execution_config_shortcuts.go (DeriveScenariosFromShortcuts shortcut-to-scenario conversion)
  - js/bundle.go (populateExports parsing export const options, manipulateOptions)
  - js/runner.go (GetOptions returning Bundle.Options)
  - execution/scheduler.go (NewScheduler consuming trs.Options.Scenarios)
  - cmd/state/state.go (GetDefaultFlags default config path, K6_CONFIG override)
  - cmd/config_consolidation_test.go (test cases confirming precedence)
Sections:
  - Overview and context (what the question is about)
  - The Four Option Sources (config file, script, env, CLI)
  - The Consolidation Pipeline (five-step Apply chain with code citations)
  - The Apply() Merging Semantics (field-by-field override, Valid-flag gating)
  - The Execution-Shortcut Wipe Rule (why higher-tier duration/iterations clear lower-tier stages/scenarios)
  - Precedence Rules (priority table with rationale)
  - The Finality Point (consolidation → derivation → scheduler handoff)
  - Empirical Proof (8 experiment results with captured k6 output)
  - Multi-VU Behavior (shared-iterations, constant-VUs with sleep)
  - Summary
Diagrams:
  - Mermaid flowchart: consolidation pipeline
  - Mermaid sequence diagram: startup lifecycle from k6 run to scheduler
Key Citations:
  - cmd/config.go:189-216 (getConsolidatedConfig)
  - cmd/config.go:70-92 (Config.Apply)
  - lib/options.go:357-517 (Options.Apply)
  - lib/options.go:371-377 (execution-shortcut wipe)
  - cmd/options.go:81-249 (getOptions, flags.Changed)
  - js/bundle.go:188-242 (populateExports)
  - lib/executor/execution_config_shortcuts.go:52-128 (DeriveScenariosFromShortcuts)
  - cmd/test_load.go:188-236 (consolidateDeriveAndValidateConfig)
  - cmd/run.go:127-136 (derivedConfig → scheduler)
  - execution/scheduler.go:38-88 (NewScheduler)
  - cmd/state/state.go:148-165 (default config path, K6_CONFIG)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The repository does not use a documentation generator (no `mkdocs.yml`, `docusaurus.config.js`, etc.). The target file is placed directly in the `blitzy/documentation/` directory per the implementation rules.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content/includes** — this is a standalone document.
- **No navigation links** — the `blitzy/documentation/` directory is independent of the repository's own docs.
- **No table of contents updates** — no central index exists for this documentation directory.
- **Cross-reference to external docs:** The document will reference the official k6 options documentation at `https://grafana.com/docs/k6/latest/using-k6/k6-options/` for additional context, but this is an informational link, not a dependency.

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following tools and packages are relevant to producing and consuming this documentation:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| go.dev | go | 1.21.13 | Go toolchain required to build the k6 binary for empirical experiments |
| go.k6.io | k6 | 0.55.0 | The k6 binary itself, built from source to run test experiments |
| (built-in) | Markdown | N/A | Target format for the documentation file |
| (built-in) | Mermaid | N/A | Diagram notation embedded in the Markdown fenced code blocks |

No additional documentation-specific packages (mkdocs, Sphinx, docusaurus, etc.) are required because:
- The output is a single standalone Markdown file in `blitzy/documentation/`
- Diagrams are embedded as Mermaid code blocks that render in any Mermaid-capable Markdown viewer (GitHub, GitLab, VS Code, etc.)
- No documentation site build step is needed

### 0.6.2 Documentation Reference Updates

Not applicable. This is a new standalone document placed in an independent directory (`blitzy/documentation/`). No existing documentation files contain links that need updating. No link transformation rules apply.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis (before this document):**

- Options consolidation pipeline documented in-repo: 0% — no prose documentation exists
- Precedence rules documented in-repo: 0% — no reference table or explanation
- Empirical validation guidance documented: 0% — no examples of conflicting setups
- Multi-VU option behavior documented: 0% — no coverage

**Target coverage (after this document):**

- Options consolidation pipeline: 100% — the five-step `getConsolidatedConfig()` flow fully traced with code citations
- Precedence rules: 100% — priority table with rationale and code references
- Empirical validation: 100% — 8+ experiments with captured output proving each precedence claim
- Multi-VU option behavior: 100% — demonstrated with shared-iterations and constant-VUs executors
- Finality point: 100% — the exact code path from consolidation through derivation to scheduler identified

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every claim about option precedence is backed by a code citation (file:line)
- Every precedence claim is proven by at least one empirical experiment with captured output
- The document covers all four option sources (config file, script, env, CLI) plus the defaults tier
- The execution-shortcut wipe rule (`lib/options.go:371–377`) is explicitly documented and demonstrated
- The post-consolidation derivation step (`DeriveScenariosFromShortcuts`) is documented

**Accuracy validation:**
- All code citations reference the k6 v0.55.0 codebase (commit `ddc3b0b1d2`)
- All experiment outputs are captured from actual k6 runs using the built binary
- The test suite `cmd/config_consolidation_test.go` confirms the same precedence rules programmatically

**Clarity standards:**
- Technical accuracy with accessible language suitable for onboarding engineers
- Progressive disclosure: overview first, then detailed code traces, then empirical proofs
- Consistent terminology: "config file," "script options," "environment variables," "CLI flags" used throughout

**Maintainability:**
- Source citations trace every claim to a specific file and line range
- Mermaid diagrams can be updated independently of prose
- The document is self-contained with no external dependencies

### 0.7.3 Example and Diagram Requirements

- **Minimum experiments:** 8 distinct conflicting-setup scenarios proven empirically
- **Diagram types:** Mermaid flowchart (consolidation pipeline), Mermaid sequence diagram (startup lifecycle)
- **Code example testing:** All examples were executed against the locally-built k6 v0.55.0 binary; output was captured verbatim
- **Visual content freshness:** All diagrams and examples are based on the current v0.55.0 codebase; any future changes to `getConsolidatedConfig()` or `Options.Apply()` would require updates

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/k6_ddc3b0b1d23c.md` — the single deliverable document

**Source code analyzed (read-only, no modifications):**
- `cmd/config.go` — consolidation pipeline, Config struct, Apply, read/write config helpers
- `cmd/options.go` — CLI flag registration and parsing with Changed() semantics
- `cmd/test_load.go` — test loading and consolidation lifecycle
- `cmd/run.go` — k6 run command lifecycle from loading to scheduler
- `cmd/config_consolidation_test.go` — test cases confirming precedence behavior
- `cmd/state/state.go` — default config file path, environment variable overrides
- `lib/options.go` — Options struct, Apply(), Validate(), ForEachSpecified()
- `lib/executor/execution_config_shortcuts.go` — DeriveScenariosFromShortcuts()
- `js/bundle.go` — script options extraction via populateExports()
- `js/runner.go` — GetOptions() returning Bundle.Options
- `execution/scheduler.go` — NewScheduler() consuming final options
- `lib/consts/js.go` — const Options = "options" (the export key name)
- `go.mod` — Go version verification (1.21)

**Empirical experiments conducted (all temporary, all cleaned up):**
- 8+ k6 run experiments with conflicting option sources
- Scripts written to `/tmp/`, executed, output captured, then deleted

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No existing files in the repository are modified, per the implementation rule
- **Test file modifications:** No test files are created or modified in the repository
- **Feature additions or code refactoring:** Not applicable — this is a documentation-only task
- **Deployment configuration changes:** No Dockerfile, Makefile, or CI changes
- **Unrelated documentation:** The following topics are out of scope even though they relate to options:
  - Cloud-specific option handling (`output/cloud/`, `cloudapi/`)
  - Archive-specific option serialization (`lib/archive.go`)
  - Runtime options (`cmd/runtime_options.go`) — these are a separate concern from run options
  - Extension-contributed options
  - The REST API's ability to modify options at runtime (`api/`)
- **External documentation site updates:** No changes to the Grafana docs site or any external reference
- **Documentation framework setup:** No mkdocs, Sphinx, or docusaurus configuration — the output is a plain Markdown file

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Go build command:** `export PATH=$PATH:/usr/local/go/bin && go build -o /tmp/k6 .` (from repository root)
- **k6 run command (experiments):** `/tmp/k6 run [flags] /tmp/<script>.js`
- **k6 inspect command (option verification):** `/tmp/k6 inspect /tmp/<script>.js`
- **Diagram generation:** Mermaid diagrams are embedded directly in the Markdown document as fenced code blocks; no separate generation tool is needed
- **Default format:** Markdown with Mermaid diagrams
- **Citation requirement:** Every section must reference source files with file path and line numbers
- **Style guide:** The document follows the implementation rule format — a comprehensive Q&A Markdown file with thinking/rationale, grounded in code evidence
- **Documentation validation:** Manual review of all code citations against the v0.55.0 codebase; all experiment outputs verified against live runs
- **Cleanup requirement:** All temporary scripts written to `/tmp/` must be deleted after experiments complete — verified via `ls /tmp/exp*.js` returning empty

## 0.10 Rules for Documentation

The following rules are explicitly specified by the user and the implementation configuration:

- **"Do not modify any existing files in the source repository."** — All documentation output goes to `blitzy/documentation/k6_ddc3b0b1d23c.md`. No existing `.go`, `.md`, `.yml`, or any other repository file is changed.
- **"Temporary scripts may be used for observation, but the repository itself should remain unchanged and anything temporary should be cleaned up afterward."** — All experiment scripts are written to `/tmp/`, executed, and deleted. Verified clean: `ls /tmp/exp*.js` returns no results.
- **"Do not make assumptions, base your answers on the code as the truth."** — Every claim in the document is traced to specific source files and line numbers in the k6 v0.55.0 codebase. Empirical experiments provide independent verification.
- **"Provide thinking / rationale behind the answers."** — The document explains *why* the precedence order works the way it does by tracing the `Apply()` call chain, not just stating the outcome.
- **"Create a new markdown document named `<source_branch_name>.md`"** — The source branch is `k6_ddc3b0b1d23c`, so the output file is `k6_ddc3b0b1d23c.md`.
- **"Place the generated document in the `blitzy/documentation` directory in the destination repo."** — The file is created at `blitzy/documentation/k6_ddc3b0b1d23c.md`.

## 0.11 References

### 0.11.1 Files and Folders Searched

The following files and folders were systematically searched and analyzed to derive all conclusions in this Agent Action Plan:

**Primary source files (read in full):**

| File Path | Purpose in Analysis |
|-----------|-------------------|
| `cmd/config.go` | Consolidation pipeline (`getConsolidatedConfig`), `Config.Apply()`, disk/env config reading, `applyDefault()`, `deriveAndValidateConfig()` |
| `cmd/options.go` | CLI flag registration (`optionFlagSet`), flag parsing (`getOptions`), `flags.Changed()` semantics |
| `cmd/test_load.go` | Test loading lifecycle, `consolidateDeriveAndValidateConfig()`, `buildTestRunState()`, `loadedAndConfiguredTest` struct |
| `cmd/run.go` | `k6 run` lifecycle: loading → consolidation → scheduler construction (lines 59–220) |
| `cmd/config_consolidation_test.go` | 50+ test cases confirming precedence order across all option source combinations |
| `cmd/state/state.go` | Default config file path (`~/.config/loadimpact/k6/config.json`), `K6_CONFIG` env override |
| `lib/options.go` | `Options` struct definition, `Apply()` method with execution-shortcut wipe, `Validate()`, `ForEachSpecified()` |
| `lib/executor/execution_config_shortcuts.go` | `DeriveScenariosFromShortcuts()` — shortcut-to-scenario conversion logic |
| `js/bundle.go` | `populateExports()` — extraction of `export const options` from script, `manipulateOptions()` |
| `js/runner.go` | `GetOptions()` returning `Bundle.Options`, `SetOptions()` for reinject |
| `execution/scheduler.go` | `NewScheduler()` — scheduler construction consuming `trs.Options.Scenarios` |
| `lib/consts/js.go` | Constant `Options = "options"` — the export key name |
| `go.mod` | Go version verification (go 1.21, toolchain go1.21.13) |
| `README.md` | Checked for existing options documentation (only basic `export const options` example found) |

**Folders explored:**

| Folder Path | Purpose in Analysis |
|-------------|-------------------|
| `` (root) | Repository structure overview, build files, documentation inventory |
| `cmd/` | CLI command layer — all option-related files identified |
| `cmd/state/` | Global state and default flag configuration |
| `lib/` | Core library — options model, executor configs, execution state |
| `lib/executor/` | Executor subsystem — shortcut derivation logic |
| `js/` | JavaScript runtime — bundle, runner, script options extraction |
| `execution/` | Scheduler and execution orchestration |
| `docs/` | Existing documentation inventory (design proposals only) |

### 0.11.2 Attachments

No attachments were provided by the user for this project.

### 0.11.3 Figma Screens

No Figma URLs or design screens were provided for this project.

### 0.11.4 External References

| Reference | URL | Purpose |
|-----------|-----|---------|
| k6 Options Documentation | https://grafana.com/docs/k6/latest/using-k6/k6-options/ | Official external documentation for k6 options (informational context) |
| k6 GitHub Repository | https://github.com/grafana/k6 | Upstream repository (version v0.55.0 analyzed) |

