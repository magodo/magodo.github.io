---
layout: "post"
title: "Archlinux: Make laptop as AP"
categories: blog
tags: ['tool','network']
published: true
comments: true 
script: [post.js]
excerpted: |
    本文尝试在archlinux系统中，将当前笔记本作为AP，分享网络连接...
---

* Do not remove this line (it will not be displayed)
{: toc}

# 1. 前提条件

## 1.1 Wifi 设备支持AP模式

首先，需要一个兼容nl80211的无线设备，并且该设备支持AP模式。通过`iw list`指令，查询`Supported interface modes`字段下是否列有`AP`:

	[magodo@t460p hostapd]$ iw list | grep "Supported interface" -A 10
			Supported interface modes:
					 * IBSS
					 * managed
					 * AP
					 * AP/VLAN
					 * monitor
					 * P2P-client
					 * P2P-GO
					 * P2P-device
			Band 1:
					Capabilities: 0x11ef

## 1.2 使用一个无线网卡联网并作为AP

通常，创建一个AP时无所谓当前设备是通过什么方式联网的(以太网，无线,...)。有些无线网卡支持`simultaneous`操作，可同时作为AP和无线客户端（联网）。 由于我没有使用这种方式，因此不展开。具体配置步骤请看引用[1]。

# 2. 配置

配置步骤分为两步：

1. 配置 **Wi-Fi link layer**，这样其他无线设备可以连接到你的AP，并且向/从 你的AP 发送/接收IP包，`hostapd`包会完成这些工作。
2. 在AP主机上配置网络，从而该主机可以从它的联网接口中将IP包转发给其他无线设备。

## 2.1 Wi-Fi link layer

首先，安装`hostapd`. 安装完毕后，将 *hostapd.conf* 文件拷贝至 */etc/hostapd* 目录下，根据主机信息修改其中几个默认配置：

1. 在我的机器上，无线网卡的interface名为`wlp3s0`(`iw dev`)，而默认配置文件中的`interface=wlan0`,修改之。
2. 按需修改配置文件中的`ssid`和`wpa_passphrase`。
3. 可能需要设置`channel=n`(例如n可以取`iw dev`输出的channel)，具体见引用[4].

确保无线网卡的interface的状态是 **UP** 的 (`ip link`)，如果不是，执行：`ip link set dev wlp3s0 up`。

启动`hostapd.service`。如果失败原因是"nl80211: Could not configure driver mode", 可以参考引用[3]。

## 2.2 Network configuration

有两种方法配置网络：

1. **bridge**: 在AP主机上创建一个 *bridge*，使连接的无线设备与主机使用同一个interface和子网
2. **NAT**: 通过IP转发/伪装(masquerading)与DHCP服务，使连接的无线设备可以使用一个单独的子网，子网通过NAT进行数据的接收/发送（就像连接DSL/有线调制解调器的路由器一样）

第一种方式比较简单，但是无线设备需要的那些外部服务（例如DHCP）需要AP主机的外部接口提供。这就意味着对于拨号上网或者运营商仅通过DHCP提供给你一个IP地址的调制解调器上网方式来说，是不适用的（试验下来一旦无线设备连接主机后，主机就断开网络了）。

第二种方式适用于所有连网方式，对外界而言是完全透明的。并且还可以通过iptables引入更多功能（例如：流量控制）。

### 2.2.1 Bridge 

首先，创建一个bridge设备，并且设置为UP：

    # ip link add name [bridge_name] type bridge
    # ip link set [bridge_name] up

通过设置interface的master为`bridge_name`，将与互联网连接的interface加入到bridge：

    # ip link set [internet0] master [bridge_name]

另外，如果要去掉被bridge的interface或者删除`bridge_name`，可以使用如下指令：

    # ip link set [internet0] nomaster
    # ip link delete [bridge_name] type bridge

设置完毕后，需要将`hostapd.conf`中的`bridge=`中设置为`bridge_name`。重启`hostapd.service`即可。

但是正如上文所说，我在笔记本上试验发现，如果一旦无线设备连接了，主机就断开了ethernet连接（但是无线设备可以连接internet）。

### 2.2.2 NAT setup

*待研究...*

# 引用

[1] [ArchWiki Software access point](https://wiki.archlinux.org/index.php/software_access_point)

[2] [ArchWiki Internet sharing](https://wiki.archlinux.org/index.php/Internet_sharing)

[3] [Hostapd error "nl80211: Could not configure driver mode"](https://askubuntu.com/questions/472794/hostapd-error-nl80211-could-not-configure-driver-mode)

[4] [Regdomain](https://wiki.archlinux.org/index.php/Wireless_network_configuration#Respecting_the_regulatory_domain)

[5] [ArchWiki Network bridge](https://wiki.archlinux.org/index.php/Network_bridge)


