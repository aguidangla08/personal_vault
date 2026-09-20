# GitLab Runner — Notes

## Where things live

|What|Location|
|---|---|
|Main config|`/etc/gitlab-runner/config.toml` (or `~/.gitlab-runner/config.toml` for a user-level install)|
|Service|managed by `systemd` as `gitlab-runner`|
|Logs|`journalctl -u gitlab-runner`|

## Checking the existing setup

```bash
sudo cat /etc/gitlab-runner/config.toml   # full config: executor, tags, image, limits
sudo gitlab-runner list                    # registered runners, from the binary
gitlab-runner status                       # is it running
gitlab-runner --version
sudo systemctl status gitlab-runner
sudo journalctl -u gitlab-runner -n 100 --no-pager
```

Also check in GitLab itself: **Project/Group → Settings → CI/CD → Runners** — shows tags, online status, and whether the runner is scoped to one project, a group, or shared instance-wide (not visible from the config file alone).

## Architecture options for using it on a shared machine

1. **Single runner, higher `concurrent`** — simplest, one executor/environment, more parallel jobs.
2. **Multiple runner registrations, same executor, different tags** — route jobs by tag (e.g. `simulation` vs `synthesis`), separate concurrency/limits per category, same underlying image.
3. **Multiple runners, different executors** — e.g. `shell` executor for fast/lint jobs, `docker` executor for heavier sim/synth jobs pulling from Nexus. ← chosen approach for reusing this host for a second project.

## Reusing a runner for a different repo — 3 options

1. **Re-register** the existing runner to the new project (`unregister` + `register`) — destructive, stops serving the current repo.
2. **Rescope in GitLab UI** (Settings → CI/CD → Runners) — no host changes, needs Maintainer/Owner access.
3. **Add a second `[[runners]]` block** in `config.toml`, registered against the new project — additive, doesn't touch the existing block. _(used here)_

## Adding a second runner (option 3)

```bash
sudo gitlab-runner register
```

Prompts: GitLab URL, registration token (from new project's Settings → CI/CD → Runners), description, tags, executor, default image (if docker).

This appends a new `[[runners]]` block to `config.toml`; the existing block is untouched.

```toml
concurrent = 4   # shared across ALL runner blocks — raise if needed

[[runners]]
  name = "existing-runner"
  ...              # unchanged

[[runners]]
  name = "my-second-project-runner"
  url = "https://gitlab.example.com"
  token = "glrt-..."
  executor = "docker"
  tag_list = ["my-tests"]
  limit = 2                     # optional: cap concurrent jobs on this runner
  [runners.docker]
    image = "nexus.example.com/rtl-toolchain:latest"
    cpus = "4"                  # optional: hard resource cap
    memory = "8g"               # optional: hard resource cap
```

Verify:

```bash
sudo gitlab-runner list
```

Should show "Online" in the new project's GitLab Runners UI shortly after.

## Hand-editing after registration

Safe to edit directly in `config.toml`: `tag_list`, `limit`, `concurrent`/`request_concurrency`, and anything under `[runners.docker]` (`cpus`, `memory`, `image`, `volumes`, etc.). Changes are picked up automatically (config is watched), or force it with:

```bash
sudo systemctl restart gitlab-runner
```

**Do not** hand-edit `token`, `id`, or `token_obtained_at` — these are issued by GitLab at registration time.

## Undoing the second runner

```bash
sudo gitlab-runner list                                  # find its name
sudo gitlab-runner unregister --name "my-second-project-runner"
```

This removes the block from `config.toml` _and_ deletes the registration on GitLab's side. If `unregister` isn't available, delete the block manually from `config.toml` and restart the service — but then also remove the stale runner from the project's GitLab UI by hand.

## Resource monitoring / auto-cancel options (shared host)

1. **Hard caps** via `[runners.docker] cpus` / `memory` — prevents over-use rather than detecting it.
2. **In-job logging** (`free -h`, `uptime`, etc. in the job script) — visible in job log, manual cancel via GitLab UI.
3. **Host-level watcher script** — checks load/memory and calls the GitLab API to auto-cancel the pipeline if a threshold is crossed.

## Which user runs the jobs

- **`shell` executor**: runs as the OS user the `gitlab-runner` service runs as (usually the `gitlab-runner` system user, not your own account).
- **`docker` executor**: host side still uses the `gitlab-runner` system user (to talk to the Docker daemon); _inside_ the container it runs as the image's default user (often `root`) unless overridden with `[runners.docker] user = "uid:gid"`.

Check with:

```bash
sudo systemctl show gitlab-runner -p User
docker run --rm nexus.example.com/rtl-toolchain:latest whoami
```

## Checking if a job is running on a runner

**From the host:**

```bash
ps aux | grep gitlab-runner
docker ps                                                  # docker executor: running job containers
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
```

Job containers are typically named `runner-<short-token>-project-<id>-concurrent-<n>` — match the short token against `config.toml` to tell runners apart.

**From logs (live):**

```bash
sudo journalctl -u gitlab-runner -f
```

Shows job pickup (`received job=...`) and completion lines as they happen.

**From GitLab UI:**

- Project → **Build → Jobs** (or **CI/CD → Pipelines**) — job status, live log, and which runner picked it up.
- Settings → CI/CD → Runners → click the specific runner → **Jobs** — most precise way to check a specific runner when more than one is registered on the same host.

## Docker group

```bash
getent group docker        # who's currently in it
sudo usermod -aG docker $USER
newgrp docker               # apply in current shell, or log out/in
```

Note: this only affects commands _you_ run manually — the runner service already talks to Docker as its own system user.