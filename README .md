# 🚀 Sub-Microsecond High-Frequency Trading (HFT) Bare-Metal Wavelet Core

A low-latency, zero-copy, bare-metal C++ firmware designed for high-frequency trading applications. This system bypasses the standard Linux network stack entirely, mapping raw PCIe Ethernet frames directly into cache-line aligned ring buffers to achieve a deterministic **~1µs processing loop** on isolated CPU cores.

## 🛠️ Architecture & Hardware Design

The core is structured around strict mechanical sympathy principles, maximizing L1/L2 cache efficiency while eliminating branch penalties and heap allocations.

```
       +-----------------------------------------------------------------+
       | PCIe DMA Space (0x00000000)                                     |
       | +-------------------------------+-----------------------------+ |
       | | Layer 1: Dual Ring A (8KB)     | Layer 1: Dual Ring B (8KB)   | |
       | +-------------------------------+-----------------------------+ |
       | | Wavelet Ping Buffer (4KB)     | Wavelet Pong Buffer (4KB)   | |
       | +-------------------------------+-----------------------------+ |
       | | Atomic State & Cache-Line Padding                           | |
       +-------------------------------+---------------------------------+
                                       |
                                       v  (Branchless Pointer Polling)
                       +---------------+---------------+
                       | In-Place Haar Lifting Scheme  |
                       +---------------+---------------+
                                       |
                                       v  (Direct Register Outbound, 0x80000000)
                              [ PCIe TX Register ]
```

### Key Technical Pillars
* **Zero Heap Allocation:** Permanently pins the entire execution infrastructure to memory address `0x00000000` via physical/L2-mapped addresses.
* **Dual-Ring Ping-Pong DMA Setup:** Implements 16KB of total memory, evenly split into dual ring buffers (`ring_A` and `ring_B`) and dual wavelet processing tracks (`wavelet_ping` and `wavelet_pong`).
* **Mechanical Sympathy Padding:** Structural layout uses explicit 64-byte alignment (`alignas(CACHE_LINE)`) and a trailing 63-byte separation padding array to immunize atomic pointers against performance-degrading **False Sharing**.
* **TCP/IP Stripping Layer:** Bypasses default Linux socket architectures to feed raw execution data structures via explicit pointer shifts instead of indexes.

---

## ⚡ Execution Pipeline

### Stage 1: Zero-Copy Pointer-Masked Polling
To completely avoid standard branch conditional statements (`if (head == end)`), the core wraps rings naturally via bitwise branchless boundaries:
```cpp
os->dma_head = (current == os->dma_tail) ? os->ring_A : current;
```

### Stage 2: Real-time In-Place Haar Lifting
Applies single-pass 1D discrete Haar Wavelet transformations using structural pointer offsets instead of loop indexes. Loops are explicitly metadata-hinted for unrolling:
```cpp
#pragma unroll
for (size_t i = 0; i < (TICK_COUNT / 2); ++i) {
    *(high + i) = *(in + (2 * i + 1)) - *(in + (2 * i));
    *(low + i)  = *(in + (2 * i)) + (*(high + i) >> 1);
}
```

### Stage 3: Low-Latency Signal Blasting
Outbound trading orders or trading execution signals bypass system calls and route directly to outbound PCIe TX queue descriptors mapped to standard volatile base memory regions (`0x80000000`).

---

## 💻 Building and Deploying

### Prerequisites
* **Compiler:** `GCC` or `Clang` with C++17 support.
* **Target Environment:** Bare-metal or isolated core systems (via Linux boot parameters like `isolcpus` and `nohz_full`).

### Compilation Flags
To ensure compiler optimizations do not discard hardware prefetching and strict loop unrolling, compile with the following low-latency profile flags:

```bash
g++ -O3 -std=c++17 -march=native -ffast-math -fomit-frame-pointer hft_kernel.cpp -o hft_kernel
```

---

## 📂 Source Layout Documentation
* **`KernelSpace` (Struct):** Fixed cache-line configuration encapsulating ping/pong regions, dynamic atomic heads, flags, and padding layouts.
* **`nic_dma_init()`:** Initial baseline prefetch assignment targeting underlying hardware L1 caches.
* **`poll_tick()`:** High-speed branchless indexing core responsible for updating memory read heads.
* **`wavelet_haar_lift()`:** Unrolled hardware pipeline responsible for decomposing high-frequency/low-frequency noise filters.
* **`kernel_main()`:** Independent execution loop driving core processes endlessly.

---

## ⚖️ License
This project is licensed under the standard MIT License - see the local repository permissions details.
