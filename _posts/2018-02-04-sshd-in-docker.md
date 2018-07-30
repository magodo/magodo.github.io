---
layout: "post"
title: "Dockerfile：sshd service"
categories: "blog"
tags: ['devop']
published: true
comments: true
script: [post.js]
excerpted: |
    A short description...
---

* TOC
{:toc}

# 介绍

为了学习ansible，我需要一个“远程”服务器，并且保证可以通过ssh通信。此时，一个运行了sshd的Docker应该可以很好的充当这个角色。

网上搜了下发现一些教程：

* [官方](https://docs.docker.com/engine/examples/running_ssh_service/) 的教程是通过密码的方式登录，不够方便
* 其他一些教程似乎是通过手动的方式拷贝公钥，这个应该由`Dockerfile`完成

然后，就打算自己试写一个 `Dockerfile` 。

# Dockerfile 

内容如下，比较简单：

**Ubuntu 版本**

    FROM ubuntu

    # PUBLIC_KEY holds the content of host ssh public key, you should assign it when invoking
    # the build. E.g. $ docker build --build-arg PUBLIC_KEY="$(cat ~/.ssh/id_rsa.pub)" .
    ARG PUBLIC_KEY

    # python is needed in managed node
    RUN apt-get update && apt-get install -y \
        openssh-server \
        python-simplejson

    # sshd needs this directory
    RUN mkdir /var/run/sshd

    # remember the public key of the host
    RUN mkdir /root/.ssh/
    RUN echo $PUBLIC_KEY >> /root/.ssh/authorized_keys

    EXPOSE 22

    CMD ["/usr/sbin/sshd", "-D"]

**Centos 版本**

    FROM centos:6.8

    # PUBLIC_KEY holds the content of host ssh public key, you should assign it when invoking
    # the build. E.g. $ docker build --build-arg PUBLIC_KEY="$(cat ~/.ssh/id_rsa.pub)" .
    ARG PUBLIC_KEY

    RUN yum update -y && yum install -y \
        openssh-server \
        python-simplejson

    # sshd needs this directory
    RUN mkdir /var/run/sshd

    # remember the public key of the host
    RUN mkdir /root/.ssh/
    RUN echo $PUBLIC_KEY >> /root/.ssh/authorized_keys

    # ensure the host keys are generated
    RUN ["/sbin/service", "sshd", "start"]
    RUN ["/sbin/service", "sshd", "stop"]

    EXPOSE 22

    CMD ["/usr/sbin/sshd", "-D"]

(很奇怪，后面测试下来觉得centos版本的连接速度很慢。)

# Build

执行：

{%highlight shell%}
$ docker build -t ssh --build-arg PUBLIC_KEY="$(cat ~/.ssh/id_rsa.pub)" .
{%endhighlight%}

# Run

启动容器：

{%highlight shell%}
$ docker run --rm  -Pd ssh
{%endhighlight%}

如果需要用主机的 IP 去连，则可以检查下容器22端口映射到主机的端口:

{%highlight shell%}
$ docker port <container>
22/tcp -> 0.0.0.0:32773
{%endhighlight%}

如果直接以容器的IP去连，则可以检查下容器的IP：

{%highlight shell%}
$ docker container inspect <container> | grep IPAddress -C 3
                ]
            },
            "SandboxKey": "/var/run/docker/netns/4e5bfb98fffb",
            "SecondaryIPAddresses": null,
            "SecondaryIPv6Addresses": null,
            "EndpointID": "977e34e90bcb25e29c71c50436e76bad3517498d897fa5d61e4228d403cf93ff",
            "Gateway": "170.17.0.1",
            "GlobalIPv6Address": "",
            "GlobalIPv6PrefixLen": 0,
            "IPAddress": "170.17.0.2",
            "IPPrefixLen": 16,
            "IPv6Gateway": "",
            "MacAddress": "02:42:aa:11:00:02",
--
                    "NetworkID": "6a46a90c45ce75e0773a834cba5dcc4b9952086cce67c82e6686ba6f0e89194f",
                    "EndpointID": "977e34e90bcb25e29c71c50436e76bad3517498d897fa5d61e4228d403cf93ff",
                    "Gateway": "170.17.0.1",
                    "IPAddress": "170.17.0.2",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
{%endhighlight%}

连接容器：

{%highlight shell%}
$ ssh -p 32773 root@127.0.0.1
# OR
$ ssh root@170.17.0.2
{%endhighlight%}

# Clean Up

{%highlight shell%}
$ docker container stop <container>
$ docker image rm ssh
{%endhighlight%}


