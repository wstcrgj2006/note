本文主要讲述 Android 动态链接库的加载过程及其涉及的底层原理，包含了如下内容：

- 以一个Linux示例描述native层加载动态链接库的过程
- 以一个Android示例描述在Android应用开发中动态链接库的创建及native函数的使用方法
- 结合ART虚拟机源码详细的分析上述示例中动态链接库的加载过程（`System.loadLibrary`）

我们知道在Android（Java）中加载一个动态链接库非常简单，只需一行代码:

```java
System.loadLibrary("native-lib");
```

事实上这是Java提供的API，对于Java层实现基本一致，但是对于不同的JVM，其底层（native）实现会有所差异。本文分析的代码基于Android 6.0系统（[ART](https://android.googlesource.com/platform/art/) 源码分支：`marshmallow-release`）。

## Linux 系统加载动态库过程分析

Android是基于Linux系统的，为了理解Android动态链接库的加载过程，首先需要了解Linux系统动态链接库的加载过程。

Linux环境下加载动态库主要包括如下函数，它们位于头文件`#include <dlfcn.h>`中：

```C
void *dlopen(const char *filename, int flag);  // 打开动态链接库
void *dlsym(void *handle, const char *symbol);  // 获取方法指针
char *dlerror(void);  // 获取错误信息
int dlclose(void *handle);  // 关闭动态链接库
```

可以通过`man`命令查看上述函数的具体使用方法，例如：

```
man dlopen
```

会显示`dlopen`函数的详细信息

```C
DLOPEN(3)                BSD Library Functions Manual                DLOPEN(3)

NAME
     dlopen -- load and link a dynamic library or bundle

SYNOPSIS
     #include <dlfcn.h>

     void*
     dlopen(const char* path, int mode);
...
```

那么如何在Linux环境下生成动态链接库，然后加载并使用其中的函数呢？

首先创建一个C++文件，在其中实现一个简单的两数相加函数，然后由其生成动态链接库。

[caculate.cpp]

```C
extern "C"
int add(int a, int b) {
    return a + b;
}
```

C++文件函数前的 `extern “C”` 不能省略，原因是C++编译之后会修改函数名，之后动态加载函数的时候会找不到该函数。加上 `extern “C”` 是告诉编译器以C的方式编译，不用修改函数名。

然后通过下述命令编译成动态链接库：

```
g++ -fPIC -shared caculate.cpp -o libcaculate.so
```

执行此命令后会在同级目录生成一个动态库文件`libcaculate.so`

然后编写加载动态库并使用的代码：

[call_native.cpp]

```C++
#include <iostream>
#include <dlfcn.h>

using namespace std;

int main() {
    // 1.打开动态库，拿到一个动态库句柄
    void* handle = dlopen("./libcaculate.so", RTLD_NOW);
    if(handle == nullptr) {
        cout<<"load error!"<<endl;
        return -1;
    }

    // 若有错误输出错误信息
    char* errorMsg;
    if ((errorMsg = dlerror()) != nullptr) {
        cout<<"dlopen failed, errorMsg:"<<errorMsg<<endl;
        return -1;
    }

    cout<<"dlopen success!"<<endl;

    // 2.通过句柄和方法名获取方法指针地址
    void* symAdd = dlsym(handle, "add");
    if(symAdd == nullptr) {
        cout<<"dlsym failed!"<<endl;
        // 若有错误输出错误信息
        if ((errorMsg = dlerror()) != nullptr) {
            cout<<"dlsym failed, errorMsg:"<<errorMsg<<endl;
            return -1;
        }
    }

    // 3.将方法地址强制类型转换成方法指针
    typedef int (*CACULATE_ADD_Fn)(int, int);
    CACULATE_ADD_Fn addFn = reinterpret_cast<CACULATE_ADD_Fn>(symAdd);

    // 4.调用动态库中的方法
    cout<<"1 + 2 = "<<addFn(1, 2)<<endl;

    // 5.通过句柄关闭动态库
    dlclose(handle);
    return 0;
}
```

`call_native.cpp` 使用了上面提到的4个函数，具体流程如下：
1. 打开动态库，拿到一个动态库句柄
1. 通过句柄和方法名获取方法指针地址
1. 将方法地址强制类型转换成方法指针
1. 调用动态库中的方法
1. 通过句柄关闭动态库

在此流程中会使用`dlerror`检测是否有错误。

这里解释一下方法指针地址到方法指针的转换，为了方便这里定义了一个方法指针的别名：

```C++
typedef int (*CACULATE_ADD_Fn)(int, int);
```

指明该方法接受两个int类型参数返回一个int值。

拿到地址之后强制类型转换成方法指针用于调用：

```C++
CACULATE_ADD_Fn addFn = reinterpret_cast<CACULATE_ADD_Fn>(symAdd);
```

最后编译运行：

```
g++ -std=c++11 -ldl call_native.cpp -o call_native
./call_native
```

因为代码中使用了C++11标准新加的特性，所以编译的时候带上`-std=c++11`，另外使用了头文件`dlfcn.h`需要带上`-ldl`，编译生成的`call_native`文件即是二进制可执行文件，需要将动态库放在同级目录下执行。

上面就是Linux环境下创建动态库，加载并使用动态库的全部过程。

接下来以一个具体的Android示例介绍在Android应用开发中动态库的使用方法。

## Android开发中一个简单的JNI示例

首先确保在“SDK Manager”中安装NDK，然后在AndroidStudio中依次选择 File -> Project Structure... -> SDK Location -> Android NDK Location 设置NDK。

1. 新建Android工程`AndroidTest`，创建`HelloJNI.java`文件，在此文件中加载动态链接库和定义native方法，如下：

```java
package com.example.androidtest;

public class HelloJNI {
    static {
        System.loadLibrary("hello-jni");
    }

    public static native String stringFromJNI();
}
```

2. 创建`MainActivity.java`，当点击Button时在logcat中输出调用native方法获得的字符串。

```java
package com.example.androidtest;

import android.app.Activity;
import android.os.Bundle;
import android.util.Log;
import android.view.View;

public class MainActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        findViewById(R.id.btn).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Log.e("zzz", "Get string from native = " + HelloJNI.stringFromJNI());
            }
        });
    }

}
```

3. 创建Activity用到的`activity_main.xml`

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <Button
        android:id="@+id/btn"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

</LinearLayout>
```

4. 进入`HelloJNI.java`文件所在目录，使用`javac`命令编译`HelloJNI.java`：

```
javac HelloJNI.java
```

得到`HelloJNI.class`文件

5. 进入module的java根目录（即：${PROJECT}/${MODULE}/src/main/java），执行`javah`命令：

```
javah com.example.androidtest.HelloJNI
```

此时将生成头文件`com_example_androidtest_HelloJNI.h`，为了方便使用，更名为`hello-jni.h`

6. 在java根目录（即：${PROJECT}/${MODULE}/src/main/java）目录下创建`jni`目录，用于保存jni相关的文件。将`hello-jni.h`文件移动到此目录中，接着在此目录中创建`hello-jni.c`，`Android.mk`，`Application.mk`，文件内容分别如下：

[hello-jni.c]

```C
/*
 * Copyright (C) 2016 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 *
 */
#include <string.h>
#include <jni.h>

/* This is a trivial JNI example where we use a native method
 * to return a new VM String. See the corresponding Java source
 * file located at:
 *
 *   ~/app/src/main/java/com/example/androidtest/HelloJNI.java
 */
JNIEXPORT jstring JNICALL
Java_com_example_androidtest_HelloJNI_stringFromJNI( JNIEnv* env,
                                                  jclass clazz )
{
#if defined(__arm__)
    #if defined(__ARM_ARCH_7A__)
    #if defined(__ARM_NEON__)
      #if defined(__ARM_PCS_VFP)
        #define ABI "armeabi-v7a/NEON (hard-float)"
      #else
        #define ABI "armeabi-v7a/NEON"
      #endif
    #else
      #if defined(__ARM_PCS_VFP)
        #define ABI "armeabi-v7a (hard-float)"
      #else
        #define ABI "armeabi-v7a"
      #endif
    #endif
  #else
   #define ABI "armeabi"
  #endif
#elif defined(__i386__)
#define ABI "x86"
#elif defined(__x86_64__)
#define ABI "x86_64"
#elif defined(__mips64)  /* mips64el-* toolchain defines __mips__ too */
#define ABI "mips64"
#elif defined(__mips__)
#define ABI "mips"
#elif defined(__aarch64__)
#define ABI "arm64-v8a"
#else
#define ABI "unknown"
#endif

    return (*env)->NewStringUTF(env, "Hello from JNI !  Compiled with ABI " ABI ".");
}
```

[Android.mk]

```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE    := hello-jni
LOCAL_SRC_FILES := hello-jni.c

include $(BUILD_SHARED_LIBRARY)
```

[Application.mk]

```
APP_ABI := all
```

7. 在`Android.mk`文件目录下执行`ndk-build`命令，将在module的java根目录下生成`libs`文件夹，此文件夹中即是生成的so库，在module的java根目录的同级目录下创建`jniLibs`目录，将`libs`文件夹中的内容拷贝`jniLibs`目录中，此时整个项目的结构如下：

```
AndroidTest
--app
----src
------main
--------java
----------com.example.androidtest
------------HelloJNI.java
------------HelloJNI.class
------------MainActivity.java
----------jni
------------Android.mk
------------Application.mk
------------hello-jni.c
------------hello-jni.h
----------libs
------------arm64-v8a
--------------libhello-jni.so
------------armeabi-v7a
--------------libhello-jni.so
------------x86
--------------libhello-jni.so
------------x86_64
--------------libhello-jni.so
--------jniLibs
----------arm64-v8a
------------libhello-jni.so
----------armeabi-v7a
------------libhello-jni.so
----------x86
------------libhello-jni.so
----------x86_64
------------libhello-jni.so
```

8. 运行项目，点击界面上的按钮，即可在logcat中看到调用native函数得到的字符串。

由于Android基于Linux系统，所以我们有理由猜测Android系统底层也是通过Linux系统加载动态链接库的方式加载并使用动态库的。下面从Android 上层 Java 代码入手开始分析。

## ART虚拟机动态链接库加载过程分析

注：下文中`aosp`表示Android源码根目录

动态链接库的加载是通过执行`System.loadLibrary`实现的，所以下面将以`System.loadLibrary`为入口分析整个加载流程。

[aosp/libcore/luni/src/main/java/java/lang/**System.java**]

```java
public static void loadLibrary(String libName) {
    Runtime.getRuntime().loadLibrary(libName, VMStack.getCallingClassLoader());
}
```

此处`Runtime.getRuntime()`拿到一个`Runtime`单例，见

[aosp/libcore/luni/src/main/java/java/lang/**Runtime.java**]

```java
private static final Runtime mRuntime = new Runtime();
public static Runtime getRuntime() {
    return mRuntime;
}
```

此处`VMStack.getCallingClassLoader()`拿到的是调用者的ClassLoader，一般情况下是PathClassLoader。（为什么是PathClassLoader，参考文章[Android应用中的ClassLoader](./Android应用中的ClassLoader.md)）

[aosp/libcore/libart/src/main/java/dalvik/system/**VMStack.java**]

```java
/**
 * Returns the defining class loader of the caller's caller.
 *
 * @return the requested class loader, or {@code null} if this is the
 *         bootstrap class loader.
 */
native public static ClassLoader getCallingClassLoader();
```

[aosp/libcore/luni/src/main/java/java/lang/**Runtime.java**]

```java
/*
 * Searches for and loads the given shared library using the given ClassLoader.
 */
void loadLibrary(String libraryName, ClassLoader loader) {
    if (loader != null) {
        String filename = loader.findLibrary(libraryName);
        if (filename == null) {
            // It's not necessarily true that the ClassLoader used
            // System.mapLibraryName, but the default setup does, and it's
            // misleading to say we didn't find "libMyLibrary.so" when we
            // actually searched for "liblibMyLibrary.so.so".
            throw new UnsatisfiedLinkError(loader + " couldn't find \"" +
                    System.mapLibraryName(libraryName) + "\"");
        }
        String error = doLoad(filename, loader);
        if (error != null) {
            throw new UnsatisfiedLinkError(error);
        }
        return;
    }

    String filename = System.mapLibraryName(libraryName);
    List<String> candidates = new ArrayList<String>();
    String lastError = null;
    for (String directory : mLibPaths) {
        String candidate = directory + filename;
        candidates.add(candidate);

        if (IoUtils.canOpenReadOnly(candidate)) {
            String error = doLoad(candidate, loader);
            if (error == null) {
                return; // We successfully loaded the library. Job done.
            }
            lastError = error;
        }
    }

    if (lastError != null) {
        throw new UnsatisfiedLinkError(lastError);
    }
    throw new UnsatisfiedLinkError("Library " + libraryName + " not found; tried " + candidates);
}
```

这里根据ClassLoader是否存在分了两种情况，当ClasssLoader存在的时候通过loader的findLibrary()查看目标库所在路径，当ClassLoader不存在的时候通过mLibPaths加载路径。最终都会调用doLoad加载动态库。

**当ClassLoader存在时**，执行`loader.findLibrary`，`findLibrary`位于PathClassLoader的父类BaseDexClassLoader中：

[aosp/libcore/dalvik/src/main/java/dalvik/system/**BaseDexClassLoader.java**]

```java
private final DexPathList pathList;

@Override
public String findLibrary(String name) {
    return pathList.findLibrary(name);
}
```

`pathList`的类型为DexPathList，它的构造方法如下：

[aosp/libcore/dalvik/src/main/java/dalvik/system/**DexPathList.java**]

```java
/**
 * Constructs an instance.
 *
 * @param definingContext the context in which any as-yet unresolved
 * classes should be defined
 * @param dexPath list of dex/resource path elements, separated by
 * {@code File.pathSeparator}
 * @param libraryPath list of native library directory path elements,
 * separated by {@code File.pathSeparator}
 * @param optimizedDirectory directory where optimized {@code .dex} files
 * should be found and written to, or {@code null} to use the default
 * system directory for same
 */
public DexPathList(ClassLoader definingContext, String dexPath,
                   String libraryPath, File optimizedDirectory) {

    if (definingContext == null) {
        throw new NullPointerException("definingContext == null");
    }

    if (dexPath == null) {
        throw new NullPointerException("dexPath == null");
    }

    if (optimizedDirectory != null) {
        if (!optimizedDirectory.exists())  {
            throw new IllegalArgumentException(
                    "optimizedDirectory doesn't exist: "
                            + optimizedDirectory);
        }

        if (!(optimizedDirectory.canRead()
                && optimizedDirectory.canWrite())) {
            throw new IllegalArgumentException(
                    "optimizedDirectory not readable/writable: "
                            + optimizedDirectory);
        }
    }

    this.definingContext = definingContext;

    ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
    // save dexPath for BaseDexClassLoader
    this.dexElements = makePathElements(splitDexPath(dexPath), optimizedDirectory,
            suppressedExceptions);

    // Native libraries may exist in both the system and
    // application library paths, and we use this search order:
    //
    //   1. This class loader's library path for application libraries (libraryPath):
    //   1.1. Native library directories
    //   1.2. Path to libraries in apk-files
    //   2. The VM's library path from the system property for system libraries
    //      also known as java.library.path
    //
    // This order was reversed prior to Gingerbread; see http://b/2933456.
    this.nativeLibraryDirectories = splitPaths(libraryPath, false);
    this.systemNativeLibraryDirectories =
            splitPaths(System.getProperty("java.library.path"), true);
    List<File> allNativeLibraryDirectories = new ArrayList<>(nativeLibraryDirectories);
    allNativeLibraryDirectories.addAll(systemNativeLibraryDirectories);

    this.nativeLibraryPathElements = makePathElements(allNativeLibraryDirectories, null,
            suppressedExceptions);

    if (suppressedExceptions.size() > 0) {
        this.dexElementsSuppressedExceptions =
                suppressedExceptions.toArray(new IOException[suppressedExceptions.size()]);
    } else {
        dexElementsSuppressedExceptions = null;
    }
}
```

这里收集了apk的so目录，例如/data/app-lib/${package-name}/，
还有系统的so目录：System.getProperty(“java.library.path”)，在32位系统上其值为`/vendor/lib:/system/lib`（即：/vendor/lib和/system/lib两个目录），64位系统是/vendor/lib64:/system/lib64。

最终查找so文件的时候就会在这三个路径中查找，优先查找apk目录。

[aosp/libcore/dalvik/src/main/java/dalvik/system/**DexPathList.java**]

```java
/**
 * Finds the named native code library on any of the library
 * directories pointed at by this instance. This will find the
 * one in the earliest listed directory, ignoring any that are not
 * readable regular files.
 *
 * @return the complete path to the library or {@code null} if no
 * library was found
 */
public String findLibrary(String libraryName) {
    String fileName = System.mapLibraryName(libraryName);

    for (Element element : nativeLibraryPathElements) {
        String path = element.findNativeLibrary(fileName);

        if (path != null) {
            return path;
        }
    }

    return null;
}
```

`System.mapLibraryName(libraryName)`的实现很简单：

```java
/**
 * Returns the platform specific file name format for the shared library
 * named by the argument. On Android, this would turn {@code "MyLibrary"} into
 * {@code "libMyLibrary.so"}.
 */
public static String mapLibraryName(String nickname) {
    if (nickname == null) {
        throw new NullPointerException("nickname == null");
    }
    return "lib" + nickname + ".so";
}
```

也就是为什么动态库的命名必须以`lib`开头的原因。

然后会遍历`nativeLibraryPathElements`查找某个目录下是否有改文件，有的话就返回：

[aosp/libcore/dalvik/src/main/java/dalvik/system/**DexPathList.java**]

```java
public String findNativeLibrary(String name) {
    maybeInit();

    if (isDirectory) {
        String path = new File(dir, name).getPath();
        if (IoUtils.canOpenReadOnly(path)) {
            return path;
        }
    } else if (zipFile != null) {
        String entryName = new File(dir, name).getPath();
        if (isZipEntryExistsAndStored(zipFile, entryName)) {
            return zip.getPath() + zipSeparator + entryName;
        }
    }

    return null;
}
```

回到`Runtime`的`loadLibrary`方法，通过`ClassLoader`找到目标文件之后会调用`doLoad`方法：

[aosp/libcore/luni/src/main/java/java/lang/**Runtime.java**]

```java
private String doLoad(String name, ClassLoader loader) {
    // Android apps are forked from the zygote, so they can't have a custom LD_LIBRARY_PATH,
    // which means that by default an app's shared library directory isn't on LD_LIBRARY_PATH.

    // The PathClassLoader set up by frameworks/base knows the appropriate path, so we can load
    // libraries with no dependencies just fine, but an app that has multiple libraries that
    // depend on each other needed to load them in most-dependent-first order.

    // We added API to Android's dynamic linker so we can update the library path used for
    // the currently-running process. We pull the desired path out of the ClassLoader here
    // and pass it to nativeLoad so that it can call the private dynamic linker API.

    // We didn't just change frameworks/base to update the LD_LIBRARY_PATH once at the
    // beginning because multiple apks can run in the same process and third party code can
    // use its own BaseDexClassLoader.

    // We didn't just add a dlopen_with_custom_LD_LIBRARY_PATH call because we wanted any
    // dlopen(3) calls made from a .so's JNI_OnLoad to work too.

    // So, find out what the native library search path is for the ClassLoader in question...
    String ldLibraryPath = null;
    String dexPath = null;
    if (loader == null) {
        // We use the given library path for the boot class loader. This is the path
        // also used in loadLibraryName if loader is null.
        ldLibraryPath = System.getProperty("java.library.path");
    } else if (loader instanceof BaseDexClassLoader) {
        BaseDexClassLoader dexClassLoader = (BaseDexClassLoader) loader;
        ldLibraryPath = dexClassLoader.getLdLibraryPath();
    }
    // nativeLoad should be synchronized so there's only one LD_LIBRARY_PATH in use regardless
    // of how many ClassLoaders are in the system, but dalvik doesn't support synchronized
    // internal natives.
    synchronized (this) {
        return nativeLoad(name, loader, ldLibraryPath);
    }
}
```

这里的`ldLibraryPath`和之前所述类似，`loader`为空时使用系统目录，否则使用`ClassLoader`提供的目录，`ClassLoader`提供的目录中包括apk目录和系统目录。

最后调用native代码：

```java
private static native String nativeLoad(String filename, ClassLoader loader,
            String ldLibraryPath);
```

[aosp/art/runtime/native/**java_lang_Runtime.cc**]

```C++
static jstring Runtime_nativeLoad(JNIEnv* env, jclass, jstring javaFilename, jobject javaLoader,
                                  jstring javaLdLibraryPathJstr) {
    ScopedUtfChars filename(env, javaFilename);
    if (filename.c_str() == nullptr) {
        return nullptr;
    }

    SetLdLibraryPath(env, javaLdLibraryPathJstr);

    std::string error_msg;
    {
        JavaVMExt* vm = Runtime::Current()->GetJavaVM();
        bool success = vm->LoadNativeLibrary(env, filename.c_str(), javaLoader, &error_msg);
        if (success) {
            return nullptr;
        }
    }

    // Don't let a pending exception from JNI_OnLoad cause a CheckJNI issue with NewStringUTF.
    env->ExceptionClear();
    return env->NewStringUTF(error_msg.c_str());
}
```

接着调用`JavaVMExt`对象的`LoadNativeLibrary`方法：

[aosp/art/runtime/**java_vm_ext.cc**]

```C++
bool JavaVMExt::LoadNativeLibrary(JNIEnv* env, const std::string& path, jobject class_loader,
                                  std::string* error_msg) {
  error_msg->clear();

  // See if we've already loaded this library.  If we have, and the class loader
  // matches, return successfully without doing anything.
  // TODO: for better results we should canonicalize the pathname (or even compare
  // inodes). This implementation is fine if everybody is using System.loadLibrary.
  SharedLibrary* library;
  Thread* self = Thread::Current();
  {
    // TODO: move the locking (and more of this logic) into Libraries.
    MutexLock mu(self, *Locks::jni_libraries_lock_);
    library = libraries_->Get(path);
  }
  if (library != nullptr) {
    if (env->IsSameObject(library->GetClassLoader(), class_loader) == JNI_FALSE) {
      // The library will be associated with class_loader. The JNI
      // spec says we can't load the same library into more than one
      // class loader.
      StringAppendF(error_msg, "Shared library \"%s\" already opened by "
          "ClassLoader %p; can't open in ClassLoader %p",
          path.c_str(), library->GetClassLoader(), class_loader);
      LOG(WARNING) << error_msg;
      return false;
    }
    VLOG(jni) << "[Shared library \"" << path << "\" already loaded in "
              << " ClassLoader " << class_loader << "]";
    if (!library->CheckOnLoadResult()) {
      StringAppendF(error_msg, "JNI_OnLoad failed on a previous attempt "
          "to load \"%s\"", path.c_str());
      return false;
    }
    return true;
  }

  // Open the shared library.  Because we're using a full path, the system
  // doesn't have to search through LD_LIBRARY_PATH.  (It may do so to
  // resolve this library's dependencies though.)

  // Failures here are expected when java.library.path has several entries
  // and we have to hunt for the lib.

  // Below we dlopen but there is no paired dlclose, this would be necessary if we supported
  // class unloading. Libraries will only be unloaded when the reference count (incremented by
  // dlopen) becomes zero from dlclose.

  Locks::mutator_lock_->AssertNotHeld(self);
  const char* path_str = path.empty() ? nullptr : path.c_str();
  // 1.打开动态链接库
  void* handle = dlopen(path_str, RTLD_NOW);
  bool needs_native_bridge = false;
  if (handle == nullptr) {
    if (android::NativeBridgeIsSupported(path_str)) {
      handle = android::NativeBridgeLoadLibrary(path_str, RTLD_NOW);
      needs_native_bridge = true;
    }
  }

  VLOG(jni) << "[Call to dlopen(\"" << path << "\", RTLD_NOW) returned " << handle << "]";

  if (handle == nullptr) {
    // 检查错误信息
    *error_msg = dlerror();
    VLOG(jni) << "dlopen(\"" << path << "\", RTLD_NOW) failed: " << *error_msg;
    return false;
  }

  if (env->ExceptionCheck() == JNI_TRUE) {
    LOG(ERROR) << "Unexpected exception:";
    env->ExceptionDescribe();
    env->ExceptionClear();
  }
  // Create a new entry.
  // TODO: move the locking (and more of this logic) into Libraries.
  bool created_library = false;
  {
    // Create SharedLibrary ahead of taking the libraries lock to maintain lock ordering.
    std::unique_ptr<SharedLibrary> new_library(
        new SharedLibrary(env, self, path, handle, class_loader));
    MutexLock mu(self, *Locks::jni_libraries_lock_);
    library = libraries_->Get(path);
    if (library == nullptr) {  // We won race to get libraries_lock.
      library = new_library.release();
      libraries_->Put(path, library);
      created_library = true;
    }
  }
  if (!created_library) {
    LOG(INFO) << "WOW: we lost a race to add shared library: "
        << "\"" << path << "\" ClassLoader=" << class_loader;
    return library->CheckOnLoadResult();
  }
  VLOG(jni) << "[Added shared library \"" << path << "\" for ClassLoader " << class_loader << "]";

  bool was_successful = false;
  void* sym;
  if (needs_native_bridge) {
    library->SetNeedsNativeBridge();
    sym = library->FindSymbolWithNativeBridge("JNI_OnLoad", nullptr);
  } else {
    // 2.获取方法地址
    sym = dlsym(handle, "JNI_OnLoad");
  }
  if (sym == nullptr) {
    VLOG(jni) << "[No JNI_OnLoad found in \"" << path << "\"]";
    // 'sym'为NULL时，so库加载成功，原因参考文章[JNI原理分析]
    was_successful = true;
  } else {
    // Call JNI_OnLoad.  We have to override the current class
    // loader, which will always be "null" since the stuff at the
    // top of the stack is around Runtime.loadLibrary().  (See
    // the comments in the JNI FindClass function.)
    ScopedLocalRef<jobject> old_class_loader(env, env->NewLocalRef(self->GetClassLoaderOverride()));
    self->SetClassLoaderOverride(class_loader);

    VLOG(jni) << "[Calling JNI_OnLoad in \"" << path << "\"]";
    typedef int (*JNI_OnLoadFn)(JavaVM*, void*);
    // 3.强制类型转换成函数指针
    JNI_OnLoadFn jni_on_load = reinterpret_cast<JNI_OnLoadFn>(sym);
    // 4.调用函数
    int version = (*jni_on_load)(this, nullptr);

    if (runtime_->GetTargetSdkVersion() != 0 && runtime_->GetTargetSdkVersion() <= 21) {
      fault_manager.EnsureArtActionInFrontOfSignalChain();
    }

    self->SetClassLoaderOverride(old_class_loader.get());

    if (version == JNI_ERR) {
      StringAppendF(error_msg, "JNI_ERR returned from JNI_OnLoad in \"%s\"", path.c_str());
    } else if (IsBadJniVersion(version)) {
      StringAppendF(error_msg, "Bad JNI version returned from JNI_OnLoad in \"%s\": %d",
                    path.c_str(), version);
      // It's unwise to call dlclose() here, but we can mark it
      // as bad and ensure that future load attempts will fail.
      // We don't know how far JNI_OnLoad got, so there could
      // be some partially-initialized stuff accessible through
      // newly-registered native method calls.  We could try to
      // unregister them, but that doesn't seem worthwhile.
    } else {
      was_successful = true;
    }
    VLOG(jni) << "[Returned " << (was_successful ? "successfully" : "failure")
              << " from JNI_OnLoad in \"" << path << "\"]";
  }

  library->SetResult(was_successful);
  return was_successful;
}
```

开始的时候会去缓存查看是否已经加载过动态库，如果已经加载过，会判断上次加载的ClassLoader和这次加载的ClassLoader是否一致，如果不一致则加载失败，如果一致则返回上次加载的结果，换句话说就是不允许不同的ClassLoader加载同一个动态库。为什么这么做值得推敲。

之后会通过`dlopen`打开动态共享库。然后会获取动态库中的`JNI_OnLoad`方法，如果有的话调用之。最后会通过`JNI_OnLoad`的返回值确定是否加载成功：

[aosp/art/runtime/**java_vm_ext.cc**]

```C++
static bool IsBadJniVersion(int version) {
  // We don't support JNI_VERSION_1_1. These are the only other valid versions.
  return version != JNI_VERSION_1_2 && version != JNI_VERSION_1_4 && version != JNI_VERSION_1_6;
}
```

这也是为什么在JNI_OnLoad函数中必须正确返回的原因。

可以看到最终没有调用`dlclose`，当然也不能调用，这里只是加载，真正的函数调用还没有开始，之后就会使用`dlopen`拿到的句柄来访问动态库中的方法了。

**当ClassLoader不存在时**，参考如下源码：

[aosp/libcore/luni/src/main/java/java/lang/**Runtime.java**]

```java
/*
 * Searches for and loads the given shared library using the given ClassLoader.
 */
void loadLibrary(String libraryName, ClassLoader loader) {
    if (loader != null) {
        ...
    }

    // ClassLoader不存在的情况
    String filename = System.mapLibraryName(libraryName);
    List<String> candidates = new ArrayList<String>();
    String lastError = null;
    for (String directory : mLibPaths) {
        String candidate = directory + filename;
        candidates.add(candidate);

        if (IoUtils.canOpenReadOnly(candidate)) {
            String error = doLoad(candidate, loader);
            if (error == null) {
                return; // We successfully loaded the library. Job done.
            }
            lastError = error;
        }
    }

    if (lastError != null) {
        throw new UnsatisfiedLinkError(lastError);
    }
    throw new UnsatisfiedLinkError("Library " + libraryName + " not found; tried " + candidates);
}
```

调用`System.mapLibraryName`（与ClassLoader存在时的实现相同，可参考上文），将动态库的名称转换为`"lib" + libname + ".so"`，接着遍历`mLibPaths`得到so库，`mLibPaths`实现如下：

[aosp/libcore/luni/src/main/java/java/lang/**Runtime.java**]

```java
private final String[] mLibPaths = initLibPaths();

private static String[] initLibPaths() {
    String javaLibraryPath = System.getProperty("java.library.path");
    if (javaLibraryPath == null) {
        return EmptyArray.STRING;
    }
    String[] paths = javaLibraryPath.split(":");
    // Add a '/' to the end of each directory so we don't have to do it every time.
    for (int i = 0; i < paths.length; ++i) {
        if (!paths[i].endsWith("/")) {
            paths[i] += "/";
        }
    }
    return paths;
}
```

其取值分为两种情况：
- 32系统为/system/lib/和/vendor/lib/
- 64系统为/system/lib64/和/vendor/lib64/

通过上述内容，我们了解到了如下几点：
1. System.loadLibrary会优先查找apk中的so目录，再查找系统目录，系统目录包括：/vendor/lib(64)，/system/lib(64)
1. 不能使用不同的ClassLoader加载同一个动态库
1. System.loadLibrary加载过程中会调用目标库的JNI_OnLoad方法，我们可以在动态库中加一个JNI_OnLoad方法用于动态注册
1. 如果加了JNI_OnLoad方法，其的返回值为JNI_VERSION_1_2 ，JNI_VERSION_1_4， JNI_VERSION_1_6其一。我们一般使用JNI_VERSION_1_4即可
1. Android动态库的加载与Linux一致使用dlopen系列函数，通过动态库的句柄和函数名称来调用动态库的函数

## 参考
[深入理解 System.loadLibrary](https://pqpo.me/2017/05/31/system-loadlibrary/)

[loadLibrary动态库加载过程分析](http://gityuan.com/2017/03/26/load_library/)
