---
layout:     post
title:      "操作系统的三个主题——并发性"
subtitle:   "The part 1 of Operating Systems: Three Easy Pieces"
date:       2019-02-23
author:     "TXXT"
catalog:    true
tags:
    - Operating System
---

# 26 Concurrency: Introduction

从现在开始引入 **thread** 概念，和 process 最大的不同在于thread可以共享一个 address space，因此访问到同样的数据。所以类似地，它们也有独立的PC，独立的寄存器，比如 **context switch** 的时候就要保存恢复寄存器状态。而这些状态在 **thread control blocks** （TCB）里放着。但是正在使用的page table不需要换。

另外一个不同在于对 stack 的设计，下图很直观地说明了每个thread的stack都是独立的：

> 虽然右边破坏了我们美丽的address space设计，但是往往还OK，因为stack一般不大。

![1547001854631](/img/in-post/并发性Concurrency.assets/1547001854631.png)



#### 26.1 Example: Thread Creation

编译时尽量使用 `-pthread`

![1547002986376](/img/in-post/并发性Concurrency.assets/1547002986376.png)

例子中创建两个线程，最后使用 `pthread_join()` 等待某个线程完成工作。不过顺序可能是乱的。因为系统创建了线程后，线程的工作就独立出去了，顺序取决于调度器。

![1547003199528](/img/in-post/并发性Concurrency.assets/1547003199528.png)

#### 26.2 Shared Data & Uncontrolled Scheduling

如果在一个例子中，创建两个线程，让它们循环多次对一个 global shared variable 做+1操作，会发现结果很可能多次不同。

Why?

这是X86的汇编：

![1547004978771](/img/in-post/并发性Concurrency.assets/1547004978771.png)

但是因为并发，有可能当一个线程已经取出对应地址的数据到eax后，另外一个线程也取了一份，这样两边都+1放回去后等于少加了一次。

![1547005435773](/img/in-post/并发性Concurrency.assets/1547005435773.png)

刚刚看到的就是 **race condition**：结果取决于代码的不同推进顺序。这样的话，结果就是 **indeterminate**。这种能引起 race condition 的代码叫：**critical section**。对于这种 critical section，我们想要 **mutual exclusion**。保证一次只能一个线程访问到这个section。



因此，我们希望某些代码能够原子性的执行，这样就不会被打断而出错。硬件会帮助这件事完善，硬件可以看中断的时候是不是执行某些原子操作。



除了上面说到的 shared variable ，还有一种情况就是一个进程等另外一个动作执行完。后面 **condition variables** 会提到的。



# 27 Thread API

介绍常用的线程 API，作为参考章节。编译带有pthread.h的程序时，编译链接 `pthread` 库。

#### 27.1 Thread Creation

在POSIX中，创建线程的函数的方法如下：

![1547014394254](/img/in-post/并发性Concurrency.assets/1547014394254.png)

- 第一个参数 thread 是一个结构的指针，用这个结构才能和线程互动；
- 第二个参数 attr 指明这个线程的属性，可以设置 stack size 或者 scheduling priority，默认就是NULL；
- 第三个参数声明线程执行的函数内容，在C语言，用一个 **function pointer** 即可。当然返回类型和参数类型都是视情况而定，不一定 void *；
- 第四个参数就是传递给函数执行的参数。为什么用 void * ？因为这样我们就可以传递任何类型了，void * 作为返回类型也同理。

![](/img/in-post/并发性Concurrency.assets/1547015452712.png)

#### 27.2 Thread Completion

使用 pthread_join() 等待一个线程完成。方法如下：

![1547015632005](/img/in-post/并发性Concurrency.assets/1547015632005.png)

这个方法只有2个参数。

- 第一个是指明 pthread_t 类型的线程；
- 第二个是一个指向线程函数返回值的指针（注意，不要返回线程函数里声明的内容的指针）；

![1547015592742](/img/in-post/并发性Concurrency.assets/1547015592742.png)

#### 27.3 Locks

提供互斥机制可以通过 Locks，如果没有其它的线程持有这个lock，那么就会获得lock并进入临界区。方法如下：

![1547017086723](/img/in-post/并发性Concurrency.assets/1547017086723.png)

使用中注意一些东西：

1. 锁要初始化，比如 `pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;`
2. 锁要检查错误，防止代码错误，比如用以下封装加锁过程：

![1547017867526](/img/in-post/并发性Concurrency.assets/1547017867526.png)

3. 还有其它的和锁相关的使用方法。以下的例子，前者 trylock 会在锁已经加上后返回 failure；后者会在成功获取锁或者超时后返回。

![1547018056865](/img/in-post/并发性Concurrency.assets/1547018056865.png)

#### 27.4 Condition Variable

用于在线程之间完成一些配合工作，用于在两个不同线程之间传递信号。用法如下：

![1547020326928](/img/in-post/并发性Concurrency.assets/1547020326928.png)

使用条件变量，必须有一个和这个条件关联的Lock，在调用上面两个方法时，必须持有锁。

这是一个例子，看ready是不是非0，wait用来把调用的那个线程睡眠；

![1547020691231](/img/in-post/并发性Concurrency.assets/1547020691231.png)

第一段代码表示看 ready 是否不为0，然后wait并sleep了。

![1547020700654](/img/in-post/并发性Concurrency.assets/1547020700654.png)

第二段代码在另一个线程，修改了 ready 并发起 signal。

以上的例子中注意：

1. 发起signal时要在临界区
2. 为什么wait里有一个锁变量？因为caller休眠时释放了锁，但是在wait的线程唤醒后，立即获取了这个锁，这样在运行的状态下保证一致在临界区
3. caller线程使用一个while来再次检查是为了 简单 和 安全。（因为怕它醒了后以为ready已经修改其实还没有）

**PS: wake up 应该被视为暗示某些事情可能发生了变化（而非绝对）**

警告：不要用这种方法来实现 Signal：

![1547021251984](/img/in-post/并发性Concurrency.assets/1547021251984.png)

原因：1. 空转循环占用 CPU； 2.可能会出错

#### 27.5 总结

使用poxis threads必须包含头文件pthread.h，而且要显示地增加 `-pthread` 标志。对于C++用法一致。

**Thread API 准则**

1.线程之间临界的代码尽可能简单

2.尽可能减少线程的相互作用

3.记得初始化 locks 和 condition variables

4.检查return内容

5.小心传递和返回的内容，比如在栈上分配的内容，返回后就会出错。

6.每个线程有独立栈！想在线程之间共享只能用 heap 中的数据。

7.记得用条件变量来作为信号量，不要用一个简单的while判断。



# 28 Locks

在引入中讲过了，锁被放在临界区，以实现原子操作。

#### 28.1 Locks:基本概念

![1550213592415](/img/in-post/并发性Concurrency.assets/1550213592415.png)

假定例子中要修改余额balance，那么需要用到锁，锁其实就是一个变量，要么是free，要么是held。当然这个数据类型也许可以存储一些信息：谁hold了锁，一个等待的队列，但这些信息不暴漏给用户。

某种意义上，lock在OS的调度之上提供了一些控制权给程序员，让程序员能干预到OS的正常调度以实现对临界区的某种保护和控制。

#### 28.2 Pthread Locks

POSIX对应锁的东西叫做 **mutex**，因为如字面意思（互斥），下面就是用POSIX的方式：

![1550213748642](/img/in-post/并发性Concurrency.assets/1550213748642.png)

#### 28.3 构建Lock

构建一个Lock一般需要硬件的协助，我们这里无需知道硬件怎么做的，只是用它们而已。

#### 28.4 评价Locks

评价一个锁：

1. 是否完成基本互斥的任务
2. 是否公平，保证没有饥饿
3. 性能如何

#### 28.5 控制中断

早期为单处理器设计的解决方案就是：禁止临界区的中断。

这样做虽然粗暴简单，但是问题很多：**第一**，这是特权操作，因此很有可能OS被普通的程序崩溃。有些程序可能会垄断处理器资源，使用禁止中断需要你对这个程序绝对的信任；**第二**，多处理器无法这样做，因为还有其他的处理器跑着其他的线程，所以中断没有意义了；**第三**，关闭中断导致其他的系统错误，比如其他用途的中断无法知晓；**第四**，这样做效率也不高。

那么我们仅仅用一个简单变量做锁如何，比如谁上锁，就把锁变量设为1，这样做无法保证正确，因为上锁本身又是一个临界区，有可能出现同时上锁。而且也会造成 **spin-waiting** 的情况发生，浪费性能。

#### 28.7 使用Test-and-Set构建Spin Locks

![1550216666830](/img/in-post/并发性Concurrency.assets/1550216666830.png)

最简单的硬件支持就是 test and set，大概流程如下，但是返回 *old_ptr的同时更新了新的值上去，一系列的操作是原子性的。

![1550216067657](/img/in-post/并发性Concurrency.assets/1550216067657.png)

以上就是利用硬件的 test-and-set 来实现的Lock。这种也叫做 **spin lock**，这是最简单的一种锁。



**评价 spin locks**

正确性？毋庸置疑！

但是公平性无法保证，简单的 spin lock 可能导致饥饿问题。性能嘛，也一般般。

#### 28.9 Compare-And-Swap

![1550216679037](/img/in-post/并发性Concurrency.assets/1550216679037.png)

基本概念如上，锁的实现如下：

![1550217083840](/img/in-post/并发性Concurrency.assets/1550217083840.png)

这种硬件指令更加强大，功能更丰富。

#### 28.10 Load-Linked and Store-Conditional

某些平台构建了一队协同工作的指令来构造临界区，如MIPS架构，这两个指令可以串联地构造并发结构。

![1550217267258](/img/in-post/并发性Concurrency.assets/1550217267258.png)

（略）

#### 28.11 Fetch-And-Add

![1550217382758](/img/in-post/并发性Concurrency.assets/1550217382758.png)

以上是该指令C伪代码形式，下面是锁的构建，其余略

![1550217414989](/img/in-post/并发性Concurrency.assets/1550217414989.png)

#### 28.12 解决spin过多？

如果一个线程还没做完临界区的事情被时间中断了，其他的线程仅仅能spin，这样就浪费了多个时间片，需要OS的帮助才能避免这种无意义的spin。

#### 28.13 简单方法：Yield（让步）

当发现spin的时候，主动让出CPU的时间片。

![1550218550462](/img/in-post/并发性Concurrency.assets/1550218550462.png)

但是这个方法不够好，假如有100个线程，99个线程都要执行 lock() -> yield() 操作。而且饥饿问题还是没能解决，所以我们需要直接解决这个问题。

#### 28.14 使用队列：Sleeping而非Spinning

早先的方法都很麻烦，改一处而动全身。直接显示地控制每次获取锁的对象岂不美哉。这就要OS支持，用一个队列追踪哪个线程是正在等待锁的。Solaris提供了如下的接口。

![1550219489196](/img/in-post/并发性Concurrency.assets/1550219489196.png)

用 park() 来让调用的线程休眠，用unpark(id)让对应id的线程唤醒。上图的代码就是这种用法。

#### 28.15 不同OS，不同支持

Linux和Solaris不一样，提供的是 **futex**，效果类似。`futex wait(address, expected)` 用来将调用线程休眠，`futex wake(address)` 用来唤醒。

![1550220873076](/img/in-post/并发性Concurrency.assets/1550220873076.png)

#### 28.16 Two-Phase Locks

第一阶段，先spin一会，看看可以用锁不，然后第二阶段进入休眠。Linux的版本就是两阶段，第一阶段spin一次，然后休眠。

# 29 Lock-based Concurrent Data Structures

对lock的最后讨论就是在常见的数据结构中如何使用锁。

#### 29.1 并发计数器

这样一个简单的计数器怎么保证 **线程安全** 呢？下面是一个带锁的例子：

![1550368800935](/img/in-post/并发性Concurrency.assets/1550368800935.png)

这样的处理很简单也有效，大部分简单的并发结构都可以这样加一个锁进行改造。但是性能就不一定很好了，随着可用线程增多，这种数据结构的性能直线下滑，我们的理想情况是多线程的情况下，完成任务的速度和单线程差不多，因为任务是并行完成的，我们的目标是：**perfect scaling**

**Scalable Counting**

没有 scalable counting 的情况，Linux会在多核心机器上面临严重的 scalability 问题。

解决的一个方式是 **approximate counter**，通过给每个核心分配一个local counter，再加上一个global counter，以让每个CPU更新时不产生竞争，这样更新counter就变得scalable。周期性地，local counter通过获取global lock后把值加到global上，然后把自己的值清空。

从local到global的转变由一个阈值 S 决定。S越小，计数器越像一个 non-scalable 计数器。S越大，global越不精准。

下面是approximate counter的一个实现版本：

![1550542796968](/img/in-post/并发性Concurrency.assets/1550542796968.png)

#### 29.2 Concurrent Linked Lists

下面是一个并发式链表的实现：

![1550545977356](/img/in-post/并发性Concurrency.assets/1550545977356.png)

这个时候我们又遇到了 scaling 问题。

**scaling linked lists**

一种思路是给每个node都分配一个锁，但是这样每次获取和释放锁都需要很高的开销，所以不见得性能就会好于普通的并发链表。

所以对于这种数据结构就要清楚控制流，不是说做成了scaling就会性能好。

#### 29.3 Concurrent Queues

至今为止，让数据结构并发的方式就是加一个大锁。但是队列不是简单的这么做，下面是并发队列的实现。

不过在实际的应用中，队列都是用在多线程应用中，这样只有锁的队列可能会让程序处于等待，所以后面章节的条件变量会解决这个问题。

![1550547810229](/img/in-post/并发性Concurrency.assets/1550547810229.png)

#### 29.4 Concurrent Hash Table

下面是一个不能resize的简单实现：

![1550548102100](/img/in-post/并发性Concurrency.assets/1550548102100.png)



# 30 Condition Variable

#### 30.1 定义和操作

为了实现等待一个条件为真，线程可以利用叫做 **condition variable** 的东西。当某个线程完成后，可以通知这个条件变量的相关线程唤醒去处理。条件变量的用法就类似： `pthread_cond_t c;`，这声明了c作为一个条件变量。它有两个操作接口，wait和signal。

```c
pthread_cond_wait(pthread_cond_t *c, pthread_mutex_t *m);
pthread_cond_signal(pthread_cond_t *c);
```

wait的操作必须有一个锁，释放后休眠；这样睡醒后它还要拿到这个锁。这样设计的目的是避免当一个线程准备休眠时竞争条件的产生。

![1550630445383](/img/in-post/并发性Concurrency.assets/1550630445383.png)

在上面的例子中，thr_join()里判断一下 done == 0 很有必要，因为可能创建的线程已经完成了signal，这样直接wait永远没人来wake up它。

另外一点就是，条件变量要和锁一起用，因为避免条件竞争，有可能主线程马上休眠前发生中断切换，然后子线程完成了工作，这样主线程回来开始休眠，并永远长眠。

后面讨论 **producer/consumer** 问题。

#### 30.2 生产者/消费者 问题

正是这个问题让 Dijkstra 发明了 semaphore (既能作为锁也能作为条件变量)。生产者线程产生一些数据在buffer，消费者线程使用buffer的数据。

举例，在多线程web服务器中，生产者把HTTP请求放到一个工作队列，然后消费者从这里取出并处理。

下面是一个例子代码，

![1550631315392](/img/in-post/并发性Concurrency.assets/1550631315392.png)

![1550631510755](/img/in-post/并发性Concurrency.assets/1550631510755.png)

图30.5是我们的第一个版本，生产者不断往buffer里放数据，消费者去拿，但是这没有考虑多线程的优化。

下面是一次尝试，（**虽然结果有问题**）

![1550632305832](/img/in-post/并发性Concurrency.assets/1550632305832.png)

记得用 while 来在每次wait的时候，以确认唤醒后真的有所需的条件了。

但是有个bug，当两个消费者都先运行后进入sleep了，生产者完成后，一个消费者唤醒了，Tc1完成后通知了Tc2，因为Tc2在队列的头部。但是Tc2发现消息被人拿走了，所以再次回到sleep。然后这三个线程就这么长眠了。

![1550632442727](/img/in-post/并发性Concurrency.assets/1550632442727.png)

当然想要解决也不是很困难，增加 条件变量 的种类，以区分empty和fill。最终完善后的版本如下：

![1550632969552](/img/in-post/并发性Concurrency.assets/1550632969552.png)

30.11中，增加了两个指针，这样放置和拿取可以多线程一起了。30.12中，正确的wait和signal逻辑如图，生产者只有在所有buffer都满了才sleep，消费者在所有buffer都空才sleep。这样就算是解决了 **生产者/消费者** 问题了。



# 31 Semaphores

#### 31.1 定义

信号量是一个带有整型值的对象，用两个操作接口操作，在 posix 标准中，这些方法是 `sem_wait()` 和 `sem_post()` 。在使用前要初始化成某个值，正如下图：

![1550633694216](/img/in-post/并发性Concurrency.assets/1550633694216.png)

第二个参数设为0代表在同进程下的所有线程都共享，第三个参数就是初始的值。

sem_wait 也许会直接返回，因为semaphore大于等于1，或者让调用者休眠；sem_post 增加semaphore的值，然后如果有一个线程正在等待唤醒，就唤醒它；其次，semephore的值为负数时，等同于等待的threads数量，虽然这对用户来说不用关心。

![1550645749977](/img/in-post/并发性Concurrency.assets/1550645749977.png)



#### 31.2 二进制semaphore（锁）

可以把信号量用于锁，比如把X设置为1。这样保证只有一个线程进入

![1550645758522](/img/in-post/并发性Concurrency.assets/1550645758522.png)

图31.5展示了这次追踪，除了线程的行为，还展示了调度器的行为：run、ready、sleep。这样一个用于锁的semaphore叫 **binary semaphore**。

![1550646996058](/img/in-post/并发性Concurrency.assets/1550646996058.png)



#### 31.3 用于Ordering的Semaphore

semaphore可以用于排序事件，和之前我们用的 **condition variable** 相似。

下面展示的是父线程等待子线程完成后才继续的例子：

![1550647308668](/img/in-post/并发性Concurrency.assets/1550647308668.png)

![1550647323663](/img/in-post/并发性Concurrency.assets/1550647323663.png)

#### 31.4 生产者/消费者 问题

下面就是一个解决方案，为了避免死锁，加mutex一定要在临界区前后加，不要加到信号量前面去了，不然就会死锁。

![1550650227230](/img/in-post/并发性Concurrency.assets/1550650227230.png)

#### 31.5 Reader-Writer Locks

另一个问题是在并发的链表，可以有多个读取操作。如果有线程想写入数据

![1550652520313](/img/in-post/并发性Concurrency.assets/1550652520313.png)

#### 31.6 哲学家就餐问题

这个问题很有趣，但是实用性不咋高。问题假设有5个哲学家坐在一起，它们中间放了一把叉子，哲学家有些时间不吃饭，有些时间要吃饭，吃的时候需要左右的叉子。每个人做的事情如下：

![1550665367842](/img/in-post/并发性Concurrency.assets/1550665367842.png)

其中需要两个辅助函数表示用左右叉子：

![1550665677189](/img/in-post/并发性Concurrency.assets/1550665677189.png)

如果单纯的在获取叉子前sem_wait，然后在放下叉子后sem_post，就可能会发生死锁问题。因为所有人刚好都拿起了左手边的叉子，他们都卡在了准备拿右边的叉子步骤中。

**一个解决方法：破坏依赖**

只要打破这个循环就行，所以我们让最后一个哲学家（4号）拿叉子的顺序反着就打破了整个循环依赖了。

![1550666726878](/img/in-post/并发性Concurrency.assets/1550666726878.png)

#### 31.8 总结

信号量非常强大，以致于能替代了锁和条件变量。

# 32 常见并发Bug

> 略

# 33 基于事件的并发（高级）

迄今为止我们的并发都是使用线程实现的，但是很多时候比如在服务器，使用的是基于事件的并发，包括服务端的框架比如 **node.js**，都越来越流行。

这种基于事件的并发有两个大问题：1.在多线程应用内部正确管理并发；2.程序员无法控制多线程应用的调度。

#### 33.1 基本概念：事件循环

我们要用的基本方法是叫做 **event-based concurrency**。方法很简单，你仅仅在等一个事件发生，当发生后检查事件类型并完成其请求（I/O请求、调度事件等）。我们先看一下事件循环的伪代码：

```c
while (1) {
	events = getEvents();
	for (e in events)
		processEvent(e);
}
```

当一个handler处理一个事件，它就是系统中唯一的活动者，决定后续怎么调度操作，这种显示的控制就是基于事件并发的最大好处。

#### 33.2 重要API：select() poll()

通过select() 或者 poll() API，可以检查是否有正在达到的需要处理的 I/O。举例，一个网络应用希望检查是否任何的网络包到达并处理。 

![1550669546003](/img/in-post/并发性Concurrency.assets/1550669546003.png)

select()检查文件描述符，并看它们传递的地址以检查是否可读可写是否有错。也就是说，从0到nfds-1的描述符都要检查。返回时，select()用包含那些为所请求操作做好准备的描述符的子集替换给定的描述符集。 select（）返回所有集合中的ready描述符总数。

这允许你检查是否可读或者可写，前者让服务器决定当一个packet到达时怎么处理；后者让服务知道是不是可以去回应。

而且时间参数可以设置为NULL，这样除非描述符准备好了，不然一直阻塞。也可以把其设为0，代表立刻返回。

这些基本的原语给我们构建无阻塞的事件循环一种方法，就是简单地看看正在达到的packet，从sockets中读取，按需回应。

#### 33.3 使用select()

为了让描述更具体实在，看看select()怎么用的。下图是一个例子：

![1550671992324](/img/in-post/并发性Concurrency.assets/1550671992324.png)

代码很简单，初始化后，服务器进入了一个无限循环。从minFD到maxFD也许代表服务器应该关心的网络sockets。最终，服务器调用了 select() 看哪个链接是可用的。使用 FD_ISSET() 来看哪些描述符有准备好的数据并处理正在到来的数据。（当然真实服务器应用更加复杂）

#### 33.4 不需要Lock所以简单

多线程并发的问题不会出现了，因为只有一个CPU，单线程，不会被其它线程中断，并发的BUG也不会在基本的基于事件的方法中出现。

#### 33.5 问题：阻塞系统调用

虽然迄今为止基于事件的编程看起来可以，只要写一个循环，一直处理事件就行了。但是如果事件要求你执行系统调用怎么办？

在简单的基于事件的方法中，仅仅运行主要的event loop，因此暗示没有其它线程运行着。当需要执行一个阻塞的系统调用，导致系统处于idle状态，造成潜在资源浪费。

#### 33.6 解决：异步I/O

用 **asynchronous I/O** 的方法可以解决上面的限制，这个接口直接返回给应用，此外还有接口让应用知道是否 I/O 工作已经完成了。

这些API一般和一个叫做 **AIO control block** 相关，最简的一个模型如下：

![1550928561029](/img/in-post/并发性Concurrency.assets/1550928561029.png)

当这个结构体被填充后，应用就可以发起系统调用了，比如使用 `int aio_read(struct aiocb *aiocbp);` 这个调用如果成功就直接返回了（此时I/O工作还没完成）。应用需要 `int aio_error(const struct aiocb *aiocbp);` 这个API检查对应的aiocb请求是否完成，完成就返回0。

因此应用可以周期性地通过系统调用 `aio_error()` 来 **poll** 系统以确认I/O是否完成。但是应用一直检查也不是个事，所以一些系统采用 **中断** 的方式。比如使用 信号 通知应用异步I/O完成了。

#### 33.7 问题：状态管理

基于事件的并发比传统的基于线程的代码更加难以编写。因为发起异步调用后，需要为下一个事件（当I/O完成）处理打包程序状态。这种额外的工作在基于线程的方式中，由线程栈管理了。

所以我们需要手动的管理栈。

![1550929459767](/img/in-post/并发性Concurrency.assets/1550929459767.png)

上面的例子中，假如是多线程模式，当read返回后，程序就知道是哪个socket(sd)需要写，这是因为sd存储在栈信息中。

但是基于事件的编程，aio_read()的 I/O 请求完成后，基于事件的服务器并不知道下一步该做什么了。

解决方法也很简单，就是用某个结构把后续的信息记录在案，比如上面的例子中，用哈希表记录 sd ，用 fd 索引，当I/O完成后，通过fd找到后续信息，然后完成剩余的工作。

#### 33.8 事件还有什么难题

1. 当从单核到多核后，多个 event handlers 并行化了，就会出现临界区问题和加锁之类的麻烦事，所以多核后的事件处理不可能不用锁。
2. 虽然显式地避免了一些阻塞问题，但是 page fault 这种隐式的阻塞问题还是没有办法阻止。
3. 程序员要一直管理基于事件的代码，因为非阻塞到阻塞状态一直在变。
4. 反正就是编写起来考虑更多。