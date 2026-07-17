# Steps
## 1. Generate an SSH key per account

```bash
ssh-keygen -t ed25519 -C "your_email@example.com" -f ~/.ssh/id_ed25519_github_personal
```

Repeat for each account (e.g. `id_ed25519_github_work`). Then add each `.pub` key to the matching GitHub account: **Settings → SSH and GPG keys → New SSH key**.

## 2. Configure SSH host aliases — `~/.ssh/config`

```
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github_personal
    IdentitiesOnly yes

Host github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github_work
    IdentitiesOnly yes

Host gitlab-1.company.x gitlab-2.company.x
    User git
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

- `IdentityFile` must point to the **private** key (no `.pub`).
- `IdentitiesOnly yes` forces SSH to only try the specified key.
- `chmod 600 ~/.ssh/config` and `chmod 600` on private keys, `644` on `.pub` files.

Test each alias:

```bash
ssh -T git@github-personal
ssh -T git@github.com
```

## 3. Organize repos by folder, one folder per account

```
~/projects/.../github-personal/
~/projects/.../github-work/
```

## 4. Main config — `~/.gitconfig`

Keep this file to **only** the default identity + includes (no stray `[user]` blocks after them — they override everything).

```ini
[user]
    name = Default Name
    email = default@example.com

[includeIf "gitdir:~/projects/.../github-personal/"]
    path = ~/.gitconfig-github-personal

[includeIf "gitdir:~/projects/.../github-work/"]
    path = ~/.gitconfig-github-work
```

## 5. Per-account config files

`~/.gitconfig-github-personal`:

```ini
[user]
    name = Your Personal Name
    email = you@personal-email.com

[url "git@github-personal:"]
    insteadOf = git@github.com:
```

`~/.gitconfig-github-work`:

```ini
[user]
    name = Your Work Name
    email = you@company.x
```

> The `url.insteadOf` trick lets you clone with the normal `git@github.com:...` URL — git rewrites it to the right host alias automatically based on which folder you're in.

## 6. Clone into the right folder

```bash
mkdir -p ~/projects/.../github-personal/
cd ~/projects/.../github-personal/
git clone git@github.com:username/repo.git
```

## 7. Verify

```bash
git config --show-origin user.email   # confirms which file supplied it
git config --show-origin user.name
GIT_SSH_COMMAND="ssh -v" git fetch 2>&1 | grep "identity file"   # confirms which key was used
```

## Notes / gotchas

- `includeIf gitdir:` matches paths **after symlink resolution** — if any parent dir is a symlink, matching can silently fail.
- `gitdir:` patterns should end with `/` (auto-treated as `/**`).
- Wildcards in SSH `Host` (e.g. `gitlab*`) match first-in-file, so put specific hosts before wildcards.
- `remote -v` still displays the original `git@github.com:...` URL even after rewrite — that's expected.