# log2

## Verilog

**Ceiling** (written \($\lceil x \rceil$\)) means **rounding up** to the nearest integer that is greater than or equal to \(x\).

## Simple examples

- $(\lceil 3.2 \rceil = 4)$
- $(\lceil 5.0 \rceil = 5)$ (already an integer → stays the same)
- $(\lceil 7.9 \rceil = 8)$
- $(\lceil -2.3 \rceil = -2)$ (rounds toward $(+\infty\))$

## Why it appears with log₂

In hardware design we almost always want the **smallest integer number of bits** that can represent \(N\) different values (or address \(N\) locations).

- Exact $(\log_2(17) \approx 4.087)$
- $(\lceil \log_2(17) \rceil = 5)$ → you need **5 bits**

Without the ceiling you would get a fractional result that cannot be used as a bit-width.

## Quick comparison with floor

| Operation | Symbol     | Behavior          | Example \(\log_2(17)\) |
|-----------|------------|-------------------|------------------------|
| Floor     | $(\lfloor x \rfloor)$ | Round **down**    | 4                      |
| Ceiling   | $(\lceil x \rceil)$   | Round **up**      | 5                      |

So when you see `$clog2()` in SystemVerilog or `ceil(log2(...))` in VHDL, the “c” / “ceil” part is exactly this rounding-up operation.

# Signal size examples

## Grok Model Example

### VHDL

```vhdl
use ieee.math_real.all; -- only for constant calculation

DATA_WIDTH : positive := 8;
DEPTH : positive := 16

wr_data : in std_logic_vector(DATA_WIDTH-1 downto 0);

constant ADDR_W : natural := integer(ceil(log2(real(DEPTH))));
constant CNT_W : natural := integer(ceil(log2(real(DEPTH+1))));

type mem_t is array (0 to DEPTH-1) of std_logic_vector(DATA_WIDTH-1 downto 0); signal mem : mem_t;

signal wr_ptr, rd_ptr : unsigned(ADDR_W downto 0); -- +1 bit

signal count : unsigned(CNT_W-1 downto 0);

```

### Verilog

```verilog
parameter int DATA_WIDTH = 8;
parameter int DEPTH = 16;

input logic [DATA_WIDTH-1:0] wr_data,

// pointer width (DEPTH number of values, range 0 -> DEPTH-1)
localparam int ADDR_W = $clog2(DEPTH); 
// occupancy width (DEPTH +1 number of values, range 0 -> DEPTH)
localparam int CNT_W = $clog2(DEPTH+1); 

logic [DATA_WIDTH-1:0] mem [0:DEPTH-1];

// Pointers (with optional extra MSB for full/empty)
logic [ADDR_W:0] wr_ptr_tc, rd_ptr_tc; // +1 bit technique

// Pointers (with NO optional extra MSB for full/empty)
logic [ADDR_W-1:0] wr_ptr, rd_ptr;

// Occupancy counter
logic [CNT_W-1:0] count;
```

## BSC / pulp-platform

[pulp fifo example](https://github.com/pulp-platform/common_cells/blob/0989ff73d0315922791bf42137c0ce0cbb4a76ca/src/fifo_v3.sv)

### Verilog

```verilog
parameter int unsigned DATA_WIDTH   = 8;
parameter int unsigned DEPTH        = 16;
parameter int unsigned ADDR_DEPTH   = (DEPTH > 1) ? $clog2(DEPTH) : 1

input logic [DATA_WIDTH-1:0] wr_data,

logic [DATA_WIDTH-1:0] mem [DEPTH-1:0];

// Pointers (with optional extra MSB for full/empty)
logic [ADDR_DEPTH:0] wr_ptr_tc, rd_ptr_tc;

// Pointers (with NO optional extra MSB for full/empty)
logic [ADDR_DEPTH - 1:0] wr_ptr, rd_ptr;

// Occupancy counter
logic [ADDR_DEPTH:0] count;

```
There are 2 differences with the grok version:
- Depth = 1 guard
- For A depth in a power of 2, both designs are equivalent, but in case it is not, this version will have 1 extra bit

|DEPTH|`$clog2(DEPTH)`|`$clog2(DEPTH)+1`|`$clog2(DEPTH+1)`|Same?|Notes|
|---|---|---|---|---|---|
|1|0|1|1|Yes*|*but first code has special guard|
|2|1|2|2|Yes|power of 2|
|3|2|3|2|**No**||
|4|2|3|3|Yes|power of 2|
|5|3|4|3|**No**||
|6|3|4|3|**No**||
|7|3|4|3|**No**||
|8|3|4|4|Yes|power of 2|
|9|4|5|4|**No**||
|15|4|5|4|**No**||
|16|4|5|5|Yes|power of 2|
|17|5|6|5|**No**||
|31|5|6|5|**No**||
|32|5|6|6|Yes|power of 2|


## Counter

### VHDL
Current design

```vhdl
constant TIMER_CYCLES : real := 16.0;
constant TIMER_WIDTH : integer := integer(ceil(log2(TIMER_CYCLES)));
signal timer : unsigned(TIMER_WIDTH-1 downto 0) := (others => '0');
```

### Verilog
My verification option

```verilog
localparam TIMER_CYCLES = 16;
typedef logic [$clog2(TIMER_CYCLES)-1 : 0] timer_cnt_t;
timer_cnt_t timer_ref;
```

### Grok SVerilog proposal
```verilog
parameter int unsigned CYCLES = 16

localparam int WIDTH = (CYCLES > 1) ? $clog2(CYCLES) : 1;
logic [WIDTH-1:0] cnt;
```

For CYCLES ≥ 2 → all three give exactly the same width.

# Definitions
**VHDL vs Verilog/SystemVerilog – Main Definitions Comparison**

| Category                  | VHDL                                      | Verilog / SystemVerilog                          | Purpose / Notes |
|---------------------------|-------------------------------------------|--------------------------------------------------|-----------------|
| **Module / Entity**       | `entity` + `architecture`                 | `module` … `endmodule`                           | Top-level design unit |
| **Static (elaboration-time)** |                                           |                                                  | |
| Configurable constants    | `generic`                                 | `parameter`                                      | Can be overridden at instantiation |
| Internal constants        | `constant`                                | `localparam`                                     | Cannot be overridden |
| **Dynamic (runtime)**     |                                           |                                                  | |
| Interface                 | `port` (`in`/ `out` / `inout`??)          | `input` / `output` / `inout`                     | Module interface |
| Internal signals          | `signal`                                  | `wire` / `reg` / `logic`                         | Connects processes / always blocks |
| Variables (process-local) | `variable`                                | Automatic variables inside `always` / functions  | Immediate update (no delta delay) |
| **Types**                 |                                           |                                                  | |
| Type definition           | `type` / `subtype`                        | `typedef` (SystemVerilog)                        | User-defined types |
| Common data types         | `std_logic`, `std_logic_vector`, `unsigned`, `signed`, `integer`… | `logic`, `bit`, `reg`, `wire`, `int`, `integer`… | |
| **Structural**            |                                           |                                                  | |
| Component declaration     | `component`                               | Not needed (direct instantiation)                | VHDL requires it for older styles |
| Instantiation             | `label: entity work.xxx generic map (…) port map (…)` | `xxx #(.PARAM(val)) inst (.port(sig));` | Connecting sub-modules |
| **Other common**          |                                           |                                                  | |
| Packages / libraries      | `package` + `library` / `use`             | `` `include `` / packages (SV)                   | Sharing declarations |
| Functions / Procedures    | `function` / `procedure`                  | `function` / `task`                              | Reusable sequential code |
| Generate                  | `generate` / `for` / `if`                 | `generate` / `for` / `if`                        | Conditional/repetitive hardware |

# Data types
**VHDL vs Verilog/SystemVerilog – Data Type Comparison**

| Category                    | VHDL                                      | Verilog / SystemVerilog                          | Notes / Typical Use |
|-----------------------------|-------------------------------------------|--------------------------------------------------|---------------------|
| **1-bit logic**             | `std_logic`                               | `logic` / `reg` / `wire` / `bit`                 | Most common single-bit type |
| **Multi-bit vector**        | `std_logic_vector(N-1 downto 0)`          | `logic [N-1:0]`                                  | Preferred modern style. In VHDL is used when arithmetic is not needed |
| **Unsigned number**         | `unsigned(N-1 downto 0)`                  | `logic [N-1:0]` (or `unsigned` in SV)            | Arithmetic on bit vectors. In VHDL is natural numeric meaning |
| **Signed number**           | `signed(N-1 downto 0)`                    | `logic signed [N-1:0]`                           | Signed arithmetic |
| **Integer**                 | `integer` / `natural` / `positive`        | `integer` / `int` / `int unsigned`               | Simulation & generics/parameters |
| **Boolean**                 | `boolean`                                 | `bit` or `logic` (0/1)                           | VHDL has true Boolean |
| **Real / floating**         | `real`                                    | `real` / `shortreal`                             | Simulation only (not synthesizable) |
| **Time**                    | `time`                                    | `time`                                           | Delays & simulation |
| **Enumerated**              | `type state_t is (IDLE, RUN, DONE);`      | `typedef enum {IDLE, RUN, DONE} state_t;`        | FSMs |
| **Array (unconstrained)**   | `type mem_t is array (natural range <>) of …` | `logic [W-1:0] mem [];` (SV)                  | Flexible arrays |
| **Array (constrained)**     | `type mem_t is array (0 to DEPTH-1) of …` | `logic [W-1:0] mem [0:DEPTH-1];`                 | Memories, register files |
| **Record / Struct**         | `type rec_t is record … end record;`      | `typedef struct {…} rec_t;`                      | Grouping signals |
| **Resolved vs unresolved**  | `std_logic` (resolved) vs `std_ulogic`    | `logic` / `wire` (multiple drivers allowed with care) | VHDL is stricter |
| **4-state vs 2-state**      | Almost everything is 4-state (`std_logic`) | `logic`/`reg`/`wire` = 4-state<br>`bit`/`bit vector` = 2-state | SV offers both |
| **User-defined subtype**    | `subtype byte is std_logic_vector(7 downto 0);` | `typedef logic [7:0] byte_t;`                  | Convenience aliases |

```vhdl
type integer  is range -2147483648 to +2147483647;  -- typical
subtype natural  is integer range 0 to integer'high;
subtype positive is integer range 1 to integer'high;
```

## Examples
```vhdl
RAM_DEPTH : integer range 2 to 2**20;

wr_count : out std_logic_vector (integer(ceil(log2(real(RAM_DEPTH))))-1 downto 0);
```
|Part of the expression|Type / Function|What it does|Why it is used|
|---|---|---|---|
|`RAM_DEPTH`|Usually `positive` or `natural` (generic)|The depth of the memory/FIFO|Input parameter|
|`real(RAM_DEPTH)`|Conversion to `real`|Converts integer → floating-point|`log2` only accepts `real`|
|`log2(...)`|`real` → `real`|Computes log⁡2(RAM_DEPTH) \log_2(\text{RAM\_DEPTH}) log2​(RAM_DEPTH)|From `ieee.math_real`|
|`ceil(...)`|`real` → `real`|Rounds **up** to the next integer|We need enough bits (ceiling)|
|`integer(...)`|`real` → `integer`|Converts the real result back to integer|Array bounds must be integer|
|`... - 1 downto 0`|Integer expression|Creates the range `(WIDTH-1 downto 0)`|Classic vector range|
|`std_logic_vector(...)`|Vector type|The actual type of the port|Most common type for multi-bit ports. Standard, synthesizable, multi-bit port type. Compatible with almost everything.|

# VHDL vs Verilog/SystemVerilog – Concurrent vs Sequential

| Aspect                        | VHDL                                      | Verilog / SystemVerilog                          | Notes |
|-------------------------------|-------------------------------------------|--------------------------------------------------|-------|
| **Concurrent code**           | Outside any process                       | Outside any `always` / `initial`                 | Executes continuously / in parallel |
| **Sequential code**           | Inside a `process`                        | Inside `always`, `always_ff`, `always_comb`, `initial` | Executes in order, one statement after another |
| **How hardware is described** | Both concurrent and sequential styles are common | Same                                             | Final hardware is always parallel |

## Main concurrent constructs

| Construct                     | VHDL                                      | Verilog / SystemVerilog                          | Typical use |
|-------------------------------|-------------------------------------------|--------------------------------------------------|-------------|
| Continuous assignment         | `signal <= expression;`                   | `assign signal = expression;`                    | Combinational logic |
| Conditional assignment        | `signal <= A when cond else B;`           | `assign signal = cond ? A : B;`                  | Mux / simple combo |
| Selected assignment           | `with sel select signal <= …;`            | `assign signal = …` with `case` inside `always_comb` | Mux |
| Component / module instantiation | `inst: entity work.mod port map (…);`  | `mod inst (.port(sig));`                         | Hierarchy |
| Generate                      | `generate` / `for` / `if`                 | `generate` / `for` / `if`                        | Repetitive / conditional hardware |

## Main sequential constructs

| Construct                     | VHDL                                      | Verilog / SystemVerilog                          | Typical use |
|-------------------------------|-------------------------------------------|--------------------------------------------------|-------------|
| Process / Always block        | `process (sensitivity_list)`              | `always @(…)` / `always_ff` / `always_comb` / `always_latch` | Sequential or combo logic |
| Clocked process               | `if rising_edge(clk) then …`              | `always_ff @(posedge clk) …`                     | Flip-flops, registers |
| Combinational process         | `process (all)` or full sensitivity list  | `always_comb`                                    | Pure combinational logic |
| Sequential statements         | `if`, `case`, `loop`, `wait`              | `if`, `case`, `for`, `while`, `fork…join`        | Control flow inside process/always |
| Variable assignment           | `variable := value;` (immediate)          | Automatic variables (immediate)                  | Temporary storage |
| Signal assignment (inside process) | `signal <= value;` (scheduled)       | `signal <= value;` or `signal = value;`          | Non-blocking / blocking |

## Key behavioural differences

| Topic                         | VHDL                                      | Verilog / SystemVerilog                          |
|-------------------------------|-------------------------------------------|--------------------------------------------------|
| Concurrent assignments        | Always concurrent                         | `assign` is concurrent                           |
| Sequential assignments        | Only inside `process`                     | Only inside `always` / `initial`                 |
| Signal update                 | After a delta cycle (projected)           | Non-blocking `<=` → after timestep<br>Blocking `=` → immediate |
| Sensitivity list              | Must be complete (or use `process(all)`)  | `always_comb` / `always_ff` handle it automatically (recommended) |
| Latch inference               | Easy if incomplete assignment             | Same risk with incomplete `always_comb`          |
| Variable vs Signal            | `variable` = immediate, `signal` = delayed | Blocking (`=`) ≈ variable<br>Non-blocking (`<=`) ≈ signal |

## Classic templates

**Clocked register (sequential)**

```vhdl
-- VHDL
process (clk)
begin
  if rising_edge(clk) then
    q <= d;
  end if;
end process;
```

```systemverilog
// SystemVerilog
always_ff @(posedge clk)
  q <= d;
```

**Combinational logic (concurrent style)**

```vhdl
-- VHDL
y <= a and b;
```

```systemverilog
// SystemVerilog
assign y = a & b;
```

**Combinational logic (sequential style)**

```vhdl
-- VHDL
process (all)
begin
  y <= a and b;
end process;
```

```systemverilog
// SystemVerilog
always_comb
  y = a & b;
```

## Summary table – When to use what

| Goal                              | Preferred style          | VHDL example                  | SystemVerilog example              |
|-----------------------------------|--------------------------|-------------------------------|------------------------------------|
| Simple combinational logic        | Concurrent               | `y <= a and b;`               | `assign y = a & b;`                |
| Complex combinational logic       | Sequential (`process` / `always_comb`) | `process(all) …`     | `always_comb …`                    |
| Flip-flops / registers            | Sequential clocked       | `if rising_edge(clk)`         | `always_ff @(posedge clk)`         |
| Finite State Machine              | Sequential clocked       | process with `case`           | `always_ff` + `always_comb`        |
| Hierarchy / structure             | Concurrent               | component instantiation       | module instantiation               |

**Golden rule**  
- **Concurrent** = hardware that is always “alive” (wires, continuous assignments, instances).  
- **Sequential** = ordered descriptions that the tool turns into flip-flops or combinational logic depending on how you write the process/`always` block.