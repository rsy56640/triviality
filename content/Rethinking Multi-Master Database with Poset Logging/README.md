# Rethinking Multi-Master Database with Poset Logging

存储层怎么扩展？

- shared nothing (parition)
  - 计算层协调执行：spanner
  - 计算层确定性执行：calvin
- shared everything
  - Aurora (the log is the database)

下面只讨论 shared everything，不讨论 partition。

log is db 原理就是**确定性状态机**，这里的 log 是 total order，只要（逻辑上，允许并发）顺序 replay log 即可。数据放在远端，提供 fetch-page 服务，log 提供顺序写服务，只读结点可以消费 log 进行回放。

计算层怎么扩展？

- 通常 TP 会从存储层拉数据（包括索引）到计算结点，相当于在计算结点做了缓存，多计算结点就有多份缓存，需要考虑缓存一致性，类似 CPU 的 MOESI
- log 是全序寻址的，所以是一个不能 scalable 的瓶颈，**那么能否将 log 进行 partition 做成偏序的**？

记 log 的原因在于需要 replay，通常认为按照全序进行确定性回放就能保证状态一致。那如果 log 是偏序呢？

首先，log 怎样是偏序呢？很自然的想法是 lamport clock，对每个 obj 的修改（或“访问”？load 也要视为 store，为了 RO 的 seq_cst 语义？）都带有 sequence，每个计算结点自增，当拿到一个 obj，若其 sequence > 该计算结点当前 sequence 时，更新当前 sequence。这样**对于同一个 obj 的更新的 sequence 满足 coherence order**；**对于不同 obj 的更新的 sequence 也满足 partial order**。问题在于偏序的展开是有多种情况，我们能否将这样的 sequence 当做一种展开并认为其满足语义？

假设我们按照 lsn 顺序在 RO 结点回放，考虑 P1 上的写操作 log 为 [W(A) lsn1=100]，P2 上的写操作 log 为 [W(B) lsn2=80]。于是 RO 结点认为 B 的修改在 A 之前。那么对于 RW 结点，是否存在一种情况，使得可以读到 W(A)，但是之后读不到 W(B)。**这是有可能的，因为对 obj 更新 sequence 的时机仅发生在 store 时，而非 load 时**。换句话说，如果希望不同结点对 store order 达成一致，需要在所有“访问” obj 的时机去更改其 sequence，这种访问当然也包括 load。（这对于 RW 结点来说当然是一种负担，因为这仅仅保证了 RO 结点对 store order 的看法与 RW 结点保持一致）

> "Any two stores are seen in a consistent order by processors other than those performing the stores" - IA32 TSO 的额外福利


总结一下：分布式 TP 的逻辑主要有两套

- **对于 RW 结点**：**相当于 HTM，额外加上偏序的 log**
- **对于 RO 结点或 recovery**：**通过偏序 log 构建确定性状态机**
  - 这里的“**构建**”暗示了一种人为的手段，并且可能会蕴含不同的**语义**

