## `artifacts:name`

Purely cosmetic. It sets the filename of the zip GitLab produces when someone downloads the artifact manually from the pipeline UI or API. It has **no effect** on how `needs:` / `dependencies:` resolve artifacts between jobs — those are matched by job, not by this name.

```yaml
setup:rocky:
  artifacts:
    paths: [venv/]
    name: "venv-$CI_JOB_NAME-$CI_COMMIT_SHORT_SHA"
```

Safe to set or skip. Use it if you want readable filenames when browsing artifacts in the UI; skip it if you don't care.

## `artifacts:paths`

This is what actually matters, and only in one specific scenario.

GitLab scopes artifacts **per job**, not by path string. So two different jobs can both use `paths: [venv/]` with zero conflict — `test:rocky` pulling from `setup:rocky` and `test:ubuntu` pulling from `setup:ubuntu` each get their own isolated workspace. Same path name, no collision.

The path only needs to change if **one job consumes artifacts from more than one upstream job at the same time**:

```yaml
some-job:
  needs: [setup:rocky, setup:ubuntu]
```

Here both artifacts extract into the _same_ job workspace. If both were built with `paths: [venv/]`, the second one downloaded silently overwrites the first. That's the only case where you'd need distinct paths:

```yaml
setup:rocky:
  artifacts:
    paths: [venv-rocky/]

setup:ubuntu:
  artifacts:
    paths: [venv-ubuntu/]
```

## Summary

|Situation|Action needed|
|---|---|
|Each downstream job needs only its matching venv (your current setup)|No path change needed — `venv/` is fine everywhere|
|A job needs artifacts from multiple setup jobs at once|Give each a distinct path (e.g. `venv-rocky/`, `venv-ubuntu/`)|
|Want nicer filenames in the GitLab UI artifacts browser|Optionally set `artifacts:name` — cosmetic only|