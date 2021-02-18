---
title: "重学数据结构--链表"
date: 2020-08-13T20:24:14+08:00
lastmod: 2020-08-13T20:24:14+08:00
draft: false
keywords: ["Go"]
description: "golang实现链表"
tags: ["Go"]
categories: ["golang"]
author: "Jeremy-boo"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: true
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
reward: true


# You unlisted posts you might want not want the header or footer to show
hideHeaderAndFooter: false

# You can enable or disable out-of-date content warning for individual post.
# Comment this out to use the global config.
#enableOutdatedInfoWarning: false

flowchartDiagrams:
  enable: false
  options: ""

sequenceDiagrams: 
  enable: false
  options: ""

---

<!--more-->

## 1. 链表介绍
### 1.1 链表的定义
> 链表是一种在物理存储上非连续、非顺序的存储结构，数据元素的逻辑顺序是通过链表中的指针链接顺序依次实现的

![linked-list](/images/linked-list.PNG)

### 1.2 链表的分类
> 链表主要分为单向链表、双向链表和循环链表
- 单向链表：单向链表中只表示 next的关联，最后一个节点的next为空--> next表示指向下一个节点的指针
如图所示：
![single-list](/images/single-list.PNG)
- 双向链表：双向链表中即包含next指针，也包含了prev指针，即节点中可以指向下一个节点地址，也可以指向前一个节点地址，首尾指针除外
如图所示：
![double-list](/images/double-list.PNG)
- 循环链表：循环链表表示收尾相连，构成循环。

  1. 单向循环列表： 

  2. 双向循环列表：

## 2. golang实现链表
### 2.1 单链表

