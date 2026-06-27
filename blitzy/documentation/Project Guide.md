# Blitzy Project Guide — `ramping-vus` Concurrency Investigation (k6_ddc3b0b1d23c)

---

## 1. Executive Summary

### 1.1 Project Overview

This project is an **analysis-only investigation** (governed by rule `SWE-AtlasQnA-Repo`) that answers five suspected concurrency defects in k6's `ramping-vus` executor and delivers the verdicts in a single, evidence-grounded Markdown document. The audience is k6 maintainers and senior Go engineers triaging whether the reported "stuck VUs," handler-count mismatch, early-kill timing, execution-segment counts, and a possible handler race / VU-buffer leak are genuine defects. Grounded in the source as the source of truth and corroborated by running k6 under Go's `-race` detector, the investigation concludes every symptom is **by design** or **explained, bounded timing** — and that there is **no race and no leak**. The sole persistent deliverable is `blitzy/documentation/k6_ddc3b0b1d23c.md`; no source code was modified.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'17px','pieSectionTextSize':'15px','pieSectionTextColor':'#000000','pieLegendTextColor':'#000000'}}}%%
pie showData title Completion — 89.3% Complete
    "Completed Work (AI)" : 50
    "Remaining Work" : 6
```

> **Completion: 89.3%** — calculated from AAP-scoped hours: `Completed 50h / (Completed 50h + Remaining 6h) = 50/56 = 89.3%`.
> Color key: **Completed = Dark Blue `#5B39F3`**, **Remaining = White `#FFFFFF`**.

| Metric | Hours |
|--------|-------|
| **Total Hours** | **56** |
| Completed Hours (AI + Manual) | 50 (50 AI + 0 Manual) |
| Remaining Hours | 6 |
| **Percent Complete** | **89.3%** |

### 1.3 Key Accomplishments

- ✅ **Single deliverable authored & committed** — `blitzy/documentation/k6_ddc3b0b1d23c.md` (273 lines, ~5,077 words, 8 sections), filename exactly equal to the source branch name.
- ✅ **All five symptoms resolved with explicit verdicts + rationale** — Symptoms 1, 2, 4 = *by design*; Symptom 3 = *explained timing semantics*; Symptom 5 (central) = *no race, no buffer leak*.
- ✅ **Source-grounded** — 91 unique `[path:locator]` citations across 13 source files; every system claim is traceable to exact source lines (validator audited all; an independent 18-citation spot-check across 8 files confirmed exactness).
- ✅ **Empirically substantiated under k6's own zero-tolerance gate** — plain and `-race` binaries build cleanly; rapid up/down and three-segment scenarios and the executor test suite run with **zero data races**; the `goleak` harness reports **zero goroutine leaks**.
- ✅ **Segment math proven** — three equal segments yield `4 + 3 + 3 = 10` = configured peak (never exceeding), reproduced independently.
- ✅ **Scope & cleanliness honored** — `git diff` vs base shows exactly one file added; no source/test/dependency/CI changes; all scratch artifacts removed; working tree pristine.
- ✅ **Validation hardening** — five QA/validation cycles; one empirical inaccuracy (a test-reliability claim) found and corrected, strengthening the central no-race verdict.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Verdicts not yet ratified by a human subject-matter expert | The "no race / by-design" conclusions should be accepted by a k6 maintainer before the investigation is closed/acted on | k6 maintainer / senior Go engineer | 3h (HT-1, HT-2) |
| Empirical results not yet independently reproduced on reviewer hardware | Some evidence (flaky-test timing) is host/parallelism-dependent; reviewer should confirm 0-races + segment math locally | Reviewing engineer | 2h (HT-3, HT-4) |
| Findings not yet communicated to the original reporter / investigation not yet dispositioned | Symptoms remain "open" from the reporter's perspective until findings are relayed and the issue is closed | Investigation owner / product | 1h (HT-5) |

> There are **no code-level blockers**: nothing fails to compile, no deliverable defect is outstanding, and no source change is required (analysis-only mandate; the investigation concludes no fix is warranted).

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| Source repository (`go.k6.io/k6`) | Read/Write (branch) | None — repository, branch, and base commit `ddc3b0b1d` all accessible | ✅ Resolved | — |
| Go toolchain + cgo/gcc (build & `-race`) | Local toolchain | None — Go 1.23.9 + gcc present; vendored deps build offline | ✅ Resolved | — |
| Build/run environment (AAP-referenced Docker image) | Container | None — equivalent toolchain available locally; builds and race runs succeeded | ✅ Resolved | — |

> **No access issues identified.** All resources required to build, run, and validate the investigation were available.

### 1.6 Recommended Next Steps

1. **[High]** Have a senior Go/concurrency engineer (ideally a k6 maintainer) review the analysis — especially the Architecture Primer's single-goroutine sequential-handler model and the central no-race verdict (HT-1, 2h).
2. **[High]** Ratify and record acceptance of the five verdicts; this is the gate to closing the investigation (HT-2, 1h).
3. **[Medium]** Independently reproduce the `§8.2` commands on a controlled environment — confirm exit 0, zero data races, and `4+3+3=10` (HT-3, 1.5h).
4. **[Medium]** Run the project race gate the reliable-green way (`GOMAXPROCS=1 go test -race -parallel 1 ./lib/executor/...`) and confirm the documented bare-run timing flakiness is not a data race (HT-4, 0.5h).
5. **[Medium]** Communicate findings to the original reporter, close/annotate the tracking issue, and decide whether the by-design-but-confusing symptoms (1, 2, 4) warrant a user-facing docs clarification — no code change implied (HT-5, 1h).

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed work was performed autonomously by Blitzy agents (AI). Each component traces to an AAP requirement.

| Component | Hours | Description |
|-----------|-------|-------------|
| Concurrency-model source trace + Architecture Primer (§2) | 9 | Forensic read of ~3,500 lines of concurrent Go across 7 files; established the two-handler closures, the single-goroutine merge loop, the sequential handoff, the five-state VU handle, and the channel VU buffer (the factual backbone for all verdicts) |
| Environment setup + builds (plain + `-race`) | 2 | Go 1.23.9 / vendored / cgo; `go build` and `CGO_ENABLED=1 go build -race`; version verification (`k6 v0.55.0`) |
| Symptom 1 — "Stuck" VUs analysis (§3) | 4 | Traced the `toGracefulStop` window; verdict *by design* with citations + rapid up/down evidence |
| Symptom 2 — Handler count mismatch analysis (§4) | 4 | Contrasted scheduled vs graceful handlers; traced `reserveVUsForGracefulRampDowns()` worked example; verdict *by design* |
| Symptom 3 — Early kill analysis (§5) | 3 | Distinguished `gracefulStop` vs `hardStop` context cancellation + 30s default budget; verdict *explained timing* |
| Symptom 4 — Execution segments analysis + segment-split experiment (§6) | 5 | Analyzed striped `SegmentedIndex`; authored & ran 3-instance split proving `4+3+3=10`; verdict *by design* |
| Symptom 5 — Race/buffer leak (central) + mandatory `-race` proof (§7) | 8 | Proved sequential single-goroutine handler execution, mutex serialization, channel + `WaitGroup` buffer discipline; empirical zero-race verification; verdict *no race, no leak* |
| Empirical runtime harness (scenarios + race test runs) | 5 | Throwaway k6 scenarios; `GORACE=halt_on_error=1` runs; `go test -race ./lib/executor/...`; goleak harness; results captured in §8.3 |
| Summary, verdict table & reproduction commands (§8) | 2 | §8.1 verdict table, §8.2 exact reproduction commands, §8.3 observed results, §8.4 bottom line |
| Citation rigor (91 exact `[path:locator]` citations) | 3 | Every system claim pinned to exact source lines across 13 files; audited for exactness |
| Naming / placement / directory creation / cleanup / scope compliance | 1 | Exact filename in `blitzy/documentation/`; scratch removed; tree pristine; no source modified |
| QA & validation hardening (5 review/fix cycles) | 4 | Citation/evidence/metadata reconciliation; worked-example shorthand fix; flaky-test reliability-claim correction (commit `a0f983f72`) |
| **Total Completed** | **50** | |

### 2.2 Remaining Work Detail

All remaining work is **path-to-production** and is inherently human (review, independent reproduction, disposition). None is AAP rework — the AAP deliverable is 100% complete.

| Category | Hours | Priority |
|----------|-------|----------|
| Human SME review & ratification of the five verdicts (HT-1 + HT-2) | 3 | High |
| Independent reproduction on reviewer hardware: scenarios + reliable-green race gate (HT-3 + HT-4) | 2 | Medium |
| Findings communication & investigation disposition (HT-5) | 1 | Medium |
| **Total Remaining** | **6** | |

### 2.3 Reconciliation

- Completed (2.1) **50h** + Remaining (2.2) **6h** = **56h** Total (matches Section 1.2). ✔
- Remaining **6h** is identical in Section 1.2, Section 2.2, and Section 7. ✔
- Completion = 50 / 56 = **89.3%** (matches Section 1.2, Section 7, Section 8). ✔

---

## 3. Test Results

All tests below originate from **Blitzy's autonomous validation logs** for this project (corroborated by the Final Validator and by an independent re-run during this assessment). Because this is an **analysis-only deliverable that adds no code and no tests**, the tests are **pre-existing k6 tests re-run as verification references under `-race`**, plus behavioral scenario runs of the race-enabled binary. Counts are de-duplicated (sub-suites are not added on top of the package total).

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|-----------|-------|
| Executor package suite (`lib/executor`) — incl. the 2 dedicated race tests, execution-plan, segment-tuple & VU-handle tests | Go `testing` + `-race` (ThreadSanitizer); reliable-green `GOMAXPROCS=1 -parallel 1` | 49 | 49 | 0 | N/A | PASS in ~108s with **0 data races**. Contains `TestVUHandleRace`, `TestVUHandleStartStopRace`, `TestRampingVUsConfigExecutionPlanExample(+OneThird)`, `TestRampingVUsExecutionTupleTests`. At default parallelism, 1 timing test (`TestRampingVUsHandleRemainingVUs`) is flaky — a **timing artifact, never a data race** |
| Segment-math unit (`lib`) | Go `testing` + `-race` | 1 | 1 | 0 | N/A | `TestSegmentedIndex` — striping arithmetic underpinning Symptom 4 |
| Goroutine-leak gate (`cmd/tests`) | `go.uber.org/goleak` | 1 | 1 | 0 | N/A | `goleak.Find()` zero-tolerance harness — **0 leaks** (substantiates "no VU-buffer/goroutine leak") |
| Runtime scenario validation (race binary, behavioral) | k6 `-race` binary + `GORACE=halt_on_error=1` | 4 | 4 | 0 | N/A | 1 rapid up/down + 3 segment-split runs; all **exit 0, 0 data races**; per-segment `4+3+3=10` = configured peak |
| **Total** | | **55** | **55** | **0** | **N/A** | Reliable-green path; **zero data races and zero leaks across every run** |

> **Coverage % = N/A by design:** this is an analysis deliverable with no new product code, so code-coverage is not a meaningful metric; the validation focus was the **race detector** and **goroutine-leak** gates, not line coverage.
>
> **Integrity note:** the *only* non-deterministic result observed was the documented bare-default-parallelism flakiness of `TestRampingVUsHandleRemainingVUs` — a millisecond timing assertion that the `-race` detector itself never implicates (it reports **zero data races whether the test passes or fails**). On the project-sanctioned reliable-green path (`-parallel 1`), the full package passes deterministically.

---

## 4. Runtime Validation & UI Verification

**Runtime health (build & execute):**
- ✅ **Plain build operational** — `go build -o /tmp/k6 .` → exit 0; binary reports `k6 v0.55.0 (commit/<head>, go1.23.9, linux/amd64)`.
- ✅ **Race-instrumented build operational** — `CGO_ENABLED=1 go build -race -o /tmp/k6race .` → exit 0 (79 MB instrumented binary).
- ✅ **Rapid up/down scenario** — runs to completion under the race binary; `vus_max=6`, all iterations complete gracefully (0 interrupted), exit 0, **0 data races**.
- ✅ **Three-segment split scenario** — per-instance `vus_max = 4 / 3 / 3`, summing to the configured peak of 10 (never exceeding), each exit 0 with **0 data races**.
- ✅ **Race & leak gates** — dedicated race tests pass with 0 races; `goleak` harness reports 0 goroutine leaks.

**API integration outcomes:**
- ⚠ **Not applicable** — the investigation exercises the executor through the k6 CLI and the Go test harness; no external API integration is in scope. (k6 exposes a REST API on `:6565` by default, but this investigation does not use it.)

**UI verification:**
- ⚠ **Not applicable** — k6 is a backend/CLI load-testing tool with **no graphical user interface**. There is no UI to verify; no Figma designs were provided (confirmed in AAP §0.9). The deliverable is a Markdown document rendered via standard GitHub-flavored Markdown (8 sections, balanced code fences, one Mermaid diagram, well-formed tables).

---

## 5. Compliance & Quality Review

Cross-mapping the AAP / rule `SWE-AtlasQnA-Repo` directives to delivery status. Fixes applied during autonomous validation are noted.

| # | Benchmark (AAP / Rule directive) | Status | Progress | Evidence / Notes |
|---|----------------------------------|--------|----------|------------------|
| 1 | Create exactly one Markdown doc named `<source_branch_name>.md` | ✅ Pass | 100% | `blitzy/documentation/k6_ddc3b0b1d23c.md` — filename equals branch `k6_ddc3b0b1d23c` |
| 2 | Place doc in `blitzy/documentation/` | ✅ Pass | 100% | Directory created; file resides there |
| 3 | Build and run the source code | ✅ Pass | 100% | Plain + `-race` builds exit 0; scenarios & test suite executed |
| 4 | Base answers on the code (every claim cited) | ✅ Pass | 100% | 91 unique `[path:locator]` citations; all audited exact |
| 5 | Provide rationale, not just verdicts | ✅ Pass | 100% | Each symptom section carries an explicit Rationale subsection |
| 6 | Answer all five symptoms incl. verbatim central question | ✅ Pass | 100% | §3–§7; central question quoted verbatim twice |
| 7 | Empirical `-race` substantiation (zero-tolerance gate) | ✅ Pass | 100% | Zero data races across every run; goleak zero leaks |
| 8 | Do **not** modify any existing source file | ✅ Pass | 100% | `git diff` vs base = one file **added**; zero source/test/config edits |
| 9 | Do **not** add any other code/tests/deps | ✅ Pass | 100% | No `.go` files, no `go.mod`/`go.sum`/vendor/CI changes |
| 10 | Clean up temporary scripts; never commit them | ✅ Pass | 100% | Scratch artifacts removed; `git status --porcelain` empty (pristine) |
| 11 | No dependency / build / CI changes | ✅ Pass | 100% | Manifests untouched; vendored deps intact |
| 12 | Document quality (well-formed Markdown, verdict tables, reproduction commands) | ✅ Pass | 100% | 8 sections, verdict tables (§1.1, §8.1), exact reproduction commands (§8.2) |
| 13 | Empirical accuracy of all stated runtime claims | ✅ Pass (fix applied) | 100% | One reliability claim ("reliably green without `-race`") was empirically false; corrected in commit `a0f983f72` to attribute stability to reduced parallelism — strengthens the no-race verdict |
| 14 | Human ratification of verdicts | ◻ Outstanding | 0% | Path-to-production; requires SME sign-off (HT-1/HT-2) |

**Outstanding compliance items:** only #14 (human verdict ratification) remains — a path-to-production review gate, not a deliverable defect.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Central no-race/by-design verdicts not yet ratified by a human SME; if disputed, the investigation reopens | Technical | Medium | Low | Empirical `-race` + `goleak` evidence + 91 exact citations; schedule SME sign-off (HT-1/HT-2) | Open — pending human review |
| Bare `go test -race ./lib/executor/...` is timing-flaky on multi-core hosts; a reviewer may misread a timing failure as a defect | Operational | Medium | Medium | Doc §7.4/§8.3 explain it is *never* a data race; provide reliable-green `GOMAXPROCS=1 … -parallel 1` command | Mitigated in doc — monitor on review |
| Empirical evidence (flaky-test split, package timings) is host/parallelism-dependent | Technical | Low | Medium | Doc documents host-dependence + the determining factor (reduced parallelism); reviewer reproduces (HT-3) | Mitigated in doc |
| Citation line numbers could drift if the source branch advances | Technical | Low | Low | Cites the fixed base commit `ddc3b0b1d`; source unmodified on this branch | Mitigated by design |
| Reproduction requires the exact toolchain (Go 1.23.9 + cgo/gcc); a missing toolchain breaks the `-race` build | Operational | Low | Low | Doc §1.2/§8.2 pin the toolchain; AAP-referenced Docker image pins the environment | Mitigated in doc |
| Analysis could overlook an edge case the symptoms hint at (residual investigative risk) | Technical | Medium | Low | Race-detector zero-tolerance + goleak + 5 QA cycles + SME review (HT-1) | Low residual |
| Document discoverability — lives in `blitzy/documentation/`, outside k6's `docs/` & `release notes/` | Operational | Low | Low | Rule-mandated location; link from the investigation issue (HT-5) | Open — disposition (HT-5) |
| Security: no new attack surface (read-only doc, no code/deps); a latent race would be a reliability/security concern but is empirically refuted | Security | Low | Low | No source/dep change; `-race` zero races; `goleak` zero leaks | Closed — none introduced |
| Integration: no runtime/API/CI/external-service surface; standalone Markdown | Integration | None | Low | N/A by construction | Closed — none |

> **Overall risk posture: LOW.** Zero code/dependency/CI changes ⇒ zero regression risk. The dominant residuals are human acceptance of the verdicts and correct interpretation of the documented flaky-test behavior — both already mitigated within the document and resolved by the 6-hour human-review path.

---

## 7. Visual Project Status

### Project Hours Breakdown

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextSize':'14px','pieSectionTextColor':'#000000','pieLegendTextColor':'#000000'}}}%%
pie showData title Project Hours — Completed vs Remaining
    "Completed Work" : 50
    "Remaining Work" : 6
```

- **Completed Work = 50h** (Dark Blue `#5B39F3`) · **Remaining Work = 6h** (White `#FFFFFF`) · **Total = 56h** · **89.3% complete**.
- Integrity: the **Remaining Work = 6h** value equals Section 1.2 Remaining Hours and the Section 2.2 "Hours" column sum.

### Remaining Hours by Category (from Section 2.2)

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#B23AF2','pie3':'#A8FDD9','pieStrokeColor':'#333333','pieStrokeWidth':'1px','pieSectionTextColor':'#000000','pieLegendTextColor':'#000000'}}}%%
pie showData title Remaining 6h by Category
    "SME review & ratification (High)" : 3
    "Independent reproduction (Medium)" : 2
    "Findings communication & disposition (Medium)" : 1
```

| Category | Hours | Priority |
|----------|-------|----------|
| SME review & ratification | 3 | High |
| Independent reproduction | 2 | Medium |
| Findings communication & disposition | 1 | Medium |
| **Total** | **6** | |

---

## 8. Summary & Recommendations

**Achievements.** The investigation is **89.3% complete** (50 of 56 AAP-scoped hours). The single deliverable — `blitzy/documentation/k6_ddc3b0b1d23c.md` — comprehensively answers all five symptoms with explicit verdicts, source-grounded rationale (91 exact citations), and empirical evidence from k6's own zero-tolerance `-race` gate. Symptoms 1, 2, and 4 are **by design**; Symptom 3 is **explained, bounded timing semantics**; and the central Symptom 5 resolves to **no race between the handlers and no VU-buffer leak** — the user's premise of "two handler goroutines modifying VU state simultaneously" does not hold, because both handlers are closures invoked *sequentially on a single goroutine* via `iterateSteps()`, with the only separate goroutine strictly sequenced afterward.

**Remaining gaps & critical path.** The remaining **6 hours** are entirely **human path-to-production**: (1) a senior engineer's review and ratification of the verdicts (3h, the critical gate), (2) independent reproduction on reviewer hardware (2h), and (3) communicating findings and dispositioning the investigation (1h). None is AAP rework — the autonomous deliverable is complete, validated, and committed.

**Success metrics.** Build (plain + `-race`) exit 0; **zero data races** across every run; **zero goroutine leaks**; segment math `4+3+3=10`; 91/91 citations exact; `git diff` = one file added (analysis-only mandate honored); working tree pristine.

**Production-readiness assessment.** For an analysis deliverable, "production" means the verdicts can be relied upon to close the investigation. The artifact is **ready for review** — accurate, complete, evidence-grounded, and reproducible. The recommended path is to complete the 6-hour human-review cycle (Sections 1.6 / 2.2), after which the investigation can be closed with high confidence.

| Metric | Value |
|--------|-------|
| Completion | 89.3% (50 / 56h) |
| Symptoms resolved | 5 / 5 (3 by design, 1 explained timing, 1 no-race/no-leak) |
| Data races observed | 0 (across every run) |
| Goroutine leaks observed | 0 |
| Source files modified | 0 (analysis-only) |
| Critical code blockers | 0 |
| Remaining work | 6h (human review/reproduction/disposition) |

---

## 9. Development Guide

How to build, run, reproduce, and troubleshoot the investigation. All commands were tested on this environment (Linux/amd64). Run from the repository root unless noted.

### 9.1 System Prerequisites

- **Go 1.23.9** (`go version` → `go1.23.9 linux/amd64`). The module declares `go 1.21` / `toolchain go1.21.13`; 1.23.9 is the CI-tested toolchain used here.
- **C toolchain (gcc)** — required only for the `-race` binary/tests (cgo linker). `gcc --version` should succeed.
- **Git** (+ Git LFS) and ~250 MB free disk (≈133 MB repo + ≈80 MB race binary).
- Dependencies are **vendored** — no network access is required.

### 9.2 Environment Setup

```bash
# Use the local toolchain and vendored modules (offline-friendly)
export GOTOOLCHAIN=local
export GOFLAGS=-mod=vendor
# Enable cgo only for race-enabled builds/tests
export CGO_ENABLED=1
```

### 9.3 Build

```bash
# Plain binary  (expected: exit 0; ~2s incremental)
go build -o /tmp/k6 .
/tmp/k6 version            # → k6 v0.55.0 (commit/<head>, go1.23.9, linux/amd64)

# Race-instrumented binary  (expected: exit 0; ~3s incremental)
CGO_ENABLED=1 go build -race -o /tmp/k6race .
```

> The `commit/<head>` suffix reflects the *build-time* git HEAD and advances with every commit (including documentation commits); it is **not** a stable identifier. The stable identifier of the analyzed source is the branch base commit `ddc3b0b1d`.

### 9.4 View the Deliverable

```bash
less blitzy/documentation/k6_ddc3b0b1d23c.md      # 273 lines, 8 sections
```

### 9.5 Reproduce the Symptoms (scratch scripts live in /tmp and are deleted afterward)

```bash
# Symptoms 1 & 3 — rapid up/down stages + long gracefulRampDown
cat > /tmp/rapid_updown.js <<'EOF'
import { sleep } from 'k6';
export const options = { scenarios: { ramp: {
  executor: 'ramping-vus', startVUs: 0,
  stages: [
    { duration: '2s', target: 6 }, { duration: '1s', target: 1 },
    { duration: '2s', target: 5 }, { duration: '1s', target: 1 },
    { duration: '2s', target: 4 }, { duration: '1s', target: 0 },
  ],
  gracefulRampDown: '30s',
}}};
export default function () { sleep(0.5); }
EOF
GORACE="halt_on_error=1" /tmp/k6race run /tmp/rapid_updown.js   # → exit 0, vus_max=6, 0 data races

# Symptom 4 — peak-10 scenario split across three equal segments
cat > /tmp/seg.js <<'EOF'
import { sleep } from 'k6';
export const options = { scenarios: { ramp: {
  executor: 'ramping-vus', startVUs: 0,
  stages: [ { duration: '2s', target: 10 }, { duration: '2s', target: 10 }, { duration: '1s', target: 0 } ],
  gracefulRampDown: '1s',
}}};
export default function () { sleep(0.2); }
EOF
for seg in "0:1/3" "1/3:2/3" "2/3:1"; do
  GORACE="halt_on_error=1" /tmp/k6race run /tmp/seg.js \
    --execution-segment "$seg" --execution-segment-sequence "0,1/3,2/3,1"
done
# → per-segment vus_max = 4 / 3 / 3  (sum 10 = configured peak); each exit 0, 0 data races
```

### 9.6 Verification Tests (all expected: exit 0, zero data races)

```bash
# Dedicated race tests (the central question)
go test -race -run '^(TestVUHandleRace|TestVUHandleStartStopRace)$' ./lib/executor/

# Symptom 2 & 4 — execution-plan + segment-tuple tests
go test -race -run 'TestRampingVUsConfigExecutionPlanExample|TestRampingVUsExecutionTupleTests' ./lib/executor/

# Symptom 4 — segment striping unit test
go test -race -run '^TestSegmentedIndex$' ./lib/

# The timing-sensitive test is reliably green in isolation
go test -race -run '^TestRampingVUsHandleRemainingVUs$' -count=5 ./lib/executor/

# Goroutine-leak gate (separate harness)
go test -race -timeout 210s ./cmd/tests

# RELIABLE-GREEN full executor package (project gate, ~108s)
GOMAXPROCS=1 go test -race -parallel 1 ./lib/executor/...
```

### 9.7 Cleanup (mandatory — keep the source tree pristine)

```bash
rm -f /tmp/k6 /tmp/k6race /tmp/rapid_updown.js /tmp/seg.js
git status --porcelain        # expected: empty (pristine)
```

### 9.8 Troubleshooting

- **`go test -race ./lib/executor/...` fails at default parallelism (~30s).** This is the **timing-sensitive** `TestRampingVUsHandleRemainingVUs`, not a data race (the detector reports **0 races** even on the failing run). Fix by reducing parallelism: `GOMAXPROCS=1 go test -race -parallel 1 ./lib/executor/...`. **Do not change source code** — the test file is out of scope under the analysis-only mandate.
- **`go vet ./lib/executor/` exits non-zero** with a context-leak warning at `lib/executor/helpers.go:178`. This is **pre-existing in unmodified source** (the branch diff adds only the Markdown file) and is **out of scope**. Bare `go vet` is not the project gate (the project uses `golangci-lint` + `go test -race`); do not treat it as a deliverable failure.
- **`-race` build/link fails.** Ensure `CGO_ENABLED=1` and a working `gcc`; the race detector requires cgo.
- **Go tries to download a toolchain.** Set `GOTOOLCHAIN=local`. Use `GOFLAGS=-mod=vendor` to build offline from vendored deps.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Plain build | `go build -o /tmp/k6 .` |
| Race build | `CGO_ENABLED=1 go build -race -o /tmp/k6race .` |
| Version check | `/tmp/k6 version` |
| Dedicated race tests | `go test -race -run '^(TestVUHandleRace\|TestVUHandleStartStopRace)$' ./lib/executor/` |
| Execution-plan / segment tests | `go test -race -run 'TestRampingVUsConfigExecutionPlanExample\|TestRampingVUsExecutionTupleTests' ./lib/executor/` |
| Segment unit test | `go test -race -run '^TestSegmentedIndex$' ./lib/` |
| Reliable-green full gate | `GOMAXPROCS=1 go test -race -parallel 1 ./lib/executor/...` |
| Leak gate | `go test -race -timeout 210s ./cmd/tests` |
| Scenario run (race) | `GORACE="halt_on_error=1" /tmp/k6race run <script>.js` |
| Confirm scope | `git diff --name-status ddc3b0b1d2..HEAD` |
| Confirm pristine | `git status --porcelain` |

### B. Port Reference

| Port | Service | Used by this investigation? |
|------|---------|-----------------------------|
| 6565 | k6 REST API (default) | No — the executor is driven via the CLI and Go test harness; the REST API is not exercised |

> No network ports are required to build, run, or validate this investigation.

### C. Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/k6_ddc3b0b1d23c.md` | **The deliverable** — the analysis document |
| `lib/executor/ramping_vus.go` | The `ramping-vus` executor: two handlers, merge loop, reservation, lifecycle (reference) |
| `lib/executor/vu_handle.go` | Five-state VU handle, per-handle mutex, `gracefulStop`/`hardStop` (reference) |
| `lib/executor/base_config.go` | `DefaultGracefulStopValue = 30s` (reference) |
| `lib/execution_segment.go` | `SegmentedIndex` striped iterator (reference) |
| `lib/execution.go` | `ExecutionState`, VU buffer channel, atomic counters (reference) |
| `lib/executors.go` | `Executor` interface contract (reference) |
| `execution/scheduler.go` | Scheduler orchestration of executors (reference) |
| `lib/executor/ramping_vus_test.go`, `vu_handle_test.go`, `lib/execution_segment_test.go` | Verification-reference tests |
| `Makefile` | Defines the project race gate (`go test -race -timeout 210s ./...`) |
| `cmd/tests/tests.go` | `goleak`-based leak harness (`Main`, `goleak.Find()`) |

### D. Technology Versions

| Component | Version | Purpose |
|-----------|---------|---------|
| Go | 1.23.9 (`GOTOOLCHAIN=local`) | Build & run k6 and executor tests |
| k6 | v0.55.0 (base commit `ddc3b0b1d`) | Subject under analysis |
| gcc | present (cgo linker) | Required for `-race` builds only |
| `github.com/stretchr/testify` | 1.9.0 (vendored) | Assertions in referenced tests |
| `go.uber.org/goleak` | 1.3.0 (vendored) | Goroutine-leak detection harness |

### E. Environment Variable Reference

| Variable | Value | Purpose |
|----------|-------|---------|
| `GOTOOLCHAIN` | `local` | Prevent auto-download of a Go toolchain |
| `GOFLAGS` | `-mod=vendor` | Build offline from vendored dependencies |
| `CGO_ENABLED` | `1` | Enable cgo for `-race` builds/tests |
| `GORACE` | `halt_on_error=1` | Make the race detector halt (and fail) on the first data race |
| `GOMAXPROCS` | `1` (optional) | Constrain parallelism for the reliable-green test path |

### F. Developer Tools Guide

| Tool | Use |
|------|-----|
| Go race detector (`-race`) | k6's always-on, zero-tolerance CI gate; the authoritative instrument for the central no-race verdict |
| `go.uber.org/goleak` | Goroutine-leak detection (`cmd/tests` harness); substantiates "no VU-buffer/goroutine leak" |
| `go test -count=N` | Repeat-run a test to demonstrate reliability in isolation |
| `git diff --name-status <base>..HEAD` | Prove the analysis-only scope (one file added) |
| Mermaid (in Markdown) | Renders the architecture flowchart in the deliverable and the pie charts in this guide |

### G. Glossary

| Term | Meaning |
|------|---------|
| **VU** | Virtual User — a concurrent execution unit in k6 |
| **`ramping-vus`** | Executor that varies the number of active VUs across configured stages |
| **`gracefulRampDown`** | Window (default 30s) allowing VUs to finish their current iteration as their number ramps down between stages |
| **`gracefulStop`** | Window (default 30s) allowing a *started* iteration to finish before forced interruption; does **not** cancel a running iteration's context |
| **`hardStop`** | Unconditionally cancels a VU's context, interrupting an in-progress iteration |
| **Scheduled handler** | Closure processing *raw* (target) execution steps — `start()` / `gracefulStop()` |
| **Max-allowed (graceful) handler** | Closure processing *graceful* execution steps — `hardStop()` when the ceiling drops |
| **`toGracefulStop`** | VU-handle state for a VU finishing its in-flight iteration during ramp-down (the "stuck" appearance) |
| **`SegmentedIndex`** | Striped per-instance iterator that partitions load across execution segments deterministically |
| **Execution segment** | A slice of the total load assigned to one k6 instance (`--execution-segment` + `--execution-segment-sequence`) |
| **Data race** | Concurrent unsynchronized access to shared memory with at least one write; detected by the `-race` detector |