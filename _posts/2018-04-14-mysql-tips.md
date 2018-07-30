---
layout: "post"
title: "mysql笔记"
categories: "blog"
tags: ['db']
published: true
comments: true
script: [post.js]
excerpted: |
    记录mysql相关的知识碎片...
---

* TOC
{:toc}

`change master` 与 mysqldump
===

数据迁移可以通过 **mysqldump** 将源库数据dump-then-recover到新的目标库，然后再建立源库和目标库的主从关系保持数据更新。

在执行 **mysqldump** 的时候通常会加上 `--master-data(=1)` ，使其dump出来的数据中包含同步点（这里不讨论gtid的情况）。但是，如果你按照如下的顺序去做迁移：

1. 使用 **mysqldump** `--master-data` 产生dump文件，然后在目标库导入
2. 在目标库执行 `change master` 指令，设置主库的ip, port, user, password
3. 在目标库执行 `start slave`

那么你应该会遇到如下的错误信息：

    error 'Duplicate entry '%-test-' for key 'PRIMARY'' on query.

这是因为，在第二步结束后，同步点已经被复位成('',4)了。这一点在 ["change master to"的页面](https://dev.mysql.com/doc/refman/5.7/en/change-master-to.html) 上有说明:

> If you specify the MASTER_HOST or MASTER_PORT option, the slave assumes that the master server is different from before (even if the option value is the same as its current value.) In this case, the old values for the master binary log file name and position are considered no longer applicable, so if you do not specify MASTER_LOG_FILE and MASTER_LOG_POS in the statement, MASTER_LOG_FILE='' and MASTER_LOG_POS=4 are silently appended to it.

所以，正确的顺序是：

1. 在目标库执行 `change master` 指令，设置主库的ip, port, user, password
2. 使用 **mysqldump** `--master-data` 产生dump文件，然后在目标库导入
3. 在目标库执行 `start slave`

mysql client `--bind-address`
===

`--bind-address` 是 `mysql` 在5.6.1以后才有的选项，用来指定连接mysqld时使用的interface. 
