# Build, Deploy & Test Guide

This guide documents the complete workflow for modifying code in htp-ops-lib, rebuilding, deploying to a device, and running tests.

---

## Project Overview: What to Rebuild After Editing

The project compiles into two separate shared libraries targeting different processors. Understanding which library a file belongs to determines what needs to be rebuilt.

```
htp-ops-lib/
├── src/host/          → Android build only (libhtp_ops.so + htp_ops_test)
│   ├── session.c          # DSP session open/close via FastRPC
│   ├── op_export.c        # RPC call wrappers for each operator
│   └── test.c             # Test harness (htp_ops_test binary)
│
├── src/dsp/           → Hexagon DSP build only (libhtp_ops_skel.so)
│   ├── ops/
│   │   ├── flash_attn.c       # Flash Attention kernel
│   │   ├── flash_attn_sp_hdim.c
│   │   ├── mat_mul.c          # Matrix multiplication kernel
│   │   ├── rms_norm.c         # RMS normalization kernel
│   │   ├── mm_benchmark.c     # GEMM benchmark
│   │   └── precompute_table.c # Lookup table generation
│   ├── commu.c            # FastRPC entry points (parameter unpacking)
│   ├── hmx_mgr.c          # HMX accelerator acquisition/release
│   ├── worker_pool.c      # Thread pool for multi-core execution
│   ├── op_executor.cc     # Operator dispatch
│   ├── vtcm_mgr.cc        # VTCM on-chip memory management
│   ├── mmap_mgr.cc        # Memory mapping manager
│   ├── op_tests.cc        # DSP-side operator tests
│   └── power.c            # Power management
│
└── include/           → Both builds (triggers rebuild of both if changed)
    ├── htp_ops.idl        # FastRPC interface: changing this rebuilds both
    ├── op_reg.h           # Operator enum + parameter structs: rebuilds both
    ├── message.h          # Message protocol: rebuilds both
    └── dsp/               # DSP-specific headers: Hexagon build only
```

### Quick Rebuild Reference

| Modified file(s) | Rebuild required |
|---|---|
| `src/host/*.c` | Android only |
| `src/dsp/**/*.c`, `src/dsp/**/*.cc` | Hexagon only |
| `include/dsp/*.h` | Hexagon only |
| `include/htp_ops.idl` | **Both** |
| `include/op_reg.h` | **Both** |
| `include/message.h` | **Both** |
| `CMakeLists.txt` | **Both** |

---

## 1. Environment Setup

```bash
source $HEXAGON_SDK_ROOT/setup_sdk_env.source
```

This must be run once per shell session before any build commands.

---

## 2. Build

### Android CPU build (produces `libhtp_ops.so` and `htp_ops_test`)

```bash
build_cmake android
```

Output directory: `android_ReleaseG_aarch64/`
```
android_ReleaseG_aarch64/
├── libhtp_ops.so      # FastRPC stub (runs on CPU)
└── htp_ops_test       # Test executable (runs on CPU)
```

### Hexagon DSP build (produces `libhtp_ops_skel.so`)

```bash
build_cmake hexagon DSP_ARCH=v75
```

Output directory: `hexagon_ReleaseG_toolv87_v75/`
```
hexagon_ReleaseG_toolv87_v75/
└── libhtp_ops_skel.so # FastRPC skeleton (runs on DSP/NPU)
```

---

## 3. Deploy to Device

```bash
# Verify device is connected
adb devices

# Create target directories on device (if not already present)
adb shell mkdir -p /data/local/tmp/scaling_llm/lib

# Push CPU library and test binary
adb push android_ReleaseG_aarch64/libhtp_ops.so /data/local/tmp/scaling_llm/lib/
adb push android_ReleaseG_aarch64/htp_ops_test /data/local/tmp/scaling_llm/

# Push DSP skeleton library
adb push hexagon_ReleaseG_toolv87_v75/libhtp_ops_skel.so /data/local/tmp/scaling_llm/lib/
```

> **Note**: The DSP loads `libhtp_ops_skel.so` at runtime via FastRPC. Both the CPU stub (`libhtp_ops.so`) and DSP skeleton (`libhtp_ops_skel.so`) must be present in `ADSP_LIBRARY_PATH` for RPC to succeed.

---

## 4. Run Tests

```bash
adb shell

# On device:
chmod +x /data/local/tmp/scaling_llm/htp_ops_test

export LD_LIBRARY_PATH=/data/local/tmp/scaling_llm/lib:$LD_LIBRARY_PATH
export ADSP_LIBRARY_PATH="/data/local/tmp/scaling_llm/lib;/vendor/lib/rfsa/dsp"

cd /data/local/tmp/scaling_llm
./htp_ops_test
```

---

## 5. Monitor DSP Logs (Separate Terminal)

Open a second terminal and run the following to monitor DSP-side FARF logs and relevant system messages in real time:

```bash
adb logcat | grep -iE "hmx|ctx_id|mat_mul|setup|init_backend|FARF"
```

### Enable FARF Logging

FARF (Fast And Reliable Framework) logging from the DSP is disabled by default. To enable it, create a `.farf` file named after the test executable:

```bash
adb shell "echo 0x1f > /data/local/tmp/scaling_llm/htp_ops_test.farf"
```

The `0x1f` bitmask enables all FARF log levels (LOW, MED, HIGH, ERROR, FATAL). After pushing this file, rerun the test — DSP logs will appear in `adb logcat`.

---

## 6. Full Rebuild + Deploy Script (One-Shot)

For convenience when iterating on DSP-side code:

```bash
#!/bin/bash
set -e

source $HEXAGON_SDK_ROOT/setup_sdk_env.source

# Build
build_cmake android
build_cmake hexagon DSP_ARCH=v75

# Deploy
adb shell mkdir -p /data/local/tmp/scaling_llm/lib
adb push android_ReleaseG_aarch64/libhtp_ops.so       /data/local/tmp/scaling_llm/lib/
adb push android_ReleaseG_aarch64/htp_ops_test         /data/local/tmp/scaling_llm/
adb push hexagon_ReleaseG_toolv87_v75/libhtp_ops_skel.so /data/local/tmp/scaling_llm/lib/

# Run
adb shell "
  chmod +x /data/local/tmp/scaling_llm/htp_ops_test
  export LD_LIBRARY_PATH=/data/local/tmp/scaling_llm/lib:\$LD_LIBRARY_PATH
  export ADSP_LIBRARY_PATH='/data/local/tmp/scaling_llm/lib;/vendor/lib/rfsa/dsp'
  cd /data/local/tmp/scaling_llm
  ./htp_ops_test
"
```

---

## Known Issues

### DMA Transfer Fails Silently (Snapdragon 8 Gen 3)

`dma_issue_load_from_ddr` may return success but not transfer data, causing HMX outputs to be all zeros (~3.6e-12). Workaround: disable DMA and use manual HVX copy.

In `src/dsp/ops/mat_mul.c`, `transfer_permuted_weight_chunk_fp16`:
```c
const bool use_dma = false;  // set to false if HMX output is all zeros
```

### DSP v79 Floating-Point Errors

Avoid `DSP_ARCH=v79` with newer SDK versions — it causes silent FP calculation errors. Use `v75` (or `v73`) instead. If using `v73`/`v75` and flash attention is broken:
```c
// In src/dsp/ops/flash_attn.c:
const bool enable_vgather_exp = false;
```
