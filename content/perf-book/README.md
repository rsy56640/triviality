# [Is Parallel Programming Hard, And, If So, What Can You Do About It?](https://mirrors.edge.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html) 学习笔记

- [1-4 Introduction](#1-4)
- [5 Counting](#5)
- [6 Partitioning and Synchronization Design](#6)
- [7 Locking](#7)
- [8 Data Ownership](#8)
- [9 Deferred Processing](#9)
- [10 Data Structures](#10)
- [11 Validation](#11)
- [](#)
- [](#)


&nbsp;   
<a id="1-4"></a>
## 1-4 Introduction

<img src="assets/Table3-1.png" width="400"/>

- load tearing
  - 编译器用多个 load 表示一个 access
- store tearing
  - 编译器用多个 store 表示一个 access
- load fusing
  - 编译器使用之前的 load 结果（register），而非再次 load
  - `READ_ONCE(x)`
- store fusing
  - 编译器会把对同一内存位置的 store 合并成只有最后一个（*什么鬼玩意？*）
- compiler reordering
  - 事实上，cpp 保证 [Order of evaluation](https://en.cppreference.com/w/cpp/language/eval_order), [6.9.1 Sequential execution](http://eel.is/c++draft/intro.execution#8)
- invented loads：什么玩意
- invented stores：什么玩意
- volatile
  - [P1152R4 Deprecating volatile](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1152r4.html)
  - [P1382R1 volatile_load<T> and volatile_store<T>](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1382r1.pdf)
  - [Nine ways to break your systems code using volatile](https://blog.regehr.org/archives/28)


&nbsp;   
<a id="5"></a>
## 5 Counting


&nbsp;   
<a id="6"></a>
## 6 Partitioning and Synchronization Design


&nbsp;   
<a id="7"></a>
## 7 Locking


&nbsp;   
<a id="8"></a>
## 8 Data Ownership


&nbsp;   
<a id="9"></a>
## 9 Deferred Processing


&nbsp;   
<a id="10"></a>
## 10 Data Structures


&nbsp;   
<a id="11"></a>
## 11 Validation


&nbsp;   
<a id=""></a>
## 

<img src="assets/.png" width="400"/>