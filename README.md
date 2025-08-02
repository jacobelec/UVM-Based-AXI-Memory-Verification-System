# AXI Verification with UVM
## Project Overview
This project implements and verifies an AXI-compliant slave using a full UVM testbench.  
You’ll find:

- **Design**: `AXI_Design_Code.sv`  
- **Testbench**: `AXI_tb.sv` (includes interface, sequences, driver, monitor, agent, env, test, and top‐level TB)

I’ve also included Jupyter Notebook files that break down the code step by step, making it easier to understand.  
You'll find:

- **Design**: `Breaking_down_the_design_code.ipynb`
- **Testbench**: `Breaking_down_the_testbench.ipynb`

## AXI Transaction Flow 
<img width="206" height="373" alt="image" src="https://github.com/user-attachments/assets/bd7c81db-6072-40f0-ae15-2a40041378d0" />

- In AXI3 and AXI4, there are only two AXI interface types, master and slave. All AXI connections are between master and slave interfaces.
- The diagram above show how AXI connections join master and slave interfaces,
- The direct connection gives maximum bandwidth between the master and slave components with no extra logic.
  And with AXI, there is only a single protocol to validate!

## AXI Channels
<img width="598" height="302" alt="image" src="https://github.com/user-attachments/assets/4dac3956-32af-4b1c-b478-13f811a0806b" />

- Write Address Channel (AW)
  - Used by the master to send the target write address to the slave.
  - Defines where the data should be written.
    
- Write Data Channel (W)
  - Used by the master to transfer the actual data values to be written.
  - Works in parallel with the AW channel (address and data are separate).
    
- Write Response Channel (B)
  - Used by the slave to respond to the master after completing a write.
  - Can indicate success or error
    
- Read Address Channel (AR)
  - Used by the master to send the target read address to the slave.
  - Defines where the data should be read from.

- Read Data Channel (R)
  - Used by the slave to send the requested read data back to the master.
  - Can also return an error code if the read fails (e.g., invalid address, corrupted data, or security violation).

## ⚙️ Design: `AXI_Design_Code.sv`

- **Module:** `axi_slave`  
- **Channels supported:**  
  - Write-Address (AW)  
  - Write-Data (W)  
  - Write-Response (B)  
  - Read-Address (AR)  
  - Read-Data (R)  
- **Burst types:** FIXED, INCR, WRAP  
- **Size/length parameters:** awsize, awlen, etc.  
- **Internal memory:** 128‐byte array (mem[128])  
- **Next-address functions** for each burst mode, used to compute and expose nextaddr signals for the monitor.

## 🔧 Testbench: `AXI_tb.sv`

1. **Interface**  
   - `axi_if` bundles all AXI signals plus internal pointers (`next_addrwr`, `next_addrrd`).

2. **Transaction**  
   - `class transaction extends uvm_sequence_item`  
   - Carries all AXI fields + an `oper_mode` enum (`wrrdfixed`, `wrrdincr`, `wrrdwrap`, `wrrderrfix`, `rstdut`).

3. **Sequences**  
   - `rst_dut` → generates reset operations.  
   - `valid_wrrd_fixed` → valid write-read fixed burst.  
   - `valid_wrrd_incr` → valid write-read incrementing burst.  
   - `valid_wrrd_wrap` → valid write-read wrapping burst.  
   - `err_wrrd_fix` → error-injection fixed burst (illegal address).

4. **Driver** (`uvm_driver#(transaction)`)  
   - Implements tasks:  
     - `reset_dut()`  
     - `wrrd_fixed_wr()/wrrd_fixed_rd()`  
     - `wrrd_incr_wr()/wrrd_incr_rd()`  
     - `wrrd_wrap_wr()/wrrd_wrap_rd()`  
     - `err_wr()/err_rd()`  
   - Converts each `transaction` into pin-level AXI stimulus.

5. **Monitor** (`uvm_monitor`)  
   - Snoops the `axi_if`, rebuilds write data in a local array, compares subsequent reads (or checks for error responses), and logs pass/fail.

6. **Agent** (`uvm_agent`)  
   - Groups the `driver`, `uvm_sequencer`, and `monitor`.  
   - Wires TLM ports in `connect_phase`.

7. **Environment** (`uvm_env`)  
   - Instantiates one `agent` in its `build_phase`.

8. **Test** (`uvm_test`)  
   - Creates the `env` and all sequences.  
   - In `run_phase`, raises an objection, starts whichever sequence(s) you choose on `env.a.seqr`, waits, then drops the objection.

9. **Top‐Level TB Module**  
   - Instantiates `axi_if vif()` and the DUT.  
   - Generates a 10 ns clock.  
   - Uses `uvm_config_db` to publish `vif`.  
   - Calls `run_test("test")` to launch UVM.  
   - Dumps a `dump.vcd` waveform.  
   - Connects DUT’s `nextaddr` → `vif.next_addrwr` and `rdnextaddr` → `vif.next_addrrd` for the monitor.
