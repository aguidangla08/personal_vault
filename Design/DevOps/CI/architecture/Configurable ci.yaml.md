# Design Document: Externalizing GitLab CI Job Variables

## 1. Problem Statement

`ci.yaml` currently hardcodes pipeline and job-specific variables inline. This makes them hard to audit, hard to reuse across pipelines/projects, and forces a YAML edit (and MR) for every value change — even trivial ones like a version bump or a feature flag toggle.

**Goal:** move these values into a configuration file that `ci.yaml` reads at pipeline/job time, without losing GitLab CI functionality (variable precedence, masking, per-job overrides, `rules:` conditionals, etc).

> **Constraint to keep in mind while comparing options:** `rules:` is evaluated when the pipeline is _created_, before any job's `before_script`/`script` runs. Any approach that only sets variables by running shell code inside a job (e.g. `source ci-vars.sh`) means those variables **do not exist yet** when GitLab decides whether to include that job or a later job via `rules:`. This rules out shell-sourcing as a general-purpose solution and is the main weakness of the approach currently in use.

---

## 2. Candidate Architectures

### Option A — Declarative YAML variable files, merged via `extends` / `include`

Store variables as data, not code. Each job (or logical group) gets its own hidden template containing only a `variables:` block, kept in versioned `.yml` files under `ci/vars/`. Real jobs `extends` the relevant template(s) instead of sourcing a script.

```
ci/
  vars/
    common.yml
    build-job.yml
    deploy-prod.yml
```

```yaml
# ci/vars/common.yml
.vars-common:
  variables:
    ENV_NAME: "prod"
    WORKSPACE_DIR: "/builds/workspace"

# ci/vars/build-job.yml
.vars-build-job:
  variables:
    IMAGE_TAG: "1.4.2"
    FEATURE_X_ENABLED: "true"

# ci/vars/deploy-prod.yml
.vars-deploy-prod:
  variables:
    REPLICAS: "5"
    DEPLOY_STRATEGY: "rolling"
```

```yaml
# ci.yaml
include:
  - local: 'ci/vars/common.yml'
  - local: 'ci/vars/build-job.yml'
  - local: 'ci/vars/deploy-prod.yml'

.base_job:
  extends: .vars-common
  before_script:
    - mkdir -p ${WORKSPACE_DIR}
    - echo "Executing job in ${CI_JOB_STAGE} stage..."

build-job:
  extends: [.base_job, .vars-build-job]
  script:
    - echo "Building image ${IMAGE_TAG}"
  rules:
    - if: '$FEATURE_X_ENABLED == "true"'   # works — variable exists at pipeline creation

deploy-prod:
  extends: [.base_job, .vars-deploy-prod]
  script:
    - echo "Deploying with ${REPLICAS} replicas"
```

Because `extends` performs a proper deep-merge of `variables:` blocks (right-most/most-specific wins, same as normal GitLab precedence rules), this is the direct equivalent of "one script per job" — except it's a data file, not a script, so it composes correctly with `rules:`.

Consumer repos can still override: an `include:` of a consumer-provided `ci/vars/*.yml` file (project-local, or a separate small repo/MR) works with the same mechanism, and multiple included files with the same top-level `variables:` key are merged by GitLab, with later includes winning per key.

**Pros**

- No parsing/sourcing step — native GitLab merge semantics (predictable precedence).
- Fully visible to `rules:`, `only:`/`except:`, and interpolation in `script:`.
- Diffable, reviewable, git-blame-able per job.
- Trivial to add a per-job file without touching `ci.yaml` logic.

**Cons**

- Not a place for secrets — anything in a committed YAML file is plaintext and cannot be "masked" in job logs (masking is a runtime feature tied to variables registered in Settings → CI/CD → Variables, not something you can declare in a file).
- Wildcard `include: local: 'ci/vars/*.yml'` requires a reasonably recent GitLab version; older versions need each file listed explicitly (or a small generator step that writes an aggregate `include` list — adds complexity back).

---

### Option B — Shell-sourced overrides

Consumer repos own a `ci-vars.sh` at the repo root; a shared `.load-vars` template sources it in `before_script`, so any job built on `.base_job` picks up whatever variables that script exports.

```yaml
.load-vars:
  before_script:
    # Consumer repos can configure this pipeline by providing their own ci-vars.sh,
    # which sets/exports any CI variables they want to override.
    - 'source ci-vars.sh'

.base_job:
  variables:
    ENV_NAME: "prod"
  before_script:
    - !reference [.load-vars, before_script]
    - mkdir -p ${WORKSPACE_DIR}
    - echo "Executing job in ${CI_JOB_STAGE} stage..."

build-job:
  extends: .base_job
  script:
    - echo "Building ${IMAGE_TAG}"
```

```bash
# ci-vars.sh (consumer repo)
export IMAGE_TAG="1.4.2"
export FEATURE_X_ENABLED="true"
```

**Pros**

- Consumers configure the pipeline with a single file they fully own, using real shell (conditionals, computed values, calls to other tools) — the most flexible option of the four.
- Zero changes to `ci.yaml` needed per consumer repo; onboarding a new repo is "drop a `ci-vars.sh` in."
- No dependency on GitLab version features (`extends` merge behavior, dotenv reports, wildcard includes) — works on any GitLab CI runner.

**Cons**

- **Breaks `rules:`.** Variables only exist once `before_script` runs, which is after GitLab has already decided which jobs to create and evaluated their `rules:`/`only:`/`except:`. A job that should be skipped based on `FEATURE_X_ENABLED` can't be, and `IMAGE_TAG`-based conditionals on job inclusion are impossible. This is disqualifying for any variable that needs to influence pipeline shape, not just script behavior.
- **No masking/protection.** Anything exported here is plaintext in the script and in job logs if echoed — GitLab's masked/protected variable machinery doesn't apply to shell-sourced values.
- **Weak auditability.** A `git blame` on `ci.yaml` won't show why a build behaved differently — the actual value lives in a separate untyped script, easy to drift between repos, no schema/validation.
- **Ordering/precedence is manual.** GitLab's built-in precedence (project var > group var > `.gitlab-ci.yml` > pipeline default) doesn't apply; if you want per-job overrides you have to hand-roll the logic inside the script (`case "$CI_JOB_NAME" in ...`), which is exactly the "per-job script" duplication this project is trying to get away from.
- **Silent failures.** `source ci-vars.sh` with no guard fails the job if the file is missing (the commented-out guarded version avoids this but then silently no-ops with no visibility that overrides didn't apply).

### Option C — Runtime JSON Parsing (jq/yq in `before_script`)

`ci.yaml` keeps its current job structure. A shared `before_script` (via `extends:` or a YAML anchor) uses `jq` to read `ci-vars.json` at **job runtime** and exports values as shell env vars.

```yaml
.load-vars: &load-vars
  before_script:
    - export APP_VERSION=$(jq -r '.app_version' ci-vars.json)
    - export DEPLOY_TARGET=$(jq -r '.deploy_target' ci-vars.json)

build:
  extends: .load-vars
  script:
    - echo "Building $APP_VERSION for $DEPLOY_TARGET"
```

**How it works:** the JSON file is just data read by shell commands inside each job's container. GitLab itself never "sees" the JSON — it only sees the resulting shell env vars.

**Pros**

- Simplest to implement and understand; no generator/templating layer.
- No new pipeline stage, no child pipelines — pipeline structure is unchanged.
- Easy to test locally (`jq` command works outside CI too).

**Cons**

- Values aren't real GitLab CI/CD variables — they don't exist until `before_script` runs, so they can't be used in `rules:`, `only:/except:`, or other job-definition-time YAML logic.
- `jq` expression repeated/extended per job unless carefully centralized via `extends`.
- No masking of secret values (don't put secrets in this JSON — use GitLab CI/CD variables for those regardless of architecture).

**Best fit:** values that are only ever consumed _inside_ `script:` blocks (build args, app config, feature flags) and never need to gate which jobs run.
