```bash
#!/usr/bin/env bash

set -uo pipefail

  

# --- fill these in for your setup ---

GITLAB_HOST="<company-gitlab-url>"

TEST_REPO="https://${GITLAB_HOST}/<repository-to-test-path>"

TEST_BRANCH="master"

GITLAB_API_URL="https://${GITLAB_HOST}/api/v2/version" # or any lightweight endpoint you can reach

# If you serve Python packages through GitLab's own package registry, put that URL here too:

GITLAB_PYPI_INDEX="" # e.g. https://${GITLAB_HOST}/api/v4/projects/<id>/packages/pypi/simple

# -------------------------------------

  

pass() { echo "PASS: $1"; }

fail() { echo "FAIL: $1"; }

  

echo "=== 1) TLS handshake (system trust store, no custom CA) ==="

tls_result=$(echo | openssl s_client -connect "${GITLAB_HOST}:443" -servername "${GITLAB_HOST}" -CApath /etc/ssl/certs 2>&1 | grep "Verify return code")

echo "${tls_result}"

echo "${tls_result}" | grep -q "Verify return code: 0" && pass "openssl trusts ${GITLAB_HOST}" || fail "openssl does not trust ${GITLAB_HOST}"

  

echo

echo "=== 2) git clone (OpenSSL-backed, same trust store as openssl above) ==="

rm -rf /tmp/ca-verify-clone

git clone --depth 1 --branch "${TEST_BRANCH}" --single-branch "${TEST_REPO}" /tmp/ca-verify-clone 2>&1 \

&& pass "git clone succeeded" || fail "git clone failed"

  

echo

echo "=== 3) curl against GitLab REST API (also OpenSSL-backed) ==="

if curl -fsS "${GITLAB_API_URL}" > /tmp/curl-verify.json 2>/tmp/curl-verify.err; then

pass "curl reached ${GITLAB_API_URL}"

else

fail "curl failed — $(cat /tmp/curl-verify.err)"

fi

  

echo

echo "=== 4) wget against the same host ==="

if wget -q --spider "https://${GITLAB_HOST}" 2>/tmp/wget-verify.err; then

pass "wget reached https://${GITLAB_HOST}"

else

fail "wget failed — $(cat /tmp/wget-verify.err)"

fi

  

echo

echo "=== 5) pip install (uses certifi's bundled CA by default, NOT the system store) ==="

# Public PyPI test — confirms pip/requests trust public CAs generally.

if pip install --no-cache-dir --target /tmp/pip-verify --quiet requests 2>/tmp/pip-verify.err; then

pass "pip install from public PyPI succeeded"

else

fail "pip install from public PyPI failed — $(cat /tmp/pip-verify.err)"

fi

  

if [ -n "${GITLAB_PYPI_INDEX}" ]; then

echo

echo "=== 5b) pip against GitLab's own package registry (the real test if you use this) ==="

if pip install --no-cache-dir --target /tmp/pip-verify-gitlab --index-url "${GITLAB_PYPI_INDEX}" --quiet some-internal-package 2>/tmp/pip-gitlab-verify.err; then

pass "pip install against GitLab's PyPI index succeeded"

else

fail "pip install against GitLab's PyPI index failed — $(cat /tmp/pip-gitlab-verify.err)"

echo "If this fails while step 1-4 pass: it's the certifi-vs-system-store gap, not a missing CA in the OS trust store."

fi

fi

  

echo

echo "=== Summary ==="

echo "1-4 share the system OpenSSL trust store; 5/5b use pip's own (certifi) bundle."

echo "If everything passes, the custom CA copy is confirmed unnecessary for all these tool paths."
```