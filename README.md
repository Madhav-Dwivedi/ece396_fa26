# Big Core, Little Cores, or HLS Blocks

**A Comparative Study of Performance, Power, and Area (PPA) for Linear Algebra on a Spartan-7 FPGA**

> **ECE 396: Independent Study — Fall 2026**  
> **Student:** Dhruv Behl, Madhav Dwivedi  
> **Advisor:** Prof. Zuofu Cheng  
> **Institution:** Department of Electrical & Computer Engineering, University of Illinois Urbana-Champaign (UIUC)

---

## 📌 Project Overview

Modern FPGA system designers face a fundamental architectural choice when allocating a fixed budget of programmable logic for compute-intensive workloads:
1. **One Big General-Purpose Core:** A single, high-feature processor with floating-point hardware, caches, and branch prediction.
2. **Many Little Cores:** A parallel cluster of small, stripped-down cores with explicitly managed local scratchpad memory and DMA engines (reminiscent of the Cell Broadband Engine's SPU architecture).
3. **Dedicated HLS Blocks:** Fixed, application-specific hardware pipelines generated via High-Level Synthesis (AMD Vitis BLAS).

This research project conducts an empirical evaluation of these three paradigms on an **AMD/Xilinx Spartan-7 (XC7S50)** FPGA. We systematically characterize performance (GFLOPS), area (LUTs, FFs, BRAMs, DSPs), clock frequency ($F_{\max}$), and power consumption across standard linear algebra workloads to discover **crossover points** and determine which approach maximizes compute efficiency per unit of silicon area, per watt, and per joule.

---

## 🔬 Architectural Paradigms Under Test

All implementations run on the same base hardware platform (shared DDR3 memory controller, AXI bus infrastructure, system timers, and UART interface) using the **RISC-V-based MicroBlaze V** ISA:

| Paradigm | Architecture Details | Memory Hierarchy |
| :--- | :--- | :--- |
| **1. Big Core** | Single MicroBlaze V with RV32IMF, hardware branch prediction, and single-precision FPU. | 32 KB I-Cache + 32 KB D-Cache backed by DDR3. |
| **2. Little Cores** | Minimal MicroBlaze V host core + cluster of lean worker cores without caches. | Tightly-coupled local scratchpad memory (BRAM) + dedicated DMA engine per core. |
| **3. HLS Accelerators** | Minimal MicroBlaze V host core controlling dedicated, pipelined BLAS compute blocks. | Streaming AXI interfaces + AXI DMA directly to/from DDR3. |

---

## 📊 Benchmark Workloads

We evaluate all three architectures across standard Basic Linear Algebra Subprograms (BLAS Levels 1–3) and an iterative solver, scaling problem sizes from on-chip cache-resident dimensions to large off-chip DDR3-bound arrays:

* **BLAS Level 1:**
  * `dot`: Vector-vector inner product ($\alpha = \mathbf{x}^T \mathbf{y}$)
  * `axpy`: Scaled vector addition ($\mathbf{y} \leftarrow \alpha \mathbf{x} + \mathbf{y}$)
* **BLAS Level 2:**
  * `gemv`: Matrix-vector multiplication ($\mathbf{y} \leftarrow \alpha \mathbf{A}\mathbf{x} + \beta \mathbf{y}$)
* **BLAS Level 3:**
  * `gemm`: Matrix-matrix multiplication ($\mathbf{C} \leftarrow \alpha \mathbf{A}\mathbf{B} + \beta \mathbf{C}$)
* **Composite Workload:**
  * `conjugate gradient`: Iterative solver for linear systems ($\mathbf{A}\mathbf{x} = \mathbf{b}$), exercising dynamic memory access patterns and mixed BLAS Level 1/2 operations.

---

## 🛠️ Hardware & Toolchain Environment

* **Target Board:** RealDigital Urbana FPGA Board
* **FPGA Part:** AMD/Xilinx Spartan-7 `xc7s50csga324-1`
  * 52,160 Logic Cells (~8,150 Slices / 32,600 LUT6s)
  * 120 DSP48E1 Slices
  * 75 Block RAMs (36 Kb each; 2,700 Kb total)
  * 512 MB DDR3L Memory
* **EDA Toolchain:** AMD Vivado & Vitis 2024.x / 2025.x
* **Core Processor:** AMD MicroBlaze V (RISC-V RV32)
* **Host Control & Validation:** Python 3.10+ (NumPy, PySerial)

---

## 📁 Repository Structure

```text
├── fpga/              # Bare-metal C runtime code executed on FPGA
│   ├── big_core/          # Firmware for single large MicroBlaze V
│   ├── common/            # Shared packet drivers, cycle counters, headers
│   ├── hls_host/          # Host MicroBlaze firmware driving HLS/DMA IP
│   └── little_cores/      # Host dispatcher and worker scratchpad firmware
├── host/                  # Host-side Python automation & testing harness
│   ├── data/              # Benchmark telemetry outputs, CSVs, and figures
│   ├── generator.py       # NumPy test vector generation (reproducible seeds)
│   ├── protocol.py        # Framed binary UART transport layer with CRC-16
│   ├── runner.py          # Automated benchmark sweeper and metrics logger
│   └── validator.py       # Golden reference comparison and FP32 tolerance checks
├── hw/                    # Vivado project source files and automation
│   ├── ip/                # Custom IP, HLS cores, and peripheral wrappers
│   └── tcl/               # Batch scripts for synthesis, PnR, and report extraction
└── tests/                 # Unit tests for host-FPGA communication and framing