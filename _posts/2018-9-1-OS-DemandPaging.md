---
layout:     post
title:      "请求分页管理方式"
subtitle:   " 请求分页管理 "
date:       2018-09-01 21:40:00
author:     "Aliyang"
header-img: "img/post-bg-OS-DemandPaging.jpg"
tags:
    - 操作系统
---
在请求分页系统中，只要求当前需要的一部分页面装入驻村，在执行过程中，再通过**调页**功能将所缺的页调入，同时通过**置换**功能将暂时不用的页面换到外存。请求分页系统需要页表机制、缺页中断机构和地址变换机构的支持。
## 页表机制
|页号 | 物理块号 | 状态位P | 访问字段A | 修改位M | 外存地址|

* 状态位P：表示该页是否已经装入内存，作为程序访问时的参考
* 访问字段A：用于记录本页在一段时间内被访问的次数，或者记录本页已有多长时间未被访问
* 修改位M：表示该页调入内存后是否被修改过
* 外存地址：指出该页在外存上的地址，通常是物理块号

## 缺页中断机构
请求分页系统中，每当所要访问的页在内存中没有，就会产生一次缺页中断，请求操作系统会将所缺的页调入内存。如果此时内存中有空闲块，就会把所缺的页调入内存，否则的话，从当前内存中置换页出去，然后再调入，缺页中断和一般中断的区别：

* 在指令执行期间产生和处理中断信号，并不是在一条指令执行完后才处理，属于内部中断
* 一条指令在执行的时候，可能产生多次中断

## 地址变换机构
借用网上的图说明整个过程：
![请求分页](https://github.com/CoderAssassin/markdownImg/blob/master/operate_system/qingqiufenye.jpg?raw=true)

## 页面置换算法
当出现缺页的时候内存没有空闲空间存放页面，那么需要将内存中现有的页面置换出去，页面置换算法就是决定哪些页面要置换出去
#### 最佳置换算法(OPT)
最佳置换算法选择以后永不使用的、或者在最长时间内不再被访问的页面置换出去。
注意：因为我们无法预知将来，无法直到在未来的一段时间内哪个页面不再被访问，所以该算法无法实现。

#### 先进先出页面置换算法(FIFO)
将当前内存中最先进来的页面置换出去。算法的思想很简单，但是**最先进来的页面可能是经常被访问的**，所以如果将经常被访问的页面置换出去，接下来可能又要将该页面换进来，这样就增加了很多次的开销。在极端的情况下，我们按照1,2,3,4的顺序加入能存放4页的内存，接下来要访问页5，此时需要置换一页出去，那么就将1置换出去，假设接下来要访问1，那么又将2置换出去，一直以1,2,3,4,5的顺序访问，那么一直都在置换。
另外，FIFO因为其特性，所以**可能出现Belady异常，也就是说，当内存页面数增加的时候，置换的次数不减反增，只有在FIFO算法中才会出现这种现象。**

#### 最近最久未使用置换算法(LRU)
将最近最久没有使用过的页面替换出去。要实现该算法要用访问字段记录上次访问到现在经过的时间，在置换的时候选择值最大的替换出去。

#### 时钟置换算法(CLOCK)
又叫最近未用(NRU)算法。因为LRU算法的实现开销比较大，FIFO实现简单但是性能差，所以设计出了CLOCK这样的开销小的并且性能接近LRU的算法。
简单版CLOCK基本思想：
给每一个帧关联一个附加位(使用位)。当某一页第一次装入主存的时候，将该帧的使用位设为1；并且在该帧后边再被访问的时候继续把使用位设为1。将所有的候选帧集合看做一个循环缓冲区，并且有一个指针，当一页被替换时，该指针设置成指向缓冲区的下一帧。当要替换一页时，操作系统扫描缓冲区，查找使用位为0的帧。在扫描的过程中，若碰到使用位为1的帧，将使用位设为0。比如最开始的时候，第一个帧用来替换；当所有帧都是1的时候，会把所有帧都设为0然后找到第一个帧的位置，设置为1，替换该帧的页。
改进型的CLOCK置换算法：
加了一个字段，修改位，用来判断该页是否被修改过，两个字段会形成如下四种情况：

 * 最近没被访问，没被修改(u=0,m=0)
 * 最近被访问，没被修改(u=1,m=0)
 * 最近没被访问，被修改(u=0,m=1)
 * 最近被访问，被修改(u=1,m=1)

基本过程如下：

1. 首先从指针的位置开始，扫描缓冲区，不修改使用位，当碰到第一个(u=0,m=0)的帧，替换
2. 若1失败，那么重新扫描，找到第一个(u=0,m=1)的帧，替换，并且过程中将经过的帧的使用位设为0
3. 若2失败，从指针最初位置开始，此时所有帧的使用位都是0，继续1的步骤开始，重复