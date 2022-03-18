解决C++使用dl在主函数退出后出现SIGSEGV或double free or corruption问题
--------------------------------------------------------------------------------------



### 问题描述

在使用`dlopen`等方法加载动态库并在主函数返回后出现以下报错（或出现段错误）

```plain
1.
*** Error in `/workspace/xxx': double free or corruption (!prev): 0x00000000016064e0 ***

2.
Program terminated with signal 11, Segmentation fault.
```



### 问题原因

这个问题一般是因为在多个动态库中有多个相同的符号（比如说析构函数）导致。在我们使用`dlopen`加载动态库并卸载后，会执行定义在动态库中全局变量的析构函数，如果同时主程序也使用了这些符号（相互依赖导致）则会在主程序返回后glic清理环境时触发`double free or corruption`问题。通过gdb可以看到`SIGSEGV`发生在`glic`之内

```plain
(gdb) where
#0  0x00007ff256be018b in __cxa_finalize (d=0x2) at cxa_finalize.c:33
#1  0x0000000000000000 in ?? ()

(gdb) bt
#0  0x00007f523b2bc45b in malloc_consolidate () from /lib64/libc.so.6
#1  0x00007f523b2bd17e in _int_free () from /lib64/libc.so.6
#2  0x00007f5238e0552c in HOOPS::HI_Destroy_Memory_Pool(HOOPS::Memory_Pool*) ()
   from /workspace/libs/libhps_core.so
#3  0x00007f5238ace4b3 in HI_Reset_System(bool) () from /workspace/libs/libhps_core.so
#4  0x00007f5237d6739f in ?? () from /workspace/libs/libhps_core.so
#5  0x00007f523b27605a in __cxa_finalize () from /lib64/libc.so.6
#6  0x00007f5237569033 in ?? () from /workspace/libs/libhps_core.so
#7  0x00007ffca3f79170 in ?? ()
#8  0x00007f523c14008a in _dl_fini () from /lib64/ld-linux-x86-64.so.2
```



### 解决方案

将具有相同符号的动态库放在一起，并设置`LD_LIBRARY_PATH`保证主程序和通过`dlopen`加载的是一个库文件。

#### 屏蔽错误

也可以直接屏蔽glic对这个错误的检查，但是治标不治本，BUG还是要修。

```bash
export MALLOC_CHECK_=0
```



### 参考资料

[\*** glibc detected \*** double free or corruption message](https://www.ibm.com/support/pages/glibc-detected-double-free-or-corruption-message-information-server-datastage-job-log)

>**Problem**
>
>Randomly a error message \*** glibc detected \*** double free or corruption (out): 0x097eaac0 \*** is displayed in the job log
>
>**Cause**
>
>This error usually indicates a detection of heap corruption
>
>This message is seen in recent versions of Linux libc and GNU libc that include a malloc implementation. This is tunable via environment variables.
>
>When MALLOC_CHECK_ is set, a special implementation is used which is designed to be tolerant against simple errors, such as double calls of free() with the same argument, or overruns of a single byte.
>
>If MALLOC_CHECK_ is set to 0, any detected heap corruption is silently ignored and an error message is not generated.
>
>**Resolving The Problem**
>
>Set environment variable MALLOC_CHECK_=0

[Segmentation fault using shared library](https://stackoverflow.com/questions/1178538/segmentation-fault-using-shared-library)

> Now that you've posted `GDB` output, it's clear exactly what your problem is.
>
> You are **defining the same symbols** in `libokFrontPanel.so` and in the `libLoadLibrary.so` (for lack of a better name -- it is *so* much easier to explain things when they are named properly), **and *that* is causing the infinite recursion**.
>
> By default on UNIX (unlike on Windows) all global symbols from all shared libraries (and the main executable) go into single "loader symbol name space".
>
> Among other things, this means that if you define `malloc` in the main executable, *that* `malloc` will be called by all shared libraries, including `libc` (even though `libc` has its own `malloc` definition).
>
> So, here is what's happening: in `libLoadLibrary.so` you defined `okCUsbFrontPanel` constructor. I assert that there is also a definition of that exact symbol in `libokFrontPanel.so`. *All* calls to this constructor (by default) go to the first definition (the one that the dynamic loader first observed), even though the creators of `libokFrontPanel.so` did not intend for this to happen. The loop is (in the same order `GDB` printed them -- innermost frame on top):
>
> ```cpp
>  #1 okCUsbFrontPanel () at okFrontPanelDLL.cpp:169
>  #3 okUsbFrontPanel_Construct () from libokFrontPanel.so
>  #2 okUsbFrontPanel_Construct () at okFrontPanelDLL.cpp:1107
>  #1 okCUsbFrontPanel () at okFrontPanelDLL.cpp:169
> ```
>
> The call to constructor from `#3` was intended to go to symbol #4 -- `okCUsbFrontPanel` constructor *inside* `libokFrontPanel.so`. Instead it went to previously seen definition inside `libLoadLibrary.so`: you "preempted" symbol #4, and thus formed an infinite recursion loop.
>
> Moral: do not define the same symbols in multiple libraries, unless you understand the rules by which the runtime loader decides which symbol references are bound to which definitions.
>
> EDIT: To answer 'EDIT2' of the question:
> Yes, the call to `_ZN16okCUsbFrontPanelC1Ev` from `okUsbFrontPanel_Construct` is going to the definition of that method inside your `okFrontPanelDLL.cpp`. It might be illuminating to examine `objdump -d okFrontPanelDLL.o`

