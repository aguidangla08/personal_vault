## Scope
I want you to generate me a RTL verification CI architecture example.

## Generation process notes
- I need you to make me 3 architecture alternatives for each one of the actions
- I will send the current code so that you can work from there, but I don't want you to verify the code, just take a look to the structure to start from there

## Requirements
- This CI is intended to execute simulation and formal runs
- Use generic jobs that must be reusable for multiple repositories
- verification stages (smoke and regression), (lint, simulation, formal, ...)
- specify inputs and outputs between stages
- Define verification levels/stages (smoke, regression, formal, coverage, ...)
	- Specify different levels with short and brief, and long and complete regressions

# Global defaults (e.g., container image configuration, caching)
default:
  image:
    name: nvc_ubuntu:latest
    pull_policy: never
  tags:
    - docker-ready

include:
  - local: 'ci/vars/common.yml'
  - local: 'ci/vars/lint-job.yml'
  - local: 'ci/vars/smoke-test.yml'
  - local: 'ci/vars/regression-test.yml'

variables:
  WORKSPACE_DIR: "${CI_PROJECT_DIR}/workspace"
  LICENSE_SERVER: "license-server.local"
  GIT_SUBMODULE_STRATEGY: recursive

stages:
  - setup
  - lint
  - test
  - report
  - cleanup

# ---------------------------------------------------------------
# Base / Anchor Templates
# ---------------------------------------------------------------

# Base setup for jobs needing workspace paths and basic tooling env
.base_job:
  extends: .vars-common
  variables:
    ENV_NAME: "prod"
  before_script:
    - mkdir -p ${WORKSPACE_DIR}
    - echo "Executing job in ${CI_JOB_STAGE} stage..."

# ---------------------------------------------------------------
# Stage 1: Setup
# ---------------------------------------------------------------
.setup_template:
  extends: .base_job
  stage: setup
  script:
    - echo "Verifying tool availability and right versions and license server connectivity to ${LICENSE_SERVER}..."
    - echo "Preparing cache and checking out submodules if needed..."

# ---------------------------------------------------------------
# Stage 2: Lint
# ---------------------------------------------------------------
.lint_template:
  extends: .base_job
  stage: lint
  script:
    - echo "Running syntax, style, and compile checks on RTL and testbench code..."
  allow_failure: false

# ---------------------------------------------------------------
# Stage 3: Test
# ---------------------------------------------------------------
.test_template:
  extends: .base_job
  stage: test
  variables:
    TEST_SUITE: "smoke"
    SEED: "random"
  script:
    - echo "Running simulation suite '${TEST_SUITE}' with seed '${SEED}'..."
  artifacts:
    name: "sim_outputs_${CI_JOB_NAME}_${CI_COMMIT_SHORT_SHA}"
    when: always
    expire_in: 1 week
    paths:
      - ${WORKSPACE_DIR}/logs/
      - ${WORKSPACE_DIR}/waveforms/
      - ${WORKSPACE_DIR}/reports/
    reports:
      junit: ${WORKSPACE_DIR}/reports/junit.xml

# ---------------------------------------------------------------
# Stage 4: Report
# ---------------------------------------------------------------
.report_template:
  extends: .base_job
  stage: report
  script:
    - echo "Aggregating simulation results and parsing logs..."
    - echo "Generating summary reports and coverage metrics..."
  artifacts:
    name: "verification_summary_${CI_COMMIT_SHORT_SHA}"
    when: always
    expire_in: 1 month
    paths:
      - ${WORKSPACE_DIR}/summary/

# ---------------------------------------------------------------
# Stage 5: Cleanup
# ---------------------------------------------------------------
.cleanup_template:
  stage: cleanup
  script:
    - echo "Releasing commercial licenses..."
    - echo "Cleaning up local workspace temporary directories..."
  when: always

# ===============================================================
# Concrete Job Implementations (Applying Verification Stage Gating)
# ===============================================================

setup:
  extends: .setup_template
  variables:
    TEMP_CLONE: "true"
  script:
    - python3 -m venv .venv
    - source .venv/bin/activate
    - echo "Install repository dependencies as python packages"
    - |
      for entry in ${REPO_DEPENDENCIES}; do
        repo="${entry%@*}"
        branch="${entry##*@}"
        name=$(basename "${repo}" .git)
        echo "Cloning ${repo} @ ${branch} and installing as python package"
        git clone --depth 1 --branch "${branch}" --single-branch "${repo}.git" "${name}"
        echo "${name} Commit ID:$(git -C ${name} rev-parse HEAD)"
        echo "${name} Commit name:$(git -C ${name} log -1 --pretty=%s)"
        pip install "./${name}"
      done
    # --- TEMP: remove this block once CI is integrated inside ethernet-core ---
    - |
      if [ "$TEMP_CLONE" = "true" ]; then
        echo "Running ethernet clone"
        entry="${TEMP_ETH_DEPENDENCY}"
        repo="${entry%@*}"
        branch="${entry##*@}"
        name=$(basename "${repo}" .git)
        echo "Cloning ${repo} @ ${branch}"
        git clone --depth 1 --branch "${branch}" --single-branch "${repo}.git" "${name}"
        cd ${name}
      fi
    # --- END TEMP ---
  artifacts:
    paths:
      - .venv/
      # --- TEMP: remove this block once CI is integrated inside ethernet-core ---
      - ${name}/
      # --- END TEMP ---

lint-job:
  extends: [.lint_template, .vars-lint-job]
  needs:
    - job: setup
      artifacts: true
  variables:
    TEMP_CLONE: "true"
  script:
    # --- TEMP: remove this block once CI is integrated inside ethernet-core ---
    - cd ethernet-core
    # --- END TEMP ---
    - source ${CI_PROJECT_DIR}/.venv/bin/activate
    - cd $TEST_MODULE_PATH
    - pip install ./ver/sim/cocotb/
    - rtl-make cocotb build
  artifacts:
    paths:
      - .venv/
      # --- TEMP: remove this block once CI is integrated inside ethernet-core ---
      - ethernet-core/
      # --- END TEMP ---

# Level 1: Smoke Test - Runs on all pushes and MRs for fast feedback
smoke-test:
  extends: [.test_template, .vars-smoke-test]
  needs:
    - job: lint-job
      artifacts: true
  variables:
    TEMP_CLONE: "true"
  rules:
    - if: $CI_PIPELINE_SOURCE == "push"
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  script:
    # --- TEMP: remove this block once CI is integrated inside ethernet-core ---
    - cd ethernet-core
    # --- END TEMP ---
    - source ${CI_PROJECT_DIR}/.venv/bin/activate
    - cd $TEST_MODULE_PATH
    - pip install ./ver/sim/cocotb/
    - rtl-make cocotb run -t $TEST_NAME --dcfg $DUT_CFG
  artifacts:
    paths:
      - .venv/
      # --- TEMP: remove this block once CI is integrated inside ethernet-core ---
      - ethernet-core/
      # --- END TEMP ---

# Level 2: Regressions - Runs on MRs to main/develop or on schedule builds
regression-test:
  extends: [.test_template, .vars-regression-test]
  needs:
    - job: setup
      artifacts: true
  variables:
    TEST_SUITE: "regression"
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && ($CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "develop" || $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "main" || $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "master")
    - when: never
  script:
    # --- TEMP: remove this block once CI is integrated inside ethernet-core ---
    - cd ethernet-core
    # --- END TEMP ---
    - source ${CI_PROJECT_DIR}/.venv/bin/activate
    - cd $TEST_MODULE_PATH
    - pip install ./ver/sim/cocotb/
    - rtl-make cocotb regression -cf $REGRESSION_CFG
  artifacts:
    paths:
      - .venv/
      # --- TEMP: remove this block once CI is integrated inside ethernet-core ---
      - ethernet-core/
      # --- END TEMP ---

report-job:
  extends: .report_template
  needs:
    - job: regression-test
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && ($CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "develop" || $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "main" || $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "master")
    - when: never

cleanup-job:
  extends: .cleanup_template