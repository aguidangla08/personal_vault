Listed directory rtl-make
Listed directory tests
Listed directory test_cocotb
Viewed test_cocotb.py:1-91

Here are the clarifications regarding the code in [test_cocotb.py](file:///home/albertaguilera/projects/docker-top/src/gitlab-work/AI-friendly/rtl-make/tests/test_cocotb/test_cocotb.py):

---

### 1. What does `pytestmark` do? Does it skip all tests if `nvc` is not found?

Yes, exactly. 

`pytestmark` is a special global variable recognized by `pytest`. When you assign a pytest mark to this variable at the module level (the top of a Python file), Pytest applies that mark to **every single test function and class** within that module.

In this file:
```python
NVC_AVAILABLE = shutil.which("nvc") is not None
pytestmark = pytest.mark.skipif(not NVC_AVAILABLE, reason="nvc simulator not available on PATH")
```
If `nvc` (the VHDL simulator) is not installed on the system (`shutil.which("nvc")` returns `None`), then `not NVC_AVAILABLE` evaluates to `True`. As a result, the entire module is skipped when Pytest runs.

---

### 2. What is `monkeypatch` and what does it contain?

`monkeypatch` is a built-in fixture provided by `pytest` that allows you to safely modify (or "patch") objects, dictionaries, environment variables, or the current working directory during the execution of a test. When the test finishes, Pytest automatically undoes all of those changes to prevent tests from leaking side effects to each other.

In the [built_driver](file:///home/albertaguilera/projects/docker-top/src/gitlab-work/AI-friendly/rtl-make/tests/test_cocotb/test_cocotb.py#L34-L50) fixture:
```python
@pytest.fixture
def built_driver(tmp_path, monkeypatch):
    monkeypatch.chdir(tmp_path)
    monkeypatch.delenv("PYTEST_CURRENT_TEST", raising=False)
```

It is used to:
* `monkeypatch.chdir(tmp_path)`: Temporarily change the current working directory to the test's temporary folder (`tmp_path`) so that build logs and simulator outputs are generated there.
* `monkeypatch.delenv("PYTEST_CURRENT_TEST", raising=False)`: Delete the `"PYTEST_CURRENT_TEST"` environment variable. This is done because `cocotb` behaves differently (masking internal errors) when it detects it's running inside pytest. Deleting it forces cocotb to run as if it were running in production.

#### Common methods contained in the `monkeypatch` object:
* `setattr(obj, name, value, raising=True)`: Set an attribute on an object or module.
* `delattr(obj, name, raising=True)`: Delete an attribute.
* `setitem(dic, name, value)`: Set a dictionary/mapping entry.
* `delitem(dic, name, raising=True)`: Delete a dictionary/mapping entry.
* `setenv(name, value)`: Set an environment variable.
* `delenv(name, raising=True)`: Remove an environment variable.
* `syspath_prepend(path)`: Prepend a path to `sys.path`.

---

### 3. Explanation of `@pytest.mark.parametrize`

`@pytest.mark.parametrize` is a decorator that runs the decorated test function multiple times with different sets of inputs. This avoids duplicating code for different test cases.

In [test_cocotb.py](file:///home/albertaguilera/projects/docker-top/src/gitlab-work/AI-friendly/rtl-make/tests/test_cocotb/test_cocotb.py#L52):
```python
@pytest.mark.parametrize(
    "params, should_raise",
    [
        pytest.param({}, False, id="passes"),
        pytest.param(
            {"BUG_ADDER": "true"},
            True,
            ...
        ),
        pytest.param(
            {"CRASH_SIM": "true", "CRASH_AT_CYCLE": "3"},
            True,
            ...
        ),
    ],
)
```
* **First Argument (`"params, should_raise"`)**: Defines the names of the parameters that will be passed into the test function.
* **Second Argument (List of `pytest.param`)**: A list containing the test inputs for each run. Each element represents one test run:
  1. **Run 1**: `params` is `{}` and `should_raise` is `False`. (Labled with `id="passes"`).
  2. **Run 2**: `params` is `{"BUG_ADDER": "true"}` and `should_raise` is `True`. Labeled as `failing_testcase` and marked with `xfail` (meaning we currently expect it to fail, but eventually it should pass as a strict failure check once implemented).
  3. **Run 3**: `params` is `{"CRASH_SIM": "true", "CRASH_AT_CYCLE": "3"}` and `should_raise` is `True`. (Labeled with `id="simulator_crash"`).

The `pytest.param()` helper function takes the following arguments:

```python
pytest.param(*values, marks=(), id=None)
```
1. `*values` (Positional arguments)
These are the actual test inputs. You must provide one value for each parameter specified in the `@pytest.mark.parametrize` decorator, in the exact same order.
* **Example:** If you defined `@pytest.mark.parametrize("a, b, expected", ...)`, you would write `pytest.param(1, 2, 3)`.

1. `id` (Keyword-only argument, optional)
A string that serves as a custom name or identifier for this specific test case. If provided, Pytest will use it in the test execution output instead of auto-generating one.
* **Example:** `pytest.param(..., id="test_with_negative_inputs")`
* **Output display:** `test_my_func[test_with_negative_inputs]` instead of `test_my_func[1-2-3]`.

1. `marks` (Keyword-only argument, optional)
A single mark or a list/tuple of marks (like `pytest.mark.skip` or `pytest.mark.xfail`) that you want to apply **only** to this particular parameter set.
* **Example:** `pytest.param(..., marks=pytest.mark.xfail(reason="not implemented yet"))`
* **Example:** `pytest.param(..., marks=[pytest.mark.skip(reason="unsupported OS"), pytest.mark.slow])`

---

### 4. How arguments of `test_run_test_detects_failures` are used

The test function is defined as:
```python
def test_run_test_detects_failures(built_driver, tmp_path, params, should_raise):
```

Here is where each argument comes from and how it is used:

1. **`built_driver`**: 
   * **Source**: This comes from the custom [built_driver](file:///home/albertaguilera/projects/docker-top/src/gitlab-work/AI-friendly/rtl-make/tests/test_cocotb/test_cocotb.py#L34-L50) fixture. 
   * **Usage**: On line 73, it is unpacked into `driver, runner = built_driver`. `driver` is an instance of `CocotbDriver` and `runner` is the `cocotb_tools.runner` instance compiled for NVC.
2. **`tmp_path`**:
   * **Source**: This is a built-in Pytest fixture that returns a unique temporary directory path (`pathlib.Path`) for the duration of the test.
   * **Usage**: It is used to specify the file path for the run logs: `log_file=tmp_path / "run.log"`.
3. **`params`**:
   * **Source**: Injected by `@pytest.mark.parametrize` for the current test run.
   * **Usage**: Passed into `driver._run_test(..., params=params, ...)`. These parameters are passed to the cocotb environment and HDL simulation to simulate specific scenarios (like introducing a design bug or causing the simulator to crash).
4. **`should_raise`**:
   * **Source**: Injected by `@pytest.mark.parametrize` for the current test run.
   * **Usage**: Determines whether the code expects an exception to be thrown by `driver._run_test`.
     ```python
     if should_raise:
         with pytest.raises(Exception):
             call()
     else:
         call()
     ```
     * If `True`, the test wraps the execution in `pytest.raises(Exception)`. If `driver._run_test()` throws an exception, the test passes.
     * If `False`, it executes `call()` normally. If it throws an exception, the test fails.