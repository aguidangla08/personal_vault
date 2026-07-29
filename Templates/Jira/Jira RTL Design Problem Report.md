<!-- Set Issue fields

Details:
- Type
- Priority
- Components
- Labels
- Team
- Programs Affected

People:
- Assignee
- Reporter
- Owner

Dates

Attachments

Issue links

-->

h1. Summary

<!-- Description of the issue -->

---

h1. Environment

- *Component:*
- *Module:*
- *Simulator / Tool:*
- *version:*
- *OS / Server:*
- *Testbench / Verification IP:*
- *Regression / Test name:*
- *Seed (if applicable):*

---

h1. Description

h2. Expected Behavior

<!-- What the RTL/design is supposed to do per spec/microarchitecture doc -->

h2. Actual Behavior

<!-- What actually happened -->

h2. Spec / Documentation Reference

<!-- Link or section reference to microarchitecture spec, IP spec, protocol standard, etc. -->

---

h1. Steps to Reproduce

<!-- Structure example

1. Set up the required tool version (note OS version and other dependencies if they affect reproducibility)
2. Clone the `<repository-name>` repository and download submodules
3. Run `<tool-name>`
4. Inside `<tool-name>`, run `<script/command>`
5. Run `<specific check/test/step that triggers the issue>`

-->

---

h1. Signal / Waveform Evidence

|Signal Name|Expected Value|Actual Value|Time / Cycle|
|---|---|---|---|
|||||
|||||

<!-- Attach waveform screenshot or .fsdb/.vcd/.wlf file -->

---

h1. Root Cause Analysis

<!-- Fill in once investigated -->

- *Suspected block/logic:*
- *Suspected cause:* (e.g. timing/CDC issue, incorrect state machine transition, off-by-one in counter, missing reset, race condition, incorrect sensitivity list, X-propagation)
- *Analysis notes:*

---

h1. Impact

- *Affected functionality:*
- *Affected configurations/modes:*
- *Downstream impact:* (e.g. blocks regression, blocks tapeout, blocks synthesis timing closure)

---

h1. Proposed Fix

<!-- RTL diff, pseudocode, or description of the fix -->

{code}
// proposed change

{code}

---

h1. References

---

h1. Verification of Fix

- [ ] Fix implemented in RTL
- [ ] Fix documented
- [ ] Fix added as a requirement if needed
- [ ] Unit-level test added/updated
- [ ] Regression re-run and passing
- [ ] Formal property re-checked (if applicable)
- [ ] Lint/CDC clean
- [ ] Code review completed