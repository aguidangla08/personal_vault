# Signals

## Signal & Port Naming

|Convention|Example|Purpose|
|---|---|---|
|`_i` suffix|`data_i`, `clk_i`|Input port|
|`_o` suffix|`data_o`, `valid_o`|Output port|
|`_io` / `_b` suffix|`data_io`|Bidirectional/inout port|
|`_n` or `n_` prefix|`rst_n`, `n_cs`|Active-low signal|
|`_reg` / `_r` suffix|`count_reg`, `state_r`|Registered (flopped) signal|
|`_next` / `_nxt` / `_d` suffix|`state_next`, `count_d`|Combinational "next state" feeding a register|

## Clock, Reset & Timing

- `clk`, `clk_100m`, `clk_div2` — clock signals, name embeds frequency/division if relevant
- `rst`, `rst_n`, `rstn`, `arst_n` (async), `srst` (sync) — clearly distinguish sync vs async, active-high vs active-low
- `_pe` / `_posedge`, `_ne` / `_negedge` — edge-triggered signal naming when relevant

## FSM (Finite State Machine)

- `state`, `state_reg`, `state_next` / `nstate`
- State enum values in ALL_CAPS with a common prefix: `IDLE`, `S_IDLE`, `FSM_IDLE`
- `is_<state>` for derived state flags: `is_idle`, `is_busy` -`_sel` — selects among predefined options/encodings (e.g. `op_sel`, `mux_sel`)

## Handshake / Protocol Signals

- `valid`, `ready`, `_vld`, `_rdy` — standard valid/ready handshake (AXI-style)
- `req` / `ack`, `req` / `gack` — request/acknowledge handshake
- `_en` — enable
- `_hold`, `_stall` — pipeline control

## Status & State Indicator Signals

Signals that report a condition, event, or retained value — as opposed to control signals that _cause_ something to happen.

### Flag

`_flag` is a generic suffix used for a **single-bit status/indicator signal** — something that signals "a condition is true right now" or "an event happened," rather than carrying data itself.

**Typical meaning:** a boolean that reports a state or event, often read-only from the consumer's perspective (set by logic, checked elsewhere).

**Examples:**

- `error_flag` — an error condition occurred
- `overflow_flag` — a counter/buffer overflowed
- `done_flag` — an operation completed
- `busy_flag` — the block is currently busy
- `hold_flag` — holding is currently active

**How it differs from other suffixes:**

- `_flag` — generic "this condition is currently true" (status indicator)
- `_en` — an enable/control input that causes something to happen
- `_valid` / `_vld` — specifically means "the accompanying data is valid this cycle" (handshake semantics)
- `_pending` — condition is queued/waiting, not necessarily active now
- `_held` — specifically means "a value is being retained," a more specific case of `_flag`

Here's the updated **Held / Captured** section with a clearer explanation of `hold` vs `held`, plus a direct clarification of what `count_hold` means:

### Held / Captured

For a signal that indicates a value was held (captured/frozen) under certain conditions:

- `_hold` suffix — e.g. `data_hold`, `count_hold` — for the held value itself, or the control signal that triggers holding
- `_latched` suffix — e.g. `data_latched` — if the semantic is "this was captured and retained," closer to a latch/sample-and-hold meaning
- `_captured` suffix — e.g. `err_captured` — common when the value is grabbed on an event and kept until cleared

For a flag that indicates "holding is active" (a status/condition flag, not the value itself):

- `_hold` suffix as a boolean — e.g. `pipe_hold`, `out_hold` — matches the `_hold` / `_stall` entry under Handshake/Protocol Signals
- `is_held` / `_held` suffix — e.g. `value_held`, `is_held` — reads naturally as "this is currently in a held state"
- `_hold_flag` — more explicit if you want to distinguish it clearly from a hold control signal — e.g. `data_hold_flag`

**`hold` vs `held` — grammatical tense signals direction of causality:**

- **`hold`** (present tense / imperative) → a _command_ or _active condition_ signal. It tells something to freeze **now**, or reports that freezing is happening **right now**. Example: `count_hold = 1` means "hold the counter _this cycle_" — an instruction, not a report of history.
- **`held`** (past tense) → a _state_ signal. It reports that something _has been_ frozen — the action already happened and the result persists. Example: `count_held = 1` means "the counter _is currently in a held state_," describing a condition that resulted from a past hold event.

**So what is `count_hold` specifically?**

Because it uses `_hold` (not `_held`), `count_hold` is ambiguous by itself — it could mean either:

1. A **control signal** driving the counter: "assert this to freeze the counter" (input to the counter logic), or
2. A **boolean status output** meaning "the counter is holding right now" (used as a `_hold` status per the second bullet list above, which explicitly allows `_hold` as a boolean too).

This is exactly why the doc lists `_hold` in _both_ categories — as the control/value name **and** as an acceptable status flag name. If you want to remove that ambiguity in your own signals, the cleaner convention is:

- Use `_hold` **only** for the control/cause signal (e.g. `count_hold_en` or just `count_hold` as an input that triggers holding)
- Use `_held` **only** for the resulting status (e.g. `count_held` as an output reporting "currently frozen")

That way `count_hold` (input, cause) and `count_held` (output, effect) read unambiguously as a cause/effect pair.

### Sample / Capture Trigger (future action)

For a signal that indicates a value **will be collected on a given cycle** — a forward-looking trigger, as opposed to `_held`/`_captured` which report that collection already happened:

- `_sample_en` — most idiomatic; reads as "sampling is enabled/happening this cycle." Fits the `_en` convention (control signal that causes something) from the Handshake/Protocol section
- `_capture_en` — same pattern, used when "capture" is the more natural verb for the context (e.g. grabbing a value into a register on an event)
- `_sample_pulse` — use when the signal is a single-cycle strobe rather than a level-style enable (e.g. generated by a pulse generator or edge detector)
- `_capture_strobe` — same idea as `_sample_pulse`; "strobe" is a common term of art for a one-cycle pulse that triggers a capture
- `_sample` (bare) — acceptable but slightly ambiguous on its own; less preferred unless context makes clear it's a trigger and not a captured value

Distinguish from `_hold` / `_held`: `_sample_en` / `_capture_en` mark the single cycle on which collection occurs, while `_hold` / `_held` describe retaining that value across the cycles that follow.

### Sticky

For "sticky" behavior — a signal that gets set once a condition occurs and stays set for the rest of the window (even if the original condition goes away):

- **`_sticky`** — e.g. `valid_sticky`, `err_sticky` — the most standard verification term for "latched until explicitly cleared," widely understood in RTL/verification
- **`_seen`** — e.g. `valid_seen` — reads naturally as "was this observed at least once"
- **`_ever`** — e.g. `valid_ever`, `hold_ever` — less common but sometimes used ("did it ever happen")
- **`_once`** — e.g. `valid_once`, `seen_once` — emphasizes "happened at least once" more than the sticky/latch mechanism
- **`_latched`** — e.g. `valid_latched` — usable, but slightly ambiguous since `_latched` more commonly implies a _value_ was captured, not just an event flag
- **`_window_`** or **`_win_`** — "window" time

**Most idiomatic choice:** `valid_sticky` or `valid_seen_flag` (combining `_seen` + `_flag`) — clearest for a reviewer to understand "this bit turns 1 the first time valid asserts in the window, and stays 1 until the window resets/clears."

If the window itself resets the flag, some teams also name the clear condition explicitly, e.g. `valid_sticky` + `valid_sticky_clr`.

## Parameters, Constants, Macros

- Parameters/localparams: ALL_CAPS with underscores — `DATA_WIDTH`, `ADDR_WIDTH`, `FIFO_DEPTH`
- `` `define `` macros: ALL_CAPS, often prefixed by module/block — `` `MOD_TIMEOUT_CYCLES ``
- Generate block labels: `gen_<purpose>` — `gen_pipeline_stage`

## Verification-Specific (Testbench / UVM)

|Convention|Example|Purpose|
|---|---|---|
|`_pkg`|`axi_pkg`|SystemVerilog package|

## Module / File / Instance Naming

- Module names: lowercase snake_case — `fifo_sync`, `axi_master_wrapper`
- File name matches module name: `fifo_sync.v` / `fifo_sync.sv`
- Instance names prefixed `u_` or `i_`: `u_fifo`, `i_arbiter`
- Generated/array instances indexed clearly: `u_lane[i]`, `gen_lane[i].u_core`

## Assertions & Checkers (SVA)

- `_assert` / `_a` suffix — `req_ack_a`
- `_cover` / `_c` suffix — `fifo_full_c`
- `_property` / `_p` suffix — `no_overflow_p`
- Prefix by block/module for traceability: `fifo_no_overflow_a`