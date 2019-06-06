---
layout: "post"
title: "docker-compose指定容器在主机上监听的ip"
categories: "blog"
tags: ['docker']
published: True
comments: true
script: [post.js]
excerpted: |
    A short description...
---

* TOC
{:toc}

使用bridge网络模式的时候，启动的容器默认会监听`0.0.0.0`（ipv4）。如果host同时绑定多个ip，那么监听所有ip会有安全隐患。dockerd在启动的时候有一个选项提供用户指定bridge网络中需要监听的ip地址：

>--ip=""
>
>   Default IP address to use when binding container ports. Default is 0.0.0.0.

所以，在启动`dockerd`的时候加上这个选项`--ip=1.2.3.4`，或者在*/etc/docker/daemon.json*中指定`"ip": "1.2.3.4"`。那么，所有通过`docker run`启动的容器在host上监听的ip都是`1.2.3.4`.

但是，`docker-compose`对这个选项的处理有点不同。。。

如果用`docker-compose`启动容器，并且网络模式为bridge，那么默认情况下`docker-compose`会启动一个新的network。

例如：

	# docker network inspect consul_default
	[                                                                                                
		{                                                                                            
			"Name": "consul_default",                                                                
			"Id": "eee62b4846db1191c75e2dca22f0764c39de0ebd8748cc47dc68384a4e472389",                
			"Created": "2019-06-05T17:13:00.164176347+08:00",                                        
			"Scope": "local",                                                                        
			"Driver": "bridge",                                                                      
			"EnableIPv6": false,                                                                     
			"IPAM": {                                                                                
				"Driver": "default",                                                                 
				"Options": null,                                                                     
				"Config": [                                                                          
					{                                                                                
						"Subnet": "192.168.16.0/20",                                                 
						"Gateway": "192.168.16.1"                                                    
					}                                                                                
				]                                                                                    
			},                                                                                       
			"Internal": false,                                                                       
			"Attachable": true,                                                                      
			"Ingress": false,                                                                        
			"ConfigFrom": {                                                                          
				"Network": ""                                                                        
			},                                                                                       
			"ConfigOnly": false,                                                                     
			"Containers": {                                                                          
				"6ee33e7bf262b003bebac07cb715a82f6e08553e703ed6443e2bd36fd0784f6a": {                
					"Name": "dev-consul",                                                            
					"EndpointID": "cf800227b25e1231d9c0f4ce08b289755eb1e7dbf901f53823f10e48de6d0b11",
					"MacAddress": "02:42:c0:a8:10:02",                                               
					"IPv4Address": "192.168.16.2/20",                                                
					"IPv6Address": ""                                                                
				}                                                                                    
			},                                                                                       
			"Options": {},                                                                           
			"Labels": {                                                                              
				"com.docker.compose.network": "default",                                             
				"com.docker.compose.project": "consul"                                               
			}                                                                                        
		}                                                                                            
	]                                                                                                

此时，这个容器绑定的网络并不是`1.2.3.4`。而这时候查看默认的`bridge`网络，有如下输出：

	# docker network inspect bridge         
	[                                                                                
		{                                                                            
			"Name": "bridge",                                                        
			"Id": "59717795394bbcdcd2497d52a1f986d077ba7dff7f9abe47f05c5e1aed3efd66",
			"Created": "2019-06-05T17:09:34.526511325+08:00",                        
			"Scope": "local",                                                        
			"Driver": "bridge",                                                      
			"EnableIPv6": false,                                                     
			"IPAM": {                                                                
				"Driver": "default",                                                 
				"Options": null,                                                     
				"Config": [                                                          
					{                                                                
						"Subnet": "192.168.0.0/20",                                  
						"Gateway": "192.168.0.1"                                     
					}                                                                
				]                                                                    
			},                                                                       
			"Internal": false,                                                       
			"Attachable": false,                                                     
			"Ingress": false,                                                        
			"ConfigFrom": {                                                          
				"Network": ""                                                        
			},                                                                       
			"ConfigOnly": false,                                                     
			"Containers": {},                                                        
			"Options": {                                                             
				"com.docker.network.bridge.default_bridge": "true",                  
				"com.docker.network.bridge.enable_icc": "true",                      
				"com.docker.network.bridge.enable_ip_masquerade": "true",            
				"com.docker.network.bridge.host_binding_ipv4": "1.2.3.4",      
				"com.docker.network.bridge.name": "docker0",                         
				"com.docker.network.driver.mtu": "1500"                              
			},                                                                       
			"Labels": {}                                                             
		}                                                                            
	]

注意，区别是默认的`bridge`网络的`Options`中包含：`"com.docker.network.bridge.host_binding_ipv4": "1.2.3.4"`。

那么，对于`docker-compose`启动的容器怎么能够监听`dockerd`中指定的ip呢？这其实是docker-compose的已知[issue](https://github.com/docker/compose/issues/2999)，按里面的讨论来看，有两个方法：

1. 将service的`network_mode`设置为`bridge`或者`default`（等效）。这种做法相当于让docker-compose在启动新的project的时候不要默认自动创建一个网络，而是沿用系统默认的`bridge`那个网络。

    这种做法的弊端在于:

    1. project之间环境无法隔离
    2. 原本docker-compose创建的网络中所有容器之间可以通过hostname进行访问，但是如果指定用`bridge`网络，那么就没有这个能力了：

        > By default Compose sets up a single network for your app. Each container for a service joins the default network and is both reachable by other containers on that network, and discoverable by them at a hostname identical to the container name.

2. 在`ports`里面的`[host-port]`部分指定ip。例如：

		ports:
		  - "1.2.3.4:8080:8080"

3. 将 `com.docker.network.bridge.host_binding_ipv4` 在docker-compose中的networks的`driver_opts`中指定，例如：

    ```
    networks:
      foo_network:
        driver_opts:
          com.docker.network.bridge.host_binding_ipv4: "1.2.3.4"
    ```
