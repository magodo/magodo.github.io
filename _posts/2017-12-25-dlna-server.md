---
layout: "post"
title: "UPnP Server"
categories: "blog"
tags: ['life']
published: true
comments: true
script: [post.js]
---

* TOC
{:toc}

首先，圣诞快乐...

其次，本来上周末一直在犹豫要买个NAS，可是觉得我的需求不是太多，主要是想：

1. 用RAID备份一些照片（移动硬盘指不定哪天挂了）
2. 用Pad看电脑里的电影

而不是（gan）想：

1. 用server跑steam游戏然后stream到笔记本
2. 搭建个git server和小伙伴一起写代码（被霍格无情拒绝）
3. ...

所以，实际上一个UPnP服务就可以搞定我现在的第一个需求了，而第二个需求，我发现我的照片加起来没超过30GB，打个包直接放在好几个地方就完了- -

这里主要讲我在Archlinux上用 [ReadyMedia](https://wiki.archlinux.org/index.php/ReadyMedia) 搭建时的大概过程和碰到的问题。

其实很简单，按照wiki上的步骤安装，enable/start。然后，配置下 **/etc/minidlna.conf** 的 `media_dir` 就行了。

可是，我遇到了几个和权限相关的问题：

1. 挂载在 */run/media/* 下面的目录没法访问

    我的OS(archlinux)会将移动存储设备默认挂载到 */run/media/${USER}/* 下面，这个目录的权限是仅允许当前用户访问(`ls -l`的输出是`drwxrwx---+`)，由ACL控制，具体可以参见[Udisks](https://wiki.archlinux.org/index.php/Udisks)，这会导致`minidlnad`不能访问要export的目录。因为 `minidlna` 默认以 **minidlna** 权限执行。

    解决方法有两种：

    1. 将 `minidlna` 的uid改为 **root** ：这需要修改service文件和配置文件中的 **user** 项。
    2. 按照Udisks wiki里面的描述，加入一个udev rule，将`UDISKS_FILESYSTEM_SHARED`设置为`1`，这会将移动设备挂载到 */media/* 下面，并且其他用户是有可读权限的。（我采用了这种方法）

2. 无法访问HOME目录下的文件

    这是由于在 service 文件中指定了 **ProtectHome=on**，这个配置项的作用如下：

        ProtectHome=
               Takes a boolean argument or "read-only". If true, the directories
               /home and /run/user are made inaccessible and empty for processes
               invoked by this unit. If set to "read-only", the two directories
               are made read-only instead. It is recommended to enable this
               setting for all long-running services (in particular
               network-facing ones), to ensure they cannot get access to private
               user data, unless the services actually require access to the
               user's private data. Note however that processes retaining the
               CAP_SYS_ADMIN capability can undo the effect of this setting.
               This setting is hence particularly useful for daemons which have
               this capability removed, for example with CapabilityBoundingSet=.
               Defaults to off.Home=
               Takes a boolean argument or "read-only". If true, the directories
               /home and /run/user are made inaccessible and empty for processes
               invoked by this unit. If set to "read-only", the two directories
               are made read-only instead. It is recommended to enable this
               setting for all long-running services (in particular
               network-facing ones), to ensure they cannot get access to private
               user data, unless the services actually require access to the
               user's private data. Note however that processes retaining the
               CAP_SYS_ADMIN capability can undo the effect of this setting.
               This setting is hence particularly useful for daemons which have
               this capability removed, for example with CapabilityBoundingSet=.
               Defaults to off.

  因此，创建一个 snippet 配置文件，将其改为 read-only，`minidlna`就可以访问了。


另外，在Pad上面感觉VLC比较好用，免费没有广告。
