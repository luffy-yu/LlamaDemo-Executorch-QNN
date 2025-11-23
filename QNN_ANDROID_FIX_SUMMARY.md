# QNN Android LlamaDemo Fix Summary

This document summarizes all the changes required to make the Android LlamaDemo work with QNN (Qualcomm Neural Network) backend.

## Problem Statement

The `qnn_llama_runner` command-line tool worked successfully, but the Android LlamaDemo app failed with "Error 18: InvalidArgument" when loading QNN models. After fixing that, we encountered "Error 1: Internal" due to QNN SDK version mismatch.

## Root Causes Identified

1. **Parameter Order Bug**: The QNN Runner constructor parameters were in the wrong order
2. **Missing QNN Configuration**: Android app wasn't passing required QNN parameters (decoder_model_version, kv_updater, eval_mode)
3. **Gradle Build Configuration**: Maven dependency overriding local AAR changes
4. **QNN SDK Version Mismatch**: APK bundled with QNN 2.33.0 libraries while model was compiled with QNN 2.37.0

## Required Changes

### 1. JNI Layer Changes (`/executorch/extension/android/jni/jni_layer_llama.cpp`)

#### Change 1.1: Fixed QNN Runner Constructor Parameter Order

**Location**: Lines ~310-319 (uint8_t) and ~337-346 (uint16_t)

**Issue**: The parameters `performance_output_path` and `dump_logits_path` were swapped.

**Correct Order**:
```cpp
runner_ = std::make_unique<example::Runner<uint8_t>>(
    std::move(module),
    decoder_model,                     // const std::string& decoder_model_version
    model_path->toStdString(),         // const std::string& model_path
    tokenizer_path->toStdString(),     // const std::string& tokenizer_path
    "",                                // const std::string& performance_output_path (param 5)
    "",                                // const std::string& dump_logits_path (param 6)
    temperature_,                      // float temperature
    eval_mode,                         // int eval_mode
    kv_updater                         // const std::string& kv_updater
);
```

**Reference**: Check `examples/qualcomm/oss_scripts/llama/runner/runner.h` for the correct constructor signature.

#### Change 1.2: Modified Constructor to Parse QNN Config from data_path

**Location**: Lines ~200-260

**Change**: Modified the QNN initialization to parse configuration from the `data_path` parameter:

```cpp
#if defined(EXECUTORCH_BUILD_QNN)
    } else if (model_type_category == MODEL_TYPE_QNN_LLAMA) {
      // Parse config string: "decoder_model_version:qwen3;kv_updater:SmartMask;eval_mode:1"
      std::string decoder_model = "qwen3";  // default
      std::string kv_updater = "SmartMask";  // default
      int eval_mode = 1;  // default (Hybrid mode)
      
      if (data_path != nullptr) {
        std::string config = data_path->toStdString();
        // Parse semicolon-separated key:value pairs
        // Extract decoder_model_version, kv_updater, eval_mode
      }
      
      // Create Module without data files
      std::vector<std::string> empty_data_files;
      module = std::make_unique<executorch::extension::Module>(
          model_path->toStdString().c_str(),
          empty_data_files,
          executorch::extension::Module::LoadMode::MmapUseMlockIgnoreErrors);
      
      // Detect kv_bitwidth and create appropriate Runner
      // ...
#endif
```

### 2. Android App Changes (`/executorch-examples/llm/android/LlamaDemo/app/src/main/java/com/example/executorchllamademo/MainActivity.java`)

#### Change 2.1: Added QNN Configuration Builder

**Location**: Added new method around line ~420

```java
private String buildQnnConfigString(ModelType modelType) {
    switch (modelType) {
        case QWEN_3:
            // Config for QWEN_3: Hybrid mode with SmartMask KV cache updater
            return "decoder_model_version:qwen3;kv_updater:SmartMask;eval_mode:1";
        // Add other model types as needed
        default:
            return "";
    }
}
```

#### Change 2.2: Modified Model Loading to Pass Config

**Location**: Around line ~380 in `loadModule()` method

```java
String dataPath = "";
if (mBackendType == BackendType.QUALCOMM) {
    dataPath = buildQnnConfigString(mModelType);
    Log.d(TAG, "QNN Config: " + dataPath);
}

mModule = new LlmModule(
    mModelFilePath,
    mTokenizerFilePath,
    dataPath,  // Pass config string instead of empty string
    temperature,
    modelCategory
);
```

### 3. Gradle Configuration Changes (`/executorch-examples/llm/android/LlamaDemo/app/build.gradle.kts`)

#### Change 3.1: Updated QNN Runtime Version

**Location**: Line ~71

**Before**:
```kotlin
implementation("com.qualcomm.qti:qnn-runtime:2.33.0")
```

**After**:
```kotlin
implementation("com.qualcomm.qti:qnn-runtime:2.37.0")
```

**Important**: The QNN runtime version MUST match the QNN SDK version used to compile the model. Check your model's QNN SDK version and update accordingly.

#### Change 3.2: Use Local AAR

**Location**: Build command

**Ensure** you build with the `-PuseLocalAar=true` flag to use your locally built AAR instead of the Maven dependency:

```bash
./gradlew clean assembleDebug -PuseLocalAar=true
```

Without this flag, Gradle will pull `org.pytorch:executorch-android:1.0.0` from Maven, ignoring your local changes.

### 4. Build Process

#### Step 1: Rebuild ExecuTorch AAR

```bash
cd /path/to/executorch
# Activate conda environment and set QNN_SDK_ROOT
eval "$(conda shell.bash hook)"
conda activate executorch
source ~/.bash_profile  # Ensure QNN_SDK_ROOT is set

# Build the AAR
./scripts/build_android_library.sh
```

#### Step 2: Copy AAR to Android Project

```bash
cp /path/to/executorch/aar-out/executorch.aar \
   /path/to/LlamaDemo/app/libs/executorch.aar
```

#### Step 3: Build Android APK

```bash
cd /path/to/LlamaDemo
./gradlew clean assembleDebug -PuseLocalAar=true
```

#### Step 4: Install on Device

```bash
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

## QNN Model Compilation Requirements

When compiling models for QNN, ensure you use the same parameters that the Android app will pass:

```bash
./qnn_llama_runner \
  --decoder_model_version qwen3 \
  --tokenizer_path tokenizer.json \
  --model_path model_SM8550.pte \
  --prompt 'test prompt' \
  --seq_len 512 \
  --kv_updater SmartMask \
  --eval_mode 1 \
  --temperature 0.8
```

Key parameters:
- `--decoder_model_version qwen3`: Model architecture type
- `--kv_updater SmartMask`: KV cache update strategy
- `--eval_mode 1`: Hybrid mode (0=KVCached, 1=Hybrid, 2=LookaheadDecoding)

## Verification Steps

### 1. Verify AAR Contains Your Changes

```bash
# Check for debug strings in the AAR
unzip -p /path/to/executorch.aar jni/arm64-v8a/libexecutorch.so | \
  strings | grep "ZZTEST"
```

If found, your changes are in the AAR.

### 2. Verify APK Contains Correct Libraries

```bash
# Check QNN library version in APK
cd /path/to/LlamaDemo
unzip -l app/build/outputs/apk/debug/app-debug.apk | grep "libQnnHtp.so"
```

Expected size for QNN 2.37.0: ~2,465,440 bytes

### 3. Monitor Logs During Model Loading

```bash
adb logcat -c
adb logcat | grep -E "(ExecuTorch|ETLogging)"
```

Expected success logs:
```
[Load] runner_->load() returned: 0
[Load] Model loaded successfully!
Model loaded time: XXXX ms
```

## Common Issues and Solutions

### Issue 1: Error 18 (InvalidArgument)

**Cause**: Wrong parameter order in Runner constructor or missing QNN config

**Solution**: Apply changes 1.1 and 1.2 from this document

### Issue 2: Error 1 (Internal) with QNN API Version Mismatch

**Symptoms**:
```
W [Qnn ExecuTorch]: Qnn API version X.XX.X is mismatched
E [Qnn ExecuTorch]: Using newer context binary on old SDK
E [Qnn ExecuTorch]: Can't create context from binary. Error 5000
```

**Cause**: Model compiled with QNN SDK version X but APK uses QNN runtime version Y

**Solution**: 
1. Update `build.gradle.kts` with matching QNN runtime version (change 3.1)
2. Or recompile model with matching QNN SDK version

### Issue 3: Native Code Changes Not Applied

**Symptoms**: Debug logs don't appear, behavior doesn't change

**Cause**: Gradle using Maven dependency instead of local AAR

**Solution**: Always build with `-PuseLocalAar=true` flag

### Issue 4: ZZTEST Logs Not Appearing

**Cause**: Wrong logging tag filter

**Solution**: QNN uses "ExecuTorch" tag, not "ETLogging":
```bash
adb logcat | grep "ExecuTorch"
```

## Environment Requirements

- **QNN SDK**: Version 2.37.0 (or version matching your model)
- **Android NDK**: r25c or compatible
- **Gradle**: 8.10+
- **Conda Environment**: executorch environment activated
- **Environment Variables**: `QNN_SDK_ROOT` must be set in `~/.bash_profile`

## Success Criteria

✅ Model loads without errors (return code 0)
✅ Load time reasonable (~1-2 seconds)
✅ No QNN API version mismatch warnings
✅ Can generate text responses
✅ Inference runs on Hexagon DSP (HTP backend)

## Additional Notes

### KV Cache Updaters
- **SmartMask**: Optimized for hybrid mode, recommended for most models
- **Full**: Updates entire KV cache every iteration
- **Sliding**: For models with sliding window attention

### Eval Modes
- **0 (KVCached)**: Single method, uses KV cache
- **1 (Hybrid)**: Separate prefill and decode, better for long sequences (recommended)
- **2 (LookaheadDecoding)**: Advanced speculative decoding

### File Locations on Device
```
/data/local/tmp/llama/
  ├── model_SM8550.pte      # QNN model file
  └── tokenizer.json        # Tokenizer file
```

Ensure files have read permissions for the app's user context.

## References

- QNN Runner implementation: `examples/qualcomm/oss_scripts/llama/runner/runner.cpp`
- QNN Runner header: `examples/qualcomm/oss_scripts/llama/runner/runner.h`
- Android JNI layer: `extension/android/jni/jni_layer_llama.cpp`
- Build script: `scripts/build_android_library.sh`

## Version Compatibility Matrix

| QNN SDK | QNN Runtime (Gradle) | Model Compatibility |
|---------|---------------------|---------------------|
| 2.37.0  | 2.37.0             | ✅ Match required   |
| 2.33.0  | 2.33.0             | ✅ Match required   |
| 2.29.0  | 2.29.0             | ✅ Match required   |
| 2.37.0  | 2.33.0             | ❌ Error 1 (Internal) |

**Critical**: Always ensure QNN SDK version (used for model compilation) matches QNN Runtime version (in build.gradle.kts).

---

**Document Version**: 1.0  
**Last Updated**: November 23, 2025  
**Status**: ✅ Verified Working
