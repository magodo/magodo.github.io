---
layout: "post"
title: "Wireshark practices"
categories: blog
tags: ['tool']
published: true
comments: true 
script: [post.js]
---

* TOC
{:toc}

最近项目中有用到wireshark来解析并提取音频文件，因此稍微了解了下这个软件。我看的是视频教程 **《Wireshark协议分析从入门到精通》视频课程[陈鑫杰主讲]**, 这个大概地讲解了下wireshark的UI以及一些常用按钮的作用，作为入门还是不错的（不过谈不上精通）。

这篇博客记录一下我用wireshark做的一些实践

1 从网易云音乐下载收费歌曲
==========================

这个有两种方式可以实现：

### 1.1 获取歌曲的URI

在播放歌曲之前，打开wireshark。通过 *Statistics->Conversations* 我们可以确定网易云音乐的服务器ip(以下假设为 *ip163*). 然后，在display filter中输入： *http.request.method == GET and ip.addr == ip163* 然后，在过滤出来的packets中找到URI为mp3的那个包，可以通过 *wget*之类的软件下载该URI对应的歌曲

### 1.2 直接从包中获取歌曲数据

在播放歌曲之前，打开wireshark。在歌曲加载完毕以后，在display filter中输入： *http.content_type matches audio* 过滤出来的packet（一般就一个）就包含了音频数据，在 *packet details* 窗口中点开 *Media Type*字段，鼠标右键点击 *Media type...* 那个字段，选择 *Export Selected Packet Bytes* ，然后将数据保存即可。
