---
layout: "post"
title: "进程的输入参数"
categories: blog
tags: ['program']
published: true
comments: true 
script: [post.js]
---

* TOC
{:toc}

# 0.

以下程序用于输出进程的输入参数：

**文件名：echoarg.c**

{%highlight CPP linenos%}
#include <stdio.h>

int main(int argc, char *argv[])
{
    int i;

    for (i = 0; i < argc; i++)
    {
        printf("Arg[%d]: %s\n", i, argv[i]);
    }
}
{%endhighlight%}

将其编译成可执行文件：

    $ gcc echoarg.c -o echoarg

# 1. 程序中的exec传递参数

以`execl`为例，该内核调用的输入参数包括*pathname*, *arg0*, *arg1*... 其中的*argN*会存在栈上并传入进程，对应了*argv[N]*. 则对于以下代码：

**文件名：main.c**

{%highlight CPP linenos%}
#include <sys/wait.h>
#include <sys/types.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

int main()
{
    pid_t pid;

    if ((pid = fork()) < 0)
    {
        perror("fork error");
        exit(-1);
    }
    if (pid == 0)            // child
    {
        if (execl("./echoarg", "arg0", "arg1", (char*)0) == -1)
        {
            perror("execl error");
            exit(-2);
        }
    }
    if (waitpid(pid, NULL, 0) < 0)
    {
        perror("waitpid error");
        exit(-3);
    }
    return 0;
}
{%endhighlight%}

输出的是：

    Arg[0]: arg0
    Arg[1]: arg1

不过，按照shell的惯例（见下一节），*arg0*一般会设置为执行文件的文件名分量或者是文件的完整路径。

# 2. 命令行传递参数

我们在命令行上执行：

    $ ./echoarg arg0 arg1

    Arg[0]: ./echoarg
    Arg[1]: arg0
    Arg[2]: arg1

这里`Arg[0]`输出`./echoarg`的原因是，shell首先会`fork`一个子进程，然后该子进程`execlp`命令行上的第一个参数`./echoarg`。__在构造`execlp`的参数的过程中，shell会将命令行上的第一个参数的文件名分量或者是路径（取决于shell的实现），作为*arg0*__, 后续的cmdline的参数则作为*arg1*, *arg2*, ...

# 3. 解释文件传递参数

假设我们有以下解释器文件：

**文件名：testinterp**

    #! ./echoarg foo

则考虑以下两种情况：通过程序中调用`exec`执行该解释器文件 和 直接在命令行上执行。

## 3.1 调用exec执行解释器文件

改写上面的**main.c**

{%highlight CPP linenos%}
#include <sys/wait.h>
#include <sys/types.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

int main()
{
    pid_t pid;

    if ((pid = fork()) < 0)
    {
        perror("fork error");
        exit(-1);
    }
    if (pid == 0)            // child
    {
        if (execl("./testinterp", "arg0", "arg1", (char*)0) == -1)
        {
            perror("execl error");
            exit(-2);
        }
    }
    if (waitpid(pid, NULL, 0) < 0)
    {
        perror("waitpid error");
        exit(-3);
    }
    return 0;
}
{%endhighlight%}

编译运行：

    $ ./main
    Arg[0]: ./echoarg
    Arg[1]: foo
    Arg[2]: ./testinterp
    Arg[3]: arg1


从结果上可以看到，`arg0`不见了! 这是因为`execl`调用过程中，内核首先会尝试直接执行第一个输入参数(`./testinterp`):

1. 如果该文件是二进制文件，则`execl`会以此代替当前进程并且不返回
2. 如果不是，则会返回一个错误，`execl`会认为该文件是一个解释器文件，它会查看该文件启示行是否以*#!*开头:
    1. 如果是，则会以后面的程序(`./echoarg`)作为解释器，并允许一个可选的输入参数(`foo`)
    2. 如果不是，则会以*/bin/sh*作为解释器

这里的情况属于第二类中的第一类。此时，内核会以如下方式重新构造由`execl`传入的输入参数：

**原始的输入参数**

arg0  arg1

**调整后的参数**

./echoarg  foo  ./testinterp  arg1

(1)        (2)  (3)           (4)

1. 内核会以解释器文件中指定的*解释器*作为*arg0*
2. 如果解释器文件中有*可选参数*，则将其作为*arg1*
3. 将以解释器文件代替原始输入参数中的*arg0*(`arg0`). 这是因为根据shell的惯例，输入参数中的`arg0`一般是设置为进程文件名或者是文件的绝对路径的。而内核认为，既然该进程是由传入`execl`的第一个参数(解释器文件)启动的，则这个参数可能会比`execl`的`arg0`含有更多的信息（尤其像本例中的`arg0`是另一个参数的情况）。由此可见，在调用`execl`的时候，尽量按照管理将`arg0`设为进程的文件名或路径
4. 原始的后续输入参数`arg1`, `arg2`,...依然作为输入参数传入

## 3.2 命令行执行解释器文件

执行：

    $ ./testinterp arg0 arg1
    Arg[0]: ./echoarg
    Arg[1]: foo
    Arg[2]: ./testinterp
    Arg[3]: arg0
    Arg[4]: arg1

根据第2章（命令行构造`exec`的方式）和第3.1章（`exec`作用于解释器文件的行为模式），可以得到如下的流程：

1. shell fork子进程，构造`exec`: execl("./testinterp", "./testinterp", "arg0", "arg1")  (注意`arg0`为命令行上第一个参数)
2. `execl`试图直接运行`./testinterp`失败，以解释器文件构造输入参数并执行。其输入参数为：
    * ./echoarg: 解释器
    * foo: 解释器可选参数
    * ./testinterp: 解释器文件（正好和shell构造`execl`时的`arg0`一致）
    * arg0
    * arg1

