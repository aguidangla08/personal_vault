```systemverilog
module diff_example (
  input  logic clk,
  input  logic rst,
  input  logic a,
  input  logic b,
  output logic c_ifonly,   // Version A
  output logic c_full      // Version B
);

  always_ff @(posedge clk or posedge rst) begin
    if (rst) begin
      c_ifonly <= 1'b0;
      c_full   <= 1'b0;
    end else begin
      // A: only handles true case
      if (a && b)
        c_ifonly <= 1'b1;

      // B: handles both cases explicitly
      c_full <= (a && b);
    end
  end

endmodule
```

## Waveform-style trace
**wavedrom**
```json
{
  "signal": [
    { "name": "clk",      "wave": "p........." },
    {},
    { "name": "rst",      "wave": "10........" },
    { "name": "a",        "wave": "0.10.1.0.." },
    { "name": "b",        "wave": "0.10......" },
    {},
    { "name": "a && b",   "wave": "0.10......", "node": ".....A" },
    {},
    { "name": "c_ifonly (if, no else)", "wave": "0.1.......", "node": "..B" },
    { "name": "c_full (c <= a&&b)",     "wave": "0.10......", "node": "..C" }
  ],
  "edge": [
    "A~>B set on match",
    "A~>C tracks match"
  ],
  "head": {
    "text": "if(a&&b) c<=1  (sticky, never clears)   vs   c <= (a&&b)  (tracks combinationally)"
  },
  "config": {
    "hscale": 1.5
  }
}
```

![[always_if_vs_comb.png]]

|cycle|a|b|a&&b|c_ifonly (A)|c_full (B)|
|---|---|---|---|---|---|
|0|0|0|0|0|0|
|1|1|1|1|1|1|
|2|0|0|0|**1** (held!)|**0**|
|3|0|0|0|**1** (held!)|0|
|4|1|0|0|**1** (held!)|0|
|5|0|0|0|**1** (held!)|0|

`c_ifonly` gets set to 1 on cycle 1 and then **never comes back down** — because nothing in the always_ff ever tells it to. It's not combinational-latch inference (this is `always_ff`, still a proper flop), it's just a flop that only has a "set" input path and no "clear" input path in your code, so it holds its last value whenever the if-condition is false.

`c_full` faithfully tracks `a && b` every single cycle, going high only when both are true and low the instant either goes false.

## Why this matters in practice

### `c <= (a && b);` — combinational tracking

The register is a **pure reflection** of current inputs, delayed by one clock. Use this when the output should always represent "what is true _right now_," with no memory of the past beyond one cycle.

**Use cases:**

- **Pipeline stage registers** — passing a computed value one stage down (e.g., `valid_stage2 <= valid_stage1 && !stall`)
- **Registered combinational outputs** — breaking a long combinational path for timing, where you want the _exact_ logic value, just delayed
- **Status signals that must self-correct** — e.g., `busy <= (state != IDLE)`, `overflow <= (sum > MAX)` — you want these to clear themselves the instant the condition is no longer true
- **Handshake/valid signals** — `req_valid <= req_a && req_b` — should drop as soon as inputs drop
- **Comparators / flags feeding downstream logic every cycle** — anything where "stale = wrong"

### `if (cond) c <= 1;` (no else) — sticky/latching register

The register **remembers** that something happened, and only forgets when you explicitly tell it to (reset, or another branch). This is a state-holding element, not a combinational echo.

**Use cases:**

- **Sticky error/fault flags** — `if (parity_error) err_flag <= 1;` — you want to know an error _ever_ occurred, even if the erroring condition is now gone, until software/logic explicitly clears it
- **Interrupt pending bits** — set on event, cleared only on ack/write-1-to-clear
- **"Seen it once" latches** — e.g., `if (start_pulse) started <= 1;` to convert a one-cycle pulse into a persistent level
- **FIFO/state "has been written" or "has been read" markers**
- **Counters, accumulators, enables that should only change on specific triggers** — e.g., `if (load) count <= data_in;` (else implicitly hold — you _want_ hold, since it's a register that should keep its value except when explicitly loaded)
- **Configuration/control registers written by software** — you only want them to change when a write-enable fires, otherwise hold indefinitely

### The real dividing line

Ask yourself: **"When the condition goes false, should the register go back to 0/its default, or should it keep whatever it last was?"**

- Goes back → combinational tracking (`c <= expr;`, or `if/else` covering both cases)
- Keeps its value → sticky (`if (cond) c <= ...;` with no else, relying on implicit hold)

The bug pattern from before happens when someone _means_ the first (self-correcting) but _writes_ the second (sticky) by forgetting the else branch — so if you ever see a bare `if` with no else in an `always_ff`, it's worth pausing and asking "is this hold-behavior intentional?"