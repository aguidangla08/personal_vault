## Plan prompt

To perform this steps, lets create a plan:
Requirements:
* tag for build image
* tag for base images if necessary
* org Extra data
   * commit
   * created 


Design details:
* I would like to save everything that is configurable in some external file


Questions:
* which are the org.opencontainers.image options and what they are used for


## Claude Sonnet 5 plan
This is the official, complete list per the OCI Image Spec:

|Key|Purpose|
|---|---|
|`org.opencontainers.image.created`|Date/time the image was built (RFC 3339 format)|
|`org.opencontainers.image.authors`|Contact info for the person/org responsible for the image|
|`org.opencontainers.image.url`|URL with more info about the image|
|`org.opencontainers.image.documentation`|URL to the image's documentation|
|`org.opencontainers.image.source`|URL to the source code used to build the image (e.g. your git repo)|
|`org.opencontainers.image.version`|Version of the packaged software — may match a git tag, SemVer-compatible|
|`org.opencontainers.image.revision`|Source control revision (commit hash) the image was built from|
|`org.opencontainers.image.vendor`|Name of the distributing org/individual|
|`org.opencontainers.image.licenses`|SPDX license expression for the contained software|
|`org.opencontainers.image.ref.name`|Reference name for a target — mostly relevant for OCI image-layout/index files, not typical Dockerfile use|
|`org.opencontainers.image.title`|Human-readable title|
|`org.opencontainers.image.description`|Human-readable description of what's in the image|
|`org.opencontainers.image.base.digest`|Digest of the base image this one was built from (`FROM` target)|
|`org.opencontainers.image.base.name`|Full reference (name:tag) of the base image — should be fully qualified, e.g. `registry.example.com/team/base-rocky:8-py3.12-v1`|

For your two requirements ("commit" and "created"), the direct mapping is `org.opencontainers.image.revision` and `org.opencontainers.image.created`.

---

## Plan

### 1. Requirements → label mapping

|Your requirement|Mechanism|
|---|---|
|Tag for build image|`docker build -t <name>:<tag>`|
|Tag for base image (if applicable)|Pinned in the `FROM` line, plus recorded via `org.opencontainers.image.base.name`|
|Commit|`org.opencontainers.image.revision` (build-arg, computed from git at build time)|
|Created|`org.opencontainers.image.created` (build-arg, computed as UTC timestamp at build time)|

### 2. Design — external config file

One config file per image, sitting next to its Dockerfile, holding everything that's **configurable and intentional** (not the commit/timestamp — those are _derived_, not configured, so they're computed by the build script, not stored).

**`base_rocky.image.env`**

```bash
IMAGE_NAME=base-rocky
IMAGE_TAG=8-py3.12-v1
PYTHON_VERSION=3.12
USER_NAME=user
```

**`onespin.image.env`**

```bash
IMAGE_NAME=onespin
IMAGE_TAG=2022.2_2-rocky8-v1
BASE_IMAGE=base-rocky
BASE_IMAGE_TAG=8-py3.12-v1
OneSpin_VERSION=2022.2_2
```

### 3. A generic build script that reads the config

**`build.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail

DOCKERFILE="$1"          # e.g. onespin.Dockerfile
ENV_FILE="$2"            # e.g. onespin.image.env

set -a
source "$ENV_FILE"
set +a

GIT_COMMIT=$(git rev-parse --short HEAD)
CREATED=$(date -u +%Y-%m-%dT%H:%M:%SZ)

# NEW: append short commit SHA to the tag for traceability at a glance
SHORT_SHA=$(git rev-parse --short=7 HEAD)
IMAGE_TAG="${IMAGE_TAG}-g${SHORT_SHA}"

BASE_ARG=""
if [[ -n "${BASE_IMAGE:-}" ]]; then
    BASE_ARG="--build-arg BASE_IMAGE_TAG=${BASE_IMAGE}:${BASE_IMAGE_TAG}"
fi

docker build \
    -f "$DOCKERFILE" \
    -t "${IMAGE_NAME}:${IMAGE_TAG}" \
    --build-arg GIT_COMMIT="$GIT_COMMIT" \
    --build-arg CREATED="$CREATED" \
    $BASE_ARG \
    $(env | grep -E '^[A-Z_]+_VERSION=' | sed 's/^/--build-arg /') \
    .
```

Usage:

```bash
./build.sh base_rocky.Dockerfile base_rocky.image.env
./build.sh onespin.Dockerfile onespin.image.env
```

### 4. Dockerfile side — consume the args, set the labels

```dockerfile
ARG GIT_COMMIT=unknown
ARG CREATED=unknown
ARG BASE_IMAGE_TAG=unknown

LABEL org.opencontainers.image.revision="${GIT_COMMIT}" \
      org.opencontainers.image.created="${CREATED}" \
      org.opencontainers.image.base.name="${BASE_IMAGE_TAG}" \
      org.opencontainers.image.source="https://your-git-remote/repo" \
      org.opencontainers.image.vendor="YourTeam"
```

And for the tool-specific image, add its own version too:

```dockerfile
ARG OneSpin_VERSION
LABEL org.opencontainers.image.version="${OneSpin_VERSION}"
```

### 5. Verifying it after the fact

```bash
docker inspect --format '{{json .Config.Labels}}' onespin:2022.2_2-rocky8-v1-g8f3a91c | jq
```

### Why this split (config file vs. computed args)

- **Config file** = things a human decides and commits to git (versions, tags, which base image) — reviewable in a diff.
- **Build-script-computed** = things that are only true "at this moment" (commit hash, timestamp) — never hand-edited, always derived, so they can't drift out of sync with reality.

This keeps your Dockerfiles themselves clean (just `ARG`/`LABEL`), keeps the actual version decisions in one small readable file per image, and keeps commit/timestamp accuracy automatic rather than something someone has to remember to update by hand.