---
layout: "post"
title: "Ansible Playbook"
categories: "blog"
tags: ['devop']
published: true
comments: true
script: [post.js]
excerpted: |
    浅析Ansible Playbook...
---

* TOC
{:toc}

# playbook 的组成

一个`playbook`由多个`play`组成，每一个`play`作用于一个或多个host，对它们作一些配置，如下图所示：

![play](/assets/img/ansible/ansible_play_book.png)

# 变量

## 变量优先级和作用域

优先级由高到低：

**extra vars**

- 描述：从命令行传入的参数。

**set_fact / register vars**

- 描述：`set_fact` 一般是作为一个task，由用户主动设置`fact`；`regiseter`则是执行了某个指令后返回的值存于某个变量中。
- 作用域：从设置/寄存直至playbook结束

**role include_vars**

**include params**

**role (and include_role) params**

- 描述：在指定role的时候指定的变量，例如：
  
      roles:
         - {role: magodo, var: foo}
         - {role: magodo, var: bar}

**task vars (only for the task)**

**block vars (only for task in block)**

**role vars**

- 描述：定义在 *role/vars/main.yml* 中的变量，用于对不同的role定义用到的变量。

**play vars_files**

- 描述：可以在 playbook 同目录下创建一个 *vars* 目录，然后在里面创建 *foobar.yml* 文件，每个文件中都是各种key-value对。这样写可以做到将逻辑与变量分离，允许用户将敏感的变量单独保存。

**play vars_prompt**

**play vars**

**host facts**

- 作用域：整个playbook周期

**playbook host_vars/***

- 描述：定义在 *<playbook_dir>/host_vars/<host_name>/xxx.<yml|yaml|json>* 中的变量。

**inventory host_vars/***

- 描述：定义在 *<inventory_dir>/host_vars/<host_name>/xxx.<yml|yaml|json>* 中的变量。

**inventory file or script host vars**

- 描述：定义在 inventory(e.g. hosts) 文件中的变量或者由所谓的 `dynamic inventory`（即针对各种服务商的inventory脚本） 产生的变量。

**playbook group_vars/***

- 描述：定义在 *<playbook_dir>/group_vars/<group_name>/xxx.<yml|yaml|json>* 中的变量。

**inventory group_vars/***

- 描述：定义在 *<inventory_dir>/group_vars/<group_name>/xxx.<yml|yaml|json>* 中的变量。

**playbook group_vars/all**

- 描述：定义在 *<playbook_dir>/group_vars/all.yml* 中的变量。

**inventory group_vars/all**

- 描述：定义在 *<inventory_dir>/group_vars/all.yml* 中的变量。

**inventory file or script group vars**

**role defaults**

- 描述：定义在 *role/defaults/main.yml* 中的变量，用于对不同的role定义用到的变量的默认值。

一般的用法为：role中定义了对于每个role的默认参数 ( **role defualts** )和变量 ( **role vars** ). inventory中定义了host和group相关的变量 ( **group/host_vars** ). 它们的优先级情况为： `role defaults < group_vars < host_vars < role vars`. 如果，对于一个已经写好的role，用户想要改变其中的 **role vars**, 可以在定义 `play` 的 `roles` 的地方传入 **role params** , 其优先级大于 **role vars** ，且是用户可控的。

# 99 Tips

## 99.1 Jinja2 批量修改list中的每一个元素

使用map+regex_replace，来对每一个元素做修改. 例如，我们要将`[a.sh, b.sh]`这个list中的每个元素都加入前缀路径，可以写成下面这样：

{% raw %}
    - name: prepend dir for each file
      debug:
        msg: "{{ [a.sh, b.sh] | map('regex_replace', '^(.*)$', '/path/\\1') | list }}"
{% endraw %}

那么，如果这个前缀路径是保存在变量中的，那么应该写成下面这样:

{% raw %}
    # path is a variable
    - name: prepend dir for each file
      debug:
        msg: "{{ [a.sh, b.sh] | map('regex_replace', '^(.*)$', path + '/\\1') | list }}"
{% endraw %}

这是因为，在jinja的template里面`{{...}}`，可以看成是一段python的代码，支持大部分python的基本语法。


