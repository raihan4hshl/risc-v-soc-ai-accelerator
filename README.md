# RISC-V SoC with AI/ML Accelerator
### A Hardware-Software Co-design of an AI-Accelerated RISC-V SoC on FPGA

![Platform](https://img.shields.io/badge/Platform-Artix--7%20FPGA-blue)
![Core](https://img.shields.io/badge/CPU-NEORV32%20RISC--V-orange)
![ISA](https://img.shields.io/badge/ISA-RV32IMC-green)
![Tool](https://img.shields.io/badge/Tool-Vivado%202022.x-red)
![Status](https://img.shields.io/badge/Status-In%20Progress-yellow)

---

## Overview

This project implements a **RISC-V SoC** on an Artix-7 FPGA using the [NEORV32](https://github.com/stnolting/neorv32) softcore processor, extended with a custom **MAC (Multiply-Accumulate) hardware accelerator** for AI/ML inference workloads.

The central research objective is to quantify the performance gain of offloading matrix/vector operations from the RISC-V CPU to a dedicated hardware accelerator — measuring execution time, speedup, and resource overhead on real silicon.

**Key contributions:**
- NEORV32 RV32IMC softcore deployed on Artix-7 FPGA
- Custom MAC engine integrated as a memory-mapped peripheral (Wishbone slave)
- C firmware HAL for seamless CPU ↔ accelerator offload
- Cycle-accurate benchmarking using RISC-V `MCYCLE` CSR
- Comparative performance analysis: software-only vs hardware-accelerated execution

---

## Project Status

| Phase | Description | Status |
|-------|-------------|--------|
| 1 | NEORV32 bringup on Artix-7 | ✅ Complete |
| 2 | SoC peripheral configuration | 🔄 In Progress |
| 3 | MAC engine RTL design + simulation | ⏳ Pending |
| 4 | Wishbone bus integration | ⏳ Pending |
| 5 | Software driver + firmware | ⏳ Pending |
| 6 | CPU vs accelerator benchmarking | ⏳ Pending |
| 7 | Scale-up / DMA / platform migration | ⏳ Pending |
| 8 | Thesis report | ⏳ Pending |

---

## Repository Structure

```
risc-v-soc-ai-accelerator/
├── hw/
│   ├── neorv32/              # NEORV32 submodule (do not edit)
│   ├── soc/                  # SoC top-level wrapper + clock PLL
│   ├── accelerator/          # MAC engine RTL + testbench
│   └── constraints/          # Vivado XDC pin constraint files
├── sw/
│   ├── common/               # Linker script, shared headers
│   ├── drivers/              # MAC accelerator HAL (mac_accel.h/c)
│   ├── benchmarks/           # Benchmark programs + CSV results
│   └── tests/                # Bringup tests (hello world, blink, ISA check)
├── sim/
│   ├── ghdl/                 # GHDL simulation scripts
│   ├── modelsim/             # ModelSim wave config files
│   └── waveforms/            # Captured .vcd waveforms
├── vivado/
│   ├── create_project.tcl    # Recreates Vivado project from script
│   ├── synth_report/         # Utilisation reports (baseline + with MAC)
│   └── timing_report/        # Timing summary reports
├── docs/
│   ├── report/               # Thesis chapters
│   ├── diagrams/             # Architecture block diagrams
│   └── datasheets/           # Referenced component datasheets
└── results/
    ├── benchmark_data.csv    # All benchmark measurements
    └── plots/                # Speedup and resource plots
```

---

## Hardware Platform

| Parameter | Details |
|-----------|---------|
| FPGA | Xilinx Artix-7 (Nexys A7-100T / Basys 3) |
| CPU | NEORV32 RISC-V softcore (RV32IMC) |
| Clock | 50 MHz (via MMCM from 100 MHz board clock) |
| IMEM | 16 KB internal BRAM |
| DMEM | 8 KB internal BRAM |
| Accelerator | Custom MAC engine (Wishbone-mapped) |
| Debug | JTAG On-Chip Debugger (OCD) |
| Host interface | USB-UART (115200 baud) |

---

## Quick Start

### Prerequisites

- Xilinx Vivado 2022.x (or later)
- RISC-V GCC toolchain: `riscv32-unknown-elf-gcc`
- OpenOCD (for JTAG debug)
- Python 3 + NumPy (for simulation golden reference)

### 1. Clone the repository

```bash
git clone --recurse-submodules https://github.com/<your-username>/risc-v-soc-ai-accelerator.git
cd risc-v-soc-ai-accelerator
```

> The `--recurse-submodules` flag is required to pull the NEORV32 submodule.

### 2. Recreate the Vivado project

```bash
vivado -mode batch -source vivado/create_project.tcl
```

This generates the full Vivado project from the Tcl script — no binary project files are stored in the repo.

### 3. Build and flash

Open the generated Vivado project, run Synthesis + Implementation + Generate Bitstream, then program via Hardware Manager.

### 4. Build firmware

```bash
cd sw/tests/hello_world
make MARCH=rv32imc MABI=ilp32 clean_all all
```

Upload `neorv32_exe.bin` via the UART bootloader (19200 baud, send binary after pressing `u` at the `CMD:>` prompt).

### 5. Run benchmarks

```bash
cd sw/benchmarks
make MARCH=rv32imc MABI=ilp32 TARGET=bench_sw_mac clean_all all
# upload and record cycle counts printed over UART
```

---

## Architecture

```
         ┌─────────────────────────────────────────────────────┐
         │                 NEORV32 SoC (Artix-7)               │
         │                                                       │
         │  ┌───────────────┐       ┌──────────────────────┐   │
         │  │  NEORV32 CPU  │       │   MAC Accelerator    │   │
         │  │   RV32IMC     │◄─────►│  (Wishbone Slave)    │   │
         │  │               │ MMIO  │  8-wide parallel MAC │   │
         │  └───────┬───────┘       └──────────────────────┘   │
         │          │                                            │
         │   ┌──────┴──────────────────────────┐               │
         │   │         Internal Bus             │               │
         │   └──┬──────────┬──────────┬────────┘               │
         │      │          │          │                          │
         │  ┌───┴──┐  ┌────┴───┐  ┌──┴─────┐                  │
         │  │ IMEM │  │  DMEM  │  │  GPIO  │                   │
         │  │ 16KB │  │   8KB  │  │  UART  │                   │
         │  └──────┘  └────────┘  │  MTIME │                   │
         │                        └────────┘                    │
         └─────────────────────────────────────────────────────┘
                        │               │
                    UART/USB         JTAG OCD
                  (bootloader,     (GDB debug)
                   benchmark I/O)
```

---

## Benchmark Results

> Results will be updated as each phase completes.

| Test | Vector Size | CPU Cycles | Accel Cycles | Speedup |
|------|-------------|-----------|--------------|---------|
| Dot product | N=16 | TBD | TBD | TBD |
| Dot product | N=64 | TBD | TBD | TBD |
| Dot product | N=256 | TBD | TBD | TBD |
| FC layer (MLP) | 64×64 | TBD | TBD | TBD |

---

## Resource Utilisation (Artix-7)

> Vivado implementation reports — updated per phase.

| Resource | Baseline (CPU only) | With MAC Accelerator | Overhead |
|----------|--------------------|--------------------|---------|
| LUTs | TBD | TBD | TBD |
| FFs | TBD | TBD | TBD |
| BRAM | TBD | TBD | TBD |
| DSP48 | TBD | TBD | TBD |
| fmax | TBD | TBD | TBD |

---

## Tools & Dependencies

| Tool | Version | Purpose |
|------|---------|---------|
| Xilinx Vivado | 2022.x | Synthesis, implementation, bitstream |
| riscv32-unknown-elf-gcc | 12.x | Bare-metal C firmware compilation |
| GHDL | 3.x | RTL simulation (MAC testbench) |
| OpenOCD | 0.12.x | JTAG on-chip debug |
| Python + NumPy | 3.10+ | Golden reference for verification |
| GTKWave | 3.3.x | Waveform analysis |

---

## References

- [NEORV32 Processor Datasheet](https://stnolting.github.io/neorv32/)
- [NEORV32 GitHub Repository](https://github.com/stnolting/neorv32)
- [Xilinx Artix-7 FPGA Datasheet (DS180)](https://www.xilinx.com/support/documentation/data_sheets/ds180_7Series_Overview.pdf)
- [RISC-V ISA Specification](https://riscv.org/technical/specifications/)
- [Wishbone B4 Bus Specification](https://cdn.opencores.org/downloads/wbspec_b4.pdf)

---

## Author

**[Your Name]**  
8 ECTA — Hardware-Software Co-design  
[Your Institution]  
[Year]

---

## License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.  
The NEORV32 processor is licensed under the BSD 3-Clause License by Stephan Nolting.
