---
layout: "post"
title: "部署flaskbb(ubuntu)"
categories: blog
tags: ['web']
published: true
comments: true 
script: [post.js]
---

* TOC
{:toc}

# 1 激活 `www-data` 用户

我们希望nginx和uwsgi都以 `www-data` 这个系统默认的给web应用使用的用户来执行。这要求我们将项目文件和python的虚拟环境的owner设置为 `www-data`。所以，在一开始的时候就以这个用户来配置环境会比较方便。

所谓的激活，就是使用 `su` 指令切换当前用户为 `www-data` 而已。先看下这个用户的默认配置（也可以直接查看 `/etc/passwd` ）:

    www-data@hiv8039u:/srv/www/flaskbb$ finger www-data
    Login: www-data                         Name: www-data
    Directory: /var/www                     Shell: /bin/sh
    Never logged in.
    No mail.
    No Plan.

可见这个用户并不是 *no login* 用户，也就是说它是有shell的。

似乎这些系统用户（例如: root也是）是没有默认密码的，所以需要首先给它们设个密码才可以使用 `su` 切换：

    $ sudo passwd www-data
    # 输入密码
    # 验证密码

现在，可以正常切换到 `www-data` 了：

    $ su -s /bin/bash www-data

下面的操作如果没有特殊说明，我们都以 `www-data` 的用户来执行（以 `>` 来标识）。

# 2 下载安装flaskbb

这里按照[官网](http://flaskbb.readthedocs.io/en/latest/installation.html#installation)上面的 *简陋* 的指令去安装flaskbb.

同时，根据[FHS](http://www.pathname.com/fhs/) 的建议，我们将工程目录放置在 `/srv/www` 下面。

## 2.1 virtualenvwrapper

首先，安装这个virtualenv的封装工具：

    > sudo pip install virtualenvwrapper

从github上面下载flaskbb:

    > git clone https://github.com/sh4nks/flaskbb.git

为了便于管理，我选择将 `virtualenv` 放置在flaskbb的项目工程下：
    
    > # 设置环境变量，source脚本
    > export WORKON_HOME=/srv/www/flaskbb/.virtualenvs
    > export VIRTUALENVWRAPPER_SCRIPT=/usr/local/bin/virtualenvwrapper.sh
    > source /usr/local/bin/virtualenvwrapper_lazy.sh
    > # 创建virtualenv
    > cd /srv/www/flaskbb
    > mkvirtualenv -a . -p $(which python2) flaskbb
    > # 自动进入virtualenv flaskbb
    > # 安装dependency
    > pip install -r requirements.txt
    > # 接下来也可以安装一些可选的依赖包（例如redis），这里就不介绍了
    > # 退出 virtualenv
    > deactivate

# 2.2 flaskbb的配置文件

flaskbb提供了2个配置文件可供选择，它们都在 `flaskbb/flaskbb/configs/` 目录下面。不过，我发现现有的配置文件貌似忘记了在文件的第一行指定encoding，如果不加的话在后面用uWSGI启动貌似会出错。所以，在这里给 `development.py.example` 和 `production.py.example` 都加上这行：

    # -*- coding: utf-8 -*-

然后，按照官网上说的，把两个 `.example` 文件都copy成不带这个后缀的文件：

    > cp flaskbb/configs/development.py.example flaskbb/configs/development.py
    > cp flaskbb/configs/production.py.example flaskbb/configs/production.py

其中，`production.py` 中我们有一些配置一定要修改才能用，否则运行时会出错。包括：

* SERVER_NAME

这个选项我把它注释掉了，因为这个选项只在sub domain name需要用到的时候才需要设置，我这台机器当前应该只跑一个site，所以先不设了。

另外，如果有SMTP邮件服务器的，也可以配置E-mail部分。不过，我现在没有配置。一来，我不知道怎么配置；二来flaskbb允许不需要邮件验证的注册，后面会提到。

# 2.3 安装flaskbb

先进入virtualenv:

    > workon flaskbb

然后，按照官网上说的，执行：

    > make install

在安装过程中，会需要你输入一些账户信息，这个账户是flaskbb会为你创建的一个账户，一般把group设置为管理员。

# 3 uWSGI

我们使用uWSGI来作为web服务器，它会被配置为通过某个UDS与Nginx通讯，将用户的请求从Nginx读取，使用uwsgi接口传给flaskbb。再将flaskbb的响应利用uwsgi接口传给Nginx。

# 3.1 安装

根据(uWSGI官网)[https://uwsgi-docs.readthedocs.io/en/latest/WSGIquickstart.html]的指令：

    # 注意，这里我将uwsgi安装到了系统当中，我没有安装到virtualenv中，因为我觉得没什么必要
    $ sudo pip install uwsgi

# 3.2 配置文件

建议的做法是，在 `/etc/uwsgi/` 下创建如下目录结构：

    ├── apps-available
    │   └── flaskbb
    │       └── flaskbb.ini
    └── apps-enable -> apps-available/flaskbb/

然后，通过 `sudo uwsgi --emperor /etc/uwsgi/apps-enabled` 来执行。

`flaskbb.ini` 的内容如下：

    ##
    # This file is the configuration for uWSGI
    # 
    # Start uWSGI server with command:
    #
    # $ uwsgi --ini flaskbb_wsgi.ini
    #
    # alternatively, you can start it in root priviledge,
    # in which case, the 'uid'/'gid' keys are considered
    #
    # For good performance, it should be used together with 
    # Nginx, and the "socket" key should be set. Otherwise,
    # comment "socket" and uncomment "http".

    [uwsgi]

    # set virtualenv and project base directory
    base = /srv/www/flaskbb
    virtualenv = /srv/www/flaskbb/.virtualenvs/flaskbb
    pythonpath = %(base)

    # module:callable
    module = wsgi:flaskbb

    # set uid and gid to "www-data", which
    # is the same u/gid nginx uses.
    # This take effects only when "uwsgi" is launched
    # by root.
    uid = www-data
    gid = www-data

    # Tell uWSGI to start up in master mode
    # and spawn five worker processes to serve
    # actual requests
    master = true
    processes = 5

    # Socket used to communicate with Nginx
    # Nginx handles actual client connections,
    # which will then pass requests to uWSGI.

    socket = /tmp/flaskbb.sock
    chmod-socket = 600
    # clean up the socket when the process stops
    vacuum = true

    # This can help ensure that the init system and
    # uWSGI have the same assumptions about what each
    # process signal means. Setting this aligns the
    # two system components, implementing the expected behavior
    die-on-term = true

    # By default, uWSGI speaks using the uwsgi protocol,
    # a fast binary protocol designed to communicate with other servers.
    # Nginx can speak this protocol natively, so it's better to use this
    # than to force communication by HTTP.


值得注意的是，由于配置文件中指定了`uid` 和 `gid` ，所以 `uwsgi` 需要以root权限执行。

在正式部署到服务器的时候，我们不应该每次都手动执行 `uwsgi` 来启动，而是应该以某种进程管理系统来控制它的自动启动，错误恢复，关机时关闭等操作。下一节介绍的 `supervisor` 就是用于此目的一种跨平台进程管理系统。

# 4 Supervisor

Supervisor是一个用于在类Unix系统中控制和监视进程的一个服务器/客户端形式的系统，类似于 `systemd`, `launchd` 之类的 `init system` 但并不是它们的替代品。

Supervisor和其他进程一样，在系统启动时被运行。同时，Supervisor会根据用户的配置，将指定的程序相应地运行起来。

## 4.1 组件

### supervisord

Supervisor的服务端进程。它负责启动，响应客户端程序，在出错的时候重启客户端程序，以及将客户端程序的标准输出和错误记录到日志中。

### supervisorctl

命令行工具，用于控制客户端程序。它是通过UDS(Unix domain socket)或者TCP socket的方式supervisord进行通信。

### Web Server

Supervisor提供了一个功能类似 `supervisorctl` 的网页界面。通过访问服务端的URL(e.g. `http://localhost:9001/` )，用户可以查看和控制客户端程序的状态。

这个组件工作的前提条件是配置文件中定义了 `[inet_http_server]` 段。

### XML-RPC Interface

启动上面那个Web UI的HTTP服务器同时启动了一个 XML-RPC 接口服务，它可以用于查询和控制supervisor和它启动的程序。

## 4.2 安装Supervisor

安装用的是发行版提供的包：

    $ sudo apt-get install supervisor

安装完毕后，可以查看下 `supervisor` 这个service是否被enable了：

    $ find /etc/rc* -name "*supervisor*"
    /etc/rc0.d/K20supervisor
    /etc/rc1.d/K20supervisor
    /etc/rc2.d/S20supervisor
    /etc/rc3.d/S20supervisor
    /etc/rc4.d/S20supervisor
    /etc/rc5.d/S20supervisor
    /etc/rc6.d/K20supervisor


## 4.3 配置文件

安装完毕后，可以在命令行中执行 `echo_supervisord_conf`. 它会打印出一个配置文件的样本。我们可以利用这个样本来创建。不过arch版本的包中已经自带了配置文件，因此使用默认的配置文件。

具体的配置项还请参考 (官方文档)[http://supervisord.org/configuration.html].

## 4.4 运行程序

supervisord会在启动的时候解析配置文件，其中关于要启动哪些客户端程序的信息记录在配置文件中的 `[program:xxx]` 段（其中的 `xxx` 准确地应该称为 *同类别程序组*  (详见)[http://supervisord.org/configuration.html#program-x-section-settings]

在添加某个程序的时候，用户可以直接修改配置文件，不过这是不推荐的。推荐的做法是在配置文件中加入一个 `[include]` 段，在其中指定需要append的文件的路径（模式）。例如，arch的supervisor包中的配置文件有如下配置：

    [include]
    files = /etc/supervisor.d/*.conf

假设我需要运行 `uwsgi`，那么我可以在 `/etc/supervisor.d/` 中创建文件 `uwsgi.conf` ：

    [program:uwsgi]
    command=/usr/local/bin/uwsgi --emperor /etc/uwsgi/apps-enabled
    stopsignal=QUIT
    autostart=true
    autorestart=true
    redirect_stderr=true

这里如果不指定 `user` ，uwsgi进程将会以supervisord的uid 启动，在这里是以root的权限启动的。因此，在之后uwsgi的配置文件中可以调用 `setuid()` / `setgid()` 将uid/gid设置为配置的用户（www-data）.

配置完毕后，执行:

    $ sudo service supervisor restart

来重启 `supervisor`。不过此时由于Nginx还没有配置完成，所以即使启动了uwsgi，用户还无法通过浏览器访问它。

# 5 Nginx

## 5.1 安装

使用发行版的包安装：

    $ sudo apt-get install nginx

## 5.2 启动

因为现在公司里的运行环境是Ubuntu12.04LTS, 使用的是sysv init初始化系统，所以需要执行：

    # start
    $ sudo service nginx start
    # enable
    $ sudo update-rc.d nginx defaults

验证Nginx是否正确启动，打开浏览器，输入你的eth0的ip地址，页面上会出现类似 **welcome nginx** 之类的字眼就代表安装成功。

## 5.3 配置flaskbb

根据flaskbb官网的建议配置，做一点微小的改动即可，大概包括：

* server_name 使用的localhost
* 执行工程的路径使用的绝对路径
* location @flaskbb中使用UDS，而不是TCP socket

具体配置文件如下：

    server {
        listen 80;
        server_name localhost;

        access_log /var/log/nginx/access.forums.flaskbb.log;
        error_log /var/log/nginx/error.forums.flaskbb.log;

        location / {
            try_files $uri @flaskbb;
        }

        # Static files
        location /static {
           alias /srv/www/flaskbb/flaskbb/static/;
        }

        location ~ ^/_themes/([^/]+)/(.*)$ {
            alias /srv/www/flaskbb/flaskbb/themes/$1/static/$2;
        }

        # robots.txt
        location /robots.txt {
            alias /srv/www/flaskbb/flaskbb/static/robots.txt;
        }

        location @flaskbb {
            include uwsgi_params;
            uwsgi_pass unix:/tmp/flaskbb.sock;
        }
    }

文件路径为 `/etc/nginx/sites-available/flaskbb`。同样，需要创建一个link指向该文件，link为 `/etc/nginx/sites-enabled/flaskbb`

配置完毕后，别忘了重启nginx service:

    $ sudo service nginx restart 

然后，你应该可以通过访问你在配置文件中配置的URL(localhost)来访问flaskbb了！

# 6 flaskbb bug fix

flaskbb本身好像没人维护了的样子，而它官网又说是under development，不建议使用production的配置。所以可以预料的到，有很多坑在里面。这里我列举了一些 *可能* 的问题。

# 7 TODO

这里列了一些按如上配置之后运行中的错误/警告，暂时还不知道原因，需要进一步调查。

* 执行uWSGI之后出现如下错误讯息
    
    ... 
    added /srv/www/flaskbb/ to pythonpath.
    Error opening file for reading: Permission denied
    ...
