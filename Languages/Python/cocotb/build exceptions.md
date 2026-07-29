Here's exactly where each exception type originates inside `cocotb_tools.runner`, tracing back to the source you pulled earlier:

## `subprocess.CalledProcessError`

Generated in `Runner._execute_cmds()`:

```python
subprocess.run(cmd, cwd=cwd, env=self.env, check=True, stdout=stdout, stderr=stderr)
```

`check=True` means: if the subprocess (compiler/elaborator — `iverilog`, `vlog`, `xrun`, `vcs`, `nvc`, `ghdl`, `verilator`...) exits with a non-zero return code, `subprocess.run` raises `CalledProcessError` itself. This is **not** a `raise` statement cocotb wrote — it's Python's own subprocess module reacting to a failed external process. It fires for things like: a syntax error in your HDL, a missing module reference, a license-server timeout, or the compiler crashing. The exception carries `.returncode` and `.cmd` (and `.output`/`.stderr` if you passed `log_file`, since only then is output actually captured instead of streamed to your terminal).

## `ValueError`

Raised directly by cocotb in several places, all before any subprocess ever runs — meaning your build config is wrong, not that the simulator failed:

- `_determine_file_type()` — a source file's extension isn't `.v/.sv/.vh/.svh` (Verilog) or `.vhd/.vhdl` (VHDL) and it wasn't wrapped in a `VHDL()`/`Verilog()` tag:
    
    ```python
    raise ValueError(f"Can't determine source file type of {filename}. Use the `VHDL`, `Verilog`, or `VerilatorControlFile` tags.")
    ```
    
- `_set_verilog_sources` / `_set_vhdl_sources` / `_set_sources` / `_set_build_args` — you tagged a file with the wrong tag type, e.g. put a `VHDL(...)` inside `verilog_sources`:
    
    ```python
    raise ValueError(f"Unsupported file type: {source}")
    ```
    
- Each simulator's `_build_command()` — you gave a VHDL file to a Verilog-only simulator (Icarus, Verilator, Vcs, Dsim) or vice versa (Ghdl, Nvc):
    
    ```python
    raise ValueError(f"{type(self).__qualname__} only supports Verilog. {str(source.value)!r} cannot be compiled.")
    ```
    
- Missing required argument, e.g. Icarus/Xcelium/Vcs requiring `hdl_toplevel`:
    
    ```python
    raise ValueError("hdl_toplevel argument is required for all Icarus builds")
    ```
    
- Ghdl's `_test_command()` — an unsupported `timescale` precision string.

## `TypeError`

Only from `_as_sv_literal()`, used when formatting `defines`/`parameters` values into simulator command-line args:

```python
def _as_sv_literal(value: object) -> str:
    if isinstance(value, (int, float)):
        return str(value)
    elif isinstance(value, str):
        return '"' + value.translate(_sv_escape_translate_table) + '"'
    else:
        raise TypeError("Can't serialize this type as an SV literal")
```

This fires if you pass e.g. a `dict` or a custom object as a define/parameter value instead of `int`/`float`/`str`.

## `RuntimeError`

Two distinct triggers, both simulator-specific:

- **Ghdl/Nvc `_get_include_options` / `_get_define_options`** — these simulators don't support `includes=`/`defines=` at all, so the methods unconditionally raise:
    
    ```python
    def _get_include_options(self, includes):    raise RuntimeError
    ```
    
    (Note: this one has no message — just a bare `RuntimeError()`.)
- **Xcelium `_build_command()`** — you passed `waves=True` to `.build()` instead of `.test()`:
    
    ```python
    raise RuntimeError("waves is not supported in the build step. Please set it in the test step.")
    ```
    

## `OSError` (and its `PermissionError`/`FileNotFoundError` subclasses)

Not raised explicitly by cocotb — it's whatever the OS/filesystem calls do when they fail:

- `self.build_dir.mkdir(parents=True, exist_ok=True)` inside `build()` — permission denied, invalid path, disk full, etc.
- `open(self.log_file, "w")` inside `_execute()` — same causes, applied to the log file path you supplied.

---

**The distinction that matters for your `run_build` wrappers**: `ValueError`/`TypeError`/`RuntimeError` all happen in Python code _before_ any external process starts — they're pure configuration mistakes, deterministic, and retrying won't help. `CalledProcessError` and `OSError` happen at the OS/subprocess boundary — `OSError` before the compiler even launches (bad path), `CalledProcessError` after it launched and failed (bad HDL, license issue, tool crash). That boundary is exactly why Design 3's retry logic earlier treated the first group as fatal and the second as potentially transient.