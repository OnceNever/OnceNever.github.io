---
title: AQS 面试复习：从队列同步器到常见并发工具
date: 2026/06/05 20:04:17
categories:
- [JAVA, 并发]
tags:
- AQS
- ReentrantLock
- Semaphore
- CountDownLatch
- CyclicBarrier
---

### AQS 面试复习：从队列同步器到常见并发工具

最近复习 Java 并发时又看了一遍 JavaGuide 的 [AQS 详解](https://javaguide.cn/java/concurrent/aqs.html#aqs-%E4%BB%8B%E7%BB%8D)。这篇文章内容很多，如果直接背源码，很容易陷入“方法名都记住了，但面试时讲不出主线”的状态。

我觉得复习 AQS 更适合按面试回答思路来理解：**AQS 是什么、解决了什么问题、核心结构是什么、独占/共享两种模式怎么工作、常见工具类如何基于它实现**。

---

### 一句话说清楚 AQS

AQS，全称 `AbstractQueuedSynchronizer`，可以理解为 JUC 中很多锁和同步工具的**基础框架**。

它本身不是一个具体的锁，而是提供了一套通用机制：

- 用一个 `volatile int state` 表示同步状态；
- 用一个 FIFO 的 CLH 变体队列管理获取资源失败的线程；
- 用模板方法把“排队、阻塞、唤醒”这些通用流程封装起来；
- 让具体同步器只需要关心“怎么获取资源、怎么释放资源”。

所以面试里如果问：**AQS 是干什么的？**

可以这样答：

> AQS 是 JUC 中构建锁和同步器的基础框架。它通过 `state` 表示同步状态，通过 CLH 变体队列管理等待线程，并提供独占和共享两种资源获取模式。像 `ReentrantLock`、`Semaphore`、`CountDownLatch`、`ReentrantReadWriteLock` 等都基于 AQS 实现。

---

### AQS 的核心：state + 等待队列

AQS 最重要的两个组成部分：

#### 1. state：同步状态

`state` 是 AQS 内部的核心变量：

```java
private volatile int state;

protected final int getState() {
    return state;
}

protected final void setState(int newState) {
    state = newState;
}

protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

它由 `volatile` 修饰，保证可见性和一定的有序性；修改时通常配合 CAS，保证多线程竞争下的原子更新。

不同同步器会给 `state` 赋予不同含义：

| 同步器 | 模式 | state 的含义 |
| --- | --- | --- |
| ReentrantLock | 独占 | 锁的重入次数，0 表示未被占用 |
| Semaphore | 共享 | 可用许可证数量 |
| CountDownLatch | 共享 | 剩余计数，减到 0 后唤醒等待线程 |
| ReentrantReadWriteLock | 独占 + 共享 | 高 16 位表示读锁数量，低 16 位表示写锁重入次数 |

这里要注意：**AQS 只定义通用同步状态，具体语义由子类决定**。

#### 2. CLH 变体队列：等待线程排队

当线程尝试获取资源失败时，AQS 会把当前线程封装成一个 Node 节点，加入等待队列。

Node 中大致会保存这些信息：

- 当前线程 `thread`；
- 前驱节点 `prev`；
- 后继节点 `next`；
- 节点等待状态 `waitStatus`。

AQS 使用的是 CLH 锁队列的变体。原始 CLH 锁偏向单向链表和自旋等待，而 AQS 做了两个重要调整：

1. **自旋 + 阻塞结合**：获取失败后不会一直空转消耗 CPU，而是在合适时机通过 `LockSupport.park()` 挂起线程；
2. **单向队列改成双向队列**：节点不仅知道前驱，也能找到后继，释放资源时更方便唤醒后续节点。

所以 AQS 的等待队列可以概括为：**获取失败就入队，排到队首再尝试获取，仍失败就阻塞，前驱释放后再被唤醒继续竞争**。

---

### AQS 为什么性能比较好？

AQS 的性能主要来自几个点：

1. **CAS 乐观更新 state**：竞争不激烈时，不需要直接加重量级互斥；
2. **失败线程进入队列**：避免所有线程一直抢同一个共享变量；
3. **短暂自旋 + 阻塞**：在吞吐量和 CPU 消耗之间做平衡；
4. **按队列唤醒后继节点**：避免无意义地唤醒大量线程造成惊群。

面试时不要只说“因为用了 CAS”，更完整的回答应该是：

> AQS 通过 CAS 修改同步状态，在竞争不激烈时可以很快成功；竞争失败的线程会进入 CLH 变体队列，减少无序竞争；等待线程不会一直自旋，而是会被挂起，等前驱释放后再唤醒。CAS、队列化和阻塞唤醒机制结合起来，使它在并发场景下有比较好的性能。

---

### 独占模式和共享模式

AQS 支持两类资源共享方式：

- **Exclusive 独占模式**：同一时刻只有一个线程能获取资源，例如 `ReentrantLock`；
- **Shared 共享模式**：同一时刻允许多个线程获取资源，例如 `Semaphore`、`CountDownLatch`。

#### 独占模式

独占模式常见模板方法是：

- `tryAcquire(int arg)`：尝试获取资源；
- `tryRelease(int arg)`：尝试释放资源。

AQS 本身不知道具体怎么获取锁，所以这两个方法通常由子类重写。

以 `ReentrantLock` 为例：

- `state == 0`：锁空闲；
- 某线程 CAS 把 `state` 从 0 改成 1，获取锁成功；
- 如果当前线程已经持有锁，再次 lock 时 `state + 1`，体现可重入；
- unlock 时 `state - 1`，直到减为 0 才真正释放锁并唤醒后继线程。

这也是为什么 `ReentrantLock` 必须 lock 几次就 unlock 几次，否则 `state` 回不到 0，其他线程永远拿不到锁。

#### 共享模式

共享模式常见模板方法是：

- `tryAcquireShared(int arg)`：尝试获取共享资源；
- `tryReleaseShared(int arg)`：尝试释放共享资源。

共享模式不一定只唤醒一个线程，因为释放资源后可能允许多个等待线程继续执行。

以 `Semaphore` 为例：

- 初始化 `permits` 个许可证，本质上就是把 `state` 设为 permits；
- `acquire()` 时 CAS 尝试减少 `state`；
- 如果减少后仍大于等于 0，说明拿到许可证；
- 如果不够，就进入 AQS 队列等待；
- `release()` 时 CAS 增加 `state`，并唤醒等待线程。

---

### ReentrantLock：最典型的 AQS 独占锁

`ReentrantLock` 是理解 AQS 独占模式最好的入口。

它的核心过程可以这样理解：

1. 线程调用 `lock()`；
2. 先尝试 CAS 修改 `state`；
3. 成功则设置当前线程为持有锁线程；
4. 失败则进入 AQS 等待队列；
5. 前驱节点释放锁后，当前线程被唤醒；
6. 当前线程再次尝试获取锁。

#### 可重入如何实现？

可重入的关键是：AQS 维护了锁的持有者以及 `state`。

如果当前线程已经是锁持有者，再次获取锁时不会阻塞，而是把 `state` 加 1。释放锁时再逐次减 1，直到 `state` 变为 0，锁才算真正释放。

#### 公平锁和非公平锁区别

`ReentrantLock` 默认是非公平锁：

```java
new ReentrantLock();       // 非公平锁
new ReentrantLock(true);   // 公平锁
```

公平锁和非公平锁的关键区别在于：**CAS 修改 state 前是否先判断队列中有没有前驱节点**。

公平锁会调用类似 `hasQueuedPredecessors()` 的逻辑：如果前面已经有人排队，当前线程不能插队。

非公平锁则允许新来的线程在锁刚释放时直接 CAS 抢锁，即使队列里已经有等待线程。

| 对比点 | 非公平锁 | 公平锁 |
| --- | --- | --- |
| 是否允许插队 | 允许 | 不允许 |
| 吞吐量 | 通常更高 | 通常更低 |
| 上下文切换 | 较少 | 较多 |
| 饥饿问题 | 极端情况下可能发生 | 基本避免 |
| 适用场景 | 大多数追求吞吐量的场景 | 对顺序、公平性要求高的场景 |

为什么非公平锁通常性能更好？

因为线程从阻塞到被唤醒再到真正运行存在调度成本。非公平锁允许正在运行的新线程直接抢到锁，减少了线程切换；公平锁严格排队，会增加唤醒和调度开销。

---

### Semaphore：控制同时访问资源的线程数量

`Semaphore` 适合用来限制某类资源的并发访问数量，比如单机限流、连接数控制等。

简单使用：

```java
Semaphore semaphore = new Semaphore(5);

semaphore.acquire();
try {
    // 同一时刻最多 5 个线程进入这里
} finally {
    semaphore.release();
}
```

它基于 AQS 的共享模式实现：

- `state` 表示剩余许可证数量；
- 获取许可证就是减少 `state`；
- 释放许可证就是增加 `state`；
- 许可证不足时线程进入 AQS 队列等待。

`Semaphore` 也支持公平和非公平模式：

```java
new Semaphore(5);       // 非公平
new Semaphore(5, true); // 公平
```

不过需要注意：`Semaphore` 更像是“许可证控制器”，它不要求 acquire 和 release 必须由同一个线程执行，这一点和锁不一样。

---

### CountDownLatch：一次性的倒计时器

`CountDownLatch` 用来让一个或多个线程等待其他线程完成任务。

典型场景：主线程等待多个子任务完成后再继续执行。

```java
CountDownLatch latch = new CountDownLatch(3);

for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        try {
            // 执行业务任务
        } finally {
            latch.countDown();
        }
    }).start();
}

latch.await();
// 所有子任务完成后继续执行
```

它也是基于 AQS 的共享模式实现：

- 初始化时把 `state` 设为 count；
- `countDown()` 时 CAS 把 `state` 减 1；
- `await()` 会判断 `state` 是否为 0；
- 如果不为 0，当前线程进入 AQS 队列等待；
- 当 `state` 减到 0，会唤醒等待线程。

`CountDownLatch` 有两个重要特点：

1. **一次性**：计数减到 0 后不能重置；
2. **使用不当可能死锁**：如果某些任务异常退出但没有执行 `countDown()`，等待线程会一直阻塞，所以一般要把 `countDown()` 放在 `finally` 中。

---

### CyclicBarrier：可重复使用的栅栏

`CyclicBarrier` 经常拿来和 `CountDownLatch` 对比。

它的作用是：让一组线程互相等待，直到所有线程都到达屏障点，再一起继续执行。

```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("所有线程都到达屏障，开始下一阶段");
});

for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        try {
            // 第一阶段任务
            barrier.await();
            // 第二阶段任务
        } catch (Exception e) {
            e.printStackTrace();
        }
    }).start();
}
```

需要注意：`CountDownLatch` 是直接基于 AQS 实现的，而 `CyclicBarrier` 本身是基于 `ReentrantLock` 和 `Condition` 实现的；由于 `ReentrantLock` 又基于 AQS，所以它属于间接使用 AQS 的同步工具。

二者区别：

| 对比点 | CountDownLatch | CyclicBarrier |
| --- | --- | --- |
| 是否可复用 | 不可复用 | 可复用 |
| 等待关系 | 一个/多个线程等待其他线程完成 | 一组线程互相等待 |
| 计数变化 | countDown 递减到 0 | await 到达屏障后重置进入下一代 |
| 底层实现 | 直接基于 AQS | 基于 ReentrantLock + Condition |
| 典型场景 | 主线程等待多个任务完成 | 多线程分阶段协作 |

---

### Condition：条件队列和同步队列

如果面试问到 `Condition`，重点是讲清楚它和 AQS 同步队列不是一个东西。

AQS 里有两类队列：

1. **同步队列**：获取锁失败的线程进入这里，等待获取锁；
2. **条件队列**：调用 `Condition.await()` 的线程进入这里，等待某个条件满足。

一个 `ReentrantLock` 可以创建多个 `Condition`：

```java
ReentrantLock lock = new ReentrantLock();
Condition notFull = lock.newCondition();
Condition notEmpty = lock.newCondition();
```

以阻塞队列为例：

- 队列满时，生产者进入 `notFull` 条件队列；
- 队列空时，消费者进入 `notEmpty` 条件队列；
- 当条件满足时，通过 `signal()` 把条件队列中的节点转移到 AQS 同步队列；
- 之后线程还要重新竞争锁，拿到锁后才能继续执行。

面试里容易踩坑的是：**signal 不是让线程立刻运行，而是把线程从条件队列转移到同步队列，后续还要竞争锁**。

---

### 面试常见问法整理

#### 1. AQS 是什么？

AQS 是构建锁和同步器的基础框架，通过 `state` 表示同步状态，通过 FIFO 等待队列管理获取资源失败的线程，并提供独占和共享两种同步模式。

#### 2. AQS 为什么要用 CLH 变体队列？

为了把竞争失败的线程组织起来，避免所有线程一直竞争同一个变量。AQS 在 CLH 思路上做了优化：从纯自旋变成自旋 + 阻塞，从单向队列变成双向队列，方便挂起和唤醒后继线程。

#### 3. state 在不同工具类里分别表示什么？

`ReentrantLock` 中表示锁重入次数；`Semaphore` 中表示许可证数量；`CountDownLatch` 中表示剩余计数；`ReentrantReadWriteLock` 中高低位分别表示读锁数量和写锁重入次数。

#### 4. ReentrantLock 如何实现可重入？

如果当前线程已经持有锁，再次获取时只增加 `state`，不会阻塞；释放时逐次减少 `state`，直到 `state` 为 0 才真正释放锁。

#### 5. 公平锁和非公平锁有什么区别？

公平锁获取锁前会判断队列中是否有前驱节点，有就排队；非公平锁允许新线程直接 CAS 抢锁。非公平锁吞吐量通常更高，但可能导致线程饥饿；公平锁顺序性更好，但上下文切换更多。

#### 6. CountDownLatch 和 CyclicBarrier 有什么区别？

`CountDownLatch` 是一次性的，适合一个线程等待多个任务完成；`CyclicBarrier` 可以复用，适合多个线程分阶段互相等待。`CountDownLatch` 直接基于 AQS，`CyclicBarrier` 基于 `ReentrantLock` 和 `Condition`。

#### 7. Condition.await 和 Object.wait 有什么区别？

`Condition` 必须配合 `Lock` 使用，一个 Lock 可以有多个 Condition 队列；`Object.wait()` 只能配合 synchronized 使用，一个对象只有一个等待队列。`Condition` 更适合实现多个等待条件，例如阻塞队列里的 notFull 和 notEmpty。

---

### 总结

复习 AQS 时，不建议一开始就死磕每一行源码。更好的路径是先把主线搭起来：

1. AQS 是同步器框架，不是具体锁；
2. `state` 表示同步状态，不同工具类语义不同；
3. CLH 变体队列负责管理等待线程；
4. 独占模式对应 `ReentrantLock`，共享模式对应 `Semaphore`、`CountDownLatch`；
5. 公平/非公平、可重入、条件队列这些问题都可以围绕 `state + 队列` 展开。

面试回答 AQS 最重要的是讲出“框架设计思路”，而不是只背 API 名字。只要抓住 `state` 和等待队列这两个核心，再结合几个典型工具类，AQS 这块内容就会清晰很多。
