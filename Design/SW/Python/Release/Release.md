For a release you don't install the package in editable mode (`pip install -e .`). You build it into a distributable package (a wheel and an sdist), test that built artifact in a clean environment, and publish it. Dev tools stay out because they live in `[dependency-groups]`, and the lock or freeze file is not part of the package.

## 1. Prepare

- Set the version in `pyproject.toml` (`version = "1.2.0"`), or use a tool that derives it from git tags (hatch-vcs, setuptools-scm).
- Declare a build backend:

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

- Make sure runtime dependencies are in `[project.dependencies]` with loose ranges (`requests>=2.31`), not exact pins. Pins are for applications and CI, and they make your library hard to install alongside other packages.
- Run lint and tests first, as in the earlier CI setup.

## 2. Build

|Tool|Command|
|---|---|
|pip|`pip install build` then `python -m build`|
|uv|`uv build`|
|Poetry|`poetry build`|

All three write a `.whl` and a `.tar.gz` into `dist/`.

## 3. Test the built package in a clean venv

This is the step people skip. It catches missing files and undeclared dependencies that an editable install hides.

```bash
python -m venv /tmp/release-test && source /tmp/release-test/bin/activate
pip install dist/yourpackage-1.2.0-py3-none-any.whl
python -c "import yourpackage; print(yourpackage.__version__)"
```

With uv: `uv run --isolated --no-project --with dist/*.whl python -c "import yourpackage"`.

## 4. Publish

|Tool|Command|
|---|---|
|pip|`pip install twine` then `twine upload dist/*`|
|uv|`uv publish`|
|Poetry|`poetry publish`|

Try TestPyPI first (`twine upload -r testpypi dist/*`, `uv publish --publish-url https://test.pypi.org/legacy/`, or `poetry publish -r testpypi`). The best practice is to publish from CI with PyPI "trusted publishing" (OIDC) instead of storing an API token, triggered when you push a git tag such as `v1.2.0`.

## 5. Tag

```bash
git tag v1.2.0 && git push origin v1.2.0
```

## What about the freeze or lock file?

Commit `uv.lock`, `poetry.lock` or `requirements.txt` for reproducible development and CI, but they aren't installed by users of your package. Users get whatever satisfies your `[project.dependencies]` ranges.

**If it's an application rather than a library** (a service, a Docker image), the release is different. You don't publish to PyPI. You install from the lock in the image:

```bash
uv sync --locked --no-dev
```

That gives exact versions and leaves out the dev groups.

Are you releasing a library to PyPI or deploying an application? I can give the exact pipeline for either.