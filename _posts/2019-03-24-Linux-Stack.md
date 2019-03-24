---
layout:     post
title:      "Linux中的栈结构"
subtitle:   "Stacks in Linux"
date:       2019-03-24
author:     "TXXT"
catalog:    true
tags:
    - Operating System
    - Linux
---

栈有很多作用：比如函数调用或者多任务支持。

下图是**函数栈帧**的结构：

![å½æ°è°ç¨æ çå¸ååå­å¸å±](/img/in-post/Linux中的栈.assets/20160901214853559)

Linux中有四种栈：**进程栈 线程栈 内核栈 中断栈**

# 1 进程栈

属于用户态栈。进程地址空间如下：

- 程序段 (Text Segment)：可执行文件代码的内存映射 
- 数据段 (Data Segment)：可执行文件的已初始化全局变量的内存映射 
- BSS段 (BSS Segment)：未初始化的全局变量或者静态变量（用零页初始化） 
- 堆区 (Heap) : 存储动态内存分配，匿名的内存映射 
- 栈区 (Stack) : 进程用户空间栈，由编译器自动分配释放，存放函数的参数值、局部变量的值等 
- 映射段(Memory Mapping Segment)：任何内存映射文件

看到栈区了嘛？那就是我们所认为的进程栈。

![Linuxæ åè¿ç¨åå­æ®µå¸å±](/img/in-post/Linux中的栈.assets/20160901214948512)

进程栈初始化大小是由编译器和链接器计算出来的，实时大小不固定。最大限制是 `RLIMIT_STACK`，一般8MB。

内核使用内存描述符来表示进程的地址空间，该描述符表示着进程所有地址空间的信息。内存描述符由 mm_struct 结构体表示，下面给出内存描述符结构中各个域的描述，结合前面的 进程内存段布局 图一起看：

```c
struct mm_struct {
    struct vm_area_struct *mmap;           /* 内存区域链表 */
    struct rb_root mm_rb;                  /* VMA 形成的红黑树 */
    ...
    struct list_head mmlist;               /* 所有 mm_struct 形成的链表 */
    ...
    unsigned long total_vm;                /* 全部页面数目 */
    unsigned long locked_vm;               /* 上锁的页面数据 */
    unsigned long pinned_vm;               /* Refcount permanently increased */
    unsigned long shared_vm;               /* 共享页面数目 Shared pages (files) */
    unsigned long exec_vm;                 /* 可执行页面数目 VM_EXEC & ~VM_WRITE */
    unsigned long stack_vm;                /* 栈区页面数目 VM_GROWSUP/DOWN */
    unsigned long def_flags;
    unsigned long start_code, end_code, start_data, end_data;    /* 代码段、数据段 起始地址和结束地址 */
    unsigned long start_brk, brk, start_stack;                   /* 栈区 的起始地址，堆区 起始地址和结束地址 */
    unsigned long arg_start, arg_end, env_start, env_end;        /* 命令行参数 和 环境变量的 起始地址和结束地址 */
    ...
    /* Architecture-specific MM context */
    mm_context_t context;                  /* 体系结构特殊数据 */

    /* Must use atomic bitops to access the bits */
    unsigned long flags;                   /* 状态标志位 */
    ...
    /* Coredumping and NUMA and HugePage 相关结构体 */
};
```

![mm_struct åå­æ®µ](/img/in-post/Linux中的栈.assets/20160901215036575)

# 2 线程栈

主要区别在于是否共享地址空间。线程创建的时候将其 **线程的内存描述符直接指向父进程的内存描述符**。

线程的栈是在fork的时候生成，复制了主进程的stack空间。

# 3 进程内核栈

进程的生命周期中，必然会通过系统调用陷入内核，执行系统调用内核之后，这些内核代码所使用的就是内核栈。`thread_union` 进程内核栈和 `task_struct` 进程描述符紧密联系。由于内核经常访问 `task_struct` ，用头部的一段存放 `thread_info` 结构体。关系下图：

![è¿ç¨åæ ¸æ ä¸è¿ç¨æè¿°ç¬¦](/img/in-post/Linux中的栈.assets/20160901215111055)

# 4 中断栈

中断也是如此，当系统收到中断事件后，进行中断处理的时候，也需要中断栈来支持函数调用。

不过这个不是一定要独立的，有些例如arm是和内核栈在一起的。

# 为什么要区分这些栈？

1.为什么要单独的进程内核栈？

因为通过调度器让出A程序后，A进程的信息保存在自己的内核栈，B不能进入A的内核栈，不然A的数据被破坏。

2.为什么要单独的线程栈？

Liunx调度不区分线程进程，当调度程序要唤醒进程，就要恢复进程上下文环境，就是进程栈。如果没有线程栈，各个线程之间就会互相破坏函数调用环境。

