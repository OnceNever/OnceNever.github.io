---
title: ThreadLocal 面试复习：从线程隔离到过期 Key 清理
date: 2026/06/07 13:35:14
categories:
- [JAVA, 并发]
tags:
- ThreadLocal
- ThreadLocalMap
- Java并发
---

### ThreadLocal 面试复习：从线程隔离到过期 Key 清理

最近复习 Java 并发时看了 JavaGuide 的 [ThreadLocal 详解](https://javaguide.cn/java/concurrent/threadlocal.html#threadlocalmap-set-%E5%8E%9F%E7%90%86%E5%9B%BE%E8%A7%A3)。这篇文章源码细节比较多，尤其是 `ThreadLocalMap` 的 `set()`、`get()`、过期 key 清理，看起来很容易绕进去。

如果是面试准备，我觉得不需要把每一行源码都背下来，更重要的是能讲清楚这几个问题：

- `ThreadLocal` 是什么，解决什么问题；
- 数据到底存在哪里；
- 为什么 `ThreadLocalMap` 的 key 要设计成弱引用；
- 为什么还会发生内存泄漏；
- 过期 key 是怎么被清理的；
- 项目中应该怎么安全使用。

---

### 一句话说清楚 ThreadLocal

`ThreadLocal` 可以理解为线程级别的变量副本。

同一个 `ThreadLocal` 对象，在不同线程中访问到的是各自线程内部保存的值。线程之间互不共享，所以它不是通过加锁解决线程安全问题，而是通过**线程隔离**避免共享数据竞争。

面试时可以这样回答：

> `ThreadLocal` 是 JDK 提供的一种线程局部变量机制。每个线程内部都有一个 `ThreadLocalMap`，`ThreadLocal` 作为 key，线程自己的变量副本作为 value。不同线程访问同一个 `ThreadLocal` 时，实际读写的是各自线程内部 Map 中的值，所以可以实现线程隔离。

注意这句话里有一个关键点：**数据不是存在 `ThreadLocal` 对象里，而是存在当前线程的 `ThreadLocalMap` 里。**

---

### ThreadLocal 适合解决什么问题？

`ThreadLocal` 适合保存和当前线程强相关、需要在同一个调用链中传递的上下文信息。

常见场景有：

- 保存用户上下文，比如当前登录用户 ID；
- 保存请求链路追踪 ID，比如 `traceId`、`requestId`；
- 日志框架中的 MDC；
- 数据库连接、事务上下文；
- 避免某些对象频繁创建，例如 `SimpleDateFormat` 的旧版本线程不安全问题。

但是它不适合做跨线程数据传递。因为 `ThreadLocal` 的核心语义就是“线程隔离”，子线程默认拿不到父线程的 `ThreadLocal` 值。即使用 `InheritableThreadLocal`，在线程池场景下也容易出问题，因为线程池里的线程会复用，不一定每次都是新建线程。

---

### ThreadLocal 的数据结构

每个 `Thread` 对象内部都有一个字段：

```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```

当我们调用：

```java
threadLocal.set(value);
```

大致流程是：

```java
Thread t = Thread.currentThread();
ThreadLocalMap map = getMap(t);
if (map != null) {
    map.set(this, value);
} else {
    createMap(t, value);
}
```

也就是说：

- `ThreadLocal` 负责提供 `set()`、`get()`、`remove()` 这些操作入口；
- 当前线程 `Thread` 负责持有自己的 `ThreadLocalMap`；
- `ThreadLocalMap` 里保存真正的数据；
- `key` 是 `ThreadLocal`；
- `value` 是业务数据。

可以用一个简单图示理解：

```text
Thread-1
  └── threadLocals(ThreadLocalMap)
        ├── Entry(ThreadLocalA -> valueA1)
        └── Entry(ThreadLocalB -> valueB1)

Thread-2
  └── threadLocals(ThreadLocalMap)
        ├── Entry(ThreadLocalA -> valueA2)
        └── Entry(ThreadLocalB -> valueB2)
```

同一个 `ThreadLocalA`，在不同线程的 `ThreadLocalMap` 中对应不同 value，这就是线程隔离的来源。

---

### ThreadLocalMap 和 HashMap 有什么不同？

`ThreadLocalMap` 也是数组结构，但它不是普通的 `HashMap`。

普通 `HashMap` 解决 hash 冲突通常使用链表或红黑树，而 `ThreadLocalMap` 使用的是**开放寻址法中的线性探测**。

什么意思呢？

假设某个 `ThreadLocal` 算出来应该放在数组下标 4，但是下标 4 已经有数据了，那它不会在 4 后面挂链表，而是继续找 5、6、7……直到找到一个空槽位。

简化理解：

```text
理想位置是 4

下标: 0 1 2 3 4 5 6 7
数据: - - - - A B - -

新数据也想放 4，但 4 已经有 A
于是往后找，发现 6 是空的，就放到 6
```

这也解释了为什么 `ThreadLocalMap` 清理过期 key 时，不只是把当前位置置空这么简单。因为如果直接置空，可能会破坏线性探测形成的查找链，导致后面的正常数据查不到。

---

### 为什么 key 要用弱引用？

`ThreadLocalMap.Entry` 的源码大致是：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

也就是说：

- `Entry` 的 key 是 `ThreadLocal` 的弱引用；
- `Entry` 的 value 是强引用。

弱引用的特点是：只要一次 GC 发生，并且这个对象没有其他强引用，那么弱引用指向的对象就会被回收。

为什么要这么设计？

因为如果 key 是强引用，就可能出现这样的引用链：

```text
Thread -> ThreadLocalMap -> Entry -> ThreadLocal(key)
```

只要线程不结束，`ThreadLocalMap` 就一直存在，`ThreadLocal` 对象也会一直被 key 强引用着，无法回收。

而 key 使用弱引用后，如果业务代码里已经没有变量再引用这个 `ThreadLocal`，GC 后 key 就可以变成 `null`。这样至少 `ThreadLocal` 对象本身不会因为 `ThreadLocalMap` 而泄漏。

---

### key 都是弱引用了，为什么还会内存泄漏？

这是 ThreadLocal 面试里最高频的问题之一。

原因在于：**key 是弱引用，但 value 仍然是强引用。**

当 `ThreadLocal` 外部强引用消失后，GC 会把 key 回收掉，于是 `ThreadLocalMap` 里会出现这样的 Entry：

```text
Entry(null -> value)
```

这个 `Entry` 的 key 已经没了，但 value 还在，而且 value 的引用链仍然存在：

```text
Thread -> ThreadLocalMap -> Entry -> value
```

如果当前线程很快结束，那么问题不大，线程对象被回收，整个 `ThreadLocalMap` 和 value 都会被回收。

但在线程池中，工作线程通常会长期存活。只要线程不结束，`ThreadLocalMap` 就可能一直存在，`Entry(null -> value)` 也可能一直占着内存。这就是 `ThreadLocal` 内存泄漏的根源。

所以面试里可以这样答：

> `ThreadLocalMap` 的 key 是弱引用，GC 后 key 可能变成 null，但 value 是强引用，仍然被 `Thread -> ThreadLocalMap -> Entry -> value` 这条链路引用。如果线程长期不结束，比如线程池场景，并且没有及时清理这个 Entry，value 就无法被 GC，可能造成内存泄漏。

---

### 过期 key 是什么？

所谓过期 key，指的就是 `Entry` 里的 `key` 已经被 GC 回收，变成了 `null`。

也就是：

```text
正常 Entry：ThreadLocal 对象 -> value
过期 Entry：null -> value
```

源码里通常称这种 Entry 为 `stale entry`，可以理解为“陈旧的、失效的 Entry”。

---

### 过期 key 的清理：先记住一句话

`ThreadLocalMap` 不会有一个后台线程专门清理过期 key，它采用的是**惰性清理**。

也就是说，只有当你调用 `get()`、`set()`、`remove()` 这些方法时，才可能顺便触发清理。

面试回答可以先说这个主线：

> `ThreadLocalMap` 对过期 key 的清理是惰性的。`get()`、`set()`、`remove()` 操作过程中，如果发现某个 Entry 的 key 已经是 null，就会调用清理逻辑，把这个 Entry 的 value 置空、数组槽位置空，并对后续因为线性探测产生冲突的 Entry 做重新定位，避免查找链断裂。

这句话已经能覆盖面试中大部分问题。

---

### 通俗理解 expungeStaleEntry

`expungeStaleEntry()` 是清理过期 Entry 的核心方法。

这个方法可以通俗理解成：

> 从发现垃圾的位置开始，先把这个垃圾桶清空，然后沿着数组继续往后检查。遇到新的垃圾继续清理，遇到正常数据就判断它是不是因为 hash 冲突才被挤到当前位置，如果是，就尽量把它搬回更合适的位置。

为什么要“搬家”？

因为 `ThreadLocalMap` 使用线性探测。举个例子：

```text
下标: 0 1 2 3 4 5 6 7
数据: - - - - A X B -
```

假设：

- A 的理想位置是 4；
- X 是过期 Entry，key 已经是 null；
- B 的理想位置也是 4，但因为 4、5 被占了，所以当初被放到了 6。

如果清理 X 时只是把 5 号位置置空，会变成：

```text
下标: 0 1 2 3 4 5 6 7
数据: - - - - A - B -
```

这时如果查找 B，会从理想位置 4 开始找：

1. 看 4，是 A，不是 B；
2. 往后看 5，发现是空；
3. 线性探测认为“遇到空就说明后面没有了”，于是停止查找；
4. 结果 B 明明在 6，却查不到了。

所以 `expungeStaleEntry()` 不能只清理过期槽位，还要把后面受影响的数据重新整理一下。

整理后的效果类似：

```text
下标: 0 1 2 3 4 5 6 7
数据: - - - - A B - -
```

B 被搬到更靠近理想位置的地方，查找链恢复正常。

因此，`expungeStaleEntry()` 做了三件事：

1. 清理当前过期 Entry：`value = null`，数组槽位设为 `null`，`size--`；
2. 从当前位置继续向后扫描，直到遇到空槽位停止；
3. 扫描过程中如果又遇到过期 Entry，继续清理；如果遇到正常 Entry，就重新计算它应该在的位置，必要时重新放置。

---

### set() 时怎么清理过期 key？

`set()` 的主流程可以简化成下面几种情况：

#### 1. 理想槽位为空

直接把新 Entry 放进去。

放完之后，会调用 `cleanSomeSlots()` 做一次启发式清理，也就是顺便向后扫一小段，看有没有过期 Entry。

#### 2. 理想槽位不为空，并且 key 相同

说明是更新已有值，直接覆盖 value。

#### 3. 理想槽位不为空，但 key 不同

说明发生了 hash 冲突，需要继续向后线性探测。

探测过程中可能遇到两种情况：

- 遇到相同 key：更新 value；
- 遇到 `key == null`：说明遇到了过期 Entry，此时会调用 `replaceStaleEntry()`。

`replaceStaleEntry()` 可以理解为“复用过期槽位 + 清理周边垃圾”。它会把新数据放到这个过期槽位上，同时尝试清理附近其他过期 Entry。

面试里不一定要背 `replaceStaleEntry()` 的完整源码，只要能说出核心目的：

> `set()` 过程中如果发现过期 Entry，会优先复用这个槽位存放新值，并触发清理逻辑，清掉周围 key 为 null 的 Entry，避免无效 value 长期占用内存。

---

### get() 时怎么清理过期 key？

`get()` 的流程相对简单。

它会先根据当前 `ThreadLocal` 的 hash 值定位到理想槽位：

- 如果槽位中的 key 正好是当前 `ThreadLocal`，直接返回 value；
- 如果不是，就说明可能发生了 hash 冲突，需要向后找；
- 向后找的过程中，如果遇到 `key == null` 的 Entry，就调用 `expungeStaleEntry()` 清理。

所以 `get()` 不是每次都会清理。只有在查找过程中发生 miss，并且碰到过期 Entry 时，才会触发清理。

这也是为什么 `ThreadLocal` 不能完全依赖自动清理来避免泄漏。

如果某些过期 Entry 所在位置之后再也没有被访问到，那么它可能长时间留在 Map 中。

---

### remove() 为什么最可靠？

`remove()` 是最直接的清理方式。

调用：

```java
threadLocal.remove();
```

会从当前线程的 `ThreadLocalMap` 中移除当前 `ThreadLocal` 对应的 Entry，并触发相关清理。

和被动等待 `get()`、`set()` 时顺便清理相比，`remove()` 是主动清理。尤其在线程池场景下，线程会被复用，如果不调用 `remove()`，上一次任务留下的数据可能影响下一次任务，也可能造成内存泄漏。

推荐写法是：

```java
private static final ThreadLocal<String> TRACE_ID = new ThreadLocal<>();

public void handleRequest(String traceId) {
    try {
        TRACE_ID.set(traceId);
        // 业务逻辑
    } finally {
        TRACE_ID.remove();
    }
}
```

重点是：**set 之后，一定要在 finally 里 remove。**

---

### 高频面试题整理

#### 1. ThreadLocal 的原理是什么？

每个线程内部都有一个 `ThreadLocalMap`，`ThreadLocal` 作为 key，业务对象作为 value。调用 `set()`、`get()` 时，先拿到当前线程，再操作当前线程自己的 `ThreadLocalMap`，所以不同线程之间的数据互不影响。

#### 2. ThreadLocal 的数据存在哪里？

存在当前线程对象的 `ThreadLocalMap` 中，不是存在 `ThreadLocal` 对象本身。

#### 3. ThreadLocalMap 的 key 和 value 分别是什么？

key 是 `ThreadLocal` 对象的弱引用，value 是业务数据的强引用。

#### 4. 为什么 key 要设计成弱引用？

为了避免 `ThreadLocalMap` 强引用 `ThreadLocal`，导致业务代码已经不再使用的 `ThreadLocal` 仍然无法被 GC。弱引用可以让没有外部强引用的 `ThreadLocal` 在 GC 时被回收。

#### 5. 为什么 key 是弱引用还会内存泄漏？

因为 value 是强引用。GC 后 key 可能变成 null，但 value 仍然被 `Thread -> ThreadLocalMap -> Entry -> value` 引用。线程池中的线程长期存活时，value 可能长期无法回收。

#### 6. 过期 key 是怎么清理的？

`ThreadLocalMap` 采用惰性清理。调用 `get()`、`set()`、`remove()` 时，如果发现 `Entry.key == null`，会清理这个 Entry，并在线性探测范围内继续扫描，清理其他过期 Entry，同时对正常 Entry 重新定位，避免破坏查找链。

#### 7. ThreadLocalMap 怎么解决 Hash 冲突？

使用线性探测法。hash 冲突时，不使用链表，而是向后寻找下一个可用槽位。

#### 8. 为什么清理过期 Entry 后还要重新定位后面的 Entry？

因为线性探测依赖连续的查找链。如果中间某个槽位被直接置空，后面的 Entry 可能再也查不到。重新定位可以保证后续 Entry 仍然能被正确查找。

#### 9. ThreadLocal 在项目中怎么避免问题？

- 尽量定义为 `private static final`；
- 使用后在 `finally` 中调用 `remove()`；
- 在线程池场景特别注意清理；
- 不要用它做跨线程传值；
- 异步任务需要传递上下文时，考虑显式传参、包装任务，或者使用 TransmittableThreadLocal 等工具。

---

### 面试回答模板

如果面试官问：**讲讲 ThreadLocal，以及它的内存泄漏问题。**

可以这样答：

> `ThreadLocal` 是线程局部变量工具。每个线程内部都有一个 `ThreadLocalMap`，`ThreadLocal` 作为 key，业务对象作为 value。调用 `get()`、`set()` 时，实际操作的是当前线程自己的 Map，所以可以实现线程隔离。
>
> `ThreadLocalMap` 的 key 是弱引用，value 是强引用。key 使用弱引用是为了避免业务代码不再引用 `ThreadLocal` 后，它还被 Map 强引用导致无法回收。但这也带来一个问题：GC 后 key 可能变成 null，而 value 仍然被线程的 `ThreadLocalMap` 强引用。如果线程长期存活，比如线程池线程，就可能造成 value 无法释放，也就是内存泄漏。
>
> JDK 内部在 `get()`、`set()`、`remove()` 时会做惰性清理，发现 key 为 null 的 Entry 后，会把 value 置空、槽位置空，并对线性探测范围内的 Entry 做重新整理。但这种清理不是实时的，也不是一定会触发。所以项目里使用 ThreadLocal 后，最好在 `finally` 中主动调用 `remove()`。

---

### 总结

`ThreadLocal` 的核心并不复杂：它通过“每个线程持有自己的 `ThreadLocalMap`”实现线程隔离。

真正容易被问深的是 `ThreadLocalMap`：

- key 是弱引用，value 是强引用；
- key 被 GC 后会形成 `null -> value` 的过期 Entry；
- 线程池线程长期存活时，value 可能泄漏；
- 清理过期 key 是惰性的，依赖 `get()`、`set()`、`remove()` 触发；
- 因为使用线性探测，清理时还要重新整理后续 Entry，不能简单置空了事。

面试准备到这个程度，基本就能把 `ThreadLocal` 从“会用”讲到“理解底层原理”了。
