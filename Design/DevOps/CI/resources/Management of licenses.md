# Managing EDA licenses in GitLab CI

## Options at a glance

1. Runner concurrency
2. Dynamic child pipeline with X lanes
3. `resource_group`
4. `lmutil` gate in the job
5. FlexLM options file (server-side cap)
6. Tool-side license wait/queue option
7. Retry on license denial
8. Delayed follow-up job
9. Chain jobs with `needs` / one stage per job
10. Tier the tests
11. Split the long job into chunks
12. Hold the license briefly
13. Manual heavy job
14. Auto-cancel redundant pipelines
15. Central license project

---

## 1. Runner concurrency

Tag every license-consuming job and serve that tag with a runner that can only run X jobs at once. Jobs beyond X stay pending. X lives in the runner config, not in the pipeline, and other pipelines sharing the runner compete for the same slots.

```toml
# config.toml
concurrent = 4
[[runners]]
  name  = "license-runner"
  limit = 4
```

```yaml
.license_job:
  tags: [eda-license]
```

## 2. Dynamic child pipeline with X lanes

A generator script takes the job list sorted by load, assigns each job to the least-loaded of X lanes (heaviest first) and chains each lane with `needs`. At most X jobs run at any time and X is a CI variable.

```yaml
variables:
  MAX_LICENSES: "4"

generate:
  stage: prepare
  script: python3 ci/gen_lanes.py --lanes "$MAX_LICENSES" --jobs ci/jobs.yml > lanes.yml
  artifacts:
    paths: [lanes.yml]

run-sims:
  needs: [generate]
  trigger:
    include:
      - artifact: lanes.yml
        job: generate
    strategy: depend
```

## 3. `resource_group`

A mutex: only one job per group name runs at a time, the rest wait. It is scoped to a single project and also serializes across pipelines of that project. For X slots, define X group names and spread jobs across them.

```yaml
sim_a:
  resource_group: eda-license
  script: rtl-make cocotb run TESTS=a
```

## 4. `lmutil` gate in the job

The job polls the license server and only starts the tool when fewer than X licenses are in use. Simple, but check-then-run is not atomic, so two jobs can pass the check together.

```bash
# ci/wait_license.sh
MAX=${MAX_LICENSES:-4}
while :; do
  used=$(lmutil lmstat -f "$LIC_FEATURE" -c "$LM_LICENSE_FILE" \
         | grep -oP 'Total of \d+ licenses? in use: *\K\d+' | head -1)
  [ "${used:-0}" -lt "$MAX" ] && break
  sleep $(( 20 + RANDOM % 20 ))
done
```

```yaml
sim_job:
  script:
    - ci/wait_license.sh
    - rtl-make cocotb run
```

## 5. FlexLM options file (server-side cap)

The license server enforces the maximum for a user or group. It is the only truly atomic option, but it needs license admin access and X must be kept in sync with the pipeline.

```
MAX 4 my_sim_feature USER ci_user
```

## 6. Tool-side license wait/queue option

Many simulators can wait for a license themselves instead of failing. Check your tool's documentation for the exact flag or environment variable. Waiting jobs still occupy a runner slot.

```yaml
sim_job:
  variables:
    SIM_LICENSE_WAIT: "1"      # placeholder: use your tool's real option
  script: rtl-make cocotb run
```

## 7. Retry on license denial

Have the wrapper script exit with a distinct code when the license is denied, and let GitLab retry only for that code. Retries have no delay and are capped at 2, so combine with a wait loop. `exit_codes` needs a recent GitLab (16.x).

```yaml
sim_job:
  timeout: 3h
  retry:
    max: 2
    when: script_failure
    exit_codes: 75
  script:
    - LIC_WAIT_TIMEOUT=1800 ci/wait_license.sh || exit 75
    - rtl-make cocotb run
```

Because the check in option 4 is not atomic, the more robust variant retries the real run and greps the log for a license error:

```bash
for attempt in $(seq 1 "${LIC_ATTEMPTS:-30}"); do
  rtl-make cocotb run 2>&1 | tee run.log
  rc=${PIPESTATUS[0]}
  [ "$rc" -eq 0 ] && exit 0
  grep -qiE 'license|FLEXnet' run.log || exit "$rc"
  sleep $(( 60 + RANDOM % 60 ))
done
exit 75
```

## 8. Delayed follow-up job

A second copy of the job starts later with `when: delayed`, which frees the runner while waiting. Verify on your GitLab version that `needs`, `allow_failure` and `on_failure` interact as expected; test with a forced failure first.

```yaml
sim_job_retry:
  needs: [sim_job]
  when: delayed
  start_in: 15 minutes
  script: ci/wait_license.sh && rtl-make cocotb run
  rules:
    - when: on_failure
```

## 9. Chain jobs with `needs` / one stage per job

Jobs in the same stage run in parallel and `needs` ignores stage order, so to run one by one, chain them explicitly or give each its own stage. By default a failure skips everything after it unless you use `allow_failure`.

```yaml
sim_a:
  needs: [setup]
  script: ...
sim_b:
  needs: [setup, sim_a]
  script: ...
sim_c:
  needs: [setup, sim_b]
  script: ...
```

```yaml
stages: [setup, sim1, sim2, sim3]
```

## 10. Tier the tests

Run a short smoke subset on every merge request and the multi-hour regression only on a schedule. This is usually the biggest saving of license time.

```yaml
sim_smoke:
  resource_group: eda-license
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  script: rtl-make cocotb run TESTS=smoke

sim_full:
  resource_group: eda-license
  timeout: 10h
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
  script: rtl-make cocotb run TESTS=all
```

## 11. Split the long job into chunks

Several shorter chained jobs mean a failure or runner hiccup only loses a chunk, and you get partial results early. Keep `timeout` within the runner's maximum job timeout.

```yaml
.chunk:
  resource_group: eda-license
  timeout: 3h
  retry: { max: 1, when: [runner_system_failure, stuck_or_timeout_failure] }

sim_part1:
  extends: .chunk
  script: rtl-make cocotb run TESTS=group1

sim_part2:
  extends: .chunk
  needs: [sim_part1]
  script: rtl-make cocotb run TESTS=group2
```

## 12. Hold the license briefly

Do everything that does not need a license (checkout, dependencies, codegen, and compilation if your tool allows it) in an unlicensed job, and take the `resource_group` only for the simulation run. Running many tests in one simulator session also uses one seat instead of one per job.

```yaml
build:
  script: rtl-make cocotb build
  artifacts:
    paths: [${TEST_MODULE_PATH}/ver/sim/cocotb/build/]
    expire_in: 1 day

sim:
  needs: [build]
  resource_group: eda-license
  script: rtl-make cocotb run
```

## 13. Manual heavy job

The full regression only runs when someone presses Play, ideally when the license is free. `allow_failure: true` keeps a not-yet-run manual job from blocking the pipeline.

```yaml
sim_full:
  resource_group: eda-license
  when: manual
  allow_failure: true
  timeout: 10h
  script: rtl-make cocotb run TESTS=all
```

## 14. Auto-cancel redundant pipelines

Cancel older pipelines on the same ref when a newer commit arrives, so they do not queue for the license for hours. It only cancels when every started job is `interruptible`. Avoid it on main, develop, master and release refs, where you want every commit's full result.

```yaml
workflow:
  auto_cancel:
    on_new_commit: interruptible

sim_full:
  interruptible: true
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      interruptible: true
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      interruptible: false
```

`workflow:auto_cancel` is fairly recent (16.x); the "Auto-cancel redundant pipelines" project setting is the older equivalent.

## 15. Central license project

`resource_group` is per project. To share one license across repositories, put the license-holding job in one central project and trigger it from the others; the central project's `resource_group` then serializes all callers.

```yaml
# In each consumer repository
run-licensed-sim:
  trigger:
    project: group/license-runner
    branch: main
    strategy: depend
  variables:
    TESTS: smoke
```

```yaml
# In group/license-runner
sim:
  resource_group: eda-license
  script: rtl-make cocotb run TESTS=$TESTS
```

---

## Suggested combination

- Server enforces the cap (option 5) if you have admin access; otherwise `resource_group` (3) or the central project (15) for sharing across repos.
- `lmutil` gate (4) plus retry on denial (7) so jobs wait instead of failing.
- Tier and split the tests (10, 11), and keep license time short (12).
- Cancel redundant pipelines only on feature branches and merge requests (14).