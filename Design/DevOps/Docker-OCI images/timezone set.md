# Setting the Timezone in Docker

## The default you should question first

Before picking one of the options below: most production containers are better off left in UTC. Servers, logs, databases, and cron-like schedulers are easier to reason about and debug across hosts, regions, and daylight-saving changes when everything internally agrees on UTC, with timezone conversion happening only at the edges — in the UI, in a report, in a notification — where a human actually needs to read a local time. This is why most official base images ship in UTC and don't try to guess a "correct" zone for you.

Configuring a specific timezone inside a container is the right call when the thing running _inside_ the container itself needs to reason in local wall-clock time — a cron job that must fire at "9am local business hours," a batch process whose output filenames or logs are read by non-technical people in one timezone, or a legacy app that was never built to be timezone-aware internally.

If you do need one, here are the options, in roughly the order people reach for them.

## Option 1 — Bake it into the Dockerfile (build time)

```dockerfile
ENV TZ=Europe/Madrid
RUN apt-get update && apt-get install -y tzdata \
    && ln -snf /usr/share/zoneinfo/$TZ /etc/localtime \
    && echo $TZ > /etc/timezone \
    && apt-get clean
```

(swap `apt-get` for `apk add --no-cache tzdata` on Alpine, or `dnf install -y tzdata` on RHEL/Rocky)

**Trade-offs:** fully self-contained and reproducible — the image behaves the same everywhere it's run, with no extra flags needed. Downside: the zone is frozen into the image, so changing it means rebuilding and redeploying. Also adds a small layer/size cost for `tzdata` if it wasn't already present.

## Option 2 — Pass it at `docker run` (runtime)

```bash
docker run -e TZ=Europe/Madrid yourimage
```

**Trade-offs:** no rebuild needed to change zones, and the same image can run as different zones for different customers/environments. The catch: this _only_ works if `tzdata` is already installed in the image — setting `TZ` alone does nothing if there's no zoneinfo database to resolve it against, and some apps will silently fall back to UTC rather than error. In practice this option is usually combined with Option 1 (bake `tzdata` in, but leave `ENV TZ` out or overridable).

## Option 3 — Mount the host's time files in

```bash
docker run \
  -v /etc/localtime:/etc/localtime:ro \
  -v /etc/timezone:/etc/timezone:ro \
  yourimage
```

**Trade-offs:** zero image changes, and the container always matches whatever the host is set to. Downside: it ties the container's behavior to the specific host it happens to land on — bad for portability (doesn't work the same on a teammate's machine, in CI, or across a cluster where nodes might differ), and breaks silently if those host paths don't exist (common on minimal/other-OS hosts). Mostly seen in local dev, rarely in anything shipped.

## Option 4 — Derive it from the host dynamically

```bash
docker run -e TZ=$(timedatectl show --property=Timezone --value) yourimage
```

**Trade-offs:** a middle ground between Options 2 and 3 — portable like Option 2 (just an env var, works on any host with `tzdata` in the image), but doesn't require hardcoding a zone name. Useful for dev tooling or CI runners where you want the container to "match whatever this machine is." Still needs `tzdata` inside the image, and only resolves correctly on hosts that actually have `timedatectl` (systemd) or another reliable way to read their own zone — this can break in minimal containers or non-systemd hosts, so it's less predictable than hardcoding a zone explicitly.

## Comparison

|Option|Needs rebuild to change zone?|Needs `tzdata` in image?|Portable across hosts?|Typical use|
|---|---|---|---|---|
|1. Bake into Dockerfile|Yes|Yes (installed here)|Yes|Production images with one fixed, known zone|
|2. `-e TZ=...` at runtime|No|Yes (pre-installed)|Yes|Same image, different zone per environment|
|3. Mount host files|No|No|No — host-dependent|Local development only|
|4. `-e TZ=$(host's zone)`|No|Yes (pre-installed)|Yes, if host can report its own zone|Dev tooling, CI, "match my machine" scripts|

## What people normally do

In practice, most teams land on one of two patterns:

1. **Leave the container in UTC and never touch timezone config at all**, converting to local time only at the display/reporting layer. This is the most common choice for web services, APIs, and anything where logs get aggregated across machines/regions.
2. **Bake `tzdata` + a fixed `ENV TZ` into the Dockerfile** (Option 1), for the cases above where the app genuinely needs to think in local time — often paired with leaving `TZ` overridable via `-e` at runtime (Option 2) so the same base image can serve multiple deployments without rebuilding.

Options 3 and 4 are common in local dev and ad hoc scripting, but rarely make it into anything deployed, since both depend on properties of the host machine rather than the image itself.