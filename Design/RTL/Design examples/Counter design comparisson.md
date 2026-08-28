# Design comparison
- Design notes: Always, when adding a counter, have clear when it must start and when it must end, and when it restarts after end.

```verilog
localparam int unsigned TIMER_WIDTH = $clog2(LINK_TIMER_CYCLES)
typedef logic [TIMER_WIDTH-1 : 0] link_timer_cnt_t;

always_ff @(posedge clk) begin
    if (an_rst || power_on) begin
      timer_sync_cnt_ref <= timer_cnt_t'(0);
      link_timer_sync_done_ref <= 1'b0;
    end
    else begin
      if (sync_status)
        timer_sync_cnt_ref <= timer_cnt_t'(0);
        link_timer_sync_done_ref <= 1'b0;
      else if (timer_sync_cnt_ref < (TIMER_CYCLES-1))
        timer_sync_cnt_ref <= timer_sync_cnt_ref + timer_cnt_t'(1);
      else
        link_timer_sync_done_ref <= 1'b1;
    end
  end
```


```verilog
localparam int unsigned TIMER_WIDTH = $clog2(LINK_TIMER_CYCLES)
typedef logic [TIMER_WIDTH-1 : 0] link_timer_cnt_t;

always_ff @(posedge clk) begin
    if (an_rst || power_on) begin
      timer_sync_cnt_ref <= timer_cnt_t'(0);
    end
    else begin
      if (sync_status)
        timer_sync_cnt_ref <= timer_cnt_t'(0);
      else if (timer_sync_cnt_ref < (TIMER_CYCLES-1))
        timer_sync_cnt_ref <= timer_sync_cnt_ref + timer_cnt_t'(1);
    end
  end

  assign link_timer_sync_done_ref = timer_sync_cnt_ref >= (TIMER_CYCLES-1);
```

## Main reasons Design 1 is preferred

Design 1 generates the `done` flag **inside the same clocked process** as the counter, making it a flip-flop.

1. **Cleaner timing**  
   There is no combinational path from the counter to the `done` signal.  
   This avoids extra logic levels and makes timing closure easier.

2. **No glitches**  
   A combinational `assign done = …` can produce short glitches.  
   A registered version is stable and changes only on the clock edge.

3. **Standard RTL style**  
   Status flags (`done`, `full`, `empty`, `valid`…) are almost always registered.  
   This is the style used in the vast majority of industrial designs. Notice, it doesn't mean that the status flags must be registered, it means that these flags may be combinatorial, but with a simple logic, from other registered flags, so that the timing is short.

4. **Easier to understand and maintain**  
   Everything related to the timer lives in one sequential block.

## When Design 2 could still be acceptable
Only when you **explicitly need** the `done` flag in the exact same cycle the counter reaches its final value (rare).

## Grok alternative design

```verilog
parameter int unsigned LINK_TIMER_CYCLES = 1000,
localparam int unsigned TIMER_WIDTH = $clog2(LINK_TIMER_CYCLES + 1)

typedef logic [TIMER_WIDTH-1:0] timer_cnt_t;

timer_cnt_t timer_cnt;

always_ff @(posedge clk or negedge rst_n) begin
	if (!rst_n) begin
		timer_cnt      <= '0;
		an_sync_status <= 1'b1; // OK
	end
	else if (sync_status) begin
		// sync_status == OK
		timer_cnt      <= '0;
		an_sync_status <= 1'b1;

	end
	else begin
		// sync_status == FAIL
		if (timer_cnt < LINK_TIMER_CYCLES) begin
			timer_cnt      <= timer_cnt + 1'b1;
			an_sync_status <= 1'b1;
		end
		else begin
			// Timer expired
			an_sync_status <= 1'b0; // FAIL
		end
	end
end
```

Take care with the values, see the following options:

So I am seeing that it may be an easy solution to define width parameters and define the signal using that width: $[width-1 : 0]$. **Notice** this rule is not always followed.

## Conclusions

LINK_TIMER = 3 in this example

|Option|Condition|Counter width|Counter values|Expire condition|Cycles counted|
|---|---|---|---|---|---|
|**Slow**|`timer_cnt < LINK_TIMER_CYCLES`|`$clog2(LINK_TIMER_CYCLES + 1)`|`0 ... LINK_TIMER_CYCLES` (`0..3`)|`timer_cnt == LINK_TIMER_CYCLES`|3 cycles|
|**Medium**|`timer_cnt < LINK_TIMER_CYCLES - 1`|`$clog2(LINK_TIMER_CYCLES)`|`0 ... LINK_TIMER_CYCLES-1` (`0..2`)|`timer_cnt == LINK_TIMER_CYCLES-1`|2 counts + expire|
|**Medium Start=1**|`timer_cnt < LINK_TIMER_CYCLES`|`$clog2(LINK_TIMER_CYCLES + 1)`|`1 ... LINK_TIMER_CYCLES` (`1..3`)|`timer_cnt == LINK_TIMER_CYCLES`|3 counts (start counts as 1)|
|**Fast**|`timer_cnt < LINK_TIMER_CYCLES - 2`|`$clog2(LINK_TIMER_CYCLES - 1)`|`0 ... LINK_TIMER_CYCLES-2` (`0..1`)|`timer_cnt == LINK_TIMER_CYCLES-2`|1 count + expire|
|**Fast Start=1**|`timer_cnt < LINK_TIMER_CYCLES - 1`|`$clog2(LINK_TIMER_CYCLES)`|`1 ... LINK_TIMER_CYCLES-1` (`1..2`)|`timer_cnt == LINK_TIMER_CYCLES-1`|2 counts (start counts as 1)|

## Waveforms

**Slow**

```json
{
  "signal": [
    { "name": "start_cnt",  "wave": "0..10...." },
    { "name": "state",      "wave": "3..4.....", "data": ["IDLE", "COUNT"] },
    { "name": "cnt",        "wave": "=...===..", "data": ["0", "1", "2", "3"] },
    { "name": "cnt_expire", "wave": "0......1." }
  ]
}
```
![[slow_wave_counter_design_comparisson.png]]

**Medium**

```json
{
  "signal": [
    { "name": "start_cnt",  "wave": "0..10...." },
    { "name": "state",      "wave": "3..4.....", "data": ["IDLE", "COUNT"] },
    { "name": "cnt",        "wave": "=...==...", "data": ["0", "1", "2"] },
    { "name": "cnt_expire", "wave": "0.....1.." }
  ]
}
```
![[medium_wave_counter_design_comparisson.png]]

**Medium Start=1**

```json
{
  "signal": [
    { "name": "start_cnt",  "wave": "0..10...." },
    { "name": "state",      "wave": "3..4.....", "data": ["IDLE", "COUNT"] },
    { "name": "cnt",        "wave": "=..===...", "data": ["0", "1", "2", "3"] },
    { "name": "cnt_expire", "wave": "0.....1.." }
  ]
}
```
![[medium_1_wave_counter_design_comparisson.png]]

**Fast**

```json
{
  "signal": [
    { "name": "start_cnt",  "wave": "0..10...." },
    { "name": "state",      "wave": "3..4.....", "data": ["IDLE", "COUNT"] },
    { "name": "cnt",        "wave": "=...=....", "data": ["0", "1"] },
    { "name": "cnt_expire", "wave": "0....1..." }
  ]
}
```
![[fast_wave_counter_design_comparisson.png]]

**Fast Start=1**

```json
{
  "signal": [
    { "name": "start_cnt",  "wave": "0..10...." },
    { "name": "state",      "wave": "3..4.....", "data": ["IDLE", "COUNT"] },
    { "name": "cnt",        "wave": "=..==....", "data": ["0", "1", "2"] },
    { "name": "cnt_expire", "wave": "0....1..." }
  ]
}
```
![[fast_1_wave_counter_design_comparisson.png]]