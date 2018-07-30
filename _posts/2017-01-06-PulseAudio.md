---
layout: "post"
title: "PulseAudio"
categories: blog
tags: ['linux']
published: true
comments: true 
script: [post.js]
---

* TOC
{:toc}

# 1. 简介

PulseAudio是一款面向POSIX操作系统的音频系统。它的主要功能是允许用户在不同程序，声卡，甚至不同主机上的声音数据进行传递和混合。同时，它也可以改变音频数据的格式，采样率，通道数。

# 2. 软件架构

PulseAudio会启动一个后台进程，用于接收来自一个或多个Source的音频数据（进程或者声卡），可选择地做一些音频数据的转换（格式，采样率，通道数），接着将数据传到一个或多个Sink（其他进程，声卡，其他网络上的PulseAudio进程）。

PulseAudio的一个目标是使所有的音频流都通过自己的后台进程，包括那些直接操作声卡设备的进程。之所以PulseAudio可以做到这点，是因为它提供给那些使用其他音频系统（例如:ALSA）的应用程序各种适配器（Adapter）。

对于Linux，默认的音频系统用的是ALSA，与声卡设备交互的数据都是通过ALSA driver来实现写入或者接收的。PulseAudio在安装到Linux系统时会通过配置ALSA，将它的默认设备设置为PulseAudio提供的一个虚拟声卡设备。于是，使用ALSA的应用程序会将音频数据输出值PulseAudio(虽然这些应用程序以为是在操作ALSA设备)，然后PulseAudio再使用ALSA接口去操作真正的声卡设备。

同时，PulseAudio也提供native的接口给应用程序，使他们可以直接和PulseAudio交互。

对于不同的应用程序，在使用PulseAudio的操作系统中，音频数据的流通路线大致有以下几种：

1. 使用ALSA接口与pulse设备交互的应用程序

    Sound source -> libALSA -> PulseAudio -> ALSA driver -> hardware

2. 使用PulseAudio接口的本地应用程序

    Sound source -> PulseAudio -> ALSA driver -> hardware

3. 使用PulseAudio接口的网络应用程序

    Sound source -> PulseAudio -> network -> PulseAudio -> ALSA driver -> hardware

4. 使用ALSA接口与其他ALSA设备交互的应用程序

    Sound source -> libALSA -> ALSA driver -> hardware

5. 直接与声卡设备交互

    Sound source -> ALSA driver -> hardware

Manuel Amador Briz 提供了一幅很详细的图示：

![pulseAudio architecture](/assets/img/pulseaudio/Pulseaudio-diagram.png)

对于PulseAudio Server本身，它可以运行于 *system mode* 或者 *per-user mode*。 它的大部分工作(音频流的路由和处理)都是依靠它的modules来完成的，它自己只负责提供相应的API和管理动态加载的module.

# 3. 权限

在Posix操作系统中，非设置UID进程是以当前用户作为实际用户去执行的。与此同时，不同的发行版会对声卡设备的访问权想做限制，具体可以分为以下三种：

1. 只有在 **audio** group中的用户可以访问声卡
2. 使用udev(or HAL)和ConsoleKit来动态分配给当前"active"的用户需要的权限，但是 **audio** group中的用户总是可以访问
3. 使用udev(or HAL)和ConsoleKit来动态分配给当前"active"的用户需要的权限，不考虑用户是否在 **audio** group中

判断当前操作系统属于以上哪一类的方式是：

1. 判断系统是否使用 **audio** group:

    执行： `ls -l /dev/snd`，列出的文件的group是audio

2. 判断系统是否使用udev(or HAL) 和 ConsoleKit：

    执行同样的命令，列出的文件的权限位的最后一位是 **+**号

注意事项：

1. 如果你的发行版属于上面的第一种类型，那么你应该把所有用户都放到 **audio** group
2. 如果你的发行版属于上面的第二或者第三中类型，那么你应该保证没有用户在 **audio** group. 除非你希望pulseaudio运行在 *system-wide* 模式下（一般只有headless的机器上会用到），那么需要将特殊的 *pulse* 用户加到 **audio** group中
3. 如果有用户在 **audio** group，那么 *fast user switch*无法正常工作

# 4. 配置文件

读取配置文件的顺序是：先查看 `~/.config/pulse`；如果没有，在查看 `/etc/pulse`。

## 4.1 daemon.conf

这个文件定义了Server相关的静态配置，包括但不限于：默认采样率，重采样的方法，实时调度策略等。一旦Server开始运行，这些配置就无法改变，除非重启PulseAdudio daemon.

## 4.2 default.pa

这个文件在 `daemon.conf` 之后被解析，用于在启动时对module做配置。这些配置可以在运行时通过 `pactl` 或者 `pacmd` 进行修改( `pactl` 是 `pacmd` 的子集)，也有部分可以被client的配置所覆盖（见下文）。详细的配置项可以通过 `man pulse-cli-syntax` 来查询。建议不要直接修改 `/etc/pulse/default.pa` ，而是在 `~/.config/pulse/default.pa` 中第一行加入 `.include /etc/pulse/default.pa` ，然后修改那些默认设置。 

用户也可以选择不使用 `dafault.pa` 中的配置，而是通过命令行的方式配置：执行 `pulseaudio -nC`。这里的 `-n` 参数告诉 PulseAudio不要加载 `default.pa` 文件；`-C` 参数告诉PulseAudio启动命令行来接受配置并且将信息和错误打印于该命令行（等价于 `--load=module-cli`）。这种方式一般用于调试。

## 4.3 client.conf

这个文件在PulseAudio client library启动的时候被解析，用于控制Client端的一些配置，包括但不限于：default sink, default source等。

注意，Client端的一些配置( `client.conf` )会覆盖Server端的配置( `default.pa` )，例如：default sink/source.

# 5. 进入PulseAudio命令行配置运行时配置

首先，请检查下在当前系统中PulseAudio是否已经启动。如果没有，那么可以执行：

    # pulseaudio -nC

如果 PulseAudio 已经运行，则执行：

    # pacmd

注意：如果PulseAudio在daemonize的时候传入 `--disallow-module-loading` 参数，或者在 `daemon.conf` 中将 `allow-module-loading=` 设置为 `false`. 那么需要将该参数重置为允许加载模块，并重启PulseAudio。原因是，执行上面提到的命令，本质上都等价于加载了 `module-cli`. 单独保证在 `default.pa` 中加入 `load-module module-cli` 貌似是不够的。

# 引用

[1] [PulseAudio官网 - User](https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/)

[2] [PulseAudio ArchWiki](https://wiki.archlinux.org/index.php/PulseAudio)

[3] [PulseAudio WikiPedia](https://en.wikipedia.org/wiki/PulseAudio)
