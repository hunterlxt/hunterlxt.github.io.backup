---
layout:     post
title:      "操作系统的三个主题——虚拟化"
subtitle:   "The part 1 of Operating Systems: Three Easy Pieces"
date:       2018-09-01
author:     "TXXT"
catalog:    true
tags:
    - Operating System
---

> 参考：*Operating Systems: Three Easy Pieces*
>
> OS是一个工程化色彩很强的CS基础课程，因为跨专业的原因，没有完整学习过这门课程，现在还债了……

# 2. Introduction

一个running program做一件很简单的事：执行指令。processor **fetches** an instruction from memory, **decodes** it, and **executes** it. 处理器一直执行指令直到程序跑完。 OS希望system操作得既正确又易于使用。

OS实现上述主要通过 **virtualization**。OS把物理资源转变成更通用易用的形式。所以有时候OS就视作一个virtual machine。为了让users告诉OS想做什么，OS提供了一些API。一般OS都提供了几百个**system call**给你调用。或者我们也说OS提供了一个 **standard library** 给应用程序。

## 2.1 Virtualizing the CPU

![1544359214933](/img/in-post/虚拟化virtualization.assets/1544359214933.png)

```
./cpu A & ; ./cpu B & ; ./cpu C & ; ./cpu D &
```

你会发现4个程序都执行了，但是真实的CPU只有一个，是不是很神奇！

## 2.2 Virtualizing Memory

![1544360428875](/img/in-post/虚拟化virtualization.assets/1544360428875.png)

![1544360441028](/img/in-post/虚拟化virtualization.assets/1544360441028.png)

两个进程（对应两个PID）分别都在地址00200000分配了空间，然后修改的数据居然互不影响。这是因为每个程序都跑在自己的 private memory 中。事实上，这里正在发生的就是 **virtualizing memory**。每一个程序都在访问自己的私有的 **virtual address space** ，有时候称作 **address space** ，这段空间是OS映射在物理内存上的一部分。

## 2.3 Concurrency

![1544430405868](/img/in-post/虚拟化virtualization.assets/1544430405868.png)

![1544431373912](/img/in-post/虚拟化virtualization.assets/1544431373912.png)

当参数小的时候，两个线程对同一个数字加1，最后得到输入的两倍值；但是，当输入很大后，有时候结果并不是两倍。这是因为 load add store 这3个指令并不是一起同时执行完的（**atomically**），而是分别执行，这就产生了 **concurrency** 的问题。

## 2.4 Persistence

第三部分就是持久化了。因为系统内存中，数据很容易就丢失了，因为DRAM是易失的。需要软件和硬件保证数据持久化。这种硬件以 i/o 设备形式存在。软件部分就是熟悉的 **file system**。

![1544432050745](/img/in-post/虚拟化virtualization.assets/1544432050745.png)

图2.6展示了创建一个文件并包含一串字符。程序进行了3个调用，第一个是open()，打开创建文件；第二个是wrie()，写一些数据进去；然后是close()。这就是大体上文件系统帮我们做的，帮我们找位置存起来，帮我们找文件等等。有了文件系统，程序员才能用几个接口就能写简单程序。

## 2.5 Design Goals

1. high performance （minimize the overheads）
2. protection （isolation）
3. reliability
4. others：energy-efficiency，security，mobility



# 4 The Abstraction: The Process

OS创建了CPU的一个虚拟假象：让一群process同时运行。这种基础的技术叫做 **time sharing of the CPU** , 通过 **context switch** ，OS可以停止运行的程序然后开始运行另一个。这种时分技术被所有的现代OS所采用。实现CPU虚拟化，低级别的 **mechanism** 和高级别的 **policy** 共同协作。比如上下文切换属于前者，而调度策略属于后者。

## 4.1 The Abstraction: A Process

运行的程序的抽象就是process。通过 taking a inventory of the pieces of the system，我们可以概述出一个process。理解构成process的东西就要理解process的 **machine state**。

machine state的一个构成部分就是 memory，指令都放在memory中，program用到的data也躺在那里。因此这部分 **address space** 是process的一部分。

registers也是machine state的一部分。

部分特殊功能寄存器也是machine state的一部分。举个例子，**program couter**指示当前所运行的指令。**stack pointer** 和对应的 **frame pointer** 用于管理函数栈的参数、局部变量、返回地址等等。

## 4.2 Process API

真实的API在后续章节，这里只给出idea：

- Create：创建进程，比如在shell中运行命令。
- Destroy：有创建就要有销毁。虽然大部分时候process自己exit出去，但有时还是需要别人的帮忙才能halt a process。
- Wait：让一个running process 停止一会。
- Miscellaneous Control：比如让一个程序停止一会又恢复它
- Status：应该提供一些接口来获取status information

## 4.3 Process Creation: A Little More Detail

现在我们讨论程序创建成进程的一些细节。第一件OS要做的就是 **Load** 程序代码和静态数据到内存到 address space of the process。在简单的OS中，**loading process** 必须在运行程序前做完，但是现代的OS比较懒，它们会在后续需要时才加载一些代码和数据。后面讲到 **paging and swapping** 时会更加深入。

当Load之后，还有一些事情需要做。必须分配一些memory用于程序的 **run-time stack (or Stack)** 。 OS也会用参数初始化这个stack，而且会把参数写给main函数。

OS也许还会分配一些内存给程序的 **heap** 。在C程序中，heap被用于申请的 dynamically-allocated data；程序通过 malloc() 申请，通过free() 释放。heap和连接表，哈希表，树等数据结构息息相关。刚开始heap可能很小，随着运行，OS也许会分配越来越多的内存给 process 以保证调用的安全。

OS还会做其它的初始化工作，比如和I/O相关的。比如，在UNIX中，每个process默认有3个 open **file descriptor** 用于标准I/O和error；这些descriptor让程序易于从终端读入，并输出到屏幕。通过加载代码、静态数据到内存，创建和初始化 stack，还有其它的工作例如 I/O设置等等，OS最终为程序的执行准备好了大舞台。

PS：现在还有最后一步没讲，程序的入口是main()，通过跳转到main() routine，OS才把CPU的控制移交给新创建好的process，这个程序才真正运行起来了。

## 4.4 Process States

 以简单的视角，一个process是三种状态之一：

- Running：running on a processor.
- Ready：ready to run but OS not chose to run it.
- Blocked：percformed some kind of operation that makes it nor ready.

![1544602258924](/img/in-post/虚拟化virtualization.assets/1544602258924.png)

## 4.5 Data Structures

OS也是一个程序，就像其它的任务程序一样有关键的数据结构。为了追踪每个process的状态，OS会为那些ready的进程维护某种 **process list**，当然还有那个Process正在running的信息，还要维护那些 blocked process 的信息。 

以xv6为例，下图展示了OS需要追踪的process相关信息。首先， 为停止的Process保存**register context** （相关寄存器的信息）。

除了几个基本state外，进程还有 **initial** 和 **final** 状态，前者是正在creating，后者是已经exited但是还未完全清理干净（在unix系统中叫 **zombie state**）这个状态很有用，可以允许其它进程来坚持它是否成功运行了。这以后，parent才会进行 final call 来等待完成所有工作并通知OS可以清理了。

![1544607056105](/img/in-post/虚拟化virtualization.assets/1544607056105.png)

## 4.6 Summary

学习了进程，从低级角度看，还要知道实现的具体机制；从高级角度看，还要学习调度的策略等。



# 5 Process API

UNIX用一对系统调用：fork()和exec() 提供了创建新process的方法。第三个调用 wait() 用于一个process等待一个自己创建的process的完成。

## 5.1 The fork() System Call

![1544628427491](/img/in-post/虚拟化virtualization.assets/1544628427491.png)

fork用于创建一个新的process，以上面的程序为例。第一次开始运行后打印了自己的PID。然后它进行了fork，fork实际上拷贝了一份calling process，现在有两个process在运行了，但是它们的PID不同，孩子将会从fork返回一个0，但创建者会得到它创建的进程的PID。

PS：这里后续的两个print谁先谁后不知道，因为这取决于CPU的调度分配器。

## 5.2 The wait() System Call

![1544680104576](/img/in-post/虚拟化virtualization.assets/1544680104576.png)

这个调用函数用于parent等待它的child进程完成工作。比如在示例程序中，parent process调用wait()来延迟执行直到child完成了执行工作。当child做完之后，wait()就会返回做完的进程的PID到parent。

## 5.3 The exec() System Call

![1544681422017](/img/in-post/虚拟化virtualization.assets/1544681422017.png)

fork出来的程序一样，但是往往你想运行的是不同的程序，exec()就是干这个的。但是，它做的是从wc程序开始，用wc的代码和数据覆盖自己的进程信息，OS重新用wc初始化了这个进程，并没有创建什么新的进程。这也是为什么后面的printf没有打印的原因。总而言之，一次成功的对exec()的调用不会返回。

## 5.4 Why?

fork()和exec()的不同之处在构建UNIX shell上非常关键。因为这样就能让 shell 在fork之后，exec之前做一些 about-to-be-run program，因此带来了很多特性。

The shell is just **a user program**. 当你敲入了执行命令后，shell会找到文件系统中executable存在的地方，然后调用fork()创建一个新的child process来运行command，具体来说执行exec()的某个变种调用执行command，通过wait()来等待命令的完成。当child完成之后，shell返回到wait()并等待用户的下一次命令输入。

Unix pipes 用相似的方式实现，但是通过 pipe() 系统调用完成的。grep 和 '|' 什么的用法自己搜吧。

## Homework (code)

1. 同一个变量，哪怕malloc分配到heap中的变量，fork之后就互相独立了，互不影响；

2. pipe这个调用创建管道，用于有同父进程的进程之间通信，采用先进先出原则。任何写到管道[1]的数据都能从管道[0]读取。因为使用的是文件描述符而不是文件流，所以要用底层的read和write函数，不能用fwriteh额fread。

   管道的优势在于如果想在两个进程传递数据，当程序用fork调用创建新进程时，原先打开的文件描述符仍将保持打开。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <fcntl.h>
#include <sys/wait.h>
#define BUFSZE 32

int
main(int argc, char *argv[])
{
    // Setup our pipe
    char buff[BUFSZE];
    int p[2];

    if (pipe(p) < 0)
      exit(1);

    int rc1 = fork();
    if (rc1 < 0) {
        // fork failed; exit
        fprintf(stderr, "fork #1 failed\n");
        exit(1);
    } else if (rc1 == 0) {
	     // Child #1
       printf(" Child #1 ");
       close(p[0]);   // This one only writes
       dup2(p[1], 1); // redirect stdout to pipe write
       printf("_This is getting sent to the pipe_");
    } else {
        // Parent process
        int rc2 = fork();
        if (rc2 < 0) {
          fprintf(stderr, "fork #2 failed\n");
          exit(1);
        } else if (rc2 == 0) {
          // Child #2
          printf(" Child #2 ");
          close(p[1]);      // Only read here
          dup2(p[0], 0);    // Redirect stdin to pipe read

          char buff[512];   // Make a buffer
          read(STDIN_FILENO, buff, 512); // Read in from stdin
          printf("%s", buff);     // Print out buffer

        } else {
          // Initial parent process
          /* If we wait for rc1 then it could finish before rc2 is done,
           * giving us some strange behavior. */

          int wc = waitpid(rc2, NULL, 0);
          printf("goodbye");
        }
    }
    return 0;
}
```



# 6 Limited Direct Execution

有很多问题在虚拟化CPU这件事上。首先就是 **performance** ：减少系统的overhead；第二个就是 **control** ：在有效率运行process的同时保持对CPU的控制。Attaining performance while maintaining control 是构建操作系统的核心挑战。

## 6.1 Basic Technique: Limited Direct Execution

direct execution很简单，就是OS在process list创建一个process entry，分配一些内存空间，加载代码，定位main位置，跳转过去，开始运行用户代码。下图所示：

![1544692342630](/img/in-post/虚拟化virtualization.assets/1544692342630.png)

但OS怎么确定程序不会乱搞事情呢，而且这么做怎么实现time sharing呢，不加limited，那OS岂不是就和 library 一样了，事情显然不是这么简单。

## 6.2 Problem #1: Restricted Operations

硬件能够帮助OS提供不同的执行模式。在 **user mode** ，应用没有完全的访问硬件资源的权限。在 **kernel mode** ，OS就能完全访问硬件。有 **trap** 和 **return-from-trap** 这种特殊的指令来实现这种功能。

在X86中，当trap时，processor会把program counter，flags和一些register压入每个process都有的那个 **kernel stack**。执行 return-from-trap指令后，将会把这些压入的东西弹出恢复 user-mode 程序的执行。

当进入kernel后并不是user来告知该运行什么代码，因为这样就不安全了。实际上kernel在boot time时设置了**trap table**。OS就能告诉硬件当特定的事件发生后该运行什么代码。OS通知了硬件 **trap handlers** 的位置，这样在硬件就能根据对应的事件转移到对应的handler代码上，由kernel负责完成继续的工作。

![1544709781578](/img/in-post/虚拟化virtualization.assets/1544709781578.png)

上图展示了整个流程，特权指令被加粗显示。

## 6.3 Problem #2: Switching Between Processes

解决了上一个问题，那么怎么在两个processes之间切换呢？当一个process在运行，OS就没有运行，既然没有运行，怎么决定切换呢？问题就是CPU怎么恢复控制然后切换进程呢？



**Cooperative Approach: Wait for System Calls**

这是一个过去的系统常常用的方法。这种 **cooperative** 方式，OS信任系统中每个进程的行为是合理的。进程被假设为周期性地自愿放弃CPU来让OS决定执行什么任务。这样的系统还有一个显示的 **yield** system call，唯一作用就是移交控制给OS。

当然非法操作还是会触发 trap 的。言外之意，如果程序一直没有非法，那么理论上它可以一直占据CPU。



**Non-Cooperative Approach: The OS Takes Control**

想解决前面的问题，可以加入 **timer interrupt** 。timer可以设定每隔多久就中断一次，然后进入预设的 **interrupt handler** ，将控制移交给OS。

这里补充几点：1.OS必须在boot time通知硬件当timer中断时运行哪里的代码。 2.在boot步骤中，OS要启动这个timer，这也是个特权指令。

而且这种模式下，hardware负责在中断时保存程序的相关状态到kernel stack （听起来和trap模式差不多）



**Saving and Restoring Context**

OS获取控制之后根据是否继续运行之前的程序做出决定，如果决定switch，那么执行一段 low-level code called **context switch**。

首先把当前进程的状态存到kernel-stack。OS执行一些low-level assembly code为了把 general purpose registers, PC, kernel stack pointer存储起来。

并且restore for the soon-to-be-executing process from its kernel stack。恢复下一个进程的PC和register等等。这样就能确保 return-from-trap 之后，下一个process能成为 currently-running process。

PS：硬件和OS都会保存一次register，只不过硬件是隐形的，OS只有在确认要切换时才显式地保存这个信息。

![1544755511040](/img/in-post/虚拟化virtualization.assets/1544755511040.png)



## Homework (Measurement)

这个homework测试一下系统调用和上下文切换的开销。

比如可以重复执行一个系统调用，然后总时间除以次数就好。



# 7 Scheduling: Introduction

## 7.1 Workload Assumptions

首先我们把跑在系统上的process称作：**workload**。 决定workload是调度策略的首要任务。但这里我们的 assumptions **不很现实**，不过这不是大问题：

我们对processes or jobs running in the system作出假设：

1. Each job runs for the same amount of time.
2. All jobs arrive at the same time.
3. Once started, each job runs to completion.
4. All jobs only use the CPU (they perform no I/O)
5. The run-time of each job is known.

## 7.2 Scheduling Metrics

除了assumption，我们还要一个**scheduling metric**来衡量不同的调度策略。
$$
T_{turnaround}=T_{completioin}-T_{arrival}
$$
比如说如果所有任务同时到达，$T_{arrival}=0$ ，那么 $T_{turnaround}=T_{completioin}$。**Fairness** 也是我们的一个metric。但，performance和fairness在scheduling中经常互相抑制。比如优化了性能，但是通过阻止一些jobs的运行为代价，言外之意，降低了fairness。

## 7.3 FIFO

假设A、B、C按照顺序几乎同时在time 0到达，A运行100s，其它运行10s，看样子 average turnaround time is $\frac{100+110+120}{3}=110$ 

是不是很像你在超市里看见前面一个人拿着一大购物车的东西等待付款的场景。

FIFO很简单，但有 **convoy problem** 问题。

## 7.4 Shortest Job First (SJF)

 就是字面意思，但是有个问题，如果B和C比A稍微晚一点到，还是要等A完成任务，所以又回到了 **convoy problem** 上。

## 7.5 Shortest Time-to-Completion First (STCF)

上面的两个都是 **preemptive scheduling**，因此一旦运行就会运行完。但是STCF是一种 **non-preemptive scheduling** 因此不会遇到 **convoy problem**。

当然如果所有 jobs 同时抵达，SJF是最优的，不过不同时到的话，STCF就是最优的（暂时）

![1544775381133](/img/in-post/虚拟化virtualization.assets/1544775381133.png)

## 7.6 A New Metric: Response Time

既然我们知道 job length了，并且jobs都只会用CPU，评价指标 turnaround time 够用了，STCF就是最优的策略了。

但是现在的电脑是时分系统了，用户用的是terminal，希望反应时间越快越好，一个新metric诞生了：**response time**。形式化来讲就是：
$$
T_{response}=T_{firstrun}-T_{arrival}
$$
举个例子，在STCF下，ABC的 response time 分别是0,0,10。这样后面的 jobs 的 response time 就很长，用户体验差，问题来了：how to build a scheduler that is sensitive to response time?

## 7.7 Round Robin

**Round-Robin Scheduling** 的基本概念很简单：RR runs a job for a **time slice** ，and then switches to the next job in the run queue. Repeatedly does so until the jobs are finished. 因为这样，RR有时候叫做 **time-slicing**。这个time slice长度必须是 timer-interrupt period的倍数。

正如所见，time slice长度对于RR很关键，太短的话切换开销太大，太长反应时间太慢，这里就展现了系统设计的 trade-off。

![1544797649896](/img/in-post/虚拟化virtualization.assets/1544797649896.png)

context switching 的开销不仅仅来自于register的保存恢复，而且还有CPU cache，TLB，branch predictors and other on-chip hardware。切换任务会导致这些东西都被 flushed，所以开销特别巨大。

但是之前的问题又来了，turnaround time在这里就没考虑到了！

## 7.8 Incorporaint I/O

之前一直不考虑I/O，现在考虑上。比如这个例子中，先执行A，然后执行B，因为A需要50ms的CPU使用。但是执行I/O的时候CPU空转，浪费了CPU。

![1544799132067](/img/in-post/虚拟化virtualization.assets/1544799132067.png)

下图7.9解决了这个问题，可以把A分解成5个10ms的sub-job，然后先调度一个sub-jobA，然后当A等待I/O的时候，只有B在候选里，所以调度50ms的job-B，当第一个I/O执行完成，等同于又一个sub-job-A抵达了。

![1544799204955](/img/in-post/虚拟化virtualization.assets/1544799204955.png)

## 7.10 Summary and Homework

现在有两种思想方法：1.运行候选中最短job，最优化turnaround time；2.在所有jobs中轮转，优化了response time。下一个chapter学习一个scheduler能解决这个trade-off难题，叫做 **multi-level feedback queue**。

- 对于有同样长度的jobs，SJF和FIFO的 turnaround time 是一样的。

- 对于SJF调度，job length和response time有一个线性增长关系。



# 8 Multi-Level Feedback Queue

多级反馈队列调度希望解决两个问题：第一，最优化 turnaround time ，但是现实情况OS往往不知道job运行多久，而这些都是 SJF or STCF 需要的前置条件；第二，MLFQ想要让系统尽可能responsive，最小化 response time（RR在 turnaround time方面做的很差）。

## 8.1 MLFQ: Basic Rules

显然MLFQ中有多个队列，每个队列的优先级不同，相同队列的jobs按照RR调度法。MLFQ varies the priority of a job based on its **observed behavior**。意思就是那些多次等待I/O的jobs，提高优先级；长时间密集使用CPU的就降低它们的优先级。通过历史来预测未来的行为。

有两条基本规则：

- Rule 1: 当 Priority(A) > Priority(B), A runs.
- Rule 2: 当 Priority(A) = Priority(B), run using RR.

![1544929184122](/img/in-post/虚拟化virtualization.assets/1544929184122.png)

## 8.2 Attempt #1: How to Change Priority

- Rule 3: 当 job 进入系统后，被放置在最高优先的队列中
- Rule 4a: 如果 job 用光了整个 time slice，它的优先级降低一档
- Rule 4b: 如果 job 在time slice之前放弃了CPU，保持同样优先级不变

至今为止，还有很多问题！第一，**starvation**，如果有太多的interactive jobs，long-running jobs就会一直得不到CPU而饿死；第二，有些jobs会愚弄scheduler，比如在时间片用完前发一个I/O来放弃CPU，这样几乎就独占了CPU；第三，有些jobs会改变自己的行为，比如从CPU密集型变成了interactivity。

## 8.3 Attempt #2: The Priority Boost

简单的idea就是周期性地 **boost** 所有jobs的优先级。

- Rule 5: 每个周期S之后，把系统中的所有jobs放到 topmost queue

![1544933478861](/img/in-post/虚拟化virtualization.assets/1544933478861.png)

上面的例子就对比了加入 rule 5 的优势，但是S该设置多大呢？其实S的设置就是一个 **voo-doo constants** 。意思就是没什么固定的方法去遵循，太大了，long-running jobs will starve, 太小了，interactive jobs 可能反应迟钝。

## 8.4 Attempt #3: Better Accounting

现在考虑该怎么避免调度器被小伎俩玩耍。用更新的 **Rule 4**

- Rule 4: 当一个任务在给定的 level 上用光了allotment，那么优先级就会降一级

这样的话，就算这个job在 time slice 快用光前主动 i/o，只要allotment用完了，那么也会降级，下图就是更新 rule 4 前后的对比。

![1545379921486](/img/in-post/虚拟化virtualization.assets/1545379921486.png)

## 8.5 Tuning MLFQ and Other issues

怎么参数化 MLFQ 呢？其实没什么标准，一切全靠经验，根据不同的 workloads 进行不同的调整。

比如大部分的MLFQ变种都根据不同的优先级队列，设置不同的 time slice，越高优先级的 slice 越短。solaris 用的是配置法，freebsd用的是数学计算法（根据历史行为）

## 8.6 MLFQ：Summary & Homework

- Rule 1: If Priority(A) > Priority(B), A runs (B doesn’t).
- Rule 2: If Priority(A) = Priority(B), A & B run in RR.
- Rule 3: When a job enters the system, it is placed at the highest priority (the topmost queue).
- Rule 4: Once a job uses up its time allotment at a given level (regardless of how many times it has given up the CPU), its priority is reduced (i.e., it moves down one queue).
- Rule 5: After some time period S, move all the jobs in the system to the topmost queue.



作业：

- 如果把队列数量设置为1，那么和RR调度法一模一样了



# 10 Multiprocessor Scheduling (Advanced)

一般的C程序都是单核的，不考虑多核，所以用多核就要针对并行去写程序，比如用多线程啥的。而且一个大问题就是 **multiprocessor scheduling** ！



# 13 The Abstraction: Address Spaces

## 13.1 Early Systems

从内存的角度，早期的系统没有什么内存的抽象层，所以呢，一般机器的物理内存看起来如下图13.1所示：

![1545725533985](/img/in-post/虚拟化virtualization.assets/1545725533985.png)

那时OS其实就是一些方法的集合（可以称为 library）。程序用比如64KB后的地址空间，一切都很简单！！！

## 13.2 Multiprogramming and Time Sharing

过了一段时间，因为机器的昂贵价格，用户希望更有效地共用机器。这个时候 **multiprogramming** 技术诞生了，OS可以在不同进程之间切换，一个用I/O的时候，就切到另一个（之前讨论了）。

后来 **time sharing** 技术也诞生了，但是在 memory 这一块有些问题。一种方式是在运行一个进程时，让它能完全访问所有的内存（上图），当切换时，就保存所有的信息到非易失性存储器上，加载另一个进程。但是这种方式有大麻烦，就是速度太慢！！因为每次都要保存一堆东西！！我们想要的是切换时进程信息还在内存，这样OS用时分技术就更加有效率。

![1545726227578](/img/in-post/虚拟化virtualization.assets/1545726227578.png)

以上图为例，这三个进程A、B、C，每一个拥有512KB内存切出的一小部分。后来呢，安全也被人注意了，一个进程应该被设置不访问其他的内存空间。

## 13.3 The Address Space

所以我们要做一个容易使用的物理内存的抽象给进程，这个抽象就叫 **address space** ，这是进程的一个看内存的视角。进程的地址空间包含了该程序用到的所有内存状态。比如：1.**代码** 2.**stack**（用来追踪在函数调用链中的位置）3.**heap**（动态内存分配用的，比如C++ new 一个东西）4.其它的杂七杂八（比如静态的初始化变量等）

![1545726748617](/img/in-post/虚拟化virtualization.assets/1545726748617.png)

上图的例子中，程序code部分在地址空间的顶端（0是顶），因为code是静态的嘛。然后有两个区域可以动态变化。这样放置 stack 和 heap 仅仅是个惯例。

当然我们描述图13.3时用的是OS提供的抽象层，真实情况是它们不可能从0kb位置开始，可能是某一个随意的地址上。所以OS做这些活时，我们说OS正在 **virtualizing memory**，因为程序自己以为自己从一个特定位置（比如 0KB）开始，有很大的一个地址空间（可能是32位 or 64位）。

![1545728470676](/img/in-post/虚拟化virtualization.assets/1545728470676.png)

## 13.4 Goals

我们让OS虚拟化内存有以下几个目标：

**Transparency**

OS应该让虚拟的内存对于运行程序来讲是透明的。因此程序不知道自己的内存被虚拟了，所以程序应该按照自己完全有独立私有的内存那样工作。

**Efficiency**

OS应该努力让虚拟化更有效率，在时间和空间两方面都有效率！比如为了时间效率，需要硬件的帮助，例如TLB。

**Protection**

OS保证每个程序免受其它程序的干扰。

## 13.5 Summary and Homework

总而言之，VM系统负责提供一个巨大的、稀疏的、私有的地址空间假象给程序。OS在某些硬件的帮助下，把虚拟地址翻译成实际的物理地址。

**作业：**

运行 `free -m` 看看系统里有多少空闲内存。然后写一个程序申请一堆内存，看看空闲内存有变化吗（没啥变化）。

但是运行 `pmap -X (pid)` 就看出来了，其实分配的内容都在，卧槽，没搞懂，这些东西到底在哪？为什么系统空闲内存一点都没变化。

# 14 Memory API

## 14.1 Types of Memory

程序会分配两类内存，一种叫 **stack** memory，由编译器帮你管理这部分分配的内存（自动化）。你要声明一块用于 stack 的内存很简单，比如：`int x;` 即可。当你从函数返回出去，编译器为你释放这部分。

如果要更长的存储寿命，应该分配到 **heap** memory中，这部分由你手动地控制。

## 14.2 The malloc() Call

void *malloc(size_t size);

size_t表示你要在参数中写明你要分配多少字节，不过一般不用数字的，用sizeof(类型名)。当然sizeof是一个操作符，并不是函数，函数意思是运行时发生的，但是sizeof能在编译期间完成工作。

malloc返回的是一个地址，我们可以加入类型转换以声明我们要怎么用这个返回的地址。

## 14.3 The free() Call

free接收一个参数，就是malloc返回的那个地址，你不用主动告诉它要free多少size，因为 memory-allocation library 自己会追踪这些的。

## 14.4 Common Errors

正确的内存管理一直是个难题，有些语言支持 **自动内存管理** ，自动回收 new 出来的空间，这种玩意叫 **garbage collector**。

**1 忘记分配内存**

举个例子，`strcpy (dst, src)`拷贝字符串的，但是不注意的话，可能会犯错：

![1545752745934](/img/in-post/虚拟化virtualization.assets/1545752745934.png)

这样就会遇到一个 **segmentation fault** (关于内存的错误)

![1545752943405](/img/in-post/虚拟化virtualization.assets/1545752943405.png)

改成上图这样就OK了。

**2 没有分配足够的内存**

比如少分配一个字符空间（因为字符串结尾还有一个 \0 ）

**3 忘记初始化分配的内存**

**4 忘记释放内存**

这样会导致内存泄漏，特别是那种长时间运行的大型程序，内存泄漏最终会耗尽内存。但是小程序的话，运行完OS就帮你释放了所有空间，因此没泄漏发生。

**5 重复释放内存**
**6 错误地调用 free()**

## 14.5 Summary and Homework

刚刚讨论地那些其实不是 system call，因为它们是库函数，是系统调用的封装。

一种关于这些的系统调用是 `brk`，用来改变程序的 `break`（heap结尾的位置）的位置。还有类似的调用是 `sbrk`。这些都是由内存分配的库来做的，所以一般意义上，程序员不直接使用这些系统调用。

有的时候你需要 匿名内存区域 ，所以你用了 `mmap()` 调用（某种系统调用），具体的东西看man手册吧。

**作业**

1.用GDB调试一个 free(指向空的指针)程序 null.c，看看GDB怎么报错？

段错误

```
Program received signal SIGSEGV, Segmentation fault.
0x000055555555469c in main () at null.c:7
7		int b = *p;
```

2.使用 `valgrind` 工具中的 `memcheck` 分析发生了什么，用以下命令试试：`valgrind --leak-check=yes a.out`。

访问 0x0 的区域没有映射内容。

3.写一个malloc但是不free，运行上述命令

```
==4509== HEAP SUMMARY:
==4509==     in use at exit: 1 bytes in 1 blocks
==4509==   total heap usage: 2 allocs, 1 frees, 1,025 bytes allocated
==4509== 
==4509== 1 bytes in 1 blocks are definitely lost in loss record 1 of 1
==4509==    at 0x4C2FB0F: malloc (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==4509==    by 0x10869B: main (in /home/lxt/c-test/a.out)
==4509== 
==4509== LEAK SUMMARY:
==4509==    definitely lost: 1 bytes in 1 blocks
==4509==    indirectly lost: 0 bytes in 0 blocks
==4509==      possibly lost: 0 bytes in 0 blocks
==4509==    still reachable: 0 bytes in 0 blocks
==4509==         suppressed: 0 bytes in 0 blocks
```

**花时间去学习 GDB 和 Valgrind ！！！**



# 15 Address Translation

在虚拟化CPU的时候，说过一个概念叫：有限的直接执行。意思就是程序一般是直接运行在硬件上，只有在特定的时候，才会唤醒OS去做某些任务。效率要求我们要用到硬件的帮忙，比如用寄存器、TLB。控制权要求OS确保没有应用程序能够访问其它的部分。

一般使用的技术是 **address translation** 。这是通过硬件的帮助，把指令中的虚拟地址翻译成实际所在的物理地址，以实现重定向到真正位置的目标。当然这不代表是硬件虚拟了内存，事实上硬件做是为了效率考虑的。

OS必须参与其中，这叫做 **mange memory** ，追踪哪些是free的，那些是used的。

## 15.1 Assumptions

我们的尝试非常简单，这种做法在以后学完多级页表时，会很可笑。先不管了，继续吧。

这里先假设用户的地址空间在物理内存中连续存放；每个程序空间也不会太大，反正比物理内存小就对了；每个程序空间都一样大。

听起来扯淡！但是后面我们会慢慢变得现实起来。

## 15.2 An Example

![1545819820843](/img/in-post/虚拟化virtualization.assets/1545819820843.png)

![1545819843581](/img/in-post/虚拟化virtualization.assets/1545819843581.png)

比如说上面这个程序，在物理内存就如上图所示。但是它自己以为自己从0开始到16KB。所以真实的物理内存中，只有1个 16KB 的slot是占用状态。

## 15.3 Dynamic Relocation (基于硬件)

这种方式需要两个寄存器，一个叫做 base 寄存器，一个叫做 bound 寄存器。所以呢当有内存引用时，`physical address = virtual address + base`

由进程生成的地址都叫做 **virtual address** ，由硬件做了这个翻译的工作。比如说，当执行一个PC对应指令，那么硬件先把这个PC的地址加上base基址，然后才会给processor处理。

因为这个技术是在 run-time 的时候，所以叫它 **Dynamic Relocation**

而另外一个 bounds 寄存器是为了保护的，还是在上图的例子，如果访问的内容超过了16KB，就是非法了。

这种协助性质的硬件有人叫做它 **memory management unit** (MMU) 。MMU不止这些简单的功能，后面我们会丰富它。

## 15.4 Hardware Support: A summary

现在总结一下至今我们从 **硬件** 部分获得的支持。

CPU在内核态时，可以用所有指令，用户态就不行，这是通过一个标志位 **processor status word** 来标识的。

base 和 bounds 寄存器是CPU 的 MMU 的一部分。用来翻译虚拟地址和物理地址。

用来修改 base 和 bounds 的指令是 **特权的** 。

而且CPU检测到越过 bounds 后，执行OS安好的 "out-of-bounds" **exception handler**。同理用户态试图执行特权指令时也有一个对应的 **exception handler**

![1545821288294](/img/in-post/虚拟化virtualization.assets/1545821288294.png)

## 15.5 Operating System Issues

OS和硬件结合才能实现虚拟化内存，这就是本节内容。

第一，当进程创建的时候，OS去物理内存中找空闲的 slot ，OS一直在追踪哪些是 free 哪些是 used。

第二，进程终止后，OS要回收这部分内存区域，清空用到的相关的数据结构。

第三，当进程切换时，OS要 保存 和 恢复 base-and-bounds pair，因为对于每个进程，这个pair都不一样。OS一般保存在PCB中。当OS决定停止一个进程时，有时会改变一个进程的 address space 到新的位置，OS移动完成后，要记得更新base寄存器到新地址。

第四，OS提供 **exception handlers** 来处理各种异常情况。



下图展示了一个timeline更好地解释虚拟内存时 硬件和OS 的联动。看懂图就全懂了！

![1545822312393](/img/in-post/虚拟化virtualization.assets/1545822312393.png)

## 15.6 Summary and Homework

这一节基于 limited direct execution 的背景，我们加入了虚拟内存（**address translation**）。

但是还有问题！！！这样没有效率，虽然heap和stack没有用到那些空间，但是全部中间空间都被 wanted 了。这种浪费叫做：**internal fragmentation**。 所以我们要一个机制来避免这样。



# 16 Segmentation

接上一节，直接用 base and bounds ，会有很多的 free space 浪费掉，太浪费了！

## 16.1 Segmentation: Generalized Base/Bounds

为了解决，可以用 **segmentation** 的观点。一个segment是地址空间中特定长的一个部分，我们可以对于每个segment都弄出一个base and bounds pair。

![1545828191928](/img/in-post/虚拟化virtualization.assets/1545828191928.png)

比如这个例子中，就有3个 segment ，每个都能在物理地址中独立存储。且各自有自己的base and bounds pair。

![1545828486095](/img/in-post/虚拟化virtualization.assets/1545828486095.png)

## 16.2 which segment is being referred?

一种常见的 **显式** 方法是在虚拟地址的头几位声明。比如下图展示的就是 VAX/VMS 系统用的技术，00代表 code ，01代表 heap。后面的 offset 是偏移量，加上基址就能算出 真实地址 啦。而且 offset 有个好处，可以直接通过这个 offset 看出来超没超 bounds。

![1545829763988](/img/in-post/虚拟化virtualization.assets/1545829763988.png)

当然 **隐式** 的方法就是从寄存器来判断，比如，PC的地址那就是CODE段，如果 base pointer 就是stack，其它的在 heap 。

## 16.3 What about the Stack?

之前一直不讨论 stack ，因为stack不一样，它是负增长的。除了 base 和 bounds 我们还需要一个位来表示增长方向。更新后的段寄存器如下图：

![1545830190862](/img/in-post/虚拟化virtualization.assets/1545830190862.png)

## 16.4 Support for Sharing

![1545830327923](/img/in-post/虚拟化virtualization.assets/1545830327923.png)

为了支持 sharing，我们还要一个 protection bits ，用来指示这个程序的读写权限。把CODE设置为只读，就可以在不同的进程共享代码段，而每个进程都以为这就是自己的CODE段。上图的例子中的配置下，同一个物理上的CODE段就能被映射到多个虚拟地址空间里。

## 16.5 Fine-grained vs. Coarse-grained Segmentation

我们之前的 segments 都很少，说明这是 Coarse-grained （粗粒度）的。但是，一些早先的系统允许可以由大量的更小的段构成，被称作Fine-grained（细粒度）。

支持更多的段意味着更复杂的硬件支持功能，也就是一个存在 memory 中的 **segment table**。比如有些系统在编译的时候切割code和data成多个小段，由这个 table 管理。

## 16.6 OS support

有一些新的问题出来了，主要是OS的。第一个，OS在 context switch 的时候做什么？很显然，保存恢复 segment register 啦。第二个，也是重要的一个，创建 进程 时怎么从 free space 里分配出来空间呢？之前每个进程都是固定size，现在都是可变的啦。

![1545831485503](/img/in-post/虚拟化virtualization.assets/1545831485503.png)

这种情况呢，就叫做 **external fragmentation** （外部碎片）

一个简单的解决办法就是用一个 free-list management 算法追踪可用分配的内存区域。比如有 **best-fit** 算法（追踪一个free space 的 list，然后返回和申请的size最接近的那个）；稍微复杂的算法有 **buddy algorithm** 。不过，这些算法再智能，还是不能完全解决外部碎片，所以当没有可分配的时候只能整理全部内存来实现压缩了。

## 16.7 Summary and Homework

又进一步让内存更有效率了，但是问题也在：1.外部碎片 2.还是无法完全解决利用我们 sparse address space 的问题，比如一个heap，就算只有某一小部分，也必须全部放在内存中。



# 17 Free-Space Management

这一节，先绕道讨论下空闲空间的管理。如果 free space 是由一些变长的单元组成，管理很蛮烦。比如有30字节，中间的10字节被占用了，前后各10字节，如果申请15字节的就无处可放。这一节就是解决这个问题的。

## 17.1 Assumptions

1. 基本的软件接口是 malloc 和 free
2. 主要考虑外部碎片
3. 如果一片内存被申请过去了，就没法重定向到其它申请者
4. allocator 管理的是连续的邻接字节区域

## 17.2 Low-Level Mechanisms

**分离-合并**

假设有30字节的heap：

![1545982094820](/img/in-post/虚拟化virtualization.assets/1545982094820.png)

free list 大概长这样：

![1545982109939](/img/in-post/虚拟化virtualization.assets/1545982109939.png)

如果申请的区域比10字节大，很显然返回 NULL（失败），但是申请的比10字节小，allocator执行一个叫做 **splitting** 的动作，把一个符合要求的大块分成两个块，一个块刚好满足申请要求，另一个保留在 free list。

当一个free操作执行后，多出来的 free block 会看两边有没有刚好邻接其它的 free block，如果有，把它们两个（or 三个）合并，这个操作叫 **coalescing**



**追踪分配的区域的大小**

free 不接受一个size参数，因此需要一个额外的信息在 header block (in memory) 里。在第一个图的例子中，申请语句如 `ptr = malloc(20)` 所示。所以header里放的内容如下所示：

```
typedef struct __header_t {
	int size; 
	int magic;
} header_t;
```

所以当 free 操作时，先算出hptr的地址：

```
header_t *hptr = (void *)ptr - sizeof(header_t);
```

然后比较 magic 值是否一致，最后释放！

所以！事实上，总的大小是 head+requested size！！！（即使head很小也不该忽略）

![1545982919015](/img/in-post/虚拟化virtualization.assets/1545982919015.png)



**嵌入一个 free list**

之前这个list只是概念上的，现在要实际化了。

假设有4KB的heap可以分配，List中的node描述如下：

```
typedef struct __node_t {
	int size;
	struct __node_t *next;
} node_t;
```

假设堆是通过 mmap（Linux编程手册P1061,chapter 49） 得到的空闲空间里建立的，示例代码：

```
// mmap() returns a pointer to a chunk of free space 
node_t *head = mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_ANON|MAP_PRIVATE, -1, 0);
head->size = 4096 - sizeof(node_t); 
head->next = NULL;
```

以上代码执行后，List有了第一个entry，size是4088。head pointer 指向了这片范围的首地址，假设这就是16KB（虚拟地址）。图17.3大概就展示了这时的情况。

![1545985298391](/img/in-post/虚拟化virtualization.assets/1545985298391.png)

现在有一个 chunk 的mem被申请了，比如100字节，分配器首先找到一个特别大的 空闲chunk（只有一个），然后切割出够用的那部分。这些做好后，如上图17.4所示。

![无标题](/img/in-post/虚拟化virtualization.assets/无标题-1545996893910.png)

这个例子中，要释放中间的哪一个 chunk，因此释放完成后，head的那个entry变成了刚刚释放的chunk，并且指向了最后的那个 free chunk。



**增长 Heap**

如果heap用完了，该怎么办？大部分 unix 系统使用 sbrk 命令申请更多的空间，OS把一些 free page map到进程的地址空间中，然后扩大之前 heap 的尾部限制。

## 17.3 基本策略

最理想的分配策略肯定是 fast 并且 minimize fragmentation。但是不同的Inputs，结果差别巨大，所以各有各的特点把！

**Best Fit**

在足够的 chunk 中找寻最小的那个。返回 user 最需要的那个，找到正确的那个block需要一定性能开销

**First Fit**

第一个符合条件的 block 即可

**Next Fit**

加一个额外的指针在 free list，两个指针同时找 符合条件的 第一个block，意思就是更均匀

## 17.4 高级方法

上面那些太基本了，后面这些方法高级些：

**1 Segregated Lists** （分隔列表）

大致思想：如果经常有某几个size的申请，那么维护一个 separate list 仅仅管理这些size的内存，其它的 requests 转发到更 general 的内存分配器上。

好处就不说了，坏处就是让系统的复杂度更高。比如这个list应该管理多少的内存呢？一个特定的分配器 **slab allocator** 就是用来解决这个问题的。（文章：The Slab Allocator: An Object-Caching Kernel Memory Allocator）

深入一些的说，Kernel 启动后，会给 kernel object 分配一些 cache，这些都是被频繁申请的（eg: locks, file-system inodes等）。相反的，当一个 slab 中的 objects 的引用计数都归0后，通用分配器从专用分配器中回收它们，这往往发生在当VM系统需要更多的内存时。

**2 Buddy Allocation**

因为对于分配器来说，合并内存块非常关键，这个方法就是为了让合并更加简单。

![1546241846110](/img/in-post/虚拟化virtualization.assets/1546241846110.png)

思路很简单，所有内存都是 2^N^。当需要申请时（7KB），二分整个空余的内存块，然后直到切到刚好满足 size 要求为止，这个时候就把深色的 8KB 给出去。

当一个块返回到 free list 时候，先看它的 buddy 是不是也是free，是的话就合并然后再看合并后的那个buddy是不是free，以此类推。

这个算法在于很容易就找到它的 buddy ，因为什么？从地址上看，它和buddy只有一个bit不一样。好好思考！！！



# 18 Paging: Introduction

前面讲过了，OS一般用两种方法解决空间管理，一种是 **segmentation**；一种是 **paging**。前者有外部碎片，后者就是解决外部碎片的，分成固定长度的 page，一个 物理内存上的 page frame 对应虚拟内存上的一个 page。

## 18.1 An Example and Overview

![1546245097465](/img/in-post/虚拟化virtualization.assets/1546245097465.png)

![1546245108506](/img/in-post/虚拟化virtualization.assets/1546245108506.png)

在这个例子中，64字节的地址空间分成了4个Page放到了物理内存对应的4个frame中。好处不言而喻，1. **flexibility** 2. **simplicity**

为了记录哪些page放在哪个frame，需要一个page table，**page table** 的主要作用就是翻译地址，上面的例子中：four entries: (Virtual Page 0 → Physical Frame 3), (VP 1 → PF 7), (VP 2 → PF 5), and (VP 3 → PF 2).

页表是一个 per-process data ，因为有很多进程，每个进程的 page 0 映射的都不一样，所以一个table不够。

虚拟地址有两个部分：**VPN**（虚拟页号）和 **offset**（偏移）。

这个例子中，64字节的地址空间，2^6^=64，所以需要6位虚拟地址，又因为4个页表，所以前2位就是表示VPN的，如下图：

![1546245717225](/img/in-post/虚拟化virtualization.assets/1546245717225.png)

当翻译时，通过查询前2位确定是7号物理页表（**physical page number** or **PPN**），如下图：

![1546246976946](/img/in-post/虚拟化virtualization.assets/1546246976946.png)

## 18.2 Where are page table stored?

因为 page table 过于巨大，我们不能把当前运行的程序的对应页表放在MMU里。所以**暂时**只能放在内存中！！！

![1546247683665](/img/in-post/虚拟化virtualization.assets/1546247683665.png)

## 18.3 What's in the Page Table?

简单的实现就是用一个线性的页表，说白了就是数组，用 VPN 来索引这个数组，查询 page-table entry（PTE）来找到 physical frame number（PEN）。

有些额外的内容说明：一个 **valid bit** 通常指示这个地址翻译有效性，如果无效就trap进OS。把地址空间的没有用到的page标记为无效，就不用在物理中分配出这部分内存，从而节约了许多内存。

还有一个叫做 **protection bits** 用来说明这个page是否可以 read write execute，如果异常也会trap进OS。

后续章节会讲到的，**present bit** 用来指示这个页是不是在内存或外存中（swapped out）。

**dirty bit** 用来表示这个page从加载到内存之后有没有被修改过。

**reference bit** 追踪这个page是不是被访问过，这个位很有用，表示这个页是不是受欢迎，所以该不该保留在内存中。

下面就是一个PTE的例子：

![1546248771388](/img/in-post/虚拟化virtualization.assets/1546248771388.png)

## 18.4 Paging: Too Slow

现在继续考虑这个页表，它或许还是太慢了，具体情况如下：当硬件要找页表时，通过一个 **page-table-register** 找到当前运行进程对应的页表存放的物理地址，为了找到特定的PTE，硬件做如下操作：

```
VPN = (VirtualAddress & VPN_MASK) >> SHIFT 
PTEAddr = PageTableBaseRegister + (VPN * sizeof(PTE))
```

在上面的例子中，这个 VPN_MASK 设置为 110000，意味着取出来虚拟地址的前2位，即VPN；然后移位操作找到 PTEAddr。找到PTE后，解释出PFN，算出偏移量，找到真正的那个地址内容。

下面就是以上方法做的额外的操作，你会看出内容太多了！！！每次内存引用都要这么大的开销，page table 的介入让系统变慢。

![1546324659497](/img/in-post/虚拟化virtualization.assets/1546324659497.png)



# 19 Translation Lookaside Buffers (TLB)

使用 paging 作为虚拟内存的核心机制导致了高额的性能开销。把地址空间划分成小的定长的单元（pages），带来了大量的mapping信息。而mapping信息通常在物理内存中存储，额外的内存查询导致这个开销。每个指令的取指和隐含的load or store的内存引用都会去memory中查询pages以执行地址翻译，这太慢了！！！

OS的老朋友：hardware可以帮助我们。为了加速地址翻译，**translation-lookaside-buffer (TLB)** 诞生了，TLB是MMU的一部分。在每次虚拟内存的引用中，硬件先检查TLB看看期望的translation是否在那里，如果在就直接翻译（快）而不用查page table了。

## 19.1 TLB基本的算法

![1546347553901](/img/in-post/虚拟化virtualization.assets/1546347553901.png)

上图展示了一个粗略的过程。首先检查TLB有没有对应的VPN，如果有就说**TLB hit**了。从TLB的entry中提取PFN并访问；如果 **miss** 了，访问 page table 去看真正的PFN是什么。TLB就像一个cache，我们希望miss尽可能的少。

## 19.2 Example: 访问一个数组

![1546410966914](/img/in-post/虚拟化virtualization.assets/1546410966914.png)

假设对于虚拟地址空间，有一个数组包含10个元素，它们分散在上图中。对于有TLB的内存引用，情况是miss，hit，hit，miss，hit，hit，hit，miss，hit，hit。所以，TLB **hit rate** 是70%。由于 **空间局部性** ，TLB提升了系统的性能。

page size 在这个例子中扮演了重要的角色。如果 size 更大，那么miss就会更少。

**时间局部性：** 如果TLB很大，那么短时间访问地数据就会更容易hit

## 19.3 谁处理TLB Miss?

硬件还是OS？

早期，一般是硬件实现的，需要通过 **page-table base register** 获取page table在内存的位置，然后miss后遍历page table找到对应的PFN更新TLB。

后来现代系统用软件实现的技术叫 **software-managed TLB**。当TLB miss后，硬件仅仅发出一个异常，然后进入内核态，执行一个 **trap handler**。

第一个问题来了？返回的时候并不是PC+1，而是重复执行一次指令，即PC不变，这一次，TLB hit。所以根据异常的不同，PC保存的内容也有所区别。

第二个问题是为了避免无尽的TLB miss，需要把TLB miss handler放到物理地址上（反正就是不要再用虚拟地址做翻译了）。

用OS来处理的话，**灵活性** 和 **简单性** 非常棒。

## 19.4 TLB里面有什么？

一个普通的TLB可能有32、64、128个entries，并且是 **全相联** 的。这说明一个 translation 可能在TLB的任何地方，TLB的一个entry可能看起来如下：

**VPN | PFN | other bits**

用硬件上的术语，TLB是 **fully-associative** cache。硬件并行地检索每个entry看有没有匹配。

other bits 里存的有 **valid bit** 和 **protection bit** 等，还有其它的各种额外bit信息。

## 19.5 TLB的问题

#### context switch

不同的进程的地址空间对应不同的页表，而每次更换进程就清空TLB开销过大，所以加入额外的bit指示这个 translation 是谁的，这样就算有同样的 VPN 也不会搞混了。ASID（**address space identifier**）就是干这个活的。

![1546413897279](/img/in-post/虚拟化virtualization.assets/1546413897279.png)

#### Replacement Policy

当替换一个旧的entry时，一般替换 **least-recently-used** (LRU) entry。

## 19.7 一个真正的TLB entry

![1546414079888](/img/in-post/虚拟化virtualization.assets/1546414079888.png)

（本节内容略）

## 19.8 总结

硬件可以帮助我们做地址翻译，一个小型的、专用的片上TLB可以作为地址翻译的cache。

但是TLB对于某些程序并没有让它们觉得世界美好起来，还是有很多问题的……

# 20 Advanced Paging: Smaller Tables

现在开始解决paging技术带来的第二个问题：page tables 太大了因此消耗了过多的内存。你会发现用 linear page table 太大了，对于32位的地址空间，4KB一个page，4B一个entry的话，需要 $\frac{2^{32}}{2^{12}}$ 个pages，这样就要4MB的table。一个进程都要4MB page table 太大了。

一个暴力的方法就是增大单个Page size，这样减少不了多少page table 的 size，而且**内部碎片**的问题更加严重。

## 20.2 Hybrid Approach: paging and segments

一种方法结合了 paging 和 segmentation 来减少 page table 的内存开销。

如果直接用 single page table ,大量的entry都是 invalid，所以为了节约空间可以尝试为每个 logical segment 分配一个 page table。比如说：一个页表给 code，一个给 heap，一个给 stack。

之前章节我们说到了 base 和 bounds 寄存器，现在我们的MMU仍然有这些寄存器。但是！在这里 base 寄存器不直接指向 segment，而是指向那个segment对应的 **physical address of the page table**。bounds 寄存器用来表示page table的尾巴（也就是多少个有效的pages）。

现在的虚拟地址变成了这样：

![1546429592871](/img/in-post/虚拟化virtualization.assets/1546429592871.png)

在例子中，有3个segment，因此3对 base/bounds pairs，每次 miss 找对应seg的page table，大概流程如下把：

![1546430600498](/img/in-post/虚拟化virtualization.assets/1546430600498.png)

bounds寄存器在这种段页式管理中很重要，比如一个段只用了前3个page，那么对应的 page table 也只有前3个page的entry，bounds寄存器就被设置为3。

最后你还是注意到了，这个方法不是完美的。第一，又要使用 segmentation，说过了！扩展性不好！第二，带来外部碎片！而且每个段 page table 的大小不固定了，系统的复杂度很高。为此，设计者提出了更好的办法。

## 20.3 Multi-level Page tables

前面的技术给了一个好的思路，可以避免page table中的无效部分。一个像树型结构的 **multi-level page table** 技术诞生了，这个技术非常有效率，很多现代系统都在用。

思路：把page table分割成 page-sized 的单元，如果一个PTE对应的page table 是 invalid，那么就不分配这个page。索引用的是 **page directory**。

![1546432132359](/img/in-post/虚拟化virtualization.assets/1546432132359.png)

图20.3展示了这个例子。右边是二级页表，每个entry对应一个 **page frame number** 和 一个 **valid bit**。如果有效才会分配对应的page。这很好地让稀疏的地址空间的页表不会占用过大的内存。而且灵活性很好，如果页表需要扩容，分配一个page就好，之前的那个连续的页表需要连续完整的4MB内存才行！现在有了一级间接层，允许把page-table 的部分pages放到任何地方了。

但是注意！TLB miss后现在需要两次load了，一次是 page directory，一次是PTE本身。这是个 **time-space trade-off**。所以在这些更小的table的情况中，TLB miss造成了更大的开销。而且为了节约内存，我们让系统支持多级页表更加复杂化。

#### A Detailed Multi-level Example

![1546437128178](/img/in-post/虚拟化virtualization.assets/1546437128178.png)

在这个例子中，page 0 和 1是给code的，page 4 5给heap，以此类推。14位的地址空间，8位给VPN用，因此有256个entry。假设一个PTE是4字节，一个page table是 256*4B = 1KB。所以1KB页表可以划分成16个64字节的页表。

有了VPN，用其中的 page directory index 索引用哪个page table：

![1546437395218](/img/in-post/虚拟化virtualization.assets/1546437395218.png)

通过以下计算：

**PDEAddr = PageDirBase + (PDIndex * sizeof(PDE))**

这样就找到了 page-directory entry，如果这个entry是无效，就发出一个异常；否则就从page取出PTE。

**PTEAddr = (PDE.PFN << SHIFT) + (PTIndex * sizeof(PTE))**

![1546437535133](/img/in-post/虚拟化virtualization.assets/1546437535133.png)

（注意，PFN一定要就地移位）

获得PTE之后，用下面计算出最终的物理地址

**PhysAddr = (PTE.PFN << SHIFT) + offset**



#### More Than Two Levels

![1546438233283](/img/in-post/虚拟化virtualization.assets/1546438233283.png)

当地址空间变大后，只有2级页表还是不够小，所以实际中往往使用更多的级别。



#### Remember the TLB

![1546438322464](/img/in-post/虚拟化virtualization.assets/1546438322464.png)

结合TLB硬件总结整个 two-level page table 的流程。硬件首先检查TLB，如果hit，物理地址直接得到，不用看page table；否则执行完整的 multi-level lookup。

# 21 Swapping: Mechanisms

之前我们都假设 address space 不切实际的非常小，以匹配进物理内存的真实大小。事实上，每个正在运行的进程都有很大的地址空间。

为了实现真实的虚拟内存系统，还需要在 **memory hierarchy** 上加上额外的一层。

我们现在要支持很大的地址空间，这可以让进程不用考虑是否在内存不够的时候换出到磁盘中，OS做这件事，进程以为自己的东西都在 memory 中。

## 21.1 Swap Space

第一件事，在外存中保留一个space用于把page来回地交换，这叫做 **swap space**。现在我们假设OS可以以page为单位从 swap space 来回读取，OS需要记住特定page在外存中的位置。

![1546484576557](/img/in-post/虚拟化virtualization.assets/1546484576557.png)

## 21.2 The present bit

先来回忆下：进程生成了一次虚拟内存的引用，硬件从VPN中提取虚拟地址，检查TLB，如果hit直接返回物理地址从内存访问；但是miss了，就用 **page table base register** 定位到内存中，查询PTE，把PTE的PFN装载进TLB，然后hit。

但是想要swap的话，就要另外加一个机制。这就是 **present bit**，它在页表中，如果为1说明这个page在物理内存，如果为0说明不在内存，访问一个为0的page，称为一次 **page fault**。

遇到 page fault，OS被唤醒处理这次page fault，这部分处理代码叫 **page-fault handler**。

## 21.3 The Page Fault

声明：不管硬件还是软件管理TLB，page fault都是OS处理的。

如果page不在内存中，OS会在page table中查询disk address并且发起一个disk请求。当 I/O 完成后，OS会更新page table，下一次引用虽然没 page fault，但是还是会TLB miss。

注意：在 I/O 的时候，进程是 **blocked** 状态，因此OS可以选择其它的进程来运行。

## 21.4 What if Memory is Full?

如果内存满了，又要给新的page腾出空间，那么选择一个page踢出内存的进程叫：**page-replacement policy**

## 21.5 Page Fault Control Flow

![1546493015494](/img/in-post/虚拟化virtualization.assets/1546493015494.png)

![1546493635754](/img/in-post/虚拟化virtualization.assets/1546493635754.png)

现在根据上面两个图，还有我们学到的这些知识（Naive！！！）已经可以粗略地描绘出 memory access 的控制流程。

## 21.6 When replacements really occur

以上我们讨论的都是当memory全满了才替换出一页，这样其实不切实际的。OS一般情况是保持一小片的 free memory，而不是最后才替换。

为了实现这种工程上的技术，OS有两个标杆：**high watermark** and **low watermark**。当OS发现少于LW的pages可用，就在后台开一个线程 **swap daemon** 负责替换出页面到外存，一直到可用内存达到HW线上。

所以上图的21.3也要相应修改下：不直接执行 replacement algorithm，在此前先检查可用Pages数量，如果不够就启动 swap daemon。

# 22 Swapping: Policies

How to decide which page to evict ?

## 22.1 Cache Management

阐述下我们解决的目标：main memory可以**看作cache**，那么那么设计替换策略让cache miss最小。最大化 cache hit。

**average memory access time (AMAT) = (PHit · TM) + (PMiss · TD)**

Tm代表访问内存的开销，Td代表访问磁盘的开销，Phit代表命中概率，Pmiss是miss的概率。 Pmiss + Phit = 1.0

因为TD很大，所以哪怕Pmiss很小，最终AMAT还是很大。所以设计一个聪明的 policy 至关重要。

## 22.2 The Optimal Replacement Policy

在分析某个置换算法前，先学下最优的置换算法。这种算法可以实现最少的Misses，但却很难实现。

思想：如果扔出一个page，就扔在最远的未来访问的page。

问题：你无法知道谁是未来最久才会访问的，因为你没法预知未来！

## 22.5 LRU

> 有两个简单的笨方法 FIFO 和 Random，跳过了

我们已经知道时间和空间的局部性，如果一个page过去访问过，在未来也极大概率会访问到，因此基于过去的历史也是思路之一。一个基于历史的算法叫：**Least-Frequently-Used (LFU)** 策略替换最不频繁的page；相似的，还有一个叫：**Least-Recently-Used (LRU)** 替换最近没用到的page。 

![1546502948288](/img/in-post/虚拟化virtualization.assets/1546502948288.png)



