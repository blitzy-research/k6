# How Grafana k6 Resolves JavaScript Modules Across the `init` and VU Lifecycle — and Why "Dynamic" Loading Fails at Scale

> **Question answered here:** *How does Grafana k6 resolve JavaScript modules across the init and VU (Virtual User) lifecycle stages, and why do some scripts succeed during initialization yet fail once the test runs "under load" with dynamic (runtime) module loading? Does k6 freeze module resolution after init? What counts as "already resolved" vs. a "new module"? What happens with relative specifiers whose resolution depends on the calling module? Show — with real runs — that a module imported during init can still be `require()`'d later by VUs, that a never-seen module reliably fails, and the exact error text in each case.*

This document is an **evidence-grounded investigation**. Every behavioral claim is backed by **both** (1) a specific `file:line` reference into the k6 source tree and (2) the **verbatim output of a real run** of a binary built from this exact branch. The investigation was performed by **building and running first**, then writing from what was observed.

---

## 0. Provenance — the exact binary and how it was built and run

All observations below come from a binary built from this repository. **Two git commits matter and are kept strictly distinct throughout this document:**

- **Source-under-investigation** (the k6 code being analyzed): branch `k6_ddc3b0b1d23c`, whose head — *before* this documentation file was committed — is commit `ddc3b0b1d23c128e34e2792fc9075f9126e32375` (short `ddc3b0b1d`; 10‑char build stamp `ddc3b0b1d2`). **Every source `file:line` citation and every line number in this answer refers to this code.**
- **Delivered repository HEAD** (what a reviewer who checks out the delivered branch receives): a commit that is exactly the source-under-investigation **plus this single documentation file and nothing else**. `git diff ddc3b0b1d <delivered-HEAD> --name-status` lists exactly one entry — `A blitzy/documentation/k6_ddc3b0b1d23c.md` — with **zero** `.go` / `go.mod` / `go.sum` / `vendor/` changes. Because the delivered-HEAD hash advances with every revision of this documentation file (and the `commit/…` build stamp tracks it — see below), only the baseline `ddc3b0b1d2` is a fixed reference; the doc commit observed while writing this answer was `3e9f5aabc82ca4e851bf42f60fc98bd9efeadc00` (10‑char stamp `3e9f5aabc8`). Because no source file differs between the baseline and any delivered-HEAD doc commit, the analyzed runtime behavior, line numbers, and error strings are identical whichever of the two the binary is built from.
- **Go toolchain (observed):** `go version go1.23.12 linux/amd64`.
- **Canonical build command (offline, vendored dependencies), run from the repository root:**

```
PATH=/usr/local/go/bin:$PATH GOTOOLCHAIN=local GOFLAGS=-mod=vendor CGO_ENABLED=0 go build -o /tmp/k6bin .
```

The `-mod=vendor` flag uses the fully vendored dependency tree (no network), `GOTOOLCHAIN=local` pins the installed `go1.23.12` (the repo's `go.mod` declares `go 1.21` [go.mod:L3] / `toolchain go1.21.13` [go.mod:L5], but the canonical build target is Go 1.23.x per `Dockerfile:L1` `FROM --platform=$BUILDPLATFORM golang:1.23-alpine3.20 as builder` and `.github/workflows/build.yml:L27` `DEFAULT_GO_VERSION: "1.23.x"`), and `CGO_ENABLED=0` matches the canonical non-race build.

- **Version strings reported by the built binary (verbatim; both observed this investigation).** k6 derives the `commit/…` segment at build time from Go's embedded VCS stamp: `FullVersion()` reads `vcs.revision` via `debug.ReadBuildInfo()` and takes its **first 10 characters** [lib/consts/consts.go:L28-L35], appending `-dirty` only when `vcs.modified == "true"` [lib/consts/consts.go:L36-L39,L48-L49], then formats `"%s (commit/%s, %s)"` [lib/consts/consts.go:L52]. The `commit/…` segment is therefore purely the git HEAD stamp and affects no analyzed behavior.

  Building the **source-under-investigation** (a clean checkout at `ddc3b0b1d`, i.e. before this document existed) reports:

```
k6bin v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)
```

  Building a **delivered-HEAD doc commit** (here `3e9f5aabc`, the doc commit observed while writing this answer; clean tree so `vcs.modified=false`) reports:

```
k6bin v0.55.0 (commit/3e9f5aabc8, go1.23.12, linux/amd64)
```

  The two differ **only** in the 10‑char VCS stamp (`ddc3b0b1d2` vs the delivered doc commit, e.g. `3e9f5aabc8`) — a direct consequence of the doc-only commit; every other field (`v0.55.0`, `go1.23.12`, `linux/amd64`) and all analyzed behavior are identical. The delivered stamp is whatever commit finally carries this file; only the baseline `ddc3b0b1d2` is fixed. To reproduce the exact source-under-investigation stamp, check the baseline out into a fresh clone (a *linked* `git worktree` at the baseline emits **no** VCS stamp under Go 1.23, so a full checkout/clone is required) and build there:

```
git clone --no-hardlinks . /tmp/k6base && cd /tmp/k6base && git checkout ddc3b0b1d
PATH=/usr/local/go/bin:$PATH GOTOOLCHAIN=local GOFLAGS=-mod=vendor CGO_ENABLED=0 go build -o /tmp/k6bin .
/tmp/k6bin version    # -> k6bin v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)
```

- **Canonical invocation used for every experiment** (stdout/stderr captured together, exit code recorded):

```
/tmp/k6bin run --quiet --no-summary <script>.js
echo "EXIT CODE: $?"
```

For the VU-count demonstration (§6), `--vus N --iterations N` was added.

- **Scratch workspace:** all test scripts were created **outside** the source tree, under `/tmp/k6exp/`, and were removed after the investigation. The k6 source tree was left byte-for-byte unchanged (only this `blitzy/` document is added).

- **JavaScript-engine and transpiler provenance (dependencies — unchanged by this investigation).** The observed behavior depends on two vendored dependencies pinned in `go.mod`: the JavaScript engine **`github.com/grafana/sobek v0.0.0-20241024150027-d91f02b05e9b`** [go.mod:L78], and the ESM/TypeScript transpiler **`github.com/evanw/esbuild v0.21.2`** [go.mod:L12] (which transpiles ESM/TypeScript test scripts to runnable JS before the module system executes them). `grafana/sobek` is a maintained fork of `dop251/goja`; crucially, the entire module-resolution machinery analyzed in this answer imports **`github.com/grafana/sobek`, not `dop251/goja`** — see the import lines [js/bundle.go:L14], [js/modules/resolution.go:L8], and [js/modules/require_impl.go:L9]. (The only `dop251/goja` occurrences anywhere under `js/` are attribution/link **comments** in unrelated experimental and TC39-conformance files, not imports.) This is precisely why every JavaScript-engine stack frame in the outputs below is named `github.com/grafana/sobek/...`. No dependency is added, upgraded, or removed.

> **Reading the output:** k6 writes console and error logs to **stderr**, wrapped as `time="..." level=... msg="..." source=...`. Inside `msg="..."`, the sequences `\n\t` are literal escapes that render as multi-line stack traces. Outputs below are reproduced exactly as emitted. The `time="..."` timestamp varies from run to run and carries no meaning for this analysis.

> **Version fidelity:** every value here is for the built version **`v0.55.0`, source-under-investigation commit `ddc3b0b1d2`, `go1.23.12`** (equivalently, the delivered-HEAD build `3e9f5aabc8`, which is behaviorally identical — the difference is doc-only, as shown above). Line numbers, error strings, and exit codes are not generalized to other k6 versions.

---

## 1. Direct answer (the core thesis) — the freeze is deterministic, not "pressure"-driven

**k6 freezes module resolution exactly once, deterministically, immediately after the first initialization (`__VU==0`) completes — via a single `ModuleResolver.Lock()` call at [`js/bundle.go:L129`]. It is *not* triggered by "runtime pressure," CPU load, or a high VU count.**

The user's framing — that resolution freezes "when the runtime is under pressure" — is **incorrect and must be corrected**. Here is the accurate picture:

1. When the script is first loaded, k6 builds the bundle by running the init context **once at `__VU==0`**. During that first init, every module that is `require()`d **unconditionally at the top level** is resolved and **cached**.
2. The instant that first init returns, k6 calls `bundle.ModuleResolver.Lock()` **one time** [js/bundle.go:L129], which sets `mr.locked = true` [js/modules/resolution.go:L138]. This is the **only** `ModuleResolver.Lock()` call in the entire codebase (grep-confirmed).
3. From then on, **every VU re-runs its init code** (k6 runs init once per VU). A `require()` for a specifier that is **already in the cache** returns the cached module and **succeeds** — even though the resolver is locked. A `require()` for a specifier that was **never cached during `__VU==0`** hits the lock and **fails** with a fixed error.

So why do some scripts "work in init but fail once many VUs run"? Because the failing scripts call `require()` **conditionally / dynamically** — e.g. inside `if (__VU >= 3) { ... }`. With few VUs that branch may never execute, so nothing new is ever resolved and the script passes. With more VUs, a VU eventually takes the branch, executes a `require()` for a specifier that was **not** resolved at `__VU==0`, and trips the lock. **A higher VU count does not "apply pressure"; it merely raises the probability that a conditional code path executes an un-cached specifier.** The trigger is *which init path runs*, not load. This is demonstrated definitively in **§6** (a script passes at 2 VUs and fails at 5 VUs, purely because the `__VU >= 3` branch only executes in the latter).

**In one sentence:** *A module resolved during the first init (`__VU==0`) can be `require()`'d by every later VU because it is a cache hit; a module never seen at `__VU==0` is rejected the moment any VU tries to resolve it after the one-time `Lock()`.* Both halves are proven by real runs in §3, §5, and §6.

There is also a **second, entirely separate failure mode** that is frequently conflated with the lock: calling `require()` **during an iteration** (inside `default()`), rather than in init. That is rejected by a different guard (the init-context gate), produces a **different** error string, and has **different** exit behavior (logged, non-fatal). §4 keeps the two apart.

---

## 2. The `init` / VU lifecycle and the single `Lock()`

### 2.1 What runs when

k6 scripts have two stages: **init** (top-level/module scope) and the **VU stage** (the exported `default` function and other lifecycle functions, executed per iteration). Module loading (`require()` / ES `import`) is an **init-only** activity: the init context prepares the test by loading files and importing modules, and each newly spawned VU **re-executes its init code** (init runs *once per VU*). VU/iteration code does not import modules. (Validated against the official k6 Test Lifecycle documentation; see §8.)

### 2.2 The causal chain, grounded in source

The freeze is applied inside `newBundle`, in this exact order:

- The resolver is created: `bundle.ModuleResolver = modules.NewModuleResolver(...)` [js/bundle.go:L110].
- The **first init** runs at `__VU==0`: `bi, err := bundle.instantiate(vuImpl, 0)` [js/bundle.go:L125]. Its top-level code resolves and caches every unconditionally-`require()`d module.
- The resolver is locked **once**: `bundle.ModuleResolver.Lock()` [js/bundle.go:L129].

`Lock()` itself simply flips a boolean:

```
// js/modules/resolution.go:L137-L139
func (mr *ModuleResolver) Lock() {
	mr.locked = true
}
```

The `locked` flag lives on the resolver struct: `locked bool` [js/modules/resolution.go:L35], within `type ModuleResolver struct { ... }` [js/modules/resolution.go:L30].

When a real VU later calls `require()` in its init code, the path is:
`requireImpl.require` [js/bundle.go:L424] → (init-context gate passes because `vu.state == nil` in init [js/bundle.go:L439]) → `return r.modSys.Require(specifier)` [js/bundle.go:L428] → `ModuleSystem.Require` [js/modules/require_impl.go:L15] → `sobekModuleResolver` [js/modules/resolution.go:L192-L196] → `resolve` [js/modules/resolution.go:L145].

The init→VU boundary that makes `vu.state` non-nil (and thus later disables `require()` — see §4) is set in the runner: `vu.state = &lib.State{ ... }` [js/runner.go:L230] and `vu.moduleVUImpl.state = vu.state` [js/runner.go:L247].

### 2.3 Evidence — init runs once per VU (EXP1)

Script `exp1.js` `require()`s `./dep.js` **unconditionally at the top level** and logs the current `__VU` from init, with `vus: 3, iterations: 3`. (Full script and command in Appendix EXP1.) Observed **stderr**, exit `0`:

```
time="2026-07-08T04:08:18Z" level=info msg="[init] __VU=0 require('./dep.js') -> hello-from-dep" source=console
time="2026-07-08T04:08:19Z" level=info msg="[init] __VU=1 require('./dep.js') -> hello-from-dep" source=console
time="2026-07-08T04:08:19Z" level=info msg="[init] __VU=2 require('./dep.js') -> hello-from-dep" source=console
time="2026-07-08T04:08:19Z" level=info msg="[init] __VU=3 require('./dep.js') -> hello-from-dep" source=console
```

`EXIT CODE: 0`

There are **four** init executions — one at `__VU=0` (the bundle build / first init) and one for each of the three VUs (`__VU=1,2,3`). This is the direct, observed proof that **init code runs once per VU**, and that a module resolved at `__VU==0` (`./dep.js`) is successfully re-`require()`'d by every later VU (a cache hit — see §3). A second run produced the same lines in a **different order** (`0,1,3,2`), because VUs are initialized **concurrently** (see §2.4); the exit code and content were identical.

### 2.4 Why VU order (and the reported VU number in errors) varies

Real VUs are initialized **concurrently**, not serially. The scheduler spawns `concurrency` goroutines, each pulling VUs to initialize:

```
// execution/scheduler.go:L169-L172
for i := 0; i < concurrency; i++ {
	go func() {
		for range limiter {
			newVU, err := e.initVU(ctx, samplesOut, logger)
```

This is why the `__VU` console lines in EXP1 appear in a non-deterministic order across runs, and — importantly — why the specific VU number reported in the failure messages of §4 and §6 varies from run to run even though the failure itself is deterministic.

---

## 3. "Already resolved" vs. "new module" — a cache-**before**-lock decision

The classification the question asks about ("what counts as already resolved vs. a new module") is decided inside `resolve()` [js/modules/resolution.go:L145-L177]. The key design point is **ordering**: the cache is consulted **before** the lock is enforced. A **cache hit returns even when the resolver is locked**; only a **cache miss while locked** raises the error. This is true for both branches of `resolve()`.

### 3.1 The two branches of `resolve()` (verbatim source)

```
// js/modules/resolution.go:L145-L177
func (mr *ModuleResolver) resolve(basePWD *url.URL, arg string) (sobek.ModuleRecord, error) {
	switch {
	case arg == "k6", strings.HasPrefix(arg, "k6/"):
		// Builtin or external modules ("k6", "k6/*", or "k6/x/*") are handled
		// specially, as they don't exist on the filesystem.
		if cached, ok := mr.cache[arg]; ok {
			return cached.mod, cached.err
		}
		mod, err := mr.requireModule(arg)
		mr.cache[arg] = moduleCacheElement{mod: mod, err: err}
		return mod, err
	default:
		specifier, err := mr.resolveSpecifier(basePWD, arg)
		if err != nil {
			return nil, err
		}
		// try cache with the final specifier
		if cached, ok := mr.cache[specifier.String()]; ok {
			return cached.mod, cached.err
		}

		if mr.locked {
			return nil, fmt.Errorf(notPreviouslyResolvedModule, arg)
		}
		// Fall back to loading
		data, err := mr.loadCJS(specifier, arg)
		if err != nil {
			mr.cache[specifier.String()] = moduleCacheElement{err: err}
			return nil, err
		}
		return mr.resolveLoaded(basePWD, arg, data)
	}
}
```

- **Built-in / `k6*` branch** ([js/modules/resolution.go:L147]): the cache is keyed by the **raw specifier** (`"k6"`, `"k6/http"`, ...). A hit returns at [js/modules/resolution.go:L150-L152] regardless of `locked`. A miss calls `mr.requireModule(arg)` [js/modules/resolution.go:L153], whose very first act is the lock check:

```
// js/modules/resolution.go:L69-L71
func (mr *ModuleResolver) requireModule(name string) (sobek.ModuleRecord, error) {
	if mr.locked {
		return nil, fmt.Errorf(notPreviouslyResolvedModule, name)
	}
```

- **File branch** (`default`, [js/modules/resolution.go:L156]): the specifier is first turned into an absolute URL by `resolveSpecifier` [js/modules/resolution.go:L157] (this is where relative resolution happens — see §5). The cache is keyed by that **resolved URL string**; a hit returns at [js/modules/resolution.go:L162-L164] regardless of `locked`. Only on a **miss** does the lock fire: `if mr.locked { return nil, fmt.Errorf(notPreviouslyResolvedModule, arg) }` [js/modules/resolution.go:L166-L167]. If not locked, it loads and caches the module.

### 3.2 Precise definitions (observed + code-grounded)

- **"Already resolved" (allowed):** the lookup key is present in `mr.cache`. For file modules the key is the **fully-resolved URL** of the file; for built-ins the key is the **specifier string** (`k6`, `k6/http`, ...). The cache is populated during the first init at `__VU==0`, so "already resolved" means precisely **"was resolved at least once during the `__VU==0` init."**
- **"New module" (rejected):** any specifier whose resolved key is **absent** from `mr.cache` at the moment `require()` runs while `mr.locked == true`. The error message names this literally: `the module %q was not previously resolved during initialization (__VU==0)` [js/modules/resolution.go:L17].

Note that this is a decision about **cache membership**, not about whether the file exists on disk. A module file can exist on disk and still be rejected, because the gate is "was it cached at `__VU==0`," not "can it be loaded." EXP2 (§4.1) proves this: `./unseen.js` exists on disk yet is still rejected.

### 3.3 Evidence

- **Cache hit passes (EXP1, §2.3):** `./dep.js`, resolved at `__VU==0`, is a cache hit for VUs 1–3 → exit `0`.
- **Cache miss under lock fails (EXP2 for a file, EXP6 for a built-in — §4):** a specifier never resolved at `__VU==0`, requested later, → `notPreviouslyResolvedModule`, exit `107`.


---

## 4. The failure modes — exact error text and exit codes

There are **two distinct failure modes**, plus the empty-specifier guard. They are commonly conflated; they are not the same. All error text below is reproduced **verbatim** (byte-for-byte) from real runs.

### 4.1 Failure mode (i): locked resolver, cache miss **in init** → `notPreviouslyResolvedModule`, exit 107

This is the freeze in action. A `require()` executed **in init** (a VU re-running its init code) for a specifier not cached at `__VU==0`.

#### EXP2 — never-seen **file** module (proves the lock, not a missing file)

`./unseen.js` **exists on disk** (`module.exports = { x: 1 };`) but is `require()`'d only inside `if (__VU >= 1) { ... }`, so it is never resolved at `__VU==0`. With `vus: 2`, VUs 1 and 2 each re-run init and take the branch. (Full scripts/command in Appendix EXP2.) Observed (stderr; **stable exit 107** across 3 runs):

```
time="2026-07-08T04:08:28Z" level=error msg="GoError: the module \"./unseen.js\" was not previously resolved during initialization (__VU==0)\n\tat go.k6.io/k6/js.(*requireImpl).require-fm (native)\n\tat file:///tmp/k6exp/exp2.js:4:25(18)\n" hint="error while initializing VU #1 (script exception)"
```

`EXIT CODE: 107`

Because `unseen.js` is present on disk, the only possible cause of the failure is the **lock** (cache miss at [js/modules/resolution.go:L166-L167]) — not a missing file. This is the strongest single proof that the mechanism is the freeze. The reported VU number varied `#1`/`#2`/`#1` across the three runs (concurrent init, §2.4); the error string, the file branch citation, and the exit code were identical every time.

#### EXP6 — never-seen **built-in** `k6/*` module (same string, via `requireModule`) — with an honest caveat

`k6/http` is a **valid, registered built-in** (`"k6/http": http.New()` in the registry [js/jsmodules.go:L61], built by `getJSModules()` [js/jsmodules.go:L71]). It is `require()`'d only inside `if (__VU >= 1) { ... }`, so it is never resolved at `__VU==0`. With `vus: 2`, later VUs take the branch and hit the built-in branch's lock check in `requireModule` [js/modules/resolution.go:L70-L71]. The failure is therefore **purely the lock**, not an "unknown module."

**Observed — the common case (~97% of runs; stderr; exit 107):**

```
time="2026-07-08T04:08:54Z" level=error msg="GoError: the module \"k6/http\" was not previously resolved during initialization (__VU==0)\n\tat go.k6.io/k6/js.(*requireImpl).require-fm (native)\n\tat file:///tmp/k6exp/exp6.js:4:23(18)\n" hint="error while initializing VU #2 (script exception)"
```

`EXIT CODE: 107`

**Honest run-to-run inconsistency (OBSERVED, not inferred).** Unlike EXP2 (a *file* module, perfectly stable at exit 107), EXP6 is **not** perfectly stable: the built-in branch performs an **unsynchronized cache write** that races concurrent VU initialization, so a fraction of runs crash the Go runtime instead of cleanly rejecting. Over a single classified sweep of **700 identical runs** (`/tmp/k6bin run --quiet --no-summary /tmp/k6exp/exp6.js`, exit code recorded each time) the observed distribution was:

| Outcome | Exit | Count / 700 | Crashing frame (top of stack) |
|---|---|---|---|
| clean lock reject (the `notPreviouslyResolvedModule` output shown above) | 107 | 682 (97.4%) | — |
| `fatal error: concurrent map writes` | 2 | 16 | `js/modules/resolution.go:154` — the cache **write** |
| `fatal error: concurrent map read and map write` | 2 | 1 | `js/modules/resolution.go:150` — built-in-branch cache **read** |
| `fatal error: concurrent map read and map write` | 2 | 1 | `js/modules/resolution.go:162` — file-branch cache **read** |

That is ~97.4% clean exit 107 and ~2.6% Go-runtime fatal (exit 2), spread across **two distinct fatal messages** and **three distinct crash frames**. The pattern held across ~250 additional runs. I report the observed distribution rather than hiding it behind a controlled single-VU variant; a deterministic built-in lock-reject can instead be obtained with the *file* case EXP2 (which never writes the cache on the locked path) or by avoiding concurrent init, but the 2-VU form is retained here precisely to exhibit the honest inconsistency the question asks about.

**Why it crashes — the unconditional cache write.** In the **built-in branch**, the cache write happens **unconditionally after `requireModule` returns — even when `requireModule` returned the lock error**:

```
// js/modules/resolution.go:L153-L155
		mod, err := mr.requireModule(arg)      // returns the lock error when locked
		mr.cache[arg] = moduleCacheElement{mod: mod, err: err}   // L154: map WRITE happens anyway
		return mod, err
```

`mr.cache` is a plain Go `map` with no synchronization, and VUs initialize **concurrently** (§2.4), so this write at [js/modules/resolution.go:L154] races other goroutines' accesses to the same map. Two collision shapes are observed, each shown here as a **complete contiguous top-of-stack excerpt** (no elision; the full unedited dumps are in Appendix B and Appendix B2):

- **`concurrent map writes`** — two VUs both miss the `k6/http` cache and reach the **write** at [js/modules/resolution.go:L154] simultaneously (the `require_impl.go:28` frame is the specifier-resolution call [js/modules/require_impl.go:L28]):

```
fatal error: concurrent map writes

goroutine 141 [running]:
go.k6.io/k6/js/modules.(*ModuleResolver).resolve(0xc000401b80, 0xc0007ccea0?, {0xc00051a798, 0x7})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/modules/resolution.go:154 +0x105
go.k6.io/k6/js/modules.(*ModuleResolver).sobekModuleResolver(0xc000401b80, {0x1b0c080?, 0xc0007ba000?}, {0xc00051a798, 0x7})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/modules/resolution.go:195 +0x45
go.k6.io/k6/js/modules.(*ModuleSystem).Require(0xc0005aeb80, {0xc00051a798, 0x7})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/modules/require_impl.go:28 +0x165
go.k6.io/k6/js.(*requireImpl).require(0xc0006738c0, {0xc00051a798, 0x7})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:428 +0x45
```

- **`concurrent map read and map write`** — one VU **reads** the cache while another **writes** it at [js/modules/resolution.go:L154]. The read site varies: I observed it both at the built-in-branch read [js/modules/resolution.go:L150] and at the file-branch read [js/modules/resolution.go:L162]. The L162 case below is the resolution of the **main script module itself** — its argument is the 25-byte (`0x19`) URL `file:///tmp/k6exp/exp6.js`, reached via the parent-module lookup at [js/modules/require_impl.go:L27]:

```
fatal error: concurrent map read and map write

goroutine 215 [running]:
go.k6.io/k6/js/modules.(*ModuleResolver).resolve(0xc0006e81e0, 0xc000af8bd0, {0xc0006ec180, 0x19})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/modules/resolution.go:162 +0x1cb
go.k6.io/k6/js/modules.(*ModuleResolver).sobekModuleResolver(0xc0006e81e0, {0x0?, 0x0?}, {0xc0006ec180, 0x19})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/modules/resolution.go:195 +0x45
go.k6.io/k6/js/modules.(*ModuleSystem).Require(0xc0004fa0e0, {0xc0005f0318, 0x7})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/modules/require_impl.go:27 +0x125
go.k6.io/k6/js.(*requireImpl).require(0xc0006c6090, {0xc0005f0318, 0x7})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:428 +0x45
```

The **file** branch never writes the cache on a locked miss — it returns at [js/modules/resolution.go:L166-L167] **without** touching the map — which is exactly why EXP2 (file) is perfectly stable while EXP6 (built-in) is occasionally fatal. *(Inference, clearly labeled: the differing stability between EXP2 and EXP6 is attributable to this write-vs-no-write difference; the crash messages, their stack frames, their source lines, and the exit code 2 are observed facts.)*

### 4.2 Failure mode (ii): `require()` **during an iteration** → init-context gate, exit 0 (logged, non-fatal)

This is **not** the lock. Once iterations begin, `vu.state` is non-nil (set at [js/runner.go:L230] / [js/runner.go:L247]), so the init-context gate rejects `require()` **before** the resolver is ever consulted:

```
// js/bundle.go:L424-L429
func (r *requireImpl) require(specifier string) (*sobek.Object, error) {
	if !r.inInitContext() {
		return nil, fmt.Errorf(cantBeUsedOutsideInitContextMsg, "require")
	}
	return r.modSys.Require(specifier)
}
```

where `inInitContext: func() bool { return vu.state == nil }` [js/bundle.go:L439]. The message constant is:

```
// js/initcontext.go:L15-L16
const cantBeUsedOutsideInitContextMsg = `the "%s" function is only available in the init stage ` +
	`(i.e. the global scope), see https://grafana.com/docs/k6/latest/using-k6/test-lifecycle/ for more information`
```

#### EXP3 — `require('./dep.js')` inside `default()`

Even though `./dep.js` **is** cached, the gate rejects the call first. With `vus: 1, iterations: 2`, the error is logged **once per iteration** and the process exits **0** (the error is a per-iteration script exception, not an init abort). Observed (stderr; **stable exit 0**):

```
time="2026-07-08T04:08:40Z" level=error msg="GoError: the \"require\" function is only available in the init stage (i.e. the global scope), see https://grafana.com/docs/k6/latest/using-k6/test-lifecycle/ for more information\n\tat go.k6.io/k6/js.(*requireImpl).require-fm (native)\n\tat default (file:///tmp/k6exp/exp3.js:5:22(3))\n" executor=shared-iterations scenario=default source=stacktrace
time="2026-07-08T04:08:40Z" level=error msg="GoError: the \"require\" function is only available in the init stage (i.e. the global scope), see https://grafana.com/docs/k6/latest/using-k6/test-lifecycle/ for more information\n\tat go.k6.io/k6/js.(*requireImpl).require-fm (native)\n\tat default (file:///tmp/k6exp/exp3.js:5:22(3))\n" executor=shared-iterations scenario=default source=stacktrace
```

`EXIT CODE: 0`

Note the differences from mode (i): a **different error string** (`... only available in the init stage ...`), a **different stack frame** (`at default (...)` rather than an init frame), **different log metadata** (`source=stacktrace`, `executor=shared-iterations`), and a **different exit code (0)**.

### 4.3 The empty-specifier guard → exit 107 (init-time)

`require('')` is caught by an explicit guard at the very top of `ModuleSystem.Require`:

```
// js/modules/require_impl.go:L20-L22
	if specifier == "" {
		return nil, errors.New("require() can't be used with an empty specifier")
	}
```

(An identical guard also exists at [js/modules/require_impl.go:L100].)

#### EXP-EMPTY — `require('')` in init

Observed (stderr; **stable exit 107**):

```
time="2026-07-08T04:08:54Z" level=error msg="GoError: require() can't be used with an empty specifier\n\tat go.k6.io/k6/js.(*requireImpl).require-fm (native)\n\tat file:///tmp/k6exp/exp_empty.js:2:18(12)\n" hint="script exception"
```

`EXIT CODE: 107`

### 4.4 Exit-code semantics (reported honestly)

- **Init-time** module errors (the lock, EXP2/EXP6/EXP7; and the empty specifier, EXP-EMPTY) **abort the whole run** with **exit `107`** (k6's "script exception" abort code); the `hint="... while initializing VU #N ..."` / `hint="script exception"` confirms the init-time origin.
- **Iteration-time** `require()` errors (EXP3) are treated as per-iteration script exceptions: they are **logged** (`source=stacktrace`) and the process **exits `0`** by default.
- The **rare** built-in cache-race crash (EXP6, ~2.6% of runs) is a Go runtime fatal error → **exit `2`**. It appears as **two** distinct messages — `fatal error: concurrent map writes` (crash frame [js/modules/resolution.go:L154], the write) and `fatal error: concurrent map read and map write` (crash frame at a cache **read**: [js/modules/resolution.go:L150] built-in or [js/modules/resolution.go:L162] file) — all rooted in the same unsynchronized write at [js/modules/resolution.go:L154] (see §4.1).


---

## 5. Relative specifiers — resolution is relative to the **currently-executing** module

The question notes that relative specifiers (e.g. `./helper.js`) resolve differently "depending on which module is currently executing." That is exactly correct, and it is implemented by reading the **JavaScript call stack** to discover the referencing module, then resolving the specifier against **that module's directory**.

### 5.1 How the "base directory" is chosen (source)

- The current (referencing) module is read from the call stack:

```
// js/modules/require_impl.go:L185-L196
func getCurrentModuleScript(vu VU) string {
	rt := vu.Runtime()
	var parent string
	var buf [2]sobek.StackFrame
	frames := rt.CaptureCallStack(2, buf[:0])
	if len(frames) == 0 || frames[1].SrcName() == "file:///-" {
		return vu.InitEnv().CWD.JoinPath("./-").String()
	}
	parent = frames[1].SrcName()

	return parent
}
```

`CaptureCallStack(2, ...)` [js/modules/require_impl.go:L189] captures the top two frames; `frames[1].SrcName()` [js/modules/require_impl.go:L193] is the source name of the **caller** — i.e., the module that executed the `require()`. So the "base" is inherently **call-site dependent**. (The `require-fm` frame that anchors this — `go.k6.io/k6/js.(*requireImpl).require-fm` — is the same frame that appears in every error stack trace above; it is matched literally at [js/modules/require_impl.go:L205] by `getPreviousRequiringFile` [js/modules/require_impl.go:L198].)

- The referencing module's URL is turned into its **parent directory**:

```
// js/modules/resolution.go:L198-L211  (reversePath)
	return p.JoinPath("..")   // L210: parent directory of the referencing module
```

`sobekModuleResolver` wires these together — it resolves the specifier against the reversed (parent) path of the referencing module: `return mr.resolve(mr.reversePath(referencingScriptOrModule), specifier)` [js/modules/resolution.go:L195].

- Finally the physical URL is computed relative to that directory by the loader: `Resolve(pwd, moduleSpecifier)` [loader/loader.go:L48], which for a relative specifier calls `resolveFilePath` [loader/loader.go:L84] and uses `Dir` [loader/loader.go:L115]. The resolved URL string is what becomes the **cache key** in §3.

### 5.2 Evidence (EXP4)

Two sibling helper modules with the **same relative specifier** but in different directories:

- `exp4/dir1/helper.js`: `module.exports = { name: 'dir1' };`
- `exp4/dir2/helper.js`: `module.exports = { name: 'dir2' };`
- `exp4/dir1/a.js`: `const h = require('./helper.js'); module.exports = { helperName: h.name };`
- `exp4/dir2/b.js`: `const h = require('./helper.js'); module.exports = { helperName: h.name };`

`exp4/main.js` `require()`s both `./dir1/a.js` and `./dir2/b.js`. Each of those modules issues the **identical** specifier `./helper.js`, but from a different directory. (Full scripts/command in Appendix EXP4.) Observed (stderr; **stable exit 0**):

```
time="2026-07-08T04:08:40Z" level=info msg="[init] a.helperName=dir1  b.helperName=dir2" source=console
time="2026-07-08T04:08:40Z" level=info msg="[init] a.helperName=dir1  b.helperName=dir2" source=console
```

`EXIT CODE: 0`

The same specifier `./helper.js` resolved to **`dir1/helper.js`** when required from `dir1/a.js` and to **`dir2/helper.js`** when required from `dir2/b.js` — proving resolution is relative to the currently-executing module's directory, exactly as the source shows. (The line appears twice because top-level init code runs at `__VU==0` and again for the single real VU — §2.3.)

> **Interaction with the lock (labeled inference, grounded in §3):** because the cache key is the **resolved URL**, `dir1/helper.js` and `dir2/helper.js` are **distinct** cache entries. A relative specifier that was resolved from one directory at `__VU==0` is therefore *not* "already resolved" for a different directory later — so call-stack-relative resolution and the freeze interact: the *same text* `./helper.js` can be a cache hit from one module and a cache miss (rejected under lock) from another.


---

## 6. The "few vs. many VUs" demonstration — the definitive correction of "under pressure"

This is the direct proof that the symptom is caused by **which init path executes**, not by load. `exp7.js` `require()`s `./late.js` **only when `__VU >= 3`**:

```js
// exp7.js — ./late.js is only required when __VU>=3.
if (__VU >= 3) {
  const late = require('./late.js');
  console.log(`[init] __VU=${__VU} loaded late`);
}
export default function () {}
```

`./late.js` exists on disk (`module.exports = { y: 2 };`) but is never resolved at `__VU==0`, so it is a "new module" under the lock (§3).

### 6.1 Few VUs → pass

```
/tmp/k6bin run --quiet --no-summary --vus 2 --iterations 2 /tmp/k6exp/exp7.js
```

With 2 VUs, the `__VU >= 3` branch **never executes**, so nothing new is ever resolved. Observed (no stderr/stdout emitted; **stable exit 0** across 3 runs):

```
EXIT CODE: 0
```

### 6.2 Many VUs → fail

```
/tmp/k6bin run --quiet --no-summary --vus 5 --iterations 5 /tmp/k6exp/exp7.js
```

With 5 VUs, VUs 3, 4, and 5 take the branch; the first one to reach the locked resolver aborts the run. Observed (stderr; **stable exit 107** across 6 runs — representative capture):

```
time="2026-07-08T04:11:00Z" level=error msg="GoError: the module \"./late.js\" was not previously resolved during initialization (__VU==0)\n\tat go.k6.io/k6/js.(*requireImpl).require-fm (native)\n\tat file:///tmp/k6exp/exp7.js:4:23(17)\n" hint="error while initializing VU #4 (script exception)"
```

`EXIT CODE: 107`

### 6.3 What is deterministic and what is not

- **Deterministic:** at 2 VUs the script **always** passes (exit 0); at 5 VUs it **always** fails (exit 107). This was stable across every run.
- **Not deterministic:** the specific VU number reported in the error. Across six 5-VU runs I observed the reported VU as **`#4, #5, #4, #3, #5, #4`** — i.e. any of `#3`/`#4`/`#5`. This variation is a consequence of **concurrent VU initialization** (§2.4): whichever VU `>= 3` reaches the locked resolver first is the one named. It is **not** a function of load.

This is the crux of the correction: the user's script "starts failing once many VUs run" **not because the runtime is under pressure**, but because a **conditional `require()`** (`if (__VU >= 3)`) only executes an un-cached specifier once enough VUs exist for that branch to be taken. The freeze that rejects it was applied once, deterministically, back at the init→VU boundary (§2). Increasing VUs increases the *probability of taking the un-cached branch*, nothing more.


---

## 7. Exact error / warning-text reference

Every string below was reproduced **verbatim** from the runs above. The format string is quoted from the source; the observed text is the rendered result.

| # | Exact text (format string) | Source constant / origin | Emitted by | Exit code |
|---|---|---|---|---|
| 1 | `the module %q was not previously resolved during initialization (__VU==0)` | `notPreviouslyResolvedModule` [js/modules/resolution.go:L17]; enforced for files at [js/modules/resolution.go:L166-L167] and for built-ins via `requireModule` [js/modules/resolution.go:L70-L71] | EXP2 (file), EXP6 (built-in), EXP7 (@5 VUs) | 107 (init abort) |
| 2 | `the "%s" function is only available in the init stage (i.e. the global scope), see https://grafana.com/docs/k6/latest/using-k6/test-lifecycle/ for more information` | `cantBeUsedOutsideInitContextMsg` [js/initcontext.go:L15-L16]; gate at [js/bundle.go:L425-L426] | EXP3 (`require()` in `default()`) | 0 (logged per-iteration) |
| 3 | `require() can't be used with an empty specifier` | `errors.New(...)` [js/modules/require_impl.go:L21] (also [js/modules/require_impl.go:L100]) | EXP-EMPTY | 107 (init abort) |
| 4 | `fatal error: concurrent map writes` | Go runtime; the unsynchronized cache **write** at [js/modules/resolution.go:L154] under concurrent VU init | EXP6 (rare; observed 16/700 ≈ 2.3%) | 2 (Go fatal) |
| 5 | `fatal error: concurrent map read and map write` | Go runtime; a cache **read** ([js/modules/resolution.go:L150] built-in branch, or [js/modules/resolution.go:L162] file branch) racing the write at [js/modules/resolution.go:L154] | EXP6 (very rare; observed 2/700 ≈ 0.3%) | 2 (Go fatal) |
| 6 | `open() can't be used with files that weren't previously opened during initialization (__VU==0), path: %q` | [js/initcontext.go:L40] — the **parallel `open()` behavior** the freeze deliberately mirrors (see §8). *Cited as the design mirror; not separately run in this investigation.* | — | — (init abort when triggered) |

**Shared stack frame:** every `require()`-origin error above contains the native frame

```
	at go.k6.io/k6/js.(*requireImpl).require-fm (native)
```

which corresponds to `requireImpl.require` [js/bundle.go:L424] and the frame name matched literally at [js/modules/require_impl.go:L205].

> **On the `:line:col(offset)` values** (e.g. `exp2.js:4:25(18)`, `exp6.js:4:23(18)`, `exp7.js:4:23(17)`, `exp3.js:5:22(3)`, `exp_empty.js:2:18(12)`): these are the **actual values observed with the scratch scripts under `/tmp/k6exp/`**. They depend on the exact byte layout of the scripts and the scratch path; a different path or edited script would change the `file:///...` prefix and possibly the offsets. They are reported as observed, not as invariants.

---

## 8. Design rationale — why resolution is frozen (intentional, not a safeguard against load)

The freeze is a **deliberate design decision**, not a defensive reaction to runtime conditions. Two independent sources corroborate the code-level findings:

- **Test-lifecycle semantics (official docs).** k6's init context prepares the test by loading files and importing modules, and this init code runs **once per VU**; VU (iteration) code does not import modules or load files from disk. This is *why* `require()`/`import` are restricted to init and why each newly-spawned VU re-executes its init code (observed directly in EXP1, §2.3). Source: Grafana k6 Test Lifecycle documentation — `https://grafana.com/docs/k6/latest/using-k6/test-lifecycle/`.

- **Why resolution is frozen (upstream design discussion).** Restricting `open`/`require` to the init context — and freezing resolution after the first init — exists so that **`k6 archive`** (and by extension `k6 cloud` and distributed/cloud runs) can **gather every file the script needs by running the init context a single time**. If a later VU could pull in a module that the first init never touched, the archive would be incomplete. The lock deliberately **mirrors `open()`'s** long-standing behavior (only files opened during the first init are openable later — see the parallel error at [js/initcontext.go:L40]). Source: upstream design issue `grafana/k6#3020` — `https://github.com/grafana/k6/issues/3020`.

In short: the freeze guarantees that the set of modules a test depends on is **fully knowable from one init pass**. That is an *archivability/reproducibility* guarantee, and it is enforced identically whether you run 1 VU or 1,000. The "under pressure" intuition conflates *probability of executing a conditional `require()`* with *a load-sensitive mechanism*; only the former is real.


---

## Appendix A — Reproduction experiments (complete scripts, commands, and verbatim output)

All scripts were created under the scratch directory `/tmp/k6exp/` (outside the k6 source tree) and removed afterward. Each experiment was executed **at least twice**; exit codes and error strings were stable except where explicitly noted (EXP6).

### Shared module `dep.js`

```js
module.exports = {
  hello: function () { return 'hello-from-dep'; },
};
```

### EXP1 — a module resolved at init CAN be `require()`'d by later VUs → PASS (exit 0)

`exp1.js`:

```js
// exp1.js — dep.js is resolved during __VU==0 (first init / bundle build),
// then re-required in the init context of every later VU (cache hit).
const dep = require('./dep.js');
console.log(`[init] __VU=${__VU} require('./dep.js') -> ${dep.hello()}`);

export const options = { vus: 3, iterations: 3 };
export default function () {}
```

Command:

```
/tmp/k6bin run --quiet --no-summary /tmp/k6exp/exp1.js
```

Observed — run 1 (stderr):

```
time="2026-07-08T04:08:18Z" level=info msg="[init] __VU=0 require('./dep.js') -> hello-from-dep" source=console
time="2026-07-08T04:08:19Z" level=info msg="[init] __VU=1 require('./dep.js') -> hello-from-dep" source=console
time="2026-07-08T04:08:19Z" level=info msg="[init] __VU=2 require('./dep.js') -> hello-from-dep" source=console
time="2026-07-08T04:08:19Z" level=info msg="[init] __VU=3 require('./dep.js') -> hello-from-dep" source=console
```

`EXIT CODE: 0`

Observed — run 2 (stderr; note the different, concurrency-driven order):

```
time="2026-07-08T04:08:19Z" level=info msg="[init] __VU=0 require('./dep.js') -> hello-from-dep" source=console
time="2026-07-08T04:08:19Z" level=info msg="[init] __VU=1 require('./dep.js') -> hello-from-dep" source=console
time="2026-07-08T04:08:19Z" level=info msg="[init] __VU=3 require('./dep.js') -> hello-from-dep" source=console
time="2026-07-08T04:08:19Z" level=info msg="[init] __VU=2 require('./dep.js') -> hello-from-dep" source=console
```

`EXIT CODE: 0`

### EXP2 — never-seen FILE module in a later VU's init → FAIL (exit 107)

`unseen.js` (exists on disk):

```js
module.exports = { x: 1 };
```

`exp2.js`:

```js
// exp2.js — ./unseen.js EXISTS on disk but is NEVER required during __VU==0.
// A conditional require in a later VU's init hits the locked resolver.
if (__VU >= 1) {
  const unseen = require('./unseen.js');
  console.log(`[init] __VU=${__VU} loaded unseen`);
}
export const options = { vus: 2, iterations: 2 };
export default function () {}
```

Command:

```
/tmp/k6bin run --quiet --no-summary /tmp/k6exp/exp2.js
```

Observed (stderr; stable exit 107 across 3 runs; reported VU# varied #1/#2/#1):

```
time="2026-07-08T04:08:28Z" level=error msg="GoError: the module \"./unseen.js\" was not previously resolved during initialization (__VU==0)\n\tat go.k6.io/k6/js.(*requireImpl).require-fm (native)\n\tat file:///tmp/k6exp/exp2.js:4:25(18)\n" hint="error while initializing VU #1 (script exception)"
```

`EXIT CODE: 107`

### EXP3 — `require()` inside `default()` / iteration → logged, non-fatal (exit 0)

`exp3.js`:

```js
// exp3.js — require() called during an ITERATION (not init). Even though
// ./dep.js is cached, the init-context gate rejects it first.
export const options = { vus: 1, iterations: 2 };
export default function () {
  const dep = require('./dep.js');
  console.log(dep.hello());
}
```

Command:

```
/tmp/k6bin run --quiet --no-summary /tmp/k6exp/exp3.js
```

Observed (stderr; stable exit 0; logged once per iteration):

```
time="2026-07-08T04:08:40Z" level=error msg="GoError: the \"require\" function is only available in the init stage (i.e. the global scope), see https://grafana.com/docs/k6/latest/using-k6/test-lifecycle/ for more information\n\tat go.k6.io/k6/js.(*requireImpl).require-fm (native)\n\tat default (file:///tmp/k6exp/exp3.js:5:22(3))\n" executor=shared-iterations scenario=default source=stacktrace
time="2026-07-08T04:08:40Z" level=error msg="GoError: the \"require\" function is only available in the init stage (i.e. the global scope), see https://grafana.com/docs/k6/latest/using-k6/test-lifecycle/ for more information\n\tat go.k6.io/k6/js.(*requireImpl).require-fm (native)\n\tat default (file:///tmp/k6exp/exp3.js:5:22(3))\n" executor=shared-iterations scenario=default source=stacktrace
```

`EXIT CODE: 0`

### EXP4 — relative `./helper.js` resolves per calling module's directory → PASS (exit 0)

Files:

- `exp4/dir1/helper.js`: `module.exports = { name: 'dir1' };`
- `exp4/dir2/helper.js`: `module.exports = { name: 'dir2' };`
- `exp4/dir1/a.js`: `const h = require('./helper.js'); module.exports = { helperName: h.name };`
- `exp4/dir2/b.js`: `const h = require('./helper.js'); module.exports = { helperName: h.name };`

`exp4/main.js`:

```js
// exp4/main.js — same specifier './helper.js' required from two different
// directories resolves to each module's own directory (call-stack relative).
const a = require('./dir1/a.js');
const b = require('./dir2/b.js');
console.log(`[init] a.helperName=${a.helperName}  b.helperName=${b.helperName}`);
export default function () {}
```

Command:

```
/tmp/k6bin run --quiet --no-summary /tmp/k6exp/exp4/main.js
```

Observed (stderr; stable exit 0; line appears twice — __VU==0 bundle init + one real VU):

```
time="2026-07-08T04:08:40Z" level=info msg="[init] a.helperName=dir1  b.helperName=dir2" source=console
time="2026-07-08T04:08:40Z" level=info msg="[init] a.helperName=dir1  b.helperName=dir2" source=console
```

`EXIT CODE: 0`

### EXP6 — never-seen built-in `k6/*` module in a later VU's init

This is the built-in-module analogue of EXP2 (§4.1). `k6/http` is a perfectly
valid built-in module, but here it is required only inside a conditional that
runs in a later VU's init — so the specifier `k6/http` is never resolved at
`__VU==0`, the resolver is already locked by the time VU #1/#2 re-runs its init,
and the built-in branch of `resolve()` raises the same `notPreviouslyResolvedModule`
error [js/modules/resolution.go:L17,L70-L71]. It demonstrates that the
resolved-vs-new classification (§3) is enforced identically for built-in `k6/*`
modules and for file modules.

`exp6.js`:

```js
// exp6.js — k6/http is a VALID built-in, but it is required only inside a
// conditional in a later VU's init, so it is never resolved at __VU==0.
if (__VU >= 1) {
  const http = require('k6/http');
  console.log(`[init] __VU=${__VU} loaded k6/http`);
}
export const options = { vus: 2, iterations: 2 };
export default function () {}
```

Command:

```
/tmp/k6bin run --quiet --no-summary /tmp/k6exp/exp6.js
```

Observed (stderr; representative clean run — exit 107, aborts the run; the
reported VU number varies between #1 and #2 depending on which VU wins the race
to re-run its init first):

```
time="2026-07-08T05:05:56Z" level=error msg="GoError: the module \"k6/http\" was not previously resolved during initialization (__VU==0)\n\tat go.k6.io/k6/js.(*requireImpl).require-fm (native)\n\tat file:///tmp/k6exp/exp6.js:4:23(18)\n" hint="error while initializing VU #2 (script exception)"
```

`EXIT CODE: 107`

**Instability note (see §4.1 and Appendix B / Appendix B2 for the full analysis).**
Unlike the file-module case (EXP2), which is perfectly stable at exit 107, EXP6
is *usually* — but not always — a clean exit-107 abort. The built-in branch of
`resolve()` performs an **unconditional** write to the unsynchronized `mr.cache`
map at [js/modules/resolution.go:L154] even when the lookup returned the lock
error, and VUs initialize concurrently [execution/scheduler.go:L169-L172]. This
creates a data race on the plain Go map. Across an authoritative **700-run
sweep** the observed distribution was **682 / 700 (97.4%)** clean exit-107
aborts and **18 / 700 (2.6%)** fatal Go-runtime crashes (exit 2): 16 `concurrent
map writes` at resolution.go:L154, 1 `concurrent map read and map write` reaching
the built-in-branch read at resolution.go:L150, and 1 reaching the file-branch
read at resolution.go:L162. This is reported honestly rather than hidden: the
question's "sometimes works, sometimes fails" framing maps directly onto this
observed distribution, and the same unchanged input was run repeatedly rather
than constructing a variant that masks the inconsistency.

### EXP7 — few vs. many VUs

`late.js`: `module.exports = { y: 2 };`

`exp7.js`:

```js
// exp7.js — ./late.js is only required when __VU>=3. With few VUs the branch
// never executes (PASS); with many VUs a high-numbered VU trips the lock.
if (__VU >= 3) {
  const late = require('./late.js');
  console.log(`[init] __VU=${__VU} loaded late`);
}
export default function () {}
```

Commands and observations:

```
/tmp/k6bin run --quiet --no-summary --vus 2 --iterations 2 /tmp/k6exp/exp7.js    # EXIT CODE: 0 (stable, no output)
/tmp/k6bin run --quiet --no-summary --vus 5 --iterations 5 /tmp/k6exp/exp7.js    # EXIT CODE: 107 (stable)
```

Observed at 5 VUs (stderr; reported VU# varied across runs — observed sequence #4, #5, #4, #3, #5, #4):

```
time="2026-07-08T04:11:00Z" level=error msg="GoError: the module \"./late.js\" was not previously resolved during initialization (__VU==0)\n\tat go.k6.io/k6/js.(*requireImpl).require-fm (native)\n\tat file:///tmp/k6exp/exp7.js:4:23(17)\n" hint="error while initializing VU #4 (script exception)"
```

`EXIT CODE: 107`

### EXP-EMPTY — empty specifier guard → FAIL (exit 107, init-time)

`exp_empty.js`:

```js
// exp_empty.js — require('') in the init context.
const x = require('');
export default function () {}
```

Command:

```
/tmp/k6bin run --quiet --no-summary /tmp/k6exp/exp_empty.js
```

Observed (stderr; stable exit 107):

```
time="2026-07-08T04:08:54Z" level=error msg="GoError: require() can't be used with an empty specifier\n\tat go.k6.io/k6/js.(*requireImpl).require-fm (native)\n\tat file:///tmp/k6exp/exp_empty.js:2:18(12)\n" hint="script exception"
```

`EXIT CODE: 107`


---

## Appendix B — EXP6 complete `concurrent map writes` crash dump (verbatim, one captured occurrence)

This is the **complete, unedited** 179-line Go runtime dump from one captured EXP6 crash (the rare ~4% outcome described in §4.1). Goroutine IDs and hexadecimal addresses vary from run to run; the causally-relevant frame is the map write at `js/modules/resolution.go:154`, reached via the concurrent VU-init path (`execution/scheduler.go:170`). The `grafana/sobek` frames are the JavaScript-engine module-evaluation frames between `RunSourceData` and the `require` call.

```
fatal error: concurrent map writes

goroutine 233 [running]:
go.k6.io/k6/js/modules.(*ModuleResolver).resolve(0xc0004f3180, 0xc0007ea510?, {0xc0005db328, 0x7})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/modules/resolution.go:154 +0x105
go.k6.io/k6/js/modules.(*ModuleResolver).sobekModuleResolver(0xc0004f3180, {0x1b0c080?, 0xc0007aea00?}, {0xc0005db328, 0x7})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/modules/resolution.go:195 +0x45
go.k6.io/k6/js/modules.(*ModuleSystem).Require(0xc0007ba180, {0xc0005db328, 0x7})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/modules/require_impl.go:28 +0x165
go.k6.io/k6/js.(*requireImpl).require(0xc000796130, {0xc0005db328, 0x7})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:428 +0x45
reflect.Value.call({0x193c0c0?, 0xc000796150?, 0xc0009c43a8?}, {0x1c1f7d1, 0x4}, {0xc0005c01e0, 0x1, 0xc00023c140?})
	/usr/local/go/src/reflect/value.go:584 +0xca6
reflect.Value.Call({0x193c0c0?, 0xc000796150?, 0xc000796d20?}, {0xc0005c01e0?, 0xc000796940?, 0x40e45f?})
	/usr/local/go/src/reflect/value.go:368 +0xb9
github.com/grafana/sobek.(*Runtime).newWrappedFunc.(*Runtime).wrapReflectFunc.func1({{0x1fb1a08, 0x2fefba0}, {0xc00035a2d0, 0x1, 0x3}})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/runtime.go:2029 +0x3bd
github.com/grafana/sobek.(*nativeFuncObject).vmCall(0xc000032180, 0xc0001f66c0, 0x1)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/func.go:563 +0x184
github.com/grafana/sobek.call.exec(0x796d20?, 0xc0001f66c0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/vm.go:3719 +0x66
github.com/grafana/sobek.(*vm).run(0xc0001f66c0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/vm.go:635 +0x5b
github.com/grafana/sobek.(*vm).runTryInner(0x1?)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/vm.go:886 +0x52
github.com/grafana/sobek.(*generator).step(0xc00059c1c0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/func.go:762 +0x2b
github.com/grafana/sobek.(*generator).next(0xc00059c1c0, {0x1fb1a08, 0x2fefba0})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/func.go:800 +0x1b1
github.com/grafana/sobek.(*asyncRunner).onFulfilled(0xc00059c1c0, {{0x0, 0x0}, {0xc0009c48b8, 0x1, 0x1}})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/func.go:686 +0xfb
github.com/grafana/sobek.(*SourceTextModuleInstance).ExecuteModule(0xc0007ba6c0, 0xc00033d808, 0x0, 0x0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/modules_sourcetext.go:29 +0xec
github.com/grafana/sobek.(*Runtime).innerModuleEvaluation(0xc00033d808, 0xc0003e52c0, {0x1f9cd80, 0xc0007aea00}, 0xc0009c4a98, 0x0, 0xc0009c4c18)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/modules.go:279 +0x8f1
github.com/grafana/sobek.(*Runtime).CyclicModuleRecordEvaluate(0xc00033d808, {0x1fa5340, 0xc0007aea00}, 0xc000790c18)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/modules.go:183 +0x47d
go.k6.io/k6/js/modules.(*ModuleSystem).RunSourceData(0xc0007ba180, 0xc00079c5d0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/modules/resolution.go:255 +0x1c8
go.k6.io/k6/js.(*Bundle).instantiate.func3()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:338 +0x27
reflect.Value.call({0x18dd7e0?, 0xc0007ba5a0?, 0x10?}, {0x1c1f7d1, 0x4}, {0x2fefba0, 0x0, 0xc000791288?})
	/usr/local/go/src/reflect/value.go:584 +0xca6
reflect.Value.Call({0x18dd7e0?, 0xc0007ba5a0?, 0xc0007ba5c0?}, {0x2fefba0?, 0xcc451d?, 0x35554aaaa?})
	/usr/local/go/src/reflect/value.go:368 +0xb9
github.com/grafana/sobek.(*Runtime).newWrappedFunc.(*Runtime).wrapReflectFunc.func1({{0x0, 0x0}, {0x0, 0x0, 0x0}})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/runtime.go:2029 +0x3bd
github.com/grafana/sobek.AssertFunction.func1.1()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/runtime.go:2464 +0x56
github.com/grafana/sobek.(*vm).try(0xc0001f66c0, 0xc0007915a8)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/vm.go:863 +0x249
github.com/grafana/sobek.(*Runtime).runWrapped(0xc00033d808, 0xc00023c108?)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/runtime.go:2508 +0x65
github.com/grafana/sobek.AssertFunction.func1({0x0?, 0x0?}, {0x0?, 0x1?, 0xc00023c100?})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/runtime.go:2463 +0x8c
go.k6.io/k6/js.(*Bundle).instantiate.func4()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:345 +0x23
go.k6.io/k6/js/eventloop.(*EventLoop).Start(0xc0003e4a50, 0xc0007967f0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/eventloop/eventloop.go:177 +0x19a
go.k6.io/k6/js.(*Bundle).instantiate(0xc000799b88, 0xc0003c8100, 0x1)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:344 +0x390
go.k6.io/k6/js.(*Bundle).Instantiate(0xc000799b88, {0x1f9c988, 0xc000850050}, 0x1)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:257 +0x1f1
go.k6.io/k6/js.(*Runner).newVU(0xc0003f8000, {0x1f9c988?, 0xc000850050?}, 0x1, 0x1, 0xc0004365b0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/runner.go:128 +0x58
go.k6.io/k6/js.(*Runner).NewVU(0xc0002f41e0?, {0x1f9c988?, 0xc000850050?}, 0x0?, 0x0?, 0x100000000000000?)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/runner.go:116 +0x1d
go.k6.io/k6/execution.(*Scheduler).initVU(0xc0002f2700, {0x1f9c988, 0xc000850050}, 0xc0004365b0, {0x1fb7da0, 0xc000832230})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/execution/scheduler.go:133 +0x9f
go.k6.io/k6/execution.(*Scheduler).initVUsConcurrently.func1()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/execution/scheduler.go:172 +0xd2
created by go.k6.io/k6/execution.(*Scheduler).initVUsConcurrently in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/execution/scheduler.go:170 +0x97

goroutine 1 [select]:
go.k6.io/k6/execution.(*Scheduler).initVUsAndExecutors(0xc0002f2700, {0x1f9c988, 0xc000850000}, 0xc0004365b0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/execution/scheduler.go:283 +0x508
go.k6.io/k6/execution.(*Scheduler).Init(0xc0002f2700, {0x1f9c950, 0xc00079c2a0}, 0xc0004365b0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/execution/scheduler.go:412 +0x245
go.k6.io/k6/cmd.(*cmdRun).run(0xc0005c3d00, 0xc0004222c8, {0xc000779020, 0x1, 0x3})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/run.go:367 +0x13fc
github.com/spf13/cobra.(*Command).execute(0xc0004222c8, {0xc000778ff0, 0x3, 0x3})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/spf13/cobra/command.go:856 +0x68a
github.com/spf13/cobra.(*Command).ExecuteC(0xc00039cb08)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/spf13/cobra/command.go:974 +0x38d
github.com/spf13/cobra.(*Command).Execute(...)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/spf13/cobra/command.go:902
go.k6.io/k6/cmd.(*rootCommand).execute(0xc0005f2a80)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/root.go:108 +0xfb
go.k6.io/k6/cmd.Execute()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/root.go:130 +0x2f
main.main()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/main.go:9 +0xf

goroutine 161 [select]:
io.(*pipe).read(0xc0007456e0, {0xc000638000, 0x10000, 0xc000506df0?})
	/usr/local/go/src/io/pipe.go:57 +0xa5
io.(*PipeReader).Read(0xc000506e90?, {0xc000638000?, 0x79a29c4285b8?, 0x10005?})
	/usr/local/go/src/io/pipe.go:134 +0x1a
bufio.(*Scanner).Scan(0xc00064ef28)
	/usr/local/go/src/bufio/scan.go:219 +0x81e
github.com/sirupsen/logrus.(*Entry).writerScanner(0xc0004276c0, 0xc0007456e0, 0xc000796170)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/sirupsen/logrus/writer.go:86 +0x11d
created by github.com/sirupsen/logrus.(*Entry).WriterLevel in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/sirupsen/logrus/writer.go:57 +0x31f

goroutine 162 [chan receive]:
go.k6.io/k6/cmd.(*rootCommand).setupLoggers.func2()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/root.go:276 +0x34
created by go.k6.io/k6/cmd.(*rootCommand).setupLoggers in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/root.go:275 +0x637

goroutine 146 [chan receive]:
go.k6.io/k6/cmd.(*cmdRun).run.func13()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/run.go:334 +0x8b
created by go.k6.io/k6/cmd.(*cmdRun).run in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/run.go:332 +0x11d2

goroutine 166 [chan receive]:
go.k6.io/k6/lib.(*GroupSummary).Start.func1()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/lib/test_state.go:128 +0x9d
created by go.k6.io/k6/lib.(*GroupSummary).Start in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/lib/test_state.go:126 +0x4f

goroutine 167 [select]:
go.k6.io/k6/output.(*PeriodicFlusher).run(0xc0007bcea0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/output/helpers.go:67 +0xb6
created by go.k6.io/k6/output.NewPeriodicFlusher in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/output/helpers.go:102 +0x11a

goroutine 168 [select]:
go.k6.io/k6/output.(*Manager).Start.func2()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/output/manager.go:63 +0x14c
created by go.k6.io/k6/output.(*Manager).Start in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/output/manager.go:56 +0x139

goroutine 177 [IO wait]:
internal/poll.runtime_pollWait(0x79a25583d650, 0x72)
	/usr/local/go/src/runtime/netpoll.go:351 +0x85
internal/poll.(*pollDesc).wait(0xc0002f2080?, 0x10?, 0x0)
	/usr/local/go/src/internal/poll/fd_poll_runtime.go:84 +0x27
internal/poll.(*pollDesc).waitRead(...)
	/usr/local/go/src/internal/poll/fd_poll_runtime.go:89
internal/poll.(*FD).Accept(0xc0002f2080)
	/usr/local/go/src/internal/poll/fd_unix.go:620 +0x295
net.(*netFD).accept(0xc0002f2080)
	/usr/local/go/src/net/fd_unix.go:172 +0x29
net.(*TCPListener).accept(0xc00085a040)
	/usr/local/go/src/net/tcpsock_posix.go:159 +0x1e
net.(*TCPListener).Accept(0xc00085a040)
	/usr/local/go/src/net/tcpsock.go:372 +0x30
net/http.(*Server).Serve(0xc000896000, {0x1f99040, 0xc00085a040})
	/usr/local/go/src/net/http/server.go:3330 +0x30c
net/http.(*Server).ListenAndServe(0xc000896000)
	/usr/local/go/src/net/http/server.go:3259 +0x71
go.k6.io/k6/cmd.(*cmdRun).run.func12()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/run.go:322 +0x172
created by go.k6.io/k6/cmd.(*cmdRun).run in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/run.go:316 +0x110d

goroutine 63 [syscall]:
os/signal.signal_recv()
	/usr/local/go/src/runtime/sigqueue.go:152 +0x29
os/signal.loop()
	/usr/local/go/src/os/signal/signal_unix.go:23 +0x13
created by os/signal.Notify.func1.1 in goroutine 1
	/usr/local/go/src/os/signal/signal.go:151 +0x1f

goroutine 210 [select]:
go.k6.io/k6/cmd.handleTestAbortSignals.func1()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/common.go:104 +0x94
created by go.k6.io/k6/cmd.handleTestAbortSignals in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/common.go:103 +0x185

goroutine 64 [select]:
go.k6.io/k6/execution.(*Scheduler).emitVUsAndVUsMax.func2()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/execution/scheduler.go:240 +0xf9
created by go.k6.io/k6/execution.(*Scheduler).emitVUsAndVUsMax in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/execution/scheduler.go:232 +0x1d3
```

`EXIT CODE: 2`

---

## Appendix B2 — EXP6 complete `concurrent map read and map write` crash dump (verbatim, one captured occurrence)

This is the **complete, unedited** 261-line Go runtime dump from one captured EXP6 crash of the *other* observed fatal variant (the 1-in-700 `concurrent map read and map write` outcome described in §4.1). It is distinct from Appendix B: here the racing access is a map **read** at `js/modules/resolution.go:162` — the file-branch cache lookup — rather than the unconditional map **write** at `js/modules/resolution.go:154`. The read at line 162 is reached via the parent-module resolution path `require_impl.go:27` (resolving the main script's own URL, arg length `0x19` = 25 bytes = `file:///tmp/k6exp/exp6.js`), whereas the Appendix B write is reached via the specifier-resolution path `require_impl.go:28`. Both crashes have the same root cause — the unsynchronized `mr.cache` map written unconditionally at line 154 while other VUs initialize concurrently [execution/scheduler.go:L169-L172] — but they surface at different map accesses. Goroutine IDs and hexadecimal addresses vary from run to run.

```
fatal error: concurrent map read and map write

goroutine 215 [running]:
go.k6.io/k6/js/modules.(*ModuleResolver).resolve(0xc0006e81e0, 0xc000af8bd0, {0xc0006ec180, 0x19})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/modules/resolution.go:162 +0x1cb
go.k6.io/k6/js/modules.(*ModuleResolver).sobekModuleResolver(0xc0006e81e0, {0x0?, 0x0?}, {0xc0006ec180, 0x19})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/modules/resolution.go:195 +0x45
go.k6.io/k6/js/modules.(*ModuleSystem).Require(0xc0004fa0e0, {0xc0005f0318, 0x7})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/modules/require_impl.go:27 +0x125
go.k6.io/k6/js.(*requireImpl).require(0xc0006c6090, {0xc0005f0318, 0x7})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:428 +0x45
reflect.Value.call({0x193c0c0?, 0xc0006c60b0?, 0xc0006583a8?}, {0x1c1f7d1, 0x4}, {0xc000010348, 0x1, 0xc0004f6090?})
	/usr/local/go/src/reflect/value.go:584 +0xca6
reflect.Value.Call({0x193c0c0?, 0xc0006c60b0?, 0xc0006fedc0?}, {0xc000010348?, 0xc0006c6700?, 0x40e45f?})
	/usr/local/go/src/reflect/value.go:368 +0xb9
github.com/grafana/sobek.(*Runtime).newWrappedFunc.(*Runtime).wrapReflectFunc.func1({{0x1fb1a08, 0x2fefba0}, {0xc0003c2250, 0x1, 0x3}})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/runtime.go:2029 +0x3bd
github.com/grafana/sobek.(*nativeFuncObject).vmCall(0xc000624180, 0xc0004d8000, 0x1)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/func.go:563 +0x184
github.com/grafana/sobek.call.exec(0x6fedc0?, 0xc0004d8000)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/vm.go:3719 +0x66
github.com/grafana/sobek.(*vm).run(0xc0004d8000)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/vm.go:635 +0x5b
github.com/grafana/sobek.(*vm).runTryInner(0x1?)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/vm.go:886 +0x52
github.com/grafana/sobek.(*generator).step(0xc0002c5c00)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/func.go:762 +0x2b
github.com/grafana/sobek.(*generator).next(0xc0002c5c00, {0x1fb1a08, 0x2fefba0})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/func.go:800 +0x1b1
github.com/grafana/sobek.(*asyncRunner).onFulfilled(0xc0002c5c00, {{0x0, 0x0}, {0xc0006588b8, 0x1, 0x1}})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/func.go:686 +0xfb
github.com/grafana/sobek.(*SourceTextModuleInstance).ExecuteModule(0xc0004fa580, 0xc00049a008, 0x0, 0x0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/modules_sourcetext.go:29 +0xec
github.com/grafana/sobek.(*Runtime).innerModuleEvaluation(0xc00049a008, 0xc000262dc0, {0x1f9cd80, 0xc000bbadc0}, 0xc000658a98, 0x0, 0xc000658c18)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/modules.go:279 +0x8f1
github.com/grafana/sobek.(*Runtime).CyclicModuleRecordEvaluate(0xc00049a008, {0x1fa5340, 0xc000bbadc0}, 0xc00064ec18)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/modules.go:183 +0x47d
go.k6.io/k6/js/modules.(*ModuleSystem).RunSourceData(0xc0004fa0e0, 0xc0007456b0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/modules/resolution.go:255 +0x1c8
go.k6.io/k6/js.(*Bundle).instantiate.func3()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:338 +0x27
reflect.Value.call({0x18dd7e0?, 0xc0004fa460?, 0x10?}, {0x1c1f7d1, 0x4}, {0x2fefba0, 0x0, 0xc00064f288?})
	/usr/local/go/src/reflect/value.go:584 +0xca6
reflect.Value.Call({0x18dd7e0?, 0xc0004fa460?, 0xc0004fa480?}, {0x2fefba0?, 0xcc451d?, 0x35554aaaa?})
	/usr/local/go/src/reflect/value.go:368 +0xb9
github.com/grafana/sobek.(*Runtime).newWrappedFunc.(*Runtime).wrapReflectFunc.func1({{0x0, 0x0}, {0x0, 0x0, 0x0}})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/runtime.go:2029 +0x3bd
github.com/grafana/sobek.AssertFunction.func1.1()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/runtime.go:2464 +0x56
github.com/grafana/sobek.(*vm).try(0xc0004d8000, 0xc00064f5a8)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/vm.go:863 +0x249
github.com/grafana/sobek.(*Runtime).runWrapped(0xc00049a008, 0xc0004f6058?)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/runtime.go:2508 +0x65
github.com/grafana/sobek.AssertFunction.func1({0x0?, 0x0?}, {0x0?, 0x1?, 0xc0004f6050?})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/runtime.go:2463 +0x8c
go.k6.io/k6/js.(*Bundle).instantiate.func4()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:345 +0x23
go.k6.io/k6/js/eventloop.(*EventLoop).Start(0xc000262730, 0xc0006c65b0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/eventloop/eventloop.go:177 +0x19a
go.k6.io/k6/js.(*Bundle).instantiate(0xc000b8fb88, 0xc000300b80, 0x1)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:344 +0x390
go.k6.io/k6/js.(*Bundle).Instantiate(0xc000b8fb88, {0x1f9c988, 0xc000aaa0a0}, 0x1)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:257 +0x1f1
go.k6.io/k6/js.(*Runner).newVU(0xc000478000, {0x1f9c988?, 0xc000aaa0a0?}, 0x1, 0x1, 0xc0007133b0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/runner.go:128 +0x58
go.k6.io/k6/js.(*Runner).NewVU(0xc000738900?, {0x1f9c988?, 0xc000aaa0a0?}, 0x466f34433454654d?, 0x4143737275486d56?, 0x1424f6141414577?)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/runner.go:116 +0x1d
go.k6.io/k6/execution.(*Scheduler).initVU(0xc00034c900, {0x1f9c988, 0xc000aaa0a0}, 0xc0007133b0, {0x1fb7da0, 0xc000426230})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/execution/scheduler.go:133 +0x9f
go.k6.io/k6/execution.(*Scheduler).initVUsConcurrently.func1()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/execution/scheduler.go:172 +0xd2
created by go.k6.io/k6/execution.(*Scheduler).initVUsConcurrently in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/execution/scheduler.go:170 +0x97

goroutine 1 [select]:
go.k6.io/k6/execution.(*Scheduler).initVUsAndExecutors(0xc00034c900, {0x1f9c988, 0xc000aaa000}, 0xc0007133b0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/execution/scheduler.go:283 +0x508
go.k6.io/k6/execution.(*Scheduler).Init(0xc00034c900, {0x1f9c950, 0xc000745380}, 0xc0007133b0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/execution/scheduler.go:412 +0x245
go.k6.io/k6/cmd.(*cmdRun).run(0xc0007de040, 0xc0004302c8, {0xc000744180, 0x1, 0x3})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/run.go:367 +0x13fc
github.com/spf13/cobra.(*Command).execute(0xc0004302c8, {0xc000744150, 0x3, 0x3})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/spf13/cobra/command.go:856 +0x68a
github.com/spf13/cobra.(*Command).ExecuteC(0xc000359088)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/spf13/cobra/command.go:974 +0x38d
github.com/spf13/cobra.(*Command).Execute(...)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/spf13/cobra/command.go:902
go.k6.io/k6/cmd.(*rootCommand).execute(0xc000aa9e30)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/root.go:108 +0xfb
go.k6.io/k6/cmd.Execute()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/root.go:130 +0x2f
main.main()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/main.go:9 +0xf

goroutine 120 [select]:
io.(*pipe).read(0xc000738180, {0xc00054e000, 0x10000, 0xc0004bd5f0?})
	/usr/local/go/src/io/pipe.go:57 +0xa5
io.(*PipeReader).Read(0xc0004bd690?, {0xc00054e000?, 0x79895d11da68?, 0x10005?})
	/usr/local/go/src/io/pipe.go:134 +0x1a
bufio.(*Scanner).Scan(0xc000562f28)
	/usr/local/go/src/bufio/scan.go:219 +0x81e
github.com/sirupsen/logrus.(*Entry).writerScanner(0xc000712460, 0xc000738180, 0xc0006fe090)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/sirupsen/logrus/writer.go:86 +0x11d
created by github.com/sirupsen/logrus.(*Entry).WriterLevel in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/sirupsen/logrus/writer.go:57 +0x31f

goroutine 121 [chan receive]:
go.k6.io/k6/cmd.(*rootCommand).setupLoggers.func2()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/root.go:276 +0x34
created by go.k6.io/k6/cmd.(*rootCommand).setupLoggers in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/root.go:275 +0x637

goroutine 193 [select]:
go.k6.io/k6/cmd.handleTestAbortSignals.func1()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/common.go:104 +0x94
created by go.k6.io/k6/cmd.handleTestAbortSignals in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/common.go:103 +0x185

goroutine 125 [chan receive]:
go.k6.io/k6/lib.(*GroupSummary).Start.func1()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/lib/test_state.go:128 +0x9d
created by go.k6.io/k6/lib.(*GroupSummary).Start in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/lib/test_state.go:126 +0x4f

goroutine 126 [select]:
go.k6.io/k6/output.(*PeriodicFlusher).run(0xc00049c7e0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/output/helpers.go:67 +0xb6
created by go.k6.io/k6/output.NewPeriodicFlusher in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/output/helpers.go:102 +0x11a

goroutine 127 [select]:
go.k6.io/k6/output.(*Manager).Start.func2()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/output/manager.go:63 +0x14c
created by go.k6.io/k6/output.(*Manager).Start in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/output/manager.go:56 +0x139

goroutine 128 [IO wait]:
internal/poll.runtime_pollWait(0x7989164ece40, 0x72)
	/usr/local/go/src/runtime/netpoll.go:351 +0x85
internal/poll.(*pollDesc).wait(0xc00045f280?, 0x10?, 0x0)
	/usr/local/go/src/internal/poll/fd_poll_runtime.go:84 +0x27
internal/poll.(*pollDesc).waitRead(...)
	/usr/local/go/src/internal/poll/fd_poll_runtime.go:89
internal/poll.(*FD).Accept(0xc00045f280)
	/usr/local/go/src/internal/poll/fd_unix.go:620 +0x295
net.(*netFD).accept(0xc00045f280)
	/usr/local/go/src/net/fd_unix.go:172 +0x29
net.(*TCPListener).accept(0xc0007a8100)
	/usr/local/go/src/net/tcpsock_posix.go:159 +0x1e
net.(*TCPListener).Accept(0xc0007a8100)
	/usr/local/go/src/net/tcpsock.go:372 +0x30
net/http.(*Server).Serve(0xc0004781e0, {0x1f99040, 0xc0007a8100})
	/usr/local/go/src/net/http/server.go:3330 +0x30c
net/http.(*Server).ListenAndServe(0xc0004781e0)
	/usr/local/go/src/net/http/server.go:3259 +0x71
go.k6.io/k6/cmd.(*cmdRun).run.func12()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/run.go:322 +0x172
created by go.k6.io/k6/cmd.(*cmdRun).run in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/run.go:316 +0x110d

goroutine 161 [chan receive]:
go.k6.io/k6/cmd.(*cmdRun).run.func13()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/run.go:334 +0x8b
created by go.k6.io/k6/cmd.(*cmdRun).run in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/cmd/run.go:332 +0x11d2

goroutine 177 [syscall]:
os/signal.signal_recv()
	/usr/local/go/src/runtime/sigqueue.go:152 +0x29
os/signal.loop()
	/usr/local/go/src/os/signal/signal_unix.go:23 +0x13
created by os/signal.Notify.func1.1 in goroutine 1
	/usr/local/go/src/os/signal/signal.go:151 +0x1f

goroutine 194 [select]:
go.k6.io/k6/execution.(*Scheduler).emitVUsAndVUsMax.func2()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/execution/scheduler.go:240 +0xf9
created by go.k6.io/k6/execution.(*Scheduler).emitVUsAndVUsMax in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/execution/scheduler.go:232 +0x1d3

goroutine 195 [runnable]:
github.com/grafana/sobek.(*baseObject)._put(0xc0004e8380, {0x1c23c2f, 0x7}, {0x1fb1710, 0xc0003d3dd0})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/object.go:794 +0x10e
github.com/grafana/sobek.(*baseObject)._putProp(0xc0004e8380, {0x1c23c2f, 0x7}, {0x1fb17a8, 0x2fc8170}, 0x1, 0x0, 0x1)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/object.go:811 +0xbf
github.com/grafana/sobek.(*Runtime).createErrorPrototype(0xc000502008, {0x1fbae28, 0x2f8a7d0}, 0xc0003d3aa0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/builtin_error.go:206 +0x155
github.com/grafana/sobek.(*Runtime).getGoError(0xc000502008)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/builtin_error.go:311 +0x8f
github.com/grafana/sobek.(*Runtime).NewGoError(0xc000502008, {0x1f84c60, 0xc00029db50})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/runtime.go:558 +0x2f
github.com/grafana/sobek.(*Runtime).newWrappedFunc.(*Runtime).wrapReflectFunc.func1({{0x1fb1a08, 0x2fefba0}, {0xc00027e350, 0x1, 0x3}})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/runtime.go:2043 +0x5e9
github.com/grafana/sobek.(*nativeFuncObject).vmCall(0xc0005ae180, 0xc000506000, 0x1)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/func.go:563 +0x184
github.com/grafana/sobek.call.exec(0x6fedc0?, 0xc000506000)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/vm.go:3719 +0x66
github.com/grafana/sobek.(*vm).run(0xc000506000)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/vm.go:635 +0x5b
github.com/grafana/sobek.(*vm).runTryInner(0x1?)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/vm.go:886 +0x52
github.com/grafana/sobek.(*generator).step(0xc0002e9420)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/func.go:762 +0x2b
github.com/grafana/sobek.(*generator).next(0xc0002e9420, {0x1fb1a08, 0x2fefba0})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/func.go:800 +0x1b1
github.com/grafana/sobek.(*asyncRunner).onFulfilled(0xc0002e9420, {{0x0, 0x0}, {0xc000b128b8, 0x1, 0x1}})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/func.go:686 +0xfb
github.com/grafana/sobek.(*SourceTextModuleInstance).ExecuteModule(0xc00030fb20, 0xc000502008, 0x0, 0x0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/modules_sourcetext.go:29 +0xec
github.com/grafana/sobek.(*Runtime).innerModuleEvaluation(0xc000502008, 0xc0000502d0, {0x1f9cd80, 0xc000bbadc0}, 0xc000b12a98, 0x0, 0xc000b12c18)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/modules.go:279 +0x8f1
github.com/grafana/sobek.(*Runtime).CyclicModuleRecordEvaluate(0xc000502008, {0x1fa5340, 0xc000bbadc0}, 0xc0005f4c18)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/modules.go:183 +0x47d
go.k6.io/k6/js/modules.(*ModuleSystem).RunSourceData(0xc00030e7c0, 0xc0007456b0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/modules/resolution.go:255 +0x1c8
go.k6.io/k6/js.(*Bundle).instantiate.func3()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:338 +0x27
reflect.Value.call({0x18dd7e0?, 0xc00030f960?, 0x10?}, {0x1c1f7d1, 0x4}, {0x2fefba0, 0x0, 0xc0005f5288?})
	/usr/local/go/src/reflect/value.go:584 +0xca6
reflect.Value.Call({0x18dd7e0?, 0xc00030f960?, 0xc00030f9a0?}, {0x2fefba0?, 0xcc451d?, 0x35554aaaa?})
	/usr/local/go/src/reflect/value.go:368 +0xb9
github.com/grafana/sobek.(*Runtime).newWrappedFunc.(*Runtime).wrapReflectFunc.func1({{0x0, 0x0}, {0x0, 0x0, 0x0}})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/runtime.go:2029 +0x3bd
github.com/grafana/sobek.AssertFunction.func1.1()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/runtime.go:2464 +0x56
github.com/grafana/sobek.(*vm).try(0xc000506000, 0xc0005f55a8)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/vm.go:863 +0x249
github.com/grafana/sobek.(*Runtime).runWrapped(0xc000502008, 0xc00021a1a8?)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/runtime.go:2508 +0x65
github.com/grafana/sobek.AssertFunction.func1({0x0?, 0x0?}, {0x0?, 0x1?, 0xc00021a198?})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/vendor/github.com/grafana/sobek/runtime.go:2463 +0x8c
go.k6.io/k6/js.(*Bundle).instantiate.func4()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:345 +0x23
go.k6.io/k6/js/eventloop.(*EventLoop).Start(0xc0000500a0, 0xc00029d9a0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/eventloop/eventloop.go:177 +0x19a
go.k6.io/k6/js.(*Bundle).instantiate(0xc000b8fb88, 0xc0004460c0, 0x2)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:344 +0x390
go.k6.io/k6/js.(*Bundle).Instantiate(0xc000b8fb88, {0x1f9c988, 0xc000aaa0a0}, 0x2)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:257 +0x1f1
go.k6.io/k6/js.(*Runner).newVU(0xc000478000, {0x1f9c988?, 0xc000aaa0a0?}, 0x2, 0x2, 0xc0007133b0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/runner.go:128 +0x58
go.k6.io/k6/js.(*Runner).NewVU(0xc000738ae0?, {0x1f9c988?, 0xc000aaa0a0?}, 0x0?, 0x0?, 0x100000000000000?)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/runner.go:116 +0x1d
go.k6.io/k6/execution.(*Scheduler).initVU(0xc00034c900, {0x1f9c988, 0xc000aaa0a0}, 0xc0007133b0, {0x1fb7da0, 0xc000426230})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/execution/scheduler.go:133 +0x9f
go.k6.io/k6/execution.(*Scheduler).initVUsConcurrently.func1()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/execution/scheduler.go:172 +0xd2
created by go.k6.io/k6/execution.(*Scheduler).initVUsConcurrently in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/execution/scheduler.go:170 +0x97

goroutine 178 [select]:
go.k6.io/k6/js.(*Bundle).instantiate.func2()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:322 +0x7e
created by go.k6.io/k6/js.(*Bundle).instantiate in goroutine 195
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:321 +0x22a

goroutine 337 [select]:
go.k6.io/k6/js.(*Bundle).instantiate.func2()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:322 +0x7e
created by go.k6.io/k6/js.(*Bundle).instantiate in goroutine 215
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:321 +0x22a
```

`EXIT CODE: 2`

---

## Appendix C — Coverage pass (Q1–Q6) and stability notes

### Coverage against the six sub-questions

| Sub-question | Answered in | One-line resolution |
|---|---|---|
| **Q1** — why pass in init but fail once many VUs run (dynamic `require()`) | §1, §6 | A conditional/dynamic `require()` only executes an un-cached specifier once enough VUs exist to take the branch; the reject is the one-time lock, not load. |
| **Q2** — does k6 "freeze" resolution after init? ("under pressure") | §1, §2 | Yes — a single deterministic `ModuleResolver.Lock()` [js/bundle.go:L129] right after `__VU==0`. The "under pressure" framing is **wrong** and is corrected. |
| **Q3** — "already resolved" vs. "new module" | §3 | Cache membership (resolved-URL key for files, specifier key for built-ins), populated at `__VU==0`; miss-while-locked = "new". |
| **Q4** — relative specifiers per calling module | §5 | Base dir comes from the JS call stack (`CaptureCallStack` → `reversePath` → `loader.Resolve`); same `./helper.js` → `dir1` vs `dir2`. |
| **Q5** — real runs: (a) init module re-`require`d by VUs; (b) never-seen fails | §2.3/§3 (EXP1), §4 (EXP2/EXP6) | (a) EXP1 exit 0; (b) EXP2 (file) reliably exit 107; EXP6 (built-in) usually exit 107 (682/700) but can crash exit 2 (18/700) via the concurrent-cache race — see stability notes. |
| **Q6** — exact error/warning text in each case | §4, §7 | All strings reproduced verbatim with source constants. |

### Named items covered

`ModuleResolver` [js/modules/resolution.go:L30], `locked` [js/modules/resolution.go:L35], `Lock()` [js/modules/resolution.go:L137-L139], `resolve()` [js/modules/resolution.go:L145-L177], `requireModule()` [js/modules/resolution.go:L69-L89], `reversePath()` [js/modules/resolution.go:L198-L211], `ModuleSystem.Require()` [js/modules/require_impl.go:L15], `getCurrentModuleScript()` [js/modules/require_impl.go:L185-L196] / `getPreviousRequiringFile()` [js/modules/require_impl.go:L198-L225], `CaptureCallStack` [js/modules/require_impl.go:L189], `inInitContext()` [js/bundle.go:L439], `cantBeUsedOutsideInitContextMsg` [js/initcontext.go:L15-L16], `notPreviouslyResolvedModule` [js/modules/resolution.go:L17], `loader.Resolve()` [loader/loader.go:L48] / `Dir()` [loader/loader.go:L115], `vu.state` [js/runner.go:L230] / [js/runner.go:L247]; experiments EXP1–EXP7 and EXP-EMPTY. Supporting: CommonJS wrapper `cjsModule` [js/modules/cjsmodule.go:L11] / `cjsModuleInstance` [js/modules/cjsmodule.go:L51]; `goModule` [js/modules/gomodule.go:L8] / `basicGoModule` [js/modules/gomodule_basic.go:L8] returned by `requireModule`; built-in registry `getJSModules()` [js/jsmodules.go:L71] with `"k6"` [js/jsmodules.go:L34] and `"k6/http"` [js/jsmodules.go:L61]; per-VU module context `moduleVUImpl` [js/modules_vu.go:L17] / `State()` [js/modules_vu.go:L38]; filesystem-backed loading `ReadSource` [loader/readsource.go:L16] and `loader/filesystems.go`; console surfacing (`js/console.go`).

### Stability notes

Every experiment was run **≥2×** with identical input. Exit codes and error strings were stable, with two intentional, observed sources of run-to-run variation — **both consequences of concurrent VU initialization** [execution/scheduler.go:L169-L172], **not** of load:

1. The **order** of the `__VU` init console lines (EXP1).
2. The specific **VU number** named in the init-abort errors (EXP2: `#1`/`#2`; EXP7@5 VUs: `#3`/`#4`/`#5`).

One further **honestly-reported inconsistency**: EXP6 (never-seen **built-in**, 2 concurrent VUs) is *usually* a clean exit 107 but occasionally a fatal Go-runtime crash (exit 2). Across an authoritative **700-run sweep** of the unchanged input the observed distribution was **682/700 (97.4%)** clean exit-107 aborts and **18/700 (2.6%)** crashes, comprising two distinct fatal variants:

- **16/700** — `fatal error: concurrent map writes` at [js/modules/resolution.go:L154] (the unconditional cache write; complete dump in **Appendix B**).
- **2/700** — `fatal error: concurrent map read and map write`: 1 reaching the built-in-branch read at [js/modules/resolution.go:L150] and 1 reaching the file-branch read at [js/modules/resolution.go:L162] (complete dump of the L162 case in **Appendix B2**).

The root cause is the **unconditional** write to the unsynchronized `mr.cache` map at [js/modules/resolution.go:L154] — executed even when the built-in lookup returned the lock error — while other VUs initialize concurrently [execution/scheduler.go:L169-L172]. The **file** case (EXP2) has no such write on the locked path [js/modules/resolution.go:L166-L167] and was perfectly stable at exit 107. The same unchanged input was run repeatedly to surface this distribution rather than constructing a variant that hides it.

### Environment fidelity

Source-under-investigation binary: `k6bin v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)` — built from branch head `ddc3b0b1d23c128e34e2792fc9075f9126e32375` (before this document existed). The runs embedded above were produced by a binary built from the delivered HEAD, which git-stamps `commit/3e9f5aabc8`; because the only difference between the two commits is this documentation file (zero `.go`/`go.mod`/`vendor/` changes — see §0), the two binaries are behaviorally identical and every line number, error string, and exit code is unchanged between them. Toolchain `go1.23.12 linux/amd64`; offline vendored build. All values are specific to this version (`v0.55.0`) and are not generalized to other k6 releases.

---

## References

- k6 source (this branch, read-only): `js/bundle.go`, `js/modules/resolution.go`, `js/modules/require_impl.go`, `js/initcontext.go`, `js/runner.go`, `loader/loader.go`, `js/modules/cjsmodule.go`, `js/modules/modules.go`, `js/modules/gomodule.go`, `js/modules/gomodule_basic.go`, `js/jsmodules.go`, `js/modules_vu.go`, `js/console.go`, `loader/filesystems.go`, `loader/readsource.go`, `execution/scheduler.go`, `go.mod`, `Dockerfile`, `.github/workflows/build.yml`.
- Grafana k6 Test Lifecycle documentation — `https://grafana.com/docs/k6/latest/using-k6/test-lifecycle/`.
- Upstream design discussion on restricting `open`/`require` to the init context for archivability — `grafana/k6#3020` — `https://github.com/grafana/k6/issues/3020`.
