# Which Python dependencies to specify, and how (with and without uv)

## The key idea: intent vs result

| | Says | With uv | Without uv |
|---|---|---|---|
| `pyproject.toml` | What you **allow**: version ranges | Written by you or by `uv add` | Written by you, by hand |
| Lock file | What you **got**: exact versions | `uv.lock`, created automatically | `requirements.txt` from `pip freeze`, or `pip-compile` (pip-tools) |

The toml is the intent; the lock is the reproducible result. Because the lock fixes exact versions, the ranges in the toml can stay fairly loose. The big difference: uv creates and maintains the lock for you, while plain pip has no lock file, so you have to build one yourself.

## What to specify (same in both cases)

The content of `pyproject.toml` is the same whether or not you use uv. Only the commands differ.

### 1. Python version (always)

```toml
[project]
requires-python = ">=3.11"
```

Defines which Python versions you support. Keep it in sync with the classifiers and with mypy's `python_version` (set to the lowest supported version).

### 2. Runtime dependencies (what your code imports)

```toml
[project]
dependencies = [
  "requests>=2.32.3",
  "pandas>=2.0,<3",
]
```

- Give a **lower bound**: the oldest version whose features you actually need.
- Add an **upper bound** only when you have a reason (a known breaking major release).
- Only list packages you import directly. Their own dependencies are resolved automatically.

**Why not leave the version out?** If you list a dependency with no version, the installer accepts any version and never checks what your code actually needs, because pip and uv only read your constraints, not your source code.
A fresh install will usually pick the newest release, but if an older version is already installed, pip considers it satisfied and won't upgrade it, so your code can fail at runtime on a missing feature. A newer major release can also break your code silently. A lower bound (`requests>=2.28`) guarantees that whatever gets installed is at least the oldest version you have tested with.
With uv, the lock file also records the exact version you developed with, but the lower bound in the toml still matters for anyone who installs your project as a library, since they don't use your lock file.

### 3. Development tools (mypy, pytest, ruff, ...)

```toml
[dependency-groups]
dev = [
  "mypy",
  "pytest",
  "ruff",
]
```

- Usually **no version constraint** is needed if you have a lock file that pins them.
- Without a lock file, unconstrained dev tools can change version between machines, so consider a lower bound or pin them in your requirements file.
- Groups can be split by purpose (`dev`, `docs`, `typechecking`).

### 4. Optional features (extras)

```toml
[project.optional-dependencies]
plot = ["matplotlib>=3.8"]
```

For dependencies only some users need. Installed with `pip install yourpackage[plot]`.

## Workflow: with uv vs without uv

| Task | With uv | Without uv (pip + venv) |
|---|---|---|
| Start a project | `uv init myproject` | Create `pyproject.toml` by hand |
| Create the environment | Automatic (`.venv`) | `python -m venv .venv` then `source .venv/bin/activate` (Windows: `.venv\Scripts\activate`) |
| Add a runtime dependency | `uv add requests` | Edit `dependencies` in the toml, then `pip install -e .` |
| Add a version constraint | `uv add "pandas>=2.0,<3"` | Write it in the toml by hand, then reinstall |
| Add a dev tool | `uv add --dev mypy` | Add it to `[dependency-groups]` by hand, then `pip install --group dev` (pip 25.1+), or `pip install mypy` directly |
| Install everything | `uv sync` | `pip install -e .` (plus `--group dev` on pip 25.1+) |
| Create the lock | Automatic (`uv.lock`) | `pip freeze > requirements.txt`, or `pip-compile pyproject.toml` (needs `pip install pip-tools`) |
| Reproduce exactly (CI, teammate) | `uv sync --frozen` | `pip install -r requirements.txt` |
| Upgrade on purpose | `uv lock --upgrade` | `pip-compile --upgrade`, or `pip install --upgrade <pkg>` then `pip freeze > requirements.txt` |
| Run a command in the environment | `uv run pytest` | Activate the venv, then `pytest` |
| Install the right Python | `uv python install 3.12` | Install it yourself (system package, pyenv, python.org) |
| Run a tool without installing it | `uvx ruff check .` | `pipx run ruff check .` (needs pipx) |

### Typical session, side by side

**With uv**

```bash
uv init myproject && cd myproject
uv add requests
uv add --dev mypy pytest
uv run pytest
uv sync --frozen          # on another machine / CI
```

**Without uv**

```bash
mkdir myproject && cd myproject
python -m venv .venv
source .venv/bin/activate
# write pyproject.toml by hand, with dependencies = ["requests>=2.32.3"]
pip install -e .
pip install mypy pytest
pip freeze > requirements.txt     # the "lock"
pytest
pip install -r requirements.txt   # on another machine / CI
```

### Things that work differently without uv

- **No automatic editing of the toml.** `pip install requests` installs the package but does not add it to `pyproject.toml`. If you forget to edit the file by hand, your project will look complete on your machine but fail elsewhere.
- **`pip freeze` is a snapshot, not a resolution.** It lists everything installed right now, including tools you added only for experimenting. It also only reflects your current Python and OS.
- **`pip-compile` is closer to a real lock file** (it resolves from your toml), but it is a separate tool and produces one file per platform and Python version.
- **pip only considers the Python it runs on**, while uv resolves for the whole `requires-python` range.
- **Environment drift:** nothing checks that your environment matches the requirements, so stale or missing packages are easy to miss. `uv run` checks this every time.

## When to add an upper limit or exact version

- A package has broken your code before, or a major release is known to change its API (`pandas<3`).
- A tool's output affects your team, such as a formatter that changes style between versions.
- Two packages must match each other (a plugin and its host tool).
- **Without a lock file**, you may want to be a bit stricter with important dependencies, since nothing else fixes their versions.

Otherwise, leave it open. Unneeded upper limits cause conflicts later.

## Application vs library

| | Application (deployed by you) | Library (published for others) |
|---|---|---|
| Loose ranges in toml | Fine | Preferred |
| Upper limits | Optional | Avoid unless needed |
| Lock file | Commit it, install exactly from it | Commit it (for dev/CI), but users resolve their own |
| Why | You control the environment | Your requirements must be combinable with other people's |

## Version syntax cheat sheet

| Written | Meaning |
|---|---|
| `requests>=2.32` | 2.32 or newer |
| `pandas>=2.0,<3` | 2.x only |
| `numpy==1.26.*` | any 1.26 patch release |
| `foo~=1.4` | 1.4 or newer, but below 2.0 |
| `foo==1.4.2` | exactly that version (avoid unless necessary) |

## Rules of thumb

1. Always set `requires-python`.
2. Give runtime dependencies a lower bound (uv writes it for you; without uv, write it yourself).
3. Leave dev tools unconstrained only if you have a lock file pinning them.
4. Add an upper limit only when you have a concrete reason.
5. Commit `pyproject.toml` and the lock (`uv.lock` or `requirements.txt`); never commit `.venv/`.
6. Upgrade on purpose, then run your tests, instead of letting versions drift.
7. If you find yourself maintaining a venv, a Python installer, a lock generator and a tool runner separately, that is the problem uv is designed to remove.