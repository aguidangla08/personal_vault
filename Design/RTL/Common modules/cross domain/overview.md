# RTL Clock Domain Crossing (CDC) — Design Notes

## 1. Signal Class → CDC Technique

|Signal type|Technique|
|---|---|
|Single-bit|2-stage (or 3-stage) synchronizer|
|Multi-bit control/data|Handshake (req/ack) or Gray-coded pointer|
|Continuous stream|Asynchronous FIFO|

## 2. Async FIFO (Dual Clock, VHDL Target)

Write and read sides run on independent, unrelated clocks. Core challenges:

**Metastability** — Sampling a signal (or pointer) generated in another clock domain can drive a flip-flop into a metastable state. Every signal crossing domains must pass through a synchronizer before use in logic.

**Pointer synchronization** — Write and read pointers must be compared across domains to generate full/empty. Comparing raw binary pointers is unsafe because multiple bits can change at once, so a synchronizer can capture a non-adjacent (invalid) value. Standard flow:

1. Convert the local binary pointer to Gray code (only one bit changes per increment).
2. Send the Gray-coded pointer to the other clock domain.
3. Pass it through a 2-stage (or 3-stage, for higher MTBF) flip-flop synchronizer.
4. Use the synchronized Gray pointer, converted back if needed, to generate flags.

**Full/Empty flags** — Must be derived only from the _synchronized_ (safe) version of the opposite domain's pointer, never the raw pointer. Full = write pointer catches up to synchronized read pointer (Gray compare with wrap bit); Empty = read pointer equals synchronized write pointer.

## 3. Clock Drift

Even with correct synchronization, sustained differences between write and read clock rates (frequency drift, jitter, bursty traffic) can push the FIFO toward overflow or underflow over time if not accounted for in sizing/flow control.

### Mitigations for significant/bursty drift

- **Almost-full / almost-empty flags** — Assert before the FIFO is physically full/empty, giving upstream/downstream logic time to throttle.
- **Elastic FIFO monitoring** — Track average fill level over time to detect sustained (not just instantaneous) drift, and dynamically adjust upstream/downstream flow control (same principle used in networking elastic buffers).
- **Depth sizing** — Depth must absorb the worst-case accumulated drift over a burst:

```
FIFO depth ≥ |write_clock_rate − read_clock_rate| × max_burst_duration
```

## 4. Summary Checklist

- [ ] Every cross-domain single bit goes through a synchronizer.
- [ ] Multi-bit values use handshake or Gray-coded pointers, never raw binary sync.
- [ ] Flags computed only from synchronized pointers.
- [ ] Depth sized against worst-case drift × burst duration.
- [ ] Almost-full/empty + elastic monitoring added if drift is large or bursty.