My original version
```
  if (X_MODE == 0) begin: x_mode
  001_test_x_mode_set: assert property (
    @(posedge clk)
    disable iff (rst)
    state_ref  == prev_st
    ##1
    state_ref  == target_st
    |-> a == $past(b)
  );

  001_test_signal_x_mode_state_keep: assert property (
    @(posedge clk)
    disable iff (rst)
    state_ref  == target_st
    ##1
    state_ref  == target_st
    |-> a == $past(a)
  );

  001_test_signal_keep: assert property (
    @(posedge clk)
    disable iff (rst)
    state_ref  != target_st && state_ref  != reset_st
    |-> a == $past(a)
  );
end // x_mode
```

Claude version **only valid for rose and fell, not value setting**
```
001_test_pulse_on_state_entry: assert property (
  @(posedge clk) disable iff (rst)
  $rose(a) |-> (state_ref == target_st)
);

001_test_pulse_width_one: assert property (
  @(posedge clk) disable iff (rst)
  $rose(a) |=> $fell(a)
);
```