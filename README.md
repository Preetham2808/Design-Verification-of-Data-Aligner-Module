
# Data Aligner — UVM Verification Environment

A SystemVerilog/UVM verification environment for the **Data Aligner (CFS Aligner)** RTL module — a configurable IP that converts an unaligned input data stream into an aligned output stream based on a runtime-programmable `(SIZE, OFFSET)` configuration.

> Verified using **Cadence Xcelium 25.03** with the UVM 1.2 methodology and functional coverage.

---

## 1. Overview

The Aligner takes in an unaligned stream of data on its **RX interface** and re-aligns it according to the `CTRL.SIZE` / `CTRL.OFFSET` configuration before sending it out on its **TX interface**. Its purpose is to optimize memory writes by emitting only the writes best suited for the memory type used in the system.

**Interfaces:**
- **APB (AMBA 3)** — register configuration and status interface
- **MD (Memory Data)** — a custom valid/ready streaming protocol, used on both RX (input) and TX (output)

**Key parameters:**

| Parameter | Default | Description |
|---|---|---|
| `ALGN_DATA_WIDTH` | 32 | Width (bits) of `md_rx_data` / `md_tx_data`. Power of 2, min 8. |
| `FIFO_DEPTH` | 8 | Depth of the internal RX and TX FIFOs |

---

## 2. RTL Architecture

| Module | Responsibility |
|---|---|
| `cfs_rx_ctrl.v` | RX backpressure (`md_rx_ready`) + legality check on `(offset, size)`, drop counting |
| `cfs_synch_fifo.v` | RX/TX FIFOs (parameterized depth) |
| `cfs_ctrl.v` | Core alignment logic based on `CTRL.SIZE` / `CTRL.OFFSET` |
| `cfs_tx_ctrl.v` | Drives the TX MD interface |
| `cfs_regs.v` | APB register file (CTRL, STATUS, IRQEN, IRQ) |
| `cfs_edge_detect.v` | Edge detection used for sticky/W1C interrupt generation |
| `cfs_synch.v` | Generic synchronizer |
| `cfs_aligner_core.v` | Top-level core, instantiates all submodules |
| `cfs_aligner.v` | Top-level wrapper (DUT) |

### Register Map

| Register | Offset | Access | Description |
|---|---|---|---|
| `CTRL` | `0x0000` | RW | `SIZE[2:0]`, `OFFSET[9:8]`, `CLR[16]` (WO) |
| `STATUS` | `0x000C` | RO | `CNT_DROP[7:0]`, `RX_LVL[11:8]`, `TX_LVL[19:16]` |
| `IRQEN` | `0x00F0` | RW | Per-event interrupt enables |
| `IRQ` | `0x00F4` | W1C | Sticky interrupt status |

Interrupt events: `RX_FIFO_EMPTY`, `RX_FIFO_FULL`, `TX_FIFO_EMPTY`, `TX_FIFO_FULL`, `MAX_DROP`.

For full signal/register descriptions see [`aligner_datasheet_v_1_1.pdf`](./aligner_datasheet_v_1_1.pdf).

---

## 3. Verification Environment

Built on **UVM 1.2** with a standard layered architecture: `Test -> Environment -> Agents -> Interfaces <-> DUT`, plus a scoreboard, functional coverage model, and a RAL register model.

```
cfs_algn_test_pkg
 |- cfs_algn_test_base
     |- cfs_algn_test_reg_access
     |- cfs_algn_test_random

cfs_algn_env
 |- APB Agent      (cfs_apb_pkg)        - drives/monitors register interface
 |- MD RX Agent    (cfs_md_pkg)         - drives unaligned input stream
 |- MD TX Agent    (cfs_md_pkg, BFM)    - drives tx_ready/tx_err, monitors output
 |- Register Model (cfs_algn_reg_pkg)   - UVM RAL model of CTRL/STATUS/IRQ/IRQEN
 |- Reg Predictor  (cfs_algn_reg_predictor.sv)
 |- Scoreboard     (cfs_algn_scoreboard.sv)
 |- Coverage       (cfs_algn_coverage.sv)
 |- Virtual Sequencer (cfs_algn_virtual_sequencer.sv)
```

### Source File Map

| File | Purpose |
|---|---|
| `cfs_algn_pkg.sv` | Top-level environment package |
| `cfs_algn_env.sv` / `cfs_algn_env_config.sv` | UVM environment + configuration object |
| `cfs_algn_if.sv` | Aligner-specific interface bind |
| `cfs_apb_pkg.sv` / `cfs_apb_if.sv` | APB agent (driver, monitor, sequencer) |
| `cfs_md_pkg.sv` / `cfs_md_if.sv` | MD protocol agent (RX & TX) |
| `cfs_algn_reg_pkg.sv` | UVM RAL register model for CTRL/STATUS/IRQEN/IRQ |
| `cfs_algn_reg_predictor.sv` | Keeps the RAL mirror in sync with bus activity |
| `cfs_algn_model.sv` | Reference/prediction model for alignment behavior |
| `cfs_algn_scoreboard.sv` | Compares predicted vs. actual TX output |
| `cfs_algn_coverage.sv` | Functional coverage groups |
| `cfs_algn_types.sv` | Shared typedefs/enums |
| `cfs_algn_clr_cnt_drop.sv` | Sequence item / logic for `CTRL.CLR` -> `CNT_DROP` reset |
| `cfs_algn_reg_access_status_info.sv` | Helper info object for register-access status tracking |
| `cfs_algn_split_info.sv` | Helper for splitting/aligning RX data into TX-sized chunks |
| `cfs_algn_seq_reg_config.sv` | Sequence for configuring CTRL via APB |
| `cfs_algn_virtual_sequence_base.sv` | Base virtual sequence (handles to all sequencers) |
| `cfs_algn_virtual_sequence_reg_access_random.sv` | Random legal/illegal register access traffic |
| `cfs_algn_virtual_sequence_reg_access_unmapped.sv` | Accesses to unmapped APB addresses |
| `cfs_algn_virtual_sequence_slow_pace.sv` | Slow/backpressure-style stimulus (stalled `tx_ready`) |
| `cfs_algn_virtual_sequencer.sv` | Virtual sequencer holding APB/RX/TX sequencer handles |
| `cfs_algn_test_base.sv` | Common test setup |
| `cfs_algn_test_reg_access.sv` | Register access test (RW/RO/WO/W1C, APB error scenarios) |
| `cfs_algn_test_random.sv` | Constrained-random functional test |
| `cfs_algn_test_defines.sv` | Test-level `define`s (e.g. `ALGN_DATA_WIDTH`) |
| `cfs_algn_test_pkg.sv` | Test package, includes all tests |
| `uvm_ext_pkg.sv` | UVM extension/utility package |
| `design.sv` | RTL `include` wrapper (instantiates all DUT modules) |
| `testbench.sv` | Top-level testbench module (clock gen, DUT instance, UVM run) |
| `messages.f` | UVM verbosity/message configuration file |

### RTL Files

`cfs_synch.v`, `cfs_synch_fifo.v`, `cfs_rx_ctrl.v`, `cfs_ctrl.v`, `cfs_tx_ctrl.v`, `cfs_edge_detect.v`, `cfs_regs.v`, `cfs_aligner_core.v`, `cfs_aligner.v`

---

## 4. Tests

| Test | Description |
|---|---|
| `cfs_algn_test_reg_access` | Verifies register access rules: RW/RO/WO/W1C behavior, APB errors on unmapped addresses, writes to read-only `STATUS`, and illegal `(SIZE, OFFSET)` writes to `CTRL` |
| `cfs_algn_test_random` | Constrained-random functional test: legal and illegal MD RX transfers, alignment correctness, FIFO fill/drain, interrupt generation |

---

## 5. Functional Coverage

Coverage groups (see `cfs_algn_coverage.sv`) include:
- All legal `SIZE` x `OFFSET` combinations on `CTRL` (cross coverage)
- Legal and illegal `(rx_offset, rx_size)` combinations on the MD RX interface
- RX/TX FIFO fill levels (empty, partial, full)
- Each interrupt bit (`RX_FIFO_EMPTY/FULL`, `TX_FIFO_EMPTY/FULL`, `MAX_DROP`) - set and cleared
- APB error scenarios (unmapped access, `STATUS` write, illegal `CTRL` write)

---

## 6. Tooling

| Tool | Version |
|---|---|
| Simulator | Cadence Xcelium `25.03-s001` |
| Methodology | UVM 1.2 (CDNS) |
| Waveform viewer | SimVision |

---

## 7. References

- [`aligner_datasheet_v_1_1.pdf`](./aligner_datasheet_v_1_1.pdf) — full functional specification (interface signals, register map, alignment behavior, flow control, interrupts)

---


