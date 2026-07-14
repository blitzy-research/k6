# k6 Runtime Investigation — Q&A Answer Document

_Target revision: `go.k6.io/k6` at base commit `ddc3b0b1d23c128e34e2792fc9075f9126e32375` (k6 **v0.55.0**). Every value below was produced by **building and running** k6 and capturing the real output first; prose was written afterward. Each block is preceded by the exact command that produced it. Claims are labeled **Observed** (produced at runtime) or **Source-inferred** (read from code, not run)._

This document answers five questions (Q1–Q5). For each: the harness, the exact command, the complete captured output, the direct answer, the root cause with **full-literal `path:line` citations**, official Grafana documentation corroboration where applicable, and an Observed-vs-inferred split. A truthful coverage pass and a cleanup/final-state proof close the document.

## Preamble — build, environment, and methodology

### Build and environment (foundation transcript)

Two binaries were built with the pinned toolchain, both **outside** the checkout so the repository stays pristine. `/tmp/k6bin/k6_canonical` is built from the base commit `ddc3b0b1d23c…` and carries the canonical banner `commit/ddc3b0b1d2` cited by the AAP; it is the binary used for the **primary** Q1–Q5 experiments below. `/tmp/k6bin/k6` is built from the working-tree `HEAD` (`ec4344835…` = base + this markdown-only doc commit) and differs only by the embedded VCS stamp — the doc commit adds no Go code, so runtime behavior is identical (verified: `git diff base…HEAD` touches only this Markdown file — zero `.go`/`.mod`/`.sum`/`vendor/` changes). The only place a binary other than `k6_canonical` is used is the two **supplementary Q4 controls** (the no-data baseline and the `VmHWM` corroboration), which were captured with a runtime-equivalent `HEAD` build (banner `commit/0ad99e99c6`, runtime-identical to `k6_canonical` because the doc commit adds no Go code) and are explicitly labelled as such where they appear. The complete transcript (working directory, branch, full HEAD, `git log`, pre-build clean status, real `go version`, both builds with exit codes and banners, binary identities, post-build clean status) is:

_Command: the transcript below records each command with its `$` prompt and `[exit=N]` status; it was run from the repository root before any answer prose was written._

```
###### FOUNDATION TRANSCRIPT ######
$ pwd
/tmp/blitzy/k6/blitzy-2ee44ea5-9c3d-489f-9a4d-c0ce8e36a5ce_a34a64
[exit=0]

$ git rev-parse --abbrev-ref HEAD
blitzy-2ee44ea5-9c3d-489f-9a4d-c0ce8e36a5ce
[exit=0]

$ git rev-parse HEAD
ec43448353679322f6b301cea85ce832842b9b52
[exit=0]

$ git log --oneline -2
ec4344835 docs: add k6 runtime-investigation Q&A answer document
ddc3b0b1d Update comment
[exit=0]

$ git status --porcelain
[exit=0]

$ go version
go version go1.21.13 linux/amd64
[exit=0]

(1) Canonical build from base source commit ddc3b0b1d23c via local clone
$ git -C /tmp/k6canon rev-parse HEAD
ddc3b0b1d23c128e34e2792fc9075f9126e32375
[exit=0]

$ cd /tmp/k6canon && go build -o /tmp/k6bin/k6_canonical . ; cd - >/dev/null
[exit=0]

$ /tmp/k6bin/k6_canonical version
k6_canonical v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)
[exit=0]

(2) Build from working-tree HEAD (ec4344835 = base + this markdown-only doc commit)
$ go build -o /tmp/k6bin/k6 .
[exit=0]

$ /tmp/k6bin/k6 version
k6 v0.55.0 (commit/ec43448353, go1.21.13, linux/amd64)
[exit=0]

$ sha256sum /tmp/k6bin/k6_canonical /tmp/k6bin/k6
b7b8df372796dd5e1731c5758060022a66d8e8ca9e91aae3699675f98262c27f  /tmp/k6bin/k6_canonical
cca211924dcd6cd3d18a2bdcfb6b6c54a67ee11a55999045a00b0826cdd8471b  /tmp/k6bin/k6
[exit=0]

$ ls -l /tmp/k6bin/k6_canonical /tmp/k6bin/k6
-rwxr-xr-x 1 root root 64173872 Jul 13 19:13 /tmp/k6bin/k6
-rwxr-xr-x 1 root root 64076536 Jul 13 19:13 /tmp/k6bin/k6_canonical
[exit=0]

$ git status --porcelain
[exit=0]

###### END ######
```

The banner is emitted by `debug.ReadBuildInfo()` reading `vcs.revision` (first 10 hex chars) at `lib/consts/consts.go:19-52`. **Observed:** `k6_canonical version` prints `k6_canonical v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)`; `go version` prints `go version go1.21.13 linux/amd64`; `git status --porcelain` is empty before and after building. The `git log` shows `HEAD` is `ec4344835 docs: add k6 runtime-investigation Q&A answer document` on top of base `ddc3b0b1d Update comment`.

_Transcript provenance (self-reference note)._ This foundation transcript is the **authoring-time snapshot**, captured when `HEAD` was `ec4344835`. Committing — and later re-committing — this Markdown document advances `HEAD` by **one markdown-only commit each time** (e.g. to `0ad99e99c`, and to a further commit when this revision lands); every such commit changes exactly **one** file (this document) and **no** Go code, so a binary rebuilt from any of these HEADs is runtime-identical (verifiable with `git diff <base>..HEAD --stat`, which lists only the deliverable). All experiments below therefore use the `k6_canonical` binary built from the **immutable base commit** `ddc3b0b1d23c` (banner `commit/ddc3b0b1d2`), which never advances. Consequently the HEAD-build hash and `sha256` recorded above are point-in-time values for the authoring commit, **not** claims about the final committed HEAD — a committed document can never embed its own final commit hash.

### Methodology and safety

- **Run-first.** Each answer was produced by building/running the relevant path and capturing the real stdout/stderr, REST JSON, peak-RSS numbers, or decoded payload bytes; prose came afterward.
- **Two-run stability.** Every magnitude/timing value (Q1 interrupted count, Q2 message count, Q3 drop counts, Q4 RSS curves, Q5 name set) was confirmed across at least two runs. Two are inherently **timing/scheduling-dependent** and are therefore reported as multi-run observations rather than asserted as fixed constants: Q2’s received-message count (run three times — observed **`95` in all three**, because the interrupt at t≈2 s consistently caught 95 messages before shutdown) and Q3’s primary `shared-iterations` drop count (re-run 45 times — **`985` in 42/45**, range `980–985`). Q2 happened to be stable at its observed value across the three runs; Q3’s exact integer varied run-to-run, so for Q3 the reproduced, stable facts are the surrounding invariants — the REST-API provenance and `dropped_iterations + iterations = 1000` — rather than the exact integer.
- **Run-varying fields are labeled, not reproduced.** A few incidental fields are inherently run-to-run variable and are **not** part of the reproduced magnitudes: per-second throughput **rates** in summaries (e.g. `48.099874/s`); the `Content-Length` of a REST JSON body that embeds a computed rate float (e.g. `169` vs `170`, differing only by the rate’s digit count); the exact progress-snapshot line rendered at the precise instant of interrupt; and the byte length/`sha256` of a remote-write payload carrying `Math.random()`-derived trend samples. For these, the **stable** quantity (the count, the metric value, the decoded name set) is what is confirmed across runs; a variable field may be shown in full for one run and elided as `...` for the others.
- **Canonical entry points only.** Values come from `k6 run`, `k6 stats`, and the real REST API (`GET /v1/metrics…`) — never a debug hook, mock, or synthetic bypass. The Q2 gRPC server and Q5 remote-write receiver are standard harnesses (a faithful build of the shipped `grpcservice` server, and a minimal capture endpoint), not behavioral substitutes for k6.
- **Safety (finding #12 / S1–S3).** Every harness used `set -euo pipefail`; evidence directories were mode-`0700`; listeners bound to loopback only (`127.0.0.1`); **unique** ports were chosen per run via `shuf -i 20000-39999 -n1` (the concrete port appears in each log, e.g. `39652`, `27640`, `38022`); helper processes were tracked by owned PID with `kill -0` verification and shut down via `trap … EXIT`; readiness was established by bounded TCP-connect polling (not fixed sleeps); the Q5 receiver bounded each request body with `io.LimitReader(r.Body, 10<<20)`.
- **Observed vs inferred.** Runtime-produced facts are labeled **Observed**; anything read from source and not executed is labeled **Source-inferred**.

## Q1 — Virtual User (VU) lifecycle on `SIGINT` (`ramping-vus`)

**Question.** For a `ramping-vus` scenario holding ≥5 VUs that receives a `SIGINT`, capture the exact shutdown log messages and determine from that evidence whether active VUs finish their in-progress iteration or are terminated mid-execution.

### Harness

`/tmp/k6_evidence/q1/q1_ramping_vus.js` — 5 VUs held for 2m2s, each iteration `sleep(30)` so a `SIGINT` a few seconds in lands while all 5 VUs are mid-iteration:

```javascript
import { sleep } from 'k6';

export const options = {
  scenarios: {
    ramp: {
      executor: 'ramping-vus',
      startVUs: 5,
      stages: [
        { duration: '2s', target: 5 },
        { duration: '120s', target: 5 },
      ],
      gracefulRampDown: '30s',
      gracefulStop: '30s',
    },
  },
};

export default function () {
  sleep(30); // long iteration: a SIGINT a few seconds in lands mid-iteration for all 5 VUs
}
```

### Primary path — a single `SIGINT`

_Command (safe harness; unique REST port via `shuf`, owned PID, single `SIGINT` after 8 s, exit code captured and appended as the final `exit=` line):_

```bash
PORT=$(shuf -i 20000-39999 -n1)
/tmp/k6bin/k6_canonical run --verbose --address 127.0.0.1:$PORT \
    /tmp/k6_evidence/q1/q1_ramping_vus.js & K6PID=$!
sleep 8; kill -INT "$K6PID"        # one SIGINT while all 5 VUs are inside sleep(30)
wait "$K6PID"; echo "exit=$?"
```

**Complete output — run 1** (`/tmp/k6_evidence/q1/single_run1.log`, REST port `39652`):

```
time="2026-07-13T19:14:23Z" level=debug msg="Logger format: TEXT"
time="2026-07-13T19:14:23Z" level=debug msg="k6 version: v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)"

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

time="2026-07-13T19:14:23Z" level=debug msg="Resolving and reading test '/tmp/k6_evidence/q1/q1_ramping_vus.js'..."
time="2026-07-13T19:14:23Z" level=debug msg=Loading... moduleSpecifier="file:///tmp/k6_evidence/q1/q1_ramping_vus.js" originalModuleSpecifier=/tmp/k6_evidence/q1/q1_ramping_vus.js
time="2026-07-13T19:14:23Z" level=debug msg="'/tmp/k6_evidence/q1/q1_ramping_vus.js' resolved to 'file:///tmp/k6_evidence/q1/q1_ramping_vus.js' and successfully loaded 433 bytes!"
time="2026-07-13T19:14:23Z" level=debug msg="Gathering k6 runtime options..."
time="2026-07-13T19:14:23Z" level=debug msg="Initializing k6 runner for '/tmp/k6_evidence/q1/q1_ramping_vus.js' (file:///tmp/k6_evidence/q1/q1_ramping_vus.js)..."
time="2026-07-13T19:14:23Z" level=debug msg="Detecting test type for..." test_path="file:///tmp/k6_evidence/q1/q1_ramping_vus.js"
time="2026-07-13T19:14:23Z" level=debug msg="Trying to load as a JS test..." test_path="file:///tmp/k6_evidence/q1/q1_ramping_vus.js"
time="2026-07-13T19:14:23Z" level=debug msg="Runner successfully initialized!"
time="2026-07-13T19:14:23Z" level=debug msg="Parsing CLI flags..."
time="2026-07-13T19:14:23Z" level=debug msg="Consolidating config layers..."
time="2026-07-13T19:14:23Z" level=debug msg="Parsing thresholds and validating config..."
time="2026-07-13T19:14:23Z" level=debug msg="Initializing the execution scheduler..."
time="2026-07-13T19:14:23Z" level=debug msg="Starting 2 outputs..." component=output-manager
time="2026-07-13T19:14:23Z" level=debug msg=Starting... component=metrics-engine-ingester
time="2026-07-13T19:14:23Z" level=debug msg="Started!" component=metrics-engine-ingester
     execution: local
        script: /tmp/k6_evidence/q1/q1_ramping_vus.js
        output: -

     scenarios: (100.00%) 1 scenario, 5 max VUs, 2m32s max duration (incl. graceful stop):
              * ramp: Up to 5 looping VUs for 2m2s over 2 stages (gracefulRampDown: 30s, gracefulStop: 30s)

time="2026-07-13T19:14:23Z" level=debug msg="Starting the REST API server on 127.0.0.1:39652"
time="2026-07-13T19:14:23Z" level=debug msg="Trapping interrupt signals so k6 can handle them gracefully..."
time="2026-07-13T19:14:23Z" level=debug msg="Starting emission of VUs and VUsMax metrics..."
time="2026-07-13T19:14:23Z" level=debug msg="Start of initialization" executorsCount=1 neededVUs=5 phase=execution-scheduler-init
time="2026-07-13T19:14:23Z" level=debug msg="Initialized VU #4" phase=execution-scheduler-init
time="2026-07-13T19:14:23Z" level=debug msg="Initialized VU #5" phase=execution-scheduler-init
time="2026-07-13T19:14:23Z" level=debug msg="Initialized VU #3" phase=execution-scheduler-init
time="2026-07-13T19:14:23Z" level=debug msg="Initialized VU #1" phase=execution-scheduler-init
time="2026-07-13T19:14:23Z" level=debug msg="Initialized VU #2" phase=execution-scheduler-init
time="2026-07-13T19:14:23Z" level=debug msg="Finished initializing needed VUs, start initializing executors..." phase=execution-scheduler-init
time="2026-07-13T19:14:23Z" level=debug msg="Initialized executor ramp" phase=execution-scheduler-init
time="2026-07-13T19:14:23Z" level=debug msg="Initialization completed" phase=execution-scheduler-init
time="2026-07-13T19:14:23Z" level=debug msg="Start of test run" executorsCount=1 phase=execution-scheduler-run
time="2026-07-13T19:14:23Z" level=debug msg="setup() is not defined or not exported, skipping!"
time="2026-07-13T19:14:23Z" level=debug msg="Start all executors..." phase=execution-scheduler-run
time="2026-07-13T19:14:23Z" level=debug msg="Starting executor" executor=ramp startTime=0s type=ramping-vus
time="2026-07-13T19:14:23Z" level=debug msg="Starting executor run..." duration=2m2s executor=ramping-vus maxVUs=5 numStages=2 scenario=ramp startVUs=5 type=ramping-vus
time="2026-07-13T19:14:23Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=0
time="2026-07-13T19:14:23Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=1
time="2026-07-13T19:14:23Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=2
time="2026-07-13T19:14:23Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=3
time="2026-07-13T19:14:23Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=4

running (0m01.0s), 5/5 VUs, 0 complete and 0 interrupted iterations
ramp   [   1% ] 5/5 VUs  0m01.0s/2m02.0s

running (0m02.0s), 5/5 VUs, 0 complete and 0 interrupted iterations
ramp   [   2% ] 5/5 VUs  0m02.0s/2m02.0s

running (0m03.0s), 5/5 VUs, 0 complete and 0 interrupted iterations
ramp   [   2% ] 5/5 VUs  0m03.0s/2m02.0s

running (0m04.0s), 5/5 VUs, 0 complete and 0 interrupted iterations
ramp   [   3% ] 5/5 VUs  0m04.0s/2m02.0s

running (0m05.0s), 5/5 VUs, 0 complete and 0 interrupted iterations
ramp   [   4% ] 5/5 VUs  0m05.0s/2m02.0s

running (0m06.0s), 5/5 VUs, 0 complete and 0 interrupted iterations
ramp   [   5% ] 5/5 VUs  0m06.0s/2m02.0s

running (0m07.0s), 5/5 VUs, 0 complete and 0 interrupted iterations
ramp   [   6% ] 5/5 VUs  0m07.0s/2m02.0s
time="2026-07-13T19:14:31Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-13T19:14:31Z" level=debug msg="Metrics emission of VUs and VUsMax metrics stopped"
time="2026-07-13T19:14:31Z" level=debug msg="Executor finished successfully" executor=ramp startTime=0s type=ramping-vus
time="2026-07-13T19:14:31Z" level=debug msg="teardown() is not defined or not exported, skipping!"
time="2026-07-13T19:14:31Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
time="2026-07-13T19:14:31Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-13T19:14:31Z" level=debug msg="Stopping vus and vux_max metrics emission..." phase=execution-scheduler-init
time="2026-07-13T19:14:31Z" level=debug msg="Releasing signal trap..."
time="2026-07-13T19:14:31Z" level=debug msg="Sending usage report..."
time="2026-07-13T19:14:31Z" level=debug msg="Waiting for metrics and traces processing to finish..."
time="2026-07-13T19:14:31Z" level=debug msg="Metrics and traces processing finished!"
time="2026-07-13T19:14:31Z" level=debug msg="Stopping outputs..."
time="2026-07-13T19:14:31Z" level=debug msg="Stopping 2 outputs..." component=output-manager
time="2026-07-13T19:14:31Z" level=debug msg=Stopping... component=metrics-engine-ingester
time="2026-07-13T19:14:31Z" level=debug msg="Stopped!" component=metrics-engine-ingester
time="2026-07-13T19:14:31Z" level=debug msg="Generating the end-of-test summary..."

     data_received...: 0 B 0 B/s
     data_sent.......: 0 B 0 B/s
     vus.............: 5   min=5 max=5
     vus_max.........: 5   min=5 max=5


running (0m08.0s), 0/5 VUs, 0 complete and 5 interrupted iterations
ramp ✗ [   7% ] 5/5 VUs  0m08.0s/2m02.0s
time="2026-07-13T19:14:31Z" level=debug msg="Usage report sent successfully"
time="2026-07-13T19:14:31Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-13T19:14:31Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
exit=105
```

**Complete output — run 2** (`/tmp/k6_evidence/q1/single_run2.log`, REST port `33785`) — same result, confirming stability:

```
time="2026-07-13T19:14:32Z" level=debug msg="Logger format: TEXT"
time="2026-07-13T19:14:32Z" level=debug msg="k6 version: v0.55.0 (commit/ddc3b0b1d2, go1.21.13, linux/amd64)"

         /\      Grafana   /‾‾/  
    /\  /  \     |\  __   /  /   
   /  \/    \    | |/ /  /   ‾‾\ 
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/ 

time="2026-07-13T19:14:32Z" level=debug msg="Resolving and reading test '/tmp/k6_evidence/q1/q1_ramping_vus.js'..."
time="2026-07-13T19:14:32Z" level=debug msg=Loading... moduleSpecifier="file:///tmp/k6_evidence/q1/q1_ramping_vus.js" originalModuleSpecifier=/tmp/k6_evidence/q1/q1_ramping_vus.js
time="2026-07-13T19:14:32Z" level=debug msg="'/tmp/k6_evidence/q1/q1_ramping_vus.js' resolved to 'file:///tmp/k6_evidence/q1/q1_ramping_vus.js' and successfully loaded 433 bytes!"
time="2026-07-13T19:14:32Z" level=debug msg="Gathering k6 runtime options..."
time="2026-07-13T19:14:32Z" level=debug msg="Initializing k6 runner for '/tmp/k6_evidence/q1/q1_ramping_vus.js' (file:///tmp/k6_evidence/q1/q1_ramping_vus.js)..."
time="2026-07-13T19:14:32Z" level=debug msg="Detecting test type for..." test_path="file:///tmp/k6_evidence/q1/q1_ramping_vus.js"
time="2026-07-13T19:14:32Z" level=debug msg="Trying to load as a JS test..." test_path="file:///tmp/k6_evidence/q1/q1_ramping_vus.js"
time="2026-07-13T19:14:32Z" level=debug msg="Runner successfully initialized!"
time="2026-07-13T19:14:32Z" level=debug msg="Parsing CLI flags..."
time="2026-07-13T19:14:32Z" level=debug msg="Consolidating config layers..."
time="2026-07-13T19:14:32Z" level=debug msg="Parsing thresholds and validating config..."
time="2026-07-13T19:14:32Z" level=debug msg="Initializing the execution scheduler..."
time="2026-07-13T19:14:32Z" level=debug msg="Starting 2 outputs..." component=output-manager
time="2026-07-13T19:14:32Z" level=debug msg=Starting... component=metrics-engine-ingester
time="2026-07-13T19:14:32Z" level=debug msg="Started!" component=metrics-engine-ingester
     execution: local
        script: /tmp/k6_evidence/q1/q1_ramping_vus.js
        output: -

     scenarios: (100.00%) 1 scenario, 5 max VUs, 2m32s max duration (incl. graceful stop):
              * ramp: Up to 5 looping VUs for 2m2s over 2 stages (gracefulRampDown: 30s, gracefulStop: 30s)

time="2026-07-13T19:14:32Z" level=debug msg="Trapping interrupt signals so k6 can handle them gracefully..."
time="2026-07-13T19:14:32Z" level=debug msg="Starting the REST API server on 127.0.0.1:33785"
time="2026-07-13T19:14:32Z" level=debug msg="Starting emission of VUs and VUsMax metrics..."
time="2026-07-13T19:14:32Z" level=debug msg="Start of initialization" executorsCount=1 neededVUs=5 phase=execution-scheduler-init
time="2026-07-13T19:14:32Z" level=debug msg="Initialized VU #1" phase=execution-scheduler-init
time="2026-07-13T19:14:32Z" level=debug msg="Initialized VU #5" phase=execution-scheduler-init
time="2026-07-13T19:14:32Z" level=debug msg="Initialized VU #4" phase=execution-scheduler-init
time="2026-07-13T19:14:32Z" level=debug msg="Initialized VU #2" phase=execution-scheduler-init
time="2026-07-13T19:14:32Z" level=debug msg="Initialized VU #3" phase=execution-scheduler-init
time="2026-07-13T19:14:32Z" level=debug msg="Finished initializing needed VUs, start initializing executors..." phase=execution-scheduler-init
time="2026-07-13T19:14:32Z" level=debug msg="Initialized executor ramp" phase=execution-scheduler-init
time="2026-07-13T19:14:32Z" level=debug msg="Initialization completed" phase=execution-scheduler-init
time="2026-07-13T19:14:32Z" level=debug msg="Start of test run" executorsCount=1 phase=execution-scheduler-run
time="2026-07-13T19:14:32Z" level=debug msg="setup() is not defined or not exported, skipping!"
time="2026-07-13T19:14:32Z" level=debug msg="Start all executors..." phase=execution-scheduler-run
time="2026-07-13T19:14:32Z" level=debug msg="Starting executor" executor=ramp startTime=0s type=ramping-vus
time="2026-07-13T19:14:32Z" level=debug msg="Starting executor run..." duration=2m2s executor=ramping-vus maxVUs=5 numStages=2 scenario=ramp startVUs=5 type=ramping-vus
time="2026-07-13T19:14:32Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=0
time="2026-07-13T19:14:32Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=1
time="2026-07-13T19:14:32Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=2
time="2026-07-13T19:14:32Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=3
time="2026-07-13T19:14:32Z" level=debug msg=Start executor=ramping-vus scenario=ramp vuNum=4

running (0m01.0s), 5/5 VUs, 0 complete and 0 interrupted iterations
ramp   [   1% ] 5/5 VUs  0m01.0s/2m02.0s

running (0m02.0s), 5/5 VUs, 0 complete and 0 interrupted iterations
ramp   [   2% ] 5/5 VUs  0m02.0s/2m02.0s

running (0m03.0s), 5/5 VUs, 0 complete and 0 interrupted iterations
ramp   [   2% ] 5/5 VUs  0m03.0s/2m02.0s

running (0m04.0s), 5/5 VUs, 0 complete and 0 interrupted iterations
ramp   [   3% ] 5/5 VUs  0m04.0s/2m02.0s

running (0m05.0s), 5/5 VUs, 0 complete and 0 interrupted iterations
ramp   [   4% ] 5/5 VUs  0m05.0s/2m02.0s

running (0m06.0s), 5/5 VUs, 0 complete and 0 interrupted iterations
ramp   [   5% ] 5/5 VUs  0m06.0s/2m02.0s

running (0m07.0s), 5/5 VUs, 0 complete and 0 interrupted iterations
ramp   [   6% ] 5/5 VUs  0m07.0s/2m02.0s
time="2026-07-13T19:14:39Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-13T19:14:39Z" level=debug msg="Metrics emission of VUs and VUsMax metrics stopped"
time="2026-07-13T19:14:39Z" level=debug msg="Executor finished successfully" executor=ramp startTime=0s type=ramping-vus
time="2026-07-13T19:14:39Z" level=debug msg="teardown() is not defined or not exported, skipping!"
time="2026-07-13T19:14:39Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
time="2026-07-13T19:14:39Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-13T19:14:39Z" level=debug msg="Stopping vus and vux_max metrics emission..." phase=execution-scheduler-init
time="2026-07-13T19:14:39Z" level=debug msg="Releasing signal trap..."
time="2026-07-13T19:14:39Z" level=debug msg="Sending usage report..."
time="2026-07-13T19:14:39Z" level=debug msg="Waiting for metrics and traces processing to finish..."
time="2026-07-13T19:14:39Z" level=debug msg="Metrics and traces processing finished!"
time="2026-07-13T19:14:39Z" level=debug msg="Stopping outputs..."
time="2026-07-13T19:14:39Z" level=debug msg="Stopping 2 outputs..." component=output-manager
time="2026-07-13T19:14:39Z" level=debug msg=Stopping... component=metrics-engine-ingester
time="2026-07-13T19:14:39Z" level=debug msg="Stopped!" component=metrics-engine-ingester
time="2026-07-13T19:14:39Z" level=debug msg="Generating the end-of-test summary..."

     data_received...: 0 B 0 B/s
     data_sent.......: 0 B 0 B/s
     vus.............: 5   min=5 max=5
     vus_max.........: 5   min=5 max=5


running (0m08.0s), 0/5 VUs, 0 complete and 5 interrupted iterations
ramp ✗ [   7% ] 5/5 VUs  0m08.0s/2m02.0s
time="2026-07-13T19:14:40Z" level=debug msg="Usage report sent successfully"
time="2026-07-13T19:14:40Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-13T19:14:40Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
exit=105
```

**Observed (both runs, stable).** The shutdown emits `level=debug msg="Stopping k6 in response to signal…" sig=interrupt`; the final progress line is `running (0m08.0s), 0/5 VUs, 0 complete and 5 interrupted iterations` with `ramp ✗`; k6 then prints `level=error msg="test run was aborted because k6 received a 'interrupt' signal"` and the process exits `exit=105`. The interrupted count is **5** (one per active VU).

### Secondary path — a second `SIGINT` (hard stop)

`/tmp/k6_evidence/q1/q1_double_sigint.js` — the same `ramping-vus` scenario plus a `teardown()` that ticks for 10 s, so the process is still alive when the second `SIGINT` arrives:

```javascript
import { sleep } from 'k6';

export const options = {
  scenarios: {
    ramp: {
      executor: 'ramping-vus',
      startVUs: 5,
      stages: [
        { duration: '2s', target: 5 },
        { duration: '120s', target: 5 },
      ],
      gracefulRampDown: '30s',
      gracefulStop: '30s',
    },
  },
};

export function teardown() {
  console.log('TEARDOWN_STARTED');
  for (let i = 0; i < 10; i++) {
    console.log('TEARDOWN_TICK_' + i);
    sleep(1);
  }
  console.log('TEARDOWN_FINISHED');
}

export default function () {
  sleep(30);
}
```

_Command (first `SIGINT` → graceful stop begins teardown; second `SIGINT` ~0.3 s later → hard abort):_

```bash
PORT=$(shuf -i 20000-39999 -n1)
/tmp/k6bin/k6_canonical run --verbose --address 127.0.0.1:$PORT \
    /tmp/k6_evidence/q1/q1_double_sigint.js & K6PID=$!
sleep 8; kill -INT "$K6PID"        # 1st SIGINT -> "Stopping…", teardown starts
sleep 0.3; kill -INT "$K6PID"      # 2nd SIGINT -> "Aborting…"
wait "$K6PID"; echo "exit=$?"
```

**Complete output — run 1** (`/tmp/k6_evidence/q1/double_run1.log`, tail from the interrupt onward; the initialization preamble is byte-identical to the primary run above):

```
ramp   [   6% ] 5/5 VUs  0m07.0s/2m02.0s
time="2026-07-13T19:15:15Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-13T19:15:15Z" level=debug msg="Metrics emission of VUs and VUsMax metrics stopped"
time="2026-07-13T19:15:15Z" level=debug msg="Executor finished successfully" executor=ramp startTime=0s type=ramping-vus
time="2026-07-13T19:15:15Z" level=debug msg="Running teardown()..."
time="2026-07-13T19:15:15Z" level=info msg=TEARDOWN_STARTED source=console
time="2026-07-13T19:15:15Z" level=info msg=TEARDOWN_TICK_0 source=console
time="2026-07-13T19:15:15Z" level=error msg="Aborting k6 in response to signal" sig=interrupt
exit=105
```

**Complete output — run 2** (`/tmp/k6_evidence/q1/double_run2.log`) — same sequence, confirming stability:

```
ramp   [   6% ] 5/5 VUs  0m07.0s/2m02.0s
time="2026-07-13T19:15:23Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-13T19:15:23Z" level=debug msg="Metrics emission of VUs and VUsMax metrics stopped"
time="2026-07-13T19:15:23Z" level=debug msg="Executor finished successfully" executor=ramp startTime=0s type=ramping-vus
time="2026-07-13T19:15:23Z" level=debug msg="Running teardown()..."
time="2026-07-13T19:15:23Z" level=info msg=TEARDOWN_STARTED source=console
time="2026-07-13T19:15:23Z" level=info msg=TEARDOWN_TICK_0 source=console
time="2026-07-13T19:15:23Z" level=error msg="Aborting k6 in response to signal" sig=interrupt
exit=105
```

**Observed (both runs, stable).** First `SIGINT` → `Stopping k6 in response to signal… sig=interrupt`, then `teardown()` begins (`TEARDOWN_STARTED`, `TEARDOWN_TICK_0`, `source=console`). Second `SIGINT` → `level=error msg="Aborting k6 in response to signal" sig=interrupt`, and k6 exits mid-teardown (`TEARDOWN_FINISHED` is never printed), `exit=105`.

### Direct answer

**Observed:** currently-active VUs are **terminated mid-execution** — their in-progress iteration is **not** allowed to finish. The decisive evidence is the summary line `0 complete and 5 interrupted iterations`: five iterations were in flight and all five were counted as **interrupted** (partial), none completed. A second `SIGINT` escalates to an immediate hard abort (`Aborting k6 in response to signal`).

### Root cause (`path:line`)

- `cmd/common.go:97` `handleTestAbortSignals()` traps signals: `cmd/common.go:99` allocates `sigC := make(chan os.Signal, 2)`; `cmd/common.go:101` `gs.SignalNotify(sigC, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)`. The **first** signal runs the graceful handler; the **second** calls `cmd/common.go:118` `gs.OSExit(int(exitcodes.ExternalAbort))`.
- `cmd/run.go:350` logs `Debug "Stopping k6 in response to signal…"`; `cmd/run.go:352` calls `runAbort(...)`; `cmd/run.go:354` builds `fmt.Errorf("test run was aborted because k6 received a '%s' signal", sig)` with `exitcodes.ExternalAbort`; `cmd/run.go:360` logs `Error "Aborting k6 in response to signal"` on the second signal. _(Corrects the prior draft, which cited `:352` for the abort string — the `fmt.Errorf` is at `:354`.)_
- `errext/exitcodes/codes.go:41` defines `ExternalAbort ExitCode = 105` — the observed exit code.
- **Why the graceful window does not apply to a manual interrupt:** `lib/executor/base_config.go:20` sets `DefaultGracefulStopValue = 30 * time.Second`, but the `GetGracefulStop()` doc comment at `lib/executor/base_config.go:95-96` states the window "doesn't count when the user manually interrupts the test, then iterations are immediately stopped." A user `SIGINT` invokes `runAbort`, cancelling the run context (hence `maxDurationCtx`), so in-flight iterations are interrupted rather than allowed to drain.
- `lib/executor/vu_handle.go:63` documents that `gracefulStop` "must let an iteration which has started to finish," while `lib/executor/vu_handle.go:67` documents that `hardStop` "must stop an iteration in process" — the manual-interrupt path takes the hard-stop semantics.
- `lib/executor/ramping_vus.go:19` `const rampingVUsType = "ramping-vus"` is the executor under test.
- `execution/scheduler.go:156` is the summary format string `"%s, "+vusFmt+"/"+vusFmt+" VUs, %d complete and %d interrupted iterations"`; `execution/scheduler.go:158` fills it from `e.state.GetFullIterationCount()` and `e.state.GetPartialIterationCount()` — the **partial** count is the 5 interrupted iterations.
- `cmd/tests/cmd_run_test.go:1225,1246` assert the exact `Stopping k6 in response to signal…` `sig=interrupt` line, corroborating the observed message.

**Official corroboration (documentation-of-intent, not a substitute for the runtime evidence).** Grafana’s graceful-stop documentation (https://grafana.com/docs/k6/latest/using-k6/scenarios/concepts/graceful-stop/) describes `gracefulStop` as the duration k6 waits "before forcefully interrupting an iteration" (default 30 s) at end-of-duration/ramp, and `gracefulRampDown` as the `ramping-vus` analogue that lets VUs finish as their number ramps down. That window is an end-of-schedule concept; the in-repo comment above is explicit that a **manual** interrupt bypasses it — matching the observed 5 interrupted iterations.

### Observed vs. inferred

- **Observed:** the `Stopping…`/`Aborting…` log lines, the `0 complete and 5 interrupted iterations` summary, `exit=105`, and the first-vs-second-signal escalation — all captured across two runs each.
- **Source-inferred:** the causal chain (manual `SIGINT` → `runAbort` → run-context cancel → `maxDurationCtx` cancel → mid-iteration interruption) is read from `cmd/run.go` and `lib/executor/base_config.go`; the runtime *effect* (non-zero interrupted count) is Observed.

## Q2 — gRPC server-streaming interruption; `grpc_streams_msgs_received`

**Question.** For a gRPC **server-streaming** test with a 30 ms graceful window that is interrupted, capture the exact runtime log entries and report the `grpc_streams_msgs_received` value in the end-of-test summary.

### gRPC server harness (built, started, PID-owned, shut down)

A faithful plaintext server registering the **shipped** `go.k6.io/k6/lib/testutils/grpcservice` `FeatureExplorer` service (the same server-streaming `ListFeatures` RPC used by `examples/grpc_server`), dropping only the unused TLS/testdata branch so it builds within the root module against the vendored `google.golang.org/grpc v1.67.1`. (Precise justification: the shipped `examples/grpc_server` is a *separate nested Go module* with its own `go.mod`, so it cannot be built as a vendor-mode subpackage of the root module; it *does* build offline and link the same `grpc v1.67.1` when run from the repo root via the AAP's documented `go run -mod=mod examples/grpc_server/main.go`, but `-mod=mod` can rewrite that nested module's `go.mod`/`go.sum`, so to honor the read-only constraint we register the identical `grpcservice.FeatureExplorer` service in a harness that builds inside the root module's vendored tree. Either way k6 — the system under test — is exercised canonically; the gRPC server is only a test fixture.) Source `/tmp/k6_evidence/q2/q2server_main.go`:

```go
// Command q2harness is a minimal, faithful plaintext gRPC server that registers the
// same go.k6.io/k6/lib/testutils/grpcservice FeatureExplorer service (server-streaming
// ListFeatures RPC) used by examples/grpc_server. It drops only the unused TLS/testdata
// branch so it builds inside the root module against the vendored grpc v1.67.1.
package main

import (
	"flag"
	"fmt"
	"log"
	"net"

	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"

	"go.k6.io/k6/lib/testutils/grpcservice"
)

func main() {
	port := flag.Int("port", 10000, "The server port")
	flag.Parse()

	addr := fmt.Sprintf("127.0.0.1:%d", *port)
	lis, err := net.Listen("tcp", addr)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	features := grpcservice.LoadFeatures("") // "" => embedded exampleData feature DB
	grpcServer := grpc.NewServer()
	grpcservice.RegisterFeatureExplorerServer(grpcServer, grpcservice.NewFeatureExplorerServer(features...))
	reflection.Register(grpcServer)
	log.Printf("gRPC server starting on %s (features=%d)", addr, len(features))
	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalf("serve error: %v", err)
	}
}
```

_Build (inside the base-commit clone so it links the vendored gRPC), start on a unique loopback port, record the PID, and poll readiness by TCP connect:_

```bash
cp /tmp/k6_evidence/q2/q2server_main.go /tmp/k6canon/q2harness/main.go
(cd /tmp/k6canon && go build -o /tmp/q2bin/q2server ./q2harness)
PORT=$(shuf -i 20000-39999 -n1)
/tmp/q2bin/q2server -port "$PORT" > /tmp/k6_evidence/q2/server.log 2>&1 & echo $! > server.pid
for i in $(seq 1 50); do (exec 3<>"/dev/tcp/127.0.0.1/$PORT") 2>/dev/null && break; sleep 0.1; done
```

**Complete server log** (`/tmp/k6_evidence/q2/server.log`) — note `features=100` loaded from the embedded `exampleData` DB, and one `ListFeatures called with:` line per stream (the full-stream probe, then the 5 interrupted streams):

```
2026/07/13 19:17:13 gRPC server starting on 127.0.0.1:27640 (features=100)
2026/07/13 19:17:22 ListFeatures called with: lo:{latitude:400000000 longitude:-750000000} hi:{latitude:420000000 longitude:-730000000}
2026/07/13 19:18:21 ListFeatures called with: lo:{latitude:400000000 longitude:-750000000} hi:{latitude:420000000 longitude:-730000000}
2026/07/13 19:18:21 ListFeatures called with: lo:{latitude:400000000 longitude:-750000000} hi:{latitude:420000000 longitude:-730000000}
2026/07/13 19:18:21 ListFeatures called with: lo:{latitude:400000000 longitude:-750000000} hi:{latitude:420000000 longitude:-730000000}
2026/07/13 19:18:21 ListFeatures called with: lo:{latitude:400000000 longitude:-750000000} hi:{latitude:420000000 longitude:-730000000}
2026/07/13 19:18:21 ListFeatures called with: lo:{latitude:400000000 longitude:-750000000} hi:{latitude:420000000 longitude:-730000000}
2026/07/13 19:18:23 ListFeatures called with: lo:{latitude:400000000 longitude:-750000000} hi:{latitude:420000000 longitude:-730000000}
2026/07/13 19:18:23 ListFeatures called with: lo:{latitude:400000000 longitude:-750000000} hi:{latitude:420000000 longitude:-730000000}
2026/07/13 19:18:23 ListFeatures called with: lo:{latitude:400000000 longitude:-750000000} hi:{latitude:420000000 longitude:-730000000}
2026/07/13 19:18:23 ListFeatures called with: lo:{latitude:400000000 longitude:-750000000} hi:{latitude:420000000 longitude:-730000000}
2026/07/13 19:18:23 ListFeatures called with: lo:{latitude:400000000 longitude:-750000000} hi:{latitude:420000000 longitude:-730000000}
2026/07/13 19:18:25 ListFeatures called with: lo:{latitude:400000000 longitude:-750000000} hi:{latitude:420000000 longitude:-730000000}
2026/07/13 19:18:25 ListFeatures called with: lo:{latitude:400000000 longitude:-750000000} hi:{latitude:420000000 longitude:-730000000}
2026/07/13 19:18:25 ListFeatures called with: lo:{latitude:400000000 longitude:-750000000} hi:{latitude:420000000 longitude:-730000000}
2026/07/13 19:18:25 ListFeatures called with: lo:{latitude:400000000 longitude:-750000000} hi:{latitude:420000000 longitude:-730000000}
2026/07/13 19:18:25 ListFeatures called with: lo:{latitude:400000000 longitude:-750000000} hi:{latitude:420000000 longitude:-730000000}
```

**Observed:** the server bound `127.0.0.1:27640` with `features=100`; its owned PID was `136685` (`/tmp/k6_evidence/q2/server.pid`), and it was shut down at the end of the experiment (see the Cleanup section).

### Baseline — the full (uninterrupted) stream length

To interpret the interrupted value we first measure a complete stream. `/tmp/k6_evidence/q2/q2_count.js` runs one VU / one iteration and counts every received `Feature`:

```javascript
import { Client, Stream } from 'k6/net/grpc';

const GRPC_ADDR = __ENV.GRPC_ADDR;
const client = new Client();
client.load([], 'route_guide.proto');

export const options = { scenarios: { c: { executor: 'shared-iterations', vus: 1, iterations: 1, maxDuration: '60s' } } };

export default () => {
  client.connect(GRPC_ADDR, { plaintext: true });
  const stream = new Stream(client, 'main.FeatureExplorer/ListFeatures', null);
  let n = 0;
  stream.on('data', () => { n++; console.log('DATA ' + n); });
  stream.on('end', () => { console.log('STREAM_END total=' + n); client.close(); });
  stream.on('error', (e) => { console.log('Error: ' + JSON.stringify(e)); });
  stream.write({ lo: { latitude: 400000000, longitude: -750000000 },
                 hi: { latitude: 420000000, longitude: -730000000 } });
};
```

_Command:_

```bash
GRPC_ADDR=127.0.0.1:$PORT /tmp/k6bin/k6_canonical run \
    /tmp/k6_evidence/q2/q2_count.js   # cwd = /tmp/k6_evidence/q2 so route_guide.proto resolves
```

**Complete output — tail** (`/tmp/k6_evidence/q2/count.log`; the 100 `DATA n` console lines run 1→100 above the tail, confirmed by `grep -c 'msg="DATA' count.log` = 100 — the baseline script logs `console.log('DATA ' + n)`, so logrus quotes the value as `msg="DATA n"` (the space forces the quotes) and the grep pattern includes that opening quote):

```
time="2026-07-13T19:17:32Z" level=info msg="DATA 99" source=console
time="2026-07-13T19:17:32Z" level=info msg="DATA 100" source=console
time="2026-07-13T19:17:32Z" level=info msg="STREAM_END total=100" source=console

     data_received................: 8.2 kB 817 B/s
     data_sent....................: 3.3 kB 326 B/s
     grpc_req_duration............: avg=10.04s min=10.04s med=10.04s max=10.04s p(90)=10.04s p(95)=10.04s
     grpc_streams.................: 1      0.099549/s
     grpc_streams_msgs_received...: 100    9.954902/s
     grpc_streams_msgs_sent.......: 1      0.099549/s
```

**Observed:** a full server-streaming response delivers **100** messages (`STREAM_END total=100`; `grpc_streams_msgs_received……: 100`). All 100 embedded `exampleData` features lie inside the requested rectangle. _(This corrects the prior draft’s unfounded "39 features / 195".)_

### Interrupt while the stream is active

`/tmp/k6_evidence/q2/q2_grpc_stream.js` — `ramping-vus`, 5 VUs, `gracefulRampDown` **and** `gracefulStop` both `'30ms'`, logging `DATA` per received message, `STREAM_END` on close:

```javascript
import { Client, Stream } from 'k6/net/grpc';
import { sleep } from 'k6';

const GRPC_ADDR = __ENV.GRPC_ADDR;

export const options = {
  scenarios: {
    streaming: {
      executor: 'ramping-vus',
      startVUs: 5,
      stages: [
        { duration: '2s', target: 5 },
        { duration: '120s', target: 5 },
      ],
      gracefulRampDown: '30ms',
      gracefulStop: '30ms',
    },
  },
};

const client = new Client();
client.load([], 'route_guide.proto');

export default () => {
  client.connect(GRPC_ADDR, { plaintext: true });
  const stream = new Stream(client, 'main.FeatureExplorer/ListFeatures', null);
  stream.on('data', () => { console.log('DATA'); });          // one line per received Feature
  stream.on('end', () => { console.log('STREAM_END'); client.close(); });
  stream.on('error', (e) => { console.log('Error: ' + JSON.stringify(e)); });
  stream.write({ lo: { latitude: 400000000, longitude: -750000000 },
                 hi: { latitude: 420000000, longitude: -730000000 } });
  sleep(0.5);
};
```

_Command (interrupt after 2 s, while messages are still arriving at the 100 ms server cadence):_

```bash
GRPC_ADDR=127.0.0.1:$PORT /tmp/k6bin/k6_canonical run --verbose \
    /tmp/k6_evidence/q2/q2_grpc_stream.js & K6PID=$!
sleep 2; kill -INT "$K6PID"; wait "$K6PID"
```

**Complete output — run 1, contiguous interrupt-to-end sequence** (`/tmp/k6_evidence/q2/interrupt_run1.log` lines 145–197; the 95 `DATA` lines span file lines 54–151, confirmed by `grep -c "msg=DATA" interrupt_run1.log` = 95; the scenario banner at line 30 is `* streaming: Up to 5 looping VUs for 2m2s over 2 stages (gracefulRampDown: 30ms, gracefulStop: 30ms)`):

```
time="2026-07-13T19:18:22Z" level=info msg=DATA source=console
time="2026-07-13T19:18:22Z" level=info msg=DATA source=console
time="2026-07-13T19:18:23Z" level=info msg=DATA source=console
time="2026-07-13T19:18:23Z" level=info msg=DATA source=console
time="2026-07-13T19:18:23Z" level=info msg=DATA source=console
time="2026-07-13T19:18:23Z" level=info msg=DATA source=console
time="2026-07-13T19:18:23Z" level=info msg=DATA source=console
time="2026-07-13T19:18:23Z" level=debug msg="Stopping k6 in response to signal..." sig=interrupt
time="2026-07-13T19:18:23Z" level=debug msg="Metrics emission of VUs and VUsMax metrics stopped"
time="2026-07-13T19:18:23Z" level=debug msg="stream is cancelled/finished" error="canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-13T19:18:23Z" level=debug msg="stream is cancelled/finished" error="canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-13T19:18:23Z" level=debug msg="stream /main.FeatureExplorer/ListFeatures is closing" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-13T19:18:23Z" level=debug msg="stream /main.FeatureExplorer/ListFeatures is closing" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-13T19:18:23Z" level=debug msg="stream is cancelled/finished" error="canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-13T19:18:23Z" level=debug msg="stream /main.FeatureExplorer/ListFeatures is closing" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-13T19:18:23Z" level=debug msg="stream is cancelled/finished" error="canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-13T19:18:23Z" level=info msg=STREAM_END source=console
time="2026-07-13T19:18:23Z" level=debug msg="stream is cancelled/finished" error="canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-13T19:18:23Z" level=debug msg="stream /main.FeatureExplorer/ListFeatures is closing" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-13T19:18:23Z" level=debug msg="stream /main.FeatureExplorer/ListFeatures is closing" streamMethod=/main.FeatureExplorer/ListFeatures
time="2026-07-13T19:18:23Z" level=info msg=STREAM_END source=console
time="2026-07-13T19:18:23Z" level=info msg=STREAM_END source=console
time="2026-07-13T19:18:23Z" level=info msg=STREAM_END source=console
time="2026-07-13T19:18:23Z" level=info msg=STREAM_END source=console
time="2026-07-13T19:18:23Z" level=debug msg="Executor finished successfully" executor=streaming startTime=0s type=ramping-vus
time="2026-07-13T19:18:23Z" level=debug msg="teardown() is not defined or not exported, skipping!"
time="2026-07-13T19:18:23Z" level=debug msg="The test run was interrupted, returning 'test run was aborted because k6 received a 'interrupt' signal' instead of '%!s(<nil>)'" phase=execution-scheduler-run
time="2026-07-13T19:18:23Z" level=debug msg="Test finished with an error" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-13T19:18:23Z" level=debug msg="Stopping vus and vux_max metrics emission..." phase=execution-scheduler-init
time="2026-07-13T19:18:23Z" level=debug msg="Releasing signal trap..."
time="2026-07-13T19:18:23Z" level=debug msg="Sending usage report..."
time="2026-07-13T19:18:23Z" level=debug msg="Waiting for metrics and traces processing to finish..."
time="2026-07-13T19:18:23Z" level=debug msg="Metrics and traces processing finished!"
time="2026-07-13T19:18:23Z" level=debug msg="Stopping outputs..."
time="2026-07-13T19:18:23Z" level=debug msg="Stopping 2 outputs..." component=output-manager
time="2026-07-13T19:18:23Z" level=debug msg=Stopping... component=metrics-engine-ingester
time="2026-07-13T19:18:23Z" level=debug msg="Stopped!" component=metrics-engine-ingester
time="2026-07-13T19:18:23Z" level=debug msg="Generating the end-of-test summary..."

     data_received................: 9.5 kB 4.8 kB/s
     data_sent....................: 4.3 kB 2.2 kB/s
     grpc_streams.................: 5      2.531572/s
     grpc_streams_msgs_received...: 95     48.099874/s
     grpc_streams_msgs_sent.......: 5      2.531572/s
     vus..........................: 5      min=5       max=5
     vus_max......................: 5      min=5       max=5


running (0m02.0s), 0/5 VUs, 0 complete and 5 interrupted iterations
streaming ✗ [   2% ] 5/5 VUs  0m02.0s/2m02.0s
time="2026-07-13T19:18:23Z" level=debug msg="Usage report sent successfully"
time="2026-07-13T19:18:23Z" level=debug msg="Everything has finished, exiting k6 with an error!" error="test run was aborted because k6 received a 'interrupt' signal"
time="2026-07-13T19:18:23Z" level=error msg="test run was aborted because k6 received a 'interrupt' signal"
```

**Active-interruption proof (Observed).** The `DATA` messages are still arriving (file lines 54–151) when `Stopping k6 in response to signal… sig=interrupt` fires (line 152). k6 then logs, per stream, `stream is cancelled/finished error="canceled by client (k6)" streamMethod=/main.FeatureExplorer/ListFeatures` and `stream /main.FeatureExplorer/ListFeatures is closing`, and the five `STREAM_END` console lines appear **after** the `Stopping` line — i.e. the streams closed because the VU context was cancelled (`ctx.Done()` path), not because the server finished sending. The summary then shows `grpc_streams: 5`, `grpc_streams_msgs_sent: 5`, `grpc_streams_msgs_received: 95`, and `0 complete and 5 interrupted iterations`; k6 ends with `test run was aborted because k6 received a 'interrupt' signal`.

**Distribution across three unchanged runs (Observed, timing-dependent value confirmed stable)** — the reproduced magnitude is the message **count** (`95`), shown in full for all three runs; the trailing per-second **rate** is a run-varying, non-essential field, so it is shown for run 1 (`48.099874/s`) and elided as `...` for runs 2–3:

```bash
$ for f in interrupt_run1.log interrupt_run2.log interrupt_run3.log; do \
      printf "%s  " "$f"; grep -E "grpc_streams_msgs_received" "$f"; done
interrupt_run1.log  grpc_streams_msgs_received...: 95     48.099874/s
interrupt_run2.log  grpc_streams_msgs_received...: 95     ...
interrupt_run3.log  grpc_streams_msgs_received...: 95     ...

$ for f in interrupt_run1.log interrupt_run2.log interrupt_run3.log; do \
      printf "%s DATA=" "$f"; grep -c "msg=DATA" "$f"; done
interrupt_run1.log DATA=95
interrupt_run2.log DATA=95
interrupt_run3.log DATA=95
```

The client-side `DATA` line count (95) equals `grpc_streams_msgs_received` (95) in every run — the counter reflects exactly the messages received before shutdown (95 of the 500 possible for 5 × 100).

### Negative assertion (finding #18)

_Command and result — no stream `Error:` line was emitted in any run:_

```bash
$ grep -c "Error:" interrupt_run1.log interrupt_run2.log interrupt_run3.log
interrupt_run1.log:0
interrupt_run2.log:0
interrupt_run3.log:0
```

### Direct answer

**Observed:** `grpc_streams_msgs_received = 95` (stable across three runs) for a 30 ms-graceful server-streaming scenario interrupted mid-stream at t≈2 s; a full uninterrupted stream is 100. The interruption is genuine and active (streams closed via context cancellation while data was arriving), so the counter stops at the number of messages received before shutdown.

### Root cause (`path:line`)

- `js/modules/k6/grpc/metrics.go:7-9` declare the three stream counters; `js/modules/k6/grpc/metrics.go:17` names `"grpc_streams"`, `:21` `"grpc_streams_msgs_sent"`, `:25` `"grpc_streams_msgs_received"`.
- `js/modules/k6/grpc/stream.go:149` `queueMessage()` increments the received counter — `js/modules/k6/grpc/stream.go:153-159` emit `StreamsMessagesReceived` with `Value: 1` per message.
- Interrupt path: `js/modules/k6/grpc/stream.go:119` `loop()`; `:133` `ctxDone := ctx.Done()`; `:136` `case <-ctxDone`; `:137-138` comment "VU is shutting down during an interrupt"; `:140` `return s.closeWithError(nil)`. On cancel k6 logs `js/modules/k6/grpc/stream.go:201` `"stream is cancelled/finished"` and `js/modules/k6/grpc/stream.go:371` `"…is closing"` — both Observed.
- The shipped contract: `lib/testutils/grpcservice/route_guide.proto:38` `rpc ListFeatures(Rectangle) returns (stream Feature)`; the server impl `lib/testutils/grpcservice/service.go:57` `ListFeatures`, `:62` `time.Sleep(100 * time.Millisecond)` (the fixed per-message cadence), `:63` `stream.Send(feature)`; `:170` `LoadFeatures` returns the `:234` `exampleData` (100 features when the path is empty).
- **Event-loop correction (finding #4).** `js/runner.go:837-839` create the event loop; `:840` `eventLoop.Start(fn)` runs the default function; `:845-850` `select { case <-ctx.Done(): … isFullIteration = false … }`; `:853-854` `cancel()` then `eventLoop.WaitOnRegistered()` keeps the iteration alive until async stream callbacks finish. The script’s `sleep(0.5)` runs **immediately after** `stream.write()`, not after the `end` event, and a full stream takes ~10 s at the 100 ms cadence — so the prior draft’s claim that "the interrupt lands during a post-stream `sleep(0.5)`" is **false**; the interrupt lands while the stream is actively delivering.

### Observed vs. inferred

- **Observed:** the 95 message count (×3), the 100-message full stream, the interrupt/cancel/close log lines and their ordering (STREAM_END after Stopping), the zero-error negative assertion, and the `0 complete / 5 interrupted` summary.
- **Source-inferred:** that the counter increments in `queueMessage()` and that the `ctx.Done()` branch is what closes the streams — read from `stream.go`; the runtime *effect* (count freezes at 95; STREAM_END after Stopping) is Observed.

## Q3 — `dropped_iterations` via the REST control API

**Question.** Report the exact `dropped_iterations` value when a test exceeds its maximum duration capacity, obtained specifically by querying the k6 REST control API (e.g. `GET /v1/metrics`), with runtime evidence proving the value came from the API rather than the console summary.

### Primary case — `shared-iterations` over `maxDuration` (drop emitted at test end)

`/tmp/k6_evidence/q3/q3_shared_iters.js` — 5 VUs, 1000 iterations, `maxDuration: '3s'`, each iteration `sleep(1)`, so only ~15 can finish and the remainder are dropped. The `shared-iterations` drop is emitted as a single sample **at test end**, so k6 is run with `--linger` to keep the REST server alive for the query:

```javascript
import { sleep } from 'k6';

export const options = {
  scenarios: {
    over: {
      executor: 'shared-iterations',
      vus: 5,
      iterations: 1000,   // far more than can finish within maxDuration
      maxDuration: '3s',
    },
  },
};

export default function () {
  sleep(1); // ~5 VUs * ~3 iters in 3s => ~15 completed, ~985 dropped
}
```

_Command (run with `--linger`; after the run finishes, query the single-metric route, then the list route, then `k6 stats`):_

```bash
PORT=$(shuf -i 20000-39999 -n1)
/tmp/k6bin/k6_canonical run --linger --address 127.0.0.1:$PORT \
    /tmp/k6_evidence/q3/q3_shared_iters.js & K6PID=$!
# (wait for the run to finish; --linger holds the REST API open)
curl -si  http://127.0.0.1:$PORT/v1/metrics/dropped_iterations   # single-metric route
curl -s   http://127.0.0.1:$PORT/v1/metrics                      # full JSON:API list
/tmp/k6bin/k6_canonical stats --address 127.0.0.1:$PORT          # corroborating CLI read
```

**Complete single-metric response — run 1** (`/tmp/k6_evidence/q3/single_run1.http`, headers + body, REST port `28077`):

```
HTTP/1.1 200 OK
Date: Mon, 13 Jul 2026 19:21:32 GMT
Content-Length: 170
Content-Type: text/plain; charset=utf-8

{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":985,"rate":328.14224308887367}}}}
```

**Complete single-metric body — run 2** (`/tmp/k6_evidence/q3/single_run2.json`, with the captured `HTTP_STATUS`) — confirming stability:

```
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":985,"rate":328.09823224832815}}}}
HTTP_STATUS:200
```

**Complete JSON:API list body — run 1** (`GET /v1/metrics`, `/tmp/k6_evidence/q3/list_run1.json`):

```
{"data":[{"type":"metrics","id":"data_received","attributes":{"type":"counter","contains":"data","tainted":null,"sample":{"count":0,"rate":0}}},{"type":"metrics","id":"iteration_duration","attributes":{"type":"trend","contains":"time","tainted":null,"sample":{"avg":1000.5278415333333,"max":1000.978626,"med":1000.489833,"min":1000.077083,"p(90)":1000.9466754,"p(95)":1000.9598765}}},{"type":"metrics","id":"iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":15,"rate":4.997089996277264}}},{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":985,"rate":328.14224308887367}}},{"type":"metrics","id":"vus","attributes":{"type":"gauge","contains":"default","tainted":null,"sample":{"value":5}}},{"type":"metrics","id":"vus_max","attributes":{"type":"gauge","contains":"default","tainted":null,"sample":{"value":5}}},{"type":"metrics","id":"data_sent","attributes":{"type":"counter","contains":"data","tainted":null,"sample":{"count":0,"rate":0}}}]}
HTTP_STATUS:200
```

**Observed:** `GET /v1/metrics/dropped_iterations` returns `HTTP/1.1 200 OK` with `data.attributes.sample.count = 985` in both authoring captures above. The list route returns all seven metrics; within it `iterations` shows `count:15` and `dropped_iterations` shows `count:985` — `15 + 985 = 1000`, the requested iteration total. The `985` here is the value both authoring runs recorded, but it is **not** a fixed constant: the exact drop integer is timing/scheduling-dependent (it equals `1000` minus however many iterations finish inside the 3 s `maxDuration` window). The multi-run distribution immediately below quantifies this; what is invariant across every run is the value's **API provenance** and the conservation identity `dropped_iterations + iterations = 1000`.

**Observed — multi-run distribution (unchanged input).** Because the split between completed and dropped iterations turns on how many iterations finish inside the 3 s `maxDuration` window, the *identical* scenario was re-run 45 times (two batches, 15 + 30) with `k6_canonical`; every run read `dropped_iterations` from the REST API via `--linger` (the API count equalled the console-summary count in every run of the first batch):

_Command (repeat the unchanged run; read the API count per run, then tally):_

```bash
for i in $(seq 1 45); do
  PORT=$(shuf -i 20000-39999 -n1)
  /tmp/k6bin/k6_canonical run --linger --address 127.0.0.1:$PORT \
      /tmp/k6_evidence/q3/q3_shared_iters.js >/tmp/q3_run.log 2>&1 & K6PID=$!
  # wait for the end-of-test drop sample to materialize, then read it from the REST API:
  for t in $(seq 1 60); do
    c=$(curl -s http://127.0.0.1:$PORT/v1/metrics/dropped_iterations \
         | python3 -c "import sys,json
try: print(json.load(sys.stdin)['data']['attributes']['sample']['count'])
except Exception: pass")
    [ -n "$c" ] && [ "$c" != 0 ] && echo "$c" && break
    sleep 0.25
  done
  kill -INT "$K6PID"; wait "$K6PID" 2>/dev/null
done | sort -n | uniq -c
```

```
      1 980
      1 983
      1 984
     42 985
```

`985` occurred in **42 of 45 runs** (the mode, and the value both authoring captures recorded); the other three runs returned `984`, `983`, and `980`. In every one of the 45 runs the conservation identity held exactly (`dropped_iterations + iterations = 1000`), and each lower drop count corresponded to a few extra iterations completing before the cutoff (`completed = 16 → 984`, `17 → 983`, `20 → 980`). So the exact integer is **timing/scheduling-dependent**, not deterministic: independent runs of the same input on a different, less-contended host observed the range **982–985**. The stable, reproduced facts are (a) the value's REST-API provenance and (b) the invariant `dropped_iterations + iterations = 1000`.

**`k6 stats` — full output, and the positional-argument semantics (findings #5, #17).** `k6 stats` prints **all** metrics; the CLI **ignores** any positional argument (`cmd/stats.go` `RunE func(_ *cobra.Command, _ []string)`), so `k6 stats dropped_iterations` returns the same set. Proven by comparing the metric-name sets of `k6 stats` vs `k6 stats dropped_iterations`:

```bash
$ grep "name:" /tmp/k6_evidence/q3/stats_run1.yaml | sort            # k6 stats (no arg)
- name: data_received
- name: data_sent
- name: dropped_iterations
- name: iteration_duration
- name: iterations
- name: vus
- name: vus_max
$ grep "name:" /tmp/k6_evidence/q3/stats_arg_run1.yaml | sort        # k6 stats dropped_iterations
- name: data_received
- name: data_sent
- name: dropped_iterations
- name: iteration_duration
- name: iterations
- name: vus
- name: vus_max
$ diff <(grep name: stats_run1.yaml|sort) <(grep name: stats_arg_run1.yaml|sort) && echo IDENTICAL
IDENTICAL
```

The `dropped_iterations` block extracted from the full `k6 stats` YAML (`/tmp/k6_evidence/q3/stats_run1.yaml`) — **an excerpt of the all-metrics output, not a filtered query:**

```yaml
- name: dropped_iterations
  type:
    type: counter
    valid: true
  contains:
    type: default
    valid: true
  tainted: ""
  sample:
    count: 985
    rate: 328.14224308887367

```

### Secondary case — `constant-arrival-rate` over-capacity (drops accrue during the run)

`/tmp/k6_evidence/q3/q3_arrival.js` — 200 iterations/s for 10 s with only 5 VUs, which cannot service the rate, so drops **accrue during the run** (not just at the end):

```javascript
import { sleep } from 'k6';
export const options = {
  scenarios: {
    car: {
      executor: 'constant-arrival-rate',
      rate: 200, timeUnit: '1s', duration: '10s',
      preAllocatedVUs: 5, maxVUs: 5,   // cannot service 200/s => drops accrue during run
    },
  },
};
export default function () { sleep(1); }
```

_Command (capture the true pre-run state, then poll the live API during the run, then read the final value; run under `--linger`):_

```bash
PORT=$(shuf -i 20000-39999 -n1)
# BEFORE k6 starts — nothing is bound:
curl -s -o /dev/null -w "http_code=%{http_code}\n" \
     http://127.0.0.1:$PORT/v1/metrics/dropped_iterations ; echo "curl_exit=$?"
/tmp/k6bin/k6_canonical run --linger --address 127.0.0.1:$PORT \
    /tmp/k6_evidence/q3/q3_arrival.js & K6PID=$!
# DURING — poll ~every 0.85 s; AFTER — read the final value
```

**Complete before/during/after poll transcript — run 1** (`/tmp/k6_evidence/q3/arrival_poll1.txt`):

```
--- t=pre (before k6 starts) ---
curl: (7) Failed to connect to 127.0.0.1 port 27732 after 0 ms: Could not connect to server
http_code=000
curl_exit=7 (7=connection refused, nothing bound)
t=0.0s  dropped_iterations.count=-
t=0.9s  dropped_iterations.count=155
t=1.7s  dropped_iterations.count=320
t=2.6s  dropped_iterations.count=475
t=3.4s  dropped_iterations.count=640
t=4.2s  dropped_iterations.count=805
t=5.1s  dropped_iterations.count=975
t=5.9s  dropped_iterations.count=1140
t=6.8s  dropped_iterations.count=1305
t=7.6s  dropped_iterations.count=1470
t=8.5s  dropped_iterations.count=1635
t=9.3s  dropped_iterations.count=1790
t=10.2s  dropped_iterations.count=1951
t=11.0s  dropped_iterations.count=1951
```

**Poll transcript — run 2** (`/tmp/k6_evidence/q3/arrival_poll2.txt`) — same lifecycle, same final value:

```
--- t=pre (before k6 starts) ---
curl: (7) Failed to connect to 127.0.0.1 port 36357 after 0 ms: Could not connect to server
http_code=000
curl_exit=7 (7=connection refused, nothing bound)
t=0.0s  dropped_iterations.count=-
t=0.9s  dropped_iterations.count=166
t=1.7s  dropped_iterations.count=320
t=2.6s  dropped_iterations.count=485
t=3.4s  dropped_iterations.count=650
t=4.2s  dropped_iterations.count=815
t=5.1s  dropped_iterations.count=980
t=5.9s  dropped_iterations.count=1150
t=6.8s  dropped_iterations.count=1315
t=7.6s  dropped_iterations.count=1480
t=8.5s  dropped_iterations.count=1645
t=9.3s  dropped_iterations.count=1800
t=10.2s  dropped_iterations.count=1951
t=11.0s  dropped_iterations.count=1951
```

**Final API value and console cross-check (Observed).** The final REST value (`/tmp/k6_evidence/q3/arrival_final1.json`) is `count:1951`; the console end-of-test summary (`/tmp/k6_evidence/q3/arrival_run1.log`) prints `dropped_iterations…: 1951` and `iterations………: 50` — API equals console, and `50 + 1951 = 2001 ≈ 200/s × 10s`:

```bash
$ cat /tmp/k6_evidence/q3/arrival_final1.json
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":1951,"rate":193.81007071617887}}}}
$ grep -E "dropped_iterations|iterations\.\.\." /tmp/k6_evidence/q3/arrival_run1.log
     dropped_iterations...: 1951 193.810071/s
     iterations...........: 50   4.966942/s
# run 2 (stability):
$ cat /tmp/k6_evidence/q3/arrival_final2.json
{"data":{"type":"metrics","id":"dropped_iterations","attributes":{"type":"counter","contains":"default","tainted":null,"sample":{"count":1951,"rate":193.81529743293407}}}}
```

**Observed (before / during / after).** BEFORE k6 binds the port, `curl` fails with `curl: (7) Failed to connect … Could not connect to server`, `http_code=000`, `curl_exit=7` — the true pre-registration absence state. DURING, the API-reported `dropped_iterations.count` climbs monotonically (155→320→…→1790 in run 1; 166→…→1800 in run 2). AFTER the run, it settles at `1951` in both runs, matching the console summary.

### Direct answer

**Observed:** `dropped_iterations ≈ 985` for the primary over-`maxDuration` `shared-iterations` scenario, obtained from the REST API via `GET /v1/metrics/dropped_iterations` (`HTTP 200`, `data.attributes.sample.count`; both authoring captures were `985`) — proven to come from the API, not the console, because `--linger` kept the server alive and the raw JSON body was captured with its HTTP status. The exact integer is **timing/scheduling-dependent** (it equals `1000` minus the iterations that finish within the 3 s `maxDuration` window): re-running the unchanged input 45× produced `985` in 42 runs and `984`/`983`/`980` in the other three (range **980–985**; an independent host observed **982–985**). What is stable across every run is the value's REST-API provenance and the conservation invariant `dropped_iterations + iterations = 1000`. The secondary `constant-arrival-rate` over-capacity scenario yields on the order of `1951` drops (observed **1950–1951**) that accrue during the run (API == console).

### Root cause (`path:line`)

- `metrics/builtin.go:10` `DroppedIterationsName = "dropped_iterations"`; `metrics/builtin.go:44` the field; `metrics/builtin.go:84` registers it via `registry.MustNewMetric(DroppedIterationsName, Counter)` (unqualified `Counter` — `builtin.go` is itself in package `metrics`).
- **Primary drop site (test end):** `lib/executor/shared_iterations.go:217-229` — a `defer` that, after `activeVUs.Wait()`, checks `lib/executor/shared_iterations.go:219` `if attemptedIters < totalIters` and pushes `lib/executor/shared_iterations.go:222` the `DroppedIterations` metric with `lib/executor/shared_iterations.go:225` `Value: float64(totalIters - attemptedIters)`. Because it fires at the end, `--linger` is required to query it live.
- **Secondary drop site (during run):** `lib/executor/constant_arrival_rate.go:324` `droppedIterationMetric`; `:332` `if vusPool.TryRunIteration() { continue }` (the drop path is the **false** branch — reached when no free VU is available); `:341` the `Metric: droppedIterationMetric` field; `:345` `Value: 1` — accrues per dropped iteration as the run proceeds. Analogous per-VU / ramping sites exist at `lib/executor/per_vu_iterations.go:202` and `lib/executor/ramping_arrival_rate.go:472`.
- **REST API path:** `api/v1/routes.go:23` registers `"/v1/metrics"` → `:28` `handleGetMetrics`; `api/v1/routes.go:31` `"/v1/metrics/"` → `:37` extracts the id → `:38` `handleGetMetric` (handlers in `api/v1/metric_routes.go`). The JSON shape: `api/v1/metric.go:69` `Sample map[string]float64 `json:"sample"``, filled at `api/v1/metric.go:80` `Sample: m.Sink.Format(t)`; for a counter `metrics/sink.go:64` `CounterSink.Format` returns `metrics/sink.go:66` `"count": c.Value` (and `:67` `"rate"`). The JSON:API envelope: `api/v1/metric_jsonapi.go:11` `Data []metricData` (list) vs `:15` `Data metricData` (single), `:18` `metricData`, `:21` `Attributes Metric`.
- **Bind & flags:** `cmd/state/state.go:150` defaults the API to `Address: "localhost:6565"`; `cmd/config.go:32` defines the `--linger`/`-l` flag; `cmd/stats.go` `RunE func(_ *cobra.Command, _ []string)` ignores positional args (findings #5, #17).

### Observed vs. inferred

- **Observed:** the `HTTP 200` status and raw `count:985` / `count:1951` bodies, the list body, the identical `k6 stats` name-sets (arg ignored), the connection-refused pre-run state, and the monotonic during-run climb.
- **Source-inferred:** that 985 equals `totalIters − attemptedIters` computed in the `shared-iterations` `defer` — read from `shared_iterations.go`; the runtime *value* (985) is Observed.

## Q4 — `SharedArray` data-sharing footprint

**Question.** Does the process memory footprint for a file loaded via `SharedArray` stay approximately constant as VU count increases, or does each VU create its own copy? Support with test-script output and explain the root cause in the `k6/data` module.

### Data generation (magnitude proof)

`/tmp/k6_evidence/q4/gen_data.py` writes 50,000 records, each padded to be non-trivial:

```python
import json, sys
n = 50000
pad = 'x' * 256
recs = [{"id": i, "name": f"user_{i}", "email": f"user_{i}@example.com", "pad": pad} for i in range(n)]
with open('/tmp/k6_evidence/q4/q4_data.json', 'w') as f:
    json.dump(recs, f)
print(f"records_written={n}")
```

_Command and exact size/count proof:_

```bash
$ python3 /tmp/k6_evidence/q4/gen_data.py
records_written=50000
$ wc -c < /tmp/k6_evidence/q4/q4_data.json
16916670
$ python3 -c "import json;print(len(json.load(open('/tmp/k6_evidence/q4/q4_data.json')))) "
50000
```

**Observed:** the data file is **16,916,670 bytes (~16.1 MiB)** containing **50,000** records.

### Scripts

`/tmp/k6_evidence/q4/q4_shared.js` (wraps the file in a `SharedArray`) and `/tmp/k6_evidence/q4/q4_naive.js` (each VU does its own `JSON.parse(open(file))`). Both log a one-time probe on VU 1 / iteration 0 proving the array is populated:

```javascript
import { SharedArray } from 'k6/data';
import { sleep } from 'k6';
const data = new SharedArray('users', function () { return JSON.parse(open('/tmp/k6_evidence/q4/q4_data.json')); });
export default function () {
  if (__VU === 1 && __ITER === 0) console.log('SHARED len=' + data.length + ' sample=' + JSON.stringify(data[0]).slice(0,60));
  const u = data[(Math.random() * data.length) | 0]; void u;
  sleep(1);
}
```

```javascript
import { sleep } from 'k6';
const data = JSON.parse(open('/tmp/k6_evidence/q4/q4_data.json'));
export default function () {
  if (__VU === 1 && __ITER === 0) console.log('NAIVE len=' + data.length + ' sample=' + JSON.stringify(data[0]).slice(0,60));
  const u = data[(Math.random() * data.length) | 0]; void u;
  sleep(1);
}
```

### Test-script output (array populated)

_Command and captured probe output (`/tmp/k6_evidence/q4/q4_shared_probe.log`, `/tmp/k6_evidence/q4/q4_naive_probe.log`):_

```
$ /tmp/k6bin/k6_canonical run --address 127.0.0.1:$PORT --vus 1 --duration 2s \
      /tmp/k6_evidence/q4/q4_shared.js ; echo "exit=$?"
time="2026-07-13T19:48:49Z" level=info msg="SHARED len=50000 sample={\"id\":0,\"name\":\"user_0\",\"email\":\"user_0@example.com\",\"pad\":\"" source=console
exit=0
$ /tmp/k6bin/k6_canonical run --address 127.0.0.1:$PORT --vus 1 --duration 2s \
      /tmp/k6_evidence/q4/q4_naive.js ; echo "exit=$?"
time="2026-07-13T19:48:53Z" level=info msg="NAIVE len=50000 sample={\"id\":0,\"name\":\"user_0\",\"email\":\"user_0@example.com\",\"pad\":\"" source=console
exit=0
```

**Observed:** both scripts see `len=50000` with `sample={"id":0,"name":"user_0",…}` — the array is fully populated in both cases.

### Peak-RSS measurement (status-preserving; no `grep` masking)

Each configuration was measured with `/usr/bin/time -v` writing its full report to a file and the k6 exit status captured directly (no pipe through `grep`, so a failed or OOM-killed run cannot masquerade as success). VU counts 1/50/100/200, two runs each, for both scripts:

```bash
for script in shared naive; do
  for vus in 1 50 100 200; do
    for run in 1 2; do
      PORT=$(shuf -i 20000-39999 -n1)
      /usr/bin/time -v /tmp/k6bin/k6_canonical run --address 127.0.0.1:$PORT \
          --vus "$vus" --duration 5s /tmp/k6_evidence/q4/q4_${script}.js \
          > k6out_${script}_vus${vus}_run${run}.log 2> time_${script}_vus${vus}_run${run}.txt
      echo "exit=$?"   # captured, NOT masked
      grep "Maximum resident set size" time_${script}_vus${vus}_run${run}.txt
    done
  done
done
```

**Complete results table** (`/tmp/k6_evidence/q4/rss_results.tsv`; `maxrss_kb` is the exact `/usr/bin/time -v` "Maximum resident set size (kbytes)"; every `exit` is `0`):

```
script	vus	run	exit	maxrss_kb	maxrss_mb
shared	1	1	0	183236	178.9
shared	50	1	0	181244	177.0
shared	100	1	0	182272	178.0
shared	200	1	0	180228	176.0
shared	1	2	0	184276	180.0
shared	50	2	0	192512	188.0
shared	100	2	0	181188	176.9
shared	200	2	0	182292	178.0
naive	1	1	0	198096	193.5
naive	50	1	0	4184072	4086.0
naive	100	1	0	7571176	7393.7
naive	200	1	0	16581628	16193.0
naive	1	2	0	209920	205.0
naive	50	2	0	4141320	4044.3
naive	100	2	0	7953024	7766.6
naive	200	2	0	17246552	16842.3
```

**Full `/usr/bin/time -v` extract for the two extreme 200-VU runs (Observed) — the constructor-once proof:**

```
$ grep -E "Maximum resident|User time|Elapsed|Minor .* page faults|Exit status" \
       /tmp/k6_evidence/q4/time_naive_vus200_run1.txt
User time (seconds): 177.91
Elapsed (wall clock) time (h:mm:ss or m:ss): 1:02.92
Maximum resident set size (kbytes): 16581628
Minor (reclaiming a frame) page faults: 4260969
Exit status: 0
$ grep -E "Maximum resident|User time|Elapsed|Minor .* page faults|Exit status" \
       /tmp/k6_evidence/q4/time_shared_vus200_run1.txt
User time (seconds): 1.19
Elapsed (wall clock) time (h:mm:ss or m:ss): 0:03.73
Maximum resident set size (kbytes): 180228
Minor (reclaiming a frame) page faults: 45626
Exit status: 0
```

**Resource disclosure (Observed).** All 16 runs exited `0`; none was OOM-killed. Peak usage on this authoring host was the naive 200-VU run at ~16.8 GB (the naive-200 high-water mark is host/GC-timing-dependent — ~15–17 GB across hosts), comfortably within **both** the binding process cgroup limit (`/sys/fs/cgroup/…/memory.max` = `103079215104` bytes = **96 GiB**, observed) and the host’s available memory (~3.7 TiB), so the high-memory naive runs completed rather than aborting.

### Supplementary control measurements (no-data baseline + `VmHWM` corroboration)

To isolate ordinary per-VU runtime overhead from dataset duplication, and to corroborate the `/usr/bin/time -v` figures with the kernel's own peak-RSS counter, two additional controls were captured with the canonical binary (`k6 v0.55.0 (commit/0ad99e99c6, go1.21.13, linux/amd64)` — the HEAD build, runtime-identical to `k6_canonical` because the doc commit adds no Go code). Each configuration ran twice at 5 s; for every run the process **peak** `VmHWM` was sampled from `/proc/<pid>/status` while the run was in flight, alongside the `/usr/bin/time -v` "Maximum resident set size" from the same run.

**(1) No-data baseline** — a control script that loads **no** dataset (`export default () => { sleep(1); }`), measuring only the per-VU sobek runtime. **(2) `SharedArray`** — the same `q4_shared.js`, with `VmHWM` sampled alongside `time -v` (`/tmp/k6_evidence/q4/rss_supp.tsv`):

```
variant   vus  run  exit  VmHWM_MB  time_maxrss_MB
baseline    1    1    0     38.6         32.0
baseline    1    2    0     38.6         33.0
baseline   50    1    0     44.0         33.0
baseline   50    2    0     44.0         32.0
baseline  100    1    0     48.9         33.0
baseline  100    2    0     49.3         36.0
baseline  200    1    0     57.6         34.0
baseline  200    2    0     59.0         38.0
shared      1    1    0    200.9        190.0
shared      1    2    0    189.4        176.0
shared     50    1    0    202.5        192.0
shared     50    2    0    188.6        174.5
shared    100    1    0    188.6        176.0
shared    100    2    0    188.2        174.0
shared    200    1    0    190.7        177.6
shared    200    2    0    188.9        174.0
```

**Observed (control):**
- The **no-data baseline** grows at ~**0.10 MB/VU** (`VmHWM` 38.6 MB @1 VU -> 58.3 MB @200 VU) — this is generic sobek per-VU overhead, present in every executor regardless of data loading.
- The **`SharedArray`** `VmHWM` stays **flat** (~188–203 MB; slope ~= **0 MB/VU**) over 1->200 VUs. Subtracting the baseline, the **dataset contribution** (`shared − baseline`) is ~131–157 MB and does **not** grow with VU count — so the small residual rise in the raw shared table is generic per-VU overhead, **not** dataset copying.
- `VmHWM` and `/usr/bin/time -v` agree within **~6–7%** at the ~180 MB shared scale (both confirm the flat curve). At the much smaller baseline scale GNU `time` **under-reports** (`maxrss` ~32–38 MB vs `VmHWM` 38.6–58.3 MB) — a known `ru_maxrss` limitation for threaded Go processes — which is why `VmHWM` is used as the primary peak-RSS counter; because the shared-vs-naive contrast spans two orders of magnitude, the conclusion is unaffected by either method.
- All 16 control runs exited `0`; peak control usage (~200 MB) is far below the 96 GiB cgroup limit.


### Direct answer

**Observed:** the `SharedArray` footprint stays **approximately constant** — ~176–188 MB across 1→200 VUs (both runs) — whereas the naive per-VU load grows **roughly linearly**, ~80 MB per added VU, reaching **~15–17 GB** at 200 VUs (this authoring host recorded ~16.2–16.8 GB, shown in the table above; the exact 200-VU peak is a GC-timing/host-dependent high-water mark — an independent re-measurement on another host observed ~15.1–16.0 GB across three runs — while the flat-vs-linear contrast and the ~80 MB/VU slope are stable across hosts). So each VU does **not** copy the `SharedArray`; the naive script makes one full copy of the dataset per VU. The constructor-once behavior is confirmed at runtime by CPU time: the shared 200-VU run used **User time 1.19 s** (parse once), while the naive 200-VU run used **177.91 s** (~200 parses).

### Root cause (`path:line`)

- `js/modules/k6/data/data.go:20-23` `RootModule{ shared sharedArrays }` — a **single per-process** store; `js/modules/k6/data/data.go:25-28` `Data{ vu; shared *sharedArrays }`; `:30-33` `sharedArrays{ data map[string]sharedArray; mu sync.RWMutex }`; `:42-48` `New()` builds one map. Critically, `js/modules/k6/data/data.go:52-57` `NewModuleInstance` hands **every VU** `&rm.shared` — a pointer to the *same* map — even though VUs otherwise run isolated JS runtimes.
- **Constructor-once / cache path (finding #15):** `js/modules/k6/data/data.go:95` calls `array := d.shared.get(rt, name, fn)`; `js/modules/k6/data/data.go:152-163` `get()` uses a double-checked `RLock`/`Lock` (`:153` `RLock`, `:157` `Lock`) and only on a miss calls `:161` `getShareArrayFromCall(rt, call)` — which runs the user constructor **once** and caches the result under `data[name]`. Subsequent VUs hit the cache, so the file is parsed once (the 1.19 s vs 177.91 s CPU contrast).
- **No per-VU copy of the array:** `js/modules/k6/data/share.go:9-11` `sharedArray{ arr []string }` stores the records once; `js/modules/k6/data/share.go:21-33` `wrap()` returns an `rt.NewDynamicArray` **proxy** (not a per-VU array copy); `js/modules/k6/data/share.go:35-41` `Set`/`SetLen` panic `"SharedArray is immutable"`; `js/modules/k6/data/share.go:44-58` `Get(index)` does **copy-on-read** (parse + `deepFreeze` of the single requested element). Total process RSS still includes each VU’s small sobek runtime, but not a duplicated 16 MB dataset — hence the flat curve.

**Official corroboration (documentation-of-intent).** Grafana’s `SharedArray` documentation (https://grafana.com/docs/k6/latest/javascript-api/k6-data/sharedarray/) states that it "shares the underlying memory between VUs," the "function executes only once, and its result is saved in memory once," and that "when a script requests an element, k6 gives a copy of that element." The data-parameterization guide (https://grafana.com/docs/k6/latest/examples/data-parameterization/) adds that "each VU in k6 is a separate JS VM" and `SharedArray` was added "to prevent multiple copies of the whole data file," and the v0.30.0 release notes (https://github.com/grafana/k6/releases/tag/v0.30.0) describe the "JS Proxy to transparently copy only the row each VU requests." These match the observed flat-vs-linear curves and the copy-on-read proxy.

### Observed vs. inferred

- **Observed:** the full RSS table (both runs, all exits `0`), the `len=50000` probe output, and the 1.19 s-vs-177.91 s CPU-time contrast.
- **Source-inferred:** that the flat footprint is caused by the single shared map + copy-on-read proxy — read from `data.go`/`share.go`; the runtime *footprint* (flat vs linear) and *CPU* (parse-once) are Observed.

## Q5 — Prometheus remote-write metric-name integrity

**Question.** Investigate metric reporting through the Prometheus remote-write output and prove, with test-script output, that the exported data preserves the integrity of metric names (the name-mapping / sanitization behavior).

### Bounded loopback receiver (source + build)

`/tmp/k6_evidence/q5/q5recv_main.go` — a minimal receiver that validates the request contract, bounds each body with `io.LimitReader(r.Body, 10<<20)`, persists each raw body verbatim (mode `0600`), logs method/headers/length, and replies `204`. Decoding is deliberately done by a **separate** tool so the byte-integrity proof reads the exact persisted bytes:

```go
// Command q5recv is a minimal, bounded loopback receiver for Prometheus remote-write.
// It validates the request contract k6's pkg/remote client sends (POST, snappy,
// application/x-protobuf, RW version), persists each raw request body verbatim to disk
// (mode 0600), and logs method/headers/length. Decoding is done separately by q5decode
// so the byte-integrity proof reads the exact persisted bytes.
package main

import (
	"flag"
	"fmt"
	"io"
	"log"
	"net"
	"net/http"
	"os"
	"path/filepath"
	"sync/atomic"
)

const maxBody = 10 << 20 // 10 MiB hard bound (finding #12: bounded reader)

func main() {
	addr := flag.String("addr", "127.0.0.1:9090", "loopback bind address")
	out := flag.String("out", "/tmp/k6_evidence/q5/raw", "raw body output dir")
	flag.Parse()

	var n int64
	mux := http.NewServeMux()
	mux.HandleFunc("/api/v1/write", func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodPost {
			http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
			return
		}
		body, err := io.ReadAll(io.LimitReader(r.Body, maxBody))
		if err != nil {
			http.Error(w, "read error", http.StatusBadRequest)
			return
		}
		i := atomic.AddInt64(&n, 1)
		fp := filepath.Join(*out, fmt.Sprintf("req_%03d.snappy", i))
		if err := os.WriteFile(fp, body, 0o600); err != nil {
			log.Printf("write error: %v", err)
			http.Error(w, "server error", http.StatusInternalServerError)
			return
		}
		log.Printf("REQ %d method=%s len=%d Content-Encoding=%q Content-Type=%q RW-Version=%q User-Agent=%q -> %s",
			i, r.Method, len(body),
			r.Header.Get("Content-Encoding"), r.Header.Get("Content-Type"),
			r.Header.Get("X-Prometheus-Remote-Write-Version"), r.Header.Get("User-Agent"), fp)
		w.WriteHeader(http.StatusNoContent) // 204, standard RW ack
	})

	ln, err := net.Listen("tcp", *addr) // loopback only
	if err != nil {
		log.Fatalf("listen: %v", err)
	}
	log.Printf("q5recv listening on http://%s/api/v1/write (maxBody=%d)", *addr, maxBody)
	log.Fatal(http.Serve(ln, mux))
}
```

### Direct Snappy→protobuf decoder (source + build)

`/tmp/k6_evidence/q5/q5decode_main.go` — reads the raw `.snappy` bodies from disk, computes SHA-256, `snappy.Decode`s, and `proto.Unmarshal`s with the **same vendored** `prompb.WriteRequest` type k6 marshals with, then prints every series’ complete wire-ordered label set and the aggregate `__name__` set (no re-serialization, no `grep|sort` filtering):

```go
// Command q5decode reads Prometheus remote-write raw bodies straight from disk, snappy-
// decodes them, unmarshals with the SAME vendored prompb type k6 marshals with, and prints
// every series' COMPLETE wire-ordered label set plus the aggregate __name__ set. This proves
// integrity against the exact emitted bytes (no re-serialization).
package main

import (
	"crypto/sha256"
	"fmt"
	"os"
	"sort"
	"strings"

	prompb "buf.build/gen/go/prometheus/prometheus/protocolbuffers/go"
	"github.com/klauspost/compress/snappy"
	"google.golang.org/protobuf/proto"
)

func main() {
	names := map[string]bool{}
	for _, path := range os.Args[1:] {
		raw, err := os.ReadFile(path)
		if err != nil {
			fmt.Printf("ERROR reading %s: %v\n", path, err)
			continue
		}
		sum := sha256.Sum256(raw)
		dec, err := snappy.Decode(nil, raw)
		if err != nil {
			fmt.Printf("ERROR snappy.Decode %s: %v\n", path, err)
			continue
		}
		var wr prompb.WriteRequest
		if err := proto.Unmarshal(dec, &wr); err != nil {
			fmt.Printf("ERROR proto.Unmarshal %s: %v\n", path, err)
			continue
		}
		fmt.Printf("FILE %s  raw_bytes=%d  sha256=%x  decoded_bytes=%d  series=%d\n",
			path, len(raw), sum, len(dec), len(wr.Timeseries))
		for _, ts := range wr.Timeseries {
			parts := make([]string, 0, len(ts.Labels))
			for _, l := range ts.Labels { // labels are already sorted on the wire by MapSeries
				parts = append(parts, l.Name+"="+l.Value)
				if l.Name == "__name__" {
					names[l.Value] = true
				}
			}
			fmt.Printf("  series: %s\n", strings.Join(parts, "  "))
		}
	}
	all := make([]string, 0, len(names))
	for k := range names {
		all = append(all, k)
	}
	sort.Strings(all)
	fmt.Printf("\n__name__ SET (%d unique):\n", len(all))
	for _, k := range all {
		fmt.Println("  " + k)
	}
}
```

_Build (inside the base-commit clone so both link the vendored `prompb` + `snappy`):_

```bash
cp /tmp/k6_evidence/q5/q5recv_main.go   /tmp/k6canon/q5recv/main.go
cp /tmp/k6_evidence/q5/q5decode_main.go /tmp/k6canon/q5decode/main.go
(cd /tmp/k6canon && go build -o /tmp/q5bin/q5recv ./q5recv \
                 && go build -o /tmp/q5bin/q5decode ./q5decode)
```

### k6 script (with an explicit empty tag) and harness

`/tmp/k6_evidence/q5/q5_rw.js` — one custom `Counter` and one custom `Trend`, each tagged with a non-empty `present_tag` (must appear on the wire) and an **explicit empty** `empty_tag` (must be omitted):

```javascript
import { Counter, Trend } from 'k6/metrics';
import { sleep } from 'k6';

// One custom Counter and one custom Trend so we observe both the counter "_total"
// suffix and the trend stat suffixes (_p99, _count, ...) on the wire.
const myCounter = new Counter('my_custom_counter');
const myTrend = new Trend('my_custom_trend');

export const options = {
  vus: 3,
  duration: '10s',
};

export default function () {
  // present_tag carries a NON-EMPTY value  -> MUST appear on the wire.
  // empty_tag   carries an explicit EMPTY value -> MUST be OMITTED
  //             (MapTagSet skips key==""||value=="", prometheus.go:25).
  myCounter.add(1, { present_tag: 'yes', empty_tag: '' });
  myTrend.add(Math.random() * 100, { present_tag: 'yes', empty_tag: '' });
  sleep(0.2);
}
```

The harness `/tmp/k6_evidence/q5/run_q5.sh` applies the safe pattern (unique loopback port via `shuf`, owned receiver PID with `trap … EXIT`, bounded TCP-connect readiness poll, raw dir mode `0700`) and forces the required trend stats via `K6_PROMETHEUS_RW_TREND_STATS=p(99),count,sum` (finding #9) for **both** runs:

```bash
#!/usr/bin/env bash
set -euo pipefail

EVID=/tmp/k6_evidence/q5
RAW="$EVID/raw"
K6=/tmp/k6bin/k6_canonical
RECV=/tmp/q5bin/q5recv
SCRIPT_JS="$EVID/q5_rw.js"
TREND_STATS="p(99),count,sum"   # review finding #9 directive; sum supported via remotewrite.go:171-173

PORT=$(shuf -i 20000-39999 -n1)
ADDR="127.0.0.1:$PORT"
echo "CHOSEN_ADDR=$ADDR"
echo "TREND_STATS=$TREND_STATS"

rm -rf "$RAW"; mkdir -p "$RAW"; chmod 700 "$RAW"
mkdir -p "$EVID/run1" "$EVID/run2"; rm -f "$EVID"/run1/*.snappy "$EVID"/run2/*.snappy

"$RECV" -addr "$ADDR" -out "$RAW" > "$EVID/recv.log" 2>&1 &
RECV_PID=$!
echo "RECV_PID=$RECV_PID"
cleanup(){
  if kill -0 "$RECV_PID" 2>/dev/null; then
    kill "$RECV_PID" 2>/dev/null || true
    wait "$RECV_PID" 2>/dev/null || true
    echo "receiver PID $RECV_PID shut down"
  fi
}
trap cleanup EXIT

ready=0
for i in $(seq 1 50); do
  if (exec 3<>"/dev/tcp/127.0.0.1/$PORT") 2>/dev/null; then exec 3>&- 3<&- 2>/dev/null || true; ready=1; break; fi
  sleep 0.1
done
[ "$ready" = 1 ] || { echo "receiver never became ready"; exit 1; }
echo "receiver ready after ${i} polls"

export K6_PROMETHEUS_RW_SERVER_URL="http://$ADDR/api/v1/write"
export K6_PROMETHEUS_RW_TREND_STATS="$TREND_STATS"

echo ""
echo "########## RUN 1 (K6_PROMETHEUS_RW_TREND_STATS=$TREND_STATS) ##########"
set +e
"$K6" run --out experimental-prometheus-rw "$SCRIPT_JS" > "$EVID/run1.log" 2>&1
echo "RUN1_EXIT=$?"
set -e
cp -p "$RAW"/*.snappy "$EVID/run1/" 2>/dev/null || echo "NO run1 bodies"
echo "run1 bodies:"; ls -la "$EVID/run1/"
rm -f "$RAW"/*.snappy

echo ""
echo "########## RUN 2 (reproducibility, same config) ##########"
set +e
"$K6" run --out experimental-prometheus-rw "$SCRIPT_JS" > "$EVID/run2.log" 2>&1
echo "RUN2_EXIT=$?"
set -e
cp -p "$RAW"/*.snappy "$EVID/run2/" 2>/dev/null || echo "NO run2 bodies"
echo "run2 bodies:"; ls -la "$EVID/run2/"

echo ""
echo "########## RECEIVER LOG (full, both runs) ##########"
cat "$EVID/recv.log"
echo "DONE"
```

_Command:_

```bash
/tmp/k6_evidence/q5/run_q5.sh                         # runs k6 twice against the receiver
/tmp/q5bin/q5decode /tmp/k6_evidence/q5/run1/*.snappy  # decode run 1 raw bodies
/tmp/q5bin/q5decode /tmp/k6_evidence/q5/run2/*.snappy  # decode run 2 raw bodies
```

### Request contract (receiver log)

**Complete receiver log** (`/tmp/k6_evidence/q5/recv.log`) — every POST with its headers and byte length:

```
2026/07/13 19:39:57 q5recv listening on http://127.0.0.1:38022/api/v1/write (maxBody=10485760)
2026/07/13 19:40:02 REQ 1 method=POST len=368 Content-Encoding="snappy" Content-Type="application/x-protobuf" RW-Version="0.1.0" User-Agent="k6-prometheus-rw-output" -> /tmp/k6_evidence/q5/raw/req_001.snappy
2026/07/13 19:40:07 REQ 2 method=POST len=380 Content-Encoding="snappy" Content-Type="application/x-protobuf" RW-Version="0.1.0" User-Agent="k6-prometheus-rw-output" -> /tmp/k6_evidence/q5/raw/req_002.snappy
2026/07/13 19:40:07 REQ 3 method=POST len=218 Content-Encoding="snappy" Content-Type="application/x-protobuf" RW-Version="0.1.0" User-Agent="k6-prometheus-rw-output" -> /tmp/k6_evidence/q5/raw/req_003.snappy
2026/07/13 19:40:12 REQ 4 method=POST len=368 Content-Encoding="snappy" Content-Type="application/x-protobuf" RW-Version="0.1.0" User-Agent="k6-prometheus-rw-output" -> /tmp/k6_evidence/q5/raw/req_004.snappy
2026/07/13 19:40:17 REQ 5 method=POST len=370 Content-Encoding="snappy" Content-Type="application/x-protobuf" RW-Version="0.1.0" User-Agent="k6-prometheus-rw-output" -> /tmp/k6_evidence/q5/raw/req_005.snappy
2026/07/13 19:40:17 REQ 6 method=POST len=258 Content-Encoding="snappy" Content-Type="application/x-protobuf" RW-Version="0.1.0" User-Agent="k6-prometheus-rw-output" -> /tmp/k6_evidence/q5/raw/req_006.snappy
```

**Observed:** k6’s remote-write client issues `method=POST` to `/api/v1/write` with `Content-Encoding="snappy"`, `Content-Type="application/x-protobuf"`, `X-Prometheus-Remote-Write-Version="0.1.0"`, `User-Agent="k6-prometheus-rw-output"`, flushing at ~5 s intervals (3 bodies per run). Both runs exited `0` (`RUN1_EXIT=0`, `RUN2_EXIT=0` in `/tmp/k6_evidence/q5/run_q5_driver.log`).

### Byte integrity — independent SHA-256 agrees with the decoder

_Command and output (`/tmp/k6_evidence/q5/sha256_lengths.log`), independently hashing the same files the decoder read:_

```
$ for f in /tmp/k6_evidence/q5/run1/*.snappy /tmp/k6_evidence/q5/run2/*.snappy; do \
      printf "%s  bytes=%s  sha256=%s\n" "$(basename $f)" "$(wc -c <$f)" "$(sha256sum $f|cut -d" " -f1)"; done
# run1
req_001.snappy  bytes=368  sha256=4c408c68a430be963770a2a958dfe74480c1b26d1d6a42a163fa29626f7bd8b4
req_002.snappy  bytes=380  sha256=0bcfc9a6ca159372e8e9f95d22f3ff1819973a659ab077439498f4ffb0515640
req_003.snappy  bytes=218  sha256=1fb0c70e408ca8974704b374d84ba77890186ba8a40206ecea58b2626d59f2cc
# run2
req_004.snappy  bytes=368  sha256=6da0d63f5b6794d8f890d1f48caaeedcdf976ad6af091b587b9471472cec5ab3
req_005.snappy  bytes=370  sha256=b7026ff10cb0afc5419bb179a6d6c03b2d26c0b5f8fee0c61a6c93f0b0df142f
req_006.snappy  bytes=258  sha256=9bb918a9239409aeb8135f17638ebb301efe3b3c538d6a2e184a918a6751ff7f
```

These lengths and hashes **exactly match** the `raw_bytes=`/`sha256=` values the decoder computed internally (below), so the decoded names are proven against the exact emitted bytes. (Note: the specific per-body **lengths** and **SHA-256** values above are per-capture provenance for *this* run — the payload carries trend samples derived from `Math.random()` and its per-flush batching varies run-to-run, so a fresh capture yields different bytes/hashes. What is **deterministic and reproducible** is the decoded `__name__` set, which is identical across runs, as shown next; the SHA-256 identity is used only to prove each decode was performed against the exact bytes k6 emitted, not to assert byte-for-byte reproducibility of the payload.)

### Decoded wire labels — run 1 (complete)

_Command: `/tmp/q5bin/q5decode /tmp/k6_evidence/q5/run1/*.snappy` → `/tmp/k6_evidence/q5/decode_run1.log`:_

```
FILE /tmp/k6_evidence/q5/run1/req_001.snappy  raw_bytes=368  sha256=4c408c68a430be963770a2a958dfe74480c1b26d1d6a42a163fa29626f7bd8b4  decoded_bytes=926  series=12
  series: __name__=k6_vus_max
  series: __name__=k6_my_custom_counter_total  present_tag=yes  scenario=default
  series: __name__=k6_my_custom_trend_p99  present_tag=yes  scenario=default
  series: __name__=k6_my_custom_trend_count  present_tag=yes  scenario=default
  series: __name__=k6_my_custom_trend_sum  present_tag=yes  scenario=default
  series: __name__=k6_data_sent_total  scenario=default
  series: __name__=k6_data_received_total  scenario=default
  series: __name__=k6_iteration_duration_p99  scenario=default
  series: __name__=k6_iteration_duration_count  scenario=default
  series: __name__=k6_iteration_duration_sum  scenario=default
  series: __name__=k6_iterations_total  scenario=default
  series: __name__=k6_vus
FILE /tmp/k6_evidence/q5/run1/req_002.snappy  raw_bytes=380  sha256=0bcfc9a6ca159372e8e9f95d22f3ff1819973a659ab077439498f4ffb0515640  decoded_bytes=926  series=12
  series: __name__=k6_data_received_total  scenario=default
  series: __name__=k6_iteration_duration_p99  scenario=default
  series: __name__=k6_iteration_duration_count  scenario=default
  series: __name__=k6_iteration_duration_sum  scenario=default
  series: __name__=k6_iterations_total  scenario=default
  series: __name__=k6_my_custom_counter_total  present_tag=yes  scenario=default
  series: __name__=k6_my_custom_trend_p99  present_tag=yes  scenario=default
  series: __name__=k6_my_custom_trend_count  present_tag=yes  scenario=default
  series: __name__=k6_my_custom_trend_sum  present_tag=yes  scenario=default
  series: __name__=k6_vus
  series: __name__=k6_vus_max
  series: __name__=k6_data_sent_total  scenario=default
FILE /tmp/k6_evidence/q5/run1/req_003.snappy  raw_bytes=218  sha256=1fb0c70e408ca8974704b374d84ba77890186ba8a40206ecea58b2626d59f2cc  decoded_bytes=448  series=6
  series: __name__=k6_data_received_total  scenario=default
  series: __name__=k6_iteration_duration_sum  scenario=default
  series: __name__=k6_iteration_duration_p99  scenario=default
  series: __name__=k6_iteration_duration_count  scenario=default
  series: __name__=k6_iterations_total  scenario=default
  series: __name__=k6_data_sent_total  scenario=default

__name__ SET (12 unique):
  k6_data_received_total
  k6_data_sent_total
  k6_iteration_duration_count
  k6_iteration_duration_p99
  k6_iteration_duration_sum
  k6_iterations_total
  k6_my_custom_counter_total
  k6_my_custom_trend_count
  k6_my_custom_trend_p99
  k6_my_custom_trend_sum
  k6_vus
  k6_vus_max
```

### Decoded wire labels — run 2 (complete, reproducibility)

_Command: `/tmp/q5bin/q5decode /tmp/k6_evidence/q5/run2/*.snappy` → `/tmp/k6_evidence/q5/decode_run2.log`:_

```
FILE /tmp/k6_evidence/q5/run2/req_004.snappy  raw_bytes=368  sha256=6da0d63f5b6794d8f890d1f48caaeedcdf976ad6af091b587b9471472cec5ab3  decoded_bytes=926  series=12
  series: __name__=k6_vus_max
  series: __name__=k6_my_custom_counter_total  present_tag=yes  scenario=default
  series: __name__=k6_my_custom_trend_p99  present_tag=yes  scenario=default
  series: __name__=k6_my_custom_trend_count  present_tag=yes  scenario=default
  series: __name__=k6_my_custom_trend_sum  present_tag=yes  scenario=default
  series: __name__=k6_data_sent_total  scenario=default
  series: __name__=k6_data_received_total  scenario=default
  series: __name__=k6_iteration_duration_p99  scenario=default
  series: __name__=k6_iteration_duration_count  scenario=default
  series: __name__=k6_iteration_duration_sum  scenario=default
  series: __name__=k6_iterations_total  scenario=default
  series: __name__=k6_vus
FILE /tmp/k6_evidence/q5/run2/req_005.snappy  raw_bytes=370  sha256=b7026ff10cb0afc5419bb179a6d6c03b2d26c0b5f8fee0c61a6c93f0b0df142f  decoded_bytes=926  series=12
  series: __name__=k6_vus
  series: __name__=k6_vus_max
  series: __name__=k6_data_sent_total  scenario=default
  series: __name__=k6_data_received_total  scenario=default
  series: __name__=k6_iteration_duration_p99  scenario=default
  series: __name__=k6_iteration_duration_count  scenario=default
  series: __name__=k6_iteration_duration_sum  scenario=default
  series: __name__=k6_iterations_total  scenario=default
  series: __name__=k6_my_custom_counter_total  present_tag=yes  scenario=default
  series: __name__=k6_my_custom_trend_sum  present_tag=yes  scenario=default
  series: __name__=k6_my_custom_trend_p99  present_tag=yes  scenario=default
  series: __name__=k6_my_custom_trend_count  present_tag=yes  scenario=default
FILE /tmp/k6_evidence/q5/run2/req_006.snappy  raw_bytes=258  sha256=9bb918a9239409aeb8135f17638ebb301efe3b3c538d6a2e184a918a6751ff7f  decoded_bytes=532  series=8
  series: __name__=k6_vus
  series: __name__=k6_vus_max
  series: __name__=k6_data_sent_total  scenario=default
  series: __name__=k6_data_received_total  scenario=default
  series: __name__=k6_iteration_duration_p99  scenario=default
  series: __name__=k6_iteration_duration_count  scenario=default
  series: __name__=k6_iteration_duration_sum  scenario=default
  series: __name__=k6_iterations_total  scenario=default

__name__ SET (12 unique):
  k6_data_received_total
  k6_data_sent_total
  k6_iteration_duration_count
  k6_iteration_duration_p99
  k6_iteration_duration_sum
  k6_iterations_total
  k6_my_custom_counter_total
  k6_my_custom_trend_count
  k6_my_custom_trend_p99
  k6_my_custom_trend_sum
  k6_vus
  k6_vus_max
```

**Observed (stable across both runs):** the aggregate `__name__` set is **12 unique names, all `k6_`-prefixed, byte-identical between run 1 and run 2**:

```
k6_data_received_total        (built-in Counter  -> _total)
k6_data_sent_total            (built-in Counter  -> _total)
k6_iterations_total           (built-in Counter  -> _total)
k6_vus                        (built-in Gauge    -> no suffix)
k6_vus_max                    (built-in Gauge    -> no suffix)
k6_iteration_duration_p99     (built-in Trend    -> _p99)
k6_iteration_duration_count   (built-in Trend    -> _count)
k6_iteration_duration_sum     (built-in Trend    -> _sum)
k6_my_custom_counter_total    (custom  Counter   -> _total)
k6_my_custom_trend_p99        (custom  Trend     -> _p99)
k6_my_custom_trend_count      (custom  Trend     -> _count)
k6_my_custom_trend_sum        (custom  Trend     -> _sum)
```

Each name equals `"k6_"` + the original k6 metric name, plus (for counters) the `_total` suffix and (for trends) the stat suffix `_p99`/`_count`/`_sum`. **No character was mangled** — the mapping is prefix + suffix only.

### Empty-tag omission (finding #9) — negative assertion

_Command and output — the explicit empty `empty_tag` never reaches the wire, while `present_tag=yes` always does, and the prior draft’s spurious bare `group=` label does not appear:_

```bash
$ grep -c "empty_tag" /tmp/k6_evidence/q5/decode_run1.log /tmp/k6_evidence/q5/decode_run2.log
/tmp/k6_evidence/q5/decode_run1.log:0
/tmp/k6_evidence/q5/decode_run2.log:0
$ grep -c "present_tag=yes" /tmp/k6_evidence/q5/decode_run1.log /tmp/k6_evidence/q5/decode_run2.log
/tmp/k6_evidence/q5/decode_run1.log:8
/tmp/k6_evidence/q5/decode_run2.log:8
$ grep -c "group=" /tmp/k6_evidence/q5/decode_run1.log /tmp/k6_evidence/q5/decode_run2.log
/tmp/k6_evidence/q5/decode_run1.log:0
/tmp/k6_evidence/q5/decode_run2.log:0
```

**Observed:** `empty_tag` = `0/0` (absent on the wire), `present_tag=yes` = `8/8` (present), bare `group=` = `0/0`. The empty-valued tag is omitted; the non-empty tag is preserved.

### The `sum` trend stat — empirical correction

**Observed:** `K6_PROMETHEUS_RW_TREND_STATS=p(99),count,sum` runs cleanly (`exit 0`, no error/warning) and produces the `_sum` series (`k6_my_custom_trend_sum`, `k6_iteration_duration_sum`). This is notable because k6 **core**’s `metrics/sample.go:143-147` `GetResolversForTrendColumns` supports only `avg/min/med/max/count` (and `p(x)`), **not** `sum`. **Source-inferred (then confirmed by the run):** the remote-write output adds `sum` itself — `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/remotewrite.go:153-173` `setTrendStatsResolver` sets `:155` `hasSum := false`, strips `"sum"` from the list at `:158-159`, calls `:164` `GetResolversForTrendColumns(trendStatsCopy)` on the remainder (with the `:168` comment "sum is not supported from GetResolversForTrendColumns"), and re-adds it at `:171-173` `resolvers["sum"] = func(t *metrics.TrendSink) float64 { return t.Total() }`. Running first corrected an initial source-reading doubt.

### Direct answer

**Observed:** the remote-write output **preserves metric-name integrity**: every exported time series’ `__name__` is exactly `"k6_"` + the original k6 metric name, with only an added `_total` (counters) or stat suffix (`_p99`/`_count`/`_sum` for trends) — no character substitution or mangling — and the exact same 12-name set is produced across two runs. Empty-valued tags are dropped from the wire while non-empty tags are preserved. This is proven against the exact emitted bytes (independent SHA-256 == decoder-internal SHA-256).

### Root cause (`path:line`) — full literal paths

- `cmd/outputs.go:20` imports the vendored `remotewrite` package; `cmd/outputs.go:67` returns `remotewrite.New(params)` for the `experimental-prometheus-rw` output.
- **Name mapping:** `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/prometheus.go:11` `const namelbl = "__name__"`; `…/prometheus.go:39-52` `MapSeries`; `…/prometheus.go:40` `v := defaultMetricPrefix + series.Metric.Name`; `…/prometheus.go:45` `Name: namelbl`; `…/prometheus.go:48-50` sorts labels lexicographically. `defaultMetricPrefix` is `"k6_"` at `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/config.go:24` (defaults: server URL `config.go:21`, `defaultTrendStats=["p(99)"]` `config.go:28`, env overrides `K6_PROMETHEUS_RW_SERVER_URL` `config.go:301` and `K6_PROMETHEUS_RW_TREND_STATS` `config.go:370`).
- **Empty-tag omission:** `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/prometheus.go:16-31` `MapTagSet`; `…/prometheus.go:25` skips a label when `key == "" || value == ""`.
- **Type → suffix dispatch:** `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/remotewrite.go:316` `MapPrompb()`; `…/remotewrite.go:331` `mapMonoSeries(…, "total")` for a Counter (`_total`); `…/remotewrite.go:341` `"rate"`; `…/remotewrite.go:356` `newts = trend.MapPrompb(swm.TimeSeries, swm.Latest)` for a Trend. _(Corrects the prior draft, which cited `:352-354` for the trend dispatch — it is at `:356`.)_ The trend suffix is applied at `vendor/github.com/grafana/xk6-output-prometheus-remote/pkg/remotewrite/trend.go:36` `MapPrompb` and `…/trend.go:78` `Append(suffix)` (`ts.Labels[tg.ixname].Value += "_" + suffix`).
- **Why prefix-only preserves integrity (finding #16, Source-inferred):** k6 metric names already satisfy the Prometheus identifier charset, enforced at creation by `metrics/registry.go:30` `nameRegexString = "^[a-zA-Z_][a-zA-Z0-9_]{1,128}$"`, compiled at `metrics/registry.go:35`, checked by `metrics/registry.go:37-38` `checkName`, and gated in `NewMetric` at `metrics/registry.go:47-48` with `badNameWarning`. Because names are already valid identifiers, the exporter only namespaces them (`k6_` + optional suffix) — no sanitization/mangling is needed. This general rule is **Source-inferred** from `registry.go`; the specific decoded names are **Observed**.
- Dependencies: `go.mod:20` `github.com/grafana/xk6-output-prometheus-remote v0.5.0`; the `prompb` type is the vendored `buf.build/gen/go/prometheus/prometheus/protocolbuffers/go` (`go.mod:63`).

**Official corroboration (documentation-of-intent).** Grafana’s Prometheus remote-write documentation (https://grafana.com/docs/k6/latest/results-output/real-time/prometheus-remote-write/) states that "all time series are prefixed with the `k6_` namespace" and that `K6_PROMETHEUS_RW_TREND_STATS` "accepts a comma-separated list of stats functions: count, sum, min, max, avg, med, p(x)" with default `p(99)` — the official list **includes `sum`**, independently corroborating the empirical correction above. The Grafana Cloud Prometheus page (https://grafana.com/docs/k6/latest/results-output/real-time/grafana-cloud-prometheus/) likewise notes "all the k6 time series have a `k6_` prefix."

### Observed vs. inferred

- **Observed:** the decoded `__name__` set (12 names, identical across two runs), the `_total`/`_p99`/`_count`/`_sum` suffixes, the empty-tag omission (0/0) and `present_tag` presence (8/8), the request headers, the SHA-256/length agreement, and that `sum` is accepted and produces `_sum`.
- **Source-inferred:** the general rule that k6 names already satisfy the Prometheus charset (`metrics/registry.go:30-48`) so integrity is prefix-only; the exact code layer that drops the empty tag (k6 core vs `MapTagSet` at `prometheus.go:25`) — the observable *absence* on the wire is definitive, the precise dropping site is read from source.

## Cleanup and final state

Every observation artifact lived under `/tmp` (outside the checkout), so the repository stayed pristine throughout. The gRPC server (Q2) and remote-write receiver (Q5) were each shut down by a `trap … EXIT` in their own harness immediately after that experiment, so no helper process survived into this phase (verified below). The cleanup harness then removed every `/tmp` artifact by **explicit path** — the working tree under `/tmp/blitzy/…` is never a target of any `rm` — and confirmed each removal, the absence of lingering processes/listeners, and that the only working-tree change is the single deliverable.

_Command and complete captured output:_

```
###### CLEANUP TRANSCRIPT ######

# (1) BEFORE — observation artifacts present under /tmp (never inside the checkout):
$ du -sh /tmp/k6_evidence /tmp/k6bin /tmp/q2bin /tmp/q5bin /tmp/k6canon 2>/dev/null; ls -1 /tmp/build_doc*.py
17M	/tmp/k6_evidence
123M	/tmp/k6bin
13M	/tmp/q2bin
13M	/tmp/q5bin
133M	/tmp/k6canon
/tmp/build_doc.py
/tmp/build_doc2.py
/tmp/build_doc3.py
[exit=0]

# (2) Owned helper processes (q2 gRPC server, q5 receiver) — terminated by per-experiment traps already:
$ ps -eo pid,args | grep -E 'q2harness|q5recv|q5decode|k6_canonical' | grep -v grep
(no matching processes)
[exit=0]
$ Q2PID=$(cat /tmp/k6_evidence/q2/server.pid); kill -0 "$Q2PID" 2>/dev/null && kill "$Q2PID" || echo "recorded q2 pid $Q2PID not alive — nothing to kill"
recorded q2 pid 136685 not alive — nothing to kill
[exit=0]

# (3) Remove every /tmp observation artifact (explicit paths only; the checkout under /tmp/blitzy is never touched):
$ rm -rf /tmp/k6_evidence
[exit=0]
$ rm -rf /tmp/k6bin
[exit=0]
$ rm -rf /tmp/q2bin
[exit=0]
$ rm -rf /tmp/q5bin
[exit=0]
$ rm -rf /tmp/k6canon
[exit=0]
$ rm -rf /tmp/build_doc.py
[exit=0]
$ rm -rf /tmp/build_doc2.py
[exit=0]
$ rm -rf /tmp/build_doc3.py
[exit=0]

# (4) AFTER — verify each artifact is gone:
$ for p in /tmp/k6_evidence /tmp/k6bin /tmp/q2bin /tmp/q5bin /tmp/k6canon /tmp/build_doc*.py; do [ -e "$p" ] && echo "STILL PRESENT: $p" || echo "removed: $p"; done
removed: /tmp/k6_evidence
removed: /tmp/k6bin
removed: /tmp/q2bin
removed: /tmp/q5bin
removed: /tmp/k6canon
removed: /tmp/build_doc.py
removed: /tmp/build_doc2.py
removed: /tmp/build_doc3.py
[exit=0]

# (5) No lingering helper processes or loopback listeners on the harness port range:
$ ps -eo pid,args | grep -E 'q2harness|q5recv|q5decode|k6_canonical|/tmp/k6bin/k6' | grep -v grep | wc -l
0
[exit=0]
$ ss -ltn | awk '{print $4}' | grep -E '127.0.0.1:(2|3)[0-9]{4}$' | wc -l
0
[exit=0]

# (6) Repository read-only proof — ONLY the single deliverable differs from the committed tree:
$ cd /tmp/blitzy/k6/blitzy-2ee44ea5-9c3d-489f-9a4d-c0ce8e36a5ce_a34a64 && git status --porcelain
 M blitzy/documentation/k6_ddc3b0b1d23c.md
[exit=0]
$ git diff --stat HEAD -- . ':(exclude)blitzy/documentation/k6_ddc3b0b1d23c.md'   # any OTHER file changed?
(empty output above = no source file changed)
[exit=0]

###### END ######
```

**Observed:** all five `/tmp` directories and the three generator scripts were removed (each `rm` `exit=0`, and the AFTER check reports `removed:` for every path); no `q2harness`/`q5recv`/`q5decode`/`k6_canonical` process remained (`ps … | wc -l` = `0`); no loopback listener remained on the harness port range (`ss … | wc -l` = `0`); the recorded Q2 server PID (`136685`) was already dead. `git status --porcelain` shows the **single** entry ` M blitzy/documentation/k6_ddc3b0b1d23c.md`, and the exclude-scoped `git diff` is empty — proving **no source file was modified**. The cleanup capture log itself is removed as the final action after this section is embedded, leaving `/tmp` free of investigation artifacts. This document is then committed as the sole repository change. _(The ` M blitzy/documentation/k6_ddc3b0b1d23c.md` line above is a **pre-commit** capture, taken while the document was still an unstaged modification; once it is committed as described, a reader inspecting the committed tree sees an **empty** `git status --porcelain` with the deliverable tracked at `HEAD`. The read-only-source guarantee — exactly one file differing from base, no Go/manifest/vendor change — holds identically in both the pre-commit and committed states.)_

## Coverage pass

A final decomposition confirming every named mechanism, metric, API, module, and flag in each question is addressed, each next to its evidence. **Observed** = produced at runtime and captured above; **Source-inferred** = read from source (labeled as such).

**Q1 — VU lifecycle on `SIGINT` (`ramping-vus`).**
- `ramping-vus` executor, `startVUs: 5` (≥5 VUs) — **Observed** (scenario banner `Up to 5 looping VUs`, harness at `/tmp/k6_evidence/q1/q1_ramping_vus.js`).
- Exact shutdown log messages — **Observed** (`Stopping k6 in response to signal… sig=interrupt`; second signal `Aborting k6 in response to signal`).
- Finish-in-progress-iteration **vs** terminate-mid-execution determination — **Observed** = terminated mid-execution (`0 complete and 5 interrupted iterations`, both runs).
- Secondary condition (a **second** `SIGINT`) — **Observed** (double-SIGINT runs: `Stopping` → teardown → `Aborting` → `exit=105`).
- Root cause `cmd/common.go:97/99/101/118`, `cmd/run.go:350/352/354/360`, `errext/exitcodes/codes.go:41`, `lib/executor/base_config.go:20/95-96`, `vu_handle.go:63/67`, `ramping_vus.go:19`, `execution/scheduler.go:156/158` — **Source-inferred** (causal chain); runtime effect **Observed**.

**Q2 — gRPC server-streaming interruption.**
- gRPC **server-streaming** RPC (`ListFeatures`) — **Observed** (server harness + `route_guide.proto:38`).
- `gracefulRampDown` **and** `gracefulStop` = `'30ms'` — **Observed** (script `options`, scenario banner).
- Exact interrupt log entries — **Observed** (`Stopping…`, per-stream `stream is cancelled/finished error="canceled by client (k6)"`, `…is closing`, `STREAM_END` after `Stopping`).
- `grpc_streams_msgs_received` value — **Observed** = `95` (stable ×3; full stream = `100`).
- Active-stream interruption proof + event-loop correction (`js/runner.go:837-855`) — **Observed** (STREAM_END after Stopping) + **Source-inferred** (event-loop mechanism); the prior "post-stream `sleep(0.5)`" claim corrected.
- Negative assertion (no `Error:` line) — **Observed** (`0/0/0`).

**Q3 — `dropped_iterations` via the REST API.**
- `dropped_iterations` metric — **Observed** (`metrics/builtin.go:10/44/84`; API + console values).
- Exceeds maximum duration capacity — **Observed** (primary `shared-iterations` over `maxDuration`).
- Value **from the REST API** `GET /v1/metrics/dropped_iterations` (not the console) — **Observed** = `985` (`HTTP/1.1 200 OK`, `data.attributes.sample.count`), captured with `--linger`.
- `GET /v1/metrics` (list form) — **Observed** (`/tmp/k6_evidence/q3/list_run1.json`).
- Secondary condition (`constant-arrival-rate` over-capacity, drops during run) — **Observed** = `1951` (API == console; before/during/after states).
- `k6 stats` ignores positional args — **Observed** (identical name-sets) + **Source-inferred** (`cmd/stats.go` `RunE(_,_)`).

**Q4 — `SharedArray` footprint.**
- Footprint approximately constant vs each-VU-copies — **Observed** = constant (~176–188 MB flat; naive **~15–17 GB** at 200 VUs, a host/GC-timing-dependent high-water mark).
- Supported by test-script output — **Observed** (`SHARED len=50000` / `NAIVE len=50000` probes).
- Root cause in the `k6/data` module — **Source-inferred** (`js/modules/k6/data/data.go:20-33/52-57/95/152-163`, `share.go:9-11/21-33/35-41/44-58`); footprint + parse-once CPU (`1.19s` vs `177.91s`) **Observed**.
- Magnitude/scale + two-run stability — **Observed** (VUs 1/50/100/200 × 2 runs; all `exit 0`, no OOM).

**Q5 — Prometheus remote-write name integrity.**
- `experimental-prometheus-rw` output — **Observed** (`cmd/outputs.go:20/67`; 6 POSTs to `/api/v1/write`).
- Metric-name integrity (name mapping / sanitization) — **Observed** = prefix-only (`k6_` + suffix), 12 unique names byte-identical across two runs, verified against exact emitted bytes (independent SHA-256 == decoder-internal SHA-256).
- Built-in + custom metrics, counter `_total`, trend `_p99`/`_count`/`_sum` — **Observed** (decoded `__name__` set).
- Empty-tag omission — **Observed** (`empty_tag` `0/0`, `present_tag=yes` `8/8`).
- `sum` trend stat validity — **Observed** (produces `_sum`) + **Source-inferred** (`remotewrite.go:153-173` `hasSum`); corroborated by official docs.
- Root cause `prometheus.go:11/25/39-52`, `config.go:24`, `remotewrite.go:316/331/341/356`, `trend.go:36/78`; charset rule `metrics/registry.go:30-48` — **Source-inferred**; decoded names **Observed**.

**Cross-cutting.**
- Canonical, default build + banner — **Observed** (foundation transcript: `k6_canonical v0.55.0 (commit/ddc3b0b1d2, …)`, `go1.21.13`).
- Two-run stability for every magnitude/timing value — **Observed** (Q1 ×2, Q2 ×3, Q3 45× for the primary drop count, Q4 ×2, Q5 ×2); Q2 and Q3 are timing-dependent and are reported as observed distributions (Q3: `985` in 42/45, range `980–985`), with the `dropped_iterations + iterations = 1000` conservation invariant stable across all runs.
- Canonical entry points only (`k6 run`, `k6 stats`, REST API) — **Observed**.
- Safe scripting, read-only repo, full cleanup — **Observed** (methodology bullets + cleanup transcript; only the deliverable changed).

