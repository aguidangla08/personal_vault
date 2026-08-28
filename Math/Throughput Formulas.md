# RTL Design: Frequency, Period, Word Size & Throughput Formulas

Reference formulas for converting between clock frequency, period, data word width, and interface throughput — useful when re-sizing a bus, changing a clock domain, or matching bandwidth across two RTL interfaces (e.g. AXI, AHB, memory controllers, SerDes, FIFOs).

---

## 1. Frequency ↔ Period

$$T = \frac{1}{f} \qquad f = \frac{1}{T}$$

|Symbol|Meaning|Units|
|---|---|---|
|f|Clock frequency|Hz|
|T|Clock period|seconds|

**Example:** f = 200 MHz → T = 1 / (200×10⁶) = 5 ns

In RTL, this is what you use to derive a `create_clock -period` value for SDC constraints from a spec'd MHz number, or vice versa.

---

## 2. Throughput (Data Rate) from Frequency and Word Size

For an interface that transfers one word (bus width) per clock cycle:

$$\text{Throughput} = f \times W$$

If a transfer takes **N cycles per word** (e.g. DDR needs 0.5 cycles/word, a multi-cycle path needs N > 1):

$$\text{Throughput} = \frac{f}{N} \times W$$

|Symbol|Meaning|Units|
|---|---|---|
|f|Clock frequency|Hz (transfers/sec)|
|W|Word size / bus width|bits (or bytes)|
|N|Cycles required per word transfer|cycles|
|Throughput|Data rate|bits/sec (or bytes/sec, divide by 8)|

### Converting Hz to seconds (unit walkthrough)

Hz is defined as **1/second** (cycles per second), so converting a frequency to a time value is just the reciprocal — same relationship as Section 1, applied step-by-step with unit prefixes:

$$T \text{ (seconds)} = \frac{1}{f \text{ (Hz)}}$$

**Step-by-step example:** f = 500 MHz

1. Convert MHz → Hz: 500 MHz = 500 × 10⁶ Hz = 500,000,000 Hz
2. Take the reciprocal to get seconds per cycle: T = 1 / 500,000,000 Hz = 2 × 10⁻⁹ s = **2 ns**
3. Sanity check by going back: f = 1 / T = 1 / (2×10⁻⁹) = 500×10⁶ Hz = 500 MHz ✓

**Common Hz-to-time prefixes at a glance:**

|Frequency|In Hz|Period|
|---|---|---|
|1 kHz|10³ Hz|1 ms|
|1 MHz|10⁶ Hz|1 μs|
|1 GHz|10⁹ Hz|1 ns|
|100 MHz|100×10⁶ Hz|10 ns|
|500 MHz|500×10⁶ Hz|2 ns|
|1.5 GHz|1.5×10⁹ Hz|0.667 ns|

This T value (in seconds) is what feeds directly into the throughput formula below when you need cycle time rather than frequency — e.g. computing how many nanoseconds are available per word transfer.

**Example — AXI-like bus:** f = 500 MHz, W = 128 bits, N = 1 Throughput = 500×10⁶ × 128 = 64×10⁹ bits/s = 8 GB/s

**Example — DDR (dual data rate, 2 transfers/cycle):** Effective throughput = f × 2 × W (treat DDR as N = 0.5)

---

## 3. Converting Word Size While Preserving Throughput

This is the common RTL re-sizing problem: you're changing the bus/word width (e.g., 64-bit → 32-bit datapath) and need the new clock frequency that preserves the same total bandwidth.

Since throughput must stay constant:

$$f_1 \times W_1 = f_2 \times W_2$$

Solve for the new frequency:

$$f_2 = f_1 \times \frac{W_1}{W_2}$$

Solve for the new word size:

$$W_2 = W_1 \times \frac{f_1}{f_2}$$

**Example:** Original design: f₁ = 100 MHz, W₁ = 64 bits. You narrow the datapath to W₂ = 32 bits to save area. Required new frequency to keep the same throughput:

f₂ = 100 MHz × (64 / 32) = **200 MHz**

This is exactly the trade-off seen when converting a wide, slow internal datapath to a narrower, faster one feeding a serializer, or when matching an AXI master's data width to a narrower slave.

---

## 4. Converting Period When Word Size Changes (equivalent form)

Combining sections 1 and 3, in terms of period directly:

$$T_2 = T_1 \times \frac{W_2}{W_1}$$

**Example:** T₁ = 10 ns (100 MHz) at W₁ = 64 bits. Narrowing to W₂ = 32 bits:

T₂ = 10 ns × (32/64) = 5 ns → f₂ = 200 MHz (matches Section 3 result)

---

## 5. Quick Reference Table

|Given|Find|Formula|
|---|---|---|
|f|T|T = 1/f|
|T|f|f = 1/T|
|f, W|Throughput|f × W|
|f, W, N cycles/word|Throughput|(f/N) × W|
|f₁, W₁, W₂|f₂ (same throughput)|f₂ = f₁ × (W₁/W₂)|
|f₁, W₁, f₂|W₂ (same throughput)|W₂ = W₁ × (f₁/f₂)|
|T₁, W₁, W₂|T₂ (same throughput)|T₂ = T₁ × (W₂/W₁)|

---

_Notes: Throughput here is theoretical peak bandwidth (100% bus utilization, no stalls/bubbles). Real RTL throughput should be de-rated by protocol overhead, handshake stalls, and burst efficiency where applicable._