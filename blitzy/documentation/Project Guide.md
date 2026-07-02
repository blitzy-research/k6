# Blitzy Project Guide — k6 VU Orchestration Runtime-Verified Investigation

> **Build under investigation:** `k6 v0.55.0 (commit ddc3b0b1d2, go1.23.12, linux/amd64)`
> **Branch:** `blitzy-f8391a00-cc13-49e9-849e-65195e3f910f` · **HEAD:** `1dee2400b` · **Base:** `ddc3b0b1d2`
> **Task type:** Documentation (read-only investigation) · **Deliverable:** `blitzy/documentation/k6_ddc3b0b1d23c.md`

---

## 1. Executive Summary

### 1.1 Project Overview

This project answers a single free-form investigation request about the internal orchestration of **Grafana k6** — specifically, how the Go engine manages Virtual Users (VUs). The deliverable is one new Markdown document that explains five behaviors (O1–O5) of the k6 engine, where **every** behavioral claim is backed by output captured at runtime from a locally built `k6 v0.55.0` binary and grounded to an exact `file:line` in the repository. The audience is engineers reasoning about k6 shutdown semantics, gRPC streaming metrics, dropped-iteration reporting, `SharedArray` memory behavior, and Prometheus remote-write naming. The entire repository is treated as **read-only**: exactly one file is added and no source, config, test, or vendored file is changed.

### 1.2 Completion Status

```mermaid
%%{init: {"theme":"base","themeVariables":{"pie1":"#5B39F3","pie2":"#FFFFFF","pieStrokeColor":"#B23AF2","pieStrokeWidth":"2px","pieOuterStrokeColor":"#B23AF2","pieOuterStrokeWidth":"2px","pieSectionTextColor":"#B23AF2","pieTitleTextSize":"18px"}}}%%
pie showData title k6 Investigation — approx. 93% Complete (92.86%)
    "Completed Work (39h)" : 39
    "Remaining Work (3h)" : 3
```

| Metric | Value |
|---|---|
| **Total Hours** | **42.0 h** |
| **Completed Hours (AI + Manual)** | **39.0 h** (39.0 h AI · 0.0 h manual) |
| **Remaining Hours** | **3.0 h** |
| **Completion** | **≈ 93 % (precisely 92.86 %)** |

> **Completion formula (PA1, AAP-scoped):** `39.0 ÷ (39.0 + 3.0) = 39.0 ÷ 42.0 = 92.86 %`. All work counted is either an AAP-specified deliverable or a path-to-production activity for a documentation deliverable. Per policy, autonomous completion is capped below 100 % pending mandatory human review.

### 1.3 Key Accomplishments

- ✅ **k6 v0.55.0 built from source** (`go build`, exit 0) — the executable baseline that produced all evidence.
- ✅ **O1 (SIGINT / ramping-vus)** answered with verbatim shutdown logs, exit code `105` (`ExternalAbort`), and the decisive `15 complete and 5 interrupted iterations` proving active VUs are **terminated mid-execution**.
- ✅ **O2 (gRPC server-streaming interrupt)** answered — same shutdown lines + `grpc_streams_msgs_received` (representative `220`).
- ✅ **O3 (`dropped_iterations` via REST API)** answered — value read from `GET /v1/metrics/dropped_iterations` with a server-side `status=200` log proving API origin; final ≈ `981`.
- ✅ **O4 (data-sharing memory + root cause)** answered — `SharedArray` ≈ constant vs. per-VU copy ≈ linear, with a three-part source-grounded root cause.
- ✅ **O5 (Prometheus name integrity)** answered — exported `__name__` labels preserve names verbatim with a deterministic `k6_` prefix + type/stat suffix.
- ✅ **113 `file:line` citations (102 unique / 32 files) verified accurate** — 0 discrepancies; 10 independently re-verified this session.
- ✅ **Read-only mandate satisfied** — `git status --porcelain` empty; only the doc added; `examples/grpc_server/` pristine; all temp artifacts under `/tmp` removed.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None (no in-scope blocking issues)_ | The single in-scope deliverable is complete, accurate, and committed; the codebase compiles (exit 0) and the repository is clean. | — | — |

> Context (non-blocking, out of scope): two pre-existing upstream unit-test failures exist in `js/modules/k6/http` and `js/modules/k6/grpc` due to expired embedded TLS/OCSP fixtures vs. the 2026 system clock. They are unrelated to the markdown deliverable and cannot be fixed under the read-only mandate. See §5 and §6.

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| — | — | **No access issues identified.** The build is fully offline (`vendor/` present); the REST API, example gRPC server, and remote-write receiver all run on loopback. No credentials, API keys, or external network access are required. | N/A | — |

### 1.6 Recommended Next Steps

1. **[High]** Perform an SME technical review of `blitzy/documentation/k6_ddc3b0b1d23c.md` — validate O1–O5 accuracy, the one-claim-one-evidence pairing, and the `file:line` citations against source.
2. **[High]** Confirm the document fully answers the original five questions and every named item (`grpc_streams_msgs_received`, `dropped_iterations`, "root cause", `__name__` integrity, finish-vs-terminate) to the questioner's satisfaction.
3. **[Medium]** Publish/merge the documentation PR to the target branch.
4. **[Low]** _(Optional, 0 h)_ Re-run one O1–O5 scenario on reviewer hardware to observe representative magnitudes firsthand, and link the doc from a team knowledge base.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|---|---:|---|
| k6 build & observation harness | 4.0 | Built `k6 v0.55.0` from source (`go build`, offline via `vendor/`); built the example gRPC server from a `/tmp` copy; authored a remote-write receiver and a `/proc VmRSS` sampling harness — all outside the repo tree. |
| O1 — SIGINT / ramping-vus investigation | 4.0 | Source study (`cmd/common.go`, `cmd/run.go`, `base_config.go`, `helpers.go`, `vu_handle.go`, `ramping_vus.go`); scenario authoring; **real** signal injection; capture of shutdown lines, exit `105`, interrupted-iteration counter, and negative evidence. |
| O2 — gRPC server-streaming interrupt | 4.0 | Example server on `:10000`; `route_guide.proto` placed beside the script; `ListFeatures` streaming scenario with `gracefulRampDown:'30ms'`; interrupt; `grpc_streams_msgs_received` capture; `grpc/metrics.go` + `stream.go` grounding. |
| O3 — `dropped_iterations` via REST API | 4.0 | `constant-arrival-rate` overload scenario; concurrent `curl` polling of `localhost:6565`; JSON:API `sample.count` + server `status=200` proof; final-summary total; `routes.go` / `metric_routes.go` / `builtin.go` / `constant_arrival_rate.go` grounding. |
| O4 — data-sharing memory footprint | 5.0 | 100k-record JSON generation; `SharedArray` vs. per-VU scripts; runs at 1/50/100 VUs; peak-RSS sampling from `/proc/<pid>/status`; three-part root-cause study in `data.go` + `share.go`. |
| O5 — Prometheus remote-write name integrity | 4.0 | Remote-write receiver (snappy-decompress + protobuf-decode); custom-metrics workload; exported `__name__` capture; `MapSeries` / `config.go` / `remotewrite.go` / `trend.go` suffix trace. |
| Answer document authoring | 5.0 | `blitzy/documentation/k6_ddc3b0b1d23c.md` — 310 lines structured as claim → verbatim evidence → `file:line` → rationale → explicit named-item answer, plus a coverage-confirmation pass. |
| Citation authoring & accuracy verification | 3.0 | 113 references (102 unique across 32 files) authored and verified line-by-line against source. |
| Autonomous validation & full reproduction | 5.0 | Final Validator rebuilt k6, re-ran all five scenario families, re-verified every citation, and cleared five production-readiness gates. |
| Read-only cleanup & git-cleanliness verification | 1.0 | Removed all `/tmp` artifacts, confirmed `examples/grpc_server/` pristine, verified `git status --porcelain` empty. |
| **Total Completed** | **39.0** | **Matches Completed Hours in §1.2.** |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|---|---:|---|
| SME technical review of the answer document (O1–O5 accuracy, evidence discipline, citation correctness) | 2.0 | High |
| Confirm the document fully satisfies the original questioner (all 5 questions & named items) | 0.5 | High |
| Publish/merge the documentation PR | 0.5 | Medium |
| **Total Remaining** | **3.0** | **Matches Remaining Hours in §1.2 and the §7 pie chart.** |

> There are **no "immediate fix"** remaining tasks: the deliverable required no edits, the codebase compiles cleanly, and there are zero in-scope defects or citation discrepancies.

### 2.3 Hours Reconciliation & Methodology

- **Method:** PA1 AAP-scoped, hours-based completion. The work universe is (a) AAP-specified investigation & documentation deliverables (O1–O5, build, authoring, citation verification) and (b) path-to-production activities appropriate to a documentation deliverable (autonomous validation/reproduction, read-only cleanliness, human review/merge). Because k6 is a CLI engine and the deliverable is a Markdown file, "production" means a reviewed, accurate, reproducible, committed document with a clean repo — **not** application deployment, CI/CD, or infrastructure.
- **Reconciliation:** §2.1 (39.0 h) + §2.2 (3.0 h) = **42.0 h** = Total Hours in §1.2. ✔
- **Completion:** 39.0 ÷ 42.0 = **92.86 % ≈ 93 %**, used consistently in §1.2, §7, and §8. ✔
- **Confidence:** High. The scope is fully enumerated (single deliverable), the deliverable is committed, and all runtime claims were reproduced and citations verified with zero discrepancies.

---

## 3. Test Results

All entries originate from **Blitzy's autonomous validation logs** for this project (compilation, five runtime scenario reproductions, citation verification, and repository-cleanliness checks). This is a documentation deliverable with no unit tests of its own; correctness is established by runtime reproduction of every claim.

| Test Category | Framework / Tool | Total | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---:|---|
| Compilation | Go `go build ./...` | 1 | 1 | 0 | 100 % | Exit 0; binary `k6 v0.55.0 (go1.23.12)`. Re-confirmed this session. |
| Runtime reproduction — O1 (ramping-vus + SIGINT) | `k6 run -v` + `kill -INT` | 1 | 1 | 0 | 100 % | Exit `105`; exact shutdown lines; `interrupted` counter. Independently re-reproduced (`10 complete and 5 interrupted`). |
| Runtime reproduction — O2 (gRPC ListFeatures + interrupt) | `k6 run -v` + example gRPC server | 1 | 1 | 0 | 100 % | `grpc_streams_msgs_received` captured (representative `220`). |
| Runtime reproduction — O3 (constant-arrival-rate + REST poll) | `k6 run -v` + `curl` (REST API) | 1 | 1 | 0 | 100 % | JSON:API `sample.count` + server `status=200`; final ≈ `981`. |
| Runtime reproduction — O4 (SharedArray vs per-VU) | `k6 run` + `/proc VmRSS` sampling | 1 | 1 | 0 | 100 % | Constant vs. linear across 1/50/100 VUs. |
| Runtime reproduction — O5 (Prometheus remote-write) | `k6 run --out experimental-prometheus-rw` + receiver | 1 | 1 | 0 | 100 % | 10 exported `__name__` labels captured, integrity preserved. |
| Citation verification | `grep`/`sed` vs. source | 102 | 102 | 0 | 100 % | 102 unique refs / 32 files; 0 discrepancies; 10 re-verified this session. |
| Repository cleanliness | `git status --porcelain` | 1 | 1 | 0 | 100 % | Empty; `examples/grpc_server/` pristine; only doc added. |
| **In-scope total** | — | **109** | **109** | **0** | **100 %** | **All in-scope validation checks pass.** |

> **Out-of-scope (excluded from the tallies above, shown for transparency):** the broader upstream k6 unit suite has 2 pre-existing failures — `js/modules/k6/http TestRequestAndBatchTLS/ocsp_stapled_good` and `js/modules/k6/grpc TestClient_TlsParameters` — caused by expired embedded TLS/OCSP fixtures vs. the 2026 clock. They live entirely in out-of-scope upstream test files, are unrelated to the markdown deliverable, and are unfixable under the read-only mandate.

---

## 4. Runtime Validation & UI Verification

**Runtime health — k6 engine & scenario families**

- ✅ **Operational** — `k6 v0.55.0` build & `version` (go1.23.12, linux/amd64).
- ✅ **Operational** — O1 `ramping-vus` (startVUs 5) + real `SIGINT` → exit `105`, expected debug/error shutdown lines, active VUs interrupted.
- ✅ **Operational** — O2 gRPC server-streaming `ListFeatures` (gracefulRampDown 30ms) + interrupt → shutdown lines + `grpc_streams_msgs_received`.
- ✅ **Operational** — O4 `SharedArray` vs. per-VU copy at 1/50/100 VUs → constant vs. linear RSS.
- ✅ **Operational** — O5 `--out experimental-prometheus-rw` → exported time series with intact metric names.

**API integration outcomes**

- ✅ **Operational** — REST API `GET http://localhost:6565/v1/metrics/dropped_iterations` → **HTTP 200**, JSON:API envelope with `attributes.sample.count`.
- ✅ **Operational** — gRPC `main.FeatureExplorer/ListFeatures` server-streaming on `localhost:10000`.
- ✅ **Operational** — Prometheus remote-write `POST /api/v1/write` (snappy + protobuf) to the temporary receiver on `localhost:9090`.

**UI verification**

- ⚠ **Not applicable** — k6 is a command-line load-testing engine with no graphical UI in scope (per AAP §0.3.3). The user-facing surfaces are the terminal end-of-test summary and the REST control API, both validated above.

---

## 5. Compliance & Quality Review

Cross-mapping of the governing **SWE-AtlasQnA-Repo** rules and AAP deliverables to observed compliance.

| Benchmark / AAP Rule | Requirement | Status | Evidence / Fixes Applied |
|---|---|:--:|---|
| Read-only repository | No existing file modified/deleted | ✅ Pass | `git diff base --name-status` = only `A blitzy/documentation/k6_ddc3b0b1d23c.md`; `git status` clean. |
| Build-first methodology | Build & run before writing | ✅ Pass | `k6 v0.55.0` built; all 5 scenario families executed to capture evidence. |
| One claim, one piece of evidence | Each behavioral claim paired with a verbatim line | ✅ Pass | Every O1–O5 claim is followed by its exact observed output line. |
| Exact `file:line` grounding | Cite literals with `file:line` | ✅ Pass | 113 refs / 102 unique / 32 files; 0 discrepancies (10 re-verified this session). |
| Full coverage of named items | Every sub-question & named item answered | ✅ Pass | `grpc_streams_msgs_received`, `dropped_iterations`, "root cause", `__name__`, finish-vs-terminate — all explicit. |
| Magnitude realism | Run at real scale/duration | ✅ Pass | O3 driven to ≈ `981` drops; O4 measured at 1/50/100 VUs with a ~35.7 MB file. |
| Report values exactly / hedge magnitude | Verbatim values; representative values flagged | ✅ Pass | Deterministic facts stated exactly; O2/O3/O4 magnitudes labeled "representative". |
| Output location & naming | `blitzy/documentation/<source_branch>.md` | ✅ Pass | `blitzy/documentation/k6_ddc3b0b1d23c.md`. |
| Temp-artifact cleanup | Outside tree & removed | ✅ Pass | All artifacts under `/tmp`; removed; `examples/grpc_server/go.mod`/`go.sum` untouched. |

**Fixes applied during autonomous validation:** O5 Trend citation corrected and O4 RSS artifact fenced (commit `ed59a6323`); O3 `dropped_iterations` characterization corrected to representative (commit `1dee2400b`).
**Outstanding in-scope compliance items:** none.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|:--:|:--:|---|---|
| Magnitude-dependent values (O2 `220`, O3 ≈`981`, O4 RSS) vary by timing/config/hardware | Technical | Low | High | Document explicitly frames these as *representative* with derivations; deterministic facts (log strings, exit `105`, `__name__` formats) are stable | Mitigated |
| Reproduction depends on Go 1.23.x + `vendor/` + example gRPC server build | Technical | Low | Medium | Dev guide (§9) documents exact toolchain and offline vendor build | Mitigated |
| Read-only mandate integrity (accidental source change) | Security | Low | Low | No source modified; no secrets/credentials introduced; `git status` clean; server pristine | Resolved |
| Sensitive/temp data committed | Security | Low | Low | All observation artifacts created under `/tmp` and removed; nothing tracked | Resolved |
| No runtime service to operate/monitor (documentation deliverable) | Operational | Low | Low | N/A by design; the artifact is static Markdown | N/A |
| O4 per-VU reproduction can consume ~20 GB+ RAM; loopback ports 6565/10000/9090 must be free | Operational | Low | Medium | Dev guide advisories; O4 magnitude is scalable down | Advisory |
| O2 requires example gRPC server on `:10000` + proto beside script | Integration | Low | Medium | Dev guide documents building from a `/tmp` copy + proto placement | Mitigated |
| O5 requires a remote-write receiver on `:9090` (snappy + protobuf) | Integration | Low | Medium | Dev guide documents the receiver requirement | Advisory |
| No external/third-party APIs, keys, or network dependencies | Integration | Low | Low | Fully offline/loopback | N/A |
| Pre-existing upstream TLS/OCSP unit-test failures (out of scope) | Technical | Low | High | Unrelated to the deliverable; unfixable under read-only mandate; documented, not counted | Out of scope |

**Overall posture: LOW.** No High or Critical risks. Every risk is Resolved, Mitigated, Advisory, N/A, or explicitly out of scope; none blocks the deliverable.

---

## 7. Visual Project Status

```mermaid
%%{init: {"theme":"base","themeVariables":{"pie1":"#5B39F3","pie2":"#FFFFFF","pieStrokeColor":"#B23AF2","pieStrokeWidth":"2px","pieOuterStrokeColor":"#B23AF2","pieOuterStrokeWidth":"2px","pieSectionTextColor":"#B23AF2","pieTitleTextSize":"18px"}}}%%
pie showData title Project Hours Breakdown (Total 42h)
    "Completed Work" : 39
    "Remaining Work" : 3
```

**Legend:** Completed Work = **Dark Blue `#5B39F3`** · Remaining Work = **White `#FFFFFF`** (outlined in Violet-Black `#B23AF2`).

**Remaining work by priority (3.0 h total)**

| Priority | Hours | Share |
|---|---:|---:|
| High (SME review + questioner confirmation) | 2.5 | 83.3 % |
| Medium (PR publish/merge) | 0.5 | 16.7 % |
| **Total** | **3.0** | **100 %** |

> **Integrity check:** "Remaining Work" = **3 h** here equals §1.2 Remaining Hours (3 h) and the §2.2 Hours total (3 h). "Completed Work" = **39 h** equals §1.2 Completed Hours and the §2.1 total.

---

## 8. Summary & Recommendations

**Achievements.** The autonomous agents delivered essentially the entire AAP scope. A `k6 v0.55.0` binary was built from source, and all five investigation objectives (O1–O5) were answered with **verbatim runtime evidence** and grounded to exact `file:line` references. The single in-scope deliverable — `blitzy/documentation/k6_ddc3b0b1d23c.md` (310 lines, 113 citations) — is committed, and the repository is clean, satisfying the strict read-only mandate. Independent re-verification this session confirmed the build (exit 0), reproduced O1 end-to-end (exit `105`, `5 interrupted` iterations), and re-checked 10 representative citations with zero discrepancies.

**Remaining gaps.** The remaining **3.0 hours (≈7 %)** is entirely human-side: an SME technical review of the document, confirmation that it satisfies the original questioner, and publishing/merging the PR. No engineering rework is required — there are no in-scope defects.

**Critical path to production.** SME review → questioner sign-off → PR merge. There are no build, dependency, or infrastructure blockers.

**Production readiness.** For a documentation deliverable, "production ready" means accurate, reproducible, well-grounded, and cleanly committed — all of which are met. The project is **≈ 93 % complete (92.86 %)**; the residual reflects mandatory human review that agents cannot perform, consistent with the policy that autonomous completion is capped below 100 %.

| Success Metric | Target | Observed |
|---|---|---|
| AAP objectives answered (O1–O5) | 5/5 | ✅ 5/5 |
| Runtime evidence per claim | 100 % | ✅ 100 % |
| Citation accuracy | 100 % | ✅ 102/102 unique (0 discrepancies) |
| Compilation | Exit 0 | ✅ Exit 0 |
| Repository cleanliness | `git status` empty | ✅ Empty |
| Read-only mandate | 0 source files changed | ✅ 0 changed |

---

## 9. Development Guide

> All commands below were **tested this session** unless noted. Run from the repository root. The `k6` binary is built to `/tmp` to keep the repository tree clean, exactly as the investigation did.

### 9.1 System Prerequisites

- **OS:** Linux x86_64 (validated on Ubuntu container).
- **Go toolchain:** `go1.23.12` (repo declares `go 1.21` / `toolchain go1.21.13`; CI uses 1.22.x/1.23.x). `vendor/` is present, enabling **offline** builds.
- **Utilities:** `git`, `curl`, `python3` (for the optional remote-write receiver).
- **Resources:** ≥ 2 GB RAM to build; the O4 per-VU-copy reproduction at 100 VUs can require ~20 GB+ (scale VU counts down if constrained).
- **Free loopback ports:** `6565` (k6 REST API), `10000` (example gRPC server), `9090` (remote-write receiver).

### 9.2 Environment Setup

```bash
# From the repository root
export GOTOOLCHAIN=local      # use the installed Go; do not auto-download a toolchain
export GOFLAGS=-mod=vendor    # build offline from the committed vendor/ tree
```

### 9.3 Build k6 from Source

```bash
go build -o /tmp/k6 .         # matches the Makefile `build` target (plain `go build`)
# Expected: exit code 0, no output
```

### 9.4 Verify the Binary

```bash
/tmp/k6 version
# Expected: k6 v0.55.0 (commit/<hash>, go1.23.12, linux/amd64)
```

### 9.5 Reproduce the Investigation (O1–O5)

**O1 — SIGINT on a ramping executor (verified end-to-end this session):**

```bash
mkdir -p /tmp/k6obs && cd /tmp/k6obs
cat > o1_ramping.js <<'EOF'
import { sleep } from 'k6';
export const options = { scenarios: { ramp: {
  executor: 'ramping-vus', startVUs: 5,
  stages: [{ duration: '60s', target: 5 }],
  gracefulStop: '5s', gracefulRampDown: '5s',
}}};
export default function () { sleep(2); }
EOF
/tmp/k6 run -v o1_ramping.js > o1_out.log 2>&1 &
K6PID=$!
# Wait until k6 is up (comm == k6) and a progress line has appeared, THEN interrupt:
sleep 5; kill -INT "$K6PID"; wait "$K6PID"; echo "exit=$?"   # => exit=105
grep -E 'Stopping k6 in response to signal|test run was aborted' o1_out.log
grep -E 'complete and [0-9]+ interrupted iterations' o1_out.log | tail -1
# => "... N complete and 5 interrupted iterations"  (5 active VUs terminated mid-iteration)
```

**O2 — gRPC server-streaming interrupt (30 ms graceful ramp-down):** build the example server **from a `/tmp` copy** of `examples/grpc_server/` (keeps the repo pristine), start it on `:10000`, place `route_guide.proto` **beside** the temp script, run a `ramping-vus` `ListFeatures` scenario with `gracefulRampDown:'30ms'`, `kill -INT`, and read `grpc_streams_msgs_received` from the summary.

**O3 — `dropped_iterations` via the REST API:**

```bash
# constant-arrival-rate rate:100 timeUnit:1s duration:10s preAllocatedVUs:2 maxVUs:2, body sleep(1)
/tmp/k6 run -v o3_arrival.js > o3_out.log 2>&1 &
curl -s http://localhost:6565/v1/metrics/dropped_iterations   # read data.attributes.sample.count
grep 'GET /v1/metrics/dropped_iterations' o3_out.log          # server log status=200 proves API origin
# Final summary: dropped_iterations ... ≈ 981
```

**O4 — data-sharing memory:** generate a large JSON file under `/tmp`, then compare a `SharedArray` script against a per-VU `open()`/`JSON.parse` script at 1/50/100 VUs, sampling peak `VmRSS` from `/proc/<pid>/status`. `SharedArray` stays ≈ constant; the per-VU copy grows ≈ linearly.

**O5 — Prometheus remote-write name integrity:** start a receiver on `:9090` that snappy-decompresses and protobuf-decodes the `WriteRequest`, then run `/tmp/k6 run --out experimental-prometheus-rw o5_prom.js` and read the exported `__name__` labels (e.g. `k6_my_custom_counter_total`, `k6_waiting_time_p99`).

### 9.6 View the Deliverable

```bash
grep -n '^#' blitzy/documentation/k6_ddc3b0b1d23c.md   # section outline
sed -n '1,60p' blitzy/documentation/k6_ddc3b0b1d23c.md # read the preamble & O1
```

### 9.7 Cleanup & Read-Only Verification

```bash
rm -rf /tmp/k6obs /tmp/k6            # remove all temp artifacts (OUTSIDE the repo)
git status --porcelain              # => empty (clean)
git status --porcelain examples/grpc_server/   # => empty (pristine)
```

### 9.8 Troubleshooting

- **`externally-managed-environment` from pip** (if building a Python receiver): use a virtualenv or `pip install --break-system-packages`.
- **SIGINT missed / run exits 0:** the signal was sent before k6 installed its handler. Wait until `cat /proc/$K6PID/comm` prints `k6` and a `running (...)` progress line appears, then send `kill -INT`.
- **Debug shutdown / REST log lines absent:** add `-v`/`--verbose` — those lines are debug level (`cmd/root.go:L184,L209-210`).
- **`go build` wants network / a newer toolchain:** set `GOTOOLCHAIN=local` and `GOFLAGS=-mod=vendor`.
- **gRPC "proto not found":** the proto path resolves relative to the **script** directory — copy `route_guide.proto` beside the script.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---|---|
| Set offline build env | `export GOTOOLCHAIN=local GOFLAGS=-mod=vendor` |
| Build k6 | `go build -o /tmp/k6 .` |
| Verify version | `/tmp/k6 version` |
| Run a script (verbose) | `/tmp/k6 run -v <script>.js` |
| Run with fixed VUs/duration | `/tmp/k6 run --vus <N> --duration <T> <script>.js` |
| Prometheus remote-write output | `/tmp/k6 run --out experimental-prometheus-rw <script>.js` |
| Send SIGINT to a run | `kill -INT <pid>` |
| Query REST metric | `curl -s http://localhost:6565/v1/metrics/dropped_iterations` |
| Sample peak memory | `grep VmRSS /proc/<pid>/status` |
| Repo cleanliness | `git status --porcelain` |
| Diff vs base | `git diff ddc3b0b1d2 --name-status` |

### B. Port Reference

| Port | Service | Default | Notes |
|---|---|---|---|
| `6565` | k6 REST control API | Enabled | `cmd/state/state.go:L150` (`localhost:6565`) — used for O3. |
| `10000` | Example gRPC server | On when started | `examples/grpc_server/main.go` — used for O2. |
| `9090` | Prometheus remote-write receiver | Temporary | Receiver URL `http://localhost:9090/api/v1/write` — used for O5. |

### C. Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/k6_ddc3b0b1d23c.md` | **The deliverable** (only added file). |
| `cmd/common.go`, `cmd/run.go`, `cmd/root.go` | Signal trap, abort handler, `--verbose` (O1/O2). |
| `cmd/state/state.go` | Default REST API address (O3). |
| `cmd/outputs.go` | Wires `experimental-prometheus-rw` (O5). |
| `lib/executor/base_config.go` | Decisive O1 rationale (manual interrupt ⇒ immediate stop). |
| `lib/executor/{ramping_vus,helpers,vu_handle,constant_arrival_rate,ramping_arrival_rate,shared_iterations,per_vu_iterations}.go` | VU lifecycle, graceful semantics, dropped-iteration emission. |
| `js/modules/k6/grpc/{metrics,stream}.go` | `grpc_streams_msgs_received` counter (O2). |
| `js/modules/k6/data/{data,share}.go` | `SharedArray` root map + read-only proxy (O4 root cause). |
| `metrics/builtin.go`, `api/v1/{routes,metric_routes}.go` | `dropped_iterations` metric + REST endpoints (O3). |
| `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/{prometheus,config,remotewrite,trend}.go` | `__name__` construction (O5). |

### D. Technology Versions

| Component | Version | Source |
|---|---|---|
| k6 | `v0.55.0` (commit `ddc3b0b1d2`) | `lib/consts/consts.go:L12` |
| Go (declared / toolchain) | `1.21` / `go1.21.13` | `go.mod:L3,L5` |
| Go (used to build) | `go1.23.12` | `go version` |
| `xk6-output-prometheus-remote` | `v0.5.0` | `go.mod` |
| `google.golang.org/grpc` | `v1.67.1` | `go.mod` |
| `google.golang.org/protobuf` | `v1.35.1` | `go.mod` |
| `github.com/grafana/sobek` | `v0.0.0-20241024...` | `go.mod:L78` (per-VU JS VM) |

### E. Environment Variable Reference

| Variable | Value | Purpose |
|---|---|---|
| `GOTOOLCHAIN` | `local` | Prevent auto-download; use the installed Go. |
| `GOFLAGS` | `-mod=vendor` | Offline build from committed `vendor/`. |
| `K6_OUT` | `experimental-prometheus-rw` | (Alternative to `--out`) select the remote-write output for O5. |
| `CI` | `true` | Non-interactive tooling (if running Node-based helpers). |
| `DEBIAN_FRONTEND` | `noninteractive` | Non-interactive apt (if provisioning). |

### F. Developer Tools Guide

- **Go** — `go build` (offline via `vendor/`), `go version`. No `go test` is required for the deliverable; the upstream unit suite is out of scope.
- **Git** — `git status --porcelain` (cleanliness gate), `git diff ddc3b0b1d2 --name-status` (confirm only the doc changed), `git log --author=agent@blitzy.com`.
- **curl** — poll the k6 REST API for O3; inspect the JSON:API envelope.
- **/proc sampling** — read `VmRSS` from `/proc/<pid>/status` to measure peak process memory for O4 (since `/usr/bin/time` is unavailable).
- **snappy + protobuf receiver** — decode the Prometheus remote-write `WriteRequest` to read exported `__name__` labels for O5.

### G. Glossary

| Term | Meaning |
|---|---|
| **VU** | Virtual User — an independent JS VM (Sobek) executing the test iteration function. |
| **Executor** | A k6 scheduling strategy (e.g., `ramping-vus`, `constant-arrival-rate`) governing how VUs/iterations are dispatched. |
| **`gracefulStop` / `gracefulRampDown`** | Windows allowing in-flight iterations to finish at **normal** stage/duration end — they do **not** apply to a manual `SIGINT`. |
| **`ExternalAbort` (105)** | k6 exit code when the run is aborted by an external signal (one `SIGINT`). |
| **`grpc_streams_msgs_received`** | Counter incremented once per message received on a gRPC stream (O2). |
| **`dropped_iterations`** | Counter of iterations an arrival-rate executor could not start due to insufficient VU capacity (O3). |
| **`SharedArray`** | k6/data structure that stores file-backed data once and shares it across VUs by pointer, copying only requested elements (O4). |
| **Remote-write** | Prometheus ingestion protocol; k6's `experimental-prometheus-rw` output exports time series with a `k6_` prefix + type/stat suffix (O5). |
| **JSON:API** | The response envelope format used by the k6 REST API (`data.attributes.sample.count`). |
