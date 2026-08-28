# Docker Login & Secure Credential Storage

## Login to Nexus registry

```bash
docker login nexus-url
```

You'll be prompted for username/password (or token). The account must have push/pull permission on the target Docker hosted repository in Nexus.

## Problem: unencrypted credentials

By default, `docker login` stores credentials **base64-encoded (not encrypted)** in:

```
~/.docker/config.json
```

Example of the insecure entry:
```json
{
  "auths": {
    "nexus-url": {
      "auth": "base64-encoded-user:pass"
    }
  }
}
```

## Fix: use a credential helper

### 1. Install the helper for your OS

| OS      | Helper            |
|---------|--------------------|
| Linux   | `docker-credential-secretservice` or `docker-credential-pass` |
| macOS   | `docker-credential-osxkeychain` (usually preinstalled with Docker Desktop) |
| Windows | `docker-credential-desktop` (Docker Desktop default) |

Linux example:
```bash
sudo apt install golang-docker-credential-helpers
```

### 2. Update `~/.docker/config.json`

Remove any existing `"auths": {...}` block, and add:

```json
{
  "credsStore": "secretservice"
}
```

(Replace `secretservice` with `pass`, `osxkeychain`, or `desktop` depending on OS.)

### 3. Re-authenticate

```bash
docker login nexus-url
```

Credentials are now stored in your OS's secure keychain/credential manager instead of in plaintext.

## Verify

```bash
cat ~/.docker/config.json
```

You should see only `"credsStore"` — no `"auths"` block with visible credentials.