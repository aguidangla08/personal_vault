## The problem

A `setup` job needs to pull a repository (or produce some other output) that a later `build` job in the same pipeline needs to use, without re-cloning or recomputing it.

## Recommended approach: artifacts

GitLab's own docs confirm artifacts are the standard mechanism for this — later-stage jobs automatically fetch artifacts from earlier-stage jobs.

```yaml
stages:
  - setup
  - build

setup:
  stage: setup
  script:
    - git clone https://github.com/your-org/rtl-make.git repo
  artifacts:
    paths:
      - repo/
    expire_in: 1h

build:
  stage: build
  needs: ["setup"]
  script:
    - cd repo
    - make build
```

`needs: ["setup"]` creates the dependency and automatically pulls in `setup`'s artifacts — no extra `dependencies:` keyword required.

## When to use something else

|Use case|Better fit|
|---|---|
|Passing a small value (commit SHA, version string, flag) instead of files|`dotenv` report artifacts|
|Reusing the same content across _multiple pipeline runs_, not just jobs within one run|`cache:`|
|Files are large and transfer/storage overhead of artifacts is too costly|Shallow clone (`--depth=1`) per job, or a pinned/persistent runner workspace|

### Dotenv example

```yaml
setup:
  script:
    - echo "COMMIT_SHA=$(git rev-parse HEAD)" >> vars.env
  artifacts:
    reports:
      dotenv: vars.env

build:
  needs: ["setup"]
  script:
    - echo "Using commit $COMMIT_SHA"   # auto-available as a job variable
```

## The expiry risk

`expire_in` is a wall-clock duration starting when the artifact is created — it is **not** scoped to "until this pipeline finishes." If `build` is delayed long enough (waiting for a runner, a slow prior stage, a manual approval gate) that `expire_in` elapses before it runs, the artifact is deleted and `build` fails to fetch it.

**Mitigation:** set `expire_in` comfortably above your pipeline's worst-case duration (queue time + job runtime), not just the typical happy-path time.

### A related but different protection

GitLab keeps artifacts from **the most recent successful pipeline on each ref**, regardless of `expire_in`. This protects a pipeline that has _already completed and succeeded_ from a later cleanup sweep — it does **not** protect an in-progress pipeline where a job is delayed past its artifact's expiry before consuming it. That race is only closed by setting `expire_in` generously.

## Why cache is not a good fit for this

Cache and artifacts solve different problems, and the difference matters for correctness, not just speed:

- **No guarantee it holds *this* pipeline's data.** A cache is keyed (often by branch or a lockfile hash) and can be a leftover from a *previous* pipeline run, or not exist yet on a fresh runner. There's no hard link between "the `setup` job in *this* pipeline wrote it" and "the `build` job in *this* pipeline reads it." Artifacts, via `needs:`, tie the consuming job to that exact job's output in that exact pipeline.
- **Cache is best-effort, not guaranteed.** A cache miss doesn't fail the job — it just runs without the cached data. Fine for speeding up `npm install`/`pip install`; wrong for something a downstream job actually depends on to function, since `build` could silently proceed with no checkout at all.
- **No dependency ordering guarantee.** `artifacts` + `needs:` gives an explicit graph: `build` cannot run without `setup`'s output existing. Cache has no equivalent — nothing enforces that the producing job finished before the consuming job reads the cache.
- **Race conditions across concurrent pipelines.** Two pipelines on the same ref (e.g. quick successive pushes, retries) can share a cache key and clobber each other's cache mid-run. Artifacts are pipeline-scoped, so this doesn't happen.

**Rule of thumb:** cache is for *reuse across pipeline runs* to speed things up, where "maybe stale, maybe missing" is acceptable (dependency caches, build caches). Artifacts are for *deterministic hand-off within one pipeline run*, which is what "job A produces X, job B needs exactly that X" requires — the reason `setup`/`build` should use artifacts, not cache.

## Retry (partial mitigation, not a fix)

```yaml
build:
  retry:
    max: 2
    when:
      - runner_system_failure
      - api_failure
```

Retries on infra flakiness (dropped runner connection, failed API calls). It does **not** recreate a deleted artifact — if the artifact expired, retrying `build` alone won't help, since `setup` isn't re-run. Omitting `when:` retries on _any_ failure, which would also catch an expired-artifact failure, but risks masking real build failures behind a retry.

## Summary

Artifacts are the right, documented tool for passing the cloned repo from `setup` to `build`. The main thing to get right is a realistic `expire_in` — pad it well above your worst-case pipeline timing rather than the typical case.

## Examples

Found a solid real-world example: **[srsRAN_Project](https://github.com/srsran/srsRAN_Project/blob/main/.gitlab-ci.yml)** — a well-known open-source 4G/5G software radio suite, with a `.gitlab-ci.yml` that uses exactly the patterns we discussed:

- **`dotenv` reports** — a `get plugin commit` job writes `CI_PLUGIN_COMMIT_SHA` and other values to `info.env` and exposes it via `artifacts: reports: dotenv: info.env`, so downstream jobs pick up those variables automatically — the real-world version of the small example I built for you.
- **File artifacts with `paths:`** — jobs like `unit coverage` and `pages` pass actual files forward (e.g. `coverage_html`, `public`) using `artifacts: paths:`, same shape as our `setup`/`build` repo example.
- **`needs:` with explicit artifact control** — jobs declare things like `needs: [{job: unit coverage dev, artifacts: true}]`, which is the more explicit form of what `needs: ["setup"]` does implicitly in our example (pull in that job's artifacts as a dependency).

It's a good reference because it shows both mechanisms — file-based artifacts and dotenv variable-passing — used together in a real, actively maintained project rather than a toy snippet. Want me to add a link to this as a "real-world example" note in the markdown doc?

Sources:

- [srsRAN_Project/.gitlab-ci.yml](https://github.com/srsran/srsRAN_Project/blob/main/.gitlab-ci.yml)