# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new investigative-analysis document** that comprehensively answers a series of interconnected questions about k6's module resolution behavior during the VU execution phase versus the initialization phase. The document must be named `k6_ddc3b0b1d23c.md` and placed in the `blitzy/documentation` directory.

- **Category**: Create new documentation
- **Documentation type**: Technical deep-dive / Investigative Q&A document with empirical evidence

The user's requirements decompose into the following distinct documentation objectives:

- **Objective 1 — Init-vs-VU Behavioral Divergence**: Explain why k6 scripts that work during the init stage begin failing once VUs execute under load, particularly when code attempts dynamic module loading at runtime.
- **Objective 2 — Module Resolution Freeze Mechanism**: Determine whether k6 intentionally "freezes" (locks) module resolution after initialization, document the exact mechanism, and identify the boundary between "already resolved" (cached) versus "new" (rejected) modules.
- **Objective 3 — Relative Specifier Resolution Semantics**: Clarify how relative module specifiers (e.g., `./helper.js`, `../lib/util.js`) are resolved when the call stack originates from different modules, and whether this creates ambiguity.
- **Objective 4 — Empirical Demonstration with Real Runs**: Provide evidence from actual k6 runs showing: (a) a module imported during init can still be used by VUs, (b) a never-seen module reliably fails during VU init, and (c) the exact error or warning messages produced in each case.
- **Objective 5 — Repository Preservation**: All observations must use temporary scripts that are cleaned up afterward; the repository itself must remain unmodified.

### 0.1.2 Special Instructions and Constraints

- **Repository immutability**: The user explicitly requires "the repository itself should remain unchanged and anything temporary should be cleaned up afterward." No existing files may be modified.
- **Implementation rule**: Per the `SWE-AtlasQnA-Repo` project rule, the output document must be named `k6_ddc3b0b1d23c.md`, placed in the `blitzy/documentation` directory, and must provide thinking/rationale behind answers based on the code as truth.
- **Evidence-based answers**: All answers must be grounded in the actual source code, with no assumptions. The code is the single source of truth.
- **Real run results**: The user requests "real runs" demonstrating behavior, including exact error and warning messages.
- **Temporary experiments only**: Any experimental scripts created for observation must be ephemeral and cleaned up afterward.
- **Style**: Technical deep-dive with code citations, real output examples, and rationale for each answer.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **explain the init-vs-VU divergence**, we will trace the code path in `js/bundle.go` that calls `bundle.ModuleResolver.Lock()` at line 129, and the two-layer protection in `js/bundle.go:424-428` (init context check) and `js/modules/resolution.go:69-72, 166-168` (resolver lock check), creating narrative documentation with code citations.
- To **document the freeze mechanism**, we will analyze the `ModuleResolver` struct in `js/modules/resolution.go`, its `locked` boolean field, the `Lock()` method at line 137-139, and the cache lookup logic in the `resolve()` method at lines 145-177, producing a technical explanation with a decision-flow diagram.
- To **clarify relative specifier resolution**, we will document the `getCurrentModuleScript()` function in `js/modules/require_impl.go:185-196` which uses `rt.CaptureCallStack(2, ...)` to determine the calling module, and the `reversePath()` method in `js/modules/resolution.go:198-211` which maps module records back to their source URLs, then resolves relative paths from there.
- To **provide empirical demonstrations**, we will include captured output from multiple k6 run experiments, each targeting a specific behavior: init-imported modules used by VUs (success), never-seen modules during VU init (failure with exact error), and `require()` blocked during VU execution (failure with exact error).
- To **preserve the repository**, all experimental scripts are created in `/tmp/k6-experiments/` and cleaned up after observation. The output document is placed only in the designated `blitzy/documentation` directory.

### 0.1.4 Inferred Documentation Needs

Based on code analysis:

- The two-layer protection system (init-context guard **and** resolver lock) needs clear documentation because users may conflate the two distinct error messages: `"the 'require' function is only available in the init stage"` versus `"the module %q was not previously resolved during initialization (__VU==0)"`.
- The relationship between `__VU==0` (bundle init) and `__VU>0` (per-VU init) during the init phase must be clarified — both run during init context, but only the bundle init at `__VU==0` occurs before the resolver is locked.
- Built-in Go modules (`k6/*`) follow the same cache-or-reject pattern as filesystem modules, which may surprise users who assume built-in modules are always available.
- The `open()` function has an analogous freeze mechanism via `allowOnlyOpenedFiles()` in `js/initcontext.go:57-64`, which should be mentioned as a parallel pattern for completeness.
- The `sobekModuleResolver` callback used for ESM `import` statements resolves relative specifiers using the `reversePath()` of the referencing module record, which is conceptually different from the `getCurrentModuleScript()` stack-walking approach used by `require()`. Both mechanisms converge on the same cache, but their path-derivation strategies differ.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **flat, governance-focused documentation structure** with no dedicated user-facing documentation generator. The repository does not use MkDocs, Docusaurus, Sphinx, or any other documentation site generator — official k6 user documentation is hosted externally at `https://grafana.com/docs/k6/`.

- **Documentation framework**: None (no `mkdocs.yml`, `docusaurus.config.js`, `sphinx/conf.py`, or `.readthedocs.yml` found)
- **Documentation generator configuration**: Not applicable
- **API documentation tools**: No JSDoc, Godoc generation, or auto-doc pipelines detected in the repository
- **Diagram tools**: Mermaid is used in the tech spec sections; no PlantUML or other diagram tooling detected in the repository itself
- **Documentation hosting/deployment**: External — Grafana documentation site

Existing markdown documentation in the repository:

| File | Purpose | Relevance |
|------|---------|-----------|
| `README.md` | Project overview, badges, getting started | Low — general project info |
| `CONTRIBUTING.md` | Contributor guidelines | Low — process documentation |
| `Dependencies.md` | Dependency maintenance policy | Low — governance |
| `SECURITY.md` | Security disclosure process | None |
| `SUPPORT.md` | Support routing | None |
| `CODE_OF_CONDUCT.md` | Community guidelines | None |
| `docs/design/018-new-http-api.md` | Future HTTP API design proposal | Low — tangential |
| `docs/design/019-file-api.md` | Future file API design proposal | Medium — mentions `open()` freeze pattern |
| `docs/design/020-distributed-execution-and-test-suites.md` | Distributed execution proposal | Low |
| `js/modules/k6/experimental/README.md` | Experimental modules overview | Low |
| `js/tc39/README.md` | TC39 conformance harness documentation | Low |

No existing documentation covers k6's module resolution freeze mechanism, the `ModuleResolver.Lock()` behavior, or the init-vs-VU module loading semantics in any document within the repository.

### 0.2.2 Repository Code Analysis for Documentation

The following source files were analyzed in depth to derive answers for the documentation:

**Module Resolution System (primary focus):**

| File | Key Content Found |
|------|-------------------|
| `js/modules/resolution.go` | `ModuleResolver` struct with `locked` field; `Lock()` method (line 137); `resolve()` with cache-then-lock-check logic (lines 145-177); `requireModule()` with lock check (lines 69-72); `notPreviouslyResolvedModule` error constant (line 17); `sobekModuleResolver` callback (line 192); `reversePath()` for URL derivation (line 198) |
| `js/modules/require_impl.go` | `Require()` entry point (line 15); `getCurrentModuleScript()` using `CaptureCallStack(2)` for relative specifier resolution (line 185); `getPreviousRequiringFile()` deep stack walk (line 198); `promisesThenIgnore()` helper (line 229) |
| `js/bundle.go` | `newBundle()` calling `bundle.ModuleResolver.Lock()` at line 129 after first instantiation; `requireImpl` wrapper with init-context guard (lines 419-429); `generateFileLoad()` for filesystem module loading (line 507) |
| `js/initcontext.go` | `cantBeUsedOutsideInitContextMsg` constant (line 15); `allowOnlyOpenedFiles()` analogous freeze for file operations (line 57); `openImpl()` for init-stage file reading (line 21) |
| `js/modules_vu.go` | `moduleVUImpl` with `state` field — when `state != nil`, VU is executing (not in init context) |
| `js/runner.go` | `vu.state` assignment at line 230 marking VU as activated; `NewVU()` lifecycle |

**Module Adapters:**

| File | Key Content Found |
|------|-------------------|
| `js/modules/cjsmodule.go` | CommonJS module wrapping into Sobek module records |
| `js/modules/gomodule.go` | Go-backed module adapter with lazy export discovery |
| `js/modules/gomodule_basic.go` | Lightweight Go-value-to-module bridge |
| `js/modules/modules.go` | `Module`, `Instance`, `VU`, `Exports` interfaces; `Register()` function |

**Source Loading:**

| File | Key Content Found |
|------|-------------------|
| `loader/loader.go` | `Resolve()` function for specifier resolution (line 48); `resolveFilePath()` for relative paths (line 84); `Load()` for filesystem/HTTPS loading (line 122) |
| `loader/filesystems.go` | `CreateFilesystems` helper for filesystem registry |

**Test Files (behavioral evidence):**

| File | Key Content Found |
|------|-------------------|
| `js/runner_test.go` | `TestVUDoesNotRequireUnderConditions` (line 1409) — proves lock error for modules only required when `__VU > 0`; `TestVUDoesRequireUnderConditions` (line 1434) — proves cached modules work across VUs |
| `js/init_and_modules_test.go` | `TestNewJSRunnerWithCustomModule` — validates Go module lifecycle across init and VU phases |
| `js/path_resolution_test.go` | Tests around relative path resolution from nested require chains (GitHub issue #2674) |
| `js/module_loading_test.go` | `TestLoadOnceGlobalVars` — validates module singleton behavior across ESM and CommonJS |

### 0.2.3 Web Search Research Conducted

No external web search was needed. All answers are derivable directly from the k6 source code, which is the authoritative truth per project rules. The k6 test lifecycle documentation referenced in the `cantBeUsedOutsideInitContextMsg` error message (`https://grafana.com/docs/k6/latest/using-k6/test-lifecycle/`) was noted as a relevant external reference for users.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The documentation output is a single investigative Q&A document. All content is synthesized from the following source modules:

- **Module: `js/modules/resolution.go`**
  - Public APIs: `ModuleResolver` struct, `NewModuleResolver()`, `Lock()`, `resolve()`, `requireModule()`, `resolveLoaded()`, `resolveSpecifier()`, `sobekModuleResolver()`, `reversePath()`, `Imported()`; `ModuleSystem` struct, `NewModuleSystem()`, `RunSourceData()`
  - Current documentation: None within the repository (no internal doc beyond brief GoDoc comments)
  - Documentation needed: Deep explanation of the lock mechanism, cache semantics, and resolve flow — all to be synthesized into the output Q&A document

- **Module: `js/modules/require_impl.go`**
  - Public APIs: `Require()`, `Resolve()`, `CurrentlyRequiredModule()`, `ShouldWarnOnParentDirNotMatchingCurrentModuleParentDir()`
  - Current documentation: Minimal GoDoc comments
  - Documentation needed: Explanation of `getCurrentModuleScript()` stack-walking behavior and how it determines the parent module for relative specifiers

- **Module: `js/bundle.go`**
  - Public APIs: `NewBundle()`, `Instantiate()`, `Bundle.instantiate()`, `requireImpl.require()`
  - Current documentation: Brief GoDoc comments
  - Documentation needed: The two-phase init process (bundle init at `__VU==0` then per-VU init at `__VU>0`), the `Lock()` call timing, and the `requireImpl` init-context gate

- **Module: `js/initcontext.go`**
  - Public APIs: `openImpl()`, `allowOnlyOpenedFiles()`
  - Current documentation: `cantBeUsedOutsideInitContextMsg` constant with link to external docs
  - Documentation needed: Parallel freeze pattern for `open()` to contextualize module resolution freeze

- **Module: `loader/loader.go`**
  - Public APIs: `Resolve()`, `Dir()`, `Load()`
  - Current documentation: Package-level GoDoc, brief function comments
  - Documentation needed: How specifiers are resolved relative to a `pwd` URL, supporting the relative-specifier discussion

- **Module: `js/runner.go`**
  - Key section: `vu.state` assignment (line 230) and `vu.moduleVUImpl.state = vu.state` (line 247)
  - Documentation needed: How VU activation sets `state`, making `vu.state != nil` the signal that the init context has ended

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No existing document** in the repository explains the module resolution freeze mechanism or its rationale.
- **No existing document** maps the exact sequence: bundle init → `Lock()` → per-VU init (with locked resolver) → VU execution (with `require()` fully blocked).
- **No existing document** distinguishes the two error paths: init-context guard (`cantBeUsedOutsideInitContextMsg`) versus resolver lock guard (`notPreviouslyResolvedModule`).
- **No existing document** explains how relative specifiers are resolved differently by ESM `import` (via `sobekModuleResolver` + `reversePath()`) versus CommonJS `require()` (via `getCurrentModuleScript()` + stack walking).
- **No existing document** provides real k6 run output demonstrating the freeze behavior with exact error messages.
- **The `blitzy/documentation` directory does not yet exist** and must be created for the output document.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output is a single comprehensive markdown document. The planned hierarchy:

```
blitzy/
└── documentation/
    └── k6_ddc3b0b1d23c.md
```

Internal document structure:

```
k6_ddc3b0b1d23c.md
├── Title and Overview
├── 1. Why Scripts Fail Under VU Load (Init vs. VU Lifecycle)
│   ├── The k6 Test Lifecycle Phases
│   ├── What Happens During Bundle Init (__VU==0)
│   └── What Happens During Per-VU Init (__VU>0)
├── 2. The Module Resolution Freeze Mechanism
│   ├── The ModuleResolver.Lock() Call
│   ├── How the Cache Determines "Already Resolved"
│   ├── Two-Layer Protection System
│   │   ├── Layer 1: Init Context Guard (require() blocked in VU execution)
│   │   └── Layer 2: Resolver Lock (new modules rejected in VU init)
│   └── Decision Flow Diagram (Mermaid)
├── 3. Relative Specifier Resolution Across Call Sites
│   ├── ESM import: sobekModuleResolver + reversePath()
│   ├── CommonJS require(): getCurrentModuleScript() + Stack Walking
│   └── Why the Cache Prevents Confusion Under Load
├── 4. Empirical Demonstrations (Real k6 Runs)
│   ├── Experiment A: Init-imported module used successfully by VUs
│   ├── Experiment B: Never-seen module fails during VU init
│   ├── Experiment C: require() blocked during VU execution
│   ├── Experiment D: Built-in k6 module lock behavior
│   ├── Experiment E: Cached module re-required successfully during VU init
│   └── Experiment F: High VU count stability test
├── 5. The open() Parallel — Analogous Freeze for Files
└── 6. Summary and Practical Guidance
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract the freeze mechanism by tracing `ModuleResolver.Lock()` call in `js/bundle.go:129`, the `locked` boolean check in `js/modules/resolution.go:70` and `js/modules/resolution.go:166`, and the cache lookup in `js/modules/resolution.go:104,150,162`.
- Extract relative specifier resolution by analyzing `getCurrentModuleScript()` in `js/modules/require_impl.go:185-196` and `reversePath()` in `js/modules/resolution.go:198-211`.
- Generate empirical evidence from captured k6 run outputs showing success (init-imported modules working under VU load) and failure (exact error messages for locked modules and init-context violations).
- Create diagrams by mapping the decision flow in `resolve()` and the lifecycle phases from `newBundle()` through `Instantiate()` to VU execution.

**Documentation Standards:**

- Markdown formatting with proper headers (`#`, `##`, `###`)
- Mermaid diagram for the module resolution decision flow
- Code snippets with syntax highlighting for both k6 test scripts and Go source references
- Source citations as inline references: `Source: js/modules/resolution.go:17`
- Real k6 output presented in fenced code blocks
- Consistent use of precise terminology: "bundle init" for `__VU==0`, "per-VU init" for `__VU>0`, "VU execution" for the default function body

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create:

- **Module Resolution Decision Flowchart**: A flowchart showing the decision path in `ModuleResolver.resolve()` — does the specifier start with `k6/`? → check cache → if not cached, is the resolver locked? → reject or load from filesystem.
- **k6 Lifecycle Phase Diagram**: A sequence or state diagram showing Bundle Init → Lock → Per-VU Init → VU Activation → VU Execution, with annotations showing which module operations are permitted at each phase.

Both diagrams will be embedded in the output markdown document using Mermaid fenced blocks.

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/k6_ddc3b0b1d23c.md` | CREATE | `js/modules/resolution.go`, `js/modules/require_impl.go`, `js/bundle.go`, `js/initcontext.go`, `js/modules_vu.go`, `js/runner.go`, `loader/loader.go`, `js/modules/cjsmodule.go`, `js/modules/gomodule.go`, `js/runner_test.go`, `js/init_and_modules_test.go`, `js/path_resolution_test.go`, `js/module_loading_test.go` | Complete investigative Q&A document answering all five user questions about k6 module resolution freeze behavior, with code citations, Mermaid diagrams, and real k6 run output |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/k6_ddc3b0b1d23c.md
Type: Technical deep-dive / Investigative Q&A
Source Code:
  - js/modules/resolution.go (primary — ModuleResolver, Lock, resolve, cache)
  - js/modules/require_impl.go (Require, getCurrentModuleScript, getPreviousRequiringFile)
  - js/bundle.go (newBundle, Lock() call site, requireImpl, generateFileLoad)
  - js/initcontext.go (cantBeUsedOutsideInitContextMsg, allowOnlyOpenedFiles, openImpl)
  - js/modules_vu.go (moduleVUImpl, state field)
  - js/runner.go (vu.state assignment, VU activation)
  - loader/loader.go (Resolve, resolveFilePath, Dir, Load)
  - js/modules/cjsmodule.go (CommonJS module adapter)
  - js/modules/gomodule.go (Go module adapter)
  - js/runner_test.go (TestVUDoesNotRequireUnderConditions, TestVUDoesRequireUnderConditions)
  - js/init_and_modules_test.go (TestNewJSRunnerWithCustomModule)
  - js/path_resolution_test.go (relative path resolution tests)
  - js/module_loading_test.go (singleton module loading tests)
Sections:
  - Overview (purpose: answer user's five questions about module resolution freeze)
  - Why Scripts Fail Under VU Load (lifecycle phases, init vs VU)
  - The Module Resolution Freeze Mechanism (Lock(), cache, two-layer protection)
  - Relative Specifier Resolution Across Call Sites (ESM vs CJS resolution)
  - Empirical Demonstrations (real k6 run outputs for 6+ experiments)
  - The open() Parallel (analogous freeze for file operations)
  - Summary and Practical Guidance (actionable recommendations)
Diagrams:
  - Module resolution decision flowchart (Mermaid)
  - k6 lifecycle phase diagram showing where Lock() occurs (Mermaid)
Key Citations:
  - js/modules/resolution.go:17 (notPreviouslyResolvedModule error constant)
  - js/modules/resolution.go:69-72 (requireModule lock check)
  - js/modules/resolution.go:137-139 (Lock() method)
  - js/modules/resolution.go:145-177 (resolve() with cache and lock)
  - js/modules/require_impl.go:185-196 (getCurrentModuleScript stack walk)
  - js/modules/resolution.go:198-211 (reversePath for ESM resolution)
  - js/bundle.go:129 (Lock() call site after bundle init)
  - js/bundle.go:424-428 (requireImpl init-context guard)
  - js/initcontext.go:15-16 (cantBeUsedOutsideInitContextMsg)
  - js/initcontext.go:57-64 (allowOnlyOpenedFiles)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration files need to be updated. The repository does not use a documentation generator. The output is a standalone markdown file in a new directory.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content or includes**: The output document is self-contained.
- **No navigation links**: There is no documentation site to update.
- **No table of contents or index updates**: No central documentation index exists.
- **No glossary updates**: No glossary file exists in the repository.
- The document will reference the external k6 test lifecycle documentation at `https://grafana.com/docs/k6/latest/using-k6/test-lifecycle/` as a supplementary resource for readers.

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The documentation exercise requires building and running k6 from source to produce empirical evidence. The relevant dependencies are:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| go.dev | go | 1.21 (toolchain go1.21.13) | Go runtime required to build k6 from source per `go.mod` |
| go.k6.io | k6 | v0.55.0 | The k6 binary built from the repository (commit ddc3b0b1d2) |
| github.com | grafana/sobek | v0.0.0-20241024150027-d91f02b05e9b | JavaScript engine powering module resolution and evaluation |
| github.com | evanw/esbuild | v0.21.2 | Script compiler used during module parsing in the init phase |
| github.com | sirupsen/logrus | v1.9.3 | Structured logging used for debug output during module loading |

No additional documentation tooling (MkDocs, Sphinx, Docusaurus, etc.) is required. The output is a standalone Markdown file authored directly. Mermaid diagrams embedded in the markdown are intended for rendering by any standard Mermaid-compatible viewer (GitHub, VS Code extensions, etc.).

### 0.6.2 Documentation Reference Updates

Not applicable. The output document is a new standalone file with no existing links to update. No link transformation rules are needed.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

The user posed five interconnected questions. Coverage targets are defined per question:

| Question | Coverage Target | Source Files Required |
|----------|----------------|---------------------|
| Why do scripts fail under VU load? | 100% — Full lifecycle explanation with code citations | `js/bundle.go`, `js/runner.go`, `js/modules_vu.go` |
| Does k6 freeze module resolution? | 100% — Exact mechanism with `Lock()` code path, cache semantics | `js/modules/resolution.go` (lines 17, 31-39, 69-72, 137-139, 145-177) |
| What counts as resolved vs. new? | 100% — Cache key explanation, specifier normalization | `js/modules/resolution.go` (cache map, `resolve()`, `resolveLoaded()`) |
| How do relative specifiers work? | 100% — Both ESM and CJS resolution paths documented | `js/modules/require_impl.go` (lines 185-196), `js/modules/resolution.go` (lines 192-211), `loader/loader.go` (lines 48-82) |
| Real run demonstrations | 100% — At least 6 distinct experiments with captured output | Built k6 binary, temporary test scripts |

- **Overall target coverage**: 100% of user questions answered with code-backed evidence
- **Current coverage**: 0% (no existing documentation addresses these topics)
- **Post-implementation coverage**: 100%

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**

- Every answer cites specific source file paths and line numbers
- Every mechanism description includes the actual Go code identifiers (function names, variable names, constants)
- Both error message strings are reproduced verbatim from source code
- At least 6 real k6 run experiments are included with full captured output
- Each experiment includes the test script, the k6 command, and the relevant output lines

**Accuracy validation:**

- All code citations must reference actual line numbers verified against the current commit (ddc3b0b1d2)
- Error messages quoted in the document must match the exact strings in `resolution.go:17` and `initcontext.go:15-16`
- Real run outputs must be actual captured terminal output, not fabricated examples
- Module resolution flow descriptions must be traceable through the `resolve()` → cache check → lock check → load chain

**Clarity standards:**

- Technical accuracy with accessible language — the document targets k6 users who write JavaScript tests, not Go developers
- Progressive disclosure: start with the high-level "why" (lifecycle phases), then the "how" (code mechanism), then "proof" (real runs)
- Consistent terminology: "bundle init" for `__VU==0`, "per-VU init" for `__VU>0`, "VU execution" for the default function body, "resolver lock" for the `ModuleResolver.locked` mechanism

**Maintainability:**

- Source citations formatted as `Source: path/to/file.go:LineNumber` for traceability
- Document version tied to k6 v0.55.0 (commit ddc3b0b1d2)

### 0.7.3 Example and Diagram Requirements

- **Minimum experiments**: 6 distinct k6 run demonstrations
- **Diagram types**: Module resolution decision flowchart (Mermaid flowchart), lifecycle phase diagram (Mermaid sequence or state diagram)
- **Code example testing**: All k6 test scripts were executed against the built binary (`/tmp/k6-binary`) and their outputs captured
- **Visual content freshness**: Diagrams reflect the v0.55.0 codebase logic

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**

- `blitzy/documentation/k6_ddc3b0b1d23c.md` — the sole output artifact

**Source analysis targets (read-only, for deriving answers):**

- `js/modules/resolution.go` — ModuleResolver, Lock(), resolve(), cache, reversePath()
- `js/modules/require_impl.go` — Require(), getCurrentModuleScript(), getPreviousRequiringFile()
- `js/bundle.go` — newBundle(), Lock() call site, requireImpl, instantiate()
- `js/initcontext.go` — cantBeUsedOutsideInitContextMsg, allowOnlyOpenedFiles(), openImpl()
- `js/modules_vu.go` — moduleVUImpl state field
- `js/runner.go` — VU state assignment, VU activation lifecycle
- `loader/loader.go` — Resolve(), resolveFilePath(), Dir(), Load()
- `js/modules/cjsmodule.go` — CommonJS module adapter
- `js/modules/gomodule.go` — Go module adapter
- `js/modules/gomodule_basic.go` — Basic Go value module adapter
- `js/modules/modules.go` — Module interface, Register()
- `js/runner_test.go` — TestVUDoesNotRequireUnderConditions, TestVUDoesRequireUnderConditions
- `js/init_and_modules_test.go` — TestNewJSRunnerWithCustomModule
- `js/path_resolution_test.go` — Relative path resolution tests
- `js/module_loading_test.go` — Module loading singleton tests
- `js/esm_vs_commonjs_test.go` — ESM vs CJS behavioral tests
- `go.mod` — Go version and dependency versions

**Empirical experiments (temporary, all cleaned up):**

- Init-imported module used successfully by VUs
- Never-seen module fails with `notPreviouslyResolvedModule` error during VU init
- `require()` blocked during VU execution with `cantBeUsedOutsideInitContextMsg`
- Built-in k6 module (`k6/crypto`) lock behavior
- Cached module re-required successfully during VU init
- Relative specifier resolution from nested modules
- High VU count stability confirmation
- Dynamic `import()` behavior

### 0.8.2 Explicitly Out of Scope

- **Source code modifications**: No existing repository files will be modified (per user instruction and `SWE-AtlasQnA-Repo` rule)
- **Test file modifications**: No test files will be changed
- **Feature additions or code refactoring**: This is purely a documentation exercise
- **Deployment configuration changes**: Not applicable
- **Documentation for unrelated k6 features**: HTTP module, WebSocket module, gRPC, browser module, metrics system, CLI, cloud API, output backends, and all other k6 subsystems not related to module resolution are out of scope
- **External documentation site updates**: The Grafana docs site (`grafana.com/docs/k6/`) is external and out of scope
- **Performance benchmarking**: While VU count experiments are included, rigorous performance profiling is out of scope
- **Distributed execution behavior**: The investigation is limited to local, single-process execution
- **Archive (TAR) module resolution**: While the same `ModuleResolver` handles archives, the investigation focuses on the primary filesystem-based flow

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command**: Not applicable — the output is a standalone Markdown file with no build step
- **Documentation preview command**: Any Markdown viewer or `cat blitzy/documentation/k6_ddc3b0b1d23c.md`; Mermaid diagrams render in GitHub, GitLab, and VS Code with the Mermaid extension
- **Diagram generation command**: Not applicable — Mermaid diagrams are embedded inline in the Markdown source
- **Documentation deployment command**: Not applicable — the file is committed directly to the repository
- **Default format**: Markdown with Mermaid diagrams
- **Citation requirement**: Every technical claim must reference the source file and line number
- **Style guide**: Technical deep-dive with progressive disclosure (high-level explanation → code mechanism → empirical proof); consistent terminology as defined in the quality criteria
- **Documentation validation**: Manual review — verify that all cited line numbers match the current codebase at commit `ddc3b0b1d2`

### 0.9.2 k6 Build and Run Parameters (for Experiments)

The empirical evidence was produced using:

- **Build command**: `GOTOOLCHAIN=local go build -o /tmp/k6-binary ./.` from the repository root
- **k6 version**: `v0.55.0 (commit/ddc3b0b1d2, go1.22.2, linux/amd64)`
- **Run command pattern**: `timeout 30 /tmp/k6-binary run /tmp/k6-experiments/<script>.js`
- **Experiment directory**: `/tmp/k6-experiments/` (created, used, and deleted — no repository modification)
- **Cleanup**: `rm -rf /tmp/k6-experiments` after all experiments complete

## 0.10 Rules for Documentation

The following rules are derived from the user's instructions and the `SWE-AtlasQnA-Repo` project rule:

- **Do not modify any existing files in the source repository.** The output is a single new file only.
- **Create a new markdown document named `k6_ddc3b0b1d23c.md`** (matching the source branch name `k6_ddc3b0b1d23c`) that comprehensively answers the questions posed in the prompt.
- **Place the generated document in the `blitzy/documentation` directory** in the destination repo. The directory must be created if it does not exist.
- **Provide thinking and rationale behind the answers.** Every conclusion must be explained, not merely stated.
- **Do not make assumptions — base all answers on the code as the truth.** All claims must cite specific source files and line numbers from the k6 v0.55.0 codebase.
- **Temporary scripts may be used for observation, but the repository itself should remain unchanged and anything temporary should be cleaned up afterward.** All experimental k6 scripts are created outside the repository tree and deleted after use.
- **Show real runs** demonstrating: (a) a module imported during init that can still be required by VUs, (b) a never-seen module that reliably fails, and (c) the exact error or warning in each case.

## 0.11 References

### 0.11.1 Source Files and Folders Searched

The following files and folders were retrieved, read, or searched across the codebase to derive the conclusions in this Agent Action Plan:

**Primary Source Files (read in full):**

| File Path | Key Content Extracted |
|-----------|----------------------|
| `js/modules/resolution.go` | `ModuleResolver` struct, `locked` field, `Lock()`, `resolve()`, `requireModule()`, `resolveLoaded()`, `sobekModuleResolver()`, `reversePath()`, `notPreviouslyResolvedModule` constant, `ModuleSystem`, `NewModuleSystem()`, `RunSourceData()` |
| `js/modules/require_impl.go` | `Require()`, `getCurrentModuleScript()`, `getPreviousRequiringFile()`, `Resolve()`, `CurrentlyRequiredModule()`, `ShouldWarnOnParentDirNotMatchingCurrentModuleParentDir()`, `toESModuleExports()`, `promisesThenIgnore()` |
| `js/bundle.go` | `Bundle` struct, `newBundle()`, `Lock()` call at line 129, `Instantiate()`, `instantiate()`, `requireImpl` struct, `requireImpl.require()`, `setInitGlobals()`, `generateFileLoad()`, `populateExports()` |
| `js/initcontext.go` | `cantBeUsedOutsideInitContextMsg` constant, `openImpl()`, `readFile()`, `allowOnlyOpenedFiles()`, `generateSourceMapLoader()` |
| `js/modules_vu.go` | `moduleVUImpl` struct, `state` field, `InitEnv()`, `State()`, `Runtime()` |
| `js/runner.go` | `Runner` struct, VU state assignment (line 230), VU activation lifecycle |
| `loader/loader.go` | `SourceData`, `Resolve()`, `resolveFilePath()`, `Dir()`, `Load()`, `loadRemoteURL()`, `fetch()` |
| `js/modules/cjsmodule.go` | `cjsModule` struct, `cjsModuleInstance`, `cjsModuleFromString()` |
| `js/modules/modules.go` | `Module`, `Instance`, `VU`, `Exports` interfaces, `Register()` |
| `js/init_and_modules_test.go` | `TestNewJSRunnerWithCustomModule` — Go module lifecycle verification |
| `js/runner_test.go` (lines 1409-1477) | `TestVUDoesNotRequireUnderConditions`, `TestVUDoesRequireUnderConditions` |
| `js/path_resolution_test.go` (lines 1-100) | Path resolution tests for relative specifiers |
| `js/module_loading_test.go` (lines 1-100) | `TestLoadOnceGlobalVars` — singleton module behavior |
| `js/esm_vs_commonjs_test.go` | ESM vs CommonJS behavioral edge cases |
| `go.mod` | Go version (1.21), toolchain (go1.21.13), key dependencies |

**Folders Explored:**

| Folder Path | Purpose |
|-------------|---------|
| `` (root) | Repository structure overview |
| `js/` | JavaScript runtime layer — core analysis target |
| `js/modules/` | Module system and resolver layer — primary focus |
| `loader/` | Source loading and specifier resolution |
| `docs/` | Existing documentation (design proposals only) |

**Files Searched by Pattern (grep):**

| Search Pattern | Files Matched | Purpose |
|----------------|---------------|---------|
| `\.Lock()` in `js/` | `js/bundle.go:129`, others | Find where resolver is locked |
| `locked` in `js/` | `js/modules/resolution.go`, `js/modules/require_impl.go` | Trace all lock-related logic |
| `cantBeUsedOutsideInitContextMsg` in `js/` | `js/bundle.go`, `js/initcontext.go` | Trace init-context error message |
| `notPreviouslyResolved` in `js/` | `js/modules/resolution.go`, `js/runner_test.go` | Trace lock-rejection error message |
| `sobekModuleResolver`, `reversePath`, `getCurrentModuleScript` in `js/modules/` | Multiple files | Trace relative specifier resolution |
| `*.md` not in `vendor/` or `.git/` | 15 files | Locate existing documentation |

### 0.11.2 Attachments

No attachments were provided by the user. No Figma screens, external files, or supplementary materials were included.

### 0.11.3 External References

| Reference | URL | Relevance |
|-----------|-----|-----------|
| k6 Test Lifecycle Documentation | `https://grafana.com/docs/k6/latest/using-k6/test-lifecycle/` | Referenced in the `cantBeUsedOutsideInitContextMsg` error message; provides user-facing lifecycle explanation |
| GitHub Issue #2674 | `https://github.com/grafana/k6/issues/2674` | Referenced in `js/path_resolution_test.go` as the motivating issue for relative path resolution tests |
| k6 Modules Documentation | `https://grafana.com/docs/k6/latest/using-k6/modules/` | Referenced in the `fileSchemeCouldntBeLoadedMsg` error message in `loader/loader.go` |

### 0.11.4 Empirical Experiments Conducted

All experiments used temporary scripts in `/tmp/k6-experiments/` (now cleaned up). Summary of results:

| Experiment | Script | Result | Key Evidence |
|------------|--------|--------|--------------|
| Init-imported module used by VUs | `test_init_import.js` | SUCCESS | 3 VUs, 6 iterations, all printed `Hello, VU-N` correctly |
| Never-seen module require in VU execution | `test_dynamic_require.js` | Expected failure | Error: `the "require" function is only available in the init stage` |
| Dynamic import() of unseen module | `test_dynamic_import.js` | Expected failure | Error caught (undefined) — dynamic import not supported for new modules |
| Module required only at __VU>0 | `test_vu_init_require.js` | Expected failure | Error: `the module "./conditional_mod.js" was not previously resolved during initialization (__VU==0)` |
| Cached module re-required at VU init | `test_cached_vu_init.js` | SUCCESS | Module loaded at __VU==0, re-loaded from cache at __VU==1 and __VU==2 |
| Relative specifiers from nested modules | `test_relative_specifiers.js` | SUCCESS | 2 VUs correctly resolved `./lib/base_util.js` → `./lib/sub/deep.js` chain |
| k6 built-in module require in VU execution | `test_k6_module_vu.js` | Expected failure | Error: `the "require" function is only available in the init stage` |
| High VU stability | `test_high_vu.js` | SUCCESS | 20 VUs, 100 iterations, 0 errors, all modules resolved from cache |
| ESM import then use from VUs | `test_esm_import_then_use.js` | SUCCESS | 3 VUs, 9 iterations, all imported functions worked correctly |
| k6/crypto only at __VU>0 | `test_vu_init_new_k6_module.js` | Expected failure (non-fatal) | Error: `the module "k6/crypto" was not previously resolved during initialization (__VU==0)` — VU still ran |
| k6/crypto loaded at init, re-required at VU init | `test_k6_cached_vu_init.js` | SUCCESS | k6/crypto loaded at __VU==0, re-required from cache at __VU==1 and __VU==2; md5 hash computed correctly |

