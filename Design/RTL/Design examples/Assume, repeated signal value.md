```verilog
// 5-cycle counter (0..4), wraps around
  logic [2:0] period_cnt;

  always_ff @(posedge tx_clk) begin
      if (tx_rst || power_on)
        period_cnt <= 3'd0;
      else
        period_cnt <= (period_cnt == 3'd4) ? 3'd0 : period_cnt + 3'd1;
  end

  // 1) Hold the value stable for 5 cycles
  hold_txd_mac_value_stable: assume property (
    disable iff (tx_rst || power_on)
    @(posedge tx_clk)
    (period_cnt != 3'd0) |-> $stable(txd_mac)
  );

  hold_tx_er_mac_value_stable: assume property (
    disable iff (tx_rst || power_on)
    @(posedge tx_clk)
    (period_cnt != 3'd0) |-> $stable(tx_er_mac)
  );

  hold_tx_en_mac_value_stable: assume property (
    disable iff (tx_rst || power_on)
    @(posedge tx_clk)
    (period_cnt != 3'd0) |-> $stable(tx_en_mac)
  );

  // 2) Force a *different* value at the boundary (every 5th cycle)
  //hold_txd_value_update: assume property (
  //  disable iff (tx_rst || power_on)
  //  @(posedge tx_clk)
  //  (period_cnt == 3'd0 && $past(tx_rst)) |-> (txd_mac != $past(txd_mac))
  //);
  ```