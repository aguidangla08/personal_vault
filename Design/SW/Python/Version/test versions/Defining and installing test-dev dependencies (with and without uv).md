# Defining and installing test/dev dependencies (with and without uv)

Tools like pytest, pytest-cov, mypy, black, flake8 and isort are only needed by developers. They should **not** be installed when someone installs your package, so they don't belong in `[project.dependencies]`.

## 1. Where to define them

### Recommended: a dependency group (`[dependency-groups]`)

```toml
[dependency-groups]
dev = [
  "pytest",
  "pytest-cov",
  "mypy",
  "black",
  "flake8",
  "isort",
]
```

Dependency groups (PEP 735) are the standard place for development-only tools. They are not published with your package. You can also split them by purpose:

```toml
[dependency-groups]
test = ["pytest", "pytest-cov"]
lint = ["black", "flake8", "isort"]
typecheck = ["mypy"]
dev = [
  { include-group = "test" },
  { include-group = "lint" },
  { include-group = "typecheck" },
]
```

### Alternative: an optional-dependencies extra

```toml
[project.optional-dependencies]
dev = ["pytest", "pytest-cov", "mypy", "black", "flake8", "isort"]
```

Use this only if you must support older pip versions (before 25.1). The downside is that it is published with your package as an extra (`yourpackage[dev]`), which is less clean.

## 2. Installing them

| Task | With uv | Without uv |
|---|---|---|
| Add the tools | `uv add --dev pytest pytest-cov mypy black flake8 isort` (edits the toml for you) | Edit `[dependency-groups]` in the toml by hand |
| Install the dev group | `uv sync` (the `dev` group is included by default) | `pip install --group dev` (pip 25.1+) |
| Install a named group | `uv sync --group test` | `pip install --group test` |
| Install with the extra (alternative) | `uv sync --extra dev` | `pip install -e ".[dev]"` |
| Run a tool | `uv run pytest` | Activate the venv, then `pytest` |
| Reproduce exact versions (CI) | `uv sync --frozen` | `pip install -r requirements.txt` (if you made one) |

Without uv, remember to create and activate a virtual environment first:

```bash
python -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate
```

## 3. What about versions?

Both uv and pip pick the newest compatible version when none is given. The difference is whether the choice is **remembered**:

- **With uv**, the exact versions are saved in `uv.lock` automatically, so every machine gets the same ones.
- **Without uv**, nothing is recorded, so the next install may pick newer versions.

Recommendations:

| Tool | With uv | Without uv |
|---|---|---|
| pytest, pytest-cov, mypy | No constraint needed | No constraint, or a loose lower bound (e.g. `pytest>=8`) |
| black, isort, flake8 | No constraint needed (the lock pins them) | Pin more tightly (e.g. `black==25.*`, `flake8>=7,<8`), because new versions can change what counts as "correctly formatted" |

Without uv you have three options to keep versions stable:

1. Do nothing (fine for a personal project, but versions can drift).
2. Set versions by hand in the toml.
3. Generate a lock file yourself with `pip freeze > requirements.txt` or, better, `pip-compile` (from pip-tools), and regenerate it after every change.

## 4. Configuration so the tools agree with each other

```toml
[tool.isort]
profile = "black"          # otherwise isort and black fight over import formatting

[tool.mypy]
python_version = "3.11"    # the lowest Python version you support

[tool.pytest.ini_options]
addopts = "--cov=yourpackage"
```

flake8 does not read `pyproject.toml` natively. Use a `.flake8` file with `max-line-length = 88` to match black, or the `Flake8-pyproject` plugin.

## 5. Typical setup, side by side

**With uv**

*First time (the person who creates the lock):*

```bash
uv add --dev pytest pytest-cov mypy black flake8 isort   # edits pyproject.toml, creates .venv and uv.lock
uv run pytest
uv run mypy .
```

`uv add` does everything in one step: it adds the tools to `[dependency-groups]`, creates the `.venv`, installs the packages and writes `uv.lock`. Commit `pyproject.toml` and `uv.lock` to git (not `.venv/`).

*Another user (or CI) installing from the lock:*

```bash
git clone <repository-url> && cd <project>
uv sync --frozen               # creates .venv and installs exactly what uv.lock says
uv run pytest
uv run mypy .
```

`--frozen` installs the versions in `uv.lock` without re-resolving anything. There is no need to create or activate a venv, since `uv sync` creates it and `uv run` uses it.

*When dependencies change later:*

```bash
uv add --dev <new-tool>        # or edit pyproject.toml, then run: uv lock
git add pyproject.toml uv.lock # commit both together
uv lock --upgrade              # optional: deliberately move to newer versions, then run the tests
```

**Without uv**

*First time (the person who creates the freeze):*

```bash
python -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate
# add the tools to [dependency-groups] in pyproject.toml by hand
pip install --upgrade pip      # --group needs pip 25.1+
pip install -e .               # the project itself and its runtime dependencies
pip install --group dev        # the dev tools
pytest
pip freeze --exclude-editable > requirements.txt   # your own "lock"
```

Commit `requirements.txt` to git (not `.venv/`). `--exclude-editable` keeps your own project, which is installed in editable mode, out of the file, since it would otherwise be written as a local path that only exists on your machine.

*Another user (or CI) installing from the freeze:*

```bash
git clone <repository-url> && cd <project>
python -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt   # exact versions from the freeze
pip install -e . --no-deps        # the project itself, without changing the frozen versions
pytest
```

The freeze only fits the same setup it was created on: the same Python version and operating system. When you change dependencies in `pyproject.toml`, reinstall and run `pip freeze --exclude-editable > requirements.txt` again, otherwise the file goes stale.

### Why a freeze is interesting

Both `uv.lock` and a `pip freeze` file record the **exact versions** that worked, including the indirect dependencies (the dependencies of your dependencies). That is what makes an installation reproducible: a teammate, a CI job or you in six months get the same versions instead of whatever is newest that day. Without it, a new release of black, flake8 or mypy can change your results without you touching any code.

### `uv.lock` vs `pip freeze`: the differences

**Where the file comes from.** `uv.lock` is generated from `pyproject.toml`, so it contains what the toml asks for plus their dependencies, and nothing else. `pip freeze` is a snapshot of the venv at that moment, so it also contains anything you installed by hand (for example `pip install ipython` for a quick experiment), and anyone installing from that file gets those packages too.

**What happens to extra packages.** `uv sync` makes the venv match the lock exactly, so anything not in the lock is removed. `pip install -r requirements.txt` only adds packages, it never removes ones that are already in the venv.

**How to keep the pip freeze clean.** Freeze from a fresh venv where you installed only from the toml (`pip install -e .` and `pip install --group dev`), never from one you have experimented in. Alternatively, `pip-compile pyproject.toml -o requirements.txt` (from pip-tools) builds the file from the toml, the way uv does, so it cannot contain strays.

**`--frozen` vs `--locked`.** `uv sync --frozen` installs what `uv.lock` says without checking it against `pyproject.toml`, so if you edited the toml and forgot to update the lock, it silently installs the old versions. `uv sync --locked` fails in that case, which makes it a good choice for CI.

## 6. Tip: consider ruff

`ruff` replaces black, isort and flake8 with a single, much faster tool that reads `pyproject.toml` directly. If you are starting fresh, your list could shrink to `pytest`, `pytest-cov`, `mypy` and `ruff`.