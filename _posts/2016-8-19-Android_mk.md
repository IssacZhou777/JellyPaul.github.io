---
layout: post
title:  "NDK 学习笔记之Android.mk"
date:   2016-08-19 15:17:20 +0800
categories: Android
---


> **前言：**  Android.mk 编译文件是用来向 Android NDK描述你的 C,C++源代码文件的。 这篇文章主要参考， 更确切的说应该是翻译自<http://android.mk>文档中的介绍。主要描述了Android.mk文件的结构， 一些变量（或者翻译成属性）的含义。文中提到**变量** 一词是对**variables**的翻译， 指的是`LOCAL_PATH `这一类定义，在一些地方可能不太合宜， 请见谅。英文能力好的建议直接看英文文档。

## 一、简单了解一下Android.mk的写法

一个简单的Android.mk文件如下：

```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE    := hello-jni
LOCAL_SRC_FILES := hello-jni.c

include $(BUILD_SHARED_LIBRARY)

```

**解析：**

* `LOCAL_PATH := $(call my-dir)`  ：

一个Android.mk文件必须在开头定义LOACL_PATH变量。 该变量用于定位源文件。在这个例子中用到的`my-dir`是编译系统定义的宏方法， 返回文件的当前路径。

* `include $(CLEAR_VARS)`

`CLEAR_VARS ` 是编译系统提供的变量， 说明指定的GNU文件将会清除出来`LOCAL_PATH` 外的其他`LOCAL_XXX`变量。（比如LOCAL_MODULE, LOCAL_SRC_FILES, LOCAL_STATIC_LIBRARIES）。 需要这一步操作是因为所有编译控制文件的解析都是由一个GNU Make 执行的， 也就是说所有的变量都是全局的。

* `LOCAL_MODULE    := hello-jni `

`LOCAL_MODULE ` 变量定义每一个`MODULE` 名称， 每个`MODULE `**名称必须唯一并且不能带空格**。需要注意的是， 编译系统会对生成的文件自动添加必要的前缀和后缀, 比如， 一个`share library` 模块的名字是`foo`， 则生成的文件名为`libfoo.so`。如果你的模块名为`libfoo`, 此时编译系统不会添加前缀，生成的文件名还是`libfoo.so`。

* `LOCAL_SRC_FILES := hello-jni.c`

`LOCAL_SRC_FILES` 变量必须列出一组c/c++源文件， 这些文件将会编译并集成到一个module中。**需要注意的是：**你不需要在这里列出头文件和包含文件， 因为编译系统自动计算依赖文件。

**另外需要注意：** C++源文件的默认扩展名为`.cpp`, 当然也会存在其他情况使用不一样的扩展名， 这就需要通过`LOCAL_CPP_EXTENSION `变量设置扩展名， 但是定义的时候别忘了扩展名前面的点（.）, `.cxx` 是有效的扩展名， 而`cxx`不是。


* `include $(BUILD_SHARED_LIBRARY)`

`BUILD_SHARED_LIBRARY ` 是编译系统提供的变量，说明GNU Makefile脚本负责收集管理上一个`include $(CLEAR_VARS)` 之后的所有`LOCAL_XXX`变量的信息。并指定编译类型。`BUILD_SHARED_LIBRARY `将指定编译生产一个share library, 另外还有一个变量`BUILD_STATIC_LIBRARY` 指定编译长生一个静态库（static library）。

**Reference:**

以上列出的变量是在Android.mk文件中必须定义的， 除此之外， 也可以定义其他变量。 NDK编译环境已经储备了如下变量：

* 以`LOCAL_`开头的变量， 比如`LOCAL_MODULE `
* 以`PRIVATE_, NDK_ or APP_ ` 开头的变量（在内部使用）
* 一些小写的名字（也是在内部使用， 比如`my_dir`）

如果需要在Android.mk中定义你自己的变量名称，建议使用`MY_` 前缀，举个例子：

```
MY_SOURCES := foo.c
ifneq ($(MY_CONFIG_BAR),)
MY_SOURCES += bar.c
endif

LOCAL_SRC_FILES += $(MY_SOURCES)

```


### 二、NDK 提供的变量：

* **`CLEAR_VARS`**

表明一个构建脚本将清除module描述中定义的几乎所有`LOCAL_`变量。你必须在开始一个新的`module`之前设置。用法如下：

`include $(CLEAR_VARS) `

* **`BUILD_STATIC_LIBRARY`**

表明该构建脚本会集成module中定义的所有LOCAL_XXX变量中的信息，并将源码构建成一个shared library.需要注意在包含这个文件之前， 必须定义`LOCAL_MODULE, LOCAL_SRC_FILES` 变量。 用法如下：

`include $(BUILD_SHARED_LIBRARY)`

这样定义后， 最终会生成一个名为`lib$(LOCAL_MODULE).so`的文件。

* **`BUILD_STATIC_LIBRARY`**

该变量与`BUILD_STATIC_LIBRARY `类似， 它表明将源码构建成一个static library(静态库)。 静态库不需要拷贝到项目的包中， 但是可以用于构建shared library（共享库）。用法如下：

`include $(BUILD_STATIC_LIBRARY)`

这样定义后， 执行脚本将生成一个名为`lib$(LOCAL_MODULE).a`的文件

* **`PREBUILT_SHARED_LIBRARY`**

该变量表面构建脚本用于指定一个预编译的共享库（shared library）.与`BUILD_STATIC_LIBRARY ` 和 `BUILD_STATIC_LIBRARY ` 不同，.mk脚本中的`LOCAL_PREBUILTS ` 变量的值为一个共享库（shared library）的单一路径 而不是源文件， 比如`foo/libfoo.so`。

你可以在其他module中通过`LOCAL_PREBUILT`变量引用该共享库。

* **`PREBUILT_STATIC_LIBRARY`**

该变量与`PREBUILT_SHARED_LIBRARY `， 只不过它针对的是静态库（static librarry）

* **`TARGET_ARCH`**

指定CPU架构类型， 在任何兼容ARM架构的构建中， 它的值为'arm'。

* **`TARGET_PLATFORM`**

指定Android 平台。 比如值为'android-3' 对应Android 1.5系统镜像。 

* **`TARGET_ARCH_ABI`**

指定CPU+ABI. 当前有两种值：
	* armeabi （针对ARMv5TE）
	* armeabi-v7a

注意： 直到Android NDK 1.6_r1版本， 该变量都简单定义为'arm'. 所有基于ARM的ABI都会将`TARGET_ARCH`的值设为'arm'， 但是可能会有不同的`TARGET_ARCH_ABI `属性值。

* **`TARGET_ABI`**

该变量指定目标平台和ABI, 通常定义为`$(TARGET_PLATFORM)-$(TARGET_ARCH_ABI) `， 当你想真对一个特定的系统镜像进行测试时， 这个属性就非常有用了。

该变量的默认值为`'android-3-armeabi'`（直到Android NDK 1.6_r1版本， 该变量都简单定义为`'android-3-arm'`）

### 三、NDK 提供的方法宏

* **my-dir** 

返回最后包含的那个Makefile 的路径， 通常情况下返回的是当前Android.mk 的路径。这个方法在Android.mk开头定义LOCAL_PATH时非常有用。用法示例：

`LOCAL_PATH := $(call my-dir)`

**特别需要注意：**由于GNU Make 的工作原理， 该方法返回的是构建脚本中 **最后包含的Makefile文件**的路径，不能在包含另一文件后调用`my-dir`方法。

举个例子：

```
LOCAL_PATH := $(call my-dir)

... declare one module

include $(LOCAL_PATH)/foo/Android.mk

LOCAL_PATH := $(call my-dir)

... declare another module

```

这个例子的问题在于在第二次调用`my-dir`时返回的是`$PATH/foo` 而不是`$PATH`。所以，建议在处理完其他操作后再进行包含操作。

* **all-subdir-makefiles**

返回当前`my-dir`目录下的所有子目录中的Android.mk文件。 比如当前有这样一个目录结构：

	sources/foo/Android.mk
	sources/foo/lib1/Android.mk
   	sources/foo/lib2/Android.mk
   	
如果其中`sources/foo/Android.mk`中添加了：

`include $(call all-subdir-makefiles)`

那么它将自动包含子目录下的Android.mk文件`sources/foo/lib1/Android.mksources/foo/lib2/Android.mk`


* **this-makefile**

返回当前Makefile文件的路径。

* **parent-makefile** 

返回包含当前文件的父Makefile文件的路径。

* **grand-parent-makefile**

你猜...

* **import-module** 

该方法可以通过名字找到并且引用其它module的Android.mk。 用法示例：

`$(call import-module,<name>)`

### 四、模块描述变量（Module-description variables）

以下的变量用于描述module， 你必须在`include$(CLEAR_VARS)` 和`include$(BUILD_XXXX)` 之间定义。

* **`LOCAL_PATH`**

该变量用于给出当前文件的路径， 你必须在Android.mk文件的开头定义：

`LOCAL_PATH := $(call my-dir)`

该变量不会被$(CLEAR_VARS)清除，所以在每一个Android.mk中都需要定义。

* **LOCAL_MODULE**

该变量定义module的名字，module名必须唯一， 并且不能带空格， 切必须定义在`include $(BUILD_XXX)` 之前。默认情况下， module名将决定生成的文件名。

然而你仍然可以通过**LOCAL_MODULE_FILENAME**来修改生成的文件名。

* **`LOCAL_MODULE_FILENAME`** 

该变量是可选的， 允许你修改生成文件的文件名。用法示例：

```
LOCAL_MODULE := foo-version-1
LOCAL_MODULE_FILENAME := libfoo
```

注意：你不需要设置路径和扩展名， 编译系统会自动处理。


* **`LOCAL_SRC_FILES`** 

该变量用于指定用于编译的源文件列表。只有列出的文件会传递给编译器， 然后编译系统会自动计算依赖。

需要注意的是，列出的源文件都是相对于`LOCAL_PATH`的相对路径。另外在路径中使用Unix风格的(／)， windows风格的（\）可能会无效。

* **`LOCAL_CPP_EXTENSION`**

该变量为可选变量， 用于设置源文件的扩展名。默认情况下源文件的扩展名为`.cpp`。 使用示例：

`LOCAL_CPP_EXTENSION := .cxx`

在NDK r7之后，你可以设置多个后缀名：

`LOCAL_CPP_EXTENSION := .cxx .cpp .cc`

* **`LOCAL_CPP_FEATURES`**

该变量是可选变量。 用于设置代码引用的C++特性。比如要声明你的代码用到RTTI(RunTime Type Information), 可以设置：

`LOCAL_CPP_FEATURES := rtti`

声明使用C++异常：

`LOCAL_CPP_FEATURES := exceptions`

如果两个都需要：

`LOCAL_CPP_FEATURES := rtti exceptions`

该变量的作用在于编译源码时启用正确的编译器/链接器。

强烈建议使用这个变量配置C++ feature， 而不是在直接在`LOCAL_CPPFLAGS`中定义`-frtti，-fexceptions`


* **`LOCAL_C_INCLUDES`**

该变量（可选）用于配置一个相对于NDK根目录的路径列表。在编译源码时，这些路径将会添加到包含的检索路径中。用法示例：

`LOCAL_C_INCLUDES := sources/foo` 

或者：

`LOCAL_C_INCLUDES := $(LOCAL_PATH)/../foo`

这些配置需要在**LOCAL_CFLAGS / LOCAL_CPPFLAGS**之前。LOCAL_C _INCLUDES 配置的路径， 在通过ndk-gdb调试native时也会自动使用。

* **`LOCAL_CFLAGS`**

该变量用于在编译源文件时传递编译器的flag集。 

这在指定宏定义和编译器设置的时候非常有用。

需要重点提一下的是： 不要在Android.mk中尝试修改optimization/debugging等级， 在你指定`Application.mk`的信息时， 这些会自动处理， 并在debugging阶段让NDK产生有用的文件。

需要注意： 在android-ndk-1.5_r1版本中，定义的flag支队C文件有效，对C++ 无效。为了兼容所有编译版本， 你可以通过`LOCAL_CPPFLAGS` 指定C++文件的flag.

该变量也可以指定额外的包含路径，比如：

` LOCAL_CFLAGS += -I<path>` 

然而更推荐使用` LOCAL_C_INCLUDES`变量， 这样的话， 这些定义的路径在ndk-gdb调试的时候也可以用到。

* **`LOCAL_CXXFLAGS`**
 
该变量是`LOCAL_CPPFLAGS `的别名， 不建议使用这个变量， 因为在以后的NDK发布版本中会取消掉。

* **`LOCAL_CPPFLAGS`**

参考** LOCAL_CFLAGS** 的作用， 这个变量设置的内容只对C++文件有效。

* **`LOCAL_STATIC_LIBRARIES`**

该变量用于指定需要连接到该模块的静态库（static library），即指定编译目标为`BUILD_STATIC_LIBRARY`编译的库。该变量只能在共享库module中使用。

* **`LOCAL_SHARED_LIBRARIES`**

该变量列出该模块依赖的共享库， 这些信息在连接时是必要的， 而且也需要将这些信息嵌入到生成文件中。

* **`LOCAL_WHOLE_STATIC_LIBRARIES`**

该变量是`LOCAL_STATIC_LIBRARIES `的变体， 表示该模块用于连接器（Linker）的`"whole archives"`属性。 具体可以参考GNU 连接器文档中的`--whole-archive` 标示。

该变量通常用于静态库之间存在循环依赖的情况。需要注意的是， 在构建共享库（shared library）时， 这回强制将所有该变量中添加的对象文件都添加到最终的二进制文件中。然而这在构建可执行文件的时候是不对的。

* **`LOCAL_LDLIBS`**

该变量用于指定连接器的flag.  这对于指定需要连接的系统库时非常有方便， 只需要在对应的系统名字前添加前缀`'l'` 即可， 比如下面的这个例子就是告诉连接器生成一个在加载时连接到`/system/lib/libz.so`模块。

`LOCAL_LDLIBS := -lz`

了解更多可以用于NDK的系统库， 可以查看docs/STABLE-APIS.html

* **`LOCAL_ALLOW_UNDEFINED_SYMBOLS`**

默认情况下， 在尝试构建一个共享库时， 如果用到一个为定义的引用， 会导致`undefined symbol`错误， 这样可以很方便的定位到Bug所在。

然而， 如果处于某些原因你想取消这种检测， 就可以通过设置改变的值为true实现。 这样做的话， 需要注意在加载对应库的时候共享库时可能会失败。

* **`LOCAL_ARM_MODE`**

默认情况下， ARM 目标二进制文件会以`'thumb'`模式创建， 它的架构为16位。 如果你想强制配置目标文件的模式是32位架构的`'arm'`， 你可以通过这个变量定义模式：

`LOCAL_ARM_MODE := arm`

然而， 你也可以只指定特定的源文件构建为ARM模式， 只需要在源文件文件名后面加上`.arm`后缀。例如：

`LOCAL_SRC_FILES := foo.c bar.c.arm`

其中foo.c会以`LOCAL_ARM_MODE` 定义的模式构建， bar.c会以AMR模式构建。

需要注意：在Appplication.mk中设置`APP_OPTIM`为`'debug'`也会强制改变ARM二进制文件的创建， 这是因为工具中的debugger对于thumb代码并没有做到很好的支持。

* **`LOCAL_ARM_NEON`**

将该变量的值设为true，则可以在C/C++源文件中使用高级ARM架构扩展结构SIMD（比如NEON）中的内置方法， NEON指令在汇编文件中。

你应该只在目标架构设为对应于ARMv7指令集的`'armeabi-v7a' ABI`的时候定义该变量。 需要注意不是所有基于ARMv7的CPU都支持NEON指令集， 所以你需要在运行时监测是否可用。想要了解更多可以参考：`docs/CPU-ARM-NEON.html`， `docs/CPU-ARM-NEON.html`.

你可以指定顶特定的源文件使用NEON构建， 只需要在源文件名后添加后缀`.neon`即可， 示例：

`LOCAL_SRC_FILES = foo.c.neon bar.c zoo.c.arm.neon`

注意， 如果你既需要使用`.arm又`需要用`.neon`，`.neon`必须位于`.arm` 之后。

* **`LOCAL_DISABLE_NO_EXECUTE`**

Android NDK r4开始支持`NX bit` 安全特性。

改特性默认是激活的， 但是如果你真的需要将其取消， 可以设置该变量为true。

注意， 该特性不会修改ABI， 而且仅在内核针对ARMv6＋ 的CPU设备上有效。

更多信息请参考：<http://en.wikipedia.org/wiki/NX_bit>, <http://www.gentoo.org/proj/en/hardened/gnu-stack.xml>

* **`LOCAL_DISABLE_RELRO`**

默认情况下， NDK编译构会以只读迁移（read-only relocations）和GOT保护（） 构建。这回命令运行时连接器在迁移后标记一块内存区作为只读区，最大限度保证安全性。

该特性默认是激活的， 你可以设置该变量为true来取消。

注意：该保护尽在较新的Android设备（Jelly Bean之后版本）上有效，代码依然会在老版本上运行。

更多信息可以参考： <http://isisblogs.poly.edu/2011/06/01/relro-relocation-read-only/> , <http://www.akkadia.org/drepper/nonselsec.pdf>  (section 6)

* **`LOCAL_EXPORT_CFLAGS`**

该变量用于定义一系列C/C++编译flag, 这些flag会添加到调用这个库的module的LOCAL_CFLAGS变量中。举个例子：

module foo:

```
include $(CLEAR_VARS)
LOCAL_MODULE := foo
LOCAL_SRC_FILES := foo/foo.c
LOCAL_EXPORT_CFLAGS := -DFOO=1
include $(BUILD_STATIC_LIBRARY)

```

module bar:

```
include $(CLEAR_VARS)
LOCAL_MODULE := bar
LOCAL_SRC_FILES := bar.c
LOCAL_CFLAGS := -DBAR=2
LOCAL_STATIC_LIBRARIES := foo
include $(BUILD_SHARED_LIBRARY)

```

在编译bar.c时 变量中的flag`-DBAR=2` 也会传递到编译器， 但是需要注意， 在编译foo.c时不会。

* **`LOCAL_EXPORT_CPPFLAGS`**

与`LOCAL_EXPORT_CFLAGS` 类似， 只不过是针对C++的flag.

* **`LOCAL_EXPORT_C_INCLUDES`**

与`LOCAL_EXPORT_CFLAGS` 类似， 只不过是针对C 文件的包含路径.

* **`LOCAL_EXPORT_LDLIBS`**

与`LOCAL_EXPORT_CFLAGS` 类似， 只不过是针对连接器的flags。需要注意， 导入的编译器flag会添加到module的LOCAL_LDLIBS变量中。该变量一个典型的用法是， 当一个静态库依赖于一个系统库时，可以利用`LOCAL_EXPORT_LDLIBS `导出依赖。 举个例子：

```
include $(CLEAR_VARS)
LOCAL_MODULE := foo
LOCAL_SRC_FILES := foo/foo.c
LOCAL_EXPORT_LDLIBS := -llog
include $(BUILD_STATIC_LIBRARY)

include $(CLEAR_VARS)
LOCAL_MODULE := bar
LOCAL_SRC_FILES := bar.c
LOCAL_STATIC_LIBRARIES := foo
include $(BUILD_SHARED_LIBRARY)

```

在上例中， libbar.so构建时会带有-llog，说明该module依赖于系统的logging库。

* **`LOCAL_SHORT_COMMANDS`**

当你的module有大量的源文件或者库依赖时， 可以将这个变量设为true.这回强制编译系统使用中间列表文件， 并且在使用时以` @$(listfile)`语法来引用库和静态连接。这对于windows系统会非常有用， 因为windows系统的命令行有最长8191个字符的限制。

需要注意设置除了true以外的任何值都会回复到默认的行为。当然你也可以在Application.mk中设置`APP_SHORT_COMMANDS`来强制所有module进行这种短命令的行为。

这个特性会使编译变慢， 所以不建议开启。

* **`LOCAL_FILTER_ASM`**

该变量用于定义一个Shell 指令， 用于过滤来自于或者创建于LOCAL_SRC_FILE定义的汇编文件。在定义后， 就会：

   * 任何C/C++源文件将创建为一个临时的的汇编文件， 而不是编译成目标文件
   * 任何临时汇编文件以及LOCAL_SRC_FILE列出的汇编文件都会被传递到LOCAL_FILTER_ASM定义的指令， 再生成新的汇编文件。
   * 这些过滤后的汇编文件再编译成目标文件， 比如.o文件

换句话说： 如果你有：

```
LOCAL_SRC_FILES  := foo.c bar.S
LOCAL_FILTER_ASM := myasmfilter
```

则会发生：

``` shell
foo.c --1--> $OBJS_DIR/foo.S.original --2--> $OBJS_DIR/foo.S --3--> $OBJS_DIR/foo.o
bar.S                                 --2--> $OBJS_DIR/bar.S --3--> $OBJS_DIR/bar.o
```

其中1是对应编译器(compiler)， 2是过滤操器(filer)， 3是汇编处理器(assembler)。 过滤器必须是一个独立的shell指令， 并且第一个参数为输入文件， 第二个参数为输出文件， 比如：

``` shell
myasmfilter $OBJS_DIR/foo.S.original $OBJS_DIR/foo.S
myasmfilter bar.S $OBJS_DIR/bar.S

```

* **`NDK_TOOLCHAIN_VERSION`**

定义4.4.3 或者 4.6 作为GCC编译版本， 默认为4.6。

### 五、几种典型的.mk文件写法

#### 1. 构建一个简单的APK

```
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
   
# Build all java files in the java subdirectory
LOCAL_SRC_FILES := $(call all-subdir-java-files)
   
# Name of the APK to build
LOCAL_PACKAGE_NAME := LocalPackage
   
# Tell it to build an APK
include $(BUILD_PACKAGE)
```

#### 2. 构建一个依赖于.jar包店APK

```
  LOCAL_PATH := $(call my-dir)
  include $(CLEAR_VARS)
   
  # List of static libraries to include in the package
  LOCAL_STATIC_JAVA_LIBRARIES := static-library
   
  # Build all java files in the java subdirectory
  LOCAL_SRC_FILES := $(call all-subdir-java-files)
   
  # Name of the APK to build
  LOCAL_PACKAGE_NAME := LocalPackage
   
  # Tell it to build an APK
  include $(BUILD_PACKAGE)
```


#### 3. 构建一个用platform key签名的APK

```
  LOCAL_PATH := $(call my-dir)
  include $(CLEAR_VARS)
   
  # Build all java files in the java subdirectory
  LOCAL_SRC_FILES := $(call all-subdir-java-files)
   
  # Name of the APK to build
  LOCAL_PACKAGE_NAME := LocalPackage
   
  LOCAL_CERTIFICATE := platform
   
  # Tell it to build an APK
  include $(BUILD_PACKAGE)
```

#### 4. 构建一个用特定vendor key签名的APK

```
  LOCAL_PATH := $(call my-dir)
  include $(CLEAR_VARS)
   
  # Build all java files in the java subdirectory
  LOCAL_SRC_FILES := $(call all-subdir-java-files)
   
  # Name of the APK to build
  LOCAL_PACKAGE_NAME := LocalPackage
   
  LOCAL_CERTIFICATE := vendor/example/certs/app
   
  # Tell it to build an APK
  include $(BUILD_PACKAGE)
```


#### 5.添加一个预编译的APK

```
  LOCAL_PATH := $(call my-dir)
  include $(CLEAR_VARS)
   
  # Module name should match apk name to be installed.
  LOCAL_MODULE := LocalModuleName
  LOCAL_SRC_FILES := $(LOCAL_MODULE).apk
  LOCAL_MODULE_CLASS := APPS
  LOCAL_MODULE_SUFFIX := $(COMMON_ANDROID_PACKAGE_SUFFIX)
   
  include $(BUILD_PREBUILT)
```

#### 6.添加一个静态的Java库

```
  LOCAL_PATH := $(call my-dir)
  include $(CLEAR_VARS)
   
  # Build all java files in the java subdirectory
  LOCAL_SRC_FILES := $(call all-subdir-java-files)
   
  # Any libraries that this library depends on
  LOCAL_JAVA_LIBRARIES := android.test.runner
   
  # The name of the jar file to create
  LOCAL_MODULE := sample
   
  # Build a static jar file.
  include $(BUILD_STATIC_JAVA_LIBRARY)
```

### Android.mk 属性变量总结

在Android.mk中， 常见的变量可以分为一下几类：

* **LOCAL_**  这一类变量在每个module中设置， 并且会被`include $(CLEAR_VARS)` 清除， 在module中使用的大部分变量都是`LOCAL_`
* **PRIVATE_** 这一类变量属于`make-target-specific`变量，  这只在module的命令中有效。
* **HOST_** 和 **TARGET_** 这一类变量包含了针对于host和编译目标的目录和定义。不要在makefile中设置这一类变量。
* **BUILD_** 和 **CLEAR_VARS** 这一类变量包含了定义好的makefile模版， 在makefile中include的即可。 
* 任何其他变量名在Android.mk中都是可用的。 但是需要注意， 这是一个非递归构建系统， 你定义的变量极有可能被其他Android.mk文件改变， 当执行你的module/rule时可能就不一样了。

更多属性具体的描述请参考： <http://android.mk> 中的表格。

### 参考文献

Secrets of Android.mk <http://android.mk>