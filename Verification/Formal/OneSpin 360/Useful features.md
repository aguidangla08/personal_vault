# Export waveform to VCD
Output files are a .vcd (containing the signals data) and a .dino (containing the visualization session state (groups, cursors, ...)).

Regarding the visualization tools, VScode [Vaporview](https://marketplace.visualstudio.com/items?itemName=lramseyer.vaporview) extension is the prefered waveform visualitzation tool to read the .vcd.
Property Verdi can be used to read the .dino file but for now I haven't seen an open-source alternative. 

## Steps

Execute the following preconditions
```bash
# Enter design verification mode
set_mode mv
```

### Failed property case
After that, follow steps seen in Section 7.4.5 Debugging Using External Tools of User Manual 
> mv> debug -tool vcd <property_name> - dir .

### Hold property case
> mv> show_witness -tool vcd <property_name> -dir .

### Arguments
- **dir** path Set preferred path where counterexamples are written to
default value: {\$TMPDIR/$USER/onespin_counter_example}
- **tool** internal|internal_nowave|wave|vcd|verdi|verdi_internal|modelsim Set preferred debug-
ging tool
default value: {internal}
For now vcd seems to be the best one, waveform doesn't generate anything

## Example

### Failed property case
> mv> debug -dir . -tool vcd {sva/uut/clue_eth_sgmii_cg_tx_check_ins/clue_eth_sgmii_cg_tx_check_assert_6}

### Hold property case
> mv> show_witness -dir . -tool vcd {sva/uut/clue_eth_sgmii_cg_tx_check_ins/clue_eth_sgmii_cg_tx_check_assert_6}

# TODO Import vhdl packages inside SV assertion files


# TODO Add delay if out of window
property Ord_set_tx_009;

@(posedge tx_clk)

disable iff (tx_rst || power_on)

##1 st_spec_q == ord_tx_data_st && $past(tx_er)

|-> tx_ord_set == clue_eth_pkg::prop_err;

endproperty

assert property (Ord_set_tx_009);

# Resources
OneSpin User Manual: OneSpin 360 Version 2022.2_2

