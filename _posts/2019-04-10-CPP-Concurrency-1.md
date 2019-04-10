---
layout:     post
title:      "C++并发之线程管理"
subtitle:   "CPP concurrency: thread management"
date:       2019-04-09
author:     "TXXT"
catalog:    false
tags:
    - C++
---

# 容易犯错

## cache

cache coherence问题是底层的缓存问题，不同CPU的本地cache之间的一致性问题。解决方案比如MESI协议等（滚去看体系结构）。

因为上面的问题带来的副作用有指令重排，比如有些变量本地store了，但远程的其它线程还没有读到，怎么解决呢？

CPU提供了内存屏障指令，读屏障就是保证之前所有的load生效；写屏障就是保证之前所有的store生效。

## volatile

表示被修饰的对象可能会发生一些不可预期的情况，通俗地举个例子：

```c++
volatile int *p = ....;
a = *p;
b = *p;
```

如果忽略了 volatile，p就是一个指向 int 类型地指针，后面两句有可能就只会去内存中取一次，但实际上这个指针可能是要观察某个硬件设备地情况。（这是程序之外的情况）

所以加入了 volatile 之后，编译器优化就会出现下面情况：

- 不允许被优化消失
- 于序列上在另一个对 `volatile` 对象的访问之前

**所以 volatile 不能解决多线程的问题！！！**

在信号处理和内存映射硬件等场合是有用的。volatile不能解决原子性问题。

# 线程管理

## thread基础

<thread>头文件声明了线程类和一个swap辅助函数，还有 `this_thread` 也声明在此。

在 `this_thread` 命名空间中声明了 get_id，yield，sleep_until，sleep_for 等辅助函数。

对于std::thread，默认构造函数创建空的执行对象，初始化构造函数接受 “函数名+参数列表”。注意，可以被 joinable 的thread对象必须在它被销毁前被主线程join或者detached。

```c++
std::thread t1; //t1 is not a thread
std::thread t2(func, n); //pass by value
std::thread t3(std::move(t2)); // t3 is now running func(),t2 is no longer a thread
```

当然，Lambda表达式也可以，它允许使用一个可以捕获局部变量的局部函数，比如下面：

```c++
std::thread my_thread( []{do_something(); do_something_else();} );
```

注意有些错误可能会发生在变量所在的线程被销毁了，还在访问的情况。比如子线程用引用访问主线程的某个变量，但是主线程已经声明这个子线程被detached，所以可能访问时主线程已经结束完毕了。所以一种比较安全的做法就是将信息都复制给子线程使用，或者join等待完成。

正常情况下主线程会join，一切都美好！但是万一主线程遇到错误终止了怎么办？所以有两种方法规避：1.使用try/catch语句保证catch后做join的动作；2.或者使用RAII思想，用一个类包装thread保证自动析构时在析构函数中调用join。

通常分离后的线程是守护线程。

如果当前的 thread 不可joinable，才能被赋值，比如下面：

```c++
std::thread threads[5];
for (int i = 0; i < 5; i++) {
	threads[i] = std::thread(thread_task, i + 1);
}
```

joinable用来检查是否可以被join，意思就是这个线程是不是一个活动的执行线程。默认构造出来的不能被join。如果某个线程执行完任务，但是没有被join，依然被认为是一个活动的线程。

```c++
std::thread t;
std::cout << "before starting, joinable: " << t.joinable() << '\n';

t = std::thread(foo);
std::cout << "after starting, joinable: " << t.joinable() << '\n';
t.join();
```

谁调用了join就阻塞谁，直到所要join的线程执行完毕才返回。

detach线程意味着将执行的线程和当前线程分离，这样一旦线程执行完毕，分离后的那个线程就会被释放资源。

swap两个线程本质上就是交换它们对应的底层句柄。

## 向线程函数传参

```c++
void f(int i,std::string const& s); 
void oops(int some_param) {
	char buffer[1024]; // 1 
    sprintf(buffer, "%i",some_param); 
    std::thread t(f,3,buffer); // 2 
    t.detach();
}
```

上面的例子中，因为传递的是指针，所以可能在子线程string还没构造好时，主线程就关闭了，所以解决方案就是避免悬垂指针，传之前就构造string：

```c++
std::thread t(f,3,std::string(buffer));
```

另外一个情况是thread构造函数无视函数构造类型，盲目拷贝提供的变量本身，所以当线程入口函数需要的是一个引用时，就会编译错误。下面可以用 std::ref() 将参数转换成引用的方式。

```c++
void f(Weight &weight);

Weight w;
std::thread t(f, w);
std::thread t(f, std::ref(w));
```

成员函数有点特别，需要传递两个参数以确定函数，一个是成员函数地址，一个是对象指针（其实我们平时用的时候，第一个参数也是this指针，只是隐藏了）

```c++
class X { public: void do_lengthy_work(int); };

X my_x; 
int num(0);
std::thread t(&X::do_lengthy_work, &my_x, num);
```

因为类似 unique_ptr 这种是可移动不可以复制的，所以当往thread实例传递参数时必须用 std::move 这种约束保证起到“剪切”的效果。

```c++
std::unique_ptr<big_object> p(new big_object);
std::thread t(process_big_object,std::move(p));
```

## 转移thread所有权

C++标准库中很多类型是资源占有的，比如 ifstream, unique_ptr thread都是可移动不可复制。

```c++
std::thread t1(some_function); 
std::thread t2=std::move(t1);
```

thread本身也是可以当函数参数的：

```c++
void f(std::thread t);

f(std::thread(some_function));
std::thread t(some_function); 
f(std::move(t)); //必须用move
```

## 运行时决定线程数量的例子

下面其实就是实现并行地加和的一个程序。

```c++
template<typename Iterator, typename T>
void accumulate_block(Iterator first, Iterator last, T &result) {
    result = std::accumulate(first, last, result);
}

template<typename Iterator, typename T>
T parallel_accumulate(Iterator first, Iterator last, T init) {
    size_t len = std::distance(first, last);
    if (len == 0)
        return init;
    //这里可以通过API获得程序中的线程数量（8线程电脑就是8）
    size_t num_threads = std::thread::hardware_concurrency();
    size_t block_size = len / num_threads;
    std::vector<T> results(num_threads);
    std::vector<std::thread> threads(num_threads - 1);
    Iterator block_start = first;
    for (int i = 0; i < (num_threads - 1); ++i) {
        Iterator block_end = block_start;
        std::advance(block_end, block_size);
        threads[i] = std::thread(accumulate_block<Iterator, T>, block_start, block_end, std::ref(results[i]));
        block_start = block_end;
    }
    accumulate_block<Iterator, T>(block_start, last, results[num_threads - 1]);
    for (auto &t:threads) {
        t.join();
    }
    return std::accumulate(results.begin(), results.end(), init);
}
```

## 标识线程

下面是一种 thread::id 的用法。

```c++
std::thread::id master_thread; 

void some_core_part_of_algorithm() {
	if(std::this_thread::get_id()==master_thread)
		do_master_thread_work();
    do_common_work();
}
```

