# CLAUDE.md - AI Assistant Guide for HTP Ops Library

## Project Overview

This is the **HTP Ops Library**, a custom operator library for Qualcomm's Hexagon Tensor Processor (HTP). It provides optimized kernels for LLM inference on Qualcomm Hexagon NPU, designed to work with the [llama.cpp-npu](https://github.com/haozixu/llama.cpp-npu) project.

**Research Paper**: [Scaling LLM Test-Time Compute with Mobile NPU on Smartphones](https://arxiv.org/abs/2509.23324)

**Target Hardware**: Qualcomm Snapdragon 8 Gen 2+ (Hexagon DSP v73+) with FP16 HMX support

**Status**: Research prototype, not production-ready

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│ Host (Android/CPU - AArch64)                            │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ libhtp_ops.so (FastRPC Stub)                        │ │
│ │   - session.c    (DSP session management)           │ │
│ │   - op_export.c  (operation dispatching)            │ │
│ │   - test.c       (RPC test harness)                 │ │
│ └─────────────────────────────────────────────────────┘ │
└──────────────────┬──────────────────────────────────────┘
                   │ FastRPC + rpcmem (shared memory IPC)
┌──────────────────▼──────────────────────────────────────┐
│ DSP (Hexagon NPU - Q6DSP)                               │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ libhtp_ops_skel.so (FastRPC Skeleton)               │ │
│ │ ┌─────────────────────────────────────────────────┐ │ │
│ │ │ Operators (src/dsp/ops/):                       │ │ │
│ │ │   flash_attn.c  - Flash Attention               │ │ │
│ │ │   mat_mul.c     - Matrix multiplication (GEMM)  │ │ │
│ │ │   rms_norm.c    - RMS Normalization             │ │ │
│ │ └─────────────────────────────────────────────────┘ │ │
│ │ ┌─────────────────────────────────────────────────┐ │ │
│ │ │ Infrastructure:                                 │ │ │
│ │ │   op_executor.cc - Operator dispatch            │ │ │
│ │ │   vtcm_mgr.cc    - On-chip memory management    │ │ │
│ │ │   hmx_mgr.c      - HMX accelerator state        │ │ │
│ │ │   worker_pool.c  - Multi-threaded execution     │ │ │
│ │ └─────────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

## Directory Structure

```
htp-ops-lib/
├── include/
│   ├── dsp/                    # DSP-side headers
│   │   ├── hmx_mgr.h          # HMX (matrix accelerator) management
│   │   ├── hmx_utils.h        # HMX inline assembly helpers
│   │   ├── hvx_*.h            # HVX (SIMD) math utilities
│   │   ├── ops.h              # Operator function signatures
│   │   ├── quants.h           # Quantization format definitions
│   │   ├── vtcm_mgr.h         # VTCM memory management
│   │   └── worker_pool.h      # Worker pool interface
│   ├── host/                   # Host-side headers
│   │   ├── session.h          # DSP session management
│   │   └── op_export.h        # Operator export interface
│   ├── htp_ops.idl            # FastRPC interface definition
│   ├── message.h              # Message protocol structures
│   └── op_reg.h               # Operator registry & parameter structs
├── src/
│   ├── dsp/                    # DSP implementations (runs on NPU)
│   │   ├── ops/               # Operator kernels
│   │   │   ├── flash_attn.c   # Flash Attention implementation
│   │   │   ├── mat_mul.c      # Matrix multiplication kernels
│   │   │   ├── rms_norm.c     # RMS normalization
│   │   │   └── mm_benchmark.c # GEMM micro-benchmarks
│   │   ├── op_executor.cc     # Operator dispatch logic
│   │   ├── vtcm_mgr.cc        # Vector TCM memory manager
│   │   └── worker_pool.c      # Thread pool implementation
│   └── host/                   # Host implementations (runs on CPU)
│       ├── test.c             # RPC test suite
│       ├── session.c          # Session initialization
│       └── op_export.c        # Operator export handlers
├── CMakeLists.txt             # Build configuration
├── .clang-format              # Code formatting rules
└── README.md                  # Project documentation
```

## Build System

### Prerequisites

1. Hexagon SDK 6.x (verified with 6.0.0.2)
2. Source the SDK environment:
   ```bash
   source $HEXAGON_SDK_ROOT/setup_sdk_env.source
   ```

### Build Commands

```bash
# Build for Android CPU (AArch64) - produces libhtp_ops.so
build_cmake android

# Build for Hexagon DSP - produces libhtp_ops_skel.so
build_cmake hexagon DSP_ARCH=v73   # Recommended default
build_cmake hexagon DSP_ARCH=v75   # For newer hardware
build_cmake hexagon DSP_ARCH=v79   # May have float issues (see Known Issues)
```

### Build Outputs

| Output | Target | Description |
|--------|--------|-------------|
| `android_ReleaseG_aarch64/libhtp_ops.so` | CPU | FastRPC stub (runs on host) |
| `hexagon_ReleaseG_toolv87_v73/libhtp_ops_skel.so` | NPU | FastRPC skeleton (runs on DSP) |
| `htp_ops_test` | CPU | Test executable |

## Code Style and Conventions

### Formatting

- **Tool**: clang-format (config in `.clang-format`)
- **Language**: C++17 / C11
- **Indentation**: 2 spaces (no tabs)
- **Column limit**: 120 characters
- **Line endings**: LF

Run formatter:
```bash
clang-format -i src/**/*.c src/**/*.cc include/**/*.h
```

### Naming Conventions

| Pattern | Meaning | Example |
|---------|---------|---------|
| `hvx_*` | HVX intrinsic wrapper | `hvx_exp2_f32()` |
| `hmx_*` | HMX operation | `hmx_load_tiles_fp16()` |
| `*_f16`, `*_f32` | Floating-point precision | `rms_norm_f32()` |
| `*_qf16`, `*_qf32` | Quantized floating-point | - |
| `vtcm_*` | Vector TCM operations | `vtcm_alloc()` |

### Header Guards

Use `#pragma once` (modern style) for all headers.

### Struct Packing

Use `__attribute__((packed))` for RPC parameter structs to ensure consistent layout across CPU/DSP:

```c
struct MatMulParams {
  struct RpcmemBufAddr output;
  struct RpcmemBufAddr activation;
  struct RpcmemBufAddr weight;
  int32_t m, k, n;
} __attribute__((packed));
```

### Memory Alignment

VTCM allocations require 4KB alignment:
```c
__attribute__((aligned(4096))) static uint8_t vtcm_buffer[VTCM_SIZE];
```

Vector operations require 128-byte alignment:
```c
__attribute__((aligned(VLEN))) HVX_Vector vec;
```

## Key Concepts

### Hardware Accelerators

**HMX (Hexagon Matrix eXtension)**:
- AI accelerator for matrix operations
- Tile-based: 32x32 FP16 tiles ("Croutons")
- Used for: GEMM, Flash Attention
- Requires `-mhmx` compiler flag

**HVX (Hexagon Vector eXtension)**:
- SIMD vector processor
- 128-byte wide vectors
- Used for: RMS norm, exp2, data formatting

### Memory Hierarchy

| Memory | Speed | Size | Use Case |
|--------|-------|------|----------|
| VTCM | Fastest | ~256KB | Hot working data |
| L2 Cache | Fast | ~1MB | Intermediate data |
| DDR/rpcmem | Slow | GBs | Input/output buffers |

### Quantization Formats

Supported formats (from `quants.h`):
- **Floating-point**: FP32, FP16, BF16
- **4-bit**: Q4_0, Q4_1, IQ4_NL, IQ4_XS
- **5-bit**: Q5_0, Q5_1
- **6-bit**: Q6_K
- **8-bit**: Q8_0, Q8_1, Q8_K
- **Tiled**: TQ1_0, TQ2_0

## Operator Interface

### Operator Registration

Operators are registered in `include/op_reg.h`:

```c
enum HtpOpsIndex {
  HTP_OPS_RMS_NORM_F32,
  HTP_OPS_MAT_MUL_PERMUTED_W16A32,
  HTP_OPS_FLASH_ATTN_QO_F32_KV_F16,
  // ... add new operators here
  HTP_OPS_COUNT,
};
```

### Parameter Structs

Each operator has a packed parameter struct:

```c
struct FlashAttnParams {
  struct RpcmemBufAddr o, q, k, v, mask;
  int32_t qo_len, kv_len;
  int32_t n_heads, n_kv_heads, head_dim;
} __attribute__((packed));
```

### Buffer Addressing

Buffers are passed via file descriptor + offset (for rpcmem):

```c
struct RpcmemBufAddr {
  int32_t fd;      // rpcmem file descriptor
  int32_t offset;  // offset within the allocation
} __attribute__((packed));
```

### Adding a New Operator

1. Add enum value in `include/op_reg.h`
2. Define parameter struct in `include/op_reg.h`
3. Declare function signature in `include/dsp/ops.h`
4. Implement kernel in `src/dsp/ops/your_op.c`
5. Add dispatch case in `src/dsp/op_executor.cc`
6. Add source file to `CMakeLists.txt`
7. (Optional) Add RPC interface in `include/htp_ops.idl`

## Common Patterns

### Worker Pool Usage

```c
worker_callback_t cb = { .func = my_kernel, .data = &params };
worker_synctoken_t token;
worker_pool_submit(pool, &cb, &token);
worker_pool_wait(pool, &token);
```

### HMX Tile Operations

```c
// Load and accumulate FP16 tiles
hmx_load_tiles_fp16(row_tiles, col_tiles, n_tiles);

// Output accumulator to memory
hmx_consume_accumulator_fp16(output);

// Set per-channel scales
hmx_set_output_scales(scales);
```

### VTCM Allocation

```c
vtcm_manager_t *mgr = vtcm_manager_create();
void *buf = vtcm_alloc(mgr, size, alignment);
// ... use buffer ...
vtcm_free(mgr, buf);
```

## Testing

### Running Tests

Tests run via FastRPC from the Android host:

```bash
adb push android_ReleaseG_aarch64/* /data/local/tmp/
adb push hexagon_ReleaseG_toolv87_v73/* /vendor/lib/rfsa/dsp/
adb shell /data/local/tmp/htp_ops_test
```

### Test Pattern

1. Allocate shared memory (`rpcmem_alloc`)
2. Map to DSP domain (`fastrpc_mmap`)
3. Invoke RPC with fd + offset
4. Validate against CPU reference
5. Check numerical accuracy

## Known Issues

### DSP v79 Floating-Point Errors

Compiling for v79 with newer SDK versions may cause floating-point calculation errors.

**Workaround**: Compile for v73/v75 and disable vgather-based exp:
```c
// In src/dsp/ops/flash_attn.c, set:
const bool enable_vgather_exp = false;
```

### FP16 HMX Availability

FP16 HMX is not available on mid/low-end devices. Check hardware capabilities before deployment.

## Quick Reference

### Important Files

| File | Purpose |
|------|---------|
| `include/op_reg.h` | Operator enum and parameter structs |
| `include/htp_ops.idl` | FastRPC interface definition |
| `src/dsp/op_executor.cc` | Operator dispatch (DSP side) |
| `src/dsp/ops/*.c` | Kernel implementations |
| `include/dsp/hmx_utils.h` | HMX inline assembly helpers |
| `include/dsp/hvx_math.h` | HVX math functions |

### Compiler Flags

| Flag | Purpose |
|------|---------|
| `-mhmx` | Enable HMX inline assembly |
| `DSP_ARCH=v73` | Target Hexagon v73 architecture |

### Debug Tips

- Use `HAP_debug_v2()` for DSP-side logging
- Check `adb logcat` for FastRPC errors
- Verify buffer alignment (4KB for VTCM, 128B for HVX)

## Related Resources

- [Hexagon SDK Documentation](https://developer.qualcomm.com/software/hexagon-dsp-sdk)
- [llama.cpp-npu Integration](https://github.com/haozixu/llama.cpp-npu)
- [Research Paper](https://arxiv.org/abs/2509.23324)
