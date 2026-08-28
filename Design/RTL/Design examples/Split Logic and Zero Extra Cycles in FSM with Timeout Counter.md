**RTL Design Guide: FSM + Timeout Counter**  
*(Split logic preferred – fully sequential alternative also shown)*

### Goal
Implement a state machine that uses a timeout counter in *some* states only, with clean start / stop / restart behaviour and **no extra latency cycles**.

---

### 1. Recommended Architecture – Split Logic (Preferred)

```
┌─────────────────────────────┐
│  always_comb                │
│  - next_state               │
│  - timeout_en               │
│  - timeout_clr / timeout_load│
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐     ┌─────────────────────────────┐
│  always_ff (state)          │     │  always_ff (counter)        │
│  state <= next_state        │     │  counts only when enabled   │
└─────────────────────────────┘     │  clears/loads on demand     │
                                    └─────────────────────────────┘
```

#### Key Principles (Split)
1. Controls (`timeout_en`, `timeout_clr`/`load`) are **combinatorial**.
2. Counter is sequential and independent.
3. Restart (clear/load) is asserted on the **transition into** a timed state → zero extra cycles.
4. Enable only while the timed state is active; otherwise the counter freezes.
5. **Never derive enable/clear from the registered state alone (that adds a cycle of lag)**.

#### Template – Split Logic

```systemverilog
// ------------------------------------------------------------
// Parameters & types
// ------------------------------------------------------------
localparam int TIMEOUT_MAX = 1000;
typedef enum logic [2:0] {
  S_IDLE,
  S_WAIT,          // uses timeout
  S_PROCESS,       // does not use timeout
  S_ERROR
} state_t;

// ------------------------------------------------------------
// Registers
// ------------------------------------------------------------
state_t state, next_state;

logic [15:0] timeout_cnt;
logic        timeout;

// Control signals (combinational)
logic        timeout_en;
logic        timeout_clr;

// ------------------------------------------------------------
// Sequential: State register
// ------------------------------------------------------------
always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n)
    state <= S_IDLE;
  else
    state <= next_state;
end

// ------------------------------------------------------------
// Sequential: Timeout counter (independent)
// ------------------------------------------------------------
always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n) begin
    timeout_cnt <= '0;
    timeout     <= 1'b0;
  end else begin
  // *** Clear behavior takes priority and sets the counter to 0
  // even when the counter is enabled
    if (timeout_clr) begin
      timeout_cnt <= '0;
      timeout     <= 1'b0;
    end else if (timeout_en) begin
      if (timeout_cnt >= TIMEOUT_MAX) begin
        timeout <= 1'b1;
      end else begin
        timeout_cnt <= timeout_cnt + 1'b1;
        timeout     <= 1'b0;
      end
    end
    // else: freeze
  end
end

// ------------------------------------------------------------
// Combinational: Next-state + timer controls
// ------------------------------------------------------------
always_comb begin
  // Defaults
  next_state  = state;
  timeout_en  = 1'b0;
  timeout_clr = 1'b0;

  unique case (state)
    S_IDLE: begin
      if (start_condition) begin
        next_state  = S_WAIT;
        timeout_clr = 1'b1;            // restart on entry
      end
    end

    S_WAIT: begin
      timeout_en = 1'b1;               // run while in this state

      if (got_response) begin
        next_state  = S_PROCESS;
        timeout_clr = 1'b1;            // optional clean stop
      end else if (timeout) begin
        next_state = S_ERROR;
      end
    end

    S_PROCESS: begin
      // timeout_en = 0 → counter frozen
      if (done)
        next_state = S_IDLE;
    end

    S_ERROR: begin
      if (ack_error)
        next_state = S_IDLE;
    end

    default: ;
  endcase
end
```

#### Timing (Split – zero extra cycles)

| Cycle | Action                                          | Result                  |
|-------|-------------------------------------------------|-------------------------|
| N     | Decide to enter `S_WAIT` + assert `timeout_clr` | Counter cleared         |
| N+1   | State = `S_WAIT`, `timeout_en = 1`              | Counter starts at 0     |
| N+2…  | Counter increments                              | Normal counting         |
| M     | Exit condition or timeout detected          | Clean transition, no lag   |

![[split_logic_counter.png]]

```json
{
  "signal": [
    {"name": "clk",          "wave": "p................................"},
    {},
    {"name": "state",        "wave": "2.3...........4.2.3.........5.2.",
                             "data": ["IDLE","WAIT","PROCESS","IDLE","WAIT","ERROR","IDLE"]},
    {},
    {"name": "start",        "wave": "010................10..........."},
    {"name": "got_response", "wave": "0............10................."},
    {"name": "timeout",      "wave": "0......................1.0......"},
    {},
    {"name": "timeout_clr",  "wave": "010..............10....10......."},
    {"name": "timeout_en",   "wave": "0.1...........0...1.......0....."},
    {},
    {"name": "timeout_cnt",  "wave": "x.2.=======...x.2.=====x.2.=...x",
                             "data": ["0","1","2","3","4","5","6","7","0","1","2","3","4","0","MAX"]},
    {},
    {"name": "Comments",     "wave": "x.2...........3.x.2.......4.x...",
                             "data": ["Start + clear","Clean exit","Start + clear","Timeout fired"]}
  ],
  "config": { "hscale": 1 },
  "head": {
    "text": "FSM + Timeout Counter (Split Logic) – Counting & Restart",
    "tick": 0
  }
}
```

---

### 2. Alternative – Everything Inside One `always_ff`

This style puts the state register, next-state decisions **and** the counter in a single sequential block. It is valid but usually harder to maintain and easier to get restart wrong.

#### Template – Fully Sequential

```systemverilog
// ------------------------------------------------------------
// Parameters & types
// ------------------------------------------------------------
localparam int TIMEOUT_MAX = 1000;
typedef enum logic [2:0] {
  S_IDLE,
  S_WAIT,
  S_PROCESS,
  S_ERROR
} state_t;

state_t      state;
logic [15:0] timeout_cnt;
logic        timeout;

// ------------------------------------------------------------
// Everything in one always_ff
// ------------------------------------------------------------
always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n) begin
    state       <= S_IDLE;
    timeout_cnt <= '0;
    timeout     <= 1'b0;
  end else begin
    // Optional default for sticky flag
    // timeout <= 1'b0;   // uncomment if you want pulse behaviour

    unique case (state)

      S_IDLE: begin
        if (start_condition) begin
          state       <= S_WAIT;
          timeout_cnt <= '0;          // restart on entry
          timeout     <= 1'b0;
        end
      end

      S_WAIT: begin
        if (got_response) begin
          state       <= S_PROCESS;
          timeout_cnt <= '0;          // optional clean-up
          timeout     <= 1'b0;
        end else if (timeout_cnt >= TIMEOUT_MAX) begin
          state   <= S_ERROR;
          timeout <= 1'b1;
          // timeout_cnt can stay at MAX or be cleared
        end else begin
          timeout_cnt <= timeout_cnt + 1'b1;  // count only while staying
          timeout     <= 1'b0;
        end
      end

      S_PROCESS: begin
        // Counter is frozen because we simply do not assign to it
        if (done) begin
          state <= S_IDLE;
        end
      end

      S_ERROR: begin
        if (ack_error) begin
          state       <= S_IDLE;
          timeout_cnt <= '0;
          timeout     <= 1'b0;
        end
      end

      default: begin
        state <= S_IDLE;
      end
    endcase
  end
end
```

#### Notes on the Fully Sequential Style

- Restart is done by assigning `timeout_cnt <= '0` on **every** transition that enters a timed state.
- While staying in a timed state you must explicitly increment the counter in that branch.
- In non-timed states you simply **do not assign** to `timeout_cnt` → it freezes.
- Missing a clear on one entry path is a very common bug.
- The `case` statement grows quickly and mixes control flow with counting logic → harder to read and review.

#### When the single-`always_ff` style is still acceptable
- Very small FSMs (1–2 timed states).
- Coding guidelines that mandate a single sequential process.
- You are extremely careful with all entry-path clears.

---

### 3. Comparison

| Aspect                        | Split Logic (Recommended)      | Fully Sequential              |
|-------------------------------|--------------------------------|-------------------------------|
| Extra cycles on restart       | None                           | None (if clears are correct)  |
| Readability                   | High                           | Lower                         |
| Risk of forgotten restart     | Low                            | Higher                        |
| Reusability of counter        | Easy (can be a module)         | Hard                          |
| Explicit enable/freeze        | Yes (`timeout_en`)             | Implicit (no assignment)      |
| Typical use                   | Most designs                   | Tiny FSMs only                |

---

### 4. Common Mistakes (Both Styles)

| Mistake                                      | Result                              | Fix |
|----------------------------------------------|-------------------------------------|-----|
| Clear derived only from registered state     | 1-cycle late clear                  | Clear on the transition |
| Forgetting clear on one entry path           | Counter continues from old value    | Clear on **all** entries |
| No defaults in combinational block           | Latches                             | Assign defaults first |
| Leaving timeout flag sticky with no clear    | FSM stuck in error path             | Provide a clear path |
| Incrementing counter outside the timed state | Unwanted counting                   | Enable only in timed states |

---

### 5. Checklist

**Split style**
- [ ] `timeout_en` / `timeout_clr` generated in `always_comb`
- [ ] Counter in its own `always_ff`
- [ ] Clear/load asserted on entry to every timed state
- [ ] Defaults assigned at top of combinational block
- [ ] Counter freezes in non-timed states

**Fully sequential style**
- [ ] `timeout_cnt <= '0` on every transition into a timed state
- [ ] Explicit increment only in the timed-state branch
- [ ] No assignment to counter in other states (freeze)
- [ ] Timeout flag has a defined clear path

---

**Recommendation**  
Prefer the **split-logic** version for any non-trivial design. It keeps the FSM readable, makes start/stop/restart explicit, avoids extra cycles, and is far less prone to the classic “counter won’t restart cleanly” bugs. Use the single-`always_ff` version only for the simplest cases.