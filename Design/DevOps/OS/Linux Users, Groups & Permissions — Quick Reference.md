Notes from checking a shared SSH deploy box (used for GitLab runners): who else has access, what groups grant, and how to audit it.

## Checking other users on a system

```bash
# All local accounts (system + human), from /etc/passwd
cut -d: -f1 /etc/passwd

# Just "real" login users (UID >= 1000 is the usual convention)
awk -F: '$3 >= 1000 && $3 < 65534 {print $1, $3, $6, $7}' /etc/passwd

# Who's currently logged in / has a session
who
w
last -n 20        # recent login history
```

## Checking a user's groups and sudo rights

```bash
groups <username>          # groups a user belongs to
id <username>              # UID, primary group, all supplementary groups

sudo -l -U <username>      # what <username> is allowed to run via sudo (needs root)
getent group sudo          # or 'wheel' on RHEL/Fedora — members who can sudo
cat /etc/sudoers /etc/sudoers.d/*   # full sudo rules (needs root to read)
```

## Checking GitLab runners on the box

```bash
sudo gitlab-runner list                    # runners registered locally
sudo cat /etc/gitlab-runner/config.toml    # runner config
sudo systemctl status gitlab-runner        # service status
sudo journalctl -u gitlab-runner -f        # live logs
ps aux | grep gitlab-runner                # which OS user the runner runs as
```

For fleet-wide visibility (status, tags, last contact, jobs across many runners), the GitLab web UI is usually easier: **Admin Area → CI/CD → Runners** (instance-wide) or **Settings → CI/CD → Runners** (project/group level).

---

## What the Linux group feature is for

Groups bundle users together so permissions can be granted to the whole bundle instead of managing access user by user.

- **File/directory access control.** Every file has an owner (user) and a group, with separate `rwx` permissions for owner / group / other. Put collaborators in a shared group, `chown` the directory to that group, and set group permissions — no need to touch per-file ACLs individually.
- **Granting privileges without full root.** Membership in certain groups unlocks specific capabilities without sudo: `sudo`/`wheel` (run sudo), `docker` (talk to the Docker socket — effectively root-equivalent), `adm` (read system logs), `dialout` (serial ports), etc. Software often creates a dedicated group to gate access to a resource it manages.
- **Primary vs. supplementary groups.** Each user has one _primary group_ (in `/etc/passwd`, used by default for new files) and any number of _supplementary groups_ (in `/etc/group`). `id <user>` shows both.
- **Service isolation.** Daemons often run as a dedicated user+group (e.g. `gitlab-runner:gitlab-runner`) so their files/processes stay separated from the rest of the system unless someone is explicitly added to that group.

### Common commands

```bash
getent group <groupname>          # list members of a group
groups <username>                 # groups a user belongs to
sudo usermod -aG <group> <user>   # add a user to a supplementary group
sudo groupadd <name>              # create a new group
chgrp <group> <file>              # change a file's group ownership
chmod g+rwx <file>                # grant the group read/write/execute
```

---

## How to see what a specific group actually grants

There's no single command that dumps everything a group unlocks — check these in combination:

**1. Files/directories owned by the group**

```bash
sudo find / -group <groupname> -exec ls -ld {} \; 2>/dev/null
# or scoped to a likely area, much faster:
find /opt /etc /var -group <groupname> -exec ls -ld {} \;
```

**2. Sudo rules tied to the group** (prefixed with `%` in sudoers)

```bash
sudo grep -rn "%<groupname>" /etc/sudoers /etc/sudoers.d/
```

**3. "Magic" system groups with built-in meaning** — not discoverable by searching files; the daemon/tool itself checks membership in code:

|Group|Grants|
|---|---|
|`sudo` / `wheel`|full sudo access|
|`docker`|access to `/var/run/docker.sock` (root-equivalent)|
|`adm`|read `/var/log`|
|`systemd-journal`|read journal logs without sudo|
|`dialout`|access serial/TTY devices|
|`shadow`|read `/etc/shadow` (password hashes)|

Check `man <groupname>` or the relevant service's docs if unsure.

**4. POSIX ACLs** (finer-grained, independent of the file's actual group-owner field)

```bash
getfacl <file_or_dir>
# look for lines like: group:<groupname>:rwx
```

### Example: auditing a specific group (`gitlab-runner`)

```bash
getent group gitlab-runner
find /etc/gitlab-runner /opt -group gitlab-runner -exec ls -ld {} \;
sudo grep -rn "%gitlab-runner" /etc/sudoers /etc/sudoers.d/
```