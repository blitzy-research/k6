# k6 Module Resolution Lifecycle: Init-to-VU Freezing, Resolution, and Specifier Semantics

> **Comprehensive investigative document answering how k6 JavaScript modules behave when
> transitioning from the init stage to VU execution under load.**

---

## Metadata

| Field | Value |
|-------|-------|
| **Repository** | `go.k6.io/k6` |
| **Version** | v0.55.0 |
| **Commit** | `ddc3b0b1d2` |
| **Go toolchain** | go1.21.13 |
| **Date** | 2026-04-13 |
| **Scope** | Module resolution lifecycle investigation |
| **Binary** | Built from source: `k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)` |

---

## Table of Contents

1. [Module Resolution Freeze Mechanism](#1-module-resolution-freeze-mechanism)
2. [Resolved vs. Rejected Modules](#2-resolved-vs-rejected-modules)
3. [require() Outside Init Context — Two-Level Guard](#3-require-outside-init-context--two-level-guard)
4. [Relative Specifier Semantics](#4-relative-specifier-semantics)
5. [Dynamic import() Behavior](#5-dynamic-import-behavior)
6. [File open() Parallel Pattern](#6-file-open-parallel-pattern)
7. [Live Demonstration Output](#7-live-demonstration-output)
8. [Code Citations Index](#8-code-citations-index)
9. [Test Evidence Summary](#9-test-evidence-summary)
10. [Summary and Key Answers](#10-summary-and-key-answers)

---

## 1. Module Resolution Freeze Mechanism

### Question

**Does k6 intentionally "freeze" module resolution after initialization, and if so, what is the exact mechanism?**

### Answer

**Yes.** k6 intentionally freezes all module resolution after the first initialization pass completes. The mechanism is a single boolean flag — `locked` — on the shared `ModuleResolver` struct, toggled by the `Lock()` method. Once locked, any attempt to resolve a module that is not already present in the resolver's internal cache is rejected with a specific error.

### Rationale and Thinking

The design intent behind freezing is **deterministic, reproducible test execution**. If modules could be loaded on-the-fly during VU execution, different VUs could see different module states (e.g., if a module file changes on disk during a test, or if network-loaded modules vary between requests). By caching all modules during a single init pass and then locking the resolver, k6 guarantees that every VU operates with an identical, immutable set of modules.

### Source Code Evidence

#### 1.1 The `ModuleResolver` Struct

The central data structure lives in `js/modules/resolution.go` lines 29–40:

```go
// ModuleResolver knows how to get base Module that can be initialized
type ModuleResolver struct {
    cache     map[string]moduleCacheElement   // line 31
    goModules map[string]any                  // line 32
    loadCJS   FileLoader                      // line 33
    compiler  *compiler.Compiler              // line 34
    locked    bool                            // line 35 — THE FREEZE GATE
    reverse   map[any]*url.URL                // line 36
    base      *url.URL                        // line 37
    usage     *usage.Usage                    // line 38
    logger    logrus.FieldLogger              // line 39
}
```

The three critical fields for the freeze mechanism are:

- **`cache`** (`map[string]moduleCacheElement`): Stores every module resolved during initialization, keyed by its resolved URL string. This is the "known modules" registry.
- **`locked`** (`bool`): The freeze gate. When `false`, new modules can be loaded from the filesystem. When `true`, only cached modules are served.
- **`reverse`** (`map[any]*url.URL`): Maps module records back to their source URLs, used for relative path resolution.

#### 1.2 The `Lock()` Method

Defined in `js/modules/resolution.go` lines 133–139:

```go
// Lock locks the module's resolution from any further new resolving operation.
// It means that it relays only its internal cache and on the fact that it has already
// seen previously the module during the initialization.
// It is the same approach used for opening file operations.
func (mr *ModuleResolver) Lock() {
    mr.locked = true
}
```

Key observations from the comment:
- The word "locks" is used explicitly — this is an intentional freeze, not a side effect.
- "It relays only its internal cache" — after locking, the resolver only serves from cache.
- "It is the same approach used for opening file operations" — the `open()` function follows the identical pattern (see [Section 6](#6-file-open-parallel-pattern)).

#### 1.3 Where `Lock()` Is Called

In `js/bundle.go` lines 110–129 (the `newBundle()` function):

```go
bundle.ModuleResolver = modules.NewModuleResolver(
    getJSModules(), generateFileLoad(bundle), c, bundle.pwd, piState.Usage, piState.Logger)

// ... setup vuImpl ...

vuImpl.eventLoop = eventloop.New(vuImpl)
bi, err := bundle.instantiate(vuImpl, 0)    // line 125: __VU==0 init pass
if err != nil {
    return nil, err
}
bundle.ModuleResolver.Lock()                // line 129: FREEZE
```

The sequence is:
1. **Line 110–111**: A new `ModuleResolver` is created with an empty cache.
2. **Line 125**: `bundle.instantiate(vuImpl, 0)` executes the user's script with `__VU == 0`. During this execution, every `import` statement and `require()` call resolves modules through the (unlocked) resolver, populating the cache.
3. **Line 129**: `Lock()` is called immediately after init completes. From this point forward, the resolver is frozen.

#### 1.4 How the Lock Is Enforced in `resolve()`

The `resolve()` function in `js/modules/resolution.go` lines 145–177 implements the cache-then-lock pattern:

```go
func (mr *ModuleResolver) resolve(basePWD *url.URL, arg string) (sobek.ModuleRecord, error) {
    switch {
    case arg == "k6", strings.HasPrefix(arg, "k6/"):
        // Built-in modules
        if cached, ok := mr.cache[arg]; ok {          // line 150: cache check
            return cached.mod, cached.err              // line 151: cache HIT → return
        }
        mod, err := mr.requireModule(arg)              // line 153: cache MISS → load
        mr.cache[arg] = moduleCacheElement{mod: mod, err: err}
        return mod, err
    default:
        // Filesystem modules
        specifier, err := mr.resolveSpecifier(basePWD, arg)
        if err != nil {
            return nil, err
        }
        if cached, ok := mr.cache[specifier.String()]; ok {  // line 162: cache check
            return cached.mod, cached.err                     // line 163: cache HIT → return
        }

        if mr.locked {                                        // line 166: LOCK CHECK
            return nil, fmt.Errorf(notPreviouslyResolvedModule, arg)  // line 167: REJECTED
        }
        // Fall back to loading from filesystem                // line 170+
        data, err := mr.loadCJS(specifier, arg)
        // ...
    }
}
```

The flow for filesystem modules is:
1. Resolve the specifier to a URL.
2. Check the cache. If found → return immediately (regardless of lock state).
3. If NOT found AND `mr.locked` is `true` → return the `notPreviouslyResolvedModule` error.
4. If NOT found AND NOT locked → load from filesystem, parse, compile, and cache.

#### 1.5 The Lock Also Applies to Built-in Go Modules

The `requireModule()` function in `js/modules/resolution.go` lines 69–72:

```go
func (mr *ModuleResolver) requireModule(name string) (sobek.ModuleRecord, error) {
    if mr.locked {
        return nil, fmt.Errorf(notPreviouslyResolvedModule, name)
    }
    // ...
}
```

This means even Go-backed `k6/*` modules are subject to the same lock. If a `k6/*` module was never imported during `__VU==0`, it cannot be resolved after the lock.

#### 1.6 The Error Constant

In `js/modules/resolution.go` line 17:

```go
const notPreviouslyResolvedModule = "the module %q was not previously resolved during initialization (__VU==0)"
```

This error message is the definitive signal that the lock mechanism has rejected a module resolution request.

### Lifecycle Flow Diagram

```
newBundle()
  │
  ├─ ModuleResolver created (cache empty, locked=false)
  │
  ├─ bundle.instantiate(vuImpl, 0)     ← __VU==0 init pass
  │    │
  │    ├─ Script executes top-level code
  │    ├─ Every import/require() resolves through resolve()
  │    ├─ Cache is populated with all reachable modules
  │    └─ Init pass completes
  │
  ├─ bundle.ModuleResolver.Lock()      ← locked=true
  │
  └─ All subsequent VU instantiations use the SAME locked resolver
       │
       ├─ Cache HIT → module served ✓
       └─ Cache MISS → notPreviouslyResolvedModule error ✗
```

---

## 2. Resolved vs. Rejected Modules

### Question

**What precisely constitutes an "already resolved" module (eligible for cache hits after lock) versus a "new" module that will be rejected?**

### Answer

- **"Already resolved"** = present in the `ModuleResolver.cache` map (a `map[string]moduleCacheElement`), keyed by the resolved URL string for filesystem modules or the raw module name for built-ins.
- **"New/rejected"** = absent from the cache when `locked == true`.

### Rationale and Thinking

The distinction is purely a matter of cache membership. During the `__VU==0` init pass, every module specifier that is resolved — whether via top-level `import`, `require()`, or transitive dependency — gets stored in the cache. After `Lock()`, the cache becomes the exhaustive registry of known modules. Any specifier not in this registry is "new" and is rejected.

An important subtlety is that **even errors are cached**. If a module failed to resolve during init (e.g., a syntax error), the error is stored in the cache. Subsequent resolution of that same specifier will return the cached error rather than re-attempting the load.

### Source Code Evidence

#### 2.1 The Cache Entry Struct

In `js/modules/resolution.go` lines 24–27:

```go
type moduleCacheElement struct {
    mod sobek.ModuleRecord
    err error
}
```

Both the successfully-parsed module record (`mod`) AND any error (`err`) are stored. A module that failed to parse during init is cached with `err != nil`, so subsequent VUs will see the same error without re-attempting the load.

#### 2.2 Cache Key Semantics

For **built-in modules** (`k6/*`), the cache key is the raw module name string (e.g., `"k6/http"`, `"k6/crypto"`). This is visible at line 150:
```go
if cached, ok := mr.cache[arg]; ok {
```

For **filesystem modules**, the cache key is the fully resolved URL string (e.g., `"file:///path/to/module.js"`). This is visible at line 162:
```go
if cached, ok := mr.cache[specifier.String()]; ok {
```

#### 2.3 How the Cache Is Populated

The `resolveLoaded()` function in `js/modules/resolution.go` lines 98–131 handles cache population for filesystem modules:

```go
func (mr *ModuleResolver) resolveLoaded(basePWD *url.URL, arg string, data []byte) (sobek.ModuleRecord, error) {
    specifier, err := mr.resolveSpecifier(basePWD, arg)
    // ...
    if cached, ok := mr.cache[specifier.String()]; ok {   // line 104: check existing
        return cached.mod, cached.err
    }
    // ... parse and compile ...
    mr.reverse[mod] = specifier                            // line 128: reverse map
    mr.cache[specifier.String()] = moduleCacheElement{     // line 129: CACHE POPULATION
        mod: mod, err: err,
    }
    return mod, err
}
```

Key points:
- **Line 104**: Before parsing, it checks if the module is already cached (deduplication).
- **Line 128**: The module record is stored in the `reverse` map for relative path resolution.
- **Line 129**: The module (or error) is stored in the cache under its resolved URL.

For built-in modules, cache population happens in `resolve()` at line 154:
```go
mod, err := mr.requireModule(arg)
mr.cache[arg] = moduleCacheElement{mod: mod, err: err}
```

#### 2.4 What Triggers Resolution During Init

During the `__VU==0` init pass, modules are resolved through several entry points:

1. **Top-level `import` statements** (ESM): The Sobek engine calls `sobekModuleResolver()` for each `import` declaration, which delegates to `resolve()`.
2. **`require()` calls** (CommonJS): The `ModuleSystem.Require()` method in `require_impl.go` calls `sobekModuleResolver()`, which delegates to `resolve()`.
3. **Transitive dependencies**: When a loaded module itself contains `import` or `require()`, those are resolved recursively through the same `resolve()` path.

All three paths populate the same `cache` map. After `Lock()`, all three paths check the same `cache` map before the lock guard.

#### 2.5 The Rejection Path

When `resolve()` reaches line 166 (filesystem modules) or `requireModule()` reaches line 70 (built-in modules) with `mr.locked == true` after a cache miss:

```go
return nil, fmt.Errorf(notPreviouslyResolvedModule, arg)
```

This produces the error:
```
the module "<specifier>" was not previously resolved during initialization (__VU==0)
```

### Concrete Examples

| Scenario | Cache State | Lock State | Result |
|----------|------------|------------|--------|
| Top-level `import "./foo.js"` during `__VU==0` | Not in cache → loaded, parsed, cached | `locked=false` | ✅ Resolved and cached |
| VU init re-imports `"./foo.js"` | In cache | `locked=true` | ✅ Cache hit |
| VU init tries `require("./bar.js")` (never seen) | Not in cache | `locked=true` | ❌ `notPreviouslyResolvedModule` |
| VU init imports `"k6/http"` (used during `__VU==0`) | In cache | `locked=true` | ✅ Cache hit |
| VU init imports `"k6/ws"` (never used during `__VU==0`) | Not in cache | `locked=true` | ❌ `notPreviouslyResolvedModule` |

---

## 3. require() Outside Init Context — Two-Level Guard

### Question

**What prevents `require()` from working during VU execution (the `default` function, `setup`, `teardown`)?**

### Answer

`require()` is guarded at two independent levels, forming a defense-in-depth pattern:

- **Level 1 — Init Context Check**: The `requireImpl.require()` wrapper in `js/bundle.go` checks whether the VU is still in init context (`vu.state == nil`). If not, it immediately returns the error `the "require" function is only available in the init stage`.
- **Level 2 — Resolver Lock Check**: Even if `require()` bypassed Level 1, the `ModuleResolver.resolve()` function rejects any uncached specifier when `locked == true`.

### Rationale and Thinking

These two levels serve different purposes:

- **Level 1** provides a clean, user-facing error message for the most common mistake (calling `require()` inside the `default` function). It fires before the module resolver is even consulted.
- **Level 2** provides an architectural safeguard that prevents any code path — not just `require()` — from loading new modules after init. This protects against future code changes that might accidentally bypass the init context check.

The two guards operate at different points in the VU lifecycle:
- Level 1 activates when `vu.moduleVUImpl.state` transitions from `nil` to a non-nil `*lib.State`.
- Level 2 activates when `ModuleResolver.Lock()` is called (earlier than Level 1).

### Source Code Evidence

#### 3.1 Level 1 — The `requireImpl` Guard

In `js/bundle.go` lines 418–429:

```go
// this exists only to make the check in the init context.
type requireImpl struct {
    inInitContext func() bool          // line 420
    modSys        *modules.ModuleSystem // line 421
}

func (r *requireImpl) require(specifier string) (*sobek.Object, error) {
    if !r.inInitContext() {            // line 425: CHECK
        return nil, fmt.Errorf(cantBeUsedOutsideInitContextMsg, "require")  // line 426: REJECT
    }
    return r.modSys.Require(specifier) // line 428: PROCEED
}
```

The `inInitContext` function is wired up in `setInitGlobals()` at line 438–439:

```go
impl := requireImpl{
    inInitContext: func() bool { return vu.state == nil },
    modSys:        modSys,
}
```

So `inInitContext()` returns `true` when `vu.state == nil` (init context) and `false` when `vu.state` is set (VU execution context).

#### 3.2 The Error Message

In `js/initcontext.go` lines 15–16:

```go
const cantBeUsedOutsideInitContextMsg = `the "%s" function is only available in the init stage ` +
    `(i.e. the global scope), see https://grafana.com/docs/k6/latest/using-k6/test-lifecycle/ for more information`
```

When formatted with `"require"`, this produces:
```
the "require" function is only available in the init stage (i.e. the global scope), see https://grafana.com/docs/k6/latest/using-k6/test-lifecycle/ for more information
```

#### 3.3 The State Transition

In `js/runner.go` line 247:

```go
vu.moduleVUImpl.state = vu.state
```

This is inside `newVU()`, called when a new VU is created. Before this line, `vu.moduleVUImpl.state` is `nil` (init context). After this line, it points to a valid `*lib.State`, and `inInitContext()` returns `false`.

The `moduleVUImpl` struct is defined in `js/modules_vu.go` lines 17–24:

```go
type moduleVUImpl struct {
    ctx       context.Context
    initEnv   *common.InitEnvironment
    state     *lib.State              // line 20: nil during init, set during VU activation
    runtime   *sobek.Runtime
    eventLoop *eventloop.EventLoop
    events    events
}
```

#### 3.4 Timeline of Guard Activation

```
newBundle()
  ├─ instantiate(vuImpl, 0)    ← vu.state == nil, Level 1 ALLOWS require()
  ├─ Lock()                    ← Level 2 REJECTS new modules (but cached ones still work)
  │
newVU()  [for each VU]
  ├─ instantiate(vuImpl, N)    ← vu.state == nil, Level 1 ALLOWS require()
  │                               Level 2 serves from cache
  ├─ vu.state = lib.State{...} ← Level 1 NOW BLOCKS require()
  │
VU Execution (default/setup/teardown)
  └─ require() → Level 1 BLOCKS with cantBeUsedOutsideInitContextMsg
```

Note the subtle window: between `newVU()` starting and the state assignment at line 247, the VU is in init context. During this window, `require()` passes Level 1 but is constrained by Level 2 (resolver lock). This is why both levels are necessary.

---

## 4. Relative Specifier Semantics

### Question

**How do relative import paths (`./foo.js`, `../bar.js`) resolve when the call stack originates from different modules, and does this become ambiguous under multi-VU execution?**

### Answer

Relative paths **always resolve against the file that textually contains the `import` or `require()` statement**, not against the runtime call stack. This is deterministic and unambiguous regardless of VU count, call-chain depth, or execution order.

### Rationale and Thinking

This is a critical correctness property. If relative paths resolved against the runtime caller (like `__dirname` in Node.js CJS modules), then the same `import "./data.js"` statement could resolve to different files depending on who called the importing module. k6 avoids this ambiguity by using the Sobek engine's module record system, which tracks which source file contains each import statement at compile time, not at runtime.

Under multi-VU execution, each VU re-executes the same source code in its own Sobek runtime. The source code is immutable (compiled once during `__VU==0`), so the same import statements always reference the same module records, producing identical resolution in every VU.

### Source Code Evidence

#### 4.1 The Sobek Module Resolver Callback (ESM Path)

In `js/modules/resolution.go` lines 192–196:

```go
func (mr *ModuleResolver) sobekModuleResolver(
    referencingScriptOrModule any, specifier string,
) (sobek.ModuleRecord, error) {
    return mr.resolve(mr.reversePath(referencingScriptOrModule), specifier)
}
```

The Sobek JS engine calls this function for every ESM `import` statement. The first argument, `referencingScriptOrModule`, is the **module record that textually contains the import statement**. This is provided by the Sobek engine based on the compiled module graph, not the runtime call stack.

#### 4.2 The `reversePath()` Method

In `js/modules/resolution.go` lines 198–211:

```go
func (mr *ModuleResolver) reversePath(referencingScriptOrModule interface{}) *url.URL {
    p, ok := mr.reverse[referencingScriptOrModule]
    if !ok {
        if referencingScriptOrModule != nil {
            panic("fix this")
        }
        return mr.base                            // nil referencing → project root
    }

    if p.String() == "file:///-" {
        return mr.base                            // entry script → project root
    }
    return p.JoinPath("..")                       // non-entry → parent directory
}
```

This method:
1. Looks up the referencing module in the `reverse` map (populated during `resolveLoaded()` at line 128).
2. Returns the **parent directory** of the module's URL via `p.JoinPath("..")`.
3. For the entry script (`file:///-`) or nil references, falls back to `mr.base` (the project root).

The returned URL becomes the `pwd` (parent working directory) passed to `resolve()`, which then joins it with the relative specifier.

#### 4.3 The Specifier Classification in `loader.Resolve()`

In `loader/loader.go` lines 48–55:

```go
func Resolve(pwd *url.URL, moduleSpecifier string) (*url.URL, error) {
    if moduleSpecifier == "" {
        return nil, errors.New("local or remote path required")
    }

    if moduleSpecifier[0] == '.' || moduleSpecifier[0] == '/' || filepath.IsAbs(moduleSpecifier) {
        return resolveFilePath(pwd, moduleSpecifier)
    }
    // ...
}
```

Specifiers starting with `.` (like `./foo.js` or `../bar.js`) or `/` are classified as file paths and resolved by `resolveFilePath()`.

#### 4.4 The `resolveFilePath()` Function

In `loader/loader.go` lines 84–112:

```go
func resolveFilePath(pwd *url.URL, moduleSpecifier string) (*url.URL, error) {
    // ... handle opaque URLs and volume names ...

    finalPwd := pwd
    // Ensure pwd ends in a slash
    if !strings.HasSuffix(pwd.Path, "/") {
        finalPwd = &url.URL{}
        *finalPwd = *pwd
        finalPwd.Path += "/"
    }
    return finalPwd.Parse(moduleSpecifier)    // line 111: standard URL resolution
}
```

The `pwd` (from `reversePath()`) is joined with the `moduleSpecifier` using standard URL resolution rules. This is deterministic — given the same `pwd` and `moduleSpecifier`, the result is always the same.

#### 4.5 The CJS `require()` Path — Stack Frame Inspection

For CommonJS `require()`, the parent module is determined differently. In `js/modules/require_impl.go` lines 185–196:

```go
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

For CJS `require()`, the parent module is determined by **stack frame inspection** (`CaptureCallStack(2, ...)`). `frames[1].SrcName()` returns the source URL of the calling module. While this uses the call stack, the source name (`SrcName()`) is the **source file URL** (fixed at compile time), not a dynamic runtime concept. The same `require()` call in the same source file always reports the same `SrcName()`.

#### 4.6 Multi-VU Determinism

Under multi-VU execution:
1. Each VU gets its own Sobek runtime, but the source code (compiled programs) is shared.
2. The same `import` statements reference the same module records.
3. The `reverse` map is shared across all VUs (populated once during init, never modified after `Lock()`).
4. Therefore, `reversePath()` returns the same URL for the same module record in every VU.
5. `resolveFilePath()` is a pure function — same inputs, same output.

**Result**: Relative path resolution is fully deterministic and identical across all VUs.

---

## 5. Dynamic import() Behavior

### Question

**How does dynamic `import()` (the `import()` expression, not the `import` declaration) behave in k6?**

### Answer

Dynamic `import()` is **globally disabled** in k6. It fails at the Sobek JS engine level with the error `"dynamic modules not enabled in the host program"`, regardless of whether the target module is cached or not.

### Rationale and Thinking

k6 uses Sobek (a fork of goja) as its JavaScript engine. Sobek requires explicit opt-in to enable dynamic `import()` — the host program must configure a dynamic module resolution callback. k6 does not configure this callback, so all dynamic `import()` calls are rejected by the engine before they ever reach the k6 module resolver.

This is an important distinction from the module resolver lock:
- The **resolver lock** is a k6-level mechanism that rejects *new* modules but allows *cached* modules.
- The **dynamic import rejection** is a Sobek engine-level mechanism that rejects *all* dynamic imports unconditionally.

Even if a module was successfully resolved during init and exists in the cache, a dynamic `import()` for that module will still fail with the Sobek error. The rejection happens before the k6 resolver is consulted.

### Source Code Evidence

The Sobek engine does not expose a public configuration for dynamic module resolution in the k6 codebase. The `setInitGlobals()` function in `js/bundle.go` (lines 431–504) configures:
- `require` (CJS module loading)
- `import.meta.resolve` (specifier resolution)
- Various globals (`__VU`, `console`, etc.)

But it does **not** configure a dynamic import resolver. The Sobek engine's default behavior when no dynamic import handler is configured is to reject all dynamic `import()` calls with the string error `"dynamic modules not enabled in the host program"`.

Note that this rejection produces a **string** (not an Error object), which is why catching it in JavaScript shows `typeof e === "string"`.

---

## 6. File open() Parallel Pattern

### Question

**Does the `open()` function follow the same freeze pattern as module resolution?**

### Answer

**Yes.** The `open()` function follows an identical freeze pattern. Files opened during `__VU==0` init are cached in the filesystem layer, and after init completes, the filesystem is switched to "only cached" mode. Any attempt to open a file not previously opened during init is rejected.

### Source Code Evidence

#### 6.1 The `allowOnlyOpenedFiles()` Function

In `js/initcontext.go` lines 56–64:

```go
// allowOnlyOpenedFiles enables seen only files
func allowOnlyOpenedFiles(fs fsext.Fs) {
    alreadyOpenedFS, ok := fs.(fsext.OnlyCachedEnabler)
    if !ok {
        return
    }

    alreadyOpenedFS.AllowOnlyCached()
}
```

After init, `AllowOnlyCached()` switches the filesystem to reject any file that was not previously read. This is exactly analogous to `ModuleResolver.Lock()`.

#### 6.2 The Error Message for open()

In `js/initcontext.go` lines 34–44, the `readFile()` function wraps filesystem errors:

```go
func readFile(fileSystem fsext.Fs, filename string) (data []byte, err error) {
    defer func() {
        if errors.Is(err, fsext.ErrPathNeverRequestedBefore) {
            err = fmt.Errorf(
                "open() can't be used with files that weren't previously opened during initialization (__VU==0), path: %q",
                filename,
            )
        }
    }()
    // ...
}
```

The error message mirrors the module resolver's `notPreviouslyResolvedModule` — both reference `__VU==0` and the init stage.

#### 6.3 The Lock() Comment Confirms the Parallel

The `Lock()` comment in `js/modules/resolution.go` line 136 explicitly states:
```
// It is the same approach used for opening file operations.
```

This confirms the design team intentionally implemented both systems with the same pattern.

---

## 7. Live Demonstration Output

All experiments were conducted using a k6 binary built from the repository at commit `ddc3b0b1d2` (v0.55.0) with Go 1.21.13. All temporary scripts were created in `/tmp/k6_experiments/` and deleted after execution.

Binary identification:
```
k6 v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)
```

---

### Experiment 1: ESM Import at Init — Cache Serves All VUs

**Setup**: Two files — `helper.js` exports a `greet()` function; `exp1.js` imports it at init and calls it in the default function.

```javascript
// helper.js
export function greet(name) {
  return "Hello, " + name;
}

// exp1.js
import { greet } from "./helper.js";

export default function() {
  console.log(greet("VU " + __VU));
}
```

**Expected behavior**: `helper.js` is resolved during `__VU==0` and cached. All 3 VUs receive the cached module and execute successfully.

**k6 command**: `k6 run --vus 3 --iterations 3 exp1.js`

**Output**:
```
     scenarios: (100.00%) 1 scenario, 3 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 3 iterations shared among 3 VUs (maxDuration: 10m0s, gracefulStop: 30s)

time="2026-04-13T21:15:44Z" level=info msg="Hello, VU 2" source=console
time="2026-04-13T21:15:44Z" level=info msg="Hello, VU 3" source=console
time="2026-04-13T21:15:44Z" level=info msg="Hello, VU 1" source=console

     iterations...........: 3   9991.008093/s

running (00m00.0s), 0/3 VUs, 3 complete and 0 interrupted iterations
default ✓ [ 100% ] 3 VUs  00m00.0s/10m0s  3/3 shared iters
```

**Analysis**: All three VUs successfully imported and used the `greet()` function from the cached `helper.js` module. The module was resolved once during `__VU==0` and served from cache to VUs 1, 2, and 3. Zero errors, 100% success rate.

---

### Experiment 2: require() Inside default Function (Outside Init)

**Setup**: Script attempts to `require("k6/crypto")` inside the `default` export function.

```javascript
// exp2.js
export default function() {
  let crypto = require("k6/crypto");
  console.log(crypto);
}
```

**Expected behavior**: Level 1 guard (init context check) blocks `require()` with `cantBeUsedOutsideInitContextMsg`.

**k6 command**: `k6 run --vus 1 --iterations 1 exp2.js`

**Output**:
```
time="2026-04-13T21:15:48Z" level=error msg="GoError: the \"require\" function is only available
in the init stage (i.e. the global scope), see
https://grafana.com/docs/k6/latest/using-k6/test-lifecycle/ for more information\n\tat
go.k6.io/k6/js.(*requireImpl).require-fm (native)\n\tat default
(file:///tmp/k6_experiments/exp2.js:2:23(3))\n" executor=shared-iterations scenario=default
source=stacktrace

     iterations...........: 1   3814.973772/s

running (00m00.0s), 0/1 VUs, 1 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.0s/10m0s  1/1 shared iters
```

**Analysis**: The Level 1 guard fired. The error message exactly matches `cantBeUsedOutsideInitContextMsg` from `js/initcontext.go:15-16`, formatted with `"require"`. The stack trace shows the rejection originates from `go.k6.io/k6/js.(*requireImpl).require-fm`, confirming it is the `requireImpl.require()` method that blocked the call.

---

### Experiment 3: Conditional require() for Never-Seen Module

**Setup**: `never_seen.js` exists on disk but is only `require()`-d when `__VU > 0`, meaning it is never resolved during `__VU==0`.

```javascript
// never_seen.js
exports.value = 99;

// exp3.js
if (__VU > 0) {
  let data = require("./never_seen.js");
}

export default function() {
  console.log("iteration done");
}
```

**Expected behavior**: During `__VU==0`, the `require()` is skipped (condition is false). `never_seen.js` is never loaded into the cache. After `Lock()`, when VU 1 executes and hits `__VU > 0`, the `require()` tries to resolve `./never_seen.js` but finds a cache miss + locked resolver → `notPreviouslyResolvedModule` error.

**k6 command**: `k6 run --vus 1 --iterations 1 exp3.js`

**Output**:
```
     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 1 iterations shared among 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)

Init      [   0% ] 0/1 VUs initialized

time="2026-04-13T21:15:53Z" level=error msg="GoError: the module \"./never_seen.js\" was not
previously resolved during initialization (__VU==0)\n\tat
go.k6.io/k6/js.(*requireImpl).require-fm (native)\n\tat
file:///tmp/k6_experiments/exp3.js:2:21(17)\n" hint="error while initializing VU #1
(script exception)"
```

**Analysis**: The error message exactly matches the `notPreviouslyResolvedModule` constant from `js/modules/resolution.go:17`, formatted with `"./never_seen.js"`. The VU initialization itself failed (`hint="error while initializing VU #1"`), not the default function execution. This confirms:
1. The module was not in the cache (because it was never resolved during `__VU==0`).
2. The resolver was locked (because `Lock()` was called after init).
3. The error fires during VU init (before the `default` function runs).

---

### Experiment 4: Module Resolved at `__VU==0`, Re-required by VU Init

**Setup**: `cached_module.js` is `require()`-d during `__VU==0` (caching it), then re-required by VU init.

```javascript
// cached_module.js
exports.value = 42;

// exp4.js
if (__VU == 0) {
  require("./cached_module.js");
}

if (__VU > 0) {
  let data = require("./cached_module.js");
  console.log("VU " + __VU + " got value: " + data.value);
}

export default function() {
  // VU execution
}
```

**Expected behavior**: During `__VU==0`, `cached_module.js` is resolved and cached. During VU init (`__VU > 0`), the same specifier resolves to a cache hit → success.

**k6 command**: `k6 run --vus 2 --iterations 2 exp4.js`

**Output**:
```
     scenarios: (100.00%) 1 scenario, 2 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 2 iterations shared among 2 VUs (maxDuration: 10m0s, gracefulStop: 30s)

time="2026-04-13T21:15:58Z" level=info msg="VU 2 got value: 42" source=console
time="2026-04-13T21:15:58Z" level=info msg="VU 1 got value: 42" source=console

     iterations...........: 2   14610.80469/s

running (00m00.0s), 0/2 VUs, 2 complete and 0 interrupted iterations
default ✓ [ 100% ] 2 VUs  00m00.0s/10m0s  2/2 shared iters
```

**Analysis**: Both VUs successfully required `cached_module.js` and read its `value` property (42). This proves:
1. The module was cached during `__VU==0`.
2. The cache served the module to both VUs despite the lock being active.
3. The module data is preserved correctly across VU boundaries.

---

### Experiment 5: Relative Specifiers Across Nested Modules

**Setup**: `lib/wrapper.js` requires `./data.js` (relative to itself). The main script requires `./lib/wrapper.js`.

```javascript
// lib/data.js
exports.info = "from-lib-data";

// lib/wrapper.js
let d = require("./data.js");
exports.getData = function() { return d.info; };

// exp5.js
let wrapper = require("./lib/wrapper.js");

export default function() {
  console.log("VU " + __VU + " got: " + wrapper.getData());
}
```

**Expected behavior**: When `lib/wrapper.js` executes `require("./data.js")`, the `./` resolves relative to `lib/wrapper.js` (its parent directory `lib/`), NOT relative to the main script's directory. Therefore it correctly finds `lib/data.js`.

**k6 command**: `k6 run --vus 1 --iterations 1 exp5.js`

**Output**:
```
     scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
              * default: 1 iterations shared among 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)

time="2026-04-13T21:16:03Z" level=info msg="VU 1 got: from-lib-data" source=console

     iterations...........: 1   5013.184676/s

running (00m00.0s), 0/1 VUs, 1 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.0s/10m0s  1/1 shared iters
```

**Analysis**: The output `"from-lib-data"` proves that `./data.js` in `lib/wrapper.js` correctly resolved to `lib/data.js` (relative to the importing file), not to a hypothetical `./data.js` in the project root. This confirms the `reversePath()` mechanism: the Sobek engine passed `lib/wrapper.js`'s module record as `referencingScriptOrModule`, `reversePath()` returned its parent directory (`lib/`), and `resolveFilePath()` joined `lib/` with `./data.js` to produce `lib/data.js`.

---

### Experiment 6: Dynamic import() of a Never-Seen Module

**Setup**: Script uses dynamic `import()` to load a module that was never resolved during init.

```javascript
// exp6.js
export default async function() {
  try {
    let mod = await import("./never_seen.js");
    console.log(mod);
  } catch(e) {
    console.log("ERROR type: " + typeof e);
    console.log("ERROR toString: " + String(e));
  }
}
```

**Expected behavior**: Dynamic `import()` fails at the Sobek engine level (not the k6 resolver level) because k6 does not enable dynamic module resolution in Sobek.

**k6 command**: `k6 run --vus 1 --iterations 1 exp6.js`

**Output**:
```
time="2026-04-13T21:16:15Z" level=info msg="ERROR type: string" source=console
time="2026-04-13T21:16:15Z" level=info msg="ERROR toString: dynamic modules not enabled in the host program" source=console

     iterations...........: 1   3598.49439/s

running (00m00.0s), 0/1 VUs, 1 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.0s/10m0s  1/1 shared iters
```

**Analysis**: The error `"dynamic modules not enabled in the host program"` is a Sobek engine-level rejection. Note that `typeof e` is `"string"` (not `"object"`), confirming this is a raw string rejection from the engine, not a JavaScript `Error` object from k6. The k6 module resolver was never consulted.

---

### Experiment 7: Dynamic import() of a Previously-Resolved Module

**Setup**: `helper.js` is imported statically at init, then dynamically imported during VU execution.

```javascript
// exp7.js
import { greet } from "./helper.js";  // resolve at init (cached)

export default async function() {
  try {
    let mod = await import("./helper.js");  // try dynamic import
    console.log("dynamic import succeeded: " + mod.greet);
  } catch(e) {
    console.log("ERROR: " + String(e));
  }
}
```

**Expected behavior**: Despite `helper.js` being in the cache, the dynamic `import()` still fails because the Sobek engine rejects all dynamic imports unconditionally.

**k6 command**: `k6 run --vus 1 --iterations 1 exp7.js`

**Output**:
```
time="2026-04-13T21:16:20Z" level=info msg="ERROR: dynamic modules not enabled in the host program" source=console

     iterations...........: 1   3634.262372/s

running (00m00.0s), 0/1 VUs, 1 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.0s/10m0s  1/1 shared iters
```

**Analysis**: Same error as Experiment 6. This conclusively proves that dynamic `import()` rejection is at the Sobek engine level and is independent of the k6 module cache. Even a module that is fully resolved and cached cannot be loaded via dynamic `import()`.

---

### Experiment 8: High-Concurrency Cache Stability (20 VUs, 284,966 Iterations)

**Setup**: 20 VUs running for 2 seconds against a previously-resolved module, with an assertion that validates correctness on every iteration.

```javascript
// exp8.js
import { greet } from "./helper.js";

export default function() {
  let msg = greet("VU " + __VU);
  if (msg !== "Hello, VU " + __VU) {
    throw new Error("unexpected: " + msg);
  }
}
```

**Expected behavior**: The cached module serves all VUs under high concurrency with zero errors.

**k6 command**: `k6 run --vus 20 --duration 2s exp8.js`

**Output**:
```
     scenarios: (100.00%) 1 scenario, 20 max VUs, 32s max duration (incl. graceful stop):
              * default: 20 looping VUs for 2s (gracefulStop: 30s)

     iteration_duration...: avg=12.48µs min=1.36µs med=2.5µs max=47.57ms p(90)=5.24µs p(95)=6.89µs
     iterations...........: 284966 142022.805562/s
     vus..................: 20     min=20          max=20
     vus_max..............: 20     min=20          max=20

running (02.0s), 00/20 VUs, 284966 complete and 0 interrupted iterations
default ✓ [ 100% ] 20 VUs  2s
```

**Analysis**: **284,966 iterations** across **20 concurrent VUs** with **zero errors** and **zero interrupted iterations**. The assertion (`msg !== "Hello, VU " + __VU`) validated the module output on every single iteration. The cache served correctly under extreme concurrency (142K iterations/second). No race conditions, no stale data, no cache corruption.

---

### Experiment 9: require() of k6 Builtin Outside Init Context

**Setup**: Script attempts to `require("k6/crypto")` inside the default function.

```javascript
// exp9.js
export default function() {
  let crypto = require("k6/crypto");
  console.log("got crypto: " + typeof crypto);
}
```

**Expected behavior**: Level 1 guard blocks `require()` with the init-context error (same as Experiment 2, but for a built-in module).

**k6 command**: `k6 run --vus 1 --iterations 1 exp9.js`

**Output**:
```
time="2026-04-13T21:16:30Z" level=error msg="GoError: the \"require\" function is only available
in the init stage (i.e. the global scope), see
https://grafana.com/docs/k6/latest/using-k6/test-lifecycle/ for more information\n\tat
go.k6.io/k6/js.(*requireImpl).require-fm (native)\n\tat default
(file:///tmp/k6_experiments/exp9.js:2:23(3))\n" executor=shared-iterations scenario=default
source=stacktrace

     iterations...........: 1   3599.556535/s

running (00m00.0s), 0/1 VUs, 1 complete and 0 interrupted iterations
default ✓ [ 100% ] 1 VUs  00m00.0s/10m0s  1/1 shared iters
```

**Analysis**: The Level 1 guard fires identically for built-in modules as for filesystem modules. The error message is the same `cantBeUsedOutsideInitContextMsg` from `js/initcontext.go:15-16`. This proves that the guard is module-type-agnostic — it checks the VU state, not the module source.

---

### Experiment 10: Go Unit Tests for Lock Behavior

**Setup**: Run the two Go unit tests that directly validate the lock mechanism.

**Command**: `go test -race -run "TestVUDoesNotRequireUnderConditions|TestVUDoesRequireUnderConditions" ./js/ -v -timeout 60s`

**Output**:
```
=== RUN   TestVUDoesNotRequireUnderConditions
=== PAUSE TestVUDoesNotRequireUnderConditions
=== RUN   TestVUDoesRequireUnderConditions
=== PAUSE TestVUDoesRequireUnderConditions
=== CONT  TestVUDoesNotRequireUnderConditions
=== CONT  TestVUDoesRequireUnderConditions
time="2026-04-13T21:16:40Z" level=info msg="test.js 0" source=console
--- PASS: TestVUDoesNotRequireUnderConditions (0.00s)
time="2026-04-13T21:16:40Z" level=info msg="test2.js 0" source=console
time="2026-04-13T21:16:40Z" level=info msg="test.js 1" source=console
time="2026-04-13T21:16:40Z" level=info msg="test2.js 2" source=console
--- PASS: TestVUDoesRequireUnderConditions (0.00s)
PASS
ok  	go.k6.io/k6/js	1.044s
```

**Analysis**: Both tests pass with the race detector enabled:
- `TestVUDoesNotRequireUnderConditions`: Confirms that a module required only when `__VU > 0` is rejected when creating a VU, with the error containing `"was not previously resolved during initialization (__VU==0)"`.
- `TestVUDoesRequireUnderConditions`: Confirms that modules required during `__VU==0` are available to subsequent VUs through the cache.

---

## 8. Code Citations Index

| File | Lines | Symbol | Purpose |
|------|-------|--------|---------|
| `js/modules/resolution.go` | 17 | `notPreviouslyResolvedModule` | Error constant for post-lock module resolution |
| `js/modules/resolution.go` | 24–27 | `moduleCacheElement` | Cache entry struct (module record + error) |
| `js/modules/resolution.go` | 29–40 | `ModuleResolver` | Central resolver struct with `cache`, `locked`, `reverse` map |
| `js/modules/resolution.go` | 69–72 | `requireModule()` lock check | Lock guard for built-in Go modules |
| `js/modules/resolution.go` | 98–131 | `resolveLoaded()` | Parses, compiles, and caches loaded modules |
| `js/modules/resolution.go` | 128–129 | `reverse` + `cache` population | Stores module in both reverse map and cache |
| `js/modules/resolution.go` | 133–139 | `Lock()` | Sets `locked = true` — the freeze mechanism |
| `js/modules/resolution.go` | 145–177 | `resolve()` | Main resolution logic with cache check then lock check |
| `js/modules/resolution.go` | 150–151 | Built-in cache check | Cache hit path for `k6/*` modules |
| `js/modules/resolution.go` | 162–163 | Filesystem cache check | Cache hit path for filesystem modules |
| `js/modules/resolution.go` | 166–168 | Locked guard in `resolve()` | Returns `notPreviouslyResolvedModule` on cache miss + locked |
| `js/modules/resolution.go` | 192–196 | `sobekModuleResolver()` | Sobek callback for ESM import resolution |
| `js/modules/resolution.go` | 198–211 | `reversePath()` | Maps module record to parent directory URL |
| `js/modules/require_impl.go` | 14–65 | `Require()` | CJS require execution path |
| `js/modules/require_impl.go` | 185–196 | `getCurrentModuleScript()` | Stack-frame inspection for CJS parent module |
| `js/modules/modules.go` | 29–33 | `Module` interface | Module contract with `NewModuleInstance(VU)` |
| `js/modules/modules.go` | 36–38 | `Instance` interface | Module instance contract with `Exports()` |
| `js/modules/gomodule.go` | 20–47 | `goModule.Instantiate()` | Go module per-VU instantiation via `vubox` |
| `js/modules/cjsmodule.go` | 61–100 | `cjsModuleInstance.ExecuteModule()` | CJS module execution with `module.exports` |
| `js/bundle.go` | 110–111 | `NewModuleResolver()` call | Resolver creation with empty cache |
| `js/bundle.go` | 125 | `bundle.instantiate(vuImpl, 0)` | First init pass at `__VU==0` |
| `js/bundle.go` | 129 | `bundle.ModuleResolver.Lock()` | Lock call site after init |
| `js/bundle.go` | 418–429 | `requireImpl` struct and method | Init context guard for `require()` |
| `js/bundle.go` | 438–439 | `inInitContext` wiring | Closure checking `vu.state == nil` |
| `js/bundle.go` | 493–504 | `SetFinalImportMeta` | `import.meta.resolve` configuration |
| `js/initcontext.go` | 15–16 | `cantBeUsedOutsideInitContextMsg` | Error string for outside-init-context calls |
| `js/initcontext.go` | 34–44 | `readFile()` | File open error wrapping for post-init access |
| `js/initcontext.go` | 56–64 | `allowOnlyOpenedFiles()` | File freeze pattern parallel |
| `js/runner.go` | 247 | `vu.moduleVUImpl.state = vu.state` | VU state transition out of init context |
| `js/modules_vu.go` | 17–24 | `moduleVUImpl` struct | Per-VU module adapter with `state` field |
| `loader/loader.go` | 48–82 | `Resolve()` | Specifier-to-URL resolution with prefix classification |
| `loader/loader.go` | 53–54 | Relative path detection | `.` or `/` prefix routes to `resolveFilePath()` |
| `loader/loader.go` | 84–112 | `resolveFilePath()` | Joins pwd URL with relative specifier |
| `loader/loader.go` | 111 | `finalPwd.Parse(moduleSpecifier)` | Standard URL resolution for relative paths |
| `js/runner_test.go` | 1409–1432 | `TestVUDoesNotRequireUnderConditions` | Test proving lock rejects unseen modules |
| `js/runner_test.go` | 1434–1478 | `TestVUDoesRequireUnderConditions` | Test proving cache serves seen modules |
| `js/module_loading_test.go` | 30–55 | `TestLoadOnceGlobalVars` | Test proving module deduplication in cache |
| `js/path_resolution_test.go` | 17+ | `TestPathResolution` | Test proving relative specifiers resolve per-file |
| `js/esm_vs_commonjs_test.go` | 11–33 | `TestReturnInCommonJSModule` / `TestReturnInESMModule` | ESM vs CJS syntax behavior differences |
| `js/jsmodules.go` | 32–65 | `getInternalJSModules()` | Built-in `k6/*` module registry |

---

## 9. Test Evidence Summary

### 9.1 `TestVUDoesNotRequireUnderConditions`

**Location**: `js/runner_test.go` lines 1409–1432

**What it tests**: A script that only `require()`-s a module when `__VU > 0` — meaning the module is never loaded during the `__VU==0` init pass.

**Script under test**:
```javascript
if (__VU > 0) {
    let data = require("/home/somebody/test.js");
}
exports.default = function() {
    console.log("hey")
}
```

**Setup**: The file `/home/somebody/test.js` exists on the virtual filesystem (containing `exports=42`), so it is loadable — just never loaded during init.

**Assertions**:
- `r.NewVU(...)` returns a non-nil error.
- `err.Error()` contains `" was not previously resolved during initialization (__VU==0)"`.

**What this proves**: The lock mechanism actively rejects modules that exist on disk but were never resolved during `__VU==0`. Mere existence is not enough — the module must be in the cache.

---

### 9.2 `TestVUDoesRequireUnderConditions`

**Location**: `js/runner_test.go` lines 1434–1478

**What it tests**: A script that loads two modules during `__VU==0`, then conditionally re-requires them per VU.

**Script under test**:
```javascript
if (__VU == 0) {
    require("/home/somebody/test.js");
    require("/home/somebody/test2.js");
}

if (__VU % 2 == 1) {
    require("/home/somebody/test.js");
}

if (__VU % 2 == 0) {
    require("/home/somebody/test2.js");
}

exports.default = function() {
    console.log("hey")
}
```

**Assertions**:
- VU 1 (odd): `r.NewVU(...)` succeeds. Console log contains `"test.js 1"`.
- VU 2 (even): `r.NewVU(...)` succeeds. Console log contains `"test2.js 2"`.

**What this proves**: Modules cached during `__VU==0` remain available to all subsequent VUs through the cache, even when VUs access different subsets of the cached modules.

---

### 9.3 `TestLoadOnceGlobalVars`

**Location**: `js/module_loading_test.go` lines 30–55

**What it tests**: A module with a global variable initialized via `Math.random()` is imported from two different files (using both `module.exports` CJS style and `export` ESM style). Both importers must see the same random value.

**What this proves**: The module cache ensures that a module is evaluated once and its state is shared across all import paths. The same `moduleCacheElement` is returned for the same resolved URL regardless of how many files import it.

---

### 9.4 `TestPathResolution`

**Location**: `js/path_resolution_test.go` lines 17+

**What it tests**: Multiple scenarios of relative path resolution across nested module hierarchies, including:
- Simple relative `open()` across directories (`../../../B/data.txt`).
- Intermediate modules that use relative `require()` to load other modules.
- Complex chains where a module uses `open()` relative to the calling module's directory.
- Paths containing spaces.

**What this proves**: Relative specifiers consistently resolve against the importing module's directory, not the call stack. The test exercises the `reversePath()` → `resolveFilePath()` chain thoroughly.

---

## 10. Summary and Key Answers

### Q1: Does k6 freeze module resolution after initialization?

**Yes, absolutely and intentionally.** The mechanism is `ModuleResolver.Lock()` (`js/modules/resolution.go:137-138`), called at `js/bundle.go:129` immediately after the `__VU==0` init pass completes. This sets the `locked` boolean to `true` on the shared `ModuleResolver` instance. After this point, the `resolve()` function (lines 145–177) only serves modules from its internal `cache` map. Any cache miss results in the `notPreviouslyResolvedModule` error.

**Why**: To guarantee deterministic, reproducible test execution. Every VU operates with an identical, immutable set of modules.

### Q2: What is "already resolved" vs. "new"?

- **"Already resolved"** = present in `ModuleResolver.cache` (a `map[string]moduleCacheElement` keyed by the resolved URL string for filesystem modules, or the raw module name for built-ins). Any module resolved during `__VU==0` — whether via top-level `import`, `require()`, or transitive dependency — is in this cache. Even failed resolutions (errors) are cached.

- **"New/rejected"** = absent from the cache when `locked == true`. The `resolve()` function reaches line 166 (or `requireModule()` reaches line 70) only after a cache miss, and returns:
  ```
  the module "<specifier>" was not previously resolved during initialization (__VU==0)
  ```

### Q3: How do relative specifiers resolve?

**Relative paths resolve against the file that textually contains the `import` or `require()` statement, not against the runtime call stack.** For ESM imports, the Sobek engine provides the `referencingScriptOrModule` (the compiled module record) to `sobekModuleResolver()` (`resolution.go:192-196`), which uses `reversePath()` (`resolution.go:198-211`) to look up the module's parent directory. For CJS `require()`, `getCurrentModuleScript()` (`require_impl.go:185-196`) uses stack frame inspection, but the `SrcName()` is the compile-time source file URL, not a dynamic concept.

This is **deterministic and unambiguous** under multi-VU execution. Each VU re-executes the same compiled source code, so the same import statements reference the same module records, producing identical resolution.

### Q4: What happens with `require()` outside init?

**`require()` has a two-level guard:**
- **Level 1** — Init context check (`js/bundle.go:424-426`): Checks `vu.state == nil`. Once the VU transitions out of init (`js/runner.go:247`), returns: `the "require" function is only available in the init stage (i.e. the global scope)...`
- **Level 2** — Resolver lock check (`js/modules/resolution.go:166-168`): Even during VU init, uncached modules are rejected with `notPreviouslyResolvedModule`.

### Q5: What about dynamic `import()`?

**Dynamic `import()` is globally disabled** in k6's Sobek configuration. All dynamic `import()` calls fail at the JS engine level with the string error `"dynamic modules not enabled in the host program"`, regardless of whether the target module is in the cache. This is independent of the module resolver lock.

### Q6: Does `open()` follow the same pattern?

**Yes.** The `allowOnlyOpenedFiles()` function (`js/initcontext.go:56-64`) switches the filesystem to "only cached" mode after init. The `Lock()` comment explicitly confirms: "It is the same approach used for opening file operations."

### Error Messages Quick Reference

| Error | Source | Trigger |
|-------|--------|---------|
| `the module %q was not previously resolved during initialization (__VU==0)` | `js/modules/resolution.go:17` | Cache miss on locked resolver |
| `the "%s" function is only available in the init stage (i.e. the global scope)...` | `js/initcontext.go:15-16` | `require()` or `open()` called after VU state transition |
| `dynamic modules not enabled in the host program` | Sobek engine (not k6) | Any `import()` expression |
| `open() can't be used with files that weren't previously opened during initialization (__VU==0), path: %q` | `js/initcontext.go:39-41` | `open()` for uncached file after init |
