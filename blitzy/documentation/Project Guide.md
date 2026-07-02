# Blitzy Project Guide — grafana/k6 Onboarding Q&A

> **Deliverable under review:** `blitzy/documentation/k6_ddc3b0b1d23c.md` — a single, evidence-backed onboarding document answering a three-part investigative question about the grafana/k6 project's health and metrics subsystem.
>
> **Task class:** Read-only codebase Q&A / documentation. **Branch:** `blitzy-902a7b7e-b748-45a4-9316-9a8dac59579a` · **HEAD:** `4c94d4cecd` · **Source commit:** `ddc3b0b1d2`
>
> **Legend (Blitzy brand colors):** <span style="color:#5B39F3">■</span> Completed / AI Work = Dark Blue `#5B39F3` · <span style="color:#B23AF2">■</span> Remaining / Not Completed = White `#FFFFFF` (violet-outlined for visibility) · Accent = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This project onboards a new team member to **grafana/k6** — a Go load-testing tool ("Like unit testing, for performance") — by answering, in a single written artifact, three questions about the project's health and its metrics subsystem. It is a **read-only investigation**: the sole deliverable is one markdown document that (Q1) reports the automated test suite's pass/fail/skip health from an actual run, (Q2) explains how k6 tracks metrics by naming the specific files/modules that count iterations and collect performance data, and (Q3) traces the function calls that emit one metric end-to-end. Every claim is grounded in built-and-run evidence with exact `file:line` citations. No k6 source, test, config, or build file is modified.

### 1.2 Completion Status

The AAP-scoped work — environment setup, all three investigative answers, the answer document, and read-only compliance — is fully delivered, validated byte-exact, and committed. The only remaining work is human review and merge (standard path-to-production for a documentation artifact).

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#333333','pieTitleTextSize':'16px'}}}%%
pie showData title k6 Onboarding Q&A — 93.3% Complete
    "Completed Work (AI): 28h" : 28
    "Remaining Work: 2h" : 2
```

| Metric | Value |
|--------|-------|
| **Total Hours** | **30 h** |
| **Completed Hours (AI + Manual)** | **28 h** (AI: 28 h · Manual: 0 h) |
| **Remaining Hours** | **2 h** |
| **Percent Complete** | **93.3 %** |

> **Calculation (PA1, AAP-scoped hours):** Completion % = Completed ÷ (Completed + Remaining) = 28 ÷ (28 + 2) = 28 ÷ 30 = **93.3 %**.

### 1.3 Key Accomplishments

- ✅ **k6 binary built and version-verified** — `go build -mod=vendor` exits 0 with empty stderr; `k6 version` → `v0.55.0 (commit/4c94d4cecd, go1.23.10, linux/amd64)`.
- ✅ **Q1 — Full test-suite health captured & reproducible** — 82 packages (**52 `ok` / 2 `FAIL` / 28 no-test-files**); **4,419 PASS / 6 FAIL / 1 SKIP** (tests + subtests); **0 build-broken packages**, all from an actual `-race` run.
- ✅ **Q1 — Forensic root-cause of the 2 failing packages** — grpc TLS test-fixture key mismatch (certificates valid to **2084/3021**, i.e. *not* expiry) and an http **external OCSP** staple dependency; distinguished from the passing `cert_expired` case.
- ✅ **Q2 — Metrics architecture mapped with 107 byte-exact `file:line` citations** across `metrics/`, `metrics/engine/`, `output/`, `execution/`, `lib/`, `js/`, `cmd/`, `api/v1/`; all four metric types (`Counter`, `Gauge`, `Trend`, `Rate`) named.
- ✅ **Q3 — End-to-end iterations trace verified** — a 1-VU/1-iteration run emits `iterations...........: 1`; the 18-step call chain (with a Mermaid diagram) is confirmed byte-exact from `cmd/run.go` to `CounterSink.Add`.
- ✅ **Read-only directive honored** — `git diff` shows the single added file only; working tree clean; temporary observation scripts confined to `/tmp` and removed.
- ✅ **Document quality** — 557 lines, 54 balanced code fences, 1 Mermaid block, evidence-per-claim discipline; refined across 6 review/checkpoint commits.

### 1.4 Critical Unresolved Issues

There are **no unresolved issues blocking the in-scope deliverable**. The document is complete, accurate, reproducible, and committed. The item below is informational and **out of scope by design**.

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| 6 pre-existing k6 test failures in 2 packages (`js/modules/k6/grpc`, `js/modules/k6/http`) | **None on this deliverable.** They are the documented subject of Q1; environment-sensitive (TLS fixture drift / live external OCSP), not product bugs; **forbidden to fix** by the read-only directive & AAP §0.5.2 | k6 maintainers (upstream, optional/future) | N/A — intentionally not addressed here |

### 1.5 Access Issues

**No access issues identified.**

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| Git repository | Read/Write (branch) | None — checkout, diff, and log all succeeded | ✅ Resolved | — |
| Go toolchain + vendored deps | Build/Test | None — `go1.23.10` present; `vendor/` complete; builds/tests run offline | ✅ Resolved | — |
| k6 runtime | Execute | None — `k6 version` and `k6 run` succeeded | ✅ Resolved | — |

> Note: the `js/modules/k6/http` OCSP test reaches a **live external host**; this is a test-fixture network dependency (documented in Q1), not a repository-access issue for this task.

### 1.6 Recommended Next Steps

1. **[High]** Have a k6-familiar engineer **reproduce Q1** by running the documented CI-equivalent command and confirming the reported aggregates (≈0.5 h).
2. **[High]** **SME spot-check** a sample of the Q2 `file:line` citations and the Q3 18-step trace; run `k6 run --vus 1 --iterations 1` to confirm `iterations = 1` (≈0.5 h).
3. **[Medium]** **Approve and merge** the single documentation artifact; optionally refresh the run-to-run timing values (iterations rate, `iteration_duration`) if a fresh capture is desired — these are already explicitly caveated in the doc (≈1 h).
4. **[Low]** *(Optional, out of scope)* File a separate upstream ticket for the pre-existing grpc/http test failures if the team wishes to harden the k6 fixtures — **not part of this task**.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed work was performed autonomously by Blitzy agents (Manual = 0 h). Each component traces to a specific AAP requirement.

| Component | Hours | Description |
|-----------|------:|-------------|
| Environment setup & k6 binary build *(R1)* | 2 | Provision Go `1.23.10`, build k6 with `-mod=vendor`, verify `k6 version` = `v0.55.0`; establish the runtime foundation for all evidence gathering. |
| Q1 — Test-suite health investigation *(R3)* | 6 | Run the full `-race` suite the way the project defines it; aggregate `--- PASS/FAIL/SKIP` and package `ok/FAIL/?` markers via `grep`/`uniq`; forensic root-cause of the 2 failing TLS packages (cert decode; expiry-vs-key-mismatch distinction; external OCSP). |
| Q2(a) — Iteration-counting architecture *(R4)* | 3 | Map & cite `metrics/builtin.go`, `js/runner.go` (`RunOnce`, `iterationSamples`, `incrIteration`), `lib/executor/*` (`dropped_iterations`), `execution/scheduler.go`. |
| Q2(b) — Performance-data-collection architecture *(R5)* | 3 | Map & cite the `metrics/` core (`registry`, `metric`, `sample`, `sink`, `metric_type` — all 4 types), `metrics/engine/`, and the `output/` pipeline + json/csv/influxdb/cloud backends. |
| Q3 — End-to-end iterations trace *(R6)* | 5 | Author a minimal script; run 1-VU/1-iteration; capture the verbatim `iterations: 1` line; verify the ordered 18-step call chain + Mermaid diagram from run start to `CounterSink.Add`. |
| Answer-document authoring *(R2 + R7)* | 5 | Compose the 557-line artifact with one-evidence-per-claim discipline, exact literals, coverage pass over every named item; create `blitzy/documentation/`. |
| Iterative review & refinement *(QA)* | 3 | 6 commits addressing code-review/checkpoint findings: Q1 TLS root-cause correction, grpc cert point-of-use evidence, final-acceptance exactness, `<HEAD>` reproducibility placeholder, exact `/v1/metrics/` route literal. |
| Read-only compliance & hygiene *(R8)* | 1 | Verify only the doc was added; confirm clean working tree; ensure temporary scripts lived under `/tmp` and were removed. |
| **Total Completed** | **28** | **Matches Completed Hours in §1.2** |

### 2.2 Remaining Work Detail

Each remaining category is human-only path-to-production for a documentation artifact.

| Category | Hours | Priority |
|----------|------:|----------|
| Human SME Review & Verification — reproduce Q1 suite counts; spot-check Q2 citations & Q3 trace; run `k6 run` smoke | 1 | High |
| PR Approval, Merge & Optional Timing-Value Refresh | 1 | Medium |
| **Total Remaining** | **2** | **Matches Remaining Hours in §1.2 and §7 pie chart** |

### 2.3 Hours Reconciliation

| Check | Result |
|-------|--------|
| §2.1 Completed total | 28 h |
| §2.2 Remaining total | 2 h |
| §2.1 + §2.2 | **30 h** = Total in §1.2 ✅ |
| §1.2 ↔ §2.2 ↔ §7 Remaining | **2 h** everywhere ✅ |
| Completion % | 28 ÷ 30 = **93.3 %** ✅ |

---

## 3. Test Results

All results below originate from **Blitzy's autonomous validation runs** of the k6 project's own suite, executed during this session to answer Q1. The deliverable itself is a markdown document and has **no unit tests of its own**; its verification is structural (see last two rows) and by re-running the project suite.

**Command (CI-equivalent, per `.github/workflows/test.yml:L46`):**
```bash
GOMAXPROCS=2 go test -mod=vendor -p 2 -race -timeout 800s -json ./...   # exit code 1
```

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|------------:|-------:|-------:|-----------:|-------|
| Go unit + integration (tests + subtests) | Go `testing` + `testify` | 4,426 | 4,419 | 6 | Not aggregated¹ | 1 SKIP (`TestTC39`); `-race` run |
| Package build/compile | `go test ./...` | 82 pkgs | 80² | 2 | n/a | **0 build-broken**; 2 `FAIL` = grpc + http |
| Runtime smoke (metric emission) | k6 CLI (`k6 run`) | 1 | 1 | 0 | n/a | `iterations = 1`, exit 0 |
| Deliverable structural checks | `grep` / markdown lint | 3 | 3 | 0 | n/a | 557 lines · 54 balanced fences · 1 Mermaid |

¹ CI uploads per-package coverage profiles to Codecov; no single aggregate number was produced in the validation run, so none is claimed here.
² 52 `ok` + 28 `?  [no test files]` = 80 non-failing packages; the 28 `?` packages simply contain no `*_test.go` files (not failures).

**The 2 failing packages (verbatim markers):**
```
--- FAIL: TestRequestAndBatchTLS/ocsp_stapled_good (0.19s)
FAIL	go.k6.io/k6/js/modules/k6/http	7.995s

--- FAIL: TestClient_TlsParameters/ConnectTls (60.01s)
FAIL	go.k6.io/k6/js/modules/k6/grpc	65.690s
```

**Integrity note:** these are **pre-existing** failures (0 `.go` files changed vs source commit `ddc3b0b1d2`), environment-sensitive rather than product bugs, and **out of scope to fix**. Reporting them accurately *is* the Q1 deliverable.

---

## 4. Runtime Validation & UI Verification

k6 is a **command-line / backend tool** with a terminal end-of-test summary rather than a graphical UI (AAP §0.8 — Design System Alignment not applicable). "UI verification" therefore covers the CLI end-of-test summary rendering.

**Runtime health**
- ✅ **Operational** — Build: `go build -mod=vendor` → exit 0, empty stderr (whole binary compiles).
- ✅ **Operational** — Version: `k6 version` → `k6 v0.55.0 (commit/4c94d4cecd, go1.23.10, linux/amd64)`; commit segment equals `git rev-parse --short=10 HEAD` (reproducibility mechanism intact).
- ✅ **Operational** — Metric emission: `k6 run --vus 1 --iterations 1` → exit 0; end-of-test summary prints `iterations...........: 1` via the **shared-iterations** executor.
- ✅ **Operational** — Paired trend: `iteration_duration...: avg≈10.1ms` corroborates the script's `sleep(0.01)`.

**Terminal "UI" (end-of-test summary) verification**
- ✅ **Operational** — Banner, scenario line (`1 iterations shared among 1 VUs`), metric block, and progress line (`1 complete and 0 interrupted iterations`) all render correctly.

**API integration surface**
- ✅ **Operational (documented)** — Live-metrics REST interface exists under `api/v1/` (default address `localhost:6565`); described in Q2 as the during-run metrics exposure.

**Test-suite runtime**
- ⚠ **Partial (by design)** — 80/82 packages green; 2 packages (`grpc`, `http`) fail due to pre-existing, environment-sensitive TLS/OCSP fixtures — reported per Q1, not remediated.

---

## 5. Compliance & Quality Review

Cross-map of AAP deliverables and governing-rule (`SWE-AtlasQnA-Repo`) mandates to their validation status.

| Benchmark / Requirement | Status | Progress | Evidence |
|-------------------------|--------|----------|----------|
| Read-only directive — no existing file modified | ✅ Pass | 100% | `git diff base..HEAD --name-status` = only `A blitzy/documentation/k6_ddc3b0b1d23c.md` |
| Single artifact named `<source_branch>.md` | ✅ Pass | 100% | `blitzy/documentation/k6_ddc3b0b1d23c.md` present (557 lines) |
| Investigate by **running** the code first | ✅ Pass | 100% | Binary built; full suite run; `k6 run` executed |
| Q1 — pass/fail/skip + skipped + broken reported | ✅ Pass | 100% | 52 ok/2 FAIL/28 no-test; 4,419/6/1; 0 build-broken |
| Q2 — every named file/module + all 4 metric types | ✅ Pass | 100% | 107/107 `file:line` citations; Counter/Gauge/Trend/Rate named |
| Q3 — trace ≥1 metric, function calls, data flow to output | ✅ Pass | 100% | 18-step chain + Mermaid; verbatim `iterations: 1` |
| One claim → one piece of verbatim evidence | ✅ Pass | 100% | Evidence line beside each behavioral/structural claim |
| Exact literals cited with `file:line` | ✅ Pass | 100% | Spot-checked byte-exact (`c.Value += s.Value`, `Value: 1`, etc.) |
| Coverage pass over every named item | ✅ Pass | 100% | Coverage grep confirms all named terms present |
| Temporary scripts removed; repo unchanged | ✅ Pass | 100% | Temp files under `/tmp`; working tree clean |
| Markdown structural integrity | ✅ Pass | 100% | 54 balanced code fences; 1 Mermaid block |
| Human SME sign-off | ⏳ Pending | 0% | Path-to-production (see §2.2 / §1.6) |

**Fixes applied during autonomous validation (from commit history):** addressed code-review findings; corrected the Q1 TLS root-cause; added grpc 3021-cert point-of-use evidence and fixed a 2084/3021 cert-pair conflation; resolved final-acceptance exactness/reproducibility findings; introduced the `<HEAD>` placeholder and the exact `/v1/metrics/` route literal.

**Outstanding:** human review/sign-off only.

---

## 6. Risk Assessment

Given the read-only, validated, and fully-documented nature of the work, all risks are **Low or None** in severity.

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| 6 pre-existing test failures (grpc/http) misread as regressions | Technical | Low | Low | Documented as pre-existing (0 `.go` changed), env-sensitive, out of scope; verbatim markers + root-cause included | Accepted / Documented |
| Timing-derived doc values (iterations rate, `iteration_duration`) vary run-to-run | Technical | Low | Medium | Doc explicitly caveats them; stable claims (counts, `iterations = 1`) reproduce exactly | Mitigated |
| Citation line-anchors drift if the tree is rebased | Technical | Low | Low | Citations pinned to commit `ddc3b0b1d2`; version rendered via `<HEAD>` placeholder | Mitigated |
| Security posture change | Security | None | None | No code/dependency changed; no new attack surface (grpc failure is a *test-fixture* key mismatch, not a product defect) | N/A |
| Reproducibility depends on Go `1.23.10` + vendored deps | Operational | Low | Low | Exact toolchain/versions and commands recorded in the doc & §9 | Mitigated |
| Full `-race` suite is long-running (grpc alone ≈65 s) | Operational | Low | Low | Project's own `-timeout` documented (210 s local / 800 s CI) | Mitigated |
| `js/modules/k6/http` OCSP test depends on a **live external host** | Integration | Low | Medium | Documented as an external test dependency, not a k6 defect | Accepted / Documented |
| grpc TLS httptest cert-fixture key drift vs Go toolchain | Integration | Low | Low | Root cause documented (certs valid to 2084/3021, key mismatch — not expiry) | Accepted / Documented |

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = Dark Blue `#5B39F3`; Remaining = White `#FFFFFF`, violet-outlined for visibility):

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#333333','pieTitleTextSize':'16px'}}}%%
pie showData title Project Hours — 93.3% Complete
    "Completed Work" : 28
    "Remaining Work" : 2
```

**Remaining work by priority** (sums to the 2 h in §1.2 / §2.2):

| Priority | Hours | Share |
|----------|------:|------:|
| High (SME review & verification) | 1 | 50% |
| Medium (approval, merge, timing refresh) | 1 | 50% |
| **Total Remaining** | **2** | **100%** |

> **Integrity check:** the pie chart's "Remaining Work" = **2 h** equals §1.2 Remaining Hours and the §2.2 Hours total; "Completed Work" = **28 h** equals §1.2 Completed Hours and the §2.1 total.

---

## 8. Summary & Recommendations

**Achievements.** The task is delivered as one comprehensive, evidence-backed onboarding document that answers all three questions from *built-and-run* evidence: Q1 reports a reproducible test-suite health snapshot (52 `ok` / 2 `FAIL` / 28 no-test-files; 4,419 PASS / 6 FAIL / 1 SKIP; 0 build-broken) with a forensic root-cause of the two failing packages; Q2 maps the metrics subsystem with 107 byte-exact citations and all four metric types; and Q3 traces the `iterations` counter end-to-end across an 18-step call chain, corroborated by a live `iterations = 1` capture.

**Remaining gaps.** None within AAP scope. The outstanding **2 h** is entirely human-side path-to-production: SME verification and PR approval/merge.

**Critical path to production.** (1) Reproduce Q1 → (2) SME spot-check Q2/Q3 → (3) approve and merge the artifact. No engineering fixes are required or permitted for this deliverable.

**Success metrics.** Build exits 0; `k6 run` emits `iterations = 1`; 107/107 citations byte-exact; read-only compliance confirmed (single file added, clean tree); document structurally sound (557 lines, 54 balanced fences).

**Production-readiness assessment.** For this read-only documentation task, every applicable gate passes — dependencies resolve (vendored, offline), the code compiles (0 build-broken), the application runs (`k6 version` + `k6 run`), and the single in-scope deliverable is **complete, accurate, reproducible, and committed**. The project is **93.3 % complete**, with only human review and merge remaining. The document is **production-ready pending sign-off**.

> **Honesty note.** The k6 suite is not 100 % green (6 pre-existing, environment-sensitive failures in 2 packages). These are the *documented subject* of Q1, are out of scope to fix, and are deliberately excluded from the remaining-hours estimate so the completion figure reflects only in-scope work.

---

## 9. Development Guide

How to build, run, and reproduce every answer in the deliverable. All commands were executed during validation; run them from the repository root.

### 9.1 System Prerequisites

- **OS:** Linux or macOS (validated on Linux/amd64).
- **Go toolchain:** CI matrix targets `1.22.x` / `1.23.x`; validated with **`go1.23.10`**. (`go.mod` declares `go 1.21`, pinned `toolchain go1.21.13`; a newer Go builds it fine.)
- **Build essentials:** a C compiler (`gcc`) — required because the suite runs with `-race` (CGO).
- **Tools:** `git`, **`git-lfs` 3.7.1** (pre-push hook is LFS-only), `make`, and GNU coreutils (`grep`, `sort`, `uniq`) for result aggregation.
- **Disk:** ~2 GB for the module cache/build + vendored dependencies.

### 9.2 Environment Setup

Dependencies are **vendored** under `vendor/`, so builds and tests work **offline** — no `go mod download` is needed.

```bash
# From the repository root
go version                       # expect: go version go1.23.10 linux/amd64
head -4 go.mod                   # module go.k6.io/k6 ; go 1.21 ; toolchain go1.21.13
export GOFLAGS=-mod=vendor       # optional: make -mod=vendor the default
```

### 9.3 Dependency Installation

**None required** — all Go dependencies are vendored. Every command uses `-mod=vendor` to resolve exactly the pinned versions.

### 9.4 Build & Startup Sequence

```bash
# 1) Build the k6 binary
go build -mod=vendor -o k6 .

# 2) Verify the build
./k6 version
# → k6 v0.55.0 (commit/<HEAD>, go1.23.10, linux/amd64)
```

### 9.5 Verification Steps

**Reproduce Q1 — test-suite health:**
```bash
# Project form (Makefile:L29)
go test -race -timeout 210s ./...

# CI-equivalent form (.github/workflows/test.yml:L46) — captured for machine-countable results
GOMAXPROCS=2 go test -mod=vendor -p 2 -race -timeout 800s -json ./... > /tmp/test.json 2>/tmp/test.stderr
echo "exit=$?"                                   # → exit=1 (2 packages FAIL)

# Aggregate package-level results (no 'Test' field = package event)
grep -v '"Test":' /tmp/test.json | grep '"Elapsed":' | grep -oE '"Action":"(pass|fail|skip)"' | sort | uniq -c
# → 2 fail, 52 pass, 28 skip   (i.e. 2 FAIL / 52 ok / 28 no-test-files)

# Aggregate test+subtest results (has 'Test' field)
grep '"Test":' /tmp/test.json | grep '"Elapsed":' | grep -oE '"Action":"(pass|fail|skip)"' | sort | uniq -c
# → 6 fail, 4419 pass, 1 skip

wc -c /tmp/test.stderr                           # → 0  (zero build-broken packages)
```

**Reproduce Q3 — the iterations metric (single VU / single iteration):**
```bash
mkdir -p /tmp/k6scratch
cat > /tmp/k6scratch/trace.js <<'EOF'
import { sleep } from 'k6';
export default function () {
  sleep(0.01);
}
EOF
./k6 run --vus 1 --iterations 1 /tmp/k6scratch/trace.js
# → iterations...........: 1   <rate>/s      (exit code 0)
```

**Verify Q2 citations (example):**
```bash
sed -n '53,54p' metrics/sink.go            # CounterSink.Add: c.Value += s.Value
sed -n '90p'     metrics/engine/ingester.go # m.Sink.Add(sample)
sed -n '52p;64p' output/manager.go          # AddMetricSamples ; <-samplesChan
sed -n '899p'    js/runner.go               # Value: 1
```

### 9.6 Example Usage — reading the deliverable

```bash
# The answer document:
less blitzy/documentation/k6_ddc3b0b1d23c.md
# Structure: Orientation → Environment → Q1 (health) → Q2 (architecture) → Q3 (trace) → Coverage & caveats
```

### 9.7 Troubleshooting

- **`go test` exits 1 with 2 `FAIL` packages (grpc, http).** Expected — these are **pre-existing, environment-sensitive** failures (TLS test-fixture key mismatch; live external OCSP staple), *not* regressions. They are the documented subject of Q1 and must **not** be "fixed" for this task.
- **grpc test takes ~65 s.** Normal; keep the project's `-timeout` (210 s local / 800 s CI) so results are representative, not truncated.
- **`-race` build fails.** Ensure a C compiler (`gcc`) is installed; `-race` requires CGO.
- **Offline runs.** Everything builds/tests offline thanks to vendoring — **except** the http OCSP test, which needs network to reach the external responder.
- **Version shows a different commit than `ddc3b0b1d2`.** Expected — the binary stamps the current `HEAD`; cited **source** lines are pinned to `ddc3b0b1d2` and remain valid as long as those files are unchanged.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---------|---------|
| Check Go version | `go version` |
| Build k6 | `go build -mod=vendor -o k6 .` |
| Verify k6 | `./k6 version` |
| Run one iteration | `./k6 run --vus 1 --iterations 1 <script>.js` |
| Full suite (project) | `go test -race -timeout 210s ./...` |
| Full suite (CI form) | `GOMAXPROCS=2 go test -mod=vendor -p 2 -race -timeout 800s ./...` |
| Single package | `go test -mod=vendor ./metrics/` |
| Read-only diff check | `git diff ddc3b0b1d23c128e34e2792fc9075f9126e32375..HEAD --name-status` |
| Verify authorship | `git log --author="agent@blitzy.com" --oneline` |

### Appendix B — Port Reference

| Service | Default Address | Source |
|---------|-----------------|--------|
| k6 live-metrics REST API | `localhost:6565` | `cmd/state/state.go:150` |

### Appendix C — Key File Locations

| Item | Path |
|------|------|
| **Deliverable** | `blitzy/documentation/k6_ddc3b0b1d23c.md` |
| Built-in metric names (`iterations`, `dropped_iterations`) | `metrics/builtin.go:L8-L10` |
| Metric types (Counter/Gauge/Trend/Rate) | `metrics/metric_type.go` |
| `CounterSink.Add` accumulation | `metrics/sink.go:L53-L54` |
| Sample → sink routing | `metrics/engine/ingester.go:L90` |
| Output fan-out / channel read | `output/manager.go:L52, L64` |
| VU sample conduit | `lib/vu_state.go:L59` |
| Per-iteration emission (`Value: 1`) | `js/runner.go:L871, L899` |
| Iteration driver | `lib/executor/helpers.go:L108` |
| Run wiring | `cmd/run.go:L170-L397` |
| Test command | `Makefile:L29` · `.github/workflows/test.yml:L46` |

### Appendix D — Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Module | `go.k6.io/k6` | `go.mod:L1` |
| k6 | `v0.55.0` (source commit `ddc3b0b1d2`) | `k6 version` |
| Go (declared / pinned) | `1.21` / `go1.21.13` | `go.mod:L3-L4` |
| Go (build/test env) | `go1.23.10` | `go version` |
| JS engine (sobek) | `v0.0.0-20241024150027-d91f02b05e9b` | `go.mod` |
| CLI framework (cobra) | `v1.4.0` | `go.mod` |
| Logging (logrus) | `v1.9.3` | `go.mod` |
| Assertions (testify) | `v1.9.0` | `go.mod` |
| git-lfs | `3.7.1` | `git lfs version` |

### Appendix E — Environment Variable Reference

| Variable | Value | Purpose |
|----------|-------|---------|
| `GOFLAGS` | `-mod=vendor` | Use vendored dependencies (offline builds/tests) |
| `GOMAXPROCS` | `2` | Match CI parallelism for representative `-race` results |

### Appendix F — Developer Tools Guide

- **Test aggregation:** `go test -json` + `grep`/`sort`/`uniq` (jq not required) — see §9.5.
- **Citation verification:** `sed -n '<line>p' <file>` to confirm any `file:line` anchor byte-for-byte.
- **Read-only audit:** `git status --porcelain` (expect empty) and `git diff … --name-status` (expect only the one added file).

### Appendix G — Glossary

| Term | Meaning |
|------|---------|
| **VU** | Virtual User — a concurrent execution context running the test script. |
| **Iteration** | One complete execution of the script's `default` function by a VU. |
| **Counter / Gauge / Trend / Rate** | k6's four metric types (`metrics/metric_type.go`). |
| **Sink** | Per-metric accumulator; `CounterSink` sums sample values. |
| **Ingester** | Engine component routing incoming samples into each metric's sink. |
| **Output Manager** | Consumer that drains the sample channel and fans out to all outputs. |
| **`t.Skip` / `--- SKIP`** | Go markers for a test skipped at runtime. |
| **OCSP** | Online Certificate Status Protocol — the external dependency behind the http test failure. |
| **`<HEAD>`** | Placeholder in the doc for the current commit the binary stamps (reproducibility). |

---

*End of Blitzy Project Guide — grafana/k6 Onboarding Q&A.*