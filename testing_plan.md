# TESTING_SPEC.md: Automated Benchmarking & Verification Suite Specification

> **Project:** Comparative Study of Big Core, Little Cores, and HLS Blocks on AMD Spartan-7  
> **Target Board:** RealDigital Urbana FPGA (`xc7s50csga324-1`)  
> **Document Purpose:** Engineering specification for the host-side orchestration harness, UART transport protocol, bare-metal hardware instrumentation, and golden verification pipeline.

---

## 1. System Architecture & Testing Pipeline

The benchmarking environment functions as a closed-loop system where the **Host PC** orchestrates test execution, loads matrix/vector data into FPGA DDR3 memory, triggers execution, records cycle counts, and validates output payloads.

### 1.1 End-to-End Sequence Diagram

```text
Host PC (Python)                                FPGA (Urbana XC7S50)
   │                                                     │
   ├──[ 1. Ping / Sync Check ]──────────────────────────>│ (Host MB-V Firmware)
   │<─[ ACK / Ready ]────────────────────────────────────┤
   │                                                     │
   ├──[ 2. Load Input Buffers (Matrix A, B) ]───────────>│ -> DMA / Writes to DDR3
   │<─[ Checksum OK ]────────────────────────────────────┤
   │                                                     │
   ├──[ 3. Trigger Workload Execution ]─────────────────>│
   │                                                     │ [Flush D-Cache]
   │                                                     │ [Start 64-bit Cycle Counter]
   │                                                     │ === COMPUTE PHASE ===
   │                                                     │ (Big Core | Little Cores | HLS)
   │                                                     │ =====================
   │                                                     │ [Stop Cycle Counter]
   │                                                     │ [Invalidate D-Cache]
   │<─[ 4. Return Output Payload + Cycle Count ]─────────┤
   │                                                     │
   ├──[ 5. Golden Validation (NumPy Tolerance Check) ]
   ├──[ 6. Calculate GFLOPS, Efficiency, Metrics ]
   └──[ 7. Log to CSV / Telemetry Database ]
```

---

## 2. Communication & Transport Protocol Specification

The physical link runs over the Urbana on-board USB-UART bridge at **921,600 baud, 8-N-1 (8 data bits, no parity, 1 stop bit)** without hardware flow control.

To ensure speed and deterministic bare-metal parsing, all communication uses **framed binary packets** encoded in **Little-Endian (LE)**.

### 2.1 Host-to-FPGA Command Packet Layout

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          SYNC (0xAA55)        |    CMD_OP     |   TARGET_BUF  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          WORKLOAD_ID                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             DIM_M                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             DIM_N                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             DIM_K                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          PAYLOAD_LEN                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                   PAYLOAD DATA (0 to N Bytes)                 |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            CRC16-CCITT        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

#### Field Definitions
* **`SYNC` (16-bit):** Fixed preamble `0xAA55`.
* **`CMD_OP` (8-bit):**
  * `0x01` (`CMD_PING`): Health check and synchronization verification.
  * `0x02` (`CMD_WRITE_MEM`): Write payload directly to target buffer in DDR3.
  * `0x03` (`CMD_RUN`): Trigger computation for designated `WORKLOAD_ID`.
  * `0x04` (`CMD_READ_MEM`): Request payload from output buffer.
  * `0x05` (`CMD_SOFT_RESET`): Reset compute pipelines and buffer pointers.
* **`TARGET_BUF` (8-bit):**
  * `0x00`: Input Buffer A ($A \in \mathbb{R}^{M \times K}$)
  * `0x01`: Input Buffer B ($B \in \mathbb{R}^{K \times N}$)
  * `0x02`: Output Buffer Y ($Y \in \mathbb{R}^{M \times N}$)
* **`WORKLOAD_ID` (32-bit):**
  * `0x0001`: `DOT` (BLAS Level 1)
  * `0x0002`: `AXPY` (BLAS Level 1)
  * `0x0003`: `GEMV` (BLAS Level 2)
  * `0x0004`: `GEMM` (BLAS Level 3)
  * `0x0005`: `CONJUGATE_GRADIENT` (Iterative Solver)
* **`DIM_M`, `DIM_N`, `DIM_K` (32-bit each):** Problem dimensions (e.g., for `GEMM`: $M, N, K$; for `DOT`/`AXPY`: $N$).
* **`PAYLOAD_LEN` (32-bit):** Byte length of the following payload (must equal $\text{elements} \times 4$ for single-precision `fp32`).
* **`PAYLOAD DATA`:** Raw little-endian IEEE-754 32-bit floats.
* **`CRC16` (16-bit):** CRC-16-CCITT (`poly=0x1021`, `init=0xFFFF`) computed across all bytes from `CMD_OP` through the end of `PAYLOAD DATA`.

---

### 2.2 FPGA-to-Host Response Packet Layout

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          SYNC (0x55AA)        |  STATUS_CODE  |    RESERVED   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                      CYCLES_ELAPSED (64-bit)                  +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          PAYLOAD_LEN                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                   PAYLOAD DATA (0 to N Bytes)                 |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            CRC16-CCITT        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

#### Status Codes (`STATUS_CODE`)
* `0x00` (`STATUS_OK`): Operation succeeded.
* `0x01` (`STATUS_ERR_CRC`): Checksum validation failure on incoming packet.
* `0x02` (`STATUS_ERR_UNKNOWN_CMD`): Unrecognized opcode.
* `0x03` (`STATUS_ERR_DIM_MISMATCH`): Dimension exceeds maximum allocated DDR3 boundary.
* `0x04` (`STATUS_ERR_COMPUTE_TIMEOUT`): Hardware accelerator or worker core hung.

---

## 3. FPGA Memory Map & Core Instrumentation

The Urbana board provides 512 MB of DDR3 memory driven by the Xilinx Memory Interface Generator (MIG) IP, mapped starting at `0x8000_0000`.

### 3.1 DDR3 Memory Map

| Base Address | Size | Region Description |
| :--- | :--- | :--- |
| `0x8000_0000` | 16 MB | **Firmware Execution Region** (MicroBlaze V Text, Data, Heap, Stack) |
| `0x8100_0000` | 32 MB | **Buffer A (`BUF_A`)**: Matrix/Vector input operands |
| `0x8300_0000` | 32 MB | **Buffer B (`BUF_B`)**: Matrix/Vector input operands |
| `0x8500_0000` | 32 MB | **Buffer Y (`BUF_Y`)**: Output results |
| `0x8700_0000` | 16 MB | **Scratchpad / Inter-Core Mailbox**: Coordination ring buffers for Little Cores |

### 3.2 Cycle-Accurate Timing & Cache Maintenance

To measure compute efficiency without OS jitter or communication overhead, the timer must bound **only** the compute kernel.

```c
// Firmware execution pattern for all 3 designs
void run_benchmark_kernel(uint32_t workload_id, uint32_t m, uint32_t n, uint32_t k) {
    // 1. Ensure all memory writes to DDR3 are flushed out of the host core's D-Cache
    microblaze_flush_dcache();

    // 2. Snapshot start cycle
    uint64_t start_cycles = get_hardware_cycles();

    // 3. Dispatch to Compute Paradigm Under Test
    #if defined(DESIGN_BIG_CORE)
        big_core_execute(workload_id, m, n, k);
    #elif defined(DESIGN_LITTLE_CORES)
        dispatch_to_worker_cluster(workload_id, m, n, k);
        wait_for_workers_barrier();
    #elif defined(DESIGN_HLS_ACCELERATOR)
        trigger_hls_dma_pipeline(workload_id, m, n, k);
        wait_for_hls_done();
    #endif

    // 4. Snapshot end cycle
    uint64_t end_cycles = get_hardware_cycles();

    // 5. Invalidate D-Cache so Host reads updated DDR3 results instead of stale cache lines
    microblaze_invalidate_dcache();

    // 6. Record execution duration
    last_execution_cycles = end_cycles - start_cycles;
}
```

#### Cycle Counter Implementation (MicroBlaze V RISC-V)
MicroBlaze V implements the RISC-V `rdcycle` instruction. Reading the 64-bit cycle counter on a 32-bit architecture must account for 32-bit word rollover:

```c
static inline uint64_t get_hardware_cycles(void) {
    uint32_t hi0, hi1, lo;
    __asm__ __volatile__(
        "1:\n"
        "rdcycleh %0\n"
        "rdcycle  %1\n"
        "rdcycleh %2\n"
        "bne      %0, %2, 1b\n"
        : "=r"(hi0), "=r"(lo), "=r"(hi1)
    );
    return (((uint64_t)hi0) << 32) | lo;
}
```

---

## 4. Benchmark Workload Suite & FLOP Formulations

To ensure parity across all three implementations, algorithmic operation counts are defined identically for throughput calculations:

$$\text{Throughput (MFLOPS)} = \frac{\text{Floating Point Operations}}{\text{Cycles Elapsed} \times T_{\text{clk}}} \times 10^{-6}$$

| Workload | Mathematical Definition | Theoretical FLOP Count | Memory Footprint (FP32 Bytes) |
| :--- | :--- | :--- | :--- |
| **`dot`** | $\alpha = \mathbf{x}^T \mathbf{y}$ | $2N - 1$ | $2 \times 4N$ |
| **`axpy`** | $\mathbf{y} \leftarrow \alpha \mathbf{x} + \mathbf{y}$ | $2N$ | $3 \times 4N$ |
| **`gemv`** | $\mathbf{y} \leftarrow \alpha \mathbf{A}\mathbf{x} + \beta \mathbf{y}$ | $2MN + 2M$ | $(MN + M + N) \times 4$ |
| **`gemm`** | $\mathbf{C} \leftarrow \alpha \mathbf{A}\mathbf{B} + \beta \mathbf{C}$ | $2MNK + 2MN$ | $(MK + KN + MN) \times 4$ |
| **`cg`** | Iterative solution of $\mathbf{A}\mathbf{x} = \mathbf{b}$ ($I$ iterations) | $I \times (2N^2 + 10N)$ | $(N^2 + 4N) \times 4$ |

### 4.1 Problem Size Sweep Matrix

Experiments must sweep across two operational regimes:
1. **On-Chip / Cache-Resident Regime:** Small sizes fitting inside the Big Core’s 32 KB D-Cache and Little Cores' BRAM scratchpads (reveals maximum compute pipeline throughput).
2. **Off-Chip / DDR3-Bound Regime:** Large sizes forcing memory bus contention, DRAM row/column switching, and DMA stream latency.

```text
Dimension Grid (N or M=N=K):
[16, 32, 64, 128, 256, 512, 1024, 2048]
```

* For $N = 16$ (`GEMM`): Array size is $\approx 1 \text{ KB}$ (L1 Cache Resident).
* For $N = 1024$ (`GEMM`): Matrices $A, B, C$ consume $3 \times (1024 \times 1024 \times 4) = 12 \text{ MB}$ (Exceeds on-chip memory; bound to DDR3).

---

## 5. Golden Reference & Validation Specification

Because single-precision floating-point arithmetic is **non-associative**:
$$\left(\sum_{i=1}^{k} a_i\right)_{\text{serial CPU}} \neq \left(\sum_{i=1}^{k} a_i\right)_{\text{HLS unrolled systolic array}}$$
Direct bitwise binary comparison will cause false negatives. The validation engine (`host/validator.py`) enforces strict error tolerances against a 64-bit double-precision NumPy reference model.

### 5.1 Verification Metrics

For an FPGA result vector $\mathbf{y}_{\text{hw}}$ and NumPy reference $\mathbf{y}_{\text{ref}}$:

1. **Relative Tolerance Criteria (Pass/Fail):**
   $$\forall i, \quad |y_{\text{hw}}[i] - y_{\text{ref}}[i]| \le \text{atol} + \text{rtol} \times |y_{\text{ref}}[i]|$$
   * For single-precision `fp32`: Set $\text{rtol} = 10^{-4}$, $\text{atol} = 10^{-5}$.

2. **Maximum Absolute Error (MAE):**
   $$\text{MAE} = \max_i |y_{\text{hw}}[i] - y_{\text{ref}}[i]|$$

3. **Relative Root Mean Square Error (R-RMSE):**
   $$\text{R-RMSE} = \sqrt{\frac{\sum_{i} (y_{\text{hw}}[i] - y_{\text{ref}}[i])^2}{\sum_{i} (y_{\text{ref}}[i])^2}}$$

If any element evaluates to `NaN`, `Inf`, or exceeds the tolerance threshold, the run is flagged as `FAILED` and excluded from PPA scaling plots.

---

## 6. Vivado PPA Telemetry Extraction (Area, Timing, Power)

Automated Tcl scripts run post-implementation analysis headlessly to construct the PPA dataset.

### 6.1 Area Extraction
Parse `report_utilization -file util_route.rpt`:
* **Slice LUTs:** Total LUTs, LUTs as Logic, LUTs as Memory (LUTRAM).
* **Slice Registers:** Flip-Flops.
* **Block RAM:** Total 36 Kb BRAM equivalents (`RAMB36E1` + $0.5 \times \text{RAMB18E1}$).
* **DSPs:** `DSP48E1` count (Urbana ceiling is 120).

### 6.2 Timing & $F_{\max}$ Extraction
Parse `report_timing_summary -file timing_summary.rpt`:
* Extract **Worst Negative Slack (WNS)** on the compute clock domain:
  * If $\text{WNS} \ge 0$: Design closes timing at target clock frequency $F_{\text{target}}$.
  * Maximum achievable frequency:
    $$F_{\max} = \frac{1}{T_{\text{target}} - \text{WNS}}$$

### 6.3 Power Profiling
1. **Static / Baseline Vectorless Power:**
   Generated via `report_power -file power.rpt`.
2. **Dynamic Switching Activity File (SAIF) Injection:**
   To capture switching activity differences between Big Core stall cycles, Little Core scratchpad reads, and HLS streaming logic:
   * Run post-implementation timing simulation over 10,000 cycles for each kernel.
   * Export activity: `open_saif -file kernel.saif`, `log_saif [get_objects /DUT/*]`, `close_saif`.
   * Re-evaluate power: `read_saif kernel.saif` followed by `report_power`.

---

## 7. Step-by-Step Bring-up Checklist

Follow these engineering phases to build and verify the test harness incrementally:

- [ ] **Phase 1: Host Transport Unit Testing (Host-side only)**
  - Implement `host/protocol.py` with packet serializer, deserializer, and CRC-16 computation.
  - Verify packet packing and unpacking using synthetic payloads in Python unit tests.
- [ ] **Phase 2: Base Urbana Platform Creation (Hardware)**
  - Build minimal Vivado block design: MicroBlaze V, DDR3 MIG, AXI UARTLite (`921600` baud), AXI Timer.
  - Generate bitstream and export XSA hardware definition.
- [ ] **Phase 3: Firmware UART Echo & Memory Test**
  - Implement bare-metal firmware reading packets into DDR3 and echoing them back over UART.
  - Run host script to verify 100% data integrity across 64 MB transfers.
- [ ] **Phase 4: Hardware Timer Validation**
  - Verify that the cycle counter accurately measures software delay loops on MicroBlaze V.
- [ ] **Phase 5: Golden Validation Integration**
  - Write `host/generator.py` and `host/validator.py` for `dot` and `gemm`.
  - Execute a simple single-threaded C loop on the host MicroBlaze V (acting as Design 1: Big Core baseline) and verify NumPy tolerance check passes.
- [ ] **Phase 6: Automated Sweep & Metrics Logging**
  - Write `host/runner.py` to sweep matrix dimensions $N \in [16 \dots 1024]$, automatically recording execution cycles and writing results to `host/data/results.csv`.