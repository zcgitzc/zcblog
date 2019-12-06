---
layout:     post
title:      "分布式事务"
subtitle:   "分布式事务实现方式"
date:       2019-11-23
author:     "zark"
header-img: "img/in-post/2016.03/07/post-markdown-introduce.jpg"
tags:
    - 分布式事务
---
CAP定理：
一致性(Consistency) ： 客户端知道一系列的操作都会同时发生(生效)
可用性(Availability) ： 每个操作都必须以可预期的响应结束
分区容错性(Partition tolerance) ： 即使出现单个组件无法可用,操作依然可以完成

参考：https://blog.csdn.net/yeyazhishang/article/details/80758354

2PC（两阶段提交）：
https://blog.csdn.net/lezg_bkbj/article/details/52149863


BASE理论：
https://www.jianshu.com/p/9cb2a6fa4e0e