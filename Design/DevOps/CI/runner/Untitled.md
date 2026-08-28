# GitLab Runners: Overview, Setup, and Configuration

## What a runner is and how it fits into CI/CD

A GitLab Runner is a lightweight, standalone application (written in Go, single binary, no real dependencies) that executes the jobs defined in your `.gitlab-ci.yml`. It doesn't live inside GitLab itself — it's a separate process that polls GitLab for jobs, runs them using whatever "executor" you've configured, and reports results and logs back.

GitLab.com provides shared/hosted runners you can use out of the box. For self-managed control (custom hardware, network access, specific images, cost control) you install your own runners at the instance, group, or project level.

## Installing GitLab Runner

**Linux (Debian/Ubuntu), via the official repo:**

```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install gitlab-runner
```

**As a Docker container** (runner itself runs in a container, regardless of what executor it uses for jobs):

```bash
docker run -d --name gitlab-runner --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
```

**On Kubernetes**, via the Helm chart:

```bash
helm repo add gitlab https://charts.gitlab.io
helm install --namespace gitlab-runner --create-namespace gitlab-runner gitlab/gitlab-runner \
  --set gitlabUrl=https://gitlab.example.com/ \
  --set runnerRegistrationToken=<YOUR_TOKEN>
```

## Registering a runner

Create a runner in the GitLab UI (Settings → CI/CD → Runners → "New project/group runner") to get an authentication token, then register it on the machine:

```bash
sudo gitlab-runner register \
  --url "https://gitlab.example.com/" \
  --token "glrt-XXXXXXXXXXXXXXXXXXXX" \
  --executor "docker" \
  --docker-image "alpine:latest" \
  --description "my-project-runner"
```

This is non-interactive (flags supplied); omit the flags and it will prompt you step by step for the GitLab URL, token, description, tags, and executor.

## Configuring the runner (`config.toml`)

Everything lives in `config.toml`, usually at `/etc/gitlab-runner/config.toml`. Each `[[runners]]` block is one runner instance; you can define several in the same file to run multiple configs from one installed binary.

```toml
concurrent = 4
check_interval = 0

[[runners]]
  name = "my-project-runner"
  url = "https://gitlab.example.com/"
  token = "glrt-XXXXXXXXXXXXXXXXXXXX"
  executor = "docker"

  [runners.docker]
    image = "alpine:latest"
    privileged = false
    volumes = ["/cache"]
    tls_verify = false

  [runners.cache]
    Type = "s3"
    Shared = true
```

Key fields worth knowing:

- `concurrent` (top level) — max jobs this runner can run in parallel across all its `[[runners]]` blocks.
- `executor` — see below, this is what decides "Docker or not."
- `[runners.docker]` — only relevant when `executor = "docker"`.
- `[runners.cache]` — where build cache is stored (local, S3, GCS, etc.).

## Docker images or not: the executor choice

This is the main lever. It's the `executor` setting in each `[[runners]]` block.

### `docker` executor — jobs run inside containers

Each job gets a fresh, isolated container. Set a default image in `config.toml`:

```toml
[runners.docker]
  image = "my.registry.tld:5000/alpine:latest"
  privileged = false
  volumes = ["/cache"]
```

A job in `.gitlab-ci.yml` can override that default per job (this takes precedence over the `config.toml` default):

```yaml
build-job:
  image: node:20
  script:
    - npm install
    - npm run build

test-job:
  image: python:3.12-slim
  script:
    - pip install -r requirements.txt
    - pytest
```

Benefits: clean, reproducible, disposable environments; you can test locally with the same image (`docker run -it node:20 bash`); no dependency leakage between jobs.

### `shell` executor — jobs run directly on the host

No containers at all. Register it like this:

```bash
sudo gitlab-runner register \
  --url "https://gitlab.example.com/" \
  --token "glrt-XXXXXXXXXXXXXXXXXXXX" \
  --executor "shell" \
  --description "bare-metal-runner"
```

```toml
[[runners]]
  name = "bare-metal-runner"
  url = "https://gitlab.example.com/"
  token = "glrt-XXXXXXXXXXXXXXXXXXXX"
  executor = "shell"
```

Simpler and faster (no container startup overhead), but the host's installed tools/versions leak into every job, and jobs aren't isolated from each other or from the runner host.

### Other executors

- `kubernetes` — each job runs as a pod in a cluster; good for autoscaling and multi-tenant CI.
- `docker+machine` / `docker-autoscaler` / `instance` — autoscaling fleets of Docker hosts, spun up on demand (older Docker Machine approach is being phased out in favor of the newer autoscaler/fleeting approach).
- `docker-windows` — Docker executor for Windows containers.
- `ssh` — runs jobs on a remote machine over SSH, no local install of the executed environment required on the runner host itself.
- `custom` — you provide your own scripts for each lifecycle stage (prepare/run/cleanup).

Minimal Kubernetes executor example:

```toml
[[runners]]
  name = "k8s-runner"
  url = "https://gitlab.example.com/"
  token = "glrt-XXXXXXXXXXXXXXXXXXXX"
  executor = "kubernetes"

  [runners.kubernetes]
    namespace = "gitlab-runner"
    image = "alpine:latest"
```

## Quick reference: register flags cheat sheet

```bash
gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.example.com/" \
  --token "glrt-XXXXXXXXXXXXXXXXXXXX" \
  --executor "docker" \
  --docker-image "alpine:latest" \
  --tag-list "docker,linux,x86_64" \
  --run-untagged="true" \
  --locked="false"
```

## A note on "Cerberos"

Nothing in GitLab's runner documentation, blog, or ecosystem uses that name. Possibilities worth checking: **Kerberos** (the network authentication protocol, which GitLab supports for Git auth in some Enterprise setups, but it's unrelated to runners), a mishearing of a third-party or internal tool called "Cerberus," or something specific to wherever you first heard the term. If you can share the context, happy to dig further.

## Sources

- [GitLab Runner docs](https://docs.gitlab.com/runner/)
- [Docker executor | GitLab Docs](https://docs.gitlab.com/runner/executors/docker/)
- [Run your CI/CD jobs in Docker containers | GitLab Docs](https://docs.gitlab.com/ci/docker/using_docker_images/)
- [Install and register GitLab Runner for autoscaling with Docker Machine | GitLab Docs](https://docs.gitlab.com/runner/executors/docker_machine/)