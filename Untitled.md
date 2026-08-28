### Changes to the plan

**Requirement → mechanism table** — add a row:

|Your requirement|Mechanism|
|---|---|
|Short commit SHA in tag|Appended automatically by `build.sh` (`-g<short-sha>`)|

**`build.sh`** — insert short-SHA tag suffix before the `docker build` call:

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

**Resulting example tags** (was `2022.2_2-rocky8-v1`, now):

```
base-rocky:8-py3.12-v1-g8f3a91c
onespin:2022.2_2-rocky8-v1-g8f3a91c
```

Everything else in the plan (config files, Dockerfile `ARG`/`LABEL` block, `docker inspect` verification) is unchanged.