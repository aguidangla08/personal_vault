Here are the answers to your questions based on how `Runner.test()` is implemented in `cocotb_tools`:

### 1. Do I have to catch exceptions when running it? Which ones?

Yes, you need to be aware of how exceptions are raised and catch them accordingly:

*   **`SystemExit` (Must be caught):** If the VHDL simulator compiles but **crashes** during simulation, or the executable fails to run, `runner.test()` will log the error and call `sys.exit(simulator_exit_code)`. 
    *   This raises a **`SystemExit`** exception.
    *   *Important:* `SystemExit` inherits from `BaseException` (not `Exception`), so a standard `except Exception:` block will **not** catch it. You must explicitly catch `except SystemExit:` or `except BaseException:`.
*   **No Exceptions for Cocotb Assert Failures (in production):** If the simulation runs to completion (exit code 0) but some of your Python testcases fail their assertions, `runner.test()` **does not raise any exception** in production. It returns the XML file path normally.

---

### 2. Does the XML contain error information that is not stored inside the exceptions?

**Yes, absolutely.**

*   **Exceptions (`SystemExit`):** Only tell you that the process failed and provide the exit code (e.g., exit code `1`). They do not contain any info about which test failed or why.
*   **The XML file (`results.xml`):** Written in JUnit format, it contains the detailed reports of the Python test execution, including:
    *   The test suite summary (e.g., `<testsuite name="..." tests="2" failures="1" ...>`).
    *   For each failing test, the exact VHDL time of failure, the assertion message, and the **full Python traceback** showing the line of code that caused the failure.

Because of this, reading and parsing the XML file is the only way to know if tests failed when the simulator exited successfully, and it is the only source of structured failure details.