# Vivado Simulation & Setup Guide for Spartan-7 FPGA

This guide provides step-by-step instructions to simulate and set up the **FPGA Ring-Oscillator TRNG and AES-128 HSM** project on a Xilinx/AMD **Spartan-7** FPGA (such as the Digilent Arty S7-50, Arty S7-25, or any custom Spartan-7 board) using **Xilinx Vivado Design Suite**.

---

## Overview

The project consists of two primary verification targets:
1. **Full AES-128 HSM Top (`tb_top.v`)**: Tests the entire pipeline (TRNG key accumulation -> AES key expansion -> UART command parsing -> Encrypt/Decrypt roundtrip).
2. **TRNG Core Top (`tb_trng_top.v`)**: Tests the multi-ring oscillator entropy source, synchronizers, and Von Neumann corrector, capturing 128 random bits.

---

## Requirements

- **Software**: Xilinx Vivado Design Suite (2020.1 or newer; Vivado ML Standard or Enterprise).
- **Target Family**: Spartan-7 (`spartan7`).
- **Example Part**: `xc7s50csga324-1` (Arty S7-50) or `xc7s25csga324-1` (Arty S7-25).

---

## Method 1: Vivado GUI Step-by-Step Instructions

### Step 1: Create a New Vivado Project

1. Launch **Vivado GUI**.
2. Click **Create Project** (or go to **File > Project > New**).
3. Click **Next** on the wizard welcome screen.
4. Set **Project name** to `trng_spartan7` and select your workspace location. Click **Next**.
5. Select **RTL Project** as the Project Type. Ensure *"Do not specify sources at this time"* is **unchecked** (or checked if you prefer adding files manually later). Click **Next**.

### Step 2: Select the Spartan-7 Board / Part

1. In the **Default Part** selection window:
   - Click the **Parts** tab.
   - Set **Family**: `Spartan-7`.
   - Set **Package**: `csga324` (or your board's package).
   - Set **Speed grade**: `-1`.
   - Select `xc7s50csga324-1` (or your specific Spartan-7 device).
   - *Alternatively*, click the **Boards** tab and select **Arty S7-50** or **Arty S7-25** if you have board files installed.
2. Click **Next**, then click **Finish**.

---

### Step 3: Add Design Sources (RTL)

1. In the **Sources** window (top left), right-click **Design Sources** and select **Add Sources** (or `Alt + A`).
2. Choose **Add or create design sources** and click **Next**.
3. Click **Add Files** and navigate to the project `src/` directory.
4. Select all Verilog source files:
   - `src/aes128.v`
   - `src/key_gen.v`
   - `src/ring_osc.v`
   - `src/synchronizer.v`
   - `src/top.v`
   - `src/trng_top.v`
   - `src/uart_rx.v`
   - `src/uart_tx.v`
   - `src/von_neumann.v`
5. Ensure **Copy sources into project** is **unchecked** (so edits sync with your repo) and click **Finish**.

---

### Step 4: Add Simulation Sources (Testbenches)

1. Right-click **Simulation Sources** -> **Add Sources**.
2. Choose **Add or create simulation sources** and click **Next**.
3. Click **Add Files** and select all files from `tb/`:
   - `tb/tb_top.v`
   - `tb/tb_trng_top.v`
   - `tb/tb_ring_osc.v`
4. Click **Finish**.

---

### Step 5: Configure Simulation & Language Settings

1. **Enable SystemVerilog for Testbenches**:
   - In the **Sources** pane, expand **Simulation Sources > sim_1**.
   - Right-click `tb_trng_top.v` -> **Source Node Properties**.
   - Set **File Type** to `SystemVerilog` (required because `tb_trng_top.v` uses SystemVerilog string constructs like `string plaintext`).

2. **CRITICAL: Verilog Macro Settings for Behavioral Simulation**:
   - **DO NOT** define the `SYNTHESIS` macro for Behavioral Simulation.
   - *Why?* In `ring_osc.v`, when `SYNTHESIS` is defined, the design builds a physical combinational feedback loop without `#` delays. In XSim behavioral simulation, this creates an infinite zero-delay loop causing Vivado to hang at `0 ns`.
   - When `SYNTHESIS` is **undefined**, `ring_osc.v` automatically uses a behavioral delay model with `$random` phase jitter, allowing XSim to simulate smoothly.

3. **Output Directory Setup (`build/` folder)**:
   - `tb_trng_top.v` writes captured random bits to `build/random_bits.txt`.
   - Before running `tb_trng_top`, create a `build` directory inside the Vivado XSim execution folder, or run this command in the Vivado Tcl Console:
     ```tcl
     file mkdir [file join [get_property DIRECTORY [current_project]] trng_spartan7.sim/sim_1/behav/xsim/build]
     file mkdir build
     ```

---

### Step 6: Run Behavioral Simulation

#### Option A: Run Full HSM UART & AES-128 Simulation (`tb_top.v`)

1. In **Sources > Simulation Sources > sim_1**, right-click `tb_top.v` and select **Set as Top**.
2. In the **Flow Navigator** (left panel), click **Simulation > Run Simulation > Run Behavioral Simulation**.
3. Vivado will compile and launch **XSim Waveform Window**.
4. In the simulation toolbar, click **Run All** (`F5`) or set simulation time to `100 us` and click **Run**.
5. Observe the Tcl Console log output:
   ```text
   [TB] Reset released. Waiting for TRNG + AES key schedule (~50 us)...
   [TB] Sending ENCRYPT command: 0xEE 0x05 'Hello'
   [TB] Waiting for 16-byte ciphertext...
   [TB] Received ciphertext: ...
   [TB] Sending DECRYPT command: 0xDD 0x10 ...
   [TB] PASS: Decrypted text matches original plaintext: "Hello"
   ```
6. Inspect signals in the waveform viewer:
   - `tb_tx` (UART input to FPGA)
   - `fpga_tx` (UART output from FPGA)
   - `dut/raw_key` (Captured 128-bit TRNG AES key)
   - `dut/aes_result` (128-bit AES ciphertext / decrypted plaintext)

#### Option B: Run TRNG Core Simulation (`tb_trng_top.v`)

1. In **Sources > Simulation Sources > sim_1**, right-click `tb_trng_top.v` and select **Set as Top**.
2. Click **Simulation > Run Simulation > Run Behavioral Simulation**.
3. The simulation runs until 128 corrected random bits are captured by the Von Neumann corrector.
4. Output in the Tcl Console:
   ```text
   Random bits: 10110...
   Generated key (hex): ...
   Ciphertext (hex): ...
   Decoded text: Hello harsha
   PASS: captured 128 corrected bits and decoded 12 text bytes
   ```

---

### Step 7: Run Post-Synthesis / Post-Implementation Timing Simulation (Optional)

To simulate physical ring oscillator loop delays after synthesis:

1. Click **Run Synthesis** in Flow Navigator.
2. Under **Synthesis Options**, ensure `SYNTHESIS` macro is defined (Vivado defines this automatically during synthesis).
3. Once synthesis completes, click **Simulation > Run Simulation > Run Post-Synthesis Functional Simulation** (or **Post-Synthesis Timing Simulation**).
4. Note: Timing simulation includes actual Spartan-7 routing and gate delays for the ring oscillators.

---

## Method 2: Automated Vivado Tcl Script Setup

You can fully automate project creation and simulation using Vivado's Tcl shell.

### Create `run_sim.tcl`

Save the following script as `run_sim.tcl` in the project root directory:

```tcl
# ==============================================================================
# Vivado Tcl Script: Spartan-7 TRNG & AES Simulation Setup
# ==============================================================================

# 1. Setup Project & Target Spartan-7 Device
set proj_name "trng_spartan7"
set proj_dir  "./vivado_proj"
set part_num  "xc7s50csga324-1" ;# Spartan-7 (e.g. Arty S7-50)

file mkdir $proj_dir
create_project -force $proj_name $proj_dir -part $part_num

# 2. Add RTL Design Sources
add_files -norecurse [glob src/*.v]
update_compile_order -fileset sources_1

# 3. Add Simulation Sources
add_files -fileset sim_1 -norecurse [glob tb/*.v]
set_property file_type {SystemVerilog} [get_files tb/tb_trng_top.v]
update_compile_order -fileset sim_1

# 4. Create output directory required for simulation logs
file mkdir "build"
file mkdir "$proj_dir/$proj_name.sim/sim_1/behav/xsim/build"

# 5. Set Top Module for Simulation (tb_top for HSM UART flow)
set_property top tb_top [get_filesets sim_1]
set_property top_lib xil_defaultlib [get_filesets sim_1]
update_compile_order -fileset sim_1

# 6. Launch Behavioral Simulation
launch_simulation -mode behavioral
run all
```

### Run Script via Terminal

Execute Vivado in batch or GUI mode from your terminal:

```bash
# Run in batch mode (command line only)
vivado -mode batch -source run_sim.tcl

# OR run in GUI mode (opens Waveform Viewer automatically)
vivado -mode gui -source run_sim.tcl
```

---

## Hardware Implementation Notes for Spartan-7

When moving from simulation to physical bitstream generation on a Spartan-7 board:

### 1. Board Clock Configuration
- Top level `top.v` has a parameter `CLK_FREQ` defaulting to `50_000_000` (50 MHz).
- If your Spartan-7 board has a different oscillator frequency (e.g., Arty S7 has a **12 MHz** clock on pin `R2` or **100 MHz** clock on pin `F14`), update the parameter when instantiating `top`:
  ```verilog
  top #(
      .CLK_FREQ(12_000_000),  // Set to 12 MHz for Arty S7-50 12MHz clock
      .BAUD_RATE(115_200)
  ) u_top ( ... );
  ```

### 2. Constraints File (`.xdc`) Example for Arty S7-50
Create a constraints file `constrs_1/arty_s7.xdc` for board implementation:

```xdc
## Clock signal (12 MHz Clock on Arty S7)
set_property -dict { PACKAGE_PIN R2    IOSTANDARD LVCMOS33 } [get_ports { i_clk }];
create_clock -add -name sys_clk_pin -period 83.333 -waveform {0 41.666} [get_ports { i_clk }];

## Active-Low Reset Switch / Button (e.g. Button 0)
set_property -dict { PACKAGE_PIN G15   IOSTANDARD LVCMOS33 } [get_ports { i_reset_n }];

## USB-UART Interfaces
set_property -dict { PACKAGE_PIN V12   IOSTANDARD LVCMOS33 } [get_ports { i_uart_rx }]; # FPGA RX <- PC TX
set_property -dict { PACKAGE_PIN R12   IOSTANDARD LVCMOS33 } [get_ports { i_uart_tx }]; # FPGA TX -> PC RX

## Ring Oscillator Combinational Loop DRC Override
set_property ALLOW_COMBINATIONAL_LOOPS TRUE [get_nets u_trng/u_ro*/loop_stage*]
```

### 3. Synthesis Attributes for Ring Oscillators
`src/ring_osc.v` contains:
```verilog
(* keep = "true", dont_touch = "true" *) wire [N-1:0] loop_stage;
```
These attributes ensure Vivado's synthesis and implementation tools do not optimize away or collapse the inverter ring oscillator loops during logic synthesis.

---

## Troubleshooting Guide

| Issue / Error | Cause | Solution |
| :--- | :--- | :--- |
| **XSim hangs at 0 ns** | `SYNTHESIS` macro is defined in behavioral sim. | Do NOT define `SYNTHESIS` macro in Behavioral Simulation settings. |
| **Cannot open `build/random_bits.txt`** | Missing `build/` folder in xsim execution path. | Run `file mkdir build` in Tcl Console or create `build` folder inside project directory. |
| **Syntax error on `string` in `tb_trng_top.v`** | Verilog-2001 mode used instead of SystemVerilog. | Right-click `tb_trng_top.v` -> Properties -> Change **File Type** to **SystemVerilog**. |
| **DRC Error: Combinational Loop Detected** | Vivado flags ring oscillator loops during Synthesis. | Add `set_property ALLOW_COMBINATIONAL_LOOPS TRUE` to your `.xdc` file or set DRC rule severity to Warning. |
| **UART baud rate mismatch** | Board clock frequency does not match `CLK_FREQ`. | Ensure `CLK_FREQ` matches your board oscillator frequency (e.g. 12MHz, 50MHz, 100MHz). |
