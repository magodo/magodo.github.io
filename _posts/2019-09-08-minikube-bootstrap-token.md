---
layout: "post"
title: "minikube 设置 bootstrap token"
categories: "blog"
tags: ['minikube-bootstrap-token']
published: True
comments: true
script: [post.js]
excerpted: |
    A short description...
---

* TOC
{:toc}

通过minikube启动一个集群，默认使用的authentication机制为`client certificates`. 该certificates对应的账户拥有设置`RoleBinding`/`ClusterRoleBinding`.

# 启动minikube

首先，如果要使用bootstrap token做验证，同时需要使用rbac做访问管理，那么在启动api server的时候传入参数: `--enable-bootstrap-token-auth` 和 `--authorization-mode=rbac`. 在minikube里，等价与：

```shell
# minikube --extra-config="apiserver.authorization-mode=rbac" --extra-config="apiserver.enable-bootstrap-token-auth" start
```

# 创建bootstrap token

```shell
# kubeadm token create --ttl 0
x972sl.gk76qr3zjk7u2wrm
```

注意：返回的这串用`.`分隔的字符串就是一个bootstrap token. 它的前6位是一个`token id`，后16位是`token secret`. 通过这个token登录的用户名为：`bootstrap:<token-id>`，属于`system:bootstrapper`这个group。

# 设置rbac

刚才创建的这个token对应的user只有在`kube-system`这个namespace中拥有有限的权限，例如它无法访问`default` namespace中的资源。可以通过rbac为它bind更高的权限。

以下示例给bootstrap token对应的user绑定最高权限:

```shell
# #使用minikube默认context执行
# kubectl config use-context minikube

# #给该用户：system:bootstrap:x972sl 设置权限
# kubectl create clusterrolebinding cluster-admin-bootstrap --clusterrole=cluster-admin --user=system:bootstrap:x972sl
```

# 访问api server

接下来，我们就可以用刚才创建的这个帐号与api server进行交互.

NOTE：如果没有设置rbac，访问这个集群会得到如下错误信息：*pods is forbidden: User "system:bootstrap:x972sl" cannot list resource "pods" in API group "" in the namespace "default"*.

- 通过`kubectl`的直接访问方式：

    ```shell
    # kubectl --token=x972sl.gk76qr3zjk7u2wrm -s=https://192.168.99.107:8443 --certificate-authority=/home/magodo/.minikube/ca.crt get pods
    # #或者：kubectl --token=x972sl.gk76qr3zjk7u2wrm -s=https://192.168.99.107:8443 --insecure-skip-tls-verify=true get pods
    ```

- 通过设置config+`kubectl`的访问方式：

    在*~/.kube/config*中加入新的`user`和`context`：

    其中`user`段配置如下：

    ```shell
    - name: foo
      user:
        token: x972sl.gk76qr3zjk7u2wrm
    ```

    其中`context`段配置如下：

    ```shell
    - context:
        cluster: minikube
        user: foo
      name: foo
    ```

    访问：

    ```shell
    # kubectl config use-context foo
    # kubectl get pods
    ```

- client-go:

    构造`rest.Config`的代码大概长这样：

    ```go
    // refered to: tools/clientcmd/client_go_config_test.go

	const token = "x972sl.gk76qr3zjk7u2wrm"
	const url = "https://192.168.99.107:8443"

	config := &rest.Config{
		Host:    url,
		APIPath: "v1",
		ContentConfig: rest.ContentConfig{
			AcceptContentTypes: "application/json",
			ContentType:        "application/json",
		},
		BearerToken: token,
		TLSClientConfig: rest.TLSClientConfig{
			Insecure: true,
		},
	}
    ```
