---
layout: "post"
title: "Dockerfile"
categories: "blog"
tags: ['devop']
published: true
comments: true
script: [post.js]
excerpted: |
    Dockerfile的一些笔记...
---

* TOC
{:toc}

基础
===

Docker从一个 `Dockerfile` 和一个 `context` 开始编译image

`context` 是一组文件，它们位于某个 `PATH` 或者 `URL`：

* `PATH` 是文件系统中的一个目录
* `URL` 是某个Git repo的地址

`context` 是被循环加入的，其中的全部内容都会被发给dockerd

格式
===

`Dockerfile` 必须以 `FROM` 指令开头，它指定了base image，我们会基于这个base image开始build。

`FROM` 行前面也允许一些 `ARG` 指令，用于声明 `FROM` 行中的参数。

注释
---

注释只能单独存在一行。

解析命令
---

解析命令行也以 `#` 开始，用于定义一些影响整体编译的变量。在解析`Dockerfile`的时候一旦遇到任何注释，空行或者其他有效命令，那么Docker会把之后的所有以`#`开头的行认为是注释。因此，解析命令行一定要放在最开始。

环境变量
---

在一行上的环境变量的展开是同时生效的，例如:

    ENV abc=hello
    ENV abc=bye def=$abc
    ENV ghi=$abc

这里的`def`是`hello`而不是`bye`，而`ghi`是`bye`。

.dockerignore 文件
---

这个文件可以指定哪些文件不会作为`context`的一部分传给dockerd（例外是，如果 *.dockerignore*  和 *Dockerfile* 也被屏蔽了的话，它们还是会被传给dockerd）；同时，也作用于 `ADD` 和 `COPY` 指令。

Instruction
===

EXPOSE
---

这往往只是image作者给container执行者的一个信息，告诉她哪（几）个端口是会被使用的。而后续真正被公开到宿主机的方式是在启动container的时候使用 `-p` 或者 `-P` 指令。

ADD
---

拷贝文件到container中，如果是相对路径的话是从 `WORKDIR` 开始的。

* 所有可以拷贝的文件都应该局限在 `context` 中。

* 如果 `<src>` 是一个URL， 注意区分 `<dst>` 表示的路径是否以 `/` 结尾。如果是，则 `<dst>` 代表一个目录，否则代表拷贝后的文件名；

* 否则如果 `<src>` 代表一个本地目录，只有它的子文件和子目录以及它们的元数据被拷贝，`<src>` 目录本身没有被拷贝。

* 否则如果 `<src>` 是个本地文件， 并且它的内容（而非扩展名）表示它是某种压缩文件，那么它会被解压为 `<dst>` 作为一个目录。

* 否则如果 `<src>` 是其他本地文件，也要区分 `<dst>` 表示的路径是否以 `/` 结尾.如果是，则目标文件会是 `<dst>/base(<src>)`；否则，`<dst>` 代表拷贝后的文件。

* 如果 `<src>` 指定了多个文件，那么 `<dst>` 必须要代表一个目录，也即以 `/` 结尾。

COPY
---

拷贝文件到container中，和 `ADD` 的区别在于:

1. 当 `ADD` 的 `<src>` 是某种压缩包，它会被解压缩
2. `ADD` 接受URL作为 `<src>`
3. `COPY` 支持 `--from=<name|index>` 选项，它允许以某个之前的 *build stage* 作为 `<src>` 文件的来源，而不是来源于默认的 `context`.

* 否则如果 `<src>` 代表一个本地目录，只有它的子文件和子目录以及它们的元数据被拷贝，`<src>` 目录本身没有被拷贝。
* 否则如果 `<src>` 是其他本地文件，也要区分 `<dst>` 表示的路径是否以 `/` 结尾.如果是，则目标文件会是 `<dst>/base(<src>)`；否则，`<dst>` 代表拷贝后的文件。

RUN CMD ENTRYPOINT
---

### RUN

* 形式：

  - `RUN <command>` (shell form, run in a shell)
  - `RUN ["executable", "param1", "param2",...]` (exec form)

* 作用：用于在当前image的基础上创建新的一层，用于`Dockerfile`后续的步骤。也就是说，它指定了如何build一个image。
* 数量：允许多个。

### CMD

* 形式：

  - `CMD ["executable", "param1", "param2", ....]` (用于指定该docker的默认执行动作，如果后续有 *ENTRYPOINT*或者命令行启动docker时指定了命令 ，则这部分元素会被作为参数追加到 *ENTRYPOINT*后面（往往会报错）)
  - `CMD ["param1", "param2", ...]` (用于给 *ENTRYPOINT* 设定默认参数)
  - `CMD command param1 param2` (shell form, depracated)

* 作用：定义某个container默认执行的命令。可以以第一种形式的样子指定要运行的程序和它的参数；也可以以第二种方式只指定参数，而将待运行的程序在 `ENTRYPOINT` 中指定。
* 数量：小于等于1。
* 注意：这里的 *shell form*，官网中指出这个可执行文件生成的进程是无法收到信号，因为这个程序不是以PID=1运行在docker中的，而是由shell fork出来的；然而，在我本地测试(v18.01)，却发现该程序是以PID=1运行的。

### ENTRYPOINT

* 形式：

  - `ENTRYPOINT ["executable", "param1", "param2", ...]` (exec form)
  - `ENTRYPOINT command param1 param2` (shell form, depracated)

* 作用：如果在启动docker的命令后面有参数（`docker run <image> <args>`），则这些参数被追加到 *exec form* 的 `ENTRYPOINT` 最后。
* 数量：一般为一个（仅最后一个生效）。
* 注意：这里的 *shell form*，官网中指出这个可执行文件生成的进程是无法收到信号，因为这个程序不是以PID=1运行在docker中的，而是由shell fork出来的；然而，在我本地测试(v18.01)，却发现该程序是以PID=1运行的。

### ENTRYPOINT && CMD

下面列出一个表格表示它们两个之间的关系，可能和关网的不太一样，但是这是我在本地(v18.01)实际试验的结果:

||没有ENTRYPOINT|ENTRYPOINT exec_entry p1_entry|ENTRYPOINT ["exec_entry", "p1_entry"]|
|:-:|:-:|:-:|:-:|
|没有CMD|Error!|/bin/sh -c exec_entry p1_entry|exec_entry p1_entry|
|CMD ["exec_cmd", "p1_cmd"]|exec_cmd p1_cmd|/bin/sh -c exec_entry p1_entry|exec entry p1_entry exec_cmd p1_cmd|
|CMD "exec_cmd" "p1_cmd"|exec_cmd p1_cmd|/bin/sh -c exec_entry p1_entry|exec_entry p1_entry /bin/sh -c exec_cmd p1_cmd|

由上可见，最佳的实践是：使用 `CMD`的 *exec form* 给出默认参数；使用`ENTRYPOINT`的 *exec form* 给出执行指令和默认参数。

VOLUME
---

指定了某个目录是宿主机或者其他容器的目录的挂载点。

当这个命令执行之后，所有对该目录的改动命令都无效。

挂载（包括：volume, bind mount, tmpfs mount）的动作只能发生在运行时的参数，而不是编译时。如果运行时没有相关挂载参数，那么这个目录的内容是默认在编译时生成的内容。

ARG vs ENV
---

`ENV` 定义的变量一般用于被用于 `ADD`/`COPY`/`WORKDIR`/`RUN` 之类的命令，可以用于定制化编译时的行为，同时，它最后也会被export到image的环境变量集中。它的优先级高于`ARG`. 允许在运行docker的时候通过 `docker run --env <key>=<value>` 来修改。`ENV` 在编译全程有效。

`ARG` 一般用于`RUN`命令，也是用于定制化编译时的行为。它更多时候是为了允许用户在编译命令参数中传递不同的值而存在：`docker build --build-arg <key>=<value>`。`ARG` 仅在当前编译stage (`FROM`) 有效。

### SIGNAL

`docker stop`默认向容器的PID 1进程发送`SIGTERM`。如果没有使用exec form启动命令，那么你的进程是sh的子进程，也就是说sh是PID1进程，它负责接收信号；否则，dockerfile里指定的进程是信号的接收者。

这里有一个奇怪的现象是，不论那种情况，如果PID 1进程没有显示地注册信号处理函数，那么并不是使用默认的处理函数（例如`SIGTERM`默认是终止进程），而似乎是忽略该信号。因此，如果你使用exec form执行一个脚本，需要在脚本中是用`trap`明确地注册信号处理函数，否则docker stop的时候会超时触发强制停止，耗时比较长。

另一种做法是，使用[`tini`](https://github.com/krallin/tini)作为PID1进程，这个程序在 Docker 1.13 以后已经内置在docker中了，使用`docker run --init`即可。它会将你指定的entrypoint作为`tini`的子进程，`tini`的生命周期取决于该子进程，并且退出码也是使用该子进程的退出码。同时，`tini`负责将收到的信号forward给这个子进程。这样，entrypoint中指定的程序就不需要显示地指定信号处理函数（除非默认行为不满足需求）。
