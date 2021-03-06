---
layout: post
title: "MarkUs: Drop-in use-after-free prevention for low-level languages"
date: 2020-06-09 14:48:19 +0800
comments: true
categories: 
---

> 作者：Sam Ainsworth, Timothy M. Jones
> 
> 单位：University of Cambridge, UK
> 
> 会议: IEEE Symposium on Security and Privacy 2020
> 
> 原文链接: [MarkUs: Drop-in use-after-free prevention for low-level languages](https://www.computer.org/csdl/proceedings-article/sp/2020/349700b214/1j2LgfBq2fS)

## Abstract

​Use-After-free （UAF）漏洞对于像C,C++这样比较底层的语言来说，是一种非常常见的攻击点。攻击者对由程序员手动释放但之后又被错误重用的内存进行重新申请分配，从而达到修改数据，劫持控制流，甚至控制系统的目的。

​作者指出目前虽然已经开发了许多相关技术来处理这些漏洞，比如

* 定位所有可能的指针变量，并在其所指向内存被释放时，对其赋null。
* 延迟被释放的内存重用，以降低UAF的可能。
* 对每一次内存分配都使用单独的页表。

​	但是作者测试说明，这些方案在平均情况和最坏情况下，有比较高的性能和内存开销，或者覆盖的范围比较有限。

​	作者设计了 MarkUs 内存分配器，能在低开销下防止这种形式的攻击，可以在实际生产环境中部署。其造成的性能开销平均为1.1x，内存开销平均为1.15x，最坏情况是2x，相比之下，优于其他相关保护技术（**Oscar**，**Dhurjati**, **Adve**, **Dangsan**, **CRCount**, **pSweeper**）。

<!-- more -->

## Background

### UAF Attack

![image-20200608130107303](/images/2020-06-09/image-20200608130107303.png)

![image-20200608130149436](/images/2020-06-09/image-20200608130149436.png)

## Garbage Collection

### 标记-清除算法

#### 	标记阶段

​		该算法从根出发，通常是 Stack, BSS, Data 和 寄存器 存储的堆内存指针，进行遍历，找到所有可达的堆内存对象，标记为存活对象。

#### 清除阶段

​		在堆内存内，对所有未被标记的内存对象（垃圾对象，通常情况下永远不可能被用户访问）进行释放。

#### 存在的问题

* 如果对内存指针进行XOR等运算后进行存储，Garbage Collection 会错误地将该内存释放而造成UAF。
* 因为保守的垃圾收集算法将所有可能的数据都当成了指针，巧合的数据可能指向不再需要的内存，导致其不被释放回收，增大了内存和时间开销。
* 一般的垃圾收集器还提供了手动释放的功能，于是依然可能造成UAF，甚至Double Free的问题。



## 威胁模型

* 攻击者能够正常的申请**分配内存**，并在申请的内存区域内，**读写数据**。
* 攻击者希望**劫持程序控制流**。
* 只考虑**堆内存**的分配和释放。



## MarkUs

### 整体框架

​		MarkUs 内存分配器，主要针对C/C++类比较底层的语言，将 free/delete 函数替换成 MarkUs 实现的函数。程序员在调用 free/delete 释放内存时，内存对象不会被直接释放到空闲表中，而是将内存对象加入到隔离列表中。直到该内存对象被标记为 **重新分配是安全的** 的时候，它才会被真正释放到内存分配器的空闲列表中，等待重新分配。

### 流程实例：

​		1. 程序调用 free/delete 释放内存 ( A D F H )，被加入到 **Quarantine List**

​		2. MarkUs 标记函数被调起，从根（ Stack, BSS, Data 和 寄存器 ）开始遍历，记录每一个遍历到的可能的堆指针（ B C D E G I J K），标记**Quarantine List** 中被遍历到的指针（即该内存可能被悬空指针所指向，不能直接释放) 。

​		3. 释放**Quarantine List** 中未被标记的内存，放入空闲表。

![image-20200608153148302](/images/2020-06-09/image-20200608153148302.png)

​		



## 优化设计

### 标记程序的频率控制

​		因为 MarkUs 的标记过程有较大的内存和性能开销，所以需要控制其标记过程中执行的频率来平衡其效率与开销。-

![image-20200608191343708](/images/2020-06-09/image-20200608191343708.png)

​		*qlsize* 代表目前在隔离列表中等待标记确认释放的空间大小，*failed_frees* 代表上一轮标记中，未被标记为可以安全释放的空间大小，*usize* 和 *unmapped* 分别代表隔离列表中和堆空间中未被映射到物理页中的虚拟内存空间大小。当表达式为真值时，进行标记。

* 总地来说，即只有当程序员手动释放（实际上是被加入**Quarantine List** ）的空间足够大时，才会触发 MarkUs 程序，进而真正将内存释放，放入空闲表等待分配。

  

### 内存释放优化

​		作者将待释放的内存根据大小分成2个部分：超过一个页面大小的内存，小于一个页面大小的内存，并分别采用了 **Page Unmapping** 和 **Small-Object Block ** 方法。

#### Page Unmapping

​		对于程序员申请的较大的内存，如果其完全覆盖了整个页面，那么程序员主动释放的该内存时，就无需对其进行悬空指针的扫描标记，直接将完全覆盖的页面munmap 解除物理页的映射。在这种情况下，即使有悬空指针对该虚拟内存进行访问，因为已经解除物理页的映射，所以会触发段错误。这样就避免了UAF。同时也降低了内存和性能开销。

#### Small-Object Block 

​		针对小于一个页面的内存，作者使用 **Small-Object List**  来记录即将释放的内存块。然后，将该 list 内所有内存块所在页面的空闲链表中的块一并加入list。如果存在一个页面中所有内存块都包含在该 list 中，则 munmap 该页面，以便于其重用，减小内存开销。 

![image-20200608213245867](/images/2020-06-09/image-20200608213245867.png)

​		

### Concurrency and Parallelism

* MarkUs 是支持并行的。它可以同时在多个线程上运行，方法是分割当前要搜索的指针的对象边界，使得在多核上的标记过程更快、更有效。
* MarkUs 是与应用进程是同时执行的，但是由于可能在扫描堆指针的过程中，页表数据可能发生了改变，因此为了保证标记结果的正确性，MarkUs对在扫描过程中被修改的页表置 dirty bits。在扫描完成之后，对置 dirty bits 的页表重新进行扫描可能的堆指针。



### 评估测试

​		作者将 MarkUs 实现为一个共享库，覆盖了原来的 malloc, free, calloc, realloc, new, delete 函数。

​		评估的比较对象是 **Oscar**，**Dhurjati**, **Adve**, **Dangsan**, **CRCount**, **pSweeper** ，他们都是避免UAF漏洞的相关技术实现。

* 在对性能和内存的影响方面，MarkUs 平均最低的，额外性能开销平均为10%，额外内存开销平均为16%。

​		![image-20200608194312704](/images/2020-06-09/image-20200608194312704.png)

​		作者还评估了以下情况下 MarkUs 的额外开销情况：

* 在指针密集型的项目中，MarkUs 相比与采用 **page-table entry per allocation** 的技术，有极大的优势。

  ![image-20200609003022905](/images/2020-06-09/image-20200609003022905.png)

* 在多线程情况下，MarkUs 所带来的额外性能开销也较低。

  ![image-20200609003342670](/images/2020-06-09/image-20200609003342670.png)

* 标记程序的隔离列表大小的选择：

  ![image-20200609004451594](/images/2020-06-09/image-20200609004451594.png)

  横坐标代表**Quarantine List** 的大小
  
  * 隔离列表越大，标记的频率越低，所以内存额外开销越大，性能影响小，CPU负载也更低。
  
  * 对于 **Deall** 项目，其明显不同。作者认为是因为该项目主要是大内存的分配，而其内存的释放由于被 **page unmapping** 机制优化，直接被解除物理页映射而释放掉，所以在间隔列表较小时，高频的扫描浪费了CPU和性能，而内存开销影响较低。（因为解除了物理页映射，所以内存开销低于1，优于原本的分配器）。
  
    
  
* 对于3种优化策略（标记程序的频率优化，Page Unmapping，Small-Object Block ）作者也进行了评估

  ![image-20200609125640506](/images/2020-06-09/image-20200609125640506.png)

  在不同的项目中，由于项目的特征不一样，不同优化手段带来的收益也是不同的。

## 限制

#### MarkUs 可能存在的缺陷

* 有可能存储的数据碰巧在堆的内存范围内,导致该空间长期不能释放，造成性能与内存开销。
* 有可能寻找到的堆指针在以后是不能被用户所使用，不会造成UAF，但所指向的区域不能被正确地释放，造成额外开销。
* 对于用户通过XOR等运算隐藏的指针，无法被正确判断为悬空指针，可能造成UAF。

针对问题一、二，作者认为对于较小的内存块，其造成的影响是极低的，因此可以忽略。而对于较大的内存，由于Page Unmapping 优化机制的存在，其内存仍然可以被正确的释放，不会造成很大的内存开销。所以相较于一般的垃圾收集器而言是比较高效的。

而对于问题三，作者认为 MarkUs 只会释放被用户主动请求释放的内存，避免了像垃圾收集造成的错误释放（因为标记-释放的垃圾收集机制会直接释放没有被指针直接指向的内存），减小了UAF的危险。尽管该问题是任何涉及标识指针的技术的一个难以避免的限制，但是因为绝大部分指针不会被隐藏、能够被正常的释放和使用，所以作者认为该问题带来的影响是很小的。



## 总结

​		作者设计的分配器核心优点

* 避免了绝大部分UAF的可能
* 额外开销低，能直接用于生产环境