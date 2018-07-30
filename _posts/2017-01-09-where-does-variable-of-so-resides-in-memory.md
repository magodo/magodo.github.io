---
layout: "post"
title: "Where does variables of .so resides in memory?"
categories: blog
tags: ['linux']
published: true
comments: true 
script: [post.js]
---

* TOC
{:toc}

工作中，有时候发生了一些意想不到的结果，这时候有一种可能性是内存越界改变了某些变量的值。这时候有一个问题困扰我的是共享库中的变量存在哪里（听上去是个很蠢的问题）。

在Linux下进程的典型内存分布中，有一块处于 *stack* 和 *block* 之间的内存区域是专门给这些共享库保存其代码段，数据段之类的，因此似乎可以得到“共享库的数据都存在这个区域中”的结论。然而，有一些例外情况需要考虑，下文就此进行分类讨论。

## 1 共享库export的"全局变量"是真正的变量

当link这些库的程序不直接访问库中的全局变量，也就是说该全局变量不会出现在编译后的可执行文件中。那么这些全局变量是位于当前进程虚拟地址空间中存放该库的.data/.rodata段.

如果link这些库的程序会对库中的全局变量直接进行访问，那么这个全局变量会在link的时候重定位，并且出现在可执行文件的.data段。

举个例子：

假设有以下库文件：

*mylib.h*

    #ifndef MY_LIB_H
    #define MY_LIB_H

    extern int global_i;

    void echo();

*mylib.c*

    #include "mylib.h"
    #include <stdio.h>

    int global_i = 100;

    void echo()
    {
        printf("global_i = %d\n", global_i);
    }

编译成 .so: `# gcc -g --shared -fPIC mylib.c -o libmylib.so`

假设又有两个程序 *a.c* 和 *b.c* 会分别link *libmylib.so*：

*a.c*

    #include "mylib.h"
    #include <stdlib.h>

    void main()
    {
        echo();
        pause();
    }

*b.c*

    #include "mylib.h"
    #include <stdlib.h>

    void main()
    {
        global_i = 200;
        echo();
        pause();
    }

编译成可执行文件: 

    #  gcc -g -L . a.c -lmylib -Wl,-rpath=. -o a.out
    #  gcc -g -L . b.c -lmylib -Wl,-rpath=. -o b.out

分别执行并通过gdb调试，打印 `global_i` 的地址。

对于 *a.out*:

    (gdb) p &global_i
    $1 = (int *) 0x7ffff7dd9018 

查看进程的各个section的内存分布：

    (gdb) info files
    Symbols from "/tmp/test/a.out".
    Unix child process:
            Using the running image of child process 58183.
            While running this, GDB does not access memory from...
    Local exec file:
            `/tmp/test/a.out', file type elf64-x86-64.
            Entry point: 0x400550
            ...
            0x00007ffff7dd9010 - 0x00007ffff7dd901c is .data in ./libmylib.so
            ...

可见，在 *a.out* 中 `global_i` 位于 *libmylib.so* 的.data section 所在的区域，该区域属于内存中的 *Memory mapped region for shared libraries*。

用同样的方式可以看到 *b.out* 中的 `global_i` 位于自己的.data段。

## 2 共享库export的"全局变量"不是真正的变量

例如C标准库中的 `errno`，在线程出现之前，它是一个真正的全局变量；而在线程出现以后，为了避免 `errno` 运用于多线程的场景，标准库中通过 TLS(thread local storage) 定义了一个返回 `errno` 地址的函数，该地址对应于不同线程有不同的值。例如，在 Ubuntu 12.04 LTS 发行版中的glibc：

    # /lib/x86_64-linux-gnu/libc.so.6
    GNU C Library (Ubuntu EGLIBC 2.15-0ubuntu10.15) stable release version 2.15, by Roland McGrath et al.

`errno` 的定义如下：

    # ifndef __ASSEMBLER__
    /* Function to get address of global `errno' variable.  */
    extern int *__errno_location (void) __THROW __attribute__ ((__const__));

    #  if !defined _LIBC || defined _LIBC_REENTRANT
    /* When using threads, errno is a per-thread value.  */
    #   define errno (*__errno_location ())
    #  endif
    # endif /* !__ASSEMBLER__ */
    #endif /* _ERRNO_H */

可见，`__errno_location()` 返回的是每个线程存储自己的 `errno` 的地址。那么，在GDB中我们应该如何访问这个地址呢？

假设有以下程序：

    #include <stdio.h>
    #include <errno.h>
    #include <inttypes.h>
    #include <stdint.h>
    #include <stdlib.h>

    int main()
    {
        int i;
        printf("Address of errno: %p\n", __errno_location());
        abort();
    }

如果待测试程序是在运行中，那么可以在gdb中执行某个函数，在这里我们先在 `abort()` 处设置断点，然后执行程序。在断点处执行 `__errno_location()`:

    Address of errno: 0x7ffff7fd96a0
    (gdb) b 18
    (gdb) r
    (gdb) p __errno_location()
    $1 = -134375776
    (gdb) p (int*)__errno_location()                                                                                                         
    $3 = (int *) 0xfffffffff7fd96a0
    (gdb) printf "%p\n", __errno_location()
    0xfffffffff7fd96a0
    
程序中执行 `__errno_location()` 返回的地址是 `0x7ffff7fd96a0`。而在GDB中，用 `p` 标识符输出的地址却是 `0xfffffffff7fd96a0`，是个非法地址。这点我没有弄清楚原因，本来认为可能和虚拟内存空间不是64位的有关。查看 `/proc/cpuinfo`:

    # cat /proc/cpuinfo  | grep address
    address sizes   : 40 bits physical, 48 bits virtual

虚拟内存空间用了48位。可是即使将将[63-48]位清0，这个地址还是不对：`0xfffff7fd96a0`. 所以最后都没搞明白。。。

# Reference

[1] [Debugging thread variables](https://www.technovelty.org/linux/debugging-__thead-variables-from-coredumps.html)
