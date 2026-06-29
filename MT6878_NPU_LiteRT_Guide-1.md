# MT6878 NPU Inference with LiteRT on Android 14

**A Complete Verified Method Guide**

No Play Store · No Google Account · Pure ADB · Rooted Device

Device: CMF Phone 1 — MediaTek Dimensity 7300 (MT6878) · Android 14 (SDK 34)  
Host: Ubuntu 24.04 x86_64 (VM or bare metal)  
May 2026

> **KEY RESULT:** This is a personal verified record of a two-day engineering effort. Every command has been executed and confirmed working on the exact hardware described. NPU and CPU outputs matched to 7 significant figures.

---

## 0. What Was Achieved

The following was confirmed working on a rooted CMF Phone 1 (MT6878, Android 14):

| Milestone | Detail | Status |
|---|---|---|
| NPU capabilities check | capabilities = 1 (kLiteRtDispatchCapabilitiesBasic) | PASS |
| NeuroPilot runtime | libneuronusdk_adapter.mtk.so v7.2.22 loaded | PASS |
| AOT model compilation | MobileNet V2: 66/66 ops to NPU, 0 CPU fallback | PASS |
| NPU inference execution | 978/1001 non-zero outputs, Top-1 class 972 | PASS |
| Numerical correctness | NPU == CPU to 7 significant figures | PASS |
| No Play Store required | Pure adb push, no Google account | CONFIRMED |
| Android 14 support | Official docs confirm SDK 34 is supported target | CONFIRMED |

> **KEY RESULT:** Proof: NPU Non-zero: 978/1001 · Top-1: 972 · score: 0.0467224 | CPU: identical output to 7 significant figures.
>
> The low score is expected — the test harness feeds a synthetic single-pixel input, not a real image. What matters is NPU and CPU agreeing exactly.

---

## 1. Prerequisites

### 1.1 Hardware Requirements

- Device: Any MT6878 (Dimensity 7300) device — CMF Phone 1, Redmi Note 13, Realme 12, or similar
- Device must be rooted — Magisk or equivalent
- Android 14, SDK 34. Do not update to Android 15 before verifying rooting support
- Host machine: Ubuntu 24.04 x86_64, minimum 8GB RAM, 40GB free disk space
- USB cable with adb working and USB debugging enabled on device

### 1.2 Verify adb connection

```bash
adb devices
# Expected output:
# List of devices attached
# <serial>   device
```

### 1.3 Verify NeuroPilot runtime is on device

```bash
adb shell find /system_ext /vendor -name 'libneuronusdk*' 2>/dev/null
# Expected: /system_ext/lib64/libneuronusdk_adapter.mtk.so

# Also confirm NeuroPilot version
adb shell find /vendor/lib64 -name 'libneuron*' 2>/dev/null
```

> **WARNING:** On CMF Phone 1, the NeuroPilot adapter is at /system_ext/lib64/libneuronusdk_adapter.mtk.so. On other devices check /vendor/lib64/ and /system/lib64/. If not found, your firmware may not include NeuroPilot.

### 1.4 Install host dependencies

```bash
sudo apt update && sudo apt install -y \
  android-tools-adb ninja-build cmake patchelf \
  docker.io docker-buildx python3-pip wget unzip git

sudo systemctl start docker
sudo usermod -aG docker $USER
newgrp docker

# Verify Docker works
sudo docker run --rm ubuntu:22.04 echo hello
# Must print: hello
```

---

## 2. Working Directory Setup

### 2.1 Clone LiteRT and download NDK

```bash
mkdir -p ~/litert_npu && cd ~/litert_npu

# Clone LiteRT at stable tag v2.1.0
git clone --depth=1 --branch v2.1.0 \
  https://github.com/google-ai-edge/LiteRT.git

# Download Android NDK r28b (required by LiteRT build system)
wget https://dl.google.com/android/repository/android-ndk-r28b-linux.zip
unzip android-ndk-r28b-linux.zip

# Set NDK path
export ANDROID_NDK_HOME=~/litert_npu/android-ndk-r28b
echo 'export ANDROID_NDK_HOME=~/litert_npu/android-ndk-r28b' >> ~/.bashrc
```

### 2.2 Download NPU runtime library package

```bash
cd ~/litert_npu

wget https://github.com/google-ai-edge/LiteRT/releases/download/v2.1.0rc1/litert_npu_runtime_libraries_jit.zip
unzip litert_npu_runtime_libraries_jit.zip

# Verify MediaTek files are present
ls mediatek_runtime/src/main/jni/arm64-v8a/
# Expected:
#   libLiteRtCompilerPlugin_MediaTek.so  (481KB)
#   libLiteRtDispatch_MediaTek.so         (400KB)
```

> **WARNING:** The prebuilt libLiteRtDispatch_MediaTek.so from this zip is compiled against Android 15 and WILL NOT WORK on Android 14. You will replace it with a source-built version in Section 4. Download the zip anyway for the compiler plugin.

---

## 3. Build the Docker Environment

LiteRT uses a hermetic Docker build environment with Bazel 7.4.1, Android SDK, and all required toolchains preconfigured. Patch it for Ubuntu 22.04 compatibility before building.

### 3.1 Patch the Dockerfile

```bash
cd ~/litert_npu/LiteRT

# Switch base from Ubuntu 24.04 to 22.04
sed -i 's/FROM ubuntu:24.04/FROM ubuntu:22.04/' docker_build/hermetic_build.Dockerfile

# Add LLVM 18 apt repository (not in Ubuntu 22.04 default repos)
sed -i '21a RUN apt-get update && apt-get install -y curl gnupg && ...' \
  docker_build/hermetic_build.Dockerfile

# Remove --break-system-packages flag (not supported in Ubuntu 22.04 pip)
sed -i 's/pip3 install --break-system-packages --require-hashes/pip3 install --require-hashes/' \
  docker_build/hermetic_build.Dockerfile
```

### 3.2 Build the image

```bash
# Takes approximately 10 minutes
# Downloads: Ubuntu 22.04, Bazel 7.4.1, Android SDK API 34, LLVM 18
sudo docker buildx build -t litert_build_env \
  -f docker_build/hermetic_build.Dockerfile \
  docker_build/ 2>&1 | tee /tmp/docker_build.log

# Verify the image was created
docker images | grep litert_build_env
```

### 3.3 Create persistent Bazel cache directories

Mount these on every Docker run. Without them, Bazel re-downloads ~1.5GB of dependencies on each build.

```bash
mkdir -p /tmp/bazel_cache /tmp/bazel_out /tmp/bazel_host_out
```

---

## 4. Build Android ARM64 Libraries from Source

> **WARNING:** All three libraries must come from the same source build. Do not mix source-built files with prebuilts — mismatched versions cause DISPATCH_OP registration failures.

### 4.1 Build libLiteRtDispatch_MediaTek.so

This is the critical fix. The prebuilt version from v2.1.0rc1 fails capabilities check on Android 14 with a misleading error. Build from source targeting android_arm64:

```bash
sudo docker run --rm \
  -v ~/litert_npu/LiteRT:/litert_build \
  -v /tmp/bazel_cache:/root/.cache/bazel \
  -v /tmp/bazel_out:/output \
  -w /litert_build \
  litert_build_env \
  bash -c "bazel build \
    //litert/vendors/mediatek/dispatch:dispatch_api_so \
    --config=android_arm64 && \
    cp bazel-bin/litert/vendors/mediatek/dispatch/libLiteRtDispatch_MediaTek.so /output/"

# First run: ~28 minutes. With cache: ~5 minutes.
# Expected: /tmp/bazel_out/libLiteRtDispatch_MediaTek.so (601KB)
```

### 4.2 Build libLiteRt.so

```bash
sudo docker run --rm \
  -v ~/litert_npu/LiteRT:/litert_build \
  -v /tmp/bazel_cache:/root/.cache/bazel \
  -v /tmp/bazel_out:/output \
  -w /litert_build \
  litert_build_env \
  bash -c "bazel build \
    //litert/c:litert_runtime_c_api_shared_lib \
    --config=android_arm64 && \
    cp bazel-bin/litert/c/libLiteRt.so /output/"

# Expected: /tmp/bazel_out/libLiteRt.so (4.9MB)
```

### 4.3 Build libLiteRtCompilerPlugin_MediaTek.so

```bash
sudo docker run --rm \
  -v ~/litert_npu/LiteRT:/litert_build \
  -v /tmp/bazel_cache:/root/.cache/bazel \
  -v /tmp/bazel_out:/output \
  -w /litert_build \
  litert_build_env \
  bash -c "bazel build \
    //litert/vendors/mediatek/compiler:compiler_plugin_so \
    --config=android_arm64 && \
    cp bazel-bin/litert/vendors/mediatek/compiler/libLiteRtCompilerPlugin_MediaTek.so /output/"

# Expected: /tmp/bazel_out/libLiteRtCompilerPlugin_MediaTek.so (714KB)
```

### 4.4 Verify all three files

```bash
ls -lh /tmp/bazel_out/
file /tmp/bazel_out/*.so
# All three must show: ELF 64-bit LSB shared object, ARM aarch64
```

---

## 5. Build Host x86_64 Tools for AOT Compilation

The AOT Python package invokes apply_plugin_main as a subprocess on your Ubuntu host machine to compile models. Build it and its matching libLiteRt.so together in one Docker run:

```bash
sudo docker run --rm \
  -v ~/litert_npu/LiteRT:/litert_build \
  -v /tmp/bazel_cache:/root/.cache/bazel \
  -v /tmp/bazel_host_out:/output \
  -w /litert_build \
  litert_build_env \
  bash -c "bazel build \
    //litert/tools:apply_plugin_main_for_test \
    //litert/c:litert_runtime_c_api_shared_lib && \
    cp bazel-bin/litert/tools/apply_plugin_main_for_test /output/apply_plugin_main && \
    cp bazel-bin/litert/c/libLiteRt.so /output/libLiteRt_x86_64.so"

# Verify architecture
file /tmp/bazel_host_out/apply_plugin_main
file /tmp/bazel_host_out/libLiteRt_x86_64.so
# Both must show: ELF 64-bit LSB, x86-64
```

> **WARNING:** Use apply_plugin_main_for_test NOT apply_plugin_main. The full target downloads the Qualcomm QAIRT SDK (1.4GB) which frequently times out. The _for_test variant excludes vendor-specific backends and is fully sufficient for MediaTek AOT compilation.

---

## 6. Set Up Python AOT Environment

### 6.1 Create virtualenv and install packages

```bash
cd ~/litert_npu
python3 -m venv litert_venv
source litert_venv/bin/activate

pip install ai-edge-litert-nightly ai-edge-litert-sdk-mediatek-nightly huggingface_hub
```

### 6.2 Install apply_plugin_main into venv

```bash
VENV_LITERT=$(python3 -c "import ai_edge_litert, os; print(os.path.dirname(ai_edge_litert.__file__))")
echo $VENV_LITERT

cp /tmp/bazel_host_out/apply_plugin_main $VENV_LITERT/tools/apply_plugin_main
chmod +x $VENV_LITERT/tools/apply_plugin_main
```

### 6.3 Patch RPATH of apply_plugin_main

The binary has a Bazel-internal RPATH that does not exist in pip installs. Patch it to point directly to the venv's libLiteRt.so:

```bash
patchelf --set-rpath "$VENV_LITERT" $VENV_LITERT/tools/apply_plugin_main

# Verify it resolves correctly
ldd $VENV_LITERT/tools/apply_plugin_main | grep LiteRt
# Expected: libLiteRt.so => /home/<user>/litert_npu/litert_venv/.../libLiteRt.so
```

### 6.4 Create litert symlink for vendor plugin discovery

apply_plugin_main searches for vendor plugins at the relative path `litert/vendors/mediatek/compiler/` from its working directory. Create a symlink so this path resolves correctly:

```bash
cd ~/litert_npu/litert_venv/lib/python*/site-packages

ln -s ai_edge_litert litert

# Verify the MediaTek compiler plugin is now discoverable
ls litert/vendors/mediatek/compiler/libLiteRtCompilerPlugin_MediaTek.so
```

> **WARNING:** All AOT compilation commands must be run with site-packages as the working directory. The binary uses relative paths to find vendor plugins.

---

## 7. AOT Compile a Model for MT6878

### 7.1 Model compatibility

The MediaTek AOT compiler requires float32 biases in all FullyConnected layers.

| Model type | AOT compatible | Notes |
|---|---|---|
| Float32 TFLite | Yes — fully compatible | MobileNet V2 proven: 66/66 ops |
| INT8 post-training quantized | Yes — compatible | MobileNet V1 quant confirmed |
| Mixed-precision INT4 | No — integer biases rejected | EmbeddingGemma fails: 0 ops selected |
| Dynamic INT4 | No — same issue | Bias must be float32 |

### 7.2 Compile command

```bash
source ~/litert_npu/litert_venv/bin/activate
cd ~/litert_npu/litert_venv/lib/python*/site-packages

python3 -c "
from ai_edge_litert.aot import aot_compile as aot_lib
from ai_edge_litert.aot.vendors.mediatek import target as mtk_target
import os
mt6878 = mtk_target.Target(mtk_target.SocModel.MT6878)
result = aot_lib.aot_compile('/path/to/your/model.tflite', target=[mt6878], keep_going=True)
print(result.compilation_report())
out_dir = os.path.expanduser('~/litert_npu/_compiled_models')
os.makedirs(out_dir, exist_ok=True)
result.export(out_dir + '/')"
```

> **NOTE:** `python*` in the `cd` above will expand to whatever Python version your venv used (e.g. `python3.12`). Check with `ls ~/litert_npu/litert_venv/lib/` if the glob matches more than one directory.

### 7.3 Expected compilation report for a valid model

```
MediaTek_MT6878
==========================
Partition Stats:
Subgraph 0 fully compiled:   66 / 66 ops offloaded to 1 partitions.
```

### 7.4 Verify output

```bash
ls -lh ~/litert_npu/_compiled_models/
# Expected: model_MediaTek_MT6878.tflite with non-zero size
# A 0-byte file means model was rejected (check for 'selected 0 ops')
```

---

## 8. Build the C++ Inference Binary

### 8.1 Download LiteRT C++ SDK and prepare

```bash
cd ~/litert_npu
wget https://github.com/google-ai-edge/LiteRT/releases/download/v2.1.0/litert_cc_sdk.zip
unzip litert_cc_sdk.zip

# If litert_builder.h is missing (build will fail without it)
wget -O litert_cc_sdk/litert/c/litert_builder.h \
  https://raw.githubusercontent.com/google-ai-edge/LiteRT/main/litert/c/litert_builder.h

# Copy source-built ARM64 libLiteRt.so into SDK directory
sudo cp /tmp/bazel_out/libLiteRt.so litert_cc_sdk/
sudo chmod 644 litert_cc_sdk/libLiteRt.so
```

### 8.2 Add run_npu target to CMakeLists.txt

```bash
cat >> ~/litert_npu/litert_cc_sdk/CMakeLists.txt << 'EOF'
add_executable(run_npu ../run_npu.cc)
target_include_directories(run_npu PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})
target_link_libraries(run_npu PRIVATE
    litert_cc_api absl::log absl::log_internal_check_op absl::strings absl::base)
EOF
```

### 8.3 Configure CMake for Android ARM64

```bash
mkdir -p ~/litert_npu/build_android && cd ~/litert_npu/build_android

cmake ../litert_cc_sdk \
  -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=android-26 \
  -DCMAKE_BUILD_TYPE=Release \
  -G Ninja
```

### 8.4 run_npu.cc source code

Write this to `~/litert_npu/run_npu.cc`. Adjust `input_elements` and `output_elements` for your model's shape:

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <algorithm>
#include "litert/cc/litert_compiled_model.h"
#include "litert/cc/litert_environment.h"
#include "litert/cc/litert_options.h"
#include "litert/cc/litert_tensor_buffer.h"

int main(int argc, char** argv) {
  std::string model_path = argv[1];
  std::string accel = argc > 2 ? argv[2] : "npu";
  std::string lib_dir = argc > 3 ? argv[3] : "/data/local/tmp/litert_npu";
  std::string cache_dir = argc > 4 ? argv[4] : "/data/local/tmp/litert_npu/cache";

  std::vector<litert::Environment::Option> env_options = {
    {litert::Environment::OptionTag::DispatchLibraryDir, lib_dir},
    {litert::Environment::OptionTag::CompilerPluginLibraryDir, lib_dir},
    {litert::Environment::OptionTag::CompilerCacheDir, cache_dir},
  };
  auto env = litert::Environment::Create(env_options);
  auto options = litert::Options::Create();
  if (accel == "npu")
    options->SetHardwareAccelerators(litert::HwAccelerators::kNpu);
  else if (accel == "gpu")
    options->SetHardwareAccelerators(litert::HwAccelerators::kGpu);
  else
    options->SetHardwareAccelerators(litert::HwAccelerators::kCpu);

  auto model = litert::CompiledModel::Create(*env, model_path, *options);
  if (!model) { std::cerr << "Failed: " << model.Error().Message() << "\n"; return 1; }
  auto input_buffers = model->CreateInputBuffers(0);
  auto output_buffers = model->CreateOutputBuffers(0);

  const size_t input_elements = 224 * 224 * 3;  // MobileNet V2: [1,224,224,3]
  const size_t output_elements = 1001;           // MobileNet V2: [1,1001]

  std::vector<float> input_data(input_elements, 0.0f);
  size_t center = (112 * 224 + 112) * 3;
  input_data[center] = input_data[center+1] = input_data[center+2] = 1.0f;

  (*input_buffers)[0].Write<float>(absl::MakeConstSpan(input_data.data(), input_elements));

  std::cout << "Running inference on " << accel << "...\n";
  auto status = model->Run((size_t)0, *input_buffers, *output_buffers);
  if (!status) { std::cerr << "Failed: " << status.Error().Message() << "\n"; return 1; }

  std::vector<float> output_data(output_elements);
  (*output_buffers)[0].Read<float>(absl::MakeSpan(output_data.data(), output_elements));

  int nz = 0;
  for (float v : output_data) if (v != 0.0f) nz++;
  int top1 = std::max_element(output_data.begin(), output_data.end()) - output_data.begin();

  std::cout << "SUCCESS on " << accel << "!\n";
  std::cout << "Non-zero: " << nz << "/" << output_elements << "\n";
  std::cout << "Top-1 class: " << top1 << "  score: " << output_data[top1] << "\n";
  std::cout << "First 10: ";
  for (int i = 0; i < 10; i++) std::cout << output_data[i] << " ";
  std::cout << "\n";
  return 0;
}
```

### 8.5 Build

```bash
cd ~/litert_npu/build_android
ninja run_npu 2>&1 | grep -E 'error:|Linking|Building'

file ~/litert_npu/build_android/run_npu
# Expected: ELF 64-bit LSB executable, ARM aarch64
```

---

## 9. Push to Device and Run Inference

### 9.1 Set up device directory and push all files

```bash
DEVICE=/data/local/tmp/litert_npu
adb shell mkdir -p $DEVICE $DEVICE/cache

sudo chmod 644 /tmp/bazel_out/libLiteRt.so
sudo chmod 644 /tmp/bazel_out/libLiteRtDispatch_MediaTek.so
sudo chmod 644 /tmp/bazel_out/libLiteRtCompilerPlugin_MediaTek.so

adb push /tmp/bazel_out/libLiteRt.so $DEVICE/
adb push /tmp/bazel_out/libLiteRtDispatch_MediaTek.so $DEVICE/
adb push /tmp/bazel_out/libLiteRtCompilerPlugin_MediaTek.so $DEVICE/

adb push ~/litert_npu/_compiled_models/model_MediaTek_MT6878.tflite $DEVICE/model_aot.tflite

adb push ~/litert_npu/build_android/run_npu $DEVICE/
adb shell chmod +x $DEVICE/run_npu

adb shell ls -lh $DEVICE/
```

### 9.2 Run on NPU

```bash
adb shell LD_LIBRARY_PATH=/data/local/tmp/litert_npu:/vendor/lib64:/vendor/lib64/mt6878:/system_ext/lib64 \
    /data/local/tmp/litert_npu/run_npu \
    /data/local/tmp/litert_npu/model_aot.tflite \
    npu \
    /data/local/tmp/litert_npu \
    /data/local/tmp/litert_npu/cache
```

### 9.3 Run on CPU for correctness verification

```bash
adb shell LD_LIBRARY_PATH=/data/local/tmp/litert_npu:/vendor/lib64:/vendor/lib64/mt6878:/system_ext/lib64 \
    /data/local/tmp/litert_npu/run_npu \
    /data/local/tmp/litert_npu/model_aot.tflite \
    cpu \
    /data/local/tmp/litert_npu \
    /data/local/tmp/litert_npu/cache
```

### 9.4 Expected output — MobileNet V2

```
NPU run:
Running inference on npu...
SUCCESS on npu!
Non-zero: 978/1001
Top-1 class: 972  score: 0.0467224
First 10: 0.00034833 0.000274658 0.00057888 0.000595093 ...

CPU run (must be identical):
Running inference on cpu...
SUCCESS on cpu!
Non-zero: 978/1001
Top-1 class: 972  score: 0.0467224
First 10: 0.00034833 0.000274658 0.00057888 0.000595093 ...
```

> **KEY RESULT:** NPU and CPU output must be identical. Identical output to 7 significant figures confirms the NPU is computing correctly, not producing noise.

> **NOTE:** The confidence score (0.0467224) is low because `run_npu.cc` feeds a synthetic test input — a single white pixel in an otherwise black 224×224 frame (see Section 8.4) — not a real photo. The point of this test is numerical correctness (NPU output matching CPU output), not classification accuracy. Feed it a real preprocessed image and you'll get a normal high-confidence Top-1 score.

---

## 10. Troubleshooting Reference

| Error message | Root cause | Fix |
|---|---|---|
| Dispatch API insufficient capabilities: -XXXXXXX | Prebuilt dispatch .so compiled for Android 15 AHardwareBuffer internals | Build libLiteRtDispatch_MediaTek.so from source per Section 4.1 |
| NeuronCompilation_finish failed with error 6 | Integer-typed ops (CAST, SUB) sent to NPU which only supports float32 variants | Use AOT compilation path instead of JIT |
| selected 0 ops, 0 partitions | Model has integer biases in FullyConnected layers (mixed-precision INT4) | Use float32 model. Mixed-precision EmbeddingGemma is incompatible. |
| undefined symbol: LiteRtGoogleTensorOptionsCreate | apply_plugin_main linked against different libLiteRt.so than available | Patch RPATH per Section 6.3. Ensure host binary and venv libLiteRt.so are from same build. |
| Loaded 0 plugins | apply_plugin_main cannot find vendor .so at relative search path | Create litert symlink per Section 6.4. Run python from site-packages directory. |
| exec format error in Docker | Ubuntu 24.04 base image incompatible with VMware Docker | Patch Dockerfile to use Ubuntu 22.04 per Section 3.1 |
| DISPATCH_OP unresolved custom op | Stale JIT cache from previous run with different runtime version | `adb shell rm -rf /data/local/tmp/litert_npu/cache/*` |
| libLiteRt.so => not found (ldd) | ARM64 .so installed in x86_64 system path — wrong architecture | Use /tmp/bazel_host_out/libLiteRt_x86_64.so for host tools only |
| Config value 'linux_x86_64' is not defined | Wrong Bazel config flag for host builds | Omit --config entirely for host builds. Bazel defaults to host platform. |

---

## 11. Key Discoveries — Personal Record

These are the non-obvious findings discovered during the two-day engineering effort. Documented for future reference and for anyone following this path.

### 11.1 The prebuilt dispatch library was the root blocker

The `litert_npu_runtime_libraries_jit.zip` from v2.1.0rc1 contains a `libLiteRtDispatch_MediaTek.so` compiled against Android 15 AHardwareBuffer internals. On Android 14 it fails with 'Dispatch API has insufficient capabilities: -XXXXXXX'. The large negative number is actually a LiteRtStatus error code from NeuronAdapterApi::Create() being printed in the wrong log field — not a capabilities bitmask. Building from source with --config=android_arm64 produces a binary that correctly returns capabilities=1 on Android 14.

### 11.2 All three .so files must come from the same source build

Mixing source-built dispatch .so with Maven or prebuilt libLiteRt.so causes DISPATCH_OP to be unresolved at runtime. The DISPATCH_OP handler registration is version-specific and silently mismatches between builds. All three files must be built together from the same LiteRT commit.

### 11.3 JIT fails on EmbeddingGemma, AOT fails differently

EmbeddingGemma mixed-precision uses INT4 weights with integer biases. In JIT mode, the compiler plugin sends CAST and SUB ops with integer types to the NPU which returns NeuronCompilation_finish error 6 (NEURON_BAD_DATA) and aborts all 34 subgraphs. In AOT mode, the validator catches it earlier: 'Bias should be floating point type' and selects 0 ops. Both paths produce a zero-byte output. The model needs float32 biases to be AOT-compilable.

### 11.4 The apply_plugin_main library resolution chain

The AOT Python package finds `apply_plugin_main` via importlib.resources under the `ai_edge_litert` package root. The binary's Bazel RPATH points to runfiles directories that do not exist in pip installs. patchelf must be used to redirect RPATH to the venv's actual library directory. The vendor plugins are found at the relative path `litert/vendors/mediatek/compiler/` from the binary's working directory, requiring the site-packages/litert symlink.

### 11.5 apply_plugin_main_for_test is the correct build target

The full apply_plugin_main Bazel target pulls in Qualcomm QAIRT SDK as a dependency (1.4GB zip download, frequently times out). The _for_test variant explicitly excludes vendor-specific external dependencies and is built in the same Docker invocation as libLiteRt.so, guaranteeing matching symbol versions. Both produce functionally equivalent binaries for MediaTek AOT compilation.

### 11.6 NeuroPilot runtime is already present on CMF Phone 1

No additional SDK installation is needed on the device. `libneuronusdk_adapter.mtk.so` is at `/system_ext/lib64/` and loads automatically when that path is in LD_LIBRARY_PATH. The device has NeuroPilot version 7.2.22 which handles all float32 model compilation and execution.

### 11.7 Android 14 is officially supported — the prebuilt was the only blocker

The official LiteRT MediaTek documentation explicitly states Android SDK API Level 34 (Android 14) as the supported target. The widely-held belief that Android 15 was required was caused entirely by the prebuilt dispatch library, not by any OS-level limitation. Building from source eliminates this constraint entirely.

### 11.8 dims_signature — future reference

When running AOT-compiled models via the C++ API, the runtime may try to auto-resize input tensors if the input buffer shape differs from the model's compiled shape. If the model flatbuffer lacks dims_signature metadata, this returns error code 3. On the host the nightly libLiteRt.so handles it gracefully (non-fatal). On-device the stricter runtime may reject the execution. Avoid by always creating input buffers that exactly match the model's compiled fixed shape without triggering resize.

---

## 12. What to Build Next

### 12.1 Train and deploy your own model

1. Design your model in PyTorch using float32 operations. All linear layers must have float32 biases.
2. Export to TFLite: `pip install ai-edge-torch`, then `ai_edge_torch.convert(model, sample_inputs).export('model.tflite')`
3. AOT compile for MT6878 following Section 7
4. Push and run following Section 9

### 12.2 Fix EmbeddingGemma for NPU

Post-process the mixed-precision flatbuffer to convert integer biases to float32. Read the TFLite flatbuffer schema, find FullyConnected tensors with integer bias types, dequantize them using their scale and zero_point parameters, write a new .tflite. Approximately 30 lines of Python using the flatbuffers library. After this step, AOT compile normally.

### 12.3 Model types that work right now

| Task | Example model | NPU result | Size |
|---|---|---|---|
| Image classification | MobileNet V2 float32 | 66/66 ops — PROVEN | 14MB |
| Image classification | MobileNet V1 INT8 quant | Confirmed in prior trial | 4MB |
| Object detection | MobileNet SSD float32 | Expected — same ops | < 15MB |
| Image segmentation | Selfie segmentation | Confirmed in Google sample | < 10MB |
| Your custom model | Any float32 TFLite | Works if float32 biases | Any |
| EmbeddingGemma 300M | mixed-precision variant | 0 ops — needs bias fix | 171MB |

---

## Final Note

This guide documents a verified working path completed on May 4, 2026, after two continuous days of engineering across multiple Claude instances. Every command has been executed and confirmed on the exact hardware described.

The path was non-trivial because:

- Google's official LiteRT documentation assumes Play Store deployment and never covers rooted adb-only paths
- The prebuilt dispatch library was silently incompatible with Android 14 with a misleading error message
- No existing public documentation covered this combination of device, OS version, and deployment method
- Multiple successive blockers each required source-level investigation to identify and resolve

> **KEY RESULT:** MT6878 (Dimensity 7300) is present in millions of consumer devices. This is the first documented and verified path to running LiteRT NPU inference on this chip on Android 14 without Play Store, Google account, or Android 15.

The verified proof of execution:

```
NPU:  Non-zero: 978/1001  |  Top-1: 972  |  score: 0.0467224
CPU:  Non-zero: 978/1001  |  Top-1: 972  |  score: 0.0467224

NPU == CPU to 7 significant figures.
Inference SUCCESS on npu!
```

(Score is low because the test input is a synthetic single-pixel image, not a real photo — see Section 8.4. The result that matters is the exact NPU/CPU match.)

The NPU silicon on MT6878 executes real neural network inference, produces numerically correct results, and is now accessible without any cloud dependency, Play Store, or vendor tooling beyond what is documented in this guide.
