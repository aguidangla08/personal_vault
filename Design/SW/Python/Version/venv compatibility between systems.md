# GitLab Pipeline venv Compatibility — Rocky Linux 8.9 vs Ubuntu 24.04

## Prompt

I have a gitlab pipeline with a setup job where venv is created an user python packets are instaled,

there are 2 jobs with different images, python and pip versions that use them:

**image 1:**
```
cat /etc/os-release
NAME="Rocky Linux"
VERSION="8.9 (Green Obsidian)"
ID="rocky"
ID_LIKE="rhel centos fedora"
VERSION_ID="8.9"
PLATFORM_ID="platform:el8"
PRETTY_NAME="Rocky Linux 8.9 (Green Obsidian)"
ANSI_COLOR="0;32"
LOGO="fedora-logo-icon"
CPE_NAME="cpe:/o:rocky:rocky:8:GA"
HOME_URL="https://rockylinux.org/"
BUG_REPORT_URL="https://bugs.rockylinux.org/"
SUPPORT_END="2029-05-31"
ROCKY_SUPPORT_PRODUCT="Rocky-Linux-8"
ROCKY_SUPPORT_PRODUCT_VERSION="8.9"
REDHAT_SUPPORT_PRODUCT="Rocky Linux"
REDHAT_SUPPORT_PRODUCT_VERSION="8.9"

python --version
Python 3.12.14

pip --version
pip 23.2.1 from /usr/lib/python3.12/site-packages/pip (python 3.12)
```

**image 2:**
```
cat /etc/os-release
PRETTY_NAME="Ubuntu 24.04.5 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
VERSION="24.04.5 LTS (Noble Numbat)"
VERSION_CODENAME=noble
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=noble
LOGO=ubuntu-logo

python --version
Python 3.12.3

pip --version
pip 24.0 from /usr/lib/python3/dist-packages/pip (python 3.12)
```

Will they be compatible? should I generate 2 venv?

## Answer

Short answer: no, you shouldn't share one venv across those two jobs. The blocker isn't Python (3.12.14 vs 3.12.3 is patch-level, no ABI concern) or pip (23.2.1 vs 24.0, both have the modern resolver, negligible risk) — it's **glibc**. Rocky 8.9 ships glibc 2.28, Ubuntu 24.04 ships glibc 2.39. glibc symbol versioning is forward-compatible only: a compiled extension (wheel with C code — numpy, pandas, cryptography, psycopg2, lxml, etc.) built/linked against 2.39 will refuse to load against 2.28 (`version 'GLIBC_2.29' not found`). Pure-Python packages wouldn't care, but you rarely control that for a whole dependency tree.

So the real question is whether your requirements include any compiled deps. If yes, you need per-OS environments. Three ways to structure that:

### 1. Matrix the setup job itself (recommended)

Turn "setup" into a job template run once per image, each producing its own venv artifact/cache keyed by image name.

```yaml
.setup:
  script:
    - python -m venv venv
    - venv/bin/pip install -r requirements.txt
  cache:
    key: "venv-$CI_JOB_IMAGE"
    paths: [venv/]

setup:rocky:
  extends: .setup
  image: rocky8.9-python3.12

setup:ubuntu:
  extends: .setup
  image: ubuntu24.04-python3.12
```

Downstream jobs pull the cache matching their own image. Function/interface impact: none on your app code — only pipeline YAML changes (job names, cache keys). This is the cleanest, most explicit fix.

### 2. Build venv fresh in every job (drop the shared-setup idea entirely)

Skip caching altogether; each job does `python -m venv && pip install` itself. Simpler pipeline, slower (no cache reuse), but zero compatibility surface since nothing crosses images. No interface changes at all, just removes the "setup job + artifact passing" pattern.

### 3. Keep one venv, but restrict it to pure-Python-only dependencies

Audit `requirements.txt`, confirm none of them ship compiled `.so` extensions (check on PyPI for wheels with platform tags vs `py3-none-any`), and if genuinely all pure-Python, one venv could survive being passed between images. Riskiest long-term — a future `pip install` of a new dependency silently reintroduces a compiled package and breaks the other job. Not recommended unless you can guarantee the dependency set stays pure-Python.

**Recommendation:** option 1. It's a small, contained pipeline change (cache key + duplicated setup stanza), doesn't touch any app code or interfaces, and removes the glibc risk entirely rather than hoping it doesn't bite you later.