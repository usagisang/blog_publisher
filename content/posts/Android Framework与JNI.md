---
tags:
  - android/framework
title: Android Framework与JNI
date: 2024-10-09
---


本文以 android-12.0.0_r34 的代码进行分析。

## framework中的JNI

通常来说，Android framework 中使用到的 native 函数都是动态注册的，而且注册过程有固定的套路。我们以`Parcel`类为例来解析套路。

首先，我们在`Parcel.java`中会看到很多标记为 native 的方法（为了便于演示，去掉了很多 native 方法）：

```java
public final class Parcel {
    private static native long nativeCreate();
    private static native long nativeFreeBuffer(long nativePtr);
    private static native void nativeDestroy(long nativePtr);
}
```

然后，你可以在 Android 源码中找到相应的 jni 定义：

```cpp
// frameworks/base/core/jni/android_os_Parcel.cpp
#include "core_jni_helpers.h"

namespace android {

static struct parcel_offsets_t
{
    jclass clazz;
    jfieldID mNativePtr;
    jmethodID obtain;
    jmethodID recycle;
} gParcelOffsets;
    

static jlong android_os_Parcel_create(JNIEnv* env, jclass clazz)
{
    Parcel* parcel = new Parcel();
    return reinterpret_cast<jlong>(parcel);
}

static void android_os_Parcel_freeBuffer(JNIEnv* env, jclass clazz, jlong nativePtr)
{
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        parcel->freeData();
    }
}

static void android_os_Parcel_destroy(JNIEnv* env, jclass clazz, jlong nativePtr)
{
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    delete parcel;
}
}

// ----------------------------------------------------------------------------

static const JNINativeMethod gParcelMethods[] = {
    {"nativeCreate",              "()J", (void*)android_os_Parcel_create},
    {"nativeFreeBuffer",          "(J)V", (void*)android_os_Parcel_freeBuffer},
    {"nativeDestroy",             "(J)V", (void*)android_os_Parcel_destroy},
};

const char* const kParcelPathName = "android/os/Parcel";

int register_android_os_Parcel(JNIEnv* env)
{
    jclass clazz = FindClassOrDie(env, kParcelPathName);

    gParcelOffsets.clazz = MakeGlobalRefOrDie(env, clazz);
    gParcelOffsets.mNativePtr = GetFieldIDOrDie(env, clazz, "mNativePtr", "J");
    gParcelOffsets.obtain = GetStaticMethodIDOrDie(env, clazz, "obtain", "()Landroid/os/Parcel;");
    gParcelOffsets.recycle = GetMethodIDOrDie(env, clazz, "recycle", "()V");

    return RegisterMethodsOrDie(env, kParcelPathName, gParcelMethods, NELEM(gParcelMethods));
}

};
```

首先，我们会发现 Java 层定义的 native 方法的名字中都带有 native 前缀，但实际上，它们对应的 cpp 函数的名称都不会带有 native 前缀。

其次，源码中习惯上会使用一个全局 static 数组来统一存放 cpp 函数与 Java 层定义的映射，然后，再定义一个`register_xxx`的函数，此函数内部一般执行两件事，第一件是完成这个文件内所有 JNI 函数的动态注册；第二件是提前使用`JNIEnv`找到后续 cpp 函数回调 Java 层时可能要用到的`jclass`、`jfieldID`或`jmethodID`，这些信息会保存在另一个全局结构体中。

## RegisterMethodsOrDie

动态注册的关键在于`RegisterMethodsOrDie`这个函数，这个函数实际上来自于`frameworks/base/core/jni/core_jni_helpers.h`（正因为在同一个目录下，因此 include 的时候使用了双引号）

```cpp
// frameworks/base/core/jni/core_jni_helpers.h

#include <android_runtime/AndroidRuntime.h>
namespace android {

static inline int RegisterMethodsOrDie(JNIEnv* env, const char* className,
                                       const JNINativeMethod* gMethods, int numMethods) {
    int res = AndroidRuntime::registerNativeMethods(env, className, gMethods, numMethods);
    LOG_ALWAYS_FATAL_IF(res < 0, "Unable to register native methods.");
    return res;
}
}
```

我们继续追踪`AndroidRuntime.h`，这个头文件位于`frameworks/base/core/jni/include/android_runtime/AndroidRuntime.h`，实现位于`frameworks/base/core/jni/AndroidRuntime.cpp`

```cpp
// frameworks/base/core/jni/AndroidRuntime.cpp

#include <nativehelper/JNIHelp.h>
namespace android {
/*
 * Register native methods using JNI.
 */
/*static*/ int AndroidRuntime::registerNativeMethods(JNIEnv* env,
    const char* className, const JNINativeMethod* gMethods, int numMethods)
{
    return jniRegisterNativeMethods(env, className, gMethods, numMethods);
}
}
```

`jniRegisterNativeMethods`这个方法实际上由`JNIHelp.h`导入，实现位于`libnativehelper/JNIHelp.c`

```cpp
// libnativehelper/JNIHelp.c

int jniRegisterNativeMethods(JNIEnv* env, const char* className,
    const JNINativeMethod* methods, int numMethods)
{
    ALOGV("Registering %s's %d native methods...", className, numMethods);
    jclass clazz = (*env)->FindClass(env, className);
    ALOG_ALWAYS_FATAL_IF(clazz == NULL,
                         "Native registration unable to find class '%s'; aborting...",
                         className);
    int result = (*env)->RegisterNatives(env, clazz, methods, numMethods);
    (*env)->DeleteLocalRef(env, clazz);
    if (result == 0) {
        return 0;
    }

    // 剩下的代码与失败处理相关, 省略
    // ...
    ALOGF("RegisterNatives failed for '%s'; aborting...", className);
    return result;
}

```

## 何时调用register

摸清楚了 Android framework 如何动态注册 JNI，另一个问题便是 framework 会在什么时候调用相应的 register 函数呢？答案依然在`frameworks/base/core/jni/AndroidRuntime.cpp`中。

首先，我们会在 AndroidRuntime.cpp 中看到 extern 声明和一个全局数组：

```cpp
// frameworks/base/core/jni/AndroidRuntime.cpp

using namespace android;

// 不定义在android这个命名空间内的register函数, 数量不少, 这里仅展示一小部分
extern int register_android_opengl_jni_EGL14(JNIEnv* env);
extern int register_android_opengl_jni_EGL15(JNIEnv* env);
extern int register_android_opengl_jni_EGLExt(JNIEnv* env);
// ...

namespace android {
/*
 * JNI-based registration functions.  Note these are properly contained in
 * namespace android.
 */
// 省略...
extern int register_android_os_MessageQueue(JNIEnv* env);
extern int register_android_os_Parcel(JNIEnv* env);
// 省略...

// 定义了REG_JNI, 如果没开启debug, 这个结构体就不会存储名字
#ifdef NDEBUG
    #define REG_JNI(name)      { name }
    struct RegJNIRec {
        int (*mProc)(JNIEnv*);
    };
#else
    #define REG_JNI(name)      { name, #name }
    struct RegJNIRec {
        int (*mProc)(JNIEnv*);
        const char* mName;
    };
#endif
static const RegJNIRec gRegJNI[] = {
        REG_JNI(register_android_os_Parcel),
        REG_JNI(register_android_opengl_jni_EGL14),
        REG_JNI(register_android_opengl_jni_EGL15),
        REG_JNI(register_android_opengl_jni_EGLExt),
        REG_JNI(register_android_os_MessageQueue)
};
}
```

`gRegJNI`这个数组基本上约等于存放了一堆 lambda，等待Android 虚拟机启动时，直接遍历该数组就可以完成注册。其调用链大致为：start -> startReg -> register_jni_procs

```cpp
// frameworks/base/core/jni/AndroidRuntime.cpp

/*
 * Start the Android runtime.  This involves starting the virtual machine
 * and calling the "static void main(String[] args)" method in the class
 * named by "className".
 *
 * Passes the main function two arguments, the class name and the specified
 * options string.
 */
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    // ...
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote, primary_zygote) != 0) {
        return;
    }
    onVmCreated(env);

    /*
     * Register android functions.
     */
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }
    // ...
}

/*
 * Register android native functions with the VM.
 */
/*static*/ int AndroidRuntime::startReg(JNIEnv* env)
{
    ATRACE_NAME("RegisterAndroidNatives");
    /*
     * This hook causes all future threads created in this process to be
     * attached to the JavaVM.  (This needs to go away in favor of JNI
     * Attach calls.)
     */
    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);

    ALOGV("--- registering native functions ---\n");

    /*
     * Every "register" function calls one or more things that return
     * a local reference (e.g. FindClass).  Because we haven't really
     * started the VM yet, they're all getting stored in the base frame
     * and never released.  Use Push/Pop to manage the storage.
     */
    env->PushLocalFrame(200);

    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
        env->PopLocalFrame(NULL);
        return -1;
    }
    env->PopLocalFrame(NULL);

    //createJavaThread("fubar", quickTest, (void*) "hello");

    return 0;
}

static int register_jni_procs(const RegJNIRec array[], size_t count, JNIEnv* env)
{
    for (size_t i = 0; i < count; i++) {
        if (array[i].mProc(env) < 0) {
#ifndef NDEBUG
            ALOGD("----------!!! %s failed to load\n", array[i].mName);
#endif
            return -1;
        }
    }
    return 0;
}
```

