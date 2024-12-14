---
title: java中Atomic原子类型
date: 2024-10-15 14:19:12
tags: java基础
categories: 八股文
---

​

# Java中atomic原子类型

原子类型是 Java 中用于支持多线程并发编程的类型，属于 `java.util.concurrent.atomic` 包。原子类型提供了一种在多线程环境中执行线程安全操作的机制，它们通过底层硬件指令来实现原子操作（不可中断）。

## 常见的原子类型：

- `AtomicBoolean`

- `AtomicInteger`

- `AtomicLong`

- `AtomicReference<T>`

## 原子类型的特性：

- 支持无锁的线程安全操作，避免使用 `synchronized` 或显式锁。
- 提供了常见的原子操作，如 `get()`、`set()`、`compareAndSet()`、`incrementAndGet()` 等。

## 以AtomicInteger为例，解析原子类型的特性：

一些常见的 `AtomicInteger` 方法包括：

1. `get()`：获取当前值。
2. `set(int newValue)`：设置为指定值。
3. `getAndIncrement()`：先获取当前值，然后递增。
4. `incrementAndGet()`：先递增，然后获取当前值。
5. `compareAndSet(int expect, int update)`：如果当前值等于 `expect`，则将值设置为 `update`。

参考源码可知：

通过CAS无锁算法即Unsafe类中的compareAndSetInt方法实现原子性，保证数据修改时的线程安全。

示例代码：

```
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicIntegerExample {
    public static void main(String[] args) {
        AtomicInteger atomicInteger = new AtomicInteger(0);

        // 递增并获取值
        System.out.println(atomicInteger.incrementAndGet()); // 输出 1

        // 获取并递增值
        System.out.println(atomicInteger.getAndIncrement()); // 输出 1, 但值已变为 2

        // 比较并设置值
        boolean result = atomicInteger.compareAndSet(2, 5);
        System.out.println(result); // 输出 true，因为当前值是 2
        System.out.println(atomicInteger.get()); // 输出 5
    }
}
```

![](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw== "点击并拖拽以移动")

通过 `AtomicInteger`，你可以避免使用 `synchronized` 关键字或手动管理锁，简化了多线程操作。

​
