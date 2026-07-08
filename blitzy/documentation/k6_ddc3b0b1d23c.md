# How Grafana k6 Resolves JavaScript Modules Across the `init` and VU Lifecycle — and Why "Dynamic" Loading Fails at Scale

> **Question answered here:** *How does Grafana k6 resolve JavaScript modules across the init and VU (Virtual User) lifecycle stages, and why do some scripts succeed during initialization yet fail once the test runs "under load" with dynamic (runtime) module loading? Does k6 freeze module resolution after init? What counts as "already resolved" vs. a "new module"? What happens with relative specifiers whose resolution depends on the calling module? Show — with real runs — that a module imported during init can still be `require()`'d later by VUs, that a never-seen module reliably fails, and the exact error text in each case.*

This document is an **evidence-grounded investigation**. Every behavioral claim is backed by **both** (1) a specific `file:line` reference into the k6 source tree and (2) the **verbatim output of a real run** of a binary built from this exact branch. The investigation was performed by **building and running first**, then writing from what was observed.

---

## 0. Provenance — the exact binary and how it was built and run

All observations below come from a binary built from this repository at branch `k6_ddc3b0b1d23c`.

- **Repository / branch head commit:** `ddc3b0b1d23c128e34e2792fc9075f9126e32375` (short `ddc3b0b1d`).
- **Go toolchain (observed):** `go version go1.23.12 linux/amd64`.
- **Canonical build command (offline, vendored dependencies), run from the repository root:**

```
PATH=/usr/local/go/bin:$PATH GOTOOLCHAIN=local GOFLAGS=-mod=vendor CGO_ENABLED=0 go build -o /tmp/k6bin .
```

The `-mod=vendor` flag uses the fully vendored dependency tree (no network), `GOTOOLCHAIN=local` pins the installed `go1.23.12` (the repo's `go.mod` declares `go 1.21` [go.mod:L3] / `toolchain go1.21.13` [go.mod:L5], but the canonical build target is Go 1.23.x per `Dockerfile:L1` `FROM --platform=$BUILDPLATFORM golang:1.23-alpine3.20 as builder` and `.github/workflows/build.yml:L27` `DEFAULT_GO_VERSION: "1.23.x"`), and `CGO_ENABLED=0` matches the canonical non-race build.

- **Version string reported by the built binary (verbatim, observed):**

```
k6bin v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)
```

- **Canonical invocation used for every experiment** (stdout/stderr captured together, exit code recorded):

```
/tmp/k6bin run --quiet --no-summary <script>.js
echo "EXIT CODE: $?"
```

For the VU-count demonstration (§6), `--vus N --iterations N` was added.

- **Scratch workspace:** all test scripts were created **outside** the source tree, under `/tmp/k6exp/`, and were removed after the investigation. The k6 source tree was left byte-for-byte unchanged (only this `blitzy/` document is added).

> **Reading the output:** k6 writes console and error logs to **stderr**, wrapped as `time="..." level=... msg="..." source=...`. Inside `msg="..."`, the sequences `\n\t` are literal escapes that render as multi-line stack traces. Outputs below are reproduced exactly as emitted. The `time="..."` timestamp varies from run to run and carries no meaning for this analysis.

> **Version fidelity:** every value here is for the built version **`v0.55.0`, build commit `ddc3b0b1d2`, `go1.23.12`**. Line numbers, error strings, and exit codes are not generalized to other k6 versions.

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

- **Built-in / `k6*` branch** ([js/modules/resolution.go:L147]): the cache is keyed by the **raw specifier** (`"k6"`, `"k6/http"`, ...). A hit returns at [L150-L152] regardless of `locked`. A miss calls `mr.requireModule(arg)` [L153], whose very first act is the lock check:

```
// js/modules/resolution.go:L69-L71
func (mr *ModuleResolver) requireModule(name string) (sobek.ModuleRecord, error) {
	if mr.locked {
		return nil, fmt.Errorf(notPreviouslyResolvedModule, name)
	}
```

- **File branch** (`default`, [js/modules/resolution.go:L156]): the specifier is first turned into an absolute URL by `resolveSpecifier` [L157] (this is where relative resolution happens — see §5). The cache is keyed by that **resolved URL string**; a hit returns at [L162-L164] regardless of `locked`. Only on a **miss** does the lock fire: `if mr.locked { return nil, fmt.Errorf(notPreviouslyResolvedModule, arg) }` [L166-L167]. If not locked, it loads and caches the module.

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

**Observed — the common case (~96% of runs; stderr; exit 107):**

```
time="2026-07-08T04:08:54Z" level=error msg="GoError: the module \"k6/http\" was not previously resolved during initialization (__VU==0)\n\tat go.k6.io/k6/js.(*requireImpl).require-fm (native)\n\tat file:///tmp/k6exp/exp6.js:4:23(18)\n" hint="error while initializing VU #2 (script exception)"
```

`EXIT CODE: 107`

**Honest run-to-run inconsistency (OBSERVED, not inferred):** unlike EXP2 (file module, perfectly stable), EXP6 is **not** perfectly stable. Over **54 identical runs**, **~52 produced the clean exit 107 above and ~2 crashed** with a Go runtime `fatal error: concurrent map writes` and **exit code 2**. (The very first EXP6 invocation of the session crashed; a subsequent sweep produced 39/40 clean + 1 crash, plus one more crash caught at try 56.) I am reporting the observed distribution rather than hiding it behind a controlled single-VU variant.

The crash is real and its cause is visible in the source. In the **built-in branch**, the cache write happens **unconditionally after `requireModule` returns — even when `requireModule` returned the lock error**:

```
// js/modules/resolution.go:L153-L155
		mod, err := mr.requireModule(arg)      // returns the lock error when locked
		mr.cache[arg] = moduleCacheElement{mod: mod, err: err}   // L154: map WRITE happens anyway
		return mod, err
```

When two VUs initialize **concurrently** (§2.4) and both miss the `k6/http` cache, they both reach the map write at [js/modules/resolution.go:L154] at the same time → `fatal error: concurrent map writes`. The **crashing goroutine** captured from a real run makes the causal chain explicit (the map write at `resolution.go:154`, reached from the concurrent VU-init path):

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
...
go.k6.io/k6/js.(*Bundle).Instantiate(0xc000799b88, {0x1f9c988, 0xc000850050}, 0x1)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/bundle.go:257 +0x1f1
go.k6.io/k6/js.(*Runner).newVU(0xc0003f8000, {0x1f9c988?, 0xc000850050?}, 0x1, 0x1, 0xc0004365b0)
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/js/runner.go:128 +0x58
go.k6.io/k6/execution.(*Scheduler).initVU(0xc0002f2700, {0x1f9c988, 0xc000850050}, 0xc0004365b0, {0x1fb7da0, 0xc000832230})
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/execution/scheduler.go:133 +0x9f
go.k6.io/k6/execution.(*Scheduler).initVUsConcurrently.func1()
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/execution/scheduler.go:172 +0xd2
created by go.k6.io/k6/execution.(*Scheduler).initVUsConcurrently in goroutine 1
	/tmp/blitzy/k6/blitzy-05accdba-49bf-4421-bfc5-4f3d6c38fb4b_759c4b/execution/scheduler.go:170 +0x97
```

The complete 179-line dump is reproduced in Appendix EXP6 (the `...` above stands only for well-known intermediate `grafana/sobek` VM frames shown there in full; the remaining goroutines are k6/runtime background goroutines — scheduler, cobra, logrus writer, the REST API `net/http` server, signal handler, output flushers — unrelated to the crash site; goroutine IDs and hex addresses vary per run). The **file** branch does **not** have this race, because on a locked miss it returns at [js/modules/resolution.go:L166-L167] **without** writing the cache — which is exactly why EXP2 (file) is stable while EXP6 (built-in) is occasionally fatal. *(Inference, clearly labeled: the differing stability between EXP2 and EXP6 is attributable to this write-vs-no-write difference; the crash itself, its stack, and its exit code are observed facts.)*

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
- The **rare** built-in concurrent-map-write crash (EXP6) is a Go runtime fatal error → **exit `2`**.


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
| 1 | `the module %q was not previously resolved during initialization (__VU==0)` | `notPreviouslyResolvedModule` [js/modules/resolution.go:L17]; enforced for files at [L166-L167] and for built-ins via `requireModule` [L70-L71] | EXP2 (file), EXP6 (built-in), EXP7 (@5 VUs) | 107 (init abort) |
| 2 | `the "%s" function is only available in the init stage (i.e. the global scope), see https://grafana.com/docs/k6/latest/using-k6/test-lifecycle/ for more information` | `cantBeUsedOutsideInitContextMsg` [js/initcontext.go:L15-L16]; gate at [js/bundle.go:L425-L426] | EXP3 (`require()` in `default()`) | 0 (logged per-iteration) |
| 3 | `require() can't be used with an empty specifier` | `errors.New(...)` [js/modules/require_impl.go:L21] (also [L100]) | EXP-EMPTY | 107 (init abort) |
| 4 | `fatal error: concurrent map writes` | Go runtime, triggered at the map write [js/modules/resolution.go:L154] under concurrent VU init | EXP6 (rare, ~4%) | 2 (Go fatal) |
| 5 | `open() can't be used with files that weren't previously opened during initialization (__VU==0), path: %q` | [js/initcontext.go:L40] — the **parallel `open()` behavior** the freeze deliberately mirrors (see §8). *Cited as the design mirror; not separately run in this investigation.* | — | — (init abort when triggered) |

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

## Appendix C — Coverage pass (Q1–Q6) and stability notes

### Coverage against the six sub-questions

| Sub-question | Answered in | One-line resolution |
|---|---|---|
| **Q1** — why pass in init but fail once many VUs run (dynamic `require()`) | §1, §6 | A conditional/dynamic `require()` only executes an un-cached specifier once enough VUs exist to take the branch; the reject is the one-time lock, not load. |
| **Q2** — does k6 "freeze" resolution after init? ("under pressure") | §1, §2 | Yes — a single deterministic `ModuleResolver.Lock()` [js/bundle.go:L129] right after `__VU==0`. The "under pressure" framing is **wrong** and is corrected. |
| **Q3** — "already resolved" vs. "new module" | §3 | Cache membership (resolved-URL key for files, specifier key for built-ins), populated at `__VU==0`; miss-while-locked = "new". |
| **Q4** — relative specifiers per calling module | §5 | Base dir comes from the JS call stack (`CaptureCallStack` → `reversePath` → `loader.Resolve`); same `./helper.js` → `dir1` vs `dir2`. |
| **Q5** — real runs: (a) init module re-`require`d by VUs; (b) never-seen fails | §2.3/§3 (EXP1), §4 (EXP2/EXP6) | (a) EXP1 exit 0; (b) EXP2/EXP6 exit 107. |
| **Q6** — exact error/warning text in each case | §4, §7 | All strings reproduced verbatim with source constants. |

### Named items covered

`ModuleResolver` [resolution.go:L30], `locked` [L35], `Lock()` [L137-L139], `resolve()` [L145-L177], `requireModule()` [L69-L89], `reversePath()` [L198-L211], `ModuleSystem.Require()` [require_impl.go:L15], `getCurrentModuleScript()` [L185-L196] / `getPreviousRequiringFile()` [L198-L225], `CaptureCallStack` [L189], `inInitContext()` [bundle.go:L439], `cantBeUsedOutsideInitContextMsg` [initcontext.go:L15-L16], `notPreviouslyResolvedModule` [resolution.go:L17], `loader.Resolve()` [loader.go:L48] / `Dir()` [loader.go:L115], `vu.state` [runner.go:L230/L247]; experiments EXP1–EXP7 and EXP-EMPTY. Supporting: CommonJS wrapper `cjsModule` [cjsmodule.go:L11] / `cjsModuleInstance` [cjsmodule.go:L51]; `goModule` [gomodule.go:L8] / `basicGoModule` [gomodule_basic.go:L8] returned by `requireModule`; built-in registry `getJSModules()` [jsmodules.go:L71] with `"k6"` [jsmodules.go:L34] and `"k6/http"` [jsmodules.go:L61]; per-VU module context `moduleVUImpl` [modules_vu.go:L17] / `State()` [modules_vu.go:L38]; filesystem-backed loading `ReadSource` [readsource.go:L16] and `loader/filesystems.go`; console surfacing (`js/console.go`).

### Stability notes

Every experiment was run **≥2×** with identical input. Exit codes and error strings were stable, with two intentional, observed sources of run-to-run variation — **both consequences of concurrent VU initialization** [execution/scheduler.go:L169-L172], **not** of load:

1. The **order** of the `__VU` init console lines (EXP1).
2. The specific **VU number** named in the init-abort errors (EXP2: `#1`/`#2`; EXP7@5 VUs: `#3`/`#4`/`#5`).

One further **honestly-reported inconsistency**: EXP6 (never-seen **built-in**, 2 concurrent VUs) is usually a clean exit 107 (~52/54 runs) but occasionally a `fatal error: concurrent map writes` exit 2 (~2/54), due to the unconditional cache write at [js/modules/resolution.go:L154]. The **file** case (EXP2) has no such write on the locked path [js/modules/resolution.go:L166-L167] and was perfectly stable.

### Environment fidelity

Binary: `k6bin v0.55.0 (commit/ddc3b0b1d2, go1.23.12, linux/amd64)`; branch head `ddc3b0b1d23c128e34e2792fc9075f9126e32375`; toolchain `go1.23.12 linux/amd64`; offline vendored build. All values are specific to this version and are not generalized to other k6 releases.

---

## References

- k6 source (this branch, read-only): `js/bundle.go`, `js/modules/resolution.go`, `js/modules/require_impl.go`, `js/initcontext.go`, `js/runner.go`, `loader/loader.go`, `js/modules/cjsmodule.go`, `js/modules/modules.go`, `js/modules/gomodule.go`, `js/modules/gomodule_basic.go`, `js/jsmodules.go`, `js/modules_vu.go`, `js/console.go`, `loader/filesystems.go`, `loader/readsource.go`, `execution/scheduler.go`, `go.mod`, `Dockerfile`, `.github/workflows/build.yml`.
- Grafana k6 Test Lifecycle documentation — `https://grafana.com/docs/k6/latest/using-k6/test-lifecycle/`.
- Upstream design discussion on restricting `open`/`require` to the init context for archivability — `grafana/k6#3020` — `https://github.com/grafana/k6/issues/3020`.
