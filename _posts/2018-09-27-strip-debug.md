---
layout: "post"
title: "C/C++目标文件运行段和debug段分离"
categories: "blog"
tags: ['gdb']
published: true
comments: true
script: [post.js]
excerpted: |
---

* TOC
{:toc}

在C/C++调试过程中不免会遇到crash的情况，这时候一个有debug section的可执行文件+core文件可以方便定位问题。但是，一个没有去除debug section和其他一些符号段的ELF会很大，所以往往为了节约空间，在编译时不会带`-g`，甚至strip掉其他的不需要的段，但是这也意味着调试的时候无法精确定位。

其实，上面两个痛点GDB早就考虑到了，可以通过`strip`+`objcopy`轻松的做到轻量级上线，并且又保留debug能力。

具体的步骤为：

1. `$ objcopy --only-keep-debug <obj> <obj>.debug` ： 将`obj`的debug相关的sections拷贝到`obj.debug`
2. `$ strip --strip-unneeded <obj>` ：将`obj`中的重定向期间不需要的section（例如：debug段, symbol段）去除
3. `$ objcopy --add-gnu-debuglink=<obj>.debug <obj>` ：在`<obj>`文件中加入.gnu_debuglink section，它记录了debug section存在哪个文件里（并且通过CRC保证是出自同一个build）。这里需要注意的是，`<obj>.debug`可以是带路径的（相对/绝对），但是建议使用文件名，因为这样的话gdb会去几个默认路径去寻找这个文件，包括：

    - `<obj>`所在目录
    - `<obj>`下的*debug*目录
    - */usr/lib/debug*

另外，如果待分析的core是产生自一个被`dlopen()`（动态加载）的库（称为libfoo.so），那么在分析这个core的时候`gdb`的目标对象是这个libfoo.so。这个文件必须是和出问题的环境所处的位置一样，并且该位置下需要有libfoo.so.debug.
