# Installing from toml, installing from a freeze, and how to freeze

Comparison of three setups: **plain pip (no package manager)**, **Poetry** and **uv**. The pip and uv parts come from the dev-dependencies doc; the Poetry part is added in the same style.

Dev tools (pytest, mypy, black...) live in `[dependency-groups]` (PEP 735) in `pyproject.toml`, so they are not published with your package. Poetry also has its own `[tool.poetry.group.dev.dependencies]` table for the same purpose.

## 1. Installing from the toml

| | Plain pip | Poetry | uv |
|---|---|---|---|
| Create the venv | `python -m venv .venv` and activate it | Automatic | Automatic |
| Install project + runtime deps | `pip install -e .` | `poetry install` | `uv sync` |
| Install dev group | `pip install --group dev` (pip 25.1+) | `poetry install --with dev` (or `--only dev`) | `uv sync` (dev group is included by default) |
| Named group | `pip install --group test` | `poetry install --with test` | `uv sync --group test` |
| Add a tool | Edit the toml by hand | `poetry add --group dev pytest` | `uv add --dev pytest` |
| Run a tool | Activate the venv, then `pytest` | `poetry run pytest` | `uv run pytest` |

Note: when installing from the toml there is no lock, so pip picks the newest compatible versions on that day. Poetry and uv resolve against a lock file they create automatically.

## 2. Installing from a freeze / lock

| | Plain pip (`requirements.txt`) | Poetry (`poetry.lock`) | uv (`uv.lock`) |
|---|---|---|---|
| Install exact versions | `pip install -r requirements.txt` | `poetry install` | `uv sync --frozen` or `uv sync --locked` |
| Then the project itself | `pip install -e . --no-deps` | Included | Included |
| Removes extra packages in the venv? | No, it only adds | Only with `poetry install --sync` | Yes, `uv sync` makes the venv match the lock |
| Fails if the lock is stale? | No check | Yes, `poetry install` errors if `pyproject.toml` changed since locking | `--locked` fails, `--frozen` silently installs the old versions |

### Code: installing the frozen version

**Plain pip** (another user or CI, from `requirements.txt`)

```bash
git clone <repository-url> && cd <project>
python -m venv .venv
source .venv/bin/activate         # Windows: .venv\Scripts\activate
pip install -r requirements.txt   # exact versions from the freeze
pip install -e . --no-deps        # the project itself, without changing the frozen versions
pytest
```

**Poetry** (from `poetry.lock`)

```bash
git clone <repository-url> && cd <project>
poetry install                    # creates the venv and installs exactly what poetry.lock says
poetry install --with dev         # if the dev group is not installed by default
poetry install --sync             # optional: also remove packages not in the lock
poetry run pytest
```

**uv** (from `uv.lock`)

```bash
git clone <repository-url> && cd <project>
uv sync --locked                  # creates .venv, installs exactly uv.lock, fails if the lock is stale (good for CI)
# or: uv sync --frozen            # installs uv.lock without checking it against pyproject.toml
uv run pytest
```

If you only have a `requirements.txt` and want to use uv to install it into an active venv: `uv pip install -r requirements.txt`.

## 3. How to freeze

**Plain pip**

```bash
python -m venv .venv && source .venv/bin/activate
pip install -e .
pip install --group dev
pip freeze --exclude-editable > requirements.txt
```

Freeze from a fresh venv only, otherwise anything you installed by hand ends up in the file. Alternative: `pip-compile pyproject.toml -o requirements.txt` (pip-tools) builds it from the toml and cannot contain strays.

**Poetry**

```bash
poetry lock                  # writes poetry.lock from pyproject.toml
poetry install               # installs from it
# only if something needs a requirements.txt (Docker, CI...):
poetry export -f requirements.txt --with dev -o requirements.txt   # needs the poetry-plugin-export plugin
```

**uv**

```bash
uv lock                      # writes uv.lock (also done automatically by uv add / uv sync)
uv lock --upgrade            # deliberately move to newer versions
# only if something needs a requirements.txt:
uv export --format requirements-txt -o requirements.txt
```

Commit the lock file (`requirements.txt`, `poetry.lock` or `uv.lock`) and `pyproject.toml` together, never `.venv/`.

## 4. Pros and cons

### Plain pip + toml + `pip freeze`

**Pros**
- Nothing to install beyond Python and pip; works everywhere.
- Simple to understand; `requirements.txt` is understood by every tool and service.

**Cons**
- The freeze is a snapshot of the venv, so stray manual installs leak into it.
- Not automatically updated; it goes stale when the toml changes.
- Tied to the Python version and OS it was created on.
- `--group` needs pip 25.1+, and you edit the toml by hand.
- No removal of extra packages when installing.

### Poetry

**Pros**
- One tool for dependencies, venv, lock, build and publish.
- `poetry.lock` is generated from the toml, so it is clean and reproducible.
- Mature and widely used; good for projects that publish to PyPI.
- Fails clearly when the lock is out of date.

**Cons**
- Slower resolution and installs than uv.
- Historically used its own `[tool.poetry]` format rather than standard `[project]` / PEP 735 (newer versions support the standard `[project]` table).
- Exporting to `requirements.txt` needs an extra plugin.
- One more tool to install and keep updated.

### uv

**Pros**
- Very fast.
- `uv add` edits the toml, creates the venv, installs and writes `uv.lock` in one step.
- Uses standard `pyproject.toml` (`[project]`, `[dependency-groups]`).
- `uv sync` keeps the venv exactly equal to the lock; `--locked` is ideal for CI.
- Also offers a pip-compatible interface (`uv pip`) for gradual adoption.

**Cons**
- Newer than Poetry, so the ecosystem and habits are still catching up.
- `--frozen` silently installs a stale lock; use `--locked` in CI.
- `uv sync` deletes packages you installed by hand (use `--inexact` to keep them).
- Requires installing uv itself.

## 5. Quick recommendation

- Solo script or minimal environment where you cannot install anything: plain pip, freeze from a fresh venv.
- Library you publish, team already on it: Poetry.
- New project, speed, CI reproducibility: uv (and consider `ruff` to replace black, isort and flake8).