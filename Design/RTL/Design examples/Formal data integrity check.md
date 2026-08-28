# Input-Recording / Output-Order Checker — Formal Verification Notes

## 1. The Idea

The goal is to verify that a DUT correctly processes a sequence of input values and produces them (or a known transformation of them) on an output signal, **in the same order**, even though:

- Each distinct input value is held stable for a fixed period (5 cycles in this case), driven via an `assume`.
- The corresponding output value is known to be held for a **fixed period of 10 cycles** (different from the input's 5-cycle hold, but not unknown or variable).
- The output does not start "aligned" with the input sequence — some initial output cycles must be **skipped** until the first recorded input value is observed on the output.

The approach breaks the checker into three pieces:

1. **Recorder** — captures each new distinct input value (at each 5-cycle boundary) into a fixed-size array, in order.
2. **Synchronizer** — ignores the output completely until it first matches the first recorded value (handles the "skip initial output cycles" requirement).
3. **Checker** — once synchronized, walks through the recorded sequence, allowing the output to stay on the current expected value for any number of cycles, or advance to the next expected value — but never allows it to jump to something unexpected.

This is essentially a lightweight **scoreboard pattern**, adapted for formal (bounded arrays instead of dynamic queues, procedural helper state feeding a clean `assert property`).

---

## 2. The Input-Side Constraint (context)

Before the recorder/checker, the input signal itself is constrained to change value only every 5 cycles:

```systemverilog
// 5-cycle counter (0..4), wraps around
logic [2:0] period_cnt;

always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        period_cnt <= 3'd0;
    else
        period_cnt <= (period_cnt == 3'd4) ? 3'd0 : period_cnt + 3'd1;
end

// 1) Hold the value stable for 5 cycles
assume property (@(posedge clk) disable iff (!rst_n)
    (period_cnt != 3'd0) |-> $stable(my_sig)
);

// 2) Force a *different* value at the boundary (every 5th cycle)
//    $past(rst_n) avoids referencing garbage/pre-reset values on the
//    first post-reset cycle.
assume property (@(posedge clk) disable iff (!rst_n)
    (period_cnt == 3'd0 && $past(rst_n)) |-> (my_sig != $past(my_sig))
);
```

---

## 3. The Recorder / Synchronizer / Checker Code

```systemverilog
localparam int DEPTH = 8;              // max distinct values to track
localparam int W     = $bits(my_sig);

// --- 1) Recorder: store each new input value, in order ---
logic [W-1:0] recorded_vals [DEPTH];
int unsigned  wr_ptr;

always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        wr_ptr <= 0;
    end else if (period_cnt == 3'd0 && $past(rst_n)) begin
        if (wr_ptr < DEPTH) begin
            recorded_vals[wr_ptr] <= my_sig;
            wr_ptr                <= wr_ptr + 1;
        end
    end
end

// Bound the recorder so it never overflows the array during proof
assume property (@(posedge clk) disable iff (!rst_n) wr_ptr < DEPTH);

// --- 2) Synchronizer: skip output cycles until first match ---
logic        sync_found;
int unsigned rd_ptr;

always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        sync_found <= 1'b0;
        rd_ptr     <= 0;
    end else if (!sync_found && wr_ptr > 0) begin
        if (out_sig == recorded_vals[0]) begin
            sync_found <= 1'b1;
            rd_ptr     <= 0;
        end
    end
end

// --- 3) Checker: walk the sequence, tolerating variable hold length ---
always_ff @(posedge clk or negedge rst_n) begin
    if (rst_n && sync_found && rd_ptr < wr_ptr) begin

        if (out_sig == recorded_vals[rd_ptr]) begin
            // still on the same expected value — held N cycles, fine
        end
        else if (rd_ptr + 1 < wr_ptr && out_sig == recorded_vals[rd_ptr + 1]) begin
            // advanced to the next expected value in order
            rd_ptr <= rd_ptr + 1;
        end
        else begin
            a_seq_mismatch: assert (0)
                else $error("Output sequence mismatch: got %0h, expected %0h or %0h",
                            out_sig, recorded_vals[rd_ptr], recorded_vals[rd_ptr+1]);
        end
    end
end

// --- Coverage: confirm sync actually happens, and sequence completes ---
c_sync_happens:   cover property (@(posedge clk) sync_found);
c_seq_complete:   cover property (@(posedge clk)
                      rd_ptr == wr_ptr - 1 && out_sig == recorded_vals[wr_ptr-1]);
```

---

## 4. Assessment — Is This a Normal Formal Verification Pattern?

### What's standard practice

- **Recording values + comparing against a later signal** is a well-known technique, often called a **scoreboard** or lightweight **reference model**. Very common for FIFOs, pipelines, reorder buffers — anything needing "data in must equal data out, in order."
- **Skipping initial cycles until synchronization** is also standard — a small "alignment"/"phase-lock" state machine, common when the exact pipeline latency isn't known or fixed.

### What's less standard, and worth reconsidering

1. **Known, fixed hold duration (10 cycles) with no qualifying handshake signal.** Since the output hold length is actually known and fixed (10 cycles), the "variable hold duration" complexity in the checker above is more than what's needed — the `rd_ptr`/"advance on next expected value" polling logic could be replaced with a much simpler cycle-counted check. If the DUT additionally has any `valid` / `ready` / `enable` signal, the idiomatic (and simplest) formal pattern is still to qualify the comparison on that signal instead of polling `out_sig` every cycle:
    
    ```systemverilog
    a_data_order: assert property (@(posedge clk) disable iff (!rst_n)
        (out_valid && out_ready) |-> (out_data == recorded_vals[rd_ptr])
    );
    ```
    
    This removes the entire "how many cycles is a value held" ambiguity, because the comparison only happens on the cycle the value is actually consumed. **If no such handshake exists**, it's worth asking whether the DUT's output timing is genuinely unspecified — which may indicate the assumptions around the environment need tightening before this is a good formal target.
    
2. **State-space cost.** Every element of `recorded_vals[]`, plus `wr_ptr` and `rd_ptr`, adds state the solver must reason about. `DEPTH = 8` should be fine; scaling this up (say, 64+) risks hurting proof convergence significantly. DEPTH should be picked based on the minimum number of distinct values actually needed to demonstrate correctness — not set arbitrarily high "to be safe."
    
3. **Procedural `assert (0)` vs. native `assert property`.** The pass/fail check itself is a clean `assert property`-style construct triggered from procedural helper state, which is fine — but it's worth being aware that procedural assertions can be harder to trace in some tools' waveform/ counterexample views compared to a single self-contained SVA property. Keeping the _detection logic_ (recorder, pointers) separate from the _check_ (as done here) is the right structure to minimize that friction.
    
4. **Sync never happening.** If `out_sig` never matches `recorded_vals[0]`, the checker simply never activates — silently. This could mask real bugs. The `c_sync_happens` cover property is essential here: if it's never hit, the whole checker is vacuous and the "proof" is meaningless.
    
5. **This is effectively a simulation-style scoreboard, re-implemented for formal.** That's legitimate when the transformation between input and output is a black box (unknown operation), but if the operation is actually a known, simple function (identity, delay, arithmetic op), an **equivalence-style or algebraic property** would likely be simpler, cheaper to prove, and easier to debug than a full order-tracking scoreboard.
    

### Bottom line

- The overall recorder → synchronizer → checker structure is a legitimate and fairly common formal pattern.
- The main open question is whether `out_sig` has an associated handshake signal. If it does, this whole construct should be simplified to a handshake-qualified comparison — dropping the need for `sync_found`, `rd_ptr` polling, and most of the "variable hold duration" complexity.
- If there truly is no handshake and hold duration is genuinely unconstrained, that's a signal worth double-checking with the design intent before trusting the formal result — it may indicate the environment assumptions are incomplete rather than that this checker pattern is wrong.

---

## 5. When to Use This Pattern vs. When Not To

### Use the recorder/synchronizer/checker pattern when:

- The exact cycle at which each output value should be sampled is **not known or not fixed** — e.g. it depends on internal DUT state, arbitration, variable-latency processing, or backpressure with no exposed handshake signal to qualify on.
- The relationship between input and output is order-preserving but otherwise a black box (you don't know or don't want to hardcode the transformation/timing), so you genuinely need to track the _sequence_ of values rather than a specific cycle offset.
- You're checking something inherently variable-depth, like a FIFO, reorder buffer, or arbitrated multi-source pipeline, where "N cycles later" isn't a meaningful statement in general.

### Do NOT use this pattern — prefer a simple fixed-cycle check — when:

- **You already know the exact cycle/offset at which the output should be sampled** (as in the current case: input held 5 cycles, output known to be held 10 cycles). In that situation, a direct fixed-latency property is simpler, easier to read, easier to debug, and easier to explain to someone else reviewing the proof:
    
    ```systemverilog
    // Example: output is a known, fixed number of cycles behind the input
    // boundary — no recorder, no pointers, no synchronization needed.
    assert property (@(posedge clk) disable iff (!rst_n)
        (period_cnt == 3'd0 && $past(rst_n)) |->
            ##10 (out_sig == $past(my_sig, 10))
    );
    ```
    
    This single property expresses exactly the same intent as the entire recorder/synchronizer/checker construct, with none of the extra state (`recorded_vals[]`, `wr_ptr`, `rd_ptr`, `sync_found`), none of the ambiguity about when sync happens, and a counterexample waveform that directly shows the one relevant input/output pair — instead of requiring a reviewer to trace through array contents and pointer values.
    

### The trade-off, in short

||Recorder/Sync/Checker (this doc)|Fixed-cycle property|
|---|---|---|
|Robustness to unknown timing|High — works even if latency is unknown/variable|None — breaks if the assumed offset is wrong|
|State-space cost|Higher (array + pointers)|Minimal (just a delayed compare)|
|Debuggability|Harder — failures require tracing pointer/array state|Easy — counterexample directly shows input vs. output at the known offset|
|Explainability to others|Requires walking through 3 pieces of logic|One line, self-explanatory|
|When latency is actually known|Overkill|**Correct choice**|

**In this specific case** — now that the output hold duration is known to be a fixed 10 cycles — the fixed-cycle property is very likely the better choice. The recorder/synchronizer/checker pattern remains valuable to keep in mind for future cases where the latency or ordering truly isn't known ahead of time, but it should not be the default reach when a simple `##N` fixed-latency assertion says the same thing.