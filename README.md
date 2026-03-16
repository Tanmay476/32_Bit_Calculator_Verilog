# 32-Bit Calculator with 64-Bit Memory Organization

A SystemVerilog-based calculator design with a class-based verification environment modeled on UVM methodology. The design reads 64-bit words from dual-bank SRAM, performs 32-bit additions on each half, and writes results back to memory.

## Project Overview

This project consists of RTL implementation of a calculator system and its verification testbench. The calculator reads 64-bit memory words (split across two 32-bit SRAM blocks), adds the lower and upper 32-bit halves of each word, and stores results back to memory via a 64-bit result buffer.

### Key Features

- **Dual 32-bit SRAM banks** using Sky130 SRAM macros for 64-bit memory word organization
- **FSM-based controller** managing read, add, and write operations
- **32-bit ripple-carry adder** built from chained full adders using generate constructs
- **64-bit result buffer** for accumulating two 32-bit sums before writeback
- **Comprehensive verification** with directed and randomized testing
- **SystemVerilog Assertions (SVA)** for protocol checking
- **Class-based testbench** modeled on UVM methodology with driver, monitor, sequencer, and scoreboard

## Architecture

### RTL Components (`rtl/`)

- **`top_lvl.sv`**: Top-level integration module
  - Connects controller, SRAM, adder, and result buffer
  - Manages dual-port SRAM interface

- **`controller.sv`**: FSM-based control logic
  - States: `S_IDLE`, `S_READ`, `S_ADD`, `S_WRITE`, `S_END`
  - Reads pairs of 64-bit words, feeds each word's two 32-bit halves to the adder
  - Toggles buffer location to store lower and upper 32-bit results
  - Synchronous active-high reset

- **`adder32.sv`**: 32-bit ripple-carry adder
  - Composed of 32 full adders via generate-for construct
  - Produces 32-bit sum (carry-out is not propagated)

- **`full_adder.sv`**: Single-bit full adder building block

- **`result_buffer.sv`**: 64-bit result buffer
  - Stores two sequential 32-bit addition results (lower and upper halves)
  - Location selector for sequential writes
  - Synchronous active-high reset

- **`sky130_sram_2kbyte_1rw1r_32x512_8.sv`**: Dual-port SRAM macro
  - 512 words x 32 bits
  - Port 0: Read/Write
  - Port 1: Read-only

- **`calculator_pkg.sv`**: Design parameter package
  - `DATA_W = 32`: Data width (adder operand size)
  - `MEM_WORD_SIZE = 64`: Memory word size (two SRAM banks combined)
  - `ADDR_W = 9`: Address width (512 entries)

### Testbench Components (`tb/`)

- **`calc_tb_top.sv`**: Top-level testbench
  - Instantiates DUT and verification components
  - Runs directed and randomized tests
  - Includes SystemVerilog assertions

- **`calc_if.sv`**: Calculator interface
  - Clocking blocks for synchronous communication
  - Signal declarations for DUT connections

- **`calc_driver.svh`**: Test driver
  - Drives stimulus to DUT
  - Initializes SRAM contents via backdoor writes
  - Controls start/reset sequences

- **`calc_monitor.svh`**: Transaction monitor
  - Observes DUT outputs
  - Captures transactions via mailbox

- **`calc_sequencer.svh`**: Test sequence generator
  - Generates randomized test sequences
  - Creates constrained-random transaction items

- **`calc_seq_item.svh`**: Transaction item definition
  - Encapsulates test data with address range constraints

- **`calc_sb.svh`**: Scoreboard
  - Maintains SRAM memory mirrors
  - Computes expected 32-bit sums and compares against DUT output

- **`calc_tb_pkg.sv`**: Testbench package
  - Imports verification components

## Getting Started

### Prerequisites

- SystemVerilog simulator (VCS or Cadence)
- Waveform viewer (Verdi for VCS or SimVision for Cadence)

### Directory Structure

```
64_Bit_Calculator_Verification/
├── rtl/                    # RTL design files
│   ├── top_lvl.sv
│   ├── controller.sv
│   ├── adder32.sv
│   ├── full_adder.sv
│   ├── result_buffer.sv
│   ├── calculator_pkg.sv
│   └── sky130_sram_2kbyte_1rw1r_32x512_8.sv
└── tb/                     # Testbench files
    ├── calc_tb_top.sv
    ├── calc_if.sv
    ├── calc_driver.svh
    ├── calc_monitor.svh
    ├── calc_sequencer.svh
    ├── calc_seq_item.svh
    ├── calc_sb.svh
    └── calc_tb_pkg.sv
```

### Compilation and Simulation

#### For VCS:

```bash
# Compile
vcs -sverilog -full64 -debug_access+all \
    -timescale=1ns/1ps \
    +define+VCS \
    rtl/*.sv tb/*.sv tb/*.svh

# Run simulation
./simv
```

#### For Cadence:

```bash
# Compile
xrun -64bit -sv -access +rwc \
     +define+CADENCE \
     rtl/*.sv tb/*.sv tb/*.svh
```

### Waveform Viewing

#### VCS (Verdi):

```bash
verdi -ssf simulation.fsdb &
```

#### Cadence (SimVision):

```bash
simvision waves.shm &
```

## Operation Flow

1. **Initialization**: SRAM banks loaded with operands via testbench backdoor writes
2. **Read Phase**: Controller reads a 64-bit word (32 bits from each SRAM bank)
3. **Add Phase**: 32-bit adder computes `SRAM_A[addr] + SRAM_B[addr]`
4. **Buffer Phase**: 32-bit result stored in lower or upper half of the result buffer
5. **Repeat**: Steps 2-4 repeat for the next address, filling the other buffer half
6. **Write Phase**: Combined 64-bit result buffer written back to SRAM
7. **End State**: Controller enters `S_END` and remains there until reset

## Memory Organization

- **64-bit words** split across two SRAM banks:
  - `sram_A`: Lower 32 bits (bits [31:0])
  - `sram_B`: Upper 32 bits (bits [63:32])
- **Address space**: 512 entries (9-bit addressing)
- The adder operates on 32-bit halves independently; there is no carry propagation between lower and upper halves

## Test Cases

### Directed Tests

1. **Normal Addition** (`tb/calc_tb_top.sv:170`)
   - Basic addition: 10 + 20 = 30
   - Verifies single-address operation

2. **Overflow Handling** (`tb/calc_tb_top.sv:202`)
   - Tests 32-bit overflow: 0xFFFFFFFF + 1 = 0x00000000
   - Validates upper/lower half independence

3. **Zero Addition** (`tb/calc_tb_top.sv:247`)
   - Edge case: 0 + 0 = 0

4. **FSM Coverage Tests** (`tb/calc_tb_top.sv:273`)
   - Single-read/multi-write scenarios
   - Multi-read/single-write scenarios

5. **Toggle Stress Test** (`tb/calc_tb_top.sv:318`)
   - Exercises all address bits
   - Tests buffer location toggling

6. **Reset-at-State Tests** (`tb/calc_tb_top.sv:344`)
   - Pulses reset at each FSM state
   - Validates reset recovery behavior

### Randomized Testing

- **200 random transactions** (`tb/calc_tb_top.sv:363`)
- Generated by sequencer with constrained-random addresses
- Executed by driver, validated by scoreboard

## SystemVerilog Assertions

The testbench includes several SVA checks (`tb/calc_tb_top.sv:380`):

1. No simultaneous read/write
2. Single-cycle initialization pulse
3. No operations during reset
4. Valid address ranges (end >= start)
5. Ready signal consistency with FSM state

## Known Limitations

- Only supports addition (no subtraction, multiplication, or other operations)
- 32-bit arithmetic with no carry propagation between halves
- Sequential operation (no pipelining between transactions)
- No external output ports; results are only observable via SRAM backdoor reads
- Controller stays in `S_END` indefinitely until externally reset

## License

This project is an educational design for learning hardware verification techniques.

## Contact

For questions or issues, please open an issue in the GitHub repository.
