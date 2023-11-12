# 【日常技术批判】（未验证）mem cgroup 可能超额分配？

## 说明

**本文有一定概率不成立，甚至是完全错误的，因为笔者并不熟悉 cgroup，而且也没有进行实验，所做出的观察也只是盲人摸象。**

## 怀疑现象

`memcg` 有两个变量：`usage` 和 `limit`，分别表示当前使用和上限。而这两个值是允许并发更新的，某些场景可能会导致 `usage > limit` 的最终情况。如果这一观察属实，那么确实是有可能被恶意利用的漏洞。

## 前提条件

> 或者说我不清楚的地方。

- charge 可以并发进入
  - 应该是可以的，`do_anonymous_page` 会加 mmap read lock，更别说之后会去掉这个改成 per vma lock
- `do_try_to_free_pages` 返回 `nr_reclaimed=0`
  - 什么条件下发生？不太清楚
- 这里 per-cpu cache 不太清楚
- charge nr_page 值区间不清楚
- set limit 导致的 reclaim activity 不清楚

## 代码分析

主要观察两个函数

- `page_counter_try_charge`：分配物理内存时进行 charge usage，栈来源是 `do_anonymous_page`
- `page_counter_set_max`：设置 limit

```c++
/**
 * page_counter_try_charge - try to hierarchically charge pages
 * @counter: counter
 * @nr_pages: number of pages to charge
 * @fail: points first counter to hit its limit, if any
 *
 * Returns %true on success, or %false and @fail if the counter or one
 * of its ancestors has hit its configured limit.
 */
bool page_counter_try_charge(struct page_counter *counter,
			     unsigned long nr_pages,
			     struct page_counter **fail)
{
	struct page_counter *c;

	for (c = counter; c; c = c->parent) {
		long new;
		/*
		 * Charge speculatively to avoid an expensive CAS.  If
		 * a bigger charge fails, it might falsely lock out a
		 * racing smaller charge and send it into reclaim
		 * early, but the error is limited to the difference
		 * between the two sizes, which is less than 2M/4M in
		 * case of a THP locking out a regular page charge.
		 *
		 * The atomic_long_add_return() implies a full memory
		 * barrier between incrementing the count and reading
		 * the limit.  When racing with page_counter_limit(),
		 * we either see the new limit or the setter sees the
		 * counter has changed and retries.
		 */
		new = atomic_long_add_return(nr_pages, &c->usage);
		if (new > c->max) {
			atomic_long_sub(nr_pages, &c->usage);
			propagate_protected_usage(counter, new);
			/*
			 * This is racy, but we can live with some
			 * inaccuracy in the failcnt.
			 */
			c->failcnt++;
			*fail = c;
			goto failed;
		}
		propagate_protected_usage(counter, new);
		/*
		 * Just like with failcnt, we can live with some
		 * inaccuracy in the watermark.
		 */
		if (new > c->watermark)
			c->watermark = new;
	}
	return true;

failed:
	for (c = counter; c != *fail; c = c->parent)
		page_counter_cancel(c, nr_pages);

	return false;
}
```

只考虑一层 memcg，操作是先对 usage FAA，然后 load limit，检查是否超额，如果超了就减去使用量。

```c++
/**
 * page_counter_set_max - set the maximum number of pages allowed
 * @counter: counter
 * @nr_pages: limit to set
 *
 * Returns 0 on success, -EBUSY if the current number of pages on the
 * counter already exceeds the specified limit.
 *
 * The caller must serialize invocations on the same counter.
 */
int page_counter_set_max(struct page_counter *counter, unsigned long nr_pages)
{
	for (;;) {
		unsigned long old;
		long usage;

		/*
		 * Update the limit while making sure that it's not
		 * below the concurrently-changing counter value.
		 *
		 * The xchg implies two full memory barriers before
		 * and after, so the read-swap-read is ordered and
		 * ensures coherency with page_counter_try_charge():
		 * that function modifies the count before checking
		 * the limit, so if it sees the old limit, we see the
		 * modified counter and retry.
		 */
		usage = atomic_long_read(&counter->usage);

		if (usage > nr_pages)
			return -EBUSY;

		old = xchg(&counter->max, nr_pages);

		if (atomic_long_read(&counter->usage) <= usage)
			return 0;

		counter->max = old;
		cond_resched();
	}
}
```

每个循环做如下操作：load usage，判断符合 new limit，xchg old limit，再次 load usage，检查是否增长，如果没增长就算成功，否则回退 old limit。

## 漏洞行为模拟

现在考虑4个进程 P1~4 操作同一个 memcg，初始状态 `usage=90, limit=100`

- P1：charge 30
- P2：charge 30
- P3：charge 100
- P4：set max 200

| P1 | P2 | P3 | P4 | status |
| :----:| :----: | :----: | :----: | :----: |
| &nbsp; | &nbsp; | &nbsp; | check usage=90 ok | usage=90, limit=100 |
| charge usage:90->120 | &nbsp; | &nbsp; | &nbsp; | usage=120, limit=100 |
| &nbsp; | charge usage:120->150 | &nbsp; | &nbsp; | usage=150, limit=100 |
| check limit=100 fail | &nbsp; | &nbsp; | &nbsp; | usage=150, limit=100 |
| &nbsp; | &nbsp; | &nbsp; | xchg limit:100->200 | usage=150, limit=200 |
| &nbsp; | check limit=200 ok | &nbsp; | &nbsp; | usage=150, limit=200 |
| rollback usage:150->120 | &nbsp; | &nbsp; | &nbsp; | usage=120, limit=200 |
| &nbsp; | &nbsp; | &nbsp; | recheck usage=120 fail | usage=120, limit=200 |
| &nbsp; | &nbsp; | &nbsp; | rollback limit:200->100 | usage=120, limit=100 |
| &nbsp; | &nbsp; | &nbsp; | next loop | usage=120, limit=100 |
| &nbsp; | &nbsp; | charge usage:120->220 | &nbsp; | usage=220, limit=100 |
| &nbsp; | &nbsp; | &nbsp; | check usage=220 fail | usage=220, limit=100 |
| &nbsp; | &nbsp; | &nbsp; | return -EBUSY | usage=220, limit=100 |
| &nbsp; | &nbsp; | check limit=100 fail | &nbsp; | usage=220, limit=100 |
| &nbsp; | &nbsp; | rollback usage:220->120 | &nbsp; | usage=120, limit=100 |

注意到 `mem_cgroup_resize_max` 里面对 `page_counter_set_max` 还是循环，但是如果 `try_to_free_mem_cgroup_pages` 返回 0（这里是什么情况），就会跳出循环，返回 `-EBUSY`。

结果：初始状态 `usage=90, limit=100`，P2 charge 成功（P1 P3 可能嘎了，无所谓），并没有触发 OOM，最终变成 `usage=120, limit=100`，memcg 出现超额分配的情况。

## 最后

笔者没有鼓捣过可以 debug linux 的环境，所以就不做实验了。另外前提条件是否苛刻，可能这种场景很极端，而且每次 charge 的 `nr_pages` 大小感觉不会太大，所以并不严重？可能影响就是额外的 reclaim activity？

## 补充

主要翻一下邮件列表，似乎也有类似的怀疑。

- [Memory Resource Controller - 2.1. Design](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/memory.html#design)
- [Memory Resource Controller - 2.5 Reclaim](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/memory.html#reclaim)
- 对应 patch 和 ml
  - [mm: memcontrol: lockless page counters](https://github.com/torvalds/linux/commit/3e32cb2e0a12b6915056ff04601cf1bb9b44f967)
  - [Subject: [patch 1/3] mm: memcontrol: lockless page counters](https://lore.kernel.org/all/1413251163-8517-2-git-send-email-hannes@cmpxchg.org/)

> - to keep charging light-weight, page_counter_try_charge() charges
>   speculatively, only to roll back if the result exceeds the limit.
>   Because of this, a failing bigger charge can temporarily lock out
>   smaller charges that would otherwise succeed.  The error is bounded
>   to the difference between the smallest and the biggest possible
>   charge size, so for memcg, this means that a failing THP charge can
>   send base page charges into reclaim upto 2MB (4MB) before the limit
>   would have been reached.  This should be acceptable.

- [怀疑 page_counter_limit 和 page_counter_try_charge 有 race，不过只是从 barrier 的角度](https://lore.kernel.org/all/20140926103104.GE29445@esperanza/)
- [为什么要 loop set limit，防止 spurious -EBUSY，注意 reclaim activity](https://lore.kernel.org/all/20141002150135.GA1394@cmpxchg.org/)
- [作者怀疑 speculative charge](https://lore.kernel.org/all/20141002195214.GA2705@cmpxchg.org/)
- [per-cpu cache 减少 race](https://lore.kernel.org/all/20141003145042.GC4816@dhcp22.suse.cz/)，这里是啥意思
