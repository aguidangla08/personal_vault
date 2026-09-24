# Pipeline Naming Convention: Cost-Based Stages, Purpose-Named Jobs

## Recommendation

Name **stages** by cost/phase, and name **jobs within those stages** by purpose:

```yaml
stages:
  - setup
  - lint
  - verify-light
  - verify-medium
  - verify-heavy
  # - report
  # - cleanup

smoke-tests:
  stage: verify-light
  ...

sanity-tests:
  stage: verify-medium
  ...

regression:
  stage: verify-heavy
  ...
```

This resolves the tradeoff between the two naming philosophies discussed earlier rather than forcing a choice between them, and it's not a theoretical compromise — it's the pattern used by [CVA6](https://github.com/openhwfoundation/cva6), an actively maintained, tapeout-proven open-source RISC-V core, in its production `.gitlab-ci.yml`.

## The two philosophies, and why they conflict at the stage level

**Cost-based names** (`verify-light` / `verify-medium` / `verify-heavy`) describe _how expensive_ a stage is. They're self-explanatory to anyone regardless of testing background, and they don't presuppose _when_ a job runs — a `verify-heavy` job can be triggered on push, on schedule, or manually, and the name still makes sense. Their weakness is that they say nothing about _why_ a job exists: two jobs can both be "medium cost" for completely different reasons (one's a curated regression subset, another is a synthesis run), and the name gives no hint which is which.

**Purpose-based names** (`smoke` / `sanity` / `regression`) describe _why_ a job exists and roughly what confidence it's meant to buy you. They're immediately legible to anyone with a verification/testing background and carry real information: a `smoke` failure means "the build is broken," a `regression` failure means "something in the full suite is wrong." Their weakness is the opposite of cost-based names — the name implicitly assumes a trigger and a scope (`smoke` = fast, per-push; `regression` = slow, nightly), and if reality ever diverges from that assumption, the name becomes misleading.

Applied to **stage names**, cost-based wins: GitLab stages are a structural/scheduling concept (this stage's jobs run after that stage's jobs finish), so naming them by relative cost accurately describes their role in the pipeline's shape and keeps the door open to changing _when_ each cost tier is triggered without renaming anything.

Applied to **job names**, purpose-based wins: a job is where the actual verification intent lives, and that's exactly what a reader wants to know when a job goes red in the pipeline UI.

## Precedent: CVA6's `.gitlab-ci.yml`

Source: [github.com/openhwfoundation/cva6/blob/master/.gitlab-ci.yml](https://github.com/openhwfoundation/cva6/blob/master/.gitlab-ci.yml)

CVA6 draws this exact line. Its `stages:` list is cost/phase-based:

```yaml
stages:
  - setup
  - light tests
  - heavy tests
  - backend tests
  - find failures
  - report
```

And the jobs inside each stage are purpose-named:

|Stage (cost/phase)|Representative jobs (purpose)|
|---|---|
|`setup`|`check_env`, `build_tools`|
|`light tests`|`smoke-tests`, `smoke-gen`, `smoke-bench`, `smoke-hwconfig`, `hello-pk`, `sdtrig-tests`|
|`heavy tests`|`compliance`, `riscv_arch_test`, `riscv-tests-v`, `riscv-tests-p`, `generated_tests`, `directed_isacov-tests`, `spyglass` (lint/reporting), `asic-synthesis`|
|`backend tests`|`simu-gate` (gate-level sim), `fpga-boot`, `code_coverage-report`|
|`report`|`merge reports`|

Notice that `light tests` and `heavy tests` say nothing about _what_ is being checked — they only bound cost and rough trigger scope (light tests run on nearly every push/MR/schedule; heavy tests are gated to RTL changes on regression/verification pipeline runs). The actual meaning — "this is a build-health check," "this is architectural compliance," "this is a gate-level timing-accurate simulation" — lives entirely in the job names underneath. A reader scanning the pipeline graph gets cost/urgency from the stage column and intent from the job label, in one glance, without either axis overloading the other.

## Applied to this project

Mapping your proposed stage list onto this pattern:

- **`lint`** stage → job(s) named for what they check, e.g. `lint-rtl`, `lint-license-headers`, `lint-commit-msg` (CVA6's analogue: `spyglass`, sitting under `heavy tests` for them since spyglass is expensive in their flow — worth deciding whether lint-proper is cheap enough to keep in its own fast stage, or heavy enough to fold into `verify-heavy` the way CVA6 does).
- **`verify-light`** stage → `smoke-tests`, `sanity-tests` if the sanity subset is cheap enough to sit alongside smoke.
- **`verify-medium`** stage → whichever purpose-named jobs sit between "does it build" and "full sign-off" for your flow — e.g. `sanity-tests` if it's expensive enough to want its own tier, or a `merge-regression` job scoped to changed modules.
- **`verify-heavy`** stage → `regression`, `coverage-regression`, and anything gate-level or synthesis-adjacent, mirroring CVA6's `heavy tests` / `backend tests` split if gate-level sim ends up warranting its own stage later.

The stage list stays cost-legible and stable even if you later change which trigger fires which tier; the job names stay purpose-legible even as you add or split jobs within a tier.