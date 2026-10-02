# uv: a quick overview

## What is uv?

`uv` is a Python package and project manager made by Astral (the creators of the Ruff linter). It is written in Rust and is designed as a fast, all-in-one replacement for several Python tools: pip, venv, pyenv, pip-tools, pipx and (partly) Poetry.

## Where to find it

- Documentation: https://docs.astral.sh/uv/
- Source code: https://github.com/astral-sh/uv

Installation:

```bash
# Linux / macOS
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows (PowerShell)
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"

# Alternatives
pip install uv
brew install uv
```

## Main features

- **Very fast** installs and dependency resolution (typically 10 to 100 times faster than pip).
- **Project management** based on the standard `pyproject.toml`.
- **Lock file (`uv.lock`)** with exact versions and hashes of every dependency, for reproducible environments.
- **Automatic virtual environments**: creates and manages `.venv` for you.
- **Python version management**: downloads and installs the Python versions you need (`uv python install 3.12`).
- **pip-compatible interface**: `uv pip install ...` works as a drop-in replacement.
- **Tool runner**: run command-line tools in isolation with `uvx` (e.g. `uvx ruff check .`).
- **Cross-platform lock**: one lock file can cover Linux, macOS and Windows.

## Basic workflow

```bash
uv init myproject          # create pyproject.toml and .python-version
cd myproject
uv add requests            # add a dependency: updates the toml, resolves, installs, writes uv.lock
uv add --dev pytest        # add a development-only dependency
uv run python main.py      # run inside the project environment
uv sync                    # make the environment match uv.lock (e.g. after git pull)
uv lock --upgrade          # deliberately upgrade to the newest allowed versions
```

Which file does what:

|File|Written by|Purpose|
|---|---|---|
|`pyproject.toml`|You (or `uv add`)|What you want: allowed version ranges|
|`uv.lock`|uv only|What you got: exact versions and hashes|
|`.venv/`|uv only|The installed packages (not committed to git)|

Commit `pyproject.toml` and `uv.lock`. Do not commit `.venv/`.

## Advantages

- Much faster than pip and most alternatives.
- Reproducible installs thanks to the lock file, which avoids "works on my machine" problems.
- One tool for environments, dependencies, Python versions and tools, so less to learn and configure.
- Clear error messages when dependencies conflict.
- Uses standard files (`pyproject.toml`), so you are not locked in and can go back to pip.
- Actively developed and widely adopted.

## Disadvantages

- Newer than pip, so some older tutorials, tools and CI templates don't mention it.
- It does not remove real dependency conflicts; it only reports them more clearly.
- It does not manage non-Python system libraries (C libraries, CUDA, etc.); for that, tools like Docker or conda are still needed.
- Still evolving quickly, so commands and defaults can change between versions.
- Some team or company setups may already be standardized on pip, Poetry or conda, which makes switching a coordination effort.