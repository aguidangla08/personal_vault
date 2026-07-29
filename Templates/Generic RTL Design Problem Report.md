# RTL Design Problem Report

## Summary

<!-- One-line description of the issue -->

---

## Issue Details

|Field|Value|
|---|---|
|**Reporter**||
|**Date Found**||
|**Priority**|Blocker / Critical / Major / Minor / Trivial|
|**Severity**||
|**Component**|e.g. Core / Memory Controller / Bus Interconnect / Clock Domain|
|**Module / Instance**|e.g. `top.cpu.alu_inst`|
|**RTL Language**|Verilog / SystemVerilog / VHDL|
|**Design Version / Tag**|e.g. `rtl_v2.4.1` / commit hash|
|**Simulator / Tool**|e.g. VCS, Questa, Xcelium, Verilator|
|**Target Stage**|Simulation / Formal / Synthesis / Gate-Level Sim / Silicon|

---

## Environment

- **Simulator version:**
- **OS / Server:**
- **Testbench / Verification IP:**
- **Regression / Test name:**
- **Seed (if applicable):**
- **Waveform file location:**
- **Log file location:**

---

## Description

### Expected Behavior

<!-- What the RTL/design is supposed to do per spec/microarchitecture doc -->

### Actual Behavior

<!-- What actually happened -->

### Spec / Documentation Reference

<!-- Link or section reference to microarchitecture spec, IP spec, protocol standard, etc. -->

---

## Steps to Reproduce

**Reproducibility:** Always / Intermittent / Seed-dependent / One-time

---

## Signal / Waveform Evidence

|Signal Name|Expected Value|Actual Value|Time / Cycle|
|---|---|---|---|
|||||
|||||

<!-- Attach waveform screenshot or .fsdb/.vcd/.wlf file -->

---

## Root Cause Analysis

<!-- Fill in once investigated -->

- **Suspected block/logic:**
- **Suspected cause:** (e.g. timing/CDC issue, incorrect state machine transition, off-by-one in counter, missing reset, race condition, incorrect sensitivity list, X-propagation)
- **Analysis notes:**

---

## Impact

- **Affected functionality:**
- **Affected configurations/modes:**
- **Downstream impact:** (e.g. blocks regression, blocks tapeout, blocks synthesis timing closure)

---

## Proposed Fix

<!-- RTL diff, pseudocode, or description of the fix -->

```verilog
// proposed change

```

---

## Verification of Fix

- [ ] Fix implemented in RTL
- [ ] Unit-level test added/updated
- [ ] Regression re-run and passing
- [ ] Formal property re-checked (if applicable)
- [ ] Lint/CDC clean
- [ ] Code review completed

---

## Attachments

- [ ] Waveform dump
- [ ] Simulation log
- [ ] Schematic/block diagram
- [ ] Relevant spec excerpt

---

## Linked Issues

- **Related Jira tickets:**
- **Blocks:**
- **Blocked by:**
- **Duplicate of:**

---

## Labels / Tags

`rtl` `design-bug` `<component-name>` `<team-name>`