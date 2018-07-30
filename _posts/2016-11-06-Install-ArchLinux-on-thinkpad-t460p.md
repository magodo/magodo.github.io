---
layout: "post"
title: "Install ArchLinux on thinkpad t460p"
categories: blog
tags: ['linux']
published: true
comments: true 
script: [post.js]
---
* TOC
{:toc}

1 BIOS设置
-------

1. Security -> Secure Boot -> Secure Boot: Disabled
2. Startup -> EFI/Legacy Boot: UEFI Only
3. Config -> Display -> Total Graphics Memory: 512MB

2 使用脚本安装基础系统
------------

详情请参见[这里](https://github.com/magodo/t460p-arch)


3 安装完之后的第一次登录
----------------------

第一次登录之后，会发现终端输出类似如下的错误：

    nouveau:02:00.0: DRM: Pointer to TMDS table invalid
    nouveau 0000:02:00.0: DRM: Pointer to flat panel table invalid

这个是因为`nouveau`显卡驱动的问题吧，后面装了N卡驱动后`nouveau`这个kernel驱动会被blacklist，也就看不到这个问题了。

以及:

    mei_wdt mei::xxxxxx: Could not register event ret=-22

这个貌似网上也找不到什么线索，不过貌似没有什么影响的样子


4 显卡驱动
--------

要想安装桌面环境，我们先把显卡配置好。thinkpad t460p有两个显卡设备：

    # lspci -k | grep -A 3 -E "(VGA|3D)" 

    00:02.0 VGA compatible controller: Intel Corporation HD Grahics 530 (rev 06)
        Subsystem: Lenovo Device 5050
        Kernel driver in use: i915
        Kernel modules: i915

    02:00.0 3D controller: NVIDIA Corporation GM108M [GeForce 940MX] (rev a2)
        Subsystem: Lenovo Device 5050
        Kernel driver in use: nouveau
        Kernel modules: nouveau

NVIDIA的Optimus技术允许集成GPU与独立的NVIDIA GPU被笔记本电脑一起使用。在t460p上，想要让两张显卡正常工作（不一定同时工作）的话，有以下几种配置方式：

1. 在BIOS里将其中一张显卡关闭。不过并不是所有的BIOS都支持这种设置，例如t460p的貌似就不支持 (请参考[这里](https://www.reddit.com/r/thinkpad/comments/4q72qt/t460p_fedora_24_nvidia_940mx_driver_cant_disable/))

2. 使用NVIDIA官方驱动所提供的Optimus支持，这个能提供最好的NVIDIA性能，但是不允许GPU切换。并且这个驱动也可能比开源的驱动有更多bug

3. 使用第三方工具`bumblebee`来实现类似Optimus的功能，它提供了GPU切换(参见[这里](https://wiki.archlinux.org/index.php/Bumblebee))

### 4.1 bumblebee 方案 ###

我选择了使用`bumblebee`。

安装前，先保证删除`/etc/X11/xorg.conf`和`/etc/X11/xorg.conf.d`

安装： `# pacman -S bumblebee`

安装时，会让你选择一个提供libgl的包，根据[arch wiki](https://wiki.archlinux.org/index.php/Bumblebee):

>>> Note: bumblebee depends on mesa-libgl and provides all nvidia-libgl, nvidia-340xx-libgl and nvidia-304xx-libgl to avoid dependency conflict between the respective libgl versions.

我选择了`mesa-libgl`. 

接着安装:

* `mesa`
* `xf86-video-intel`
* nvidia的驱动：`nvidia` or `nvidia-340xx` or `nvidia-304xx`：根据这个[wiki](https://wiki.archlinux.org/index.php/NVIDIA)知道，t460p的GeForce 940MX是比400系列更新的版本，应该安装`nvidia`或者`nvidia-lts`. 我选择的是`nvidia-lts`

安装`nvidia`过程中，会让你选择安装`xf86-input-evdev` 或者 `xf86-input-libinput`。根据[这里](https://www.reddit.com/r/archlinux/comments/48tqj9/difference_between_libinput_and_evdev/)，我选择安装后者。

安装完毕后，需要将系统中的用户（例如*magodo*）加到`bumblebee`的group里：

    # gpasswd -a magodo bumblebee

然后，enable `bumblebeed.service`并且重启即可：

    # systemctl enable bumblebeed.service

重启之后，发现之前第一次重启的两个错误log(如下)没了。那是因为N卡现在用的驱动不是`nouveau`了：

    nouveau:02:00.0: DRM: Pointer to TMDS table invalid
    nouveau 0000:02:00.0: DRM: Pointer to flat panel table invalid

接下来，我们要检查一下bumblebee是否工作。安装`mesa-demos`，并且执行`glxgears`:

    # optirun glxspheres64

然后，我就遇到了如下的问题：

[[ERROR]Cannot access secondary GPU: No devices detected](https://wiki.archlinux.org/index.php/bumblebee#.5BERROR.5DCannot_access_secondary_GPU:_No_devices_detected)

这里的话，按照wiki上面是说的，首先要确保你的显卡用的是`nvidia`的驱动而不是`nouveau`:

    # lspci -k | grep -A 3 3D   # check "Kernel driver in use: ..."

然后，在Xorg配置文件搜索路径的任何一处定义NVIDIA显卡设备（例如：`/etc/bumblebee/xorg.conf.nvidia`）中，把`BusID`那行去注释，然后设上N卡的BusID（`# lspci | grep 3D`），在我这里得到的是`02:00.0`。这里有两点要注意：

1. 写到文件里的时候把最后的`.`改成`:`
2. `lspci`的输出是16进制的，而写到文件里的应该是10进制的

这下，`# optirun glxspheres64` 可以跑了，并且在log里输出当前正在是用N卡！

如果你在这一步遇到：

    [VGL] ERROR: Could not open display

除了wiki上说到的原因外，很可能是因为你在非桌面环境下执行了`optirun`的缘故。所以，先要整个DE.

5 桌面环境
--------

接着，我们可以安装桌面环境(DE)啦～这个的话有很多选择，我先选了比较大众的`gnome3`试试水

### 5.1 安装xorg

1. `# pacman -S xorg-server`
2. `# pacman -S xorg-xinit`

  * 创建`~/.xserverrc`:
  
      #! /bin/sh
      exec /usr/bin/Xorg -nolisten tcp "$@" vt$XDG_VTNR

  * `# cp /etc/X11/xinit/xinitrc ~/.xinitrc`

  测试：

  * `# pacman -S xorg-xclock xterm`
  * `# startx`

### 5.2 安装gnome3

1. `# pacman -S gnome3`
2. 按照[wiki](https://wiki.archlinux.org/index.php/GNOME), 配置成手动启动: 在`~/.xinitrc`中，把下面的部分从`twm &`开始都注释，然后加上`exec gnome-session`

### 5.3 安装GDM

GDM允许自动启动图形服务，也提供图形化登录. 在安装了`gnome3`之后，`gdm`也已经被自动安装了。只需要enable它即可：

    # systemctl enable gdm.service

重启即可。

如果在这之前创建过非root用户，并且在重启后以该用户无法登录，那么可能是因为之前创建用户的时候没有指定*initial group*, 也有可能是因为该用户没有设定密码(`# passwd -Sa`, 第二列显示L的那些)。正确的添加用户方法如下：

    # useradd -m -g initial_group -G additional_groups -s login_shell username    # e.g. useradd -m -g wheel -G bumblebee -s /bin/shell magodo
    # passwd username

如果想让新用户有`sudo`权限，执行`# visudo`进行配置。例如，由于*magodo*在`wheel`这个group里，我就在`visudo`里把`%wheel AA=(ALL) ALL`这行去注释了。表示`wheel`这个group里的用户需要输入密码来执行`sudo`指令。


6 Network Manager
==================

安裝以下包：

* `networkmanager`
* `network-manager-applet`
* `dhclient`
* `rp-pppoe`

接着，enable NetworkManager:

    systemctl enable NetworkManager.servie


7 中文支持
==========

包括中文的顯示和輸入

### 7.1 中文字體的安裝

在[這裏](https://wiki.archlinux.org/index.php/fonts#Chinese.2C_Japanese.2C_Korean.2C_Vietnamese)列出了arch支持的中文字體包,一開始我只安裝了一個字體，貌似由於不同軟件對字體的支持不同（也許是outline和bitmap的區別吧），所以最好多安裝幾個字體。總之，我是把除了AUR之外的官方包都裝了。

### 7.2 中文字體的輸入

安裝`ibus`和`ibus-rime`。然后，在`$HOME/.bashrc`中加入如下內容：

    export GTK_IM_MODULE=ibus
    export XMODIFIERS=@im=ibus
    export QT_IM_MODULE=ibus

執行`ibus-setup`對ibus進行配置。然后，在*setting*->*Region & Language*下的*input sources*中加入Other->Chinese(Rime)。重啓即可。

我不知道爲啥我的輸入法都是繁體字，有待研究下。

8 Trouble Shooting
=======================

1. ctrl-alt-NUM 切换tty显示混乱

  貌似是之前由于tty默认字体太小，我设置成了iso02-12x22（保存在`/etc/vconsole.conf`）导致的。把这个文件删了就好了... 或者，在一个显示错乱的tty里(bindly)输入`# setfont`，于是，你又用回系统的默认字体并且正确显示了。

2. 在gnome3上没法打开terminal，并且terminal图标边上一直有个东东在转个不停

  这个问题是因为之前的装机脚本里面设置完`/etc/locale.gen`之后忘记执行`# locale-gen`了

3. chromium用root执行会crash (` [0420/105918:FATAL:setuid_sandbox_client.cc(126)] Check failed: IsFileSystemAccessDenied().`)

  chromium貌似就是不准root启动，详情请见[这里](https://bbs.archlinux.org/viewtopic.php?id=196353)



