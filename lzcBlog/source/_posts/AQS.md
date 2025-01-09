---
title: AQS
date: 2024-12-24 22:29:34
tags: juc
categories: 八股文
---

**AQS（AbstractQueuedSynchronizer）** 是 Java 并发包（`java.util.concurrent`）中的核心类之一，作为构建锁和同步器的基础。很多 JUC 中的同步工具，如 `ReentrantLock`、`Semaphore`、`CountDownLatch`、`ReadWriteLock` 等，都是基于 AQS 实现的。AQS 通过维护一个**FIFO（先进先出）的等待队列**，来管理多个线程之间的同步操作。

AQS 主要通过内部的**状态（state）**和**队列**来控制锁的获取和释放，线程的阻塞和唤醒。

### 1. **AQS 的基本结构**

AQS 的核心由以下几个部分组成：

- **同步状态（state）**：一个 `int` 类型的变量，用于表示同步状态。可以由子类实现锁的状态，如 `ReentrantLock` 使用 `state == 0` 表示锁未被持有，`state == 1` 表示锁被持有。
- **等待队列**：AQS 内部维护一个 **FIFO 队列**（双向链表），所有无法获得锁的线程都会被加入到这个队列中，并且线程会被挂起，直到锁可用时被唤醒。
- **Node 节点**：队列中的每个节点（`Node`）代表一个等待锁的线程，Node 包含了线程的状态、前驱和后继节点的引用。

### 2. **AQS 的工作原理**

AQS 主要通过以下几种机制来实现线程的同步控制：

#### 1. **独占模式（Exclusive）**

在独占模式下，只有一个线程可以获取到锁，其他线程需要等待。独占模式是 `ReentrantLock` 使用的模式。

**工作流程**：

- 当线程尝试获取锁时，AQS 调用 `acquire()` 方法。
- 如果锁被占用，线程会被加入到等待队列的尾部，并进入阻塞状态（通过调用 `LockSupport.park()` 挂起线程）。
- 当持有锁的线程释放锁时，AQS 会唤醒等待队列中的第一个线程（通过 `LockSupport.unpark()` 唤醒线程）。

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

`acquire()` 方法尝试获取锁，如果获取失败，线程将进入队列等待。`addWaiter(Node.EXCLUSIVE)` 方法会将线程以独占模式加入等待队列。

#### 2. **共享模式（Shared）**

在共享模式下，多个线程可以同时获得锁，这种模式适用于 `Semaphore`、`CountDownLatch` 等允许多个线程共享资源的场景。

**工作流程**：

- 在共享模式下，多个线程可以同时获取锁。
- AQS 中的 `acquireShared()` 方法会尝试获取共享锁，如果成功，多个线程会被唤醒并共享资源。

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

### 3. **等待队列的管理**

AQS 内部通过一个双向链表来维护阻塞线程的等待队列。等待队列的节点是 `Node`，每个 `Node` 包含了一个线程的引用和线程的状态：

- **Node 的状态**：
  - `CANCELLED`：表示线程等待超时或被中断，该节点被取消。
  - `SIGNAL`：表示当前节点的线程阻塞，需要被唤醒。
  - `CONDITION`：表示线程等待在某个 `Condition` 上。
  - `PROPAGATE`：用于共享模式，表示后续节点需要被唤醒。

当某个线程释放锁时，AQS 会从队列中唤醒下一个等待线程，唤醒时调用 `LockSupport.unpark(thread)`。

### 4. **锁的实现**

#### **独占模式下的 ReentrantLock**

`ReentrantLock` 是 AQS 的典型实现之一。在独占模式下，锁的获取和释放由 AQS 提供的方法来管理。

- **获取锁**：`ReentrantLock.lock()` 方法调用 AQS 的 `acquire()`，该方法首先尝试通过 `tryAcquire()` 获取锁，如果失败，则将当前线程加入等待队列并进入阻塞状态。

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    } else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        setState(nextc);
        return true;
    }
    return false;
}
```

- **释放锁**：`ReentrantLock.unlock()` 方法调用 AQS 的 `release()`，该方法通过 `tryRelease()` 判断是否能够释放锁，并唤醒等待队列中的下一个线程。

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

### 5. **Condition 的实现**

AQS 还为每个锁提供了 `Condition` 支持，允许线程等待某个条件的发生。当线程调用 `Condition.await()` 时，线程会被放入**条件队列**，并在条件满足时由 `Condition.signal()` 唤醒。

- **await()**：线程调用 `await()` 后会释放锁并进入条件队列，挂起当前线程。
- **signal()**：唤醒等待队列中的一个线程，重新竞争锁。

```java
public final void await() throws InterruptedException {
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
    }
    acquireQueued(node, savedState);
}
```

### 6. **AQS 的优点**

- **简化并发工具的开发**：通过 AQS，开发者可以方便地实现复杂的同步工具（如自定义锁、信号量等），无需重复实现底层的同步机制。
- **可扩展性**：AQS 提供了丰富的扩展点，允许开发者自定义锁的行为（独占或共享模式）。
- **高效的线程管理**：AQS 通过队列管理等待线程，采用 `park()` 和 `unpark()` 提供了高效的线程阻塞与唤醒机制。

### 总结

AQS 是 Java 并发框架中实现锁、信号量等同步工具的基础框架，通过维护同步状态、等待队列和提供独占与共享两种模式，实现了对线程的阻塞和唤醒控制。它极大简化了同步器的实现，使得 `ReentrantLock`、`Semaphore` 等工具类变得更加灵活和高效。
