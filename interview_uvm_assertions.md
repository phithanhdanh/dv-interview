# Interview Session: UVM and Assertions

**Role:** DDR Design Verification Engineer  
**Goal:** Expose adaptability to an existing team bench — DDR training, math modeling, firmware-to-hardware debug on Verdi, UVM testbench reuse, and SVA for signal connection / timing checks.

**Format:** 3 categories. Each category has 1 leading question and 5 follow-ups. Answers are brief; deeper reading is linked.

You can add or remove questions in a later pass.

---

## Main subject

UVM + SystemVerilog assertions for joining an *existing* DDR PHY training bench — not building a toy environment from scratch. We want to see whether the candidate can extend our agents, scoreboards, and SVA without rewriting them, then debug FW-driven training against Verdi waveforms.

Ask them to walk a recent training-related fail end-to-end. If they only recite textbook UVM layers and never mention config, factory, bind, `disable iff`, or correlating CSRs to analog/digital training signals, they will struggle on this team.

---

## Questions

### Category 1 — Drop into our UVM bench, do not rebuild it

**L1.** You inherit a multi-agent DDR PHY UVM env (DFI, APB/AHB FW bus, DRAM model, training sequencer). A new training mode needs one extra knob and one extra sequence. What do you change first, and what do you refuse to touch?

1. How do you pass a virtual interface and a training-mode enum from the test into a nested agent you did not write?
2. When do you use a type override vs an instance override?
3. Why must components be created with `type_id::create()` and not `new()`?
4. How do you start a virtual sequence that coordinates FW register writes and DFI training traffic without a virtual sequencer?
5. A 2 ms training test dies at `UVM_TIMEOUT`. What do you inspect before raising the timeout?

### Category 2 — Assertions for connection and training timing

**L2.** We use SVA for “is this pin actually connected?” and “does this handshake meet the timing window?” How do you decide immediate vs concurrent, and where do you put the checker so RTL owners do not fight the TB compile?

1. Write the property shape for “after `req`, `ack` arrives in 1–5 clocks, ignore during reset or training-disable.”
2. What is the difference between `|->` and `|=>`, and when does DDR training care?
3. Why do connectivity assertions still fail when the waveform “looks connected”?
4. How do you keep timing assertions quiet during ZQ, reset, or mid-training re-init without commenting them out?
5. We need a check that two signals stay aligned within ±N ps, not N clocks. Is classic concurrent SVA enough?

### Category 3 — Math models, FW-to-HW debug, Verdi

**L3.** Training failed: FW wrote CSRs, hardware trained, scoreboard flagged a delay-line / Vref / eye result that does not match the model. Walk us from log line to the exact waveform cursor you drop in Verdi.

1. Where does a math / behavioral training model live in UVM, and how does it get stimulus?
2. FW says “write leveling done” but the PHY delay value looks wrong. What three views do you open in Verdi?
3. How do you debug an SVA failure in Verdi instead of only reading the log?
4. The model uses real/fixed-point math (eye width, tap code, PI step). Simulation compare mismatches by 1 LSB. What do you check before filing RTL?
5. How do you keep a long training test debuggable without dumping every net for 2 ms?

---

## Answers

### Category 1 — Drop into our UVM bench, do not rebuild it

**L1.** You inherit a multi-agent DDR PHY UVM env (DFI, APB/AHB FW bus, DRAM model, training sequencer). A new training mode needs one extra knob and one extra sequence. What do you change first, and what do you refuse to touch?

**Brief answer:** Change the config object + factory-created sequence. Do not hard-edit the env, agent, or driver unless the interface itself is wrong. That is how you stay compatible with regressions already owned by the team.

- In depth: [Verification Academy — UVM configuration](https://verificationacademy.com/cookbook/configuration)
- Related: [Verification Guide — UVM testbench architecture](https://verificationguide.com/uvm/uvm-testbench-architecture/)

1. How do you pass a virtual interface and a training-mode enum from the test into a nested agent you did not write?

   **Brief answer:** `uvm_config_db` set from the test / top, get in `build_phase` of the agent or driver. Use a typed config object, not a pile of loose ints.

   - In depth: [Verification Academy — UVM configuration](https://verificationacademy.com/cookbook/configuration)

2. When do you use a type override vs an instance override?

   **Brief answer:** Type override replaces every instance of that class (e.g. all training sequences). Instance override replaces one path (`env.dfi_agt.sqr`) so you do not break other interfaces.

   - In depth: [Verification Academy — factory](https://verificationacademy.com/cookbook/factory)

3. Why must components be created with `type_id::create()` and not `new()`?

   **Brief answer:** Factory overrides only work if construction goes through the factory. `new()` freezes the class and blocks reuse of the inherited bench.

   - In depth: [Verification Guide — UVM testbench architecture](https://verificationguide.com/uvm/uvm-testbench-architecture/)

4. How do you start a virtual sequence that coordinates FW register writes and DFI training traffic without a virtual sequencer?

   **Brief answer:** Hold sequencer handles in the virtual sequence (or env), `start()` child sequences on those handles. Virtual sequencers are optional and often extra hierarchy.

   - In depth: [Verification Academy — virtual sequences](https://verificationacademy.com/cookbook/sequences/virtual)

5. A 2 ms training test dies at `UVM_TIMEOUT`. What do you inspect before raising the timeout?

   **Brief answer:** Missing/dropped objections, drain time too short after the last FW write, and whether reset/training-idle phases never call `drop_objection`. Fix the objection protocol first.

   - In depth: [Verification Academy — objections](https://verificationacademy.com/cookbook/objections)

---

### Category 2 — Assertions for connection and training timing

**L2.** We use SVA for “is this pin actually connected?” and “does this handshake meet the timing window?” How do you decide immediate vs concurrent, and where do you put the checker so RTL owners do not fight the TB compile?

**Brief answer:** Immediate = point-in-time / X-check inside procedural code. Concurrent = multi-cycle protocol and timing on a clock. Prefer a bind-in assertion module or interface so the same checks ride RTL and TB.

- In depth: [systemverilog.io — SVA basics](https://www.systemverilog.io/verification/sva-basics/)
- Related: [Verification Guide — SystemVerilog assertions](https://verificationguide.com/systemverilog/systemverilog-assertions/)

1. Write the property shape for “after `req`, `ack` arrives in 1–5 clocks, ignore during reset or training-disable.”

   **Brief answer:** `@(posedge clk) disable iff (rst || train_off) req |-> ##[1:5] ack;` Concurrent, named, with a fail message.

   - In depth: [ChipVerify — assertion-based verification](https://chipverify.com/verification/assertion-based-verification)

2. What is the difference between `|->` and `|=>`, and when does DDR training care?

   **Brief answer:** `|->` checks the consequent in the same sample; `|=>` starts the next cycle. Use `|->` for same-cycle DFI/command encoding, `|=>` for “next beat / next UI” style responses.

   - In depth: [ChipVerify — assertion-based verification](https://chipverify.com/verification/assertion-based-verification)

3. Why do connectivity assertions still fail when the waveform “looks connected”?

   **Brief answer:** Concurrent SVA samples in the Preponed region. You may be looking at NBA-updated values in Verdi. Check clocking block vs raw net, and whether the assertion clock is the training clock or the FW bus clock.

   - In depth: [IEEE 1800-2017 SystemVerilog](https://ieeexplore.ieee.org/document/8299595)

4. How do you keep timing assertions quiet during ZQ, reset, or mid-training re-init without commenting them out?

   **Brief answer:** `disable iff` on a mode/reset/gate signal, plus `$assertoff`/`$asserton` only for known illegal windows. Never delete the property.

   - In depth: [Verification Guide — SystemVerilog assertions](https://verificationguide.com/systemverilog/systemverilog-assertions/)

5. We need a check that two signals stay aligned within ±N ps, not N clocks. Is classic concurrent SVA enough?

   **Brief answer:** Clock-cycle SVA is weak for sub-cycle PHY timing. Use local `realtime` variables, `@(edge)` properties, or `$setuphold`-style checks, and be explicit about the sampling event.

   - In depth: [systemverilog.io — SVA basics](https://www.systemverilog.io/verification/sva-basics/)

---

### Category 3 — Math models, FW-to-HW debug, Verdi

**L3.** Training failed: FW wrote CSRs, hardware trained, scoreboard flagged a delay-line / Vref / eye result that does not match the model. Walk us from log line to the exact waveform cursor you drop in Verdi.

**Brief answer:** Map the UVM error (time, agent, transaction) → FW register write that started that step → DFI/PHY signals for that step → assertion or model sample point. Then compare model expected vs sampled actual on the same clock. Do not start by clicking random nets.

- In depth: [Synopsys Verdi debug](https://www.synopsys.com/verification/debug.html)

1. Where does a math / behavioral training model live in UVM, and how does it get stimulus?

   **Brief answer:** In the scoreboard or a predictor subscriber. Monitors broadcast observed FW and DFI transactions on analysis ports; the model computes expected DQ/DQS/delay/Vref and compares. It must not sit inside the driver.

   - In depth: [The Art of Verification — UVM architecture](https://theartofverification.com/uvm-testbench-architecture/)

2. FW says “write leveling done” but the PHY delay value looks wrong. What three views do you open in Verdi?

   **Brief answer:** (1) FW bus + CSR writes, (2) DFI/training control and delay-update pulses, (3) the assertion/scoreboard fail marker and the sampled DQ/DQS pair. Cross-probe time from the UVM message.

   - In depth: [Synopsys Verdi debug](https://www.synopsys.com/verification/debug.html)

3. How do you debug an SVA failure in Verdi instead of only reading the log?

   **Brief answer:** Dump assertion evaluation into FSDB, add the assertion thread to the wave, inspect start/end times and local variables. That tells you which cycle of the sequence actually died.

   - In depth: [Synopsys VCS](https://www.synopsys.com/verification/simulation/vcs.html)

4. The model uses real/fixed-point math (eye width, tap code, PI step). Simulation compare mismatches by 1 LSB. What do you check before filing RTL?

   **Brief answer:** Rounding vs truncation, signedness, when the model samples vs when hardware updates the tap, and whether FW applied a post-process offset the model omitted. Off-by-one is usually model/FW contract, not a random RTL bug.

5. How do you keep a long training test debuggable without dumping every net for 2 ms?

   **Brief answer:** Dump from the failing training stage, use UVM verbosity + transaction recording, FSDB dump-on-demand around the fail time, and keep monitors/scoreboard messages tagged with step name and CSR address.

   - In depth: [SemiEngineering — waveform replay / Verdi IDX](https://semiengineering.com/intelligent-waveform-replay-for-efficient-debug/)

---

## What “good” sounds like in the room

- Speaks in *config, factory, bind, analysis ports, disable iff, sampled value, objection* — not “I would rewrite the testbench.”
- Can map a training algorithm step to a sequence, a CSR write, a DFI event, an SVA window, and a Verdi cursor.
- Treats the math model as a scoreboard contract, not a spreadsheet on the side.

## What “not ready for this bench” sounds like

- Only names driver / monitor / scoreboard with no reuse story.
- Puts protocol checks only inside the driver.
- Debug plan is “look at the waveform” with no time, transaction, or assertion thread.

---

## Next session options

Pick one category and go deeper:

1. Factory reuse and inherited UVM benches
2. SVA timing / connectivity for DDR training
3. Verdi + firmware-to-hardware debug
