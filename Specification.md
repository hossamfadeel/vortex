# Vortex GPGPU — Complete Hardware & Software Specification

This document provides a complete hardware and software specification for the Vortex GPGPU, extracted from the official repository at `https://github.com/hossamfadeel/vortex`. All information is based on the source code and documentation found within the repository.

## A. High-level Overview

The Vortex GPGPU is a full-stack, open-source General-Purpose GPU based on the RISC-V instruction set architecture. It is designed to be a scalable and configurable platform for a wide range of parallel computing workloads.

*   **Architecture Purpose**: To provide a complete, open-source GPGPU platform for research, development, and education, enabling exploration of GPU architectures and parallel programming models.
*   **Supported Workloads**: The architecture is suitable for various parallel workloads, including machine learning, graph analytics, and scientific computing. It supports the OpenCL 1.2 standard.
*   **Relationship to RISC-V ISA**: Vortex extends the base RISC-V ISA (RV32IMAF/RV64IMAFD) with custom instructions for GPU-specific operations, such as warp scheduling, thread synchronization, and control-flow divergence handling.
*   **SIMT Model**: Vortex employs a Single Instruction, Multiple Threads (SIMT) execution model. Threads are grouped into warps, and all threads within a warp execute the same instruction simultaneously.
*   **Core/Cluster Layout**: The architecture is hierarchical, with cores grouped into sockets, and sockets grouped into clusters. This allows for scalable designs with multiple levels of cache and memory sharing.

## B. Shader Core Architecture

The Vortex shader core is a 6-stage, in-order pipeline designed for SIMT execution. Each core can execute multiple warps concurrently through time-multiplexing.

### Pipeline Stages

The core pipeline consists of the following six stages, as detailed in `docs/microarchitecture.md`:

1.  **Schedule**: The warp scheduler selects the next active warp to issue an instruction from. It tracks stalled and active warps and manages control-flow divergence using an IPDOM (Immediate Post-Dominator) stack.
2.  **Fetch**: Retrieves the instruction for the scheduled warp from the instruction cache (I-cache) or main memory.
3.  **Decode**: Decodes the fetched instruction and identifies the required functional unit and operands.
4.  **Issue**: Decoded instructions are placed in a per-warp instruction buffer (IBuffer). The scoreboard checks for register dependencies, and the operands collector fetches the required operands from the register file.
5.  **Execute**: The instruction is executed by one of the available functional units (ALU, FPU, LSU, SFU).
6.  **Commit**: The result of the execution is written back to the register file, and the scoreboard is updated.

### Pipeline Diagram

```
+----------+    +-------+    +--------+    +-------+    +---------+    +--------+
| Schedule | -> | Fetch | -> | Decode | -> | Issue | -> | Execute | -> | Commit |
+----------+    +-------+    +--------+    +-------+    +---------+    +--------+
     |              |            |             |            |              |
 Warp Scheduler   I-Cache      Decoder     IBuffer,     ALU, FPU,      Writeback
 IPDOM Stack      Access                   Scoreboard,  LSU, SFU
                                           Operands
```

### Detailed Components

*   **Warp Scheduler**: A key component responsible for managing the execution of multiple warps. It selects which warp to execute next based on their status (active, stalled). The logic is implemented in `hw/rtl/core/VX_schedule.sv`.
*   **Register File**: Each thread has its own set of 32 integer and 32 floating-point registers.
*   **Thread/Warp Execution**: Warps are the unit of scheduling. A single warp is issued per cycle. Threads within a warp execute the same instruction, with a thread mask controlling which threads are active.
*   **Functional Units (FUs)**: The execute stage contains multiple FUs:
    *   **ALU Unit**: Handles integer arithmetic and branch operations (`hw/rtl/core/VX_alu_unit.sv`).
    *   **FPU Unit**: Handles floating-point operations.
    *   **LSU Unit**: Handles load and store operations to memory.
    *   **SFU Unit**: Handles special function operations, including warp control and CSR access (`hw/rtl/core/VX_sfu_unit.sv`).
*   **Configuration**: The number of warps and threads per core is configurable via `NUM_WARPS` and `NUM_THREADS` in `hw/rtl/VX_config.vh`.
*   **Scoreboarding**: A scoreboard (`hw/rtl/core/VX_scoreboard.sv`) is used to track register dependencies and prevent RAW (Read-After-Write) hazards.

## C. Memory System

The Vortex memory system is a hierarchical, multi-level cache architecture designed for high-bandwidth and low-latency memory access.

### Memory Hierarchy Diagram

```
+------------+
| Main Memory|
+------------+
      ^
      |
+------------+
|  L3 Cache  | (Optional)
+------------+
      ^
      |
+------------+
|  L2 Cache  | (Optional)
+------------+
      ^
      |
+------------+
|  L1 Cache  | (I-Cache & D-Cache)
+------------+
      ^
      |
+------------+
| Shader Core|
+------------+
```

### Components

*   **L1 Cache**: Each core has a dedicated L1 instruction cache (I-Cache) and L1 data cache (D-Cache). These are configurable and can be enabled or disabled via `ICACHE_ENABLE` and `DCACHE_ENABLE` in `hw/rtl/VX_define.vh`.
*   **L2/L3 Cache**: The architecture supports optional L2 and L3 caches, which are shared among clusters of cores. Their presence is controlled by `L2_ENABLED` and `L3_ENABLED`.
*   **Shared Memory / Scratchpad**: The architecture includes a local memory (LMEM) that can be used as a software-managed scratchpad for fast data sharing between threads in a warp. The base address is defined by `LMEM_BASE_ADDR`.
*   **Cache Coherence**: The documentation does not specify a hardware-based cache coherence protocol. Coherence is likely managed by software.
*   **Cache Microarchitecture**: The cache is a non-blocking, write-through design with multiple parallel banks and per-bank Miss Status Holding Registers (MSHRs) to handle outstanding misses. The implementation details are in `hw/rtl/cache/` and described in `docs/cache_subsystem.md`.


## D. Execution Model (SIMT)

Vortex implements the SIMT (Single Instruction, Multiple Threads) execution model, which is fundamental to its operation as a GPGPU.

*   **Warp Execution**: Threads are grouped into warps, and a single instruction is fetched and executed for all threads in a warp. The program counter (PC) is shared across the warp.
*   **Divergence Handling**: When threads within a warp take different paths in a conditional branch, the hardware handles this through predication and an IPDOM (Immediate Post-Dominator) stack. The `SPLIT` and `JOIN` instructions are used to manage divergent paths, and a thread mask determines which threads are active for each path. The IPDOM stack is implemented in `hw/rtl/core/VX_ipdom_stack.sv`.
*   **Synchronization**: Vortex provides several mechanisms for thread synchronization:
    *   **Barriers**: The `BAR` instruction provides a hardware barrier for synchronizing threads within a core. The number of barriers is configurable via `NUM_BARRIERS`.
    *   **Fences**: Standard RISC-V `FENCE` instructions are supported for memory ordering.
    *   **Atomic Operations**: The architecture supports atomic memory operations (AMOs) for thread-safe updates to shared memory.

## E. ISA / Instruction Support

Vortex extends the standard RISC-V ISA with custom instructions for GPU operations. The base ISA is configurable and can be either RV32IMAF or RV64IMAFD.

### Vortex-Specific Instructions

The following custom instructions are defined in `docs/microarchitecture.md`:

| Instruction | Description |
|---|---|
| `TMC count` | **Thread Mask Control**: Activates the specified number of threads. |
| `WSPAWN count, addr` | **Warp Spawn**: Activates the specified number of warps and jumps to the given address. |
| `SPLIT taken, predicate` | **Split**: Manages control-flow divergence by applying a predicate and saving the current state to the IPDOM stack. |
| `JOIN` | **Join**: Restores the thread mask from the IPDOM stack. |
| `PRED predicate, restore_mask` | **Predicate**: Applies a predicate to the thread mask. |
| `BAR id, count` | **Barrier**: Stalls warps at a barrier until the specified count is reached. |

### Instruction Formats

The instruction formats follow the standard RISC-V formats (R, I, S, B, U, J). The opcodes for the custom instructions are defined in `sim/simx/instr.h`.

## F. Accelerator Organization

The Vortex GPGPU is organized hierarchically to allow for scalability.

*   **Cores and Clusters**: The number of cores per cluster and the total number of clusters are configurable. `NUM_CORES` and `NUM_CLUSTERS` in `hw/rtl/VX_config.vh` control this.
*   **Interconnect**: The cores, caches, and memory are connected through a network of AXI-compliant buses and interfaces.
*   **Clocking**: The documentation does not specify the clocking strategy in detail, but it is assumed that there are multiple clock domains for the cores, caches, and memory controllers.

## G. CSR Map

The Control and Status Registers (CSRs) provide an interface for controlling and monitoring the GPU. The CSR map is defined in `hw/rtl/VX_types.vh` and `hw/rtl/core/VX_csr_data.sv`.

### Key CSRs

| Address | Name | Description |
|---|---|---|
| 0x001 | `VX_CSR_FFLAGS` | Floating-Point Flags |
| 0x002 | `VX_CSR_FRM` | Floating-Point Rounding Mode |
| 0x003 | `VX_CSR_FCSR` | Floating-Point Control and Status Register |
| 0xCC0 | `VX_CSR_THREAD_ID` | Current Thread ID |
| 0xCC1 | `VX_CSR_WARP_ID` | Current Warp ID |
| 0xCC2 | `VX_CSR_CORE_ID` | Current Core ID |
| 0xCC3 | `VX_CSR_ACTIVE_WARPS` | Bitmask of active warps |
| 0xCC4 | `VX_CSR_ACTIVE_THREADS` | Bitmask of active threads in the current warp |
| 0xFC0 | `VX_CSR_NUM_THREADS` | Number of threads per warp |
| 0xFC1 | `VX_CSR_NUM_WARPS` | Number of warps per core |
| 0xFC2 | `VX_CSR_NUM_CORES` | Total number of cores |
| 0xFC3 | `VX_CSR_LOCAL_MEM_BASE` | Base address of local memory |

## H. Software Stack

The Vortex software stack includes a runtime library and support for the OpenCL programming model.

*   **Runtime API**: The runtime API, defined in `runtime/include/vortex.h`, provides functions for device management, memory allocation, kernel launching, and synchronization.
*   **Driver**: The driver is responsible for initializing the device, loading kernels, and managing the execution of programs. The source code for the various driver backends (simx, rtlsim, etc.) is in the `runtime/` directory.
*   **Compiler/Toolchain**: Vortex uses a standard RISC-V toolchain (GCC or LLVM) to compile host code and a custom compiler to generate GPU kernels. The `pocl` (Portable Computing Language) library is used for OpenCL support.

## I. RTL-Level Insights

A deeper look at the RTL provides additional insights into the hardware implementation.

*   **Top-Level Module**: The top-level module is `Vortex.sv` (`hw/rtl/Vortex.sv`), which instantiates the clusters, caches, and memory interfaces.
*   **Core Submodules**: The `hw/rtl/core/` directory contains the SystemVerilog modules for each pipeline stage and functional unit.
*   **Hazard Detection and Scoreboarding**: The scoreboard (`hw/rtl/core/VX_scoreboard.sv`) is the primary mechanism for detecting and handling data hazards.
*   **Pipeline Control**: The `VX_schedule.sv` module is central to pipeline control, managing warp scheduling and stalling.

## J. References

*   `docs/microarchitecture.md`
*   `docs/cache_subsystem.md`
*   `hw/rtl/VX_config.vh`
*   `hw/rtl/VX_define.vh`
*   `hw/rtl/VX_types.vh`
*   `hw/rtl/core/VX_core.sv`
*   `hw/rtl/core/VX_schedule.sv`
*   `hw/rtl/core/VX_csr_data.sv`
*   `sim/simx/instr.h`
*   `runtime/include/vortex.h`
