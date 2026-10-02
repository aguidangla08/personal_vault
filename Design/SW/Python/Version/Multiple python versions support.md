It depends mostly on whether your project is a **library** others install, or an **application** you deploy yourself. Here are three options.

**Option 1: Support exactly one version**

```toml
requires-python = "==3.12.*"
classifiers = ["Programming Language :: Python :: 3.12"]

[tool.mypy]
python_version = "3.12"
```

- Least complexity: one CI job, one mypy target, you can use all modern syntax.
- The downside is that anyone on another version can't install it, and you have to migrate everyone whenever you upgrade.
- Best for an application you control (a service, an internal tool, a script you run yourself), especially if you also pin the version in Docker or `.python-version`.

**Option 2: Support a small recent window (usually 2-3 versions)**

```toml
requires-python = ">=3.11"
classifiers = [
  "Programming Language :: Python :: 3.11",
  "Programming Language :: Python :: 3.12",
  "Programming Language :: Python :: 3.13",
]

[tool.mypy]
python_version = "3.11"
```

- A good balance: users have some flexibility, and you only carry a bit of extra testing.
- You drop old versions on a schedule, ideally when they reach end-of-life (each Python version gets about 5 years of support).
- Best for a library with real users, where you don't want to force upgrades immediately.

**Option 3: Support a wide range (what you have now, 3.9 to 3.12)**

- Maximum compatibility, but the most cost: you can't use newer features (`match`, `X | Y` types, and so on), and CI runs on every version.
- Python 3.9 is already end-of-life (October 2025), so this range mostly adds burden without much benefit today.
- Only worth it if you know you have users stuck on old versions.

**My suggestion:** if it's an application for you or your team, choose Option 1. If it's a library for others, choose Option 2, and drop 3.9 now that it's past end-of-life. Either way, keep `requires-python`, the classifiers, and mypy's `python_version` in sync, with mypy set to the lowest version you support.