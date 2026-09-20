# CI Certificates & Secrets Management

Notes from setting up `base_ubuntu` for GitLab CI and local runs with `gitlab-ci-local`.

## 1. CA Certificates

`ca-certificates` (already installed in `base_ubuntu.Dockerfile`) provides the root CA bundle Ubuntu uses to verify HTTPS/TLS connections — needed by `git clone`, `pip install`, `wget`, `apt`, etc.

The Dockerfile also copies a custom CA cert, useful only if something in the pipeline sits behind a corporate/self-signed TLS proxy:

```dockerfile
COPY temp/ca-certificates.crt /usr/local/share/ca-certificates/ca-certificates.crt
RUN update-ca-certificates
```

`build.sh` populates `temp/ca-certificates.crt` from the _build host_:

```bash
mkdir -p temp
cp /etc/ssl/certs/ca-certificates.crt temp/ca-certificates.crt
```

**In GitLab CI:** cloning over HTTPS against the GitLab instance itself does **not** need this custom CA step — GitLab's own certificate is already signed by a public CA covered by the base `ca-certificates` package. The custom CA copy only matters for other endpoints behind a private proxy (internal pip index, etc.).

## 2. HTTPS clone instead of SSH keys

GitLab CI injects a short-lived `CI_JOB_TOKEN` automatically, which authenticates HTTPS git operations against the same project/group — no SSH agent, private key, or `.ssh` mount required:

```
https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.example.com/group/project.git
```

This replaces the local-dev pattern in `run.sh` (`-v "$HOME/.ssh:$DOCKER_MOUNT_DIR/.ssh:ro"`), which only applies when running the image manually — GitLab CI never calls `run.sh`.

## 3. Running the same job locally with `gitlab-ci-local`

[`gitlab-ci-local`](https://github.com/firecow/gitlab-ci-local) runs `.gitlab-ci.yml` jobs in Docker on your machine, but it has no real GitLab pipeline behind it — `CI_JOB_TOKEN` isn't automatically valid. Supply a substitute token via a variable:

```bash
gitlab-ci-local --variable CI_JOB_TOKEN=<token> <job>
```

Or via a gitignored `.gitlab-ci-local-variables.yml` in the repo root (same directory as `.gitlab-ci.yml`):

```yaml
CI_JOB_TOKEN: "glpat-xxxxxxxxxxxx"
```

Add it to `.gitignore` — never commit a real token.

## 4. Getting a token

Use a **Personal Access Token (PAT)** scoped to `read_repository` only, with an expiration set:

- GitLab UI: avatar → **Edit profile** → **Access Tokens**
- API: `GET /api/v4/personal_access_tokens` (with a valid token in the `PRIVATE-TOKEN` header)

For a team, each colleague generating their **own** PAT is safer than sharing one Project/Deploy Token — individually revocable, no shared secret to rotate.

## 5. Storing the token safely (`pass`)

`pass` (GPG-encrypted password store) works consistently across Linux, macOS, and WSL, including headless machines — unlike `secret-tool`, which needs a desktop keyring daemon.

### One-time setup

```bash
sudo apt-get update && sudo apt-get install -y pass gnupg

# Generate a key (choose "(1) RSA and RSA", 4096 bits, when prompted)
gpg --full-generate-key

# Find the key ID or use the email you registered
gpg --list-secret-keys --keyid-format=long

# Initialize the store with that key
pass init <key-id-or-email>
```

To redo a key from scratch:

```bash
gpg --delete-secret-keys <key-id>
gpg --delete-keys <key-id>
```

### Store and use the token

```bash
# Store once (prompts for the value)
pass insert gitlab/ci-job-token

# Verify
pass gitlab/ci-job-token

# Use with gitlab-ci-local
gitlab-ci-local --variable CI_JOB_TOKEN=$(pass gitlab/ci-job-token) <job>
```

## 6. Wrapper script pattern

```bash
#!/bin/bash
set -euo pipefail

if ! command -v pass &>/dev/null; then
    echo "Error: 'pass' is not installed. See setup docs." >&2
    exit 1
fi

if ! CI_JOB_TOKEN="$(pass gitlab/ci-job-token 2>/dev/null)"; then
    echo "Error: could not read 'gitlab/ci-job-token' from pass." >&2
    echo "Run: pass insert gitlab/ci-job-token" >&2
    exit 1
fi

gitlab-ci-local --variable CI_JOB_TOKEN="$CI_JOB_TOKEN" "$@"
```

Never add `set -x` above the token read/use — it would print the token to the terminal or CI log.

## 7 `CI_JOB_TOKEN` not applied to extra repo clones

### Theoretical reason

`CI_JOB_TOKEN` is a short-lived, per-pipeline token that GitLab generates automatically for every job. On a real runner, GitLab uses it to authenticate exactly one thing transparently: the checkout of the pipeline's _own_ repository. The runner does this by rewriting that repo's remote URL (or injecting an HTTP header) behind the scenes before running your script — so a job's implicit checkout "just works" without the token ever being visible in your `.yml`.

That rewriting is scoped to the one repo the pipeline belongs to. It is **not** a general git credential — any `git clone` your script runs itself, against any other repository, gets no authentication unless you configure it yourself. The token still has the same access scope: by default it can only read repos in the _same_ GitLab project, and can reach other projects only if those projects explicitly allow-list this one under _Settings → CI/CD → Token Access_.

So the underlying reason "the token is not made explicit" in normal CI usage is not that the token isn't required — it's that GitLab pre-wires it for the primary checkout only, and every additional clone falls back to needing that same setup done by hand.

### Problem

The `REPO_PY_SIM_DEPEND` clone loop does a bare `git clone` against each dependency repo, with no credentials attached:

```bash
for entry in ${REPO_PY_SIM_DEPEND}; do
  repo="${entry%@*}"
  branch="${entry##*@}"
  name=$(basename "${repo}" .git)
  git clone --depth 1 --branch "${branch}" --single-branch "${repo}.git" "${name}"
  ...
done
```

`CI_JOB_TOKEN` was assumed to authenticate these clones the same way it authenticates the pipeline's own checkout. It doesn't — resulting in an authentication failure against each dependency repo.

### Chosen solution

Set up URL rewriting once, before the loop runs, so every subsequent plain `git clone https://<host>/...` in the job picks up the token automatically — replicating what GitLab's own checkout mechanism does, without touching the loop or embedding credentials in `REPO_PY_SIM_DEPEND` entries:

```bash
git config --global url."https://gitlab-ci-token:${CI_JOB_TOKEN}@<host>/".insteadOf "https://<host>/"
```

Placed as the first line of the job's `script:` (or in `before_script:` — both run in the same shell for a given job). Works identically on a real runner, where `CI_JOB_TOKEN` is auto-injected, and locally via `gitlab-ci-local`, where `run.sh` supplies it explicitly via `--variable CI_JOB_TOKEN=...` sourced from `pass` — same code, different source for the value.

### Alternatives considered

1. **`~/.netrc` entry** — same transparency via git's standard HTTP credential mechanism instead of a git-specific config rewrite:
    
    ```bash
    cat >> ~/.netrc <<EOFmachine <host>login gitlab-ci-tokenpassword ${CI_JOB_TOKEN}EOFchmod 600 ~/.netrc
    ```
    
2. **`http.extraHeader`** — same one-line setup, sends the token as a Bearer header instead of Basic-auth-in-URL, so the credential never appears as `user:pass@` in any URL:
    
    ```bash
    git config --global http."https://<host>/".extraHeader "Authorization: Bearer ${CI_JOB_TOKEN}"
    ```
    
1. **Embed the token directly per-URL inside the loop** (`https://gitlab-ci-token:${CI_JOB_TOKEN}@${repo#https://}.git`) — works, but couples credentials into the clone line itself instead of centralizing them in one setup step; rejected in favor of the rewrite approach above.