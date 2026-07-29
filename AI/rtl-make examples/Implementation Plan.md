# Implementation Plan: Update Cocotb Tests (NVC Only)

This implementation plan describes the file-by-file changes and steps required to update the `tests/test_cocotb` folder to use `tests/test_cocotb/vhdl_test_project` for verifying driver and command methods using only the `nvc` backend.

## Proposed Changes

### VHDL Test Project Fixtures
We will correct the VHDL source files in the test project which are currently copies of the Python testbench.

#### [MODIFY] [counter.vhd](file:///home/albertaguilera/projects/docker-top/src/gitlab-work/AI-friendly/rtl-make/tests/test_cocotb/vhdl_test_project/counter.vhd)
- Replace the duplicate Python code with a valid VHDL model for a basic counter.
- Incorporate generic parameters:
  - `BUG_ADDER`: boolean (increments by 2 instead of 1 when true, causing Python assert to fail).
  - `CRASH_SIM`: boolean (forces simulator severity failure assertion, causing crash).
  - `CRASH_AT_CYCLE`: integer (controls when simulator crashes).

#### [MODIFY] [counter_broken.vhd](file:///home/albertaguilera/projects/docker-top/src/gitlab-work/AI-friendly/rtl-make/tests/test_cocotb/vhdl_test_project/counter_broken.vhd)
- Replace the duplicate Python code with an invalid VHDL file containing syntax errors (e.g. missing entity keywords or semicolons) to test build failures.

---

### Driver Test Cases

#### [MODIFY] [test_cocotb.py](file:///home/albertaguilera/projects/docker-top/src/gitlab-work/AI-friendly/rtl-make/tests/test_cocotb/test_cocotb.py)
- Update existing driver unit/integration tests to ensure they execute successfully when using the updated `counter.vhd`.
- Verify behavior under three execution states using `nvc`:
  1. **Success case** (default parameters).
  2. **Assert failure case** (`BUG_ADDER=true`). Verify driver detects failures.
  3. **Simulator crash case** (`CRASH_SIM=true`). Verify driver detects crashes.
- Conditionally run simulator-dependent integration tests only if the `nvc` simulator is available locally (using `shutil.which("nvc")`). If not available, skip them dynamically to ensure the test suite is portable.

---

### CLI Command Test Cases

#### [NEW] [test_cocotb_cli.py](file:///home/albertaguilera/projects/docker-top/src/gitlab-work/AI-friendly/rtl-make/tests/test_cocotb/test_cocotb_cli.py)
- Create new test cases utilizing `typer.testing.CliRunner` to test the CLI commands defined in `src/rtl_make/commands/cocotb_commands.py`:
  - `build` command (successful build, missing sources, invalid args).
  - `run` command (successful run, parameter passing, failure output validation).
  - `regression` command (valid config execution, results summary validation).
- Use mocks (`unittest.mock`) to decouple CLI verification from local toolchains.

---

## Verification Plan

### Automated Tests
- Run pytest locally on the new cocotb test suite:
  ```bash
  pytest tests/test_cocotb
  ```
- Verify both mock-based CLI tests and simulator-dependent integration tests behave correctly.
