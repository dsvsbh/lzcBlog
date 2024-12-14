---
title: 工厂+注册表模式设计nodeHandler
date: 2024-11-14 21:09:17
tags: 设计模式
categories: 业务场景
---

# 工厂+注册表模式获取nodeHandler

自定义抽取模板有三种不同节点，因此有三种不同节点处理器nodeHandler

其中一个是默认nodeHandler.

由于三种节点的数据结构和处理方法完全不同，而需要走统一入口操作节点，因此使用工厂模式，前端操作节点时传入一个nodeCode,工厂根据nodeCode获取对应的nodeHandler处理节点

```java
package ai.ii.supertext.app.serivce.extraction.handler;

import ai.ii.supertext.plus.common.util.SpringContext;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.ConcurrentHashMap;

@Slf4j
@Configuration
public class NodeHandlerFactory {

    private static final ConcurrentHashMap<String, NodeHandler> nodeHandlerMap = new ConcurrentHashMap<>();

    public static void registerHandler(NodeHandler handler) {
        nodeHandlerMap.put(handler.getNodeCode(), handler);
    }

    public NodeHandler getNodeHandler(String nodeCode) {
        if (nodeCode == null) {
            log.error("action=getNodeHandler node is null. please confirm config");
            return null;
        }
        return nodeHandlerMap.getOrDefault(nodeCode, SpringContext.get(DefaultNodeHandler.class));
    }
}
```

工厂通过维护一个ConcurrentHashMap,提供对象注册方法和获取方法，由于在NodeHandler接口中有个PostConstruct方法，在NodeHandler实现类在bean的初始化之后会执行方法将NodeHandler注册到工厂中，nodeCode为key，nodeHandler为value。
因此只需用@Component标记实现类，应用启动时就会把对象注册到工厂

工厂通过传入的nodeCode获取到对应的nodeHandler处理对应节点，如果工厂找不到nodeHandler会返回默认的nodeHandler.

使用工厂模式管理节点处理器，增强了可维护性和功能扩展性，而且不同的节点操作可以走统一接口，简化了代码，降低维护成本
