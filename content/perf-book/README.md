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

## 参考

- [P0668R5: Revising the C++ memory model](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0668r5.html)
- [[intro.races]](http://eel.is/c++draft/intro.races)：本质上阐述了偏序关系的概念，但似乎仍有些模糊
- [[atomics.order]](http://eel.is/c++draft/atomics.order)


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

> 防止在获取引用计数时，并发地删除数据

### Hazard Pointer


#### Reference

- [Hazard Pointers: Safe Memory Reclamation for Lock-Free Objects](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.395.378&rep=rep1&type=pdf)
- [Lock-Free Data Structures with Hazard Pointers](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.112.6406&rep=rep1&type=pdf)
- [P0233R6 Hazard Pointers - Safe Resource Reclamation for Optimistic Concurrency](http://www.open-std.org/jtc1/SC22/wg21/docs/papers/2017/p0233r6.pdf)
- [Hazard Pointers vs RCU](http://concurrencyfreaks.blogspot.com/2016/08/hazard-pointers-vs-rcu.html)
- [Are Hazard Pointers Lock-Free or Wait-Free ?](http://concurrencyfreaks.blogspot.com/2016/10/are-hazard-pointers-lock-free-or-wait.html)

### Sequence Lock

- 资源**位置固定**
- 多读一（少）写，读者不阻塞
- critical section
  - 单步操作**保证是原子的**
  - **indirect access** 的并发另说
- 退出时检查 seq 来保证整个 critical section 的读是一致的 txn

### RCU

- 资源通过间接寻址
- 多读一写，读者不阻塞
  - 写者：修改间接寻址
  - 读者：不修改间接寻址，但可以修改位置固定的内容
- 读者使用 `rcu_read_lock()` 和 `rcu_read_unlock()` 来界限临界区
- 写者先对数据进行一些修改，然后使用 `synchronize_rcu()` 来确保没有人访问到旧数据，最后释放

#### synchronize_rcu

`synchronize_rcu()` 提供的语义是：读事件（`rcu_read_lock() ... rcu_read_unlock()` 中包裹的过程）不同时并发于 写事件（`rcu_assign_pointer()`） 和 回收事件（`kfree()`）。这等价于：对于任一读事件

- 要么 读事件 *happens-before* 回收事件
- 要么 写事件 *happens-before* 读事件

更进一步：**对于任一读事件，一定存在一个事件`e`位于 `synchronize_rcu()` 过程中，使得**

- **要么 读事件 *happens-before* 事件`e1`**
- **要么 事件`e2` *happens-before* 读事件**
- 注意到 读事件 与 `synchronize_rcu()`过程 可以是并发的

#### RCU 实现

> 通过各种手段来构建 *happens-before*

- **per CPU lock**：`synchronize_rcu()` 拿一遍所有锁
  - 利用 `unlock()` *happens-before* subsequent `lock()`
- **refcount based**：`rcu_read_lock()` 增加 count；`rcu_read_unlock()` 减少 count；`synchronize_rcu()` 等待 count 清零
  - 利用 count 操作是全局有序的（[std::memory_order_seq_cst](http://eel.is/c++draft/atomics.order#7)，**虽然这个语义可能有问题，但我这里假设 load 不到 inc 就是 happens before 了！！！**），**与 `synchronize_rcu()` 中的 load 构建 *happens-before* 关系**
      - 如果 inc -> load，则 dec -> cmp 0，符合语义
      - 如果 load -> inc，符合语义
  - 缺点
      - cacheline contention
          - 解决方案：per CPU count
      - 写者饥饿
          - 解决方案：使用2个计数器。不应该等待 `synchronize_rcu()` “之后开始” 的读者，因此需要构建 *happens-before* 关系，使得这些读者不影响 count。
          - 假设两个计数器A和B，由 `synchronize_rcu()` 指定使用哪个计数器（假设开始使用A）。`synchronize_rcu()` 先更新当前使用的计数器为B，然后等待计数器A清零；然后更新当前使用的计数器为A，然后再等待计数器B清零。
          - 在两个计数器上都 wait 构建了 *happens-before*，`rcu_read_lock()` 中的 inc 与 `synchronize_rcu()` 的2个 wait 之一存在 *happens-before*，因此同上符合语义。
          - 为什么要 wait B 呢？事实上 inc B 不一定是这个 `synchronize_rcu()` “之后”发生的，可能是2轮之前的 `synchronize_rcu()` 之后的。
          - 顺序是
              - flip A to B
              - wait(A)（*与上一条组成 `flip_and_wait(A)`*）
              - flip B to A
              - wait(B)（*如果展开写这条可以放在最前面比较好*）
          - 针对 `synchronize_rcu()` 的并发进一步优化：观察到如果一开始抢不到锁，那么之后可以不等待或减少等待（如果写者并发不严峻就不需要）
              - 使用 count 记录轮数（`flip_and_wait()` 时更新），在拿锁之前第一次读 count 轮数
              - 拿锁后第二次读 count 轮数，若与之前相比跨过了几个 `flip_and_wait()`
                  - 超过 3 轮，直接返回
                      - 为什么是3？因为保证至少涉及 2 个`synchronize_rcu()`，并且最后那个一定是完整的，被第一次读和拿锁包围，用这个 `synchronize_rcu()` 来构建 *happens-before*
                  - 正好 2 轮，执行 1 次 `flip_and_wait()`
                      - 第一轮不能算，所以补一轮，和第二轮一共 2 个 wait 构建 *happens-before*
                  - 不到 2 轮，执行 2 次 `flip_and_wait()`
                      - 强行制造 2 次 wait，构建 *happens-before*
- **sequence 标记**：类似 sequence lock。`rcu_read_lock()` 存储 seq+1（奇数） 到 local；`rcu_read_unlock()` 存储当前 seq 到 local；`synchronize_rcu()` 将 seq+2，偶数递增，然后等待各 thread_local 直到其**不再是“比自己小的奇数”**
  - 通过 local 的 store 和 wait(load) 来构建 *happens-before*
  - 注意到“比自己小的偶数”是合法的，读者早已离开
  - 读性能很好，只有 load，读端不允许嵌套
  - 为什么要用递增偶数，0-1互换也可以？因为0-1互换可能导致饥饿
  - 事实上，`rcu_read_unlock()` 只要存储偶数（比如0）到 local 就可以
- **可嵌套的 sequence 标记**：将 seq 低若干位作为嵌套层数（默认不会溢出）。`rcu_read_lock()` 若为最外层，则存储 seq+1 到 local，否则只将 local+1；`rcu_read_unlock()` 将 local-1；`synchronize_rcu()` 将 seq+一个单位，然后等待各 thread_local 直到其 >= seq，或者读者早已离开（`rcu_gp_ongoing(t)`）
  - 通过最外层 `rcu_read_lock()` 的 store local 构建 *happens-before*（注意到只有最外层会 load seq，内层只有+1嵌套层数）

#### RCU 用途

若不修改间接寻址，则可以不阻塞。当然提交是否原子性要看算法逻辑了。

- RCU 作为 rwlock
- SRCU：读者允许被切
- 并发数据结构的回收

常见用法：对于共享资源，基于refcnt。但是获取资源时并非从现有资源进行共享（例如共享指针），而是从容器中搜索，这意味着获取资源操作需要被检查。

- get：获取一个资源handle
- put：释放资源handle

```c++
T* get(...) {
    rcu_read_lock();
    defer(rcu_read_unlock());
    T* ptr;
    // ... search in container
    ptr = rcu_dereference(...);
    if(ptr) {
        if(atomic_inc_not_zero(ptr->count_)) {
            return nullptr;
        }
    }
    return ptr;
}

void put(T* ptr, void(*release_callback)(T*)) {
    if(atomic_dec_and_test(ptr->count_)) {
        call_rcu(&ptr->rcu_head_, release_callback);
    }
}
```

#### RCU 总结

- 读端应该开销极小：cache miss, cacheline contention
- 读端嵌套
- 读端原语应该可以在所有上下文中使用
- **存在担保**

#### Reference

- [linux/Documentation/RCU](https://github.com/torvalds/linux/tree/master/Documentation/RCU)
- [What is RCU?.txt](https://www.kernel.org/doc/Documentation/RCU/whatisRCU.txt)
- [A Tour Through RCU's Requirements](https://www.kernel.org/doc/Documentation/RCU/Design/Requirements/Requirements.html)
- [Introduction to RCU](http://www2.rdrop.com/users/paulmck/RCU/)
- [What is RCU, Fundamentally? - LWN](https://lwn.net/Articles/262464/)
- [What is RCU? Part 2: Usage - LWN](https://lwn.net/Articles/263130/)
- [RCU part 3: the RCU API - LWN](https://lwn.net/Articles/264090/)
- [Hierarchical RCU - LWN](https://lwn.net/Articles/305782/)
- [Sleepable RCU - LWN](https://lwn.net/Articles/202847/)
- [Priority-Boosting RCU Read-Side Critical Sections - LWN](https://lwn.net/Articles/220677/)
- [Review Checklist for RCU Patches.txt](https://www.kernel.org/doc/Documentation/RCU/checklist.txt)
- [User-space RCU - LWN](https://lwn.net/Articles/573424/)
- [User-space RCU - Hacker News](https://news.ycombinator.com/item?id=17743742)
- [Read-Copy Update (RCU) for C++ - P0279R1](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0279r1.pdf)
- [Yet another introduction to Linux RCU.ppt](https://www.slideshare.net/vh21/yet-another-introduction-of-linux-rcu)
- [folly/folly/synchronization/Rcu.h](https://github.com/facebook/folly/blob/master/folly/synchronization/Rcu.h)
- [Linux 2.6内核中新的锁机制 - RCU](https://www.ibm.com/developerworks/cn/linux/l-rcu/index.html)
- [使用RCU技术实现读写线程无锁](http://codemacro.com/2015/04/19/rw_thread_gc/)
- [Linux 核心設計: RCU 同步機制](https://hackmd.io/@sysprog/linux-rcu?type=view)
- [Linux2.6.23 ：sleepable RCU的实现](http://www.wowotech.net/kernel_synchronization/linux2-6-23-RCU.html)
- [Verification of Tree-Based Hierarchical Read-Copy Update in the Linux Kernel](http://www.kroening.com/papers/date2018-rcu.pdf)
- [Supplementary Material for User-Level Implementations of Read-Copy Update](https://www.researchgate.net/publication/228865533_Supplementary_Material_for_User-Level_Implementations_of_Read-Copy_Update)


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