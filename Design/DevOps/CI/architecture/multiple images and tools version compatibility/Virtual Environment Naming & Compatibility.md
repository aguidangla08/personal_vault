Notes on keying `venv` directories so a host-mounted folder can be safely reused across multiple container images.

## The problem

A flat path like `~/venv/python3.12` gets silently reused by any image that happens to run the same Python minor version — even if the OS, libc, or CPU architecture differs. Compiled wheels (numpy, cryptography, etc.) are built against a specific OS/arch, so a venv created inside a Debian x86_64 container can fail or behave incorrectly when reused inside an Alpine or arm64 container.

## Recommended path structure

```
~/venv/{os}-{arch}/python{major}.{minor}
```

Example:

```
~/venv/debian12-x86_64/python3.12
~/venv/alpine3.19-aarch64/python3.12
```

|Component|Source|Why it's needed|
|---|---|---|
|`{os}`|`/etc/os-release` (`ID` + `VERSION_ID`)|Distinguishes libc/ABI (glibc vs musl) and system library versions|
|`{arch}`|`uname -m`|Compiled wheels are architecture-specific (x86_64 vs aarch64)|
|`python{major}.{minor}`|`sys.version_info`|Matches Python's own ABI compatibility boundary|

## Why patch version is excluded

Python guarantees ABI compatibility within the same minor version — `3.12.0` through `3.12.x` all work with the same compiled extensions. Including the patch version (`python3.12.3`) would force a new venv every time the base image bumps its bundled Python via routine OS security updates, defeating the reuse goal without any real compatibility benefit.

## Why pip version is excluded

Pip is a package manager, not something wheels are compiled against — it isn't a compatibility signal. Since setup scripts typically run `pip install --upgrade pip` right after creating the venv anyway, keying the path on pip's version would also cause unnecessary venv churn every time the image's bundled pip changes.

## Quick reference

```bash
OS_ID="$(. /etc/os-release && echo "${ID}${VERSION_ID}")"
ARCH="$(uname -m)"
PYTHON_VERSION="$(python3 -c 'import sys; print(f"{sys.version_info.major}.{sys.version_info.minor}")')"

VENV_DIR="${HOME}/venv/${OS_ID}-${ARCH}/python${PYTHON_VERSION}"
```

## Alternatives considered

|Structure|Example|Trade-off|
|---|---|---|
|Flat|`venv/debian12-x86_64-py3.12`|Simplest to `ls`/grep; no nesting benefit if only one Python version per OS/arch is ever used|
|Two-level (chosen)|`venv/debian12-x86_64/python3.12`|Groups multiple Python versions under one OS/arch combo|
|Three-level|`venv/debian12/x86_64/python3.12`|Fully separates each dimension; adds a directory level with little practical gain here|