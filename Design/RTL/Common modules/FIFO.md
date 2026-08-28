# 1. signals

|Signal|Typical name|What it really is|Width|Purpose|
|---|---|---|---|---|
|**Depth**|`DEPTH`|FIFO capacity (number of entries)|Parameter (e.g. 16)|Defines how many words the FIFO can store|
|**Addr Width**|`ADDR_WIDTH`|Bits needed to index the RAM|`$clog2(DEPTH)`|Sets width of `wr_addr`/`rd_addr`|
|**Pointer**|`wr_ptr`, `rd_ptr`|The full tracking value|`ADDR_WIDTH` or `ADDR_WIDTH+1`|Keeps track of position + full/empty status|
|**Address**|`wr_addr`, `rd_addr`|The actual index into the RAM|`ADDR_WIDTH`|Directly drives the RAM address port|
|**Counter**|`fifo_count`, `wr_count`|Number of valid entries currently stored|`$clog2(DEPTH+1)`|Used to detect full/empty/almost-full conditions|

# 2. Address and pointer relation

## 2.1. Counter computation address vs pointer

### Implementation 1 — Pointer-based (correct)

```verilog
reg [ADDR_WIDTH:0] wr_ptr, rd_ptr;   // full pointers, extra bit
wire [ADDR_WIDTH:0] fifo_count;

assign fifo_count = wr_ptr - rd_ptr; // always correct, 0..DEPTH
```

### Implementation 2 — Address-based (ambiguous)

```verilog
reg [ADDR_WIDTH-1:0] wr_addr, rd_addr; // wrapped addresses only
wire [ADDR_WIDTH-1:0] fifo_count_addr;

assign fifo_count_addr = wr_addr - rd_addr; // breaks at full/empty
```

### Table (DEPTH = 4, ADDR_WIDTH = 2), 4 pushes, no reads

|Step|wr_ptr (3b)|rd_ptr (3b)|wr_addr (2b)|rd_addr (2b)|counter (pointer)|counter (address)|
|---|---|---|---|---|---|---|
|Empty|000|000|00|00|0|0|
|Push 1|001|000|01|00|1|1|
|Push 2|010|000|10|00|2|2|
|Push 3|011|000|11|00|3|3|
|Push 4 (**FULL**)|100|000|00|00|**4** ✅|**0** ❌|

At the last row, the address-based method reports `0` (looks empty) when the FIFO is actually **full** — same addresses, same result. The pointer method disambiguates because `wr_ptr` rolled into the extra bit (`100`) instead of wrapping back to `000`.

So: use pointers for the counter/full-empty logic, and only truncate to `wr_addr`/`rd_addr` for driving the RAM.
## Why pointer and address exist

### 1.1. Extra-MSB technique (most common reason)
**Declaration**
```vhdl
-- Full pointer (with extra bit)
signal wr_ptr : unsigned(ADDR_WIDTH downto 0);   -- e.g. 5 bits for DEPTH=16

-- Real address that goes to the RAM
signal wr_addr : std_logic_vector(ADDR_WIDTH-1 downto 0);
wr_addr <= std_logic_vector(wr_ptr(ADDR_WIDTH-1 downto 0));  -- only lower bits
```

```verilog
// Full pointer (with extra bit)
reg [ADDR_WIDTH:0] wr_ptr;   // e.g. 5 bits for DEPTH=16

// Real address that goes to the RAM
wire [ADDR_WIDTH-1:0] wr_addr;
assign wr_addr = wr_ptr[ADDR_WIDTH-1:0];  // only lower bits
```

**Use**
```systemverilog
// -------------------------------------------------------
// Write pointer
// -------------------------------------------------------
always_ff @(posedge clk or negedge rst_n) begin
if (!rst_n)
  wr_ptr <= '0;
else if (wr_en && !full)
  wr_ptr <= wr_ptr + 1'b1;
end

// -------------------------------------------------------
// Memory write / read
// -------------------------------------------------------
always_ff @(posedge clk) begin
if (wr_en && !full)
  mem[wr_addr] <= wr_data;
end
assign rd_data = mem[rd_addr];


// -------------------------------------------------------
// Full / Empty detection (the magic)
// -------------------------------------------------------
assign empty = (wr_ptr == rd_ptr);

assign full = (wr_ptr[ADDR_W] != rd_ptr[ADDR_W]) && // MSBs different
              (wr_ptr[ADDR_W-1:0] == rd_ptr[ADDR_W-1:0]); // lower bits equal
```

- The **extra MSB** is used only to distinguish full vs empty.
- The RAM only needs the lower bits (0 … DEPTH-1).

|Operation|wr_ptr (3 bits)|rd_ptr (3 bits)|empty|full|
|---|---|---|---|---|
|Reset|000|000|1|0|
|Write 1|001|000|0|0|
|Write 2|010|000|0|0|
|Write 3|011|000|0|0|
|Write 4 (full)|100|000|0|**1**|
|Read 1|100|001|0|0|
|…|…|…|…|…|
|Read until empty|100|100|**1**|0|

### 2. Gray coding (asynchronous FIFOs)

```vhdl
signal wr_ptr_bin  : unsigned(ADDR_WIDTH downto 0);  -- binary pointer
signal wr_ptr_gray : std_logic_vector(...);          -- Gray-coded version (for CDC)
signal wr_addr     : std_logic_vector(ADDR_WIDTH-1 downto 0); -- binary address to RAM
```

- Binary pointer → used for arithmetic and for the RAM address.
- Gray pointer → sent across clock domains (safe for synchronizers).
- Address = lower bits of the binary pointer.

### 3. Registered / pipelined address

Sometimes the pointer is updated combinatorially and the address that actually goes to the RAM is registered (to meet timing or to match RAM read latency).

```vhdl
wr_ptr  <= wr_ptr + 1;               -- combinatorial or sequential
wr_addr <= std_logic_vector(wr_ptr); -- registered version
```

### 4. Simple naming convenience

Even in a basic synchronous FIFO many designers write:

```vhdl
signal wr_ptr  : unsigned(ADDR_WIDTH-1 downto 0);
signal wr_addr : std_logic_vector(ADDR_WIDTH-1 downto 0);

wr_addr <= std_logic_vector(wr_ptr);  -- just a type conversion
```

This is done purely for clarity or because the RAM port expects `std_logic_vector` while the counter is `unsigned`.

---

## Parameterizing by DEPTH instead of ADDR_WIDTH

In many real designs it's more natural to specify the FIFO in terms of how many entries it holds (`DEPTH`) rather than the raw address width. `ADDR_WIDTH` is then _derived_ from `DEPTH` using `$clog2`, the ceiling log base 2 function:

```
ADDR_WIDTH = $clog2(DEPTH)
```

`$clog2` gives the minimum number of address bits needed to index `DEPTH` locations. For example, `DEPTH = 16` → `$clog2(16) = 4` → a 4-bit address, matching `2^4 = 16` entries exactly.

If `DEPTH` isn't a power of 2 (e.g. `DEPTH = 10`), `$clog2(10)` still evaluates to `4` — you get 16 addressable RAM slots but only 10 are meant to be used, so the pointer wrap logic needs special handling (see note below).

### Verilog RTL — DEPTH-parameterized synchronous FIFO

```verilog
module sync_fifo #(
    parameter DATA_WIDTH = 8,
    parameter DEPTH      = 16                 // number of entries, e.g. 16
)(
    input                        clk,
    input                        rst_n,
    input                        wr_en,
    input  [DATA_WIDTH-1:0]      wr_data,
    input                        rd_en,
    output [DATA_WIDTH-1:0]      rd_data,
    output                       full,
    output                       empty,
    output [$clog2(DEPTH):0]     count
);

    localparam ADDR_WIDTH = $clog2(DEPTH);    // derived, not passed in

    reg [ADDR_WIDTH:0] wr_ptr, rd_ptr;        // ADDR_WIDTH+1 bits (extra MSB for wrap)
    wire [ADDR_WIDTH-1:0] wr_addr, rd_addr;   // ADDR_WIDTH bits only — no +1, drives the RAM
    reg [DATA_WIDTH-1:0] mem [0:DEPTH-1];     // exactly DEPTH words

    // Address = pointer with the extra MSB dropped
    assign wr_addr = wr_ptr[ADDR_WIDTH-1:0];
    assign rd_addr = rd_ptr[ADDR_WIDTH-1:0];

    wire wr_valid = wr_en && !full;
    wire rd_valid = rd_en && !empty;

    // Write pointer
    always @(posedge clk or negedge rst_n)
        if (!rst_n)
            wr_ptr <= 0;
        else if (wr_valid)
            wr_ptr <= wr_ptr + 1'b1;

    // Read pointer
    always @(posedge clk or negedge rst_n)
        if (!rst_n)
            rd_ptr <= 0;
        else if (rd_valid)
            rd_ptr <= rd_ptr + 1'b1;

    // Memory access — uses the address signals, not the raw pointers
    always @(posedge clk)
        if (wr_valid)
            mem[wr_addr] <= wr_data;

    assign rd_data = mem[rd_addr];

    // Status flags
    assign empty = (wr_ptr == rd_ptr);
    assign full  = (wr_addr == rd_addr) &&
                   (wr_ptr[ADDR_WIDTH] != rd_ptr[ADDR_WIDTH]);
    assign count = wr_ptr - rd_ptr;   // only valid for power-of-2 DEPTH — see below

endmodule
```

> **Pointer vs. address width:** `wr_ptr`/`rd_ptr` are `ADDR_WIDTH+1` bits — the extra MSB exists purely so `full` and `empty` can be told apart. `wr_addr`/`rd_addr` are plain `ADDR_WIDTH` bits with **no +1**, since the RAM only has `DEPTH` locations and doesn't know or care about the wrap bit. The address is simply the pointer's lower `ADDR_WIDTH` bits (`wr_ptr[ADDR_WIDTH-1:0]`), assigned once and reused everywhere the RAM is accessed, instead of re-slicing the pointer inline at each access.

### Correcting `count` for non-power-of-2 `DEPTH`

`count = wr_ptr - rd_ptr` only works when the pointers are free-running binary counters that wrap naturally at `2^(ADDR_WIDTH+1)`. Once the pointer wraps explicitly at `DEPTH-1` (the non-power-of-2 case), that subtraction breaks near the wrap boundary. Two fixes:

**1. Modulo-DEPTH subtraction:**

```verilog
assign count = (wr_ptr >= rd_ptr) ? (wr_ptr - rd_ptr)
                                   : (wr_ptr - rd_ptr + DEPTH);
```

**2. Dedicated up/down counter (more robust, works for any DEPTH):**

```verilog
reg [$clog2(DEPTH):0] count;

always @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        count <= 0;
    else begin
        case ({wr_valid, rd_valid})
            2'b10:   count <= count + 1;   // write only
            2'b01:   count <= count - 1;   // read only
            default: count <= count;       // both or neither -> no net change
        endcase
    end
end

assign full  = (count == DEPTH);
assign empty = (count == 0);
```

This sidesteps pointer subtraction entirely — `full`/`empty` come straight from `count`, so the MSB-toggle trick isn't even needed.

### What changes vs. the `ADDR_WIDTH`-given version

|Before (`ADDR_WIDTH` given)|Now (`DEPTH` given)|
|---|---|
|`parameter ADDR_WIDTH = 4`|`parameter DEPTH = 16`|
|`mem [0:(1<<ADDR_WIDTH)-1]`|`mem [0:DEPTH-1]` → sized directly from depth|
|Implicit depth = `2^ADDR_WIDTH`|Explicit depth = whatever you choose|
|—|`localparam ADDR_WIDTH = $clog2(DEPTH)` derives address width internally|

### Note — non-power-of-2 `DEPTH`

If `DEPTH = 10`:

- `ADDR_WIDTH = $clog2(10) = 4` (need 4 bits to count up to 9).
- `mem` has exactly 10 words: `mem[0:9]`.
- But the pointer's lower bits naturally range over `0–15` (since it's 4 bits wide), so a free-running binary pointer would eventually try to index `mem[10]` through `mem[15]`, which don't exist. The MSB-toggle full/empty trick also assumes a clean power-of-2 wrap.

For non-power-of-2 depths, the pointer needs explicit wrap-at-`DEPTH` logic instead of relying on the counter's natural rollover, e.g.:

```verilog
wr_ptr <= (wr_ptr == DEPTH-1) ? 0 : wr_ptr + 1;
```

and the full/empty comparison is typically reworked to compare an explicit `count` against `DEPTH` directly, rather than relying purely on the MSB-wraparound trick.

---

## Mixed-width FIFO (different write/read address sizes)

When the write side and read side access the FIFO in **different data widths** (e.g. write 8 bits at a time, read 32 bits at a time, or vice versa), the write and read pointers are no longer directly comparable — one write and one read no longer represent the same amount of data. This is sometimes called an asymmetric FIFO.

### Core idea

Pick the **narrower width** as the common storage unit. The RAM is organized in narrow-width words; the wide side accesses multiple narrow words per transaction.

```
RATIO = WIDE_WIDTH / NARROW_WIDTH        (must be an integer, usually a power of 2)
```

### Address / pointer sizing

If the RAM is `DEPTH_NARROW` narrow-words deep:

```
ADDR_WIDTH_NARROW = $clog2(DEPTH_NARROW)
ADDR_WIDTH_WIDE    = ADDR_WIDTH_NARROW - $clog2(RATIO)
```

The wide-side pointer only needs `ADDR_WIDTH_WIDE` bits, since each wide access consumes `RATIO` narrow slots at once.

### Pointer increment rule

- **Narrow side:** pointer increments by **1** per access.
- **Wide side:** pointer increments by **1** in its own address space, which corresponds to `RATIO` narrow-word increments underneath.

To compare fullness/emptiness correctly, convert both pointers to a **common unit** — narrow-word units:

```verilog
// If write side is the wide one:
wr_ptr_narrow_equiv = wr_ptr_wide * RATIO;

// If read side is the wide one:
rd_ptr_narrow_equiv = rd_ptr_wide * RATIO;
```

Both equivalent pointers still carry the usual extra MSB for wrap disambiguation:

```verilog
assign empty = (wr_ptr_narrow_equiv == rd_ptr_narrow_equiv);
assign full  = (wr_ptr_narrow_equiv[ADDR_WIDTH_NARROW-1:0] == rd_ptr_narrow_equiv[ADDR_WIDTH_NARROW-1:0]) &&
               (wr_ptr_narrow_equiv[ADDR_WIDTH_NARROW]     != rd_ptr_narrow_equiv[ADDR_WIDTH_NARROW]);
```

### Count / occupancy

In narrow-word units:

```verilog
count_narrow_units = wr_ptr_narrow_equiv - rd_ptr_narrow_equiv;
```

For a fill level in **wide-word units** (e.g. "how many full wide-words are ready to read"), floor-divide by `RATIO`:

```
count_wide_words = count_narrow_units >> $clog2(RATIO);   // integer division by RATIO
```

This matters because the FIFO can hold a **partial wide-word** worth of narrow data — the wide side can't be told data is "ready" until at least `RATIO` narrow words have accumulated.

### Example Verilog RTL — narrow write, wide read (8→32 bit)

```verilog
module mixed_width_fifo #(
    parameter NARROW_WIDTH = 8,
    parameter WIDE_WIDTH   = 32,
    parameter DEPTH_NARROW = 16          // RAM depth in narrow words (power of 2)
)(
    input                              clk,
    input                              rst_n,

    // Narrow write side
    input                              wr_en,
    input  [NARROW_WIDTH-1:0]          wr_data,
    output                             full,

    // Wide read side
    input                               rd_en,
    output [WIDE_WIDTH-1:0]             rd_data,
    output                              empty
);

    localparam RATIO             = WIDE_WIDTH / NARROW_WIDTH;
    localparam ADDR_WIDTH_NARROW = $clog2(DEPTH_NARROW);
    localparam ADDR_WIDTH_WIDE   = ADDR_WIDTH_NARROW - $clog2(RATIO);

    reg [ADDR_WIDTH_NARROW:0] wr_ptr;              // narrow-unit pointer, +1 MSB (wrap bit)
    reg [ADDR_WIDTH_WIDE:0]   rd_ptr;               // wide-unit pointer, +1 MSB (wrap bit)
    reg [NARROW_WIDTH-1:0]    mem [0:DEPTH_NARROW-1];

    // Dedicated occupancy counter, always expressed in NARROW-word units.
    // Holds values 0..DEPTH_NARROW, so it needs one extra bit over ADDR_WIDTH_NARROW.
    reg [ADDR_WIDTH_NARROW:0] count;

    wire [ADDR_WIDTH_NARROW:0] rd_ptr_narrow_equiv = rd_ptr * RATIO;

    // Addresses that actually drive the RAM — ADDR_WIDTH_NARROW bits, no +1 wrap bit
    wire [ADDR_WIDTH_NARROW-1:0] wr_addr = wr_ptr[ADDR_WIDTH_NARROW-1:0];
    wire [ADDR_WIDTH_NARROW-1:0] rd_addr = rd_ptr_narrow_equiv[ADDR_WIDTH_NARROW-1:0];

    // full/empty now come straight from the counter instead of pointer comparison
    assign empty = (count == 0);
    assign full  = (count == DEPTH_NARROW);

    // A wide read can only fire once a full RATIO-sized group has accumulated
    wire wr_valid = wr_en && !full;
    wire rd_valid = rd_en && !empty && (count >= RATIO);

    // Narrow-side write pointer
    always @(posedge clk or negedge rst_n)
        if (!rst_n)
            wr_ptr <= 0;
        else if (wr_valid)
            wr_ptr <= wr_ptr + 1'b1;

    // Wide-side read pointer
    always @(posedge clk or negedge rst_n)
        if (!rst_n)
            rd_ptr <= 0;
        else if (rd_valid)
            rd_ptr <= rd_ptr + 1'b1;

    // Occupancy counter: +1 narrow word per write, -RATIO narrow words per wide read
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            count <= 0;
        else begin
            case ({wr_valid, rd_valid})
                2'b10:   count <= count + 1'b1;            // write only
                2'b01:   count <= count - RATIO;           // read only (consumes RATIO narrow words)
                2'b11:   count <= count + 1'b1 - RATIO;    // simultaneous write + read
                default: count <= count;                   // neither
            endcase
        end
    end

    // Narrow-word memory write — uses wr_addr, not the raw pointer
    always @(posedge clk)
        if (wr_valid)
            mem[wr_addr] <= wr_data;

    // Wide read: pack RATIO consecutive narrow words starting at rd_addr into one wide word
    genvar i;
    generate
        for (i = 0; i < RATIO; i = i + 1) begin : pack
            assign rd_data[(i+1)*NARROW_WIDTH-1 -: NARROW_WIDTH] = mem[rd_addr + i];
        end
    endgenerate

endmodule
```

> **Pointer vs. address width, mixed-width case:** `wr_ptr` (`ADDR_WIDTH_NARROW+1` bits) and `rd_ptr` (`ADDR_WIDTH_WIDE+1` bits) each carry their own extra MSB wrap bit. Neither the RAM's write port nor its read port ever sees that extra bit — `wr_addr` and `rd_addr` are both exactly `ADDR_WIDTH_NARROW` bits (the read side after converting to narrow-word-equivalent units), with the wrap bit dropped, since the RAM only has `DEPTH_NARROW` real locations.

> Note: gating `rd_valid` on `count >= RATIO` is the fix for the earlier simplification — the wide side now only reads once a full `RATIO`-word group has actually been written, instead of relying on raw pointer equality for `empty`. The counter also removes the need for the MSB-toggle trick on `full`/`empty`, since both flags come directly from `count`, matching the "dedicated up/down counter" approach used for the single-width DEPTH case earlier in this document.

### Key practical implications

|Concern|Detail|
|---|---|
|**RATIO must divide evenly**|`WIDE_WIDTH` must be an integer multiple of `NARROW_WIDTH` (usually a power of 2) — otherwise word alignment breaks.|
|**Partial-word flag**|Often needs an extra flag (e.g. `almost_empty` / `partial_word`) since the wide side can't consume less than `RATIO` narrow words.|
|**Full/empty asymmetry**|"Full" from the write side's perspective and "full" from the read side's perspective may need separate thresholds, since one side's single access may not align with the other's.|
|**Endianness / packing order**|You must define whether the first narrow word maps to the wide word's MSBs or LSBs — a design choice, not derived automatically.|
|**Async version**|If clocks also differ (mixed-width _and_ mixed-clock), Gray-code the narrow-unit-equivalent pointers before crossing domains, same as a standard async FIFO.|

---

## Summary

|Situation|Do you need both?|Reason|
|---|---|---|
|Synchronous FIFO + extra MSB|Yes|Extra bit for full/empty, lower bits for RAM|
|Asynchronous FIFO|Yes|Binary for RAM, Gray for CDC|
|Simple synchronous FIFO|Optional|Often just for type conversion or readability|
|FIFO without RAM (registers)|Usually no|No memory address needed|
|FIFO sized by `DEPTH` (non-pow-2)|Yes|Pointer must wrap at `DEPTH`, not at `2^ADDR_WIDTH`|
|Mixed-width FIFO (asymmetric)|Yes|Address = narrow RAM index; pointer = converted to a common (narrow-word) unit for comparison|