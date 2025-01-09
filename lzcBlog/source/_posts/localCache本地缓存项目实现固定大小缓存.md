---
title: localCache本地缓存项目实现固定大小缓存
date: 2024-12-15 22:37:42
tags: localCache
categories: 项目
---

 为了便于用户使用，我们实现类似于 guava 的引导类。

所有参数都提供默认值，使用 fluent 流式写法，提升用户体验。

```java
package com.github.houbb.cache.core.bs;

import com.github.houbb.cache.api.*;
import com.github.houbb.cache.core.core.Cache;
import com.github.houbb.cache.core.support.evict.CacheEvicts;
import com.github.houbb.cache.core.support.listener.remove.CacheRemoveListeners;
import com.github.houbb.cache.core.support.listener.slow.CacheSlowListeners;
import com.github.houbb.cache.core.support.load.CacheLoads;
import com.github.houbb.cache.core.support.persist.CachePersists;
import com.github.houbb.cache.core.support.proxy.CacheProxy;
import com.github.houbb.heaven.util.common.ArgUtil;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * 缓存引导类
 * @author binbin.hou
 * @since 0.0.2
 */
public final class CacheBs<K,V> {

    private CacheBs(){}

    /**
     * 创建对象实例
     * @param <K> key
     * @param <V> value
     * @return this
     * @since 0.0.2
     */
    public static <K,V> CacheBs<K,V> newInstance() {
        return new CacheBs<>();
    }

    /**
     * map 实现
     * @since 0.0.2
     */
    private Map<K,V> map = new HashMap<>();

    /**
     * 大小限制
     * @since 0.0.2
     */
    private int size = Integer.MAX_VALUE;

    /**
     * 驱除策略
     * @since 0.0.2
     */
    private ICacheEvict<K,V> evict = CacheEvicts.fifo();

    /**
     * 删除监听类
     * @since 0.0.6
     */
    private final List<ICacheRemoveListener<K,V>> removeListeners = CacheRemoveListeners.defaults();

    /**
     * 慢操作监听类
     * @since 0.0.9
     */
    private final List<ICacheSlowListener> slowListeners = CacheSlowListeners.none();

    /**
     * 加载策略
     * @since 0.0.7
     */
    private ICacheLoad<K,V> load = CacheLoads.none();

    /**
     * 持久化实现策略
     * @since 0.0.8
     */
    private ICachePersist<K,V> persist = CachePersists.none();

    /**
     * map 实现
     * @param map map
     * @return this
     * @since 0.0.2
     */
    public CacheBs<K, V> map(Map<K, V> map) {
        ArgUtil.notNull(map, "map");

        this.map = map;
        return this;
    }

    /**
     * 设置 size 信息
     * @param size size
     * @return this
     * @since 0.0.2
     */
    public CacheBs<K, V> size(int size) {
        ArgUtil.notNegative(size, "size");

        this.size = size;
        return this;
    }

    /**
     * 设置驱除策略
     * @param evict 驱除策略
     * @return this
     * @since 0.0.2
     */
    public CacheBs<K, V> evict(ICacheEvict<K, V> evict) {
        ArgUtil.notNull(evict, "evict");

        this.evict = evict;
        return this;
    }

    /**
     * 设置加载
     * @param load 加载
     * @return this
     * @since 0.0.7
     */
    public CacheBs<K, V> load(ICacheLoad<K, V> load) {
        ArgUtil.notNull(load, "load");

        this.load = load;
        return this;
    }

    /**
     * 添加删除监听器
     * @param removeListener 监听器
     * @return this
     * @since 0.0.6
     */
    public CacheBs<K, V> addRemoveListener(ICacheRemoveListener<K,V> removeListener) {
        ArgUtil.notNull(removeListener, "removeListener");

        this.removeListeners.add(removeListener);
        return this;
    }

    /**
     * 添加慢日志监听器
     * @param slowListener 监听器
     * @return this
     * @since 0.0.9
     */
    public CacheBs<K, V> addSlowListener(ICacheSlowListener slowListener) {
        ArgUtil.notNull(slowListener, "slowListener");

        this.slowListeners.add(slowListener);
        return this;
    }

    /**
     * 设置持久化策略
     * @param persist 持久化
     * @return this
     * @since 0.0.8
     */
    public CacheBs<K, V> persist(ICachePersist<K, V> persist) {
        this.persist = persist;
        return this;
    }

    /**
     * 构建缓存信息
     * @return 缓存信息
     * @since 0.0.2
     */
    public ICache<K,V> build() {
        Cache<K,V> cache = new Cache<>();
        cache.map(map);
        cache.evict(evict);
        cache.sizeLimit(size);
        cache.removeListeners(removeListeners);
        cache.load(load);
        cache.persist(persist);
        cache.slowListeners(slowListeners);

        // 初始化
        cache.init();
        return CacheProxy.getProxy(cache);
    }

}

```



使用CacheBs类构建Cache对象，并返回Cache对象的代理对象

为了兼容 Map，我们定义缓存接口继承自 Map 接口。

```java
/**
 * 缓存接口
 * @author binbin.hou
 * @since 0.0.1
 */
public interface ICache<K, V> extends Map<K, V> {
}

```



由于缓存是设置了固定大小的，所以在存入数据的时候要先尝试进行缓存淘汰，再存入数据

```java
@Override
public V put(K key, V value) {
    //1.1 尝试驱除
    CacheEvictContext<K,V> context = new CacheEvictContext<>();
    context.key(key).size(sizeLimit).cache(this);
    cacheEvict.evict(context);
    //2. 判断驱除后的信息
    if(isSizeLimit()) {
        throw new CacheRuntimeException("当前队列已满，数据添加失败！");
    }
    //3. 执行添加
    return map.put(key, value);
}e.add(key);
    }

}
```



在CacheBs构建Cache的时候可以指定淘汰策略

```java
    /**
     * 设置驱除策略
     * @param evict 驱除策略
     * @return this
     * @since 0.0.2
     */
    public CacheBs<K, V> evict(ICacheEvict<K, V> evict) {
        ArgUtil.notNull(evict, "evict");

        this.evict = evict;
        return this;
    }
```



默认使用fifo先进先出的策略

```java
    /**
     * 驱除策略
     * @since 0.0.2
     */
    private ICacheEvict<K,V> evict = CacheEvicts.fifo();
```



fifo是先进先出，使用LinkedList做队列，当缓存满了淘汰队头元素，然后将新元素放入队尾，并淘汰掉缓存中的数据

```java
package com.github.houbb.cache.core.support.evict;

import com.github.houbb.cache.api.ICache;
import com.github.houbb.cache.api.ICacheEvictContext;
import com.github.houbb.cache.core.model.CacheEntry;

import java.util.LinkedList;
import java.util.Queue;

/**
 * 丢弃策略-先进先出
 * @author binbin.hou
 * @since 0.0.2
 */
public class CacheEvictFifo<K,V> extends AbstractCacheEvict<K,V> {

    /**
     * queue 信息
     * @since 0.0.2
     */
    private final Queue<K> queue = new LinkedList<>();

    @Override
    public CacheEntry<K,V> doEvict(ICacheEvictContext<K, V> context) {
        CacheEntry<K,V> result = null;

        final ICache<K,V> cache = context.cache();
        // 超过限制，执行移除
        if(cache.size() >= context.size()) {
            K evictKey = queue.remove();
            // 移除最开始的元素
            V evictValue = cache.remove(evictKey);
            result = new CacheEntry<>(evictKey, evictValue);
        }

        // 将新加的元素放入队尾
        final K key = context.key();
        queue.add(key);

        return result;
    }

}

```
