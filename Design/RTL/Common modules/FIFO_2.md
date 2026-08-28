# FIFO Design Guide: Pointers, Addresses, and Counters

---

## 0. Overview

A FIFO (First-In-First-Out buffer) stores data in a RAM/register array and uses two moving indices — a **write pointer** and a **read pointer** — to track where the next write and read should happen.

The key idea that trips people up: **the pointer is not the same thing as the RAM address.**

- The **address** only needs enough bits to index the `DEPTH` locations in the RAM (`0 … DEPTH-1`). It wraps around every `DEPTH` accesses.
- The **pointer** is the value your logic uses to detect _full_ and _empty_. If it wrapped the same way the address does, a full FIFO and an empty FIFO would look identical (same address, same wrapped value) — the logic couldn't tell them apart.

The fix: give the pointer **one extra bit** beyond what the address needs. The address is just the pointer's lower bits; the extra bit acts as a "lap counter" that lets full/empty be distinguished. This idea underlies almost every FIFO variant covered below.

---

## 1. Core Signals Reference

|Signal|Typical name|What it really is|Width|Purpose|
|---|---|---|---|---|
|**Depth**|`DEPTH`|FIFO capacity (number of entries)|Parameter (e.g. 16)|Defines how many words the FIFO can store|
|**Addr Width**|`ADDR_WIDTH`|Bits needed to index the RAM|`$clog2(DEPTH)`|Sets width of `wr_addr`/`rd_addr`|
|**Pointer**|`wr_ptr`, `rd_ptr`|The full tracking value|`ADDR_WIDTH+1`|Keeps track of position + full/empty status|
|**Address**|`wr_addr`, `rd_addr`|The actual index into the RAM|`ADDR_WIDTH`|Directly drives the RAM address port|
|**Counter**|`count`, `fifo_count`|Number of valid entries currently stored|`$clog2(DEPTH+1)`|Alternative way to detect full/empty/almost-full|

---

## 2. Minimal Working FIFO (Baseline Case)

Smallest complete synchronous FIFO, power-of-2 `DEPTH`, all core signals in one place.

```verilog
module sync_fifo #(
    parameter DATA_WIDTH = 8,
    parameter DEPTH      = 16                 // power of 2
)(
    input                     clk,
    input                     rst_n,
    input                     wr_en,
    input  [DATA_WIDTH-1:0]   wr_data,
    input                     rd_en,
    output [DATA_WIDTH-1:0]   rd_data,
    output                    full,
    output                    empty
);

    localparam ADDR_WIDTH = $clog2(DEPTH);

    reg [ADDR_WIDTH:0] wr_ptr, rd_ptr;        // pointer: ADDR_WIDTH+1 bits
    wire [ADDR_WIDTH-1:0] wr_addr, rd_addr;   // address: ADDR_WIDTH bits
    reg [DATA_WIDTH-1:0] mem [0:DEPTH-1];

    assign wr_addr = wr_ptr[ADDR_WIDTH-1:0];
    assign rd_addr = rd_ptr[ADDR_WIDTH-1:0];

    wire wr_valid = wr_en && !full;
    wire rd_valid = rd_en && !empty;

    always @(posedge clk or negedge rst_n)
        if (!rst_n) wr_ptr <= 0;
        else if (wr_valid) wr_ptr <= wr_ptr + 1'b1;

    always @(posedge clk or negedge rst_n)
        if (!rst_n) rd_ptr <= 0;
        else if (rd_valid) rd_ptr <= rd_ptr + 1'b1;

    always @(posedge clk)
        if (wr_valid) mem[wr_addr] <= wr_data;

    assign rd_data = mem[rd_addr];

    assign empty = (wr_ptr == rd_ptr);
    assign full  = (wr_addr == rd_addr) &&
                   (wr_ptr[ADDR_WIDTH] != rd_ptr[ADDR_WIDTH]);

endmodule
```

**Value table** (DEPTH = 4, ADDR_WIDTH = 2): push 4 times, then pop once.

|Step|wr_ptr|rd_ptr|wr_addr|rd_addr|empty|full|
|---|---|---|---|---|---|---|
|Reset|000|000|00|00|1|0|
|Push 1|001|000|01|00|0|0|
|Push 2|010|000|10|00|0|0|
|Push 3|011|000|11|00|0|0|
|Push 4|100|000|00|00|0|**1**|
|Pop 1|100|001|00|01|0|0|

---

## 3. Why the Extra Bit? (Pointer vs Address)

If you tried to detect full/empty using the wrapped **address** instead of the wider **pointer**, both conditions would look the same:

```verilog
// Address-based (ambiguous) — do NOT use for full/empty
wire [ADDR_WIDTH-1:0] fifo_count_addr = wr_addr - rd_addr;
```

|Step|wr_ptr (3b)|rd_ptr (3b)|wr_addr (2b)|rd_addr (2b)|counter (pointer)|counter (address)|
|---|---|---|---|---|---|---|
|Empty|000|000|00|00|0|0|
|Push 1|001|000|01|00|1|1|
|Push 2|010|000|10|00|2|2|
|Push 3|011|000|11|00|3|3|
|Push 4 (**FULL**)|100|000|00|00|**4** ✅|**0** ❌|

At the last row the address-based method reports `0` (looks empty) when the FIFO is actually full. The pointer method disambiguates because `wr_ptr` rolled into the extra bit (`100`) instead of wrapping back to `000`.

**Rule of thumb:** pointers (with the extra bit) drive full/empty logic; addresses (without it) drive the RAM.

---

## 4. Full/Empty Detection Methods

Two common approaches, both already used above:

**A. MSB-toggle comparison** (needs the extra-bit pointer)

```verilog
assign empty = (wr_ptr == rd_ptr);
assign full  = (wr_ptr[ADDR_WIDTH] != rd_ptr[ADDR_WIDTH]) &&
               (wr_ptr[ADDR_WIDTH-1:0] == rd_ptr[ADDR_WIDTH-1:0]);
```

**B. Dedicated up/down counter** (see §5 — sidesteps the pointer-width trick entirely)

```verilog
assign full  = (count == DEPTH);
assign empty = (count == 0);
```

|Method|Needs extra pointer bit?|Works for non-power-of-2 DEPTH?|Extra hardware|
|---|---|---|---|
|MSB-toggle (pointers)|Yes|Only with modulo-count fix (§5)|None — reuses pointers|
|Up/down counter|No|Yes, natively|One small adder/subtractor register|

---

## 5. Counter (Occupancy Tracking)

The counter answers a different question than the pointers: not _where_ to write/read next, but _how many valid entries are currently stored_. It's useful for full/empty detection, almost-full/almost-empty thresholds, and fill-level reporting — and it can be derived from the pointers or maintained independently.

### 5.1 Method 1 — Derived from pointer subtraction

Only valid when the pointers are free-running binary counters wrapping at a power-of-2 boundary (as in §2/§3):

```verilog
wire [ADDR_WIDTH:0] count = wr_ptr - rd_ptr;   // 0..DEPTH, correct for power-of-2 DEPTH
```

This works because the extra bit makes `wr_ptr - rd_ptr` behave like true unsigned distance around the circular buffer, even across a wrap.

### 5.2 Method 2 — Dedicated up/down counter

Independent register, incremented on a write, decremented on a read. Works for **any** `DEPTH`, power-of-2 or not, and removes the need for the MSB-toggle trick:

```verilog
reg [$clog2(DEPTH):0] count;

always @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        count <= 0;
    else begin
        case ({wr_valid, rd_valid})
            2'b10:   count <= count + 1'b1;   // write only
            2'b01:   count <= count - 1'b1;   // read only
            default: count <= count;          // both or neither -> no net change
        endcase
    end
end

assign full  = (count == DEPTH);
assign empty = (count == 0);
```

### 5.3 Comparing both methods

DEPTH = 4, one write followed by one simultaneous write+read, then one read:

|Step|wr_valid|rd_valid|count (pointer subtraction)|count (up/down register)|
|---|---|---|---|---|
|Reset|–|–|0|0|
|Write|1|0|1|1|
|Write+Read (same cycle)|1|1|1|1|
|Read|0|1|0|0|

Both give identical results here because `DEPTH` is a power of 2. The difference shows up once `DEPTH` is **not** a power of 2 — see §6.

|Aspect|Pointer subtraction|Up/down counter|
|---|---|---|
|Extra hardware|None (reuses pointers)|One `$clog2(DEPTH+1)`-bit adder/subtractor|
|Power-of-2 DEPTH|Works directly|Works directly|
|Non-power-of-2 DEPTH|Needs modulo correction (§6)|Works unmodified|
|Simultaneous read+write|Handled automatically by subtraction|Needs explicit `2'b11` case|

---

## 6. Parameterizing by DEPTH vs ADDR_WIDTH

It's often more natural to specify `DEPTH` (number of entries) and derive `ADDR_WIDTH`:

```
ADDR_WIDTH = $clog2(DEPTH)
```

`$clog2` gives the minimum bits needed to index `DEPTH` locations. `DEPTH = 16` → `$clog2(16) = 4` → matches `2^4 = 16` exactly.

### Non-power-of-2 DEPTH

If `DEPTH = 10`, `$clog2(10) = 4` still gives 16 addressable slots, but only 10 are meant to be used. A free-running pointer would eventually try to index `mem[10]` through `mem[15]`, which don't exist. Two things need to change:

**1. Explicit pointer wrap:**

```verilog
wr_ptr <= (wr_ptr == DEPTH-1) ? 0 : wr_ptr + 1'b1;
```

**2. Corrected count (if using pointer subtraction, §5.1):**

```verilog
assign count = (wr_ptr >= rd_ptr) ? (wr_ptr - rd_ptr)
                                   : (wr_ptr - rd_ptr + DEPTH);
```

Or simply use the **dedicated up/down counter** from §5.2, which needs no modification for non-power-of-2 `DEPTH`.

|Before (`ADDR_WIDTH` given)|Now (`DEPTH` given)|
|---|---|
|`parameter ADDR_WIDTH = 4`|`parameter DEPTH = 16`|
|`mem [0:(1<<ADDR_WIDTH)-1]`|`mem [0:DEPTH-1]` — sized directly from depth|
|Implicit depth = `2^ADDR_WIDTH`|Explicit depth = whatever you choose|
|—|`localparam ADDR_WIDTH = $clog2(DEPTH)` derives width internally|

---

## 7. Asynchronous FIFOs (CDC)

When write and read sides run on **different clocks**, pointers must cross a clock-domain boundary. Comparing raw binary pointers across clock domains is unsafe — multiple bits can appear to change at once due to metastability, giving a corrupted intermediate value.

**Fix: Gray coding.** In a Gray-coded sequence, only **one bit** changes between consecutive values, so a synchronizer capturing a mid-transition value still reads either the old or the new value — never a bogus one.

```vhdl
signal wr_ptr_bin  : unsigned(ADDR_WIDTH downto 0);   -- binary pointer, used for arithmetic + RAM address
signal wr_ptr_gray : std_logic_vector(...);            -- Gray-coded copy, sent across clock domains
signal wr_addr     : std_logic_vector(ADDR_WIDTH-1 downto 0);  -- binary address, lower bits of wr_ptr_bin
```

|Signal|Domain|Coding|Purpose|
|---|---|---|---|
|`wr_ptr_bin` / `rd_ptr_bin`|Local clock|Binary|Arithmetic, RAM addressing|
|`wr_ptr_gray` / `rd_ptr_gray`|Local clock, sampled by the other domain|Gray|Safe to synchronize (1 bit changes at a time)|
|Synchronized Gray pointer|Other domain, after 2-flop sync|Gray|Compared to generate full/empty in that domain|

Full/empty are then computed by comparing the **synchronized Gray pointer** against the local Gray pointer — same MSB-toggle idea as §3/§4, just expressed in Gray code.

---

## 8. Mixed-Width (Asymmetric) FIFOs

When the write side and read side access data in **different widths** (e.g., write 8 bits, read 32 bits), one write and one read no longer represent equal amounts of data — pointers aren't directly comparable anymore.

### Core idea

Pick the **narrower width** as the common storage unit. The RAM is organized in narrow words; the wide side accesses `RATIO` narrow words per transaction.

```
RATIO = WIDE_WIDTH / NARROW_WIDTH     -- must be an integer, usually a power of 2
```

### Sizing

```
ADDR_WIDTH_NARROW = $clog2(DEPTH_NARROW)
ADDR_WIDTH_WIDE    = ADDR_WIDTH_NARROW - $clog2(RATIO)
```

### Converting to a common unit before comparing

```verilog
// If the wide side is the write side:
wr_ptr_narrow_equiv = wr_ptr_wide * RATIO;

// If the wide side is the read side:
rd_ptr_narrow_equiv = rd_ptr_wide * RATIO;
```

Occupancy in narrow-word units:

```verilog
count_narrow_units = wr_ptr_narrow_equiv - rd_ptr_narrow_equiv;
count_wide_words    = count_narrow_units >> $clog2(RATIO);   // floor division
```

The wide side can't be told data is "ready" until at least `RATIO` narrow words have accumulated — this partial-word condition needs an explicit gate (`count >= RATIO`), not just an `empty` check.

|Concern|Detail|
|---|---|
|RATIO must divide evenly|`WIDE_WIDTH` must be an integer multiple of `NARROW_WIDTH`|
|Partial-word flag|Wide side can't consume less than `RATIO` narrow words at once|
|Full/empty asymmetry|Each side's single access may not align with the other's|
|Packing order|First narrow word → wide word's MSBs or LSBs is a design choice|
|Async + mixed-width|Gray-code the narrow-unit-equivalent pointers before crossing domains|

---

## 9. Summary / Decision Table

|Situation|Do you need pointer + address (or counter)?|Reason|
|---|---|---|
|Synchronous FIFO, power-of-2 DEPTH|Pointer + address|Extra bit for full/empty, lower bits for RAM|
|Asynchronous FIFO (CDC)|Pointer + address, plus Gray-coded copy|Binary for RAM/arithmetic, Gray for safe CDC|
|Non-power-of-2 DEPTH|Pointer with explicit wrap, or counter|Natural rollover no longer aligns with DEPTH|
|Need occupancy / almost-full flags|Dedicated counter|Simpler and DEPTH-agnostic vs pointer subtraction|
|Mixed-width (asymmetric) FIFO|Pointer converted to common unit + address|Address = narrow RAM index; pointer = shared comparison unit|
|FIFO without RAM (pure registers)|Usually just a counter|No memory address needed|