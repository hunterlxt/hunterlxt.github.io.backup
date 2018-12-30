---
layout:     post
title:      "函数调用堆栈"
subtitle:   "function call stack"
date:       2018-08-15
author:     "TXXT"
catalog:    true
tags:
    - 编译
---

call stack是一个栈数据结构，用来存放程序的active subroutines。栈的作用很多，但主要是为了追踪当active subroutine执行完毕后的返回控制。

## 1 描述

对于call stack，caller把返回的地址压入栈，当subroutine完成后，从栈里弹出后续控制的地址。如果压入的地址用完了call stack的分配空间，那么就会栈溢出错误。通常一个线程只有对应一个call stack。

## 2 结构

call stack是由stack frames构成的。这些帧包含了subroutine的状态信息。每一个stack frame对应一个对还没结束的subroutine的调用。比如如果`DrawSquare`调用了`DrawLine`，那么情形如下所示：

![img](https://upload.wikimedia.org/wikipedia/commons/thumb/d/d3/Call_stack_layout.svg/684px-Call_stack_layout.svg.png)

一般情况下fame得有下面几点：

- 传到这个routine的参数
- 返回到调用它的那个东西的地址
- routine的本地变量的空间

**stack and frame pointers**

当stack frame可能不同大小时，每个stack frame包含一个stack pointer指向紧邻下面的frame。

## 3 stack pointer & frame pointer的不同(Quora)

![img](https://qph.fs.quoracdn.net/main-qimg-24a9b0bb2ce5f71fb42740dea5774338.webp)

![img](https://qph.fs.quoracdn.net/main-qimg-fcf3c443194286143065e6809c82a517.webp)

![img](https://qph.fs.quoracdn.net/main-qimg-7af53a9c12c9336bedc01dfca7f98b34.webp)

## 4 cost of a function call

```c
function (){
  var c;
  for (var i = 0; i < 10000000; i++) {
    c = a + i;
  }
}

function (){
  var c;
  for (var i = 0; i < 10000000; i++) {
    c = add(a, i);
  }
}
//这两个执行时间分别是2.77ms和2.76ms，因为有内联优化的存在。
```

#### 4.1 什么是function call

现在看下面这个代码：

```
result = do_something( param1, param2 );
```

具体发生什么步骤取决于OS，语言，调用环境，但是每个情况下如下的步骤都会执行：

1. 解析参数的地址和类型
2. 解析result的地址和类型
3. 解析函数名，找到确切的调用函数
4. 将参数转换为接收函数的类型
5. 转换后的参数压入栈
6. 返回地址压入栈
7. 跳转到函数地址
8. 函数做的事情
9. 跳到返回地址
10. 从栈里弹出返回值
11. 从栈里弹出参数
12. 把返回值转换为"result"的类型并保存

#### 4.2 解析

大部分语言如C，解析是在编译阶段完成，运行时只要添加少量数据就能算出真实地址。

#### 4.3 push and pop

在一些编译器优化中，编译器会通过在栈上重用变量避免大量的推送和弹出操作。理想情况下，编译器实际上可以完全消除参数的push and pop。

#### 4.4 Jump

subroutine做完之后会返回到调用它的地方，return address也是保存在stack里，但是因为这个东西过于常见，CPU一般都有特殊的功能处理这类情况。CPU做比push and pop快的多。

当然在编译阶段也可以通过内敛做到这一点。这只在subroutine很小时才适合。

#### 4.5 函数调用

理想情况下，函数调用成本接近于0.对于优化很好的编译语言，是很常见的。但解释型语言的函数调用成本就比较高了。对于JavaScript这种语言，函数调用的开销非常大。

