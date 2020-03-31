---
title: LevelDB skiplist.h 源码
date: 2020/03/31 18:15
categories: 
- 数据库
tags: 
- LevelDB
- code
---

# 任务

1. 在skiplist.h中写入函数声明:

   ![image-20200326160557408](/../../media/2020-03-31-LevelDB-skiplist/skiplist.h.assets/image-20200326160557408.png)

# 源码

1. 枚举类: 定义了跳表最大高度kMaxHeight

   ![image-20200326160646372](/../../media/2020-03-31-LevelDB-skiplist/skiplist.h.assets/image-20200326160646372.png)

2. 成员变量: 

   ![image-20200326160806357](/../../media/2020-03-31-LevelDB-skiplist/skiplist.h.assets/image-20200326160806357.png)

3. 创建新节点:

   ![image-20200326160934328](/../../media/2020-03-31-LevelDB-skiplist/skiplist.h.assets/image-20200326160934328.png)