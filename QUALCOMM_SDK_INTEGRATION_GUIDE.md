# Qualcomm SDK Integration Guide for Android Apps

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Integration Approaches](#integration-approaches)
4. [Approach 1: QNN TFLite Delegate (Computer Vision)](#approach-1-qnn-tflite-delegate-computer-vision)
5. [Approach 2: Genie SDK (Large Language Models)](#approach-2-genie-sdk-large-language-models)
6. [Best Practices](#best-practices)
7. [Troubleshooting](#troubleshooting)

---

## Overview

This project demonstrates two different approaches to integrate Qualcomm's AI acceleration capabilities into Android applications:

1. **QNN TFLite Delegate**: For Computer Vision tasks (Image Classification, Object Detection, Semantic Segmentation, Super Resolution)
2. **Genie SDK**: For Large Language Model inference (Chat applications)

Both approaches leverage Qualcomm's QAIRT (Qualcomm AI Runtime) SDK to achieve hardware acceleration on Snapdragon chipsets using the NPU (Neural Processing Unit), GPU, or CPU.

### Supported Hardware
- **Snapdragon 8 Elite** (DSP arch v79)
- **Snapdragon 8 Gen 3** (DSP arch v75)
- **Snapdragon 8 Gen 2** (DSP arch v73)
- **Snapdragon 8 Gen 1 and newer** (for FP16 compute support)

### Key Benefits
- Hardware-accelerated AI inference
- Automatic fallback between NPU → GPU → CPU
- Optimized performance and power efficiency
- Cached model compilation for faster subsequent loads

---

## Prerequisites

### Required Software
- **Android Studio**: Arctic Fox or newer
- **Android NDK**: Version compatible with CMake 3.22.1+ (for Genie SDK apps)
- **Gradle**: 7.0 or higher
- **Java**: JDK 11
- **Android SDK**:
  - minSdk: 30 (Android 11)
  - targetSdk: 34 (Android 14)
  - compileSdk: 34

### Required Dependencies
```gradle
// In gradle.properties
qnnVersion=2.37.0

// For TFLite Delegate Apps (Computer Vision)
dependencies {
    implementation 'org.tensorflow:tensorflow-lite:2.16.1'
    implementation 'org.tensorflow:tensorflow-lite-support:0.4.4'
    implementation 'org.tensorflow:tensorflow-lite-gpu:2.16.1'
    implementation 'org.tensorflow:tensorflow-lite-gpu-api:2.16.1'
    implementation 'org.tensorflow:tensorflow-lite-gpu-delegate-plugin:0.4.4'
    implementation "com.qualcomm.qti:qnn-runtime:$qnnVersion"
    implementation "com.qualcomm.qti:qnn-litert-delegate:$qnnVersion"
}
```

### For Genie SDK Apps
Download **QAIRT SDK** from [Qualcomm Package Manager](https://qpm.qualcomm.com/#/main/tools/details/Qualcomm_AI_Runtime_SDK)

---

## Integration Approaches

### Decision Matrix

| Feature | QNN TFLite Delegate | Genie SDK |
|---------|---------------------|-----------|
| **Use Case** | Computer Vision (CV) | Large Language Models (LLM) |
| **Model Format** | TFLite (.tflite) | QNN Context Binary (.bin) |
| **Language** | Java/Kotlin | C++ with JNI |
| **Setup Complexity** | Low (Maven dependencies) | Medium (Local SDK required) |
| **Examples** | ImageClassification, ObjectDetection | ChatApp |

---

## Approach 1: QNN TFLite Delegate (Computer Vision)

### Architecture Overview
```
Android App (Java)
    ↓
TFLite Interpreter
    ↓
TFLite Delegate Layer
    ├─→ QNN NPU Delegate (Primary)
    ├─→ GPUv2 Delegate (Fallback)
    └─→ XNNPack CPU (Final Fallback)
```

### Step 1: Add Dependencies

In `app/build.gradle`:

```gradle
plugins {
    id 'com.android.application'
}

android {
    compileSdk 34

    defaultConfig {
        applicationId "com.example.myapp"
        minSdk 30
        targetSdk 34

        // Configure model asset names
        resValue("string", "tfLiteModelAsset", "my_model.tflite")
    }

    namespace 'com.example.myapp'
}

dependencies {
    // TensorFlow Lite dependencies
    api 'org.tensorflow:tensorflow-lite:2.16.1'
    api 'org.tensorflow:tensorflow-lite-support:0.4.4'
    implementation 'org.tensorflow:tensorflow-lite-gpu:2.16.1'
    implementation 'org.tensorflow:tensorflow-lite-gpu-api:2.16.1'
    implementation 'org.tensorflow:tensorflow-lite-gpu-delegate-plugin:0.4.4'

    // Qualcomm QNN dependencies
    implementation "com.qualcomm.qti:qnn-runtime:2.37.0"
    implementation "com.qualcomm.qti:qnn-litert-delegate:2.37.0"

    // Android libraries
    implementation 'androidx.appcompat:appcompat:1.7.0'
    implementation 'com.google.android.material:material:1.12.0'
}
```

### Step 2: Configure AndroidManifest.xml

Add native library declarations:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application>
        <!-- DSP/NPU support -->
        <uses-native-library
            android:name="libcdsprpc.so"
            android:required="false" />

        <!-- GPU support (optional) -->
        <uses-native-library
            android:name="libOpenCL.so"
            android:required="false"/>
    </application>
</manifest>
```

### Step 3: Copy TFLiteHelpers Utility Class

Copy the `TFLiteHelpers.java` utility from `/apps/android/tflite_helpers/` to your project.

**Location**: `apps/android/tflite_helpers/TFLiteHelpers.java`

This utility provides:
- Delegate creation (QNN NPU, GPUv2)
- Interpreter instantiation with fallback logic
- Model loading with MD5 hash computation
- Capability checking for hardware support

### Step 4: Implement Model Loading

```java
import android.content.Context;
import android.util.Pair;
import com.quicinc.tflite.TFLiteHelpers;
import com.quicinc.tflite.AIHubDefaults;
import org.tensorflow.lite.Interpreter;
import org.tensorflow.lite.Delegate;

import java.nio.MappedByteBuffer;
import java.util.Map;

public class MyModelWrapper {
    private Interpreter tfLiteInterpreter;
    private Map<TFLiteHelpers.DelegateType, Delegate> delegates;

    public MyModelWrapper(Context context, String modelPath) throws Exception {
        // Step 1: Load model from assets
        Pair<MappedByteBuffer, String> modelAndHash =
            TFLiteHelpers.loadModelFile(
                context.getAssets(),
                modelPath
            );

        // Step 2: Create interpreter with QNN NPU + GPU fallback
        Pair<Interpreter, Map<TFLiteHelpers.DelegateType, Delegate>> result =
            TFLiteHelpers.CreateInterpreterAndDelegatesFromOptions(
                modelAndHash.first,                              // Model buffer
                AIHubDefaults.delegatePriorityOrder,            // Delegate priority
                AIHubDefaults.numCPUThreads,                    // CPU threads
                context.getApplicationInfo().nativeLibraryDir,  // Native libs
                context.getCacheDir().getAbsolutePath(),        // Cache dir
                modelAndHash.second                             // Model hash (for caching)
            );

        this.tfLiteInterpreter = result.first;
        this.delegates = result.second;
    }

    public void runInference(float[] inputData, float[] outputData) {
        // Allocate tensors if needed
        if (tfLiteInterpreter.getInputTensorCount() == 0) {
            tfLiteInterpreter.allocateTensors();
        }

        // Run inference
        tfLiteInterpreter.run(inputData, outputData);
    }

    public void close() {
        if (tfLiteInterpreter != null) {
            tfLiteInterpreter.close();
        }
        if (delegates != null) {
            delegates.values().forEach(Delegate::close);
        }
    }
}
```

### Step 5: Understanding Delegate Priority Order

From `AIHubDefaults.java` (apps/android/tflite_helpers/AIHubDefaults.java:24-40):

```java
public static final TFLiteHelpers.DelegateType[][] delegatePriorityOrder = {
    // Attempt 1: QNN_NPU + GPUv2 + XNNPack
    // Best performance on Qualcomm devices
    { TFLiteHelpers.DelegateType.QNN_NPU, TFLiteHelpers.DelegateType.GPUv2 },

    // Attempt 2: GPUv2 + XNNPack
    // Fallback if NPU not supported
    { TFLiteHelpers.DelegateType.GPUv2 },

    // Attempt 3: XNNPack only (CPU)
    // Final fallback for compatibility
    { }
};
```

**How it works**:
1. TFLite tries to create an interpreter with **QNN_NPU + GPUv2**
2. If creation fails (e.g., device doesn't support QNN), tries **GPUv2 only**
3. If that fails, falls back to **CPU (XNNPack) only**
4. **XNNPack** is always enabled as final CPU fallback

### Step 6: Configure QNN NPU Delegate

The `CreateQNN_NPUDelegate()` method (apps/android/tflite_helpers/TFLiteHelpers.java:273-332) automatically configures:

```java
QnnDelegate.Options qnnOptions = new QnnDelegate.Options();

// Point to QNN libraries
qnnOptions.setSkelLibraryDir(nativeLibraryDir);
qnnOptions.setLogLevel(QnnDelegate.Options.LogLevel.LOG_LEVEL_WARN);

// Enable model caching (compilation cache)
qnnOptions.setCacheDir(cacheDir);
qnnOptions.setModelToken(modelIdentifier);

// Hardware detection and configuration
if (QnnDelegate.checkCapability(QnnDelegate.Capability.DSP_RUNTIME)) {
    // Older Snapdragon chipsets use DSP backend
    qnnOptions.setBackendType(QnnDelegate.Options.BackendType.DSP_BACKEND);
    qnnOptions.setDspOptions(
        QnnDelegate.Options.DspPerformanceMode.DSP_PERFORMANCE_BURST,
        QnnDelegate.Options.DspPdSession.DSP_PD_SESSION_ADAPTIVE
    );
} else {
    // Newer Snapdragon chipsets use HTP (Hexagon Tensor Processor)
    boolean hasHTP_FP16 = QnnDelegate.checkCapability(
        QnnDelegate.Capability.HTP_RUNTIME_FP16
    );

    qnnOptions.setBackendType(QnnDelegate.Options.BackendType.HTP_BACKEND);
    qnnOptions.setHtpUseConvHmx(QnnDelegate.Options.HtpUseConvHmx.HTP_CONV_HMX_ON);
    qnnOptions.setHtpPerformanceMode(
        QnnDelegate.Options.HtpPerformanceMode.HTP_PERFORMANCE_BURST
    );

    if (hasHTP_FP16) {
        qnnOptions.setHtpPrecision(
            QnnDelegate.Options.HtpPrecision.HTP_PRECISION_FP16
        );
    }
}
```

### Complete Example: Image Classification

```java
public class ImageClassification {
    private Interpreter interpreter;
    private Map<TFLiteHelpers.DelegateType, Delegate> delegates;

    public ImageClassification(Context context) throws Exception {
        // Load model
        Pair<MappedByteBuffer, String> modelAndHash =
            TFLiteHelpers.loadModelFile(
                context.getAssets(),
                "classification.tflite"
            );

        // Create interpreter
        Pair<Interpreter, Map<TFLiteHelpers.DelegateType, Delegate>> result =
            TFLiteHelpers.CreateInterpreterAndDelegatesFromOptions(
                modelAndHash.first,
                AIHubDefaults.delegatePriorityOrder,
                AIHubDefaults.numCPUThreads,
                context.getApplicationInfo().nativeLibraryDir,
                context.getCacheDir().getAbsolutePath(),
                modelAndHash.second
            );

        this.interpreter = result.first;
        this.delegates = result.second;
    }

    public String[] classify(Bitmap image) {
        // Preprocess image
        TensorBuffer inputBuffer = preprocessImage(image);

        // Run inference
        TensorBuffer outputBuffer = TensorBuffer.createFixedSize(
            interpreter.getOutputTensor(0).shape(),
            DataType.FLOAT32
        );
        interpreter.run(inputBuffer.getBuffer(), outputBuffer.getBuffer());

        // Postprocess results
        return postprocess(outputBuffer);
    }

    private TensorBuffer preprocessImage(Bitmap image) {
        // Resize, normalize, etc.
        // Implementation depends on your model
        return null; // Placeholder
    }

    private String[] postprocess(TensorBuffer output) {
        // Apply softmax, get top-k, etc.
        // Implementation depends on your model
        return new String[0]; // Placeholder
    }

    public void close() {
        if (interpreter != null) {
            interpreter.close();
        }
        if (delegates != null) {
            delegates.values().forEach(Delegate::close);
        }
    }
}
```

---

## Approach 2: Genie SDK (Large Language Models)

### Architecture Overview
```
Android App (Java)
    ↓
GenieWrapper (Java - JNI Interface)
    ↓
GenieWrapper (C++ Native)
    ↓
Genie SDK (C++)
    ↓
QNN Runtime (HTP Backend)
```

### Step 1: Download QAIRT SDK

1. Visit [Qualcomm Package Manager](https://qpm.qualcomm.com/#/main/tools/details/Qualcomm_AI_Runtime_SDK)
2. Download QAIRT SDK (version 2.37.0 or compatible)
3. Extract to a local directory (e.g., `/opt/qcom/aistack/qairt/2.37.0`)

### Step 2: Configure build.gradle

```gradle
plugins {
    id "com.android.application"
}

// Set path to local QAIRT SDK
def qnnSDKLocalPath = "/path/to/qairt/2.37.0"  // UPDATE THIS PATH

android {
    compileSdk 34

    defaultConfig {
        applicationId "com.example.chatapp"
        minSdk 31  // Genie SDK requires Android 12+
        targetSdk 34

        externalNativeBuild {
            cmake {
                cppFlags "-std=c++17"
                abiFilters "arm64-v8a"
                arguments "-DQNN_SDK_ROOT_PATH=" + qnnSDKLocalPath
            }
        }

        sourceSets {
            main {
                jniLibs.srcDir buildDir.toString() + "/libs"
            }
        }
    }

    externalNativeBuild {
        cmake {
            path file("src/main/cpp/CMakeLists.txt")
            version "3.22.1"
        }
    }

    packagingOptions {
        // Extract native libraries for filesystem access
        jniLibs.useLegacyPackaging = true
    }

    aaptOptions {
        // Don't compress model binaries and configs
        noCompress "bin", "json"
    }

    // Pre-build validation and library copying
    preBuild.doFirst {
        if (!qnnSDKLocalPath) {
            throw new RuntimeException(
                "Please download QAIRT SDK and set qnnSDKLocalPath"
            );
        }

        // Copy QNN libraries to build directory
        def libsDir = buildDir.toString() + "/libs/arm64-v8a"
        copy {
            from qnnSDKLocalPath
            include "**/lib/aarch64-android/libQnnHtp.so"
            include "**/lib/aarch64-android/libQnnHtpPrepare.so"
            include "**/lib/aarch64-android/libQnnSystem.so"
            include "**/lib/aarch64-android/libQnnSaver.so"
            include "**/lib/hexagon-v**/unsigned/libQnnHtpV**Skel.so"
            include "**/lib/aarch64-android/libQnnHtpV**Stub.so"

            into libsDir
            eachFile { path = name }
            includeEmptyDirs = false
        }
    }
}

dependencies {
    implementation "androidx.appcompat:appcompat:1.7.0"
    implementation "com.google.android.material:material:1.12.0"
}
```

### Step 3: Create CMakeLists.txt

Create `src/main/cpp/CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.22.1)
project("chatapp")

# QNN SDK paths
set(QNN_SDK_ROOT_PATH "${QNN_SDK_ROOT_PATH}" CACHE STRING "QNN SDK Path")
set(QNN_INCLUDE_DIR "${QNN_SDK_ROOT_PATH}/include")
set(QNN_LIB_DIR "${QNN_SDK_ROOT_PATH}/lib/aarch64-android")

# Include directories
include_directories(
    ${QNN_INCLUDE_DIR}/Genie
    ${QNN_INCLUDE_DIR}/QNN
)

# Add your native library
add_library(
    chatapp
    SHARED
    GenieWrapper.cpp
    GenieLib.cpp
    PromptHandler.cpp
)

# Link against Genie SDK
target_link_libraries(
    chatapp
    ${QNN_LIB_DIR}/libGenie.so
    android
    log
)
```

### Step 4: Implement C++ Wrapper

**GenieWrapper.hpp**:

```cpp
#pragma once
#include <string>
#include "GenieCommon.h"
#include "GenieDialog.h"
#include <jni.h>

class GenieWrapper {
public:
    GenieWrapper(const std::string& model_config_path,
                 const std::string& models_path,
                 const std::string& htp_config_path,
                 const std::string& tokenizer_path);

    ~GenieWrapper();

    std::string GetResponseForPrompt(const std::string& user_prompt,
                                     JNIEnv* env,
                                     jobject callback,
                                     jmethodID onNewStringMethod);

private:
    GenieDialogConfig_Handle_t m_config_handle = nullptr;
    GenieDialog_Handle_t m_dialog_handle = nullptr;
};
```

**GenieWrapper.cpp** (simplified from apps/android/ChatApp/src/main/cpp/GenieWrapper.cpp:95-180):

```cpp
#include "GenieWrapper.hpp"
#include <android/log.h>
#include <fstream>
#include <regex>

GenieWrapper::GenieWrapper(const std::string& model_config_path,
                           const std::string& models_path,
                           const std::string& htp_config_path,
                           const std::string& tokenizer_path) {
    // Load and configure Genie config JSON
    std::string config;
    std::getline(std::ifstream(model_config_path), config, '\0');

    // Replace placeholders with actual paths
    config = std::regex_replace(config, std::regex("<models_path>"), models_path);
    config = std::regex_replace(config, std::regex("<htp_backend_ext_path>"), htp_config_path);
    config = std::regex_replace(config, std::regex("<tokenizer_path>"), tokenizer_path);

    // Create Genie config from JSON
    if (GENIE_STATUS_SUCCESS !=
        GenieDialogConfig_createFromJson(config.c_str(), &m_config_handle)) {
        throw std::runtime_error("Failed to create Genie config");
    }

    // Create Genie dialog instance
    if (GENIE_STATUS_SUCCESS !=
        GenieDialog_create(m_config_handle, &m_dialog_handle)) {
        throw std::runtime_error("Failed to create Genie dialog");
    }
}

GenieWrapper::~GenieWrapper() {
    if (m_config_handle) {
        GenieDialogConfig_free(m_config_handle);
    }
    if (m_dialog_handle) {
        GenieDialog_free(m_dialog_handle);
    }
}

// Callback structure for streaming tokens
struct CallbackData {
    JNIEnv* env;
    jobject callback;
    jmethodID method;
    std::string accumulated_response;
};

void GenieCallback(const char* token,
                   const GenieDialog_SentenceCode_t sentence_code,
                   const void* user_data) {
    auto data = static_cast<CallbackData*>(const_cast<void*>(user_data));
    data->accumulated_response.append(token);

    // Call Java callback with new token
    data->env->CallVoidMethod(
        data->callback,
        data->method,
        data->env->NewStringUTF(token)
    );
}

std::string GenieWrapper::GetResponseForPrompt(
    const std::string& user_prompt,
    JNIEnv* env,
    jobject callback,
    jmethodID onNewStringMethod) {

    CallbackData data{env, callback, onNewStringMethod, ""};

    // Query Genie with user prompt
    if (GENIE_STATUS_SUCCESS !=
        GenieDialog_query(
            m_dialog_handle,
            user_prompt.c_str(),
            GENIE_DIALOG_SENTENCE_COMPLETE,
            GenieCallback,
            &data
        )) {
        __android_log_print(ANDROID_LOG_ERROR, "GenieWrapper",
                           "Failed to get response");
    }

    return data.accumulated_response;
}
```

### Step 5: Create JNI Bridge

**GenieLib.cpp**:

```cpp
#include <jni.h>
#include "GenieWrapper.hpp"
#include <android/log.h>

extern "C" {

// Load model and return native handle
JNIEXPORT jlong JNICALL
Java_com_quicinc_chatapp_GenieWrapper_loadModel(
    JNIEnv* env,
    jobject /* this */,
    jstring modelDirPath,
    jstring htpConfigPath) {

    const char* modelDir = env->GetStringUTFChars(modelDirPath, nullptr);
    const char* htpConfig = env->GetStringUTFChars(htpConfigPath, nullptr);

    try {
        // Construct paths
        std::string configPath = std::string(modelDir) + "/genie_config.json";
        std::string modelsPath = std::string(modelDir);
        std::string tokenizerPath = std::string(modelDir) + "/tokenizer.json";

        // Create wrapper instance
        GenieWrapper* wrapper = new GenieWrapper(
            configPath,
            modelsPath,
            htpConfig,
            tokenizerPath
        );

        env->ReleaseStringUTFChars(modelDirPath, modelDir);
        env->ReleaseStringUTFChars(htpConfigPath, htpConfig);

        return reinterpret_cast<jlong>(wrapper);

    } catch (const std::exception& e) {
        __android_log_print(ANDROID_LOG_ERROR, "GenieLib",
                           "Error loading model: %s", e.what());
        env->ReleaseStringUTFChars(modelDirPath, modelDir);
        env->ReleaseStringUTFChars(htpConfigPath, htpConfig);
        return 0;
    }
}

// Get response for prompt
JNIEXPORT void JNICALL
Java_com_quicinc_chatapp_GenieWrapper_getResponseForPrompt(
    JNIEnv* env,
    jobject /* this */,
    jlong nativeHandle,
    jstring userInput,
    jobject callback) {

    auto wrapper = reinterpret_cast<GenieWrapper*>(nativeHandle);
    const char* input = env->GetStringUTFChars(userInput, nullptr);

    // Get callback method
    jclass callbackClass = env->GetObjectClass(callback);
    jmethodID callbackMethod = env->GetMethodID(
        callbackClass,
        "onNewString",
        "(Ljava/lang/String;)V"
    );

    // Get response
    wrapper->GetResponseForPrompt(input, env, callback, callbackMethod);

    env->ReleaseStringUTFChars(userInput, input);
}

// Free model
JNIEXPORT void JNICALL
Java_com_quicinc_chatapp_GenieWrapper_freeModel(
    JNIEnv* env,
    jobject /* this */,
    jlong nativeHandle) {

    auto wrapper = reinterpret_cast<GenieWrapper*>(nativeHandle);
    delete wrapper;
}

} // extern "C"
```

### Step 6: Create Java Wrapper

**GenieWrapper.java** (from apps/android/ChatApp/src/main/java/com/quicinc/chatapp/GenieWrapper.java):

```java
package com.example.chatapp;

public class GenieWrapper {
    long genieWrapperNativeHandle;

    // Load native library
    static {
        System.loadLibrary("chatapp");
    }

    /**
     * Constructor - Loads model from specified directory
     * @param modelDirPath Path to model bundle (contains genie_config.json, tokenizer.json, *.bin)
     * @param htpConfigPath Path to HTP config JSON for device optimization
     */
    public GenieWrapper(String modelDirPath, String htpConfigPath) {
        genieWrapperNativeHandle = loadModel(modelDirPath, htpConfigPath);
        if (genieWrapperNativeHandle == 0) {
            throw new RuntimeException("Failed to load Genie model");
        }
    }

    /**
     * Generate response for user input
     * @param userInput The user's prompt
     * @param callback Callback to receive generated tokens
     */
    public void getResponseForPrompt(String userInput, StringCallback callback) {
        getResponseForPrompt(genieWrapperNativeHandle, userInput, callback);
    }

    @Override
    protected void finalize() {
        if (genieWrapperNativeHandle != 0) {
            freeModel(genieWrapperNativeHandle);
        }
    }

    // Native methods
    private native long loadModel(String modelDirPath, String htpConfigPath);
    private native void getResponseForPrompt(long nativeHandle, String userInput,
                                            StringCallback callback);
    private native void freeModel(long nativeHandle);
}

// Callback interface for streaming tokens
interface StringCallback {
    void onNewString(String token);
}
```

### Step 7: Configure Genie Config JSON

Create `src/main/assets/models/llm/genie_config.json`:

```json
{
    "dialog": {
        "version": 1,
        "type": "basic",
        "context": {
            "version": 1,
            "size": 2048,
            "n-vocab": 128256,
            "bos-token": -1,
            "eos-token": [128001, 128009, 128008]
        },
        "sampler": {
            "version": 1,
            "seed": 42,
            "temp": 0.8,
            "top-k": 40,
            "top-p": 0.95
        },
        "tokenizer": {
            "version": 1,
            "path": "<tokenizer_path>"
        },
        "engine": {
            "version": 1,
            "n-threads": 3,
            "backend": {
                "version": 1,
                "type": "QnnHtp",
                "QnnHtp": {
                    "version": 1,
                    "use-mmap": true,
                    "spill-fill-bufsize": 0,
                    "mmap-budget": 0,
                    "poll": true,
                    "cpu-mask": "0xe0",
                    "kv-dim": 128,
                    "allow-async-init": false
                },
                "extensions": "<htp_backend_ext_path>"
            },
            "model": {
                "version": 1,
                "type": "binary",
                "binary": {
                    "version": 1,
                    "ctx-bins": [
                        "<models_path>/llama_v3_2_3b_instruct_part_1_of_3.bin",
                        "<models_path>/llama_v3_2_3b_instruct_part_2_of_3.bin",
                        "<models_path>/llama_v3_2_3b_instruct_part_3_of_3.bin"
                    ]
                },
                "positional-encoding": {
                    "type": "rope",
                    "rope-dim": 64,
                    "rope-theta": 500000,
                    "rope-scaling": {
                        "rope-type": "llama3",
                        "factor": 8.0,
                        "low-freq-factor": 1.0,
                        "high-freq-factor": 4.0,
                        "original-max-position-embeddings": 8192
                    }
                }
            }
        }
    }
}
```

**Key Configuration Parameters**:
- `context.size`: Maximum context length (2048 tokens)
- `sampler.temp`: Temperature for generation (0.8)
- `sampler.top-k`: Top-k sampling (40)
- `sampler.top-p`: Nucleus sampling (0.95)
- `backend.type`: "QnnHtp" for HTP backend
- `backend.QnnHtp.cpu-mask`: CPU affinity mask for HTP threads
- `backend.extensions`: Path to device-specific HTP config

### Step 8: Create HTP Config (Device-Specific)

Create device-specific configs in `src/main/assets/htp_config/`:

**qualcomm-snapdragon-8-elite.json**:
```json
{
    "backend_extensions": {
        "shared_library_path": "libQnnHtp.so",
        "config_file_path": "",
        "htp_arch": "v79"
    }
}
```

**qualcomm-snapdragon-8-gen-3.json**:
```json
{
    "backend_extensions": {
        "shared_library_path": "libQnnHtp.so",
        "config_file_path": "",
        "htp_arch": "v75"
    }
}
```

### Step 9: Update AndroidManifest.xml

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application>
        <!-- DSP RPC libraries for HTP -->
        <uses-native-library
            android:name="libadsprpc.so"
            android:required="false" />
        <uses-native-library
            android:name="libcdsprpc.so"
            android:required="false" />
    </application>
</manifest>
```

### Step 10: Use in Activity

```java
public class MainActivity extends AppCompatActivity {
    private GenieWrapper genieWrapper;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // Get paths
        String modelDir = getFilesDir() + "/models/llm";
        String htpConfig = getFilesDir() + "/htp_config/device_config.json";

        // Load model
        try {
            genieWrapper = new GenieWrapper(modelDir, htpConfig);
        } catch (Exception e) {
            Log.e("MainActivity", "Failed to load model", e);
            return;
        }

        // Get response
        genieWrapper.getResponseForPrompt(
            "Hello, how are you?",
            new StringCallback() {
                @Override
                public void onNewString(String token) {
                    runOnUiThread(() -> {
                        // Update UI with new token
                        appendToChat(token);
                    });
                }
            }
        );
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (genieWrapper != null) {
            genieWrapper.finalize();
        }
    }
}
```

---

## Best Practices

### 1. Model Caching
Both approaches support compiled model caching to speed up subsequent loads:

**TFLite Delegate**:
```java
// Cache is automatically managed using MD5 hash
qnnOptions.setCacheDir(context.getCacheDir().getAbsolutePath());
qnnOptions.setModelToken(modelHash);
```

**Genie SDK**:
```json
// Cached via QNN context binaries (.bin files)
"ctx-bins": [
    "model_part_1.bin",
    "model_part_2.bin"
]
```

### 2. Error Handling

Always implement proper fallback logic:

```java
try {
    // Try to create with NPU
    interpreter = createWithQNN();
} catch (Exception e) {
    Log.w(TAG, "NPU failed, falling back to GPU");
    try {
        // Fallback to GPU
        interpreter = createWithGPU();
    } catch (Exception e2) {
        Log.w(TAG, "GPU failed, falling back to CPU");
        // Final fallback to CPU
        interpreter = createWithCPU();
    }
}
```

### 3. Memory Management

Always close resources when done:

```java
@Override
protected void onDestroy() {
    super.onDestroy();

    // Close TFLite resources
    if (interpreter != null) {
        interpreter.close();
    }

    if (delegates != null) {
        for (Delegate delegate : delegates.values()) {
            delegate.close();
        }
    }
}
```

### 4. Threading

For UI applications, run inference on background threads:

```java
ExecutorService executor = Executors.newSingleThreadExecutor();

executor.submit(() -> {
    float[] result = model.runInference(inputData);

    runOnUiThread(() -> {
        updateUI(result);
    });
});
```

### 5. Asset Management

Place models in `src/main/assets/` and ensure they're not compressed:

```gradle
aaptOptions {
    noCompress "tflite", "bin", "json"
}
```

### 6. Device Compatibility

Check hardware capabilities before using NPU:

```java
if (QnnDelegate.checkCapability(QnnDelegate.Capability.HTP_RUNTIME_FP16)) {
    // Device supports FP16 on NPU
    useNPU = true;
} else {
    // Fallback to GPU or CPU
    useNPU = false;
}
```

### 7. Performance Monitoring

Track inference times for optimization:

```java
long startTime = System.currentTimeMillis();
interpreter.run(input, output);
long inferenceTime = System.currentTimeMillis() - startTime;
Log.i(TAG, "Inference took: " + inferenceTime + "ms");
```

### 8. Batch Processing

For multiple inputs, consider batching:

```java
// Instead of multiple single inferences
for (Bitmap image : images) {
    interpreter.run(image, output);  // Inefficient
}

// Batch process if model supports it
float[][][] batchInput = prepareBatch(images);
interpreter.run(batchInput, batchOutput);
```

---

## Troubleshooting

### Common Issues

#### 1. "QNN with NPU backend is not supported on this device"

**Cause**: Device doesn't support QNN NPU acceleration

**Solution**:
- Ensure device is Snapdragon 8 Gen 1 or newer
- Verify libcdsprpc.so is accessible
- App will automatically fallback to GPU/CPU

#### 2. "Failed to create Genie config"

**Cause**: Genie config JSON is malformed or paths are incorrect

**Solution**:
- Verify JSON syntax
- Check that placeholder replacements are working:
  - `<models_path>` → actual model directory
  - `<tokenizer_path>` → actual tokenizer.json path
  - `<htp_backend_ext_path>` → HTP config path

#### 3. "libGenie.so not found"

**Cause**: QAIRT SDK not configured properly in build.gradle

**Solution**:
```gradle
// Verify qnnSDKLocalPath is set correctly
def qnnSDKLocalPath = "/correct/path/to/qairt/2.37.0"

// Check preBuild task copies libraries
preBuild.doFirst {
    copy {
        from qnnSDKLocalPath
        include "**/lib/aarch64-android/libGenie.so"
        // ...
    }
}
```

#### 4. TFLite Delegate Creation Fails

**Cause**: Missing native libraries or wrong version

**Solution**:
```gradle
// Ensure correct versions
implementation "com.qualcomm.qti:qnn-runtime:2.37.0"
implementation "com.qualcomm.qti:qnn-litert-delegate:2.37.0"

// Add to AndroidManifest.xml
<uses-native-library android:name="libcdsprpc.so" android:required="false" />
```

#### 5. Model Inference Returns Incorrect Results

**Cause**: Preprocessing or postprocessing issues

**Solution**:
- Verify input tensor format matches model expectations
- Check normalization values (e.g., [0, 1] vs [0, 255])
- Validate output tensor interpretation

#### 6. "Unable to create an interpreter of any kind"

**Cause**: All delegate creation attempts failed

**Solution**:
- Check logcat for specific delegate errors
- Verify model is not corrupted
- Ensure sufficient device memory
- Try with CPU-only fallback explicitly:
  ```java
  TFLiteHelpers.DelegateType[][] cpuOnly = {{ }};
  CreateInterpreterAndDelegatesFromOptions(..., cpuOnly, ...);
  ```

#### 7. Slow First Inference

**Cause**: Model compilation happening on first run

**Solution**: This is expected behavior. Subsequent runs will use cached compilation:
```java
// First run: ~10-30 seconds (compilation)
// Subsequent runs: <100ms (cached)
```

#### 8. App Crashes with "SIGABRT" or "SIGSEGV"

**Cause**: Native memory issues or wrong ABI

**Solution**:
- Ensure `abiFilters "arm64-v8a"` is set
- Verify you're testing on arm64-v8a device (64-bit)
- Check native library compatibility

### Debugging Tips

#### Enable QNN Logging

**TFLite Delegate**:
```java
qnnOptions.setLogLevel(QnnDelegate.Options.LogLevel.LOG_LEVEL_DEBUG);
```

**Genie SDK**:
In genie_config.json, add logging configuration (check QAIRT SDK docs)

#### Check Logcat

```bash
# Filter for Qualcomm-related logs
adb logcat | grep -E "(QNN|Genie|HTP|cdsp)"

# Check for delegate creation
adb logcat | grep "TFLiteHelpers"
```

#### Verify Library Loading

```java
try {
    System.loadLibrary("chatapp");
    Log.i(TAG, "Native library loaded successfully");
} catch (UnsatisfiedLinkError e) {
    Log.e(TAG, "Failed to load native library", e);
}
```

#### Inspect Delegate Usage

```java
// After creating interpreter, check which delegates were used
Log.i(TAG, "Active delegates: " + delegates.keySet());
```

---

## Performance Optimization

### 1. Use Appropriate Precision

For QNN NPU, FP16 offers best performance/accuracy tradeoff:

```java
if (QnnDelegate.checkCapability(QnnDelegate.Capability.HTP_RUNTIME_FP16)) {
    qnnOptions.setHtpPrecision(QnnDelegate.Options.HtpPrecision.HTP_PRECISION_FP16);
}
```

### 2. Optimize Context Size (Genie SDK)

Balance memory usage and capability:

```json
"context": {
    "size": 2048  // Reduce if memory constrained
}
```

### 3. Configure Performance Mode

For maximum performance:

```java
// TFLite
qnnOptions.setHtpPerformanceMode(
    QnnDelegate.Options.HtpPerformanceMode.HTP_PERFORMANCE_BURST
);

// Genie SDK - in genie_config.json
"backend": {
    "QnnHtp": {
        "poll": true,  // Enable polling for lower latency
        "cpu-mask": "0xe0"  // Use specific CPU cores
    }
}
```

### 4. Reuse Interpreter Instances

Don't recreate interpreters unnecessarily:

```java
// Good: Create once, reuse
Interpreter interpreter = createInterpreter();
for (Bitmap image : images) {
    interpreter.run(preprocess(image), output);
}

// Bad: Recreate each time
for (Bitmap image : images) {
    Interpreter interpreter = createInterpreter();  // Expensive!
    interpreter.run(preprocess(image), output);
    interpreter.close();
}
```

---

## Additional Resources

### Official Documentation
- [Qualcomm AI Hub](https://app.aihub.qualcomm.com/)
- [QNN SDK Documentation](https://docs.qualcomm.com/bundle/publicresource/topics/80-63442-50/introduction.html)
- [TensorFlow Lite Delegates](https://www.tensorflow.org/lite/performance/delegates)

### Sample Code Locations
- **TFLite Helper**: `apps/android/tflite_helpers/TFLiteHelpers.java`
- **Image Classification**: `apps/android/ImageClassification/`
- **Chat App**: `apps/android/ChatApp/`
- **Object Detection**: `apps/android/ObjectDetection/`

### Key Files Reference
- Build config: `apps/android/gradle.properties`
- QNN delegate: `apps/android/tflite_helpers/TFLiteHelpers.java:273`
- Genie wrapper: `apps/android/ChatApp/src/main/cpp/GenieWrapper.cpp`
- Delegate priority: `apps/android/tflite_helpers/AIHubDefaults.java:24`

---

## License

This documentation is based on sample code from Qualcomm AI Hub Android Sample Apps.
Copyright (c) 2025 Qualcomm Technologies, Inc. and/or its subsidiaries.
SPDX-License-Identifier: BSD-3-Clause
