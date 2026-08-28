# RTL Verification CI Architecture — Design Alternatives

Reference baseline: the GitLab CI you shared (stages: `setup → lint → test → report → cleanup`, `.base_job` anchor pattern, `extends`, `needs: artifacts: true`, `rules` gating on `CI_PIPELINE_SOURCE` / target branch). That skeleton is used as the starting point below — its structure is kept, its gaps (no formal stage, no coverage stage, one flat test level, no explicit inter-stage contract) are what the alternatives address.

---

## 1. Target Pipeline Shape

```
 setup ─┬─▶ lint ─┬─▶ simulation (smoke) ─┬─▶ coverage/report ─▶ cleanup
        │         │                       │
        │         └─▶ formal (lint-level) │
        │                                 ├─▶ simulation (regression-short)
        │                                 └─▶ simulation (regression-long)
        └─▶ formal (full)  ────────────────────────▶ (feeds coverage/report too)
```

Verification levels, orthogonal to stages:

|Level|Scope|Seeds|Trigger|Gate|
|---|---|---|---|---|
|**smoke**|subset tagged `smoke`, fast sanity|1|every push / every MR|blocking|
|**regression-short**|broad functional subset|3–5|MR → protected branch (develop/main), nightly schedule|blocking on target branches|
|**regression-long / full**|entire test plan|20–50+|weekly schedule, release tag|non-blocking to MR, gates release|
|**formal-lint**|structural/CDC proofs, fast|n/a|every push/MR|blocking|
|**formal-full**|full property set|n/a|nightly/weekly schedule|non-blocking, tracked|
|**coverage**|merged functional + formal coverage|n/a|after regression-short/long|reporting only|

---

## 2. Cross-Cutting Choice: How Jobs Stay Reusable Across Repos

This decision affects every stage below, so it's resolved once.

**Alternative A — `extends` + hidden job templates (current pattern, kept and generalized)** A shared `ci-templates` repo holds `.gitlab-ci-templates.yml` with `.setup_template`, `.lint_template`, `.sim_template`, `.formal_template`, `.report_template`. Each product repo does `include: - project: 'verif/ci-templates' ref: v3 file: '.../templates.yml'` and defines only `variables:` overrides + `extends:`.

- ✅ Minimal change from what you already have; simplest mental model.
- ✅ No new tooling.
- ❌ No input validation — a repo can pass a garbage `TEST_SUITE` and only fail at runtime.
- ❌ Versioning is just a git ref; no semantic contract.

**Alternative B — GitLab CI/CD Components (Catalog)** Package each stage as a versioned component with `spec: inputs:` (typed, with defaults and `options:`), published to a component project. Repos consume via `include: - component: $CI_SERVER_FQDN/verif/ci-components/sim-job@1.4.0 inputs: {level: smoke}`.

- ✅ Typed/validated inputs, semantic versioning, discoverable in the Catalog UI, easiest to document per-team.
- ✅ Cleanest separation between "what a stage needs" and "how a repo wires it up."
- ❌ Requires GitLab 16.x+ features and buy-in on the component-repo workflow; steeper initial setup.

**Alternative C — Dynamically generated child pipelines** `setup` stage runs a generator script that reads a repo-local `verif_plan.yaml` (test lists, per-level seed counts, DUT configs) and emits a child pipeline YAML, triggered via `trigger: include: artifact: generated-pipeline.yml`.

- ✅ Maximum flexibility — job count/matrix is data-driven, ideal for regression-long with hundreds of tests.
- ✅ Keeps the static parent pipeline tiny and generic.
- ❌ Debuggability drops (pipeline structure isn't visible until it runs); needs a well-tested generator script as its own maintained artifact.

**Recommendation:** B for the templating/versioning backbone, with C layered in specifically for `regression-long` and `formal-full`, where job counts are large and data-driven. A is fine as a bridge/interim step if Catalog components aren't available yet.

---

## 3. Per-Stage Alternatives

### 3.1 Setup

||Alt 1 — venv + pip per run (current)|Alt 2 — prebuilt verification image|Alt 3 — cached layer via lockfile hash|
|---|---|---|---|
|**What**|Clone deps, `pip install` each pipeline run|Bake `cocotb`, EDA tool wrappers, python deps into a versioned Docker image; `setup` only checks tool/license connectivity|`setup` computes hash of `requirements.txt` + dependency SHAs; restores from `cache:` if hit, else builds and pushes cache|
|**Inputs**|`$REPO_DEPENDENCIES`, `$TEMP_ETH_DEPENDENCY`|image tag pinned in `.gitlab-ci.yml`|lockfile / dependency manifest|
|**Outputs**|`.venv/` artifact (as today)|none needed downstream — image _is_ the environment|`.venv/` from cache, same artifact contract as Alt 1|
|**Trade-off**|Simple, but every pipeline pays install cost and drifts silently if a dependency repo force-pushes its branch|Fastest, fully reproducible; requires an image-build/release pipeline of its own|Good middle ground; cache invalidation bugs are the main risk|

Recommendation: Alt 2 for long-lived stable toolchains (EDA tools, cocotb core), Alt 3 for fast-moving in-repo Python packages layered on top of the image.

### 3.2 Lint

||Alt 1 — single monolithic job (current)|Alt 2 — parallel matrix by check type|Alt 3 — incremental (diff-only) lint|
|---|---|---|---|
|**What**|One `lint-job` runs everything sequentially|`parallel: matrix:` over `{syntax, style, cdc-structural, naming}` — one job per check|Lint only files changed vs. target branch (`git diff --name-only origin/$TARGET...HEAD`)|
|**Inputs**|full RTL/testbench tree|same, split by check|changed-file list from `git diff`|
|**Outputs**|single pass/fail|per-check JUnit report, all under `needs:` for the pipeline to require all|pass/fail scoped to touched files only|
|**Trade-off**|Easiest to read, slowest signal (one failure hides the rest until fixed)|Faster wall-clock feedback, clearer failure attribution, more job overhead|Fastest MR feedback loop, but needs a periodic full-lint job (nightly) to catch stale violations elsewhere in the tree|

Recommendation: Alt 2 as the MR-blocking gate, with a scheduled full-tree Alt-1-style sweep weekly to catch anything Alt-3-style incremental lint would miss.

### 3.3 Simulation (smoke / regression-short / regression-long)

||Alt 1 — single parametrized template (current, extended)|Alt 2 — `parallel:matrix` per level|Alt 3 — generated child pipeline per level|
|---|---|---|---|
|**What**|One `.sim_template`, level selected via `TEST_SUITE`/`SEED` variables and `rules:` (as today)|`parallel: matrix: [{TEST_NAME: t1, SEED: 1}, {TEST_NAME: t1, SEED: 2}, ...]` expanding N jobs from a fixed list in `.gitlab-ci.yml`|`setup`-stage generator reads `verif_plan.yaml` (per-level test list + seed counts) and emits/triggers a child pipeline with one job per test×seed|
|**Inputs**|`TEST_SUITE`, `SEED`, `DUT_CFG` vars|static matrix list committed to CI file|`verif_plan.yaml` (test names, seed counts per level, DUT configs)|
|**Outputs**|`logs/`, `waveforms/`, `reports/junit.xml` (as today)|same, per matrix job, one JUnit file each|same, one JUnit + coverage-db per job, uniform naming for downstream merge|
|**Trade-off**|Simplest, but regression-long with 100s of tests becomes one huge serial job or a hand-maintained matrix|Good parallelism, but the matrix list has to be edited in the CI YAML whenever the test plan changes — couples test-plan changes to pipeline changes|Test-plan changes never touch the pipeline YAML; scales cleanly to hundreds of jobs; best fit for regression-long|

Recommendation: Alt 2 for smoke and regression-short (test lists change rarely, matrix stays small and reviewable in an MR diff). Alt 3 for regression-long/full, where the test plan is large and owned by verification engineers independently of CI maintainers.

**Level definitions (concrete):**

- `smoke`: tests tagged `smoke` in `verif_plan.yaml`, `SEED_COUNT=1`, runs on every push/MR, target runtime budget < 15 min total, **blocking**.
- `regression-short`: broader functional tag set, `SEED_COUNT=3–5`, runs on MR into `develop`/`main`/`master` and nightly schedule, **blocking** on those target branches.
- `regression-long`: full test plan, `SEED_COUNT=20–50`, runs on weekly schedule or release tag, **non-blocking** for MRs but required to pass before a release is cut.

### 3.4 Formal

Not present in the reference pipeline — new stage, parallel to simulation.

||Alt 1 — dedicated formal stage, mirrors sim template|Alt 2 — formal split into lint-level (fast) + full (scheduled)|Alt 3 — per-property matrix|
|---|---|---|---|
|**What**|`.formal_template` analogous to `.sim_template`; one job per module, runs formal tool (e.g. JasperGold/VC Formal) against RTL + constraints|Cheap structural/CDC/X-prop checks run in the `lint` stage (fast, blocking); full property proofs run as their own stage on schedule|`parallel: matrix:` over property groups, one job per property set, aggregated afterward|
|**Inputs**|RTL, formal constraints/SVA bind files, tool config|same, split by depth of check|same, partitioned by property group|
|**Outputs**|proof status report (proven/CEX/inconclusive per property), waveform for CEX|fast pass/fail from lint-level; full proof report from scheduled run|per-group proof report, merged into one summary artifact|
|**Trade-off**|Simple, but a single job with a large property set can run very long and block the pipeline|Best MR feedback (structural bugs caught fast) without paying full-proof runtime on every push|Best scalability for large property counts, more job/license overhead|

Recommendation: Alt 2 as the primary shape (fast structural check gates MRs, full proof runs scheduled), with Alt 3's matrix technique used _inside_ the full-proof job set once the property count grows large enough to bottleneck a single job.

### 3.5 Coverage / Report

||Alt 1 — single aggregation job (current `report_template`)|Alt 2 — incremental merge across sim jobs|Alt 3 — external dashboard publish|
|---|---|---|---|
|**What**|One job parses all logs post-hoc after regression finishes|Each simulation job uploads its coverage DB as an artifact; report stage merges DBs (e.g. `urg`/`imc`) into one merged coverage report|Report job pushes JUnit + coverage summary to an external dashboard/API (Grafana, internal portal, Slack webhook) and stores the dashboard URL as a pipeline output var|
|**Inputs**|`${WORKSPACE_DIR}/logs,waveforms,reports` from all sim jobs via `needs:`|per-job coverage DB artifacts, named consistently (`cov_${TEST_NAME}_${SEED}`)|merged coverage report from Alt 2|
|**Outputs**|`summary/` artifact (as today)|merged coverage DB + trend-vs-history summary|dashboard link (as a `dotenv` report variable, consumable by later jobs/MR comment)|
|**Trade-off**|Simplest, but no persistent coverage-trend history across runs|Needed for meaningful coverage closure tracking over time; adds artifact-naming discipline requirement|Best visibility for reviewers/stakeholders, but adds an external dependency and auth handling|

Recommendation: Alt 2 is the functional requirement for real coverage closure; layer Alt 3 on top of it so reviewers see trend and status without opening raw logs. Alt 1 stays as-is for the lint/smoke-only pipelines where full coverage isn't collected.

### 3.6 Cleanup

||Alt 1 — single always-run cleanup job (current)|Alt 2 — per-job `after_script` release + separate housekeeping job|Alt 3 — async scheduled housekeeping pipeline|
|---|---|---|---|
|**What**|One `cleanup-job` at the end of the pipeline releases licenses and clears workspace|Each job releases its own license immediately via `after_script`; a lightweight `cleanup-job` only prunes leftover workspace/cache dirs|License/workspace housekeeping runs as a separate, independently scheduled pipeline (e.g. hourly), not part of the MR/regression pipeline critical path|
|**Inputs**|none (stage-scoped `when: always`)|per-job license handle|stale workspace/license state across runners|
|**Outputs**|none|none|cleanup log, license-usage report|
|**Trade-off**|Simple, but licenses stay checked out for the full pipeline duration, worst case under license contention|Frees licenses as soon as each job finishes — better for shared/limited license pools; slightly more boilerplate per template|Keeps the user-facing pipeline fast, but requires trusting an out-of-band process to reclaim resources reliably|

Recommendation: Alt 2 as the default (licenses are typically the scarce shared resource in EDA CI), with Alt 3 added only if you're running enough concurrent pipelines that per-job release still isn't enough to prevent contention.

---

## 4. Inter-Stage Contract (Inputs → Outputs)

|Stage|Consumes (from)|Produces (artifact / var)|Consumed by|
|---|---|---|---|
|`setup`|`$REPO_DEPENDENCIES`, dependency manifest/lockfile|`.venv/` (or pinned image tag), `verif_plan.yaml` resolved|lint, simulation, formal|
|`lint` (fast/structural, incl. formal-lint)|`.venv/`, RTL/testbench tree|JUnit lint report; pass/fail gate|pipeline gate only (no downstream artifact needed)|
|`simulation` (smoke / regr-short / regr-long)|`.venv/`, `DUT_CFG`, `TEST_NAME`, `SEED` (or generated matrix)|`logs/`, `waveforms/`, `reports/junit.xml`, `cov_<test>_<seed>/`|`report` (coverage merge), `report` (junit aggregation)|
|`formal` (lint / full / per-property)|`.venv/`, RTL, SVA/constraints|proof status report, CEX waveform (if any)|`report`|
|`report`/`coverage`|all sim + formal artifacts via `needs: [..., artifacts: true]`|merged coverage DB, trend summary, dashboard URL (dotenv var)|reviewers / release gate|
|`cleanup`|license handles held by upstream jobs|cleanup log|none (terminal)|

Use `needs:` with explicit `artifacts: true`/`false` per dependency (not blanket stage-ordering) so each job only pulls what it actually needs — this keeps the DAG-based pipeline execution model actually parallel instead of serializing on stage boundaries, which matters once `simulation` and `formal` both fan out into many jobs.

---

## 5. Suggested Combination for a First Implementation

1. **Reuse mechanism:** Alt A (`extends` + shared template repo) now, migrate to Alt B (Catalog components) once the template set stabilizes.
2. **Setup:** Alt 2 (prebuilt image) for the EDA/tool layer, Alt 3 (lockfile cache) for fast-moving Python deps on top.
3. **Lint:** Alt 2 (matrix by check type) as MR gate + scheduled full sweep.
4. **Simulation:** Alt 2 (static matrix) for smoke/regression-short; Alt 3 (generated child pipeline) for regression-long.
5. **Formal:** Alt 2 (fast structural in `lint` stage, full proofs scheduled), matrix-ing the full-proof job internally once property count grows.
6. **Coverage/Report:** Alt 2 (incremental merge) + Alt 3 (dashboard publish).
7. **Cleanup:** Alt 2 (`after_script` release per job + lightweight housekeeping job).

This combination keeps the MR feedback loop fast (matrix-based smoke + fast formal + gated lint) while giving regression-long/coverage/formal-full the scalability (generated pipelines, incremental merge) they need without slowing down every-push feedback.