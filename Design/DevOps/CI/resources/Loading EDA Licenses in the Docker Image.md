# Loading EDA Licenses in the Docker Image

## Purpose

The tools installed in the image (Xilinx/AMD, Aldec, Mentor/Siemens and others using FlexLM or OSSLM) need a license server address to run. The address is configured in two stages:

1. **Build time**: license variables are set as defaults so the tools can be installed and validated during the image build.
2. **Run time**: the same variables can be overridden with `-e`, so the license server can be changed without rebuilding the image.

## 1. Build time: default license variables

In the Dockerfile:

```dockerfile
# Setup licenses
ENV LM_LICENSE_FILE=${LM_LICENSE_FILE}
ENV ALDEC_LICENSE_FILE=${ALDEC_LICENSE_FILE}
ENV OSSLMGR_LICENSE_FILE=${OSSLMGR_LICENSE_FILE}
ENV XILINXD_LICENSE_FILE=${XILINXD_LICENSE_FILE}
```

Some installers check for a valid license while installing, so the variables must be defined during the build. They are then stored in the image as its default environment.

Note: `ENV X=${X}` only expands to a value if `X` was declared earlier as a build argument (`ARG X`) and passed with `--build-arg X=...`. Otherwise it expands to an empty string.

## 2. Run time: override the license server

In the `docker run` command:

```bash
-e LM_LICENSE_FILE="${LM_LICENSE_FILE:-}" \
-e ALDEC_LICENSE_FILE="${ALDEC_LICENSE_FILE:-}" \
-e OSSLMGR_LICENSE_FILE="${OSSLMGR_LICENSE_FILE:-}" \
-e XILINXD_LICENSE_FILE="${XILINXD_LICENSE_FILE:-}" \
```

Each `-e` passes the value from the host shell into the container, and it takes precedence over the value baked into the image.

The `${VAR:-}` syntax expands to an empty string when the host variable is unset, which avoids errors under `set -u` (nounset).

## 3. What each variable is for

|Variable|Used by|
|---|---|
|`LM_LICENSE_FILE`|Generic FlexLM variable, read by most tools (Siemens/Mentor, Synopsys, Cadence and others)|
|`XILINXD_LICENSE_FILE`|AMD/Xilinx tools (Vivado, Vitis, etc.)|
|`ALDEC_LICENSE_FILE`|Aldec tools (Riviera-PRO, Active-HDL)|
|`OSSLMGR_LICENSE_FILE`|Tools using the OSSLM license manager|

Values can be a `port@host` server address (e.g. `2100@license-server`), a path to a license file, or a colon-separated list of these (semicolon on Windows) for fallback servers.

## 4. Usage examples

Use the license server baked into the image (or none set on the host):

```bash
./run.sh
```

Override a single server for one session:

```bash
XILINXD_LICENSE_FILE=2100@other-server ./run.sh
```

Override all of them (e.g. when working from a different site or over VPN):

```bash
export LM_LICENSE_FILE=1717@lic1:1717@lic2
export XILINXD_LICENSE_FILE=2100@lic1
./run.sh
```

## 5. Behavior to be aware of

- **Empty values override too**: `-e VAR=""` sets the variable to an empty string inside the container instead of leaving the image default in place. If the host variable is unset, the run script effectively blanks the image default. To keep the image default when the host variable is unset, pass `-e` only when it is set, for example:
    
    ```bash
    LICENSE_ARGS=()
    for v in LM_LICENSE_FILE ALDEC_LICENSE_FILE OSSLMGR_LICENSE_FILE XILINXD_LICENSE_FILE; do
        [ -n "${!v:-}" ] && LICENSE_ARGS+=(-e "$v=${!v}")
    done
    ```
    
- **No secrets in the image**: license server addresses baked in via `ENV` are visible with `docker inspect` and `docker history`. This is fine for host@port addresses, but never embed license file contents or keys.
    
- **Network access**: the container must be able to reach the license server (VPN, firewall, and DNS resolution of the hostname). With the default bridge network, the server must be reachable from the host network too. `--network host` can help when the server is only reachable that way.
    
- **Host-locked licenses**: node-locked licenses tied to a MAC address or hostname will not validate inside a container by default, since these differ from the host. Use a network license server, or set `--mac-address` / `--hostname` to match the licensed values.