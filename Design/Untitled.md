 
h1. Summary 

This issue is used to trace some doubts regarding my efforts in sgmii tx code-group module verification, the scope issue is tricky to explain but the impact may be very small.

The output of the Figure 36–6—PCS transmit code-group state diagram states takes 1 cycle to be driven.
 In contrast, sgmii tx ordered set module drives the outputs in the same cycle as the state is driven.

This behavior has 3 clear impacts for sgmii tx cg module:
 # The inverted use of the tx_even signal, which is explored in the following issue.
 # TX ouput has 1 clk delay
 # sgmii TX code-group verification code deviations from what was expected
 This issue is focused on points 2 and 3.

—
h1. Environment
 - *Component:* SGMII
 - *Module:* sgmii_eth_sgmii_cg_tx
 - *Simulator / Tool:* OneSpin360 2022.2_2
 - *version:*
 - *OS / Server:* Ubuntu 24.04.4 LTS (OneSpin docker image)
 - *Testbench / Verification IP:* formal sgmii test
 - *Regression / Test name:* clue_eth_sgmii_cg_tx_check_assert_6
 - *Seed (if applicable):* No

—
h1. Description

This is a tricky issue. For that reason, this description explains the case step by step.
h2. Expected Behavior
 # It is expected that the output is driven the same cycle the state is driven, this can be seen in ID-1.
 # Clear verification code

<!-- What the RTL/design is supposed to do per spec/microarchitecture doc -->
h2. Actual Behavior
 # The output is driven 1 cycle after the state drive
 # The verification code has the following caracteristics:

 # 
 ## First, lets introduce an extra *new* but related idea, as it can be seen in  Figure 36–6, for the state SPECIAL_GO tx_o_set is set for example to /S/, the same cycle  tx_code-group <= tx_o_set, my *guess* is that this behavior is not implemented because the implementation would require the FlipFlop/Register entry of tx_ord_set, which refers to its value 1 cycle before. Using that value, the behavior may be possible at least from the logic point of view, maybe not frequency achievable, etc ....
 ## Given this idea, it is relevant to say that the verification code uses the tx_ord_set value and not its past value, which cause the verification reference model states to be computed 1 cycle before the design one.
 ### This is not a problem, which mean that all *assert* property checkings *hold* because of the following modifications/patch:
 #### tx_even is used in an inverted way as mentioned before
 #### tx_data signal drive checkings require to obtain the value 2 cycles before, to compensate for difference cycle with the design state machine
 #### 
 tx_ord_set_en signal drive checkings require to obtain the value 1 cycle before, to compensate for difference cycle with the design state machine, notice that this signal uses a diferent timing, as mentioned in [documentation|https://rtl-cores.gitlab-pages.clue.aero/ethernet-core/master/module/classclue__eth__sgmii__cg__tx.html].

<!-- What actually happened -->
h2. Spec / Documentation Reference

<!-- Link or section reference to microarchitecture spec, IP spec, protocol standard, etc. -->
 - Doxygen clue_eth_sgmii_cg_tx_pages
 - Code location: ethernet-core repository, commit: [https://gitlab-1.clue.aero/rtl-cores/ethernet-core/-/tree/9512e121abe2e9b2daa190e740e6672756cfa0a5], clue_eth_sgmii_cg_tx/src/clue_eth_sgmii_cg_tx.vhd file.
 - Related spec/section: Figure 36–6 PCS Transmit code-group state diagram and 36.2.4.2 Transmission order section of the IEEE 802.3-2018.

- 

—
h2. Steps to Reproduce
h3. Before running OneSpin
{code:bash}
 # 1. Set up the right OneSpin version
 # (OS version and other dependencies may be required to obtain the same result)
 
 # 2. Clone ethernet-core repository on the scope commit and download submodules
 git clone --depth 1 --branch feature/ethernet_top_tb_cocotb_regression -single-branch git@gitlab-1.clue.aero:rtl-cores/ethernet-core.git
 cd ethernet-core
 git checkout 9512e121abe2e9b2daa190e740e6672756cfa0a5
 git submodule update --init --recursive
 
 # 3. cd to the scripts folder
 cd clue_eth_sgmii_top/ver/formal/onespin/scripts
 
 # 4. Run OneSpin
 onespin
{code}
Manual step, enter formal.tcl inside your current directory and **comment** the **check** command line, which may be the last line.
h4. Inside OneSpin
{code:java}
 # 1. Run OneSpin testbench
 source formal.tcl
{code}
Now, manually or using tcl, run any of the scope test case assertions showed Id is **clue_eth_sgmii_cg_tx_check_assert_6**.
 See the property source for more information.

-->

—
h1. Signal / Waveform Evidence
|Signal Name|Expected Value|Actual Value|Time / Cycle|
|state_ref|idle_disparity_ok_st|generate_code_groups_st|6|
|sgmii_cg_tx_st|check_ord_set_st|check_ord_set_st|6|
|state_ref|idle_i2b_st|idle_disparity_ok_st|6|
|tx_data|bc|0|6|
|tx_even|1|1|6|
|tx_data|50|bc|8|
|tx_ord_set_en|1|1|8|
|state_ref|idle_disparity_ok_st|idle_i2b_st|10|
|tx_config_reg|0|0|20|
|tx_config_reg|ff|ff|22|
|tx_data|0|0|24|
|tx_data|ff|ff|26|
 # In time 6, tx_even is asserted but the tx_data is 0 instead of being bc as it is expected
 # In time 6, state_ref is driven 1 cycle after sgmii_cg_tx_st as idle_disparity_ok_st and check_ord_set_st are equivalent in this case
 # It can be seen the cycle of difference between tx_ord_set_en in time 8 and idle_i2b_st in time 10
 # In time 24 and 26, for a configuration transference, it can be seen how tx_data is driven from tx_config_reg from 2 cycles before 20 and 22

  !property_hold_1_clk_delay_output.png|thumbnail!
 [^waves_1_clk_delay_output.vcd]
 # Complementing the waveform, the verification code uses:
 !screenshot-1.png|thumbnail! 
 !screenshot-2.png|thumbnail! 
 !tx_even use.png|thumbnail!

# 

—
h1. Root Cause Analysis

<!-- Fill in once investigated -->
 - *Suspected block/logic:*
 - *Suspected cause:* (e.g. timing/CDC issue, incorrect state machine transition, off-by-one in counter, missing reset, race condition, incorrect sensitivity list, X-propagation)
 - *Analysis notes:*

—
h1. Impact
 - *Affected functionality:* 1 clk delay added (logic path is shorter this way), verification code
 - *Affected configurations/modes:*
 - *Downstream impact:* (e.g. blocks regression, blocks tapeout, blocks synthesis timing closure)

—
h1. Proposed Fix
 - Decide if the extra cycle delay is correct
 ** Option 1: (Preferred option) It is correct, keep as it is
 ** Option 2: It is not, use "next" value (flip flop entry) and analyze if that is possible for the desired frequency
 - Decide what to do with verification code
 ** Option 1: (Preferred option) Keep as it is and add comments about the details
 ** Option 2: Update the design to obtain the "next" value (flip flop entry) value (required for the update to the reference model) and use it only for the verification code, keep it untouch in the design code.
 ** Option 3: Thing about a new code states structure similar to the design one

—
h1. Verification of Fix
 - [ ] Fix implemented in RTL
 - [ ] Fix documented
 - [ ] Fix added as a requirement if needed
 - [ ] Unit-level test added/updated
 - [ ] Regression re-run and passing
 - [ ] Formal property re-checked (if applicable)
 - [ ] Lint/CDC clean
 - [ ] Code review completed