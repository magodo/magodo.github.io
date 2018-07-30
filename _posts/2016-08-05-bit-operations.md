---
layout: "post"
title: "位操作技巧"
categories: blog
tags: ['algorithm']
published: true
comments: true 
script: [post.js]
---

* Do not remove this line (it will not be displayed)
{: toc}

# 1. 将2's-complement的有符号数整体转为大小关系相同的无符号数

例如有带符号数(4bit): 
    
    0111, 1001, 0001, 0000, 1111, 1000

它们是从最大到最小的排列顺序。要使它们转变为相同大小关系的无符号数，则只需将最高位取反，其他位不变即可。通过下面的位操作：

    for (i = 0; i < sizeof(singed_arr)/sizeof(signed_arr[0]); i++)
    {
    	unsigned_arr[i] = signed_arr[i] ^ (1U << (4 - 1));
    }
