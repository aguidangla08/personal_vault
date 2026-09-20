## Question

In the scenario where a job needs artifacts from both the Rocky Linux and Ubuntu setup jobs, can repository cloning happen in a job running on either image, and will that cloned repo be compatible with both?

## Answer

Yes. Cloning is fine on either image, and the checked-out result is compatible with both. A `git clone`/checkout just writes plain files to disk (source code, configs, etc.) — it isn't compiled or OS-specific like a Python wheel is. Git doesn't embed platform state into a normal checkout, so whichever image runs the clone, the resulting files work identically when picked up by a job on the other image.

### Edge cases (rarely relevant, but worth knowing)

- **Executable bit / permissions**: git tracks the executable bit, and GitLab artifacts (zipped) generally preserve it, but it's worth double-checking if you have shell scripts that need `+x`.
- **Line endings**: if `.gitattributes` normalizes line endings, checkout is consistent regardless of which OS did it. Without that, it's still fine here since both images are Linux (no CRLF/LF mismatch like you'd get on Windows).
- **Symlinks**: preserved fine between Linux images.
- **Git submodules with credentials**: if using SSH/token auth for submodules, make sure whichever job clones has the right credentials configured — that's a job-config concern, not an OS-compatibility one.

## Architecture options

### 1. Let each job clone independently (GitLab's default) — recommended

Every job already does its own `git checkout` automatically via `GIT_STRATEGY: clone` (the default). `test:rocky` and `test:ubuntu` each fetch the repo themselves — no artifact needed, no shared-path collision to worry about.

```yaml
test:rocky:
  image: rocky8.9-python3.12
  needs: [setup:rocky]
  script: [venv/bin/pytest]
# git checkout happens automatically, no extra config
```

Simplest option. Default unless you have a specific reason not to.

### 2. One clone job, artifact passed to both

Useful only if you want to guarantee both jobs see the *exact same* commit/tree snapshot (e.g. avoiding a race if the branch moves between job starts), or to save repeated clone cost on a very large repo.

```yaml
clone:
  script:
    - git clone --depth 1 $CI_REPOSITORY_URL repo
  artifacts:
    paths: [repo/]
    expire_in: 1 hour
```

If this is later pulled into the same job as the venv artifacts, make sure `repo/` doesn't collide with `venv-rocky/` / `venv-ubuntu/` (see the naming doc — same path-collision rule applies).

### 3. Fold the clone into each setup job

Since `setup:rocky` / `setup:ubuntu` already build their own venv, you could rely on GitLab's automatic checkout inside those jobs and pass `venv/` plus the needed source as one combined artifact — no separate clone job at all.

## Recommendation

Go with **option 1** unless you have a concrete reason (very large repo, need a pinned commit across parallel jobs) to add a dedicated clone job.