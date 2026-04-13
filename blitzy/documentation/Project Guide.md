# Blitzy Project Guide

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a deep investigative analysis of Grafana k6 v0.55.0's internal orchestration mechanisms. Five discrete behavioral questions were answered through runtime experiments executed against the unmodified k6 source repository, with all findings documented in a single comprehensive markdown artifact (`blitzy/documentation/k6_ddc3b0b1d23c.md`). The investigation covers VU management under SIGINT, gRPC server streaming interruption, dropped iterations via REST API, SharedArray memory sharing behavior, and Prometheus remote write metric name integrity. The target audience is k6 developers and platform engineers who need authoritative, evidence-based documentation of k6's internal behavior.

### 1.2 Completion Status

```mermaid
pie title Project Completion — 90.2% Complete
    "Completed (37h)" : 37
    "Remaining (4h)" : 4
```

| Metric | Value |
|--------|-------|
| **Total Project Hours** | **41** |
| **Completed Hours (AI)** | **37** |
| **Remaining Hours (Human)** | **4** |
| **Completion Percentage** | **90.2%** |

**Calculation:** 37 completed hours / (37 + 4) total hours = 37 / 41 = **90.2% complete**

### 1.3 Key Accomplishments

- ✅ All 5 investigation questions answered with runtime evidence and source code analysis
- ✅ 1,253-line comprehensive investigation report created at `blitzy/documentation/k6_ddc3b0b1d23c.md` (48KB)
- ✅ 20+ source files analyzed across 6 subsystems (signal handling, executors, gRPC, data sharing, REST API, Prometheus output)
- ✅ 5 runtime experiments designed, executed, and independently validated
- ✅ All 12 source code reference groups verified against actual line numbers in the codebase
- ✅ Repository integrity fully maintained — zero modifications to any existing source files
- ✅ k6 binary builds successfully from source; all relevant test suites pass
- ✅ All temporary experiment artifacts cleaned up; `git status` is clean

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Runtime-specific values (memory measurements, iteration counts) may vary across environments | Low — does not affect correctness of analysis; documented as environment-dependent | Human Reviewer | During review |
| Pre-existing flaky test `TestExecutionInfoVUSharing` in `execution/` (timing-dependent) | None — pre-existing issue unrelated to deliverable; passes on rerun | k6 Core Team | N/A |

### 1.5 Access Issues

No access issues identified. The investigation used only the local source repository and locally built k6 binary. No external services, API keys, or cloud credentials were required.

### 1.6 Recommended Next Steps

1. **[High]** Conduct human technical review of the investigation report for factual accuracy and completeness
2. **[Medium]** Validate runtime experiment reproducibility on at least one additional environment (macOS or different Linux distribution)
3. **[Medium]** Review and approve the pull request for merging into the target branch
4. **[Low]** Polish documentation formatting and add environment-variability disclaimers where applicable

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Environment Setup & k6 Build | 2 | Configured Go 1.21.13 toolchain, built k6 from source with `go build -mod=vendor`, verified version output |
| Source Code Deep Analysis | 10 | Analyzed 20+ source files across signal handling (`cmd/common.go`, `cmd/run.go`), executor framework (`ramping_vus.go`, `vu_handle.go`, `helpers.go`, `constant_arrival_rate.go`), gRPC module (`stream.go`, `client.go`, `metrics.go`), data sharing (`data.go`, `share.go`), REST API (`metric_routes.go`, `routes.go`), Prometheus output (`remotewrite.go`, `prometheus.go`, `config.go`, `trend.go`) |
| Runtime Experiment Design & Execution | 12 | Designed and executed 5 experiments: SIGINT/VU behavior (2.5h), gRPC streaming with companion server (3.5h), dropped iterations API query (1.5h), SharedArray memory comparison (2.5h), Prometheus payload capture (2h) |
| Documentation Writing | 10 | Authored 1,253-line (48KB) comprehensive markdown document with source code excerpts, runtime evidence, verification tables, and summary conclusions |
| Validation & Quality Assurance | 3 | Verified all 12 source code reference groups against actual line numbers, reran experiments for consistency, executed test suites across 8+ packages, confirmed repository integrity |
| **Total Completed** | **37** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Human Technical Review | 2 | High |
| Cross-Environment Validation | 1 | Medium |
| Documentation Polish & Formatting | 0.5 | Low |
| PR Review & Merge | 0.5 | Medium |
| **Total Remaining** | **4** | |

**Verification:** 37 (completed) + 4 (remaining) = **41 total hours** ✅ (matches Section 1.2)

---

## 3. Test Results

All tests listed below were executed by Blitzy's autonomous validation agents during the project lifecycle. No modifications were made to any test files.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|-----------|-------|
| Unit — metrics | `go test` | Package suite | ✅ All | 0 | N/A | `metrics/` and `metrics/engine/` both PASS |
| Unit — api | `go test` | Package suite | ✅ All | 0 | N/A | `api/` and `api/v1/` both PASS |
| Unit — data sharing | `go test` | Package suite | ✅ All | 0 | N/A | `js/modules/k6/data/` PASS |
| Unit — gRPC module | `go test` | Package suite | ✅ All | 0 | N/A | `js/modules/k6/grpc/` PASS |
| Unit — execution | `go test` | Package suite | ✅ All | 0 | N/A | `execution/` PASS (flaky timing test passes on rerun) |
| Unit — executor | `go test` | Package suite | ✅ All | 0 | N/A | `lib/executor/` PASS |
| Unit — errext | `go test` | Package suite | ✅ All | 0 | N/A | `errext/` PASS |
| Unit — loader | `go test` | Package suite | ✅ All | 0 | N/A | `loader/` PASS |
| Unit — output | `go test` | Package suite (8 pkgs) | ✅ All | 0 | N/A | `output/`, `output/cloud/`, `output/cloud/expv2/`, `output/cloud/expv2/integration/`, `output/cloud/insights/`, `output/csv/`, `output/influxdb/`, `output/json/` all PASS |
| Static Analysis | `go vet` | Full repo | ⚠ 1 warning | 0 errors | N/A | Pre-existing warning in `lib/executor/helpers.go:178` (context leak) — not introduced by this project |
| Build Verification | `go build` | Full binary | ✅ Pass | 0 | N/A | k6 v0.55.0 binary builds successfully |
| Runtime — Experiment 1 | k6 run | SIGINT/VU test | ✅ Verified | 0 | N/A | 13 complete + 5 interrupted iterations confirmed |
| Runtime — Experiment 2 | k6 run | gRPC streaming | ✅ Verified | 0 | N/A | `grpc_streams_msgs_received=134` confirmed |
| Runtime — Experiment 3 | k6 run + curl | Dropped iterations API | ✅ Verified | 0 | N/A | API `count=35` at 4s, final `46` confirmed |
| Runtime — Experiment 4 | k6 run + ps | SharedArray memory | ✅ Verified | 0 | N/A | SharedArray=167MB vs open()=386MB confirmed |
| Runtime — Experiment 5 | k6 run + HTTP capture | Prometheus names | ✅ Verified | 0 | N/A | All 10 metric names with `k6_` prefix confirmed |

---

## 4. Runtime Validation & UI Verification

### Build & Binary Verification
- ✅ `go build -mod=vendor -o /tmp/k6 .` — Compiles successfully
- ✅ `k6 version` → `k6 v0.55.0 (commit/b708e11dc6, go1.21.13, linux/amd64)`
- ✅ `go vet -mod=vendor ./...` — No new warnings (only pre-existing `helpers.go:178`)

### Runtime Experiment Results
- ✅ **Experiment 1 (SIGINT/VU):** k6 binary correctly handles SIGINT with graceful shutdown; VUs finish current iterations; 13 complete + 5 interrupted iterations reported
- ✅ **Experiment 2 (gRPC Streaming):** Companion gRPC server built from repo proto definitions; `grpc_streams_msgs_received=134` across 6 streams; cancellation logs verified
- ✅ **Experiment 3 (Dropped Iterations):** REST API at `localhost:6565/v1/metrics/dropped_iterations` returned valid JSON:API response with `count=35`; final summary reported 46
- ✅ **Experiment 4 (SharedArray Memory):** SharedArray=167,440KB vs open()=395,212KB RSS (2.36× ratio); confirms shared-pointer architecture
- ✅ **Experiment 5 (Prometheus Names):** All 10 captured metric names (`k6_data_received_total`, `k6_my_custom_counter_total`, `k6_iteration_duration_p99`, etc.) correctly prefixed with `k6_`

### Repository Integrity
- ✅ `git status` — Clean working tree, no modified/untracked files
- ✅ `git diff origin/k6_ddc3b0b1d23c HEAD --name-status` — Only `A blitzy/documentation/k6_ddc3b0b1d23c.md`
- ✅ All temporary scripts and artifacts removed after experiments

### Source Code Reference Verification
- ✅ `cmd/common.go:96-129` — Signal trap registration confirmed
- ✅ `cmd/run.go:349-361` — Graceful/hard stop closures confirmed
- ✅ `execution/scheduler.go:425-430` — Interrupt detection confirmed
- ✅ `lib/executor/vu_handle.go:147-191` — VU state machine confirmed
- ✅ `lib/executor/helpers.go:104-141` — Iteration accounting confirmed
- ✅ `metrics/builtin.go:10,84` — Dropped iterations definition confirmed
- ✅ `js/modules/k6/grpc/metrics.go:17-27` — gRPC stream metrics confirmed
- ✅ `js/modules/k6/grpc/stream.go:80-218` — Stream lifecycle confirmed
- ✅ `js/modules/k6/data/data.go:18-57` — SharedArray module confirmed
- ✅ `vendor/.../remotewrite/config.go:24` — `defaultMetricPrefix = "k6_"` confirmed
- ✅ `vendor/.../remotewrite/prometheus.go:39-52` — `MapSeries()` confirmed
- ✅ `vendor/.../remotewrite/remotewrite.go:329-356` — Type suffix mapping confirmed

---

## 5. Compliance & Quality Review

| AAP Requirement | Status | Evidence |
|----------------|--------|----------|
| VU Management under SIGINT investigation | ✅ Pass | Source analysis of 11 components + runtime evidence (13 complete, 5 interrupted iterations) |
| gRPC Server Streaming Interruption investigation | ✅ Pass | Source analysis of 5 components + runtime evidence (`grpc_streams_msgs_received=134`) |
| Dropped Iterations via API investigation | ✅ Pass | Source analysis of 5 components + API evidence (`count=35`, final `46`) |
| SharedArray Data Sharing investigation | ✅ Pass | Source analysis of 8 sections + memory evidence (SharedArray=167MB vs open()=386MB) |
| Prometheus Metric Name Integrity investigation | ✅ Pass | Source analysis of 4 sections + payload evidence (all 10 names with `k6_` prefix) |
| Read-Only Repository Constraint | ✅ Pass | `git status` clean; `git diff --name-status` shows only added file |
| Runtime Evidence Requirement | ✅ Pass | All 5 questions backed by exact log output, API responses, or memory measurements |
| No Assumptions Rule | ✅ Pass | All answers derived from source code analysis + observed runtime behavior |
| Documentation Artifact Placement | ✅ Pass | File created at `blitzy/documentation/k6_ddc3b0b1d23c.md` |
| Temporary Artifact Cleanup | ✅ Pass | Zero temporary files remain; working tree clean |
| Source Code Reference Accuracy | ✅ Pass | All 12 reference groups verified against actual line numbers |
| k6 Build Integrity | ✅ Pass | Binary builds successfully; version matches expected output |
| Test Suite Integrity | ✅ Pass | All relevant test packages pass (metrics, api, data, grpc, execution, executor, output) |

### Autonomous Validation Fixes Applied
- Source code reference line numbers verified and corrected during validation
- Runtime experiment values confirmed through independent re-execution
- No fixes required to the deliverable document — all content validated as accurate

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Runtime values (iteration counts, memory) vary across environments | Technical | Low | Medium | Document environment-specific nature; include hardware/OS details in report | Mitigated |
| Pre-existing flaky test `TestExecutionInfoVUSharing` | Technical | Low | Low | Passes on rerun; timing-dependent; not related to deliverable | Accepted |
| Pre-existing `go vet` warning in `lib/executor/helpers.go:178` | Technical | Low | N/A | Pre-existing context leak warning; out of scope for this project | Accepted |
| gRPC experiment requires building companion server | Operational | Low | Low | Server built from repo's own proto definitions; process documented in report | Mitigated |
| Prometheus experiment requires Snappy decompression | Operational | Low | Low | Python capture server handles decompression; methodology documented | Mitigated |
| Documentation may become stale if k6 source changes | Operational | Medium | Medium | Document targets k6 v0.55.0 specifically; version/commit pinned in header | Accepted |
| No security-sensitive data in deliverable | Security | None | N/A | Documentation-only; no credentials, keys, or PII | N/A |
| No external service dependencies | Integration | None | N/A | All experiments use locally built binaries and local servers | N/A |

---

## 7. Visual Project Status

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 37
    "Remaining Work" : 4
```

**Interpretation:** 37 of 41 total project hours completed (90.2%). All 5 AAP investigation requirements fully delivered. Remaining 4 hours are human review, cross-environment validation, and merge tasks.

---

## 8. Summary & Recommendations

### Achievements

The project successfully delivered a comprehensive 1,253-line investigation report analyzing five discrete behavioral aspects of k6 v0.55.0's internal orchestration. All five questions were answered with both deep source code analysis (20+ files across 6 subsystems) and runtime evidence from independently validated experiments. The deliverable meets every AAP constraint: the repository remains unmodified, all answers are grounded in actual code and observed behavior with no assumptions, and the documentation artifact is placed at the specified path.

### Completion Assessment

The project is **90.2% complete** (37 completed hours out of 41 total hours). All AAP-scoped technical work has been delivered and validated. The remaining 4 hours consist entirely of human-side activities: technical accuracy review (2h), cross-environment reproducibility testing (1h), documentation polish (0.5h), and PR merge (0.5h).

### Critical Path to Production

The deliverable is classified as **production-ready** by the autonomous validation system. The critical path to final acceptance is:
1. Domain expert reviews the investigation report for technical accuracy
2. PR is approved and merged

### Success Metrics

| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
| Investigation questions answered | 5 | 5 | ✅ Met |
| Runtime experiments with evidence | 5 | 5 | ✅ Met |
| Source code reference groups verified | 12 | 12 | ✅ Met |
| Repository files modified | 0 | 0 | ✅ Met |
| Test suite regressions introduced | 0 | 0 | ✅ Met |
| Temporary artifacts remaining | 0 | 0 | ✅ Met |

### Production Readiness Assessment

**Ready for human review and merge.** No blocking issues exist. The single deliverable file has been thoroughly validated, and all autonomous work is complete.

---

## 9. Development Guide

### System Prerequisites

| Software | Version | Purpose |
|----------|---------|---------|
| Go | 1.21.13 | Build toolchain (must match `go.mod` toolchain directive) |
| Git | 2.x+ | Repository management |
| Linux/macOS | Any recent | Runtime environment (experiments tested on linux/amd64) |
| curl | Any | REST API queries (Experiment 3) |
| Python 3 | 3.8+ | Prometheus payload capture server (Experiment 5, optional) |

### Environment Setup

```bash
# 1. Clone the repository and switch to the feature branch
git clone <repository-url>
cd k6
git checkout blitzy-cd3859b9-d7c9-4183-bcc3-9e1aac7c7ee7

# 2. Verify Go version matches the required toolchain
go version
# Expected: go version go1.21.13 linux/amd64

# 3. Verify repository integrity — no source files should be modified
git status
# Expected: clean working tree (no modified/untracked files)
```

### Building k6 from Source

```bash
# Build the k6 binary using vendored dependencies
go build -mod=vendor -o ./k6 .

# Verify the build
./k6 version
# Expected: k6 v0.55.0 (commit/..., go1.21.13, linux/amd64)
```

### Running Test Suites

```bash
# Run all test packages relevant to the investigation
go test -mod=vendor ./metrics/...
go test -mod=vendor ./api/...
go test -mod=vendor ./js/modules/k6/data/...
go test -mod=vendor ./js/modules/k6/grpc/...
go test -mod=vendor ./execution/...
go test -mod=vendor ./lib/executor/...
go test -mod=vendor ./output/...
go test -mod=vendor ./errext/...
go test -mod=vendor ./loader/...

# Run static analysis
go vet -mod=vendor ./...
# Note: Pre-existing warning in lib/executor/helpers.go:178 is expected
```

### Viewing the Deliverable

```bash
# The investigation report is located at:
cat blitzy/documentation/k6_ddc3b0b1d23c.md

# File stats:
wc -l blitzy/documentation/k6_ddc3b0b1d23c.md
# Expected: 1253 lines

du -sh blitzy/documentation/k6_ddc3b0b1d23c.md
# Expected: 48K
```

### Reproducing Runtime Experiments

Each experiment in the documentation includes the exact test script and execution command used. To reproduce:

1. **Build k6** using the command above
2. **Copy the test script** from the relevant documentation section to a temporary file (e.g., `/tmp/exp1_sigint.js`)
3. **Run the experiment** using the documented command (e.g., `./k6 run --log-output=stdout --log-format=raw -v /tmp/exp1_sigint.js`)
4. **For SIGINT experiments:** Send `kill -INT <k6_pid>` at the documented timing
5. **For gRPC experiments:** Build the companion server from `lib/testutils/grpcservice/` first
6. **For API experiments:** Enable the API with `--address=localhost:6565` and use `curl` to query

### Troubleshooting

| Issue | Resolution |
|-------|------------|
| `go: command not found` | Ensure Go 1.21.13 is installed and `$GOPATH/bin` is in `$PATH` |
| `go build` fails with vendor errors | Run `go mod vendor` to regenerate vendor directory |
| Flaky test `TestExecutionInfoVUSharing` fails | Re-run with `-count=1`; this is a pre-existing timing-dependent test |
| gRPC experiment fails to connect | Ensure companion gRPC server is running on `localhost:10000` before starting k6 |
| Prometheus experiment produces no output | Verify the Python HTTP capture server is running on port 9090 |
| Different iteration counts than documented | Expected — exact counts depend on CPU speed and scheduling; the behavioral conclusions remain valid |
| Memory measurements differ from documented | Expected — RSS varies by OS, Go version, and hardware; the relative ratio (SharedArray vs open()) should be consistent |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `go build -mod=vendor -o ./k6 .` | Build k6 binary from source |
| `./k6 version` | Verify k6 version and build info |
| `./k6 run --log-output=stdout --log-format=raw -v <script>` | Run k6 with verbose logging to stdout |
| `./k6 run --address=localhost:6565 <script>` | Run k6 with REST API enabled |
| `./k6 run -o experimental-prometheus-rw=http://localhost:9090/api/v1/write <script>` | Run k6 with Prometheus remote write output |
| `curl -s http://localhost:6565/v1/metrics/dropped_iterations` | Query dropped iterations via REST API |
| `go test -mod=vendor ./<package>/...` | Run tests for a specific package |
| `go vet -mod=vendor ./...` | Run static analysis on all packages |
| `kill -INT <pid>` | Send SIGINT to a running k6 process |

### B. Port Reference

| Port | Service | Usage |
|------|---------|-------|
| 6565 | k6 REST API | Metric queries, status, control (`--address=localhost:6565`) |
| 10000 | gRPC companion server | Experiment 2 — ListFeatures streaming RPC |
| 9090 | Prometheus capture server | Experiment 5 — Remote write payload capture |

### C. Key File Locations

| Path | Description |
|------|-------------|
| `blitzy/documentation/k6_ddc3b0b1d23c.md` | **Deliverable** — Investigation report (1,253 lines, 48KB) |
| `cmd/common.go` | Signal handling — `handleTestAbortSignals()` |
| `cmd/run.go` | Test lifecycle — graceful/hard stop closures |
| `execution/scheduler.go` | Scheduler — `Run()`, interrupt detection |
| `lib/executor/vu_handle.go` | VU state machine — 5 states, graceful/hard stop |
| `lib/executor/helpers.go` | Iteration accounting — full vs interrupted |
| `lib/executor/ramping_vus.go` | Ramping VUs executor — stages, ramp-down |
| `lib/executor/constant_arrival_rate.go` | Constant arrival rate — dropped iteration emission |
| `js/modules/k6/grpc/stream.go` | gRPC stream lifecycle |
| `js/modules/k6/grpc/metrics.go` | gRPC streaming metrics registration |
| `js/modules/k6/data/data.go` | SharedArray module — pointer sharing |
| `js/modules/k6/data/share.go` | SharedArray internals — per-access deserialization |
| `api/v1/metric_routes.go` | REST API metric handlers |
| `metrics/builtin.go` | Built-in metric definitions |
| `vendor/.../remotewrite/config.go` | Prometheus `defaultMetricPrefix = "k6_"` |
| `vendor/.../remotewrite/prometheus.go` | Prometheus `MapSeries()` — name construction |
| `go.mod` | Go module manifest — toolchain `go1.21.13` |

### D. Technology Versions

| Technology | Version | Notes |
|------------|---------|-------|
| k6 | v0.55.0 | Load testing framework under investigation |
| Go | 1.21.13 | Build toolchain (specified in `go.mod`) |
| gRPC-Go | vendored | gRPC implementation |
| Prometheus Remote Write | v0.5.0 | `xk6-output-prometheus-remote` extension |
| Sobek (JS runtime) | vendored | JavaScript engine (fork of goja) |
| Logrus | vendored | Structured logging |
| Cobra | vendored | CLI framework |

### E. Environment Variable Reference

| Variable | Default | Purpose |
|----------|---------|---------|
| `K6_ADDRESS` | `localhost:6565` | REST API listen address (equivalent to `--address` flag) |
| `K6_LOG_OUTPUT` | `stderr` | Log output destination (use `stdout` for experiment capture) |
| `K6_LOG_FORMAT` | `json` | Log format (use `raw` for readable experiment output) |
| `K6_OUT` | (none) | Output backend (use `experimental-prometheus-rw` for Experiment 5) |
| `GOPATH` | `~/go` | Go workspace path |
| `PATH` | system | Must include Go binary directory |

### G. Glossary

| Term | Definition |
|------|------------|
| **VU** | Virtual User — a simulated user executing test iterations |
| **SIGINT** | Unix interrupt signal (Ctrl+C or `kill -INT`) |
| **Graceful Stop** | Shutdown mode where VUs finish current iterations before stopping |
| **Hard Stop** | Shutdown mode where VU contexts are immediately cancelled |
| **Dropped Iteration** | A scheduled iteration that could not execute due to insufficient VUs |
| **SharedArray** | k6 data sharing mechanism that stores data once and shares via pointer across VUs |
| **gRPC Server Streaming** | RPC pattern where server sends a stream of messages in response to a single client request |
| **Prometheus Remote Write** | Protocol for pushing metrics to Prometheus-compatible backends |
| **JSON:API** | Specification for building APIs in JSON, used by k6's REST API |
| **RSS** | Resident Set Size — physical memory used by a process |
| **Sobek** | JavaScript runtime engine used by k6 (fork of goja) |
| **Executor** | k6 component that controls VU scheduling (e.g., `ramping-vus`, `constant-arrival-rate`) |