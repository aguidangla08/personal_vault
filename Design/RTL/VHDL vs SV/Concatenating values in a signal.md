# Bit Order in Concatenation — SystemVerilog vs VHDL

## SystemVerilog

Concatenation uses `{ }`, and the **leftmost operand becomes the most significant bits (MSB)** of the result.

```systemverilog
logic [3:0] a = 4'b1010; // MSB..LSB
logic [3:0] b = 4'b0101;

logic [7:0] result;
assign result = {a, b};
// result = 1010_0101
// result[7:4] = a (MSB side)
// result[3:0] = b (LSB side)
```

**Rule:** `{first, second, third, ...}` → `first` occupies the highest bit positions, `last` occupies the lowest.

```systemverilog
logic [7:0] byte_val;
assign byte_val = {msb_nibble, lsb_nibble}; // msb_nibble -> [7:4], lsb_nibble -> [3:0]
```

Single bits work the same way:

```systemverilog
logic [3:0] w = {1'b1, 1'b0, 1'b1, 1'b1}; // w = 4'b1011
// w[3]=1, w[2]=0, w[1]=1, w[0]=1
```

---

## VHDL

Concatenation uses `&`, and ordering **follows the declared range direction of the target signal** — it is _not_ fixed like SystemVerilog. The leftmost operand goes to the **leftmost index** of the result, whatever that index means (MSB or LSB depends on how the target is declared).

### Case: `downto` (typical MSB-first declaration)

```vhdl
signal a : std_logic_vector(3 downto 0) := "1010";
signal b : std_logic_vector(3 downto 0) := "0101";
signal result : std_logic_vector(7 downto 0);

result <= a & b;
-- result = "10100101"
-- result(7 downto 4) = a
-- result(3 downto 0) = b
```

This matches SystemVerilog's behavior _when the vector is declared `downto` (descending, MSB-first)_ — which is the common convention.

### Case: `to` (ascending declaration) — order flips!

```vhdl
signal a : std_logic_vector(0 to 3) := "1010";
signal b : std_logic_vector(0 to 3) := "0101";
signal result : std_logic_vector(0 to 7);

result <= a & b;
-- result = "10100101"
-- result(0 to 3) = a  <-- now a is at the LOW indices, not necessarily MSB
```

**Key VHDL rule:** the leftmost concatenation operand always maps to the leftmost (first-declared) index of the result — but whether "leftmost index" means MSB or LSB depends entirely on whether the vector range uses `downto` or `to`.

---

## Summary Table

|Language|Syntax|Leftmost operand goes to...|
|---|---|---|
|SystemVerilog|`{a, b}`|Always the MSB side|
|VHDL (`downto` vector)|`a & b`|MSB side (matches SV)|
|VHDL (`to` vector)|`a & b`|LSB side (index 0) — **opposite of MSB**|

## Practical Takeaway

- **SystemVerilog:** concatenation order is unambiguous — first = MSB, always.
- **VHDL:** always check how your target signal's range is declared (`downto` vs `to`) before trusting the bit order of a concatenation. Mixing `downto`-declared and `to`-declared vectors in the same concatenation is a common source of bugs.