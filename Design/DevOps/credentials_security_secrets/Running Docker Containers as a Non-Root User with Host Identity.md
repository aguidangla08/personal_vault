## Purpose

For security reasons, the container does not run as `root`. The image is built with an unprivileged user, and at run time the container adopts the **host user's UID, GID and supplementary groups**. This limits the damage of a container compromise and keeps file and device permissions consistent between host and container.

## 1. Build time: define an unprivileged user

The image is built with these arguments:

```
USER_ID=1001
GROUP_ID=1001
USER_NAME=user
```

These create a default non-root user (`user`, UID/GID 1001) inside the image, so that even if no `--user` flag is passed, the container does not fall back to root.

## 2. Run time: adopt the host identity

```bash
# Capture host identity once
USER_UID="$(id -u)"
USER_GID="$(id -g)"

# Replicate every supplementary group (video/render for GPU, etc.)
# so device permissions and any group-based file access carry over intact
GROUP_ADD_ARGS=()
for gid in $(id -G); do
    GROUP_ADD_ARGS+=(--group-add "$gid")
done
```

Then, in the `docker run` command:

```bash
--user "${USER_UID}:${USER_GID}" \
    "${GROUP_ADD_ARGS[@]}" \
    -v /etc/passwd:/etc/passwd:ro \
    -v /etc/group:/etc/group:ro \
```

## 3. What each piece does

|Element|Purpose|
|---|---|
|`--user "${USER_UID}:${USER_GID}"`|Runs the main process with the host user's UID/GID instead of root or the image default. Files created in bind-mounted volumes are owned by the host user, avoiding root-owned files and `chmod`/`chown` problems.|
|`--group-add "$gid"` (loop over `id -G`)|Adds every supplementary group of the host user. This keeps access to group-protected resources such as `/dev/dri` and `/dev/nvidia*` (groups like `video`, `render`), `docker`, shared data directories, etc.|
|`-v /etc/passwd:/etc/passwd:ro`|Lets the container resolve the host UID to a real username and home directory (tools like `whoami`, `ssh`, `git`, `sudo` and some libraries fail on an unknown UID).|
|`-v /etc/group:/etc/group:ro`|Resolves group IDs to names, so the supplementary groups appear correctly.|
|`:ro` on both mounts|The container can read but never modify the host's account databases.|

## 4. Security benefits

- **No root inside the container**: a process escape or a vulnerable dependency does not immediately yield root privileges.
- **Least privilege**: the container has exactly the permissions of the invoking host user, no more. See https://gitlab.com/gitlab-org/gitlab/-/work_items/23046.
- **Correct device access**: replicating supplementary groups gives GPU and other device access without `--privileged`.
- **Consistent file ownership**: outputs written to mounted volumes belong to the host user.
- **Read-only identity files**: the container cannot tamper with host users or groups.

## 5. Caveats

- Mounting `/etc/passwd` and `/etc/group` exposes the host's list of usernames and groups to the container. This is low risk (the files contain no password hashes), but it is information disclosure. On hosts using LDAP/SSO/`sssd`, users not present in the local files will not resolve.
- The image's build-time user (UID 1001) is effectively overridden by `--user`, and the mounted `/etc/passwd` will not contain it. This is fine as long as the container is always launched through the run script.
- The user's home directory from the host's `/etc/passwd` may not exist inside the container. Mount it or set `HOME` explicitly to a writable path (e.g. `-e HOME=/tmp`) if a tool needs it.
- Supplementary groups grant real access. Only pass through groups the workload needs if the host user belongs to privileged groups (e.g. `docker`, `sudo`).
- Consider pairing this setup with `--cap-drop=ALL` and `--security-opt no-new-privileges` for further hardening.