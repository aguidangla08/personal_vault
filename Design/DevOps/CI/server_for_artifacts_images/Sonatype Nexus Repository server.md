# Nexus Repository Manager — Docker Guide

## What is it?

An artifact repository manager that stores build outputs and external dependencies. Nexus supports multiple package formats (PyPI, Docker/OCI, Raw, etc.) and acts as a **Docker registry**, so the standard Docker CLI works with it just like Docker Hub.

## Repository types

Repos have a `name`, `format` (pypi, docker, raw...) and `type`:

|Type|Purpose|
|---|---|
|**hosted**|Stores your organization's own artifacts|
|**proxy**|Caches artifacts from an external repository (e.g. Docker Hub)|
|**group**|Combines multiple repos behind one URL (virtual repository)|

Example: a `docker` group repo can front both `docker-hosted` (your images) and `docker-proxy` (cached public images) so users only need one endpoint.

## Access

- Web UI
- Package managers (docker CLI, pip, etc.)
- REST API

## REST API basics

```bash
# Service up?
curl -I https://nexus.company.com

# REST API up?
curl https://nexus.company.com/service/rest/v1/status

# List repositories
curl https://nexus.company.com/service/rest/v1/repositories
```

## How registry selection works

A Docker image reference is fully qualified: `<registry>/<repository>/<image>:<tag>`. The registry is part of the **image name**, not something you "connect" to separately.

- No registry in the name → Docker assumes Docker Hub (`docker.io/library/...`)
- `docker login REGISTRY` just stores credentials for that host in `~/.docker/config.json` — you can be logged into several registries at once
- `docker pull`/`docker push` always use the registry embedded in the image name
- `docker tag` rewrites the reference to point at a different registry

```bash
docker login nexus.company.com
docker login ghcr.io
# both sets of credentials are remembered; push/pull picks the right one
# based on the image name you use.
```

## Core Docker workflow with Nexus

```bash
# 1. Build locally (image has no registry yet)
docker build -t nvc_ubuntu:r1.17.1-v1 .

# 2. Tag with the Nexus registry + repo path
docker tag nvc_ubuntu:r1.17.1-v1 \
    nexus.company.com/docker-hosted/nvc_ubuntu:r1.17.1-v1

# 3. Push
docker push nexus.company.com/docker-hosted/nvc_ubuntu:r1.17.1-v1

# 4. Pull (anyone with access)
docker pull nexus.company.com/docker-hosted/nvc_ubuntu:r1.17.1-v1
```

If the image was already built/named with the full Nexus path, tagging isn't needed — push directly.

## Checking the catalog and tags

List all repositories/images in the registry:

```bash
curl -u user:pass https://nexus.company.com/v2/_catalog
```

List tags for a specific image:

```bash
curl -u user:pass https://nexus.company.com/v2/docker-hosted/nvc_ubuntu/tags/list
```

## Erasing images

**Local (host):**

```bash
docker rmi nexus.company.com/docker-hosted/nvc_ubuntu:r1.17.1-v1
```

**On Nexus (remote)** — two equivalent options:

- _UI:_ Browse → the Docker repo → expand the image → select the specific tag → Delete.
- _REST API:_

```bash
# find the componentId for a specific tag/version
curl -u admin:pass "https://nexus.company.com/service/rest/v1/search?repository=docker-hosted&name=nvc_ubuntu&version=r1.17.1-v1"

# delete just that tag
curl -u admin:pass -X DELETE "https://nexus.company.com/service/rest/v1/components/<componentId>"
```

Deleting one tag doesn't affect other tags of the same image. Blob storage isn't freed immediately — run Nexus's **cleanup / compact blob store** task afterward to reclaim disk space.

## CI usage example

Instead of every developer building images locally, publish once and consume from CI:

```yaml
image: nexus.company.com/docker-hosted/nvc_ubuntu:r1.17.1-v1
```

This turns container images into versioned, centrally managed artifacts reusable across projects and pipelines.

## Securing credentials

By default `docker login` stores credentials **base64-encoded, not encrypted**, in `~/.docker/config.json`. Use an OS credential helper (`osxkeychain`, `secretservice`/`pass`, `desktop`) instead — see the separate credentials doc for setup steps.