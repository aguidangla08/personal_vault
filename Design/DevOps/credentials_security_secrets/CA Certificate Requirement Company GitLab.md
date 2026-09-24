# CA Certificate Requirement — Company GitLab

## Problem

Reaching the company GitLab URL over HTTPS involves a TLS certificate chain that terminates in our internal/company CA. Tools that validate against OpenSSL's trust store — `curl` and `wget` — fail TLS verification for that URL unless the CA is present in the trust store they're running in.

`git pull` / `git clone` against the same GitLab URL does **not** hit this problem. The reason isn't confirmed yet (see Challenges) — possibly a different transport (SSH) or a different trust path than `curl`/`wget` use locally — but the practical takeaway right now is: only `curl`/`wget` (and anything else linking against the system TLS store) are blocked without the CA; plain git operations against the same host are unaffected.

## Current state

For now, the fix is: copy the CA certificate(s) from my own machine's trust store into the Docker build context, and install them into the image (`build.sh` copies `/etc/ssl/certs/ca-certificates.crt` into `temp/`, and the Dockerfile `COPY`s it in and runs `update-ca-certificates`).

This works, but I don't know if it's the _right_ way to do it — it copies my entire host trust store rather than just the one company CA that's actually needed, and it depends on my personal machine having that CA correctly installed in the first place.

```bash
# TODO Temp (WORKS)
# Copy CA certificates from the host to ensure SSL/TLS connections inside the container
# can be verified properly (e.g., downloading packages securely over HTTPS).
#mkdir -p temp
#cp /etc/ssl/certs/ca-certificates.crt temp/ca-certificates.crt

# TODO Temp (DOESN'T WORK)
# Fetch the Sectigo intermediate certificate GitLab's server fails to send itself,
# so the image can build a complete trust chain to it. Fetched here (host-side)
# rather than inside the Dockerfile, since the Docker build's RUN steps hit the
# same network restrictions as the host (build.sh already uses --network host).
curl -fsS -o temp/sectigo-ov-r36.crt \
    http://crt.sectigo.com/SectigoPublicServerAuthenticationCAOVR36.crt
```

## Proposals

1. **Extract and copy only the company CA**, not the full host bundle. Identify it on the host (likely under `/usr/local/share/ca-certificates/`, or by inspecting the merged bundle with `openssl x509 -noout -subject -issuer` to find the entry that isn't a public CA), then copy just that single `.crt` file into the image under `/usr/local/share/ca-certificates/` and run `update-ca-certificates`. This adds to Ubuntu's default trust store instead of replacing it.
    
2. **Pass the CA in at build time via a BuildKit secret**, so it's used only transiently by `update-ca-certificates` and never written into the build context (`temp/`) or persisted in an image layer's history.
    
3. **Bake the CA into a shared base image once**, built and pushed separately from day-to-day `build.sh` runs, so every derived image `FROM`s a base that already trusts the company CA instead of re-copying and re-installing it on every build.
    

## Challenges / open questions

- **Why git is unaffected** — not confirmed whether this is because git clone/pull against this host uses SSH, uses a different trust store, or some other reason. Worth confirming so the fix is scoped to the tools that actually need it.
- **Current Dockerfile bug** — it `COPY`s the host bundle directly over `/usr/local/share/ca-certificates/ca-certificates.crt`, overwriting the merged trust file instead of adding an individual cert file for `update-ca-certificates` to pick up. Should be fixed regardless of which proposal above is chosen.
- **Identifying the right cert** — the host's `/etc/ssl/certs/ca-certificates.crt` bundle merges ~100+ public CAs with whatever internal CA was added; haven't yet pinned down exactly which entry is the company one.
- **Dependency on my personal machine** — copying from my host means the image's trust depends on my machine currently having the CA correctly installed, with no independent source of truth and no tracking of when the CA is rotated or expires.
- **No verification step yet** — haven't confirmed that installing the CA actually resolves the `curl`/`wget` failures end-to-end inside a built image.