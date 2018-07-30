---
layout: "post"
title: "理一理Python包管理"
categories: "blog"
tags: ['python']
published: true
comments: true
script: [post.js]
excerpted: |
    A short description...
---

* TOC
{:toc}

# 1. History

Python包管理软件可以细分为打包软件(`build tool`)和包管理软件(`installer`)。从python3.4开始，`pip`就作为包管理软件被加入标准库中，方便用户管理（安装/更新/删除）不同的包。然而，考虑到前向兼容问题，python并没有将现代化的打包模块加入标准库中，这些模块包括`setuptools`, `wheel`, `twine`等，而只有`distutils`作为标准库的一部分存在。

## 1.1 打包模块

以下是官方对python现有的打包模块的历史简介：

> **distutils** is the original build and distribution system first added to the Python standard library in 1998. While direct use of distutils is being phased out, it still laid the foundation for the current packaging and distribution infrastructure, and it not only remains part of the standard library, but its name lives on in other ways (such as the name of the mailing list used to coordinate Python packaging standards development).

> **setuptools** is a (largely) drop-in replacement for distutils first published in 2004. Its most notable addition over the unmodified distutils tools was the ability to declare dependencies on other packages. It is currently recommended as a more regularly updated alternative to distutils that offers consistent support for more recent packaging standards across a wide range of Python versions.

> **wheel** (in this context) is a project that adds the bdist_wheel command to distutils/setuptools. This produces a cross platform binary packaging format (called “wheels” or “wheel files” and defined in PEP 427) that allows Python libraries, even those including binary extensions, to be installed on a system without needing to be built locally.

因此，它们大致的关系如下：

    distutils (standard)   <-- (override) --       setuptools

            ^                                          ^
            |                                          |
            +-------------------+----------------------+
                                |
                              wheel (cross platform format)         twine (register & upload)

## 1.2 包管理模块

包管理模块一般就用`pip`。如果有虚拟环境需求的还需要使用`virtualenv`。不过，有一个叫做`pipenv`的项目将这两者结合，并且还弥补了许多`pip`存在的不足（使用*Pipfile*和*Pipfile.lock*替换*requirements.txt*）.

# 2. Cookbook

## 从CSV上安装python包

从gitlab上安装的时候，例如项目的url输出如下：

    $ git remote -v
    origin  git@gitlab.xxx.com:foo/bar.git (fetch) 
    origin  git@gitlab.xxx.com:foo/bar.git (push) 

可以用这个url传给pip，让它从gitlab上获取源文件，然后编译安装。但是需要注意的是，似乎除了`git`协议外，其他协议需要将上面输出的url中的"...com:foo/..."部分的":"改为"/"。否则，会导致url无法解析的问题。

例如，这个例子中gitlab使用的是`ssh`协议，那么我们可以这么调用`pip`:

    git intall [-e] "git+ssh://git@gitlab.xxx.com/foo/bar.git#egg=pkg_name"

注意上面最后的参数要用引号引起来，否则里面出现的符号会被bash解析，例如当指定`subdirectory`的时候（通常用于 *setup.py* ）不在repo的根目录的时候，写法如下：

    git intall [-e] "git+ssh://git@gitlab.xxx.com/foo/bar.git#egg=pkg_name&subdirectory=sub_dir"

如果不加括号的话，`subdirectory`部分就不会被传给pip，并且pip会跑在后台。

另外，上面的`-e`是指定pip是否以可编辑模式去安装，可编辑和不可编辑的区别在于：

* 可编辑：会将源文件下载到本地的 *src/SomeProject* 目录下，编译安装
* 不可编辑：会将源文件下载到一个临时目录，然后编译安装
* `pip freeze` 只会将不可编辑的信息

## virtualenv和venv的区别

**venv**是python3标准库的一部分，安装了python3之后就有的一个module。它可以用于创建虚拟环境：`$ python3 -m venv <path>`。

**virtualenv**是对前者的一个封装，它也可以用于创建虚拟环境：`$ virtualenv <options> <path>`。

它们的一个区别是，**venv**创建的虚拟环境使用的`pip`/`setuptools`/`wheels`的版本是与创建该虚拟环境的python所使用的版本；而**virtualenv**则会去pypi源下载最新的`pip`/`setuptools`/`wheels`。因此，如果你的环境访问pypi源有问题的话，建议使用**venv**创建虚拟环境。
