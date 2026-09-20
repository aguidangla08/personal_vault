# GitLab Runner Configuration Notes

## Current config overview

- **Global settings**: `concurrent = 2` caps total parallel jobs across all runners; `session_timeout = 1800` (30 min) applies to the interactive web terminal.
- **Runner entry**: `executor = "docker"`, tagged `ubuntu`, `request_concurrency = 2`.
- **Docker executor settings**: `privileged = false`, `tls_verify = false`, default image `ubuntu:lts`, `/cache` volume mounted, `shm_size = 0` (Docker default, often too small for EDA tools).
- **Cache**: no remote backend (S3/GCS/Azure) configured — falls back to local volume caching only.

Flag for RTL/verification CI: `shm_size = 0` may cause issues with simulators relying on shared memory; increase explicitly if needed.

## Adding a non-Docker runner

Three executor options for a runner that doesn't use Docker:

1. **Shell executor** (recommended for EDA tooling) — jobs run directly on the host using whatever's installed (simulators, license-managed tools). No container isolation between jobs.
2. **SSH executor** — same as shell, but executes on a remote machine over SSH (useful if the EDA/license host is separate from the runner host).
3. **Custom executor** — scripts control job lifecycle (`prepare`/`run`/`cleanup`), useful for integrating with an existing job scheduler (LSF/Slurm/VM pool).

Minimal shell executor config:

```toml
[[runners]]
  name = "rtl-shell-runner"
  url = "<gitlab-url>"
  id = 30
  token = "<token>"
  executor = "shell"
  tags = ["rtl-shell"]
  request_concurrency = 1
```

## `docker pull` behavior depending on executor

- **Shell executor**: `docker pull` inside a job talks directly to the **host's** Docker daemon. Requires the `gitlab-runner` user to have daemon access (e.g. `docker` group). Pulled images persist in the host's local cache and are visible to/affected by every other job or container on that host — no isolation. Roughly equivalent to root access on the host from a security standpoint.
- **Docker executor (default, no socket/DinD)**: fails — the job's isolated container has no Docker daemon inside it.
- **Docker executor + Docker-outside-of-Docker** (mount `/var/run/docker.sock`): job talks to the host daemon, same caveats as shell executor's case.
- **Docker executor + true DinD** (`privileged = true` + `docker:dind` service): job gets its own isolated nested daemon — no shared state with host or other jobs, but `privileged = true` is a real security loosening.
- **Daemonless builders** (Kaniko, Buildah): can build/push images inside an unprivileged container with no daemon at all — doesn't help if you need generic `docker run`/`docker pull` semantics.

## Safe Docker-executor configuration options

1. **Locked-down baseline (recommended default)** — no socket, no privileged mode, dropped capabilities, image allowlist:
    
    ```toml
    [runners.docker]  privileged = false  shm_size = 268435456  pull_policy = ["if-not-present"]  allowed_pull_policies = ["if-not-present"]  allowed_images = ["ubuntu:*", "registry.yourcompany.com/*"]  allowed_services = ["registry.yourcompany.com/*"]  security_opt = ["no-new-privileges"]  cap_drop = ["ALL"]  memory = "4g"  memory_swap = "4g"  cpus = "2"
    ```
    
2. **Baseline + Kaniko** for jobs that need to build/push images without granting Docker access.
3. **Separate, restricted DinD runner** (`privileged = true`), tagged separately and gated to protected branches only — isolates the riskier capability instead of granting it runner-wide.

## How `image:` pulling/caching works

- The `image:` value in `.gitlab-ci.yml` (or `[runners.docker] image` fallback) determines what the job container runs as.
- `pull_policy` controls re-pull behavior: `"always"` (default) re-checks registry every time; `"if-not-present"` reuses local cache without checking; `"never"` fails if not already local.
- Pulled images are stored in the **host's** Docker image store (`/var/lib/docker`), shared across all jobs/pipelines on that host, and persist indefinitely — GitLab Runner does not auto-prune them.
- Re-tagged upstream images (e.g. `ubuntu:lts` moving to a new digest) leave the old digest as a dangling image consuming disk until pruned.

## Cleaning up images (with name restriction)

Three combined approaches (allowlist + cleanup mechanism):

1. **Allowlist + time-based cron prune**: `docker image prune -a --filter "until=168h" -f` on a schedule (e.g. daily). Simple, but ignores actual disk pressure.
2. **Pinned-tag allowlist + reconcile script**: script removes any local image not in the exact allowed list, keeping the host's image set matching the allowlist precisely.
3. **Allowlist + disk-threshold-triggered prune** _(chosen)_: script checks `/var/lib/docker` disk usage and only prunes when a threshold (e.g. 80%) is exceeded.

Chosen approach (option 3):

```toml
[runners.docker]
  allowed_images = ["ubuntu:lts", "registry.yourcompany.com/rtl/*"]
```

```bash
#!/bin/bash
# threshold-prune.sh
THRESHOLD=80
USAGE=$(df /var/lib/docker --output=pcent | tail -1 | tr -dc '0-9')
if [ "$USAGE" -ge "$THRESHOLD" ]; then
  docker image prune -a --filter "until=24h" -f
fi
```

Run via cron every 15 minutes:

```
*/15 * * * * root /usr/local/bin/threshold-prune.sh >> /var/log/threshold-prune.log 2>&1
```

## What cron is

Cron is the standard Linux job scheduler (`crond` daemon) that runs commands automatically at specified times, defined via `crontab -e` or files under `/etc/cron.d/`. Format: `minute hour day-of-month month day-of-week command`.

## Checking cron activity

- **Daemon status**: `systemctl status cron` (Debian/Ubuntu) or `systemctl status crond` (RHEL/Fedora).
- **Currently running cron job**: `ps aux | grep CRON` or `ps aux | grep <script-name>`.
- **Execution history**: `grep CRON /var/log/syslog` (Debian/Ubuntu), `grep CRON /var/log/cron` (RHEL), or `journalctl -u cron --since "1 hour ago"`.
- **Script's own output**: `tail -f /var/log/threshold-prune.log` — most useful for confirming whether a prune actually fired, not just that cron called the script.

## Open items / next steps

- Confirm actual disk size/mount point for `/var/lib/docker` to tune the 80% threshold.
- Decide whether any jobs genuinely need `docker run`/`docker pull` semantics (pushes toward needing a separate DinD runner) vs. building images only (Kaniko is sufficient).
- Write the finalized `threshold-prune.sh` + cron entry + `allowed_images` addition into the actual `config.toml`.