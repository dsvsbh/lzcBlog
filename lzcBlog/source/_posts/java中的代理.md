---
title: java中的代理
date: 2024-12-14 11:43:38
tags: java基础
categories: 八股文
---

在 Java 中，代理（Proxy）是一种设计模式，它允许你通过代理对象来间接访问目标对象。代理对象可以在不改变目标对象代码的情况下，对目标对象进行增强（例如添加日志、权限控制、事务管理等）。Java 提供了两种常见的代理方式：**静态代理**和**动态代理**。

### 1. **静态代理**

静态代理通过手动创建代理类来实现。代理类通常与目标类实现相同的接口，并在代理类中调用目标类的实际方法。

#### 示例：

```java
// 目标类
public class RealSubject implements Subject {
    @Override
    public void request() {
        System.out.println("RealSubject request.");
    }
}

// 代理类
public class ProxySubject implements Subject {
    private RealSubject realSubject;

    public ProxySubject(RealSubject realSubject) {
        this.realSubject = realSubject;
    }

    @Override
    public void request() {
        System.out.println("ProxySubject before request.");
        realSubject.request();
        System.out.println("ProxySubject after request.");
    }
}

// 接口
public interface Subject {
    void request();
}

// 使用代理
public class Main {
    public static void main(String[] args) {
        Subject subject = new ProxySubject(new RealSubject());
        subject.request();
    }
}
```

输出：

```
ProxySubject before request.
RealSubject request.
ProxySubject after request.
```

### 2. **动态代理**

动态代理是 Java 的一个特性，允许你在运行时创建代理对象，而不需要明确地创建代理类。Java 提供了 `java.lang.reflect.Proxy` 类来实现动态代理。动态代理一般用于增强类或接口的功能，常见的场景有 AOP（面向切面编程）和事务管理等。

Java 动态代理需要以下几个组成部分：

- **接口**：被代理的类或接口。
- **InvocationHandler**：代理类的处理器，实现了 `invoke` 方法，负责代理行为。

#### 示例：

```java
import java.lang.reflect.*;

interface Subject {
    void request();
}

class RealSubject implements Subject {
    @Override
    public void request() {
        System.out.println("RealSubject request.");
    }
}

class ProxyHandler implements InvocationHandler {
    private Object target;

    public ProxyHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Before invoking " + method.getName());
        Object result = method.invoke(target, args);  // 调用真实方法
        System.out.println("After invoking " + method.getName());
        return result;
    }
}

public class Main {
    public static void main(String[] args) {
        RealSubject realSubject = new RealSubject();
        Subject proxySubject = (Subject) Proxy.newProxyInstance(
            RealSubject.class.getClassLoader(),
            new Class[]{Subject.class},
            new ProxyHandler(realSubject)
        );

        proxySubject.request();
    }
}
```

输出：

```
Before invoking request
RealSubject request.
After invoking request
```

### 3. **CGLIB 代理**

除了 JDK 提供的动态代理外，CGLIB（Code Generation Library）是一种基于继承的代理方式。它不需要目标类实现接口，而是通过继承目标类来生成代理类。CGLIB 通常用于目标类没有实现接口的情况。Spring AOP 就是使用 CGLIB 实现代理的。

### 4. **代理的应用场景**

- **日志记录**：在方法调用前后记录日志。
- **性能监控**：在方法调用前后记录时间。
- **权限控制**：在方法调用前检查用户权限。
- **缓存**：使用代理实现方法的缓存。
- **事务管理**：在方法调用前开启事务，调用后提交或回滚事务。

### 总结

Java 中的代理主要分为静态代理和动态代理，静态代理通过手动编写代理类实现，而动态代理通过反射机制动态生成代理类。动态代理通常用于需要增强功能的场景，如 AOP、权限控制、事务管理等。
