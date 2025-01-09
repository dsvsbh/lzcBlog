---
title: localCache实现expire过期原理
date: 2024-12-17 11:52:42
tags: localCache
categories: 项目
---

我们首先来定义一下接口。

主要有两个：一个是多久之后过期，一个是在什么时候过期。

```java
public interface ICache<K, V> extends Map<K, V> {

    /**
     * 设置过期时间
     * （1）如果 key 不存在，则什么都不做。
     * （2）暂时不提供新建 key 指定过期时间的方式，会破坏原来的方法。
     *
     * 会做什么：
     * 类似于 redis
     * （1）惰性删除。
     * 在执行下面的方法时，如果过期则进行删除。
     * {@link ICache#get(Object)} 获取
     * {@link ICache#values()} 获取所有值
     * {@link ICache#entrySet()} 获取所有明细
     *
     * 【数据的不一致性】
     * 调用其他方法，可能得到的不是使用者的预期结果，因为此时的 expire 信息可能没有被及时更新。
     * 比如
     * {@link ICache#isEmpty()} 是否为空
     * {@link ICache#size()} 当前大小
     * 同时会导致以 size() 作为过期条件的问题。
     *
     * 解决方案：考虑添加 refresh 等方法，暂时不做一致性的考虑。
     * 对于实际的使用，我们更关心 K/V 的信息。
     *
     * （2）定时删除
     * 启动一个定时任务。每次随机选择指定大小的 key 进行是否过期判断。
     * 类似于 redis，为了简化，可以考虑设定超时时间，频率与超时时间成反比。
     *
     * 其他拓展性考虑：
     * 后期考虑提供原子性操作，保证事务性。暂时不做考虑。
     * 此处默认使用 TTL 作为比较的基准，暂时不想支持 LastAccessTime 的淘汰策略。会增加复杂度。
     * 如果增加 lastAccessTime 过期，本方法可以不做修改。
     *
     * @param key         key
     * @param timeInMills 毫秒时间之后过期
     * @return this
     * @since 0.0.3
     */
    ICache<K, V> expire(final K key, final long timeInMills);

    /**
     * 在指定的时间过期
     * @param key key
     * @param timeInMills 时间戳
     * @return this
     * @since 0.0.3
     */
    ICache<K, V> expireAt(final K key, final long timeInMills);

}
```

为了便于处理，我们将多久之后过期，进行计算。将两个问题变成同一个问题，在什么时候过期的问题。

核心的代码，主要还是看 cacheExpire 接口。

```java
@Override
public ICache<K, V> expire(K key, long timeInMills) {
    long expireTime = System.currentTimeMillis() + timeInMills;
    return this.expireAt(key, expireTime);
}

@Override
public ICache<K, V> expireAt(K key, long timeInMills) {
    this.cacheExpire.expire(key, timeInMills);
    return this;
}
```

在CacheBs构建cache对象的时候会初始化cache指定expire过期策略

```java
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
```

```java
    /**
     * 初始化
     * @since 0.0.7
     */
    public void init() {
        this.expire = new CacheExpire<>(this);
        this.load.load(this);

        // 初始化持久化
        if(this.persist != null) {
            new InnerCachePersist<>(this, persist);
        }
    }
```

cache使用的是CacheExpire过期策略并将cache对象传入策略中实现过期删除，当调用expire方法时，key和过期时间会存入到expireMap，在实例化CacheExpire对象时初始化会启动一个单线程定时轮询执行器singleThreadScheduledExecutor，每过100ms调度一次，每次去expireMap中处理100个可能过期的key实现过期删除

```java
package com.github.houbb.cache.core.support.expire;

import com.github.houbb.cache.api.ICache;
import com.github.houbb.cache.api.ICacheExpire;
import com.github.houbb.cache.api.ICacheRemoveListener;
import com.github.houbb.cache.api.ICacheRemoveListenerContext;
import com.github.houbb.cache.core.constant.enums.CacheRemoveType;
import com.github.houbb.cache.core.support.listener.remove.CacheRemoveListenerContext;
import com.github.houbb.heaven.util.util.CollectionUtil;
import com.github.houbb.heaven.util.util.MapUtil;

import java.util.*;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

/**
 * 缓存过期-普通策略
 *
 * @author binbin.hou
 * @since 0.0.3
 * @param <K> key
 * @param <V> value
 */
public class CacheExpire<K,V> implements ICacheExpire<K,V> {

    /**
     * 单次清空的数量限制
     * @since 0.0.3
     */
    private static final int LIMIT = 100;

    /**
     * 过期 map
     *
     * 空间换时间
     * @since 0.0.3
     */
    private final Map<K, Long> expireMap = new HashMap<>();

    /**
     * 缓存实现
     * @since 0.0.3
     */
    private final ICache<K,V> cache;

    /**
     * 线程执行类
     * @since 0.0.3
     */
    private static final ScheduledExecutorService EXECUTOR_SERVICE = Executors.newSingleThreadScheduledExecutor();

    public CacheExpire(ICache<K, V> cache) {
        this.cache = cache;
        this.init();
    }

    /**
     * 初始化任务
     * @since 0.0.3
     */
    private void init() {
        EXECUTOR_SERVICE.scheduleAtFixedRate(new ExpireThread(), 100, 100, TimeUnit.MILLISECONDS);
    }

    /**
     * 定时执行任务
     * @since 0.0.3
     */
    private class ExpireThread implements Runnable {
        @Override
        public void run() {
            //1.判断是否为空
            if(MapUtil.isEmpty(expireMap)) {
                return;
            }

            //2. 获取 key 进行处理
            int count = 0;
            for(Map.Entry<K, Long> entry : expireMap.entrySet()) {
                if(count >= LIMIT) {
                    return;
                }

                expireKey(entry.getKey(), entry.getValue());
                count++;
            }
        }
    }

    @Override
    public void expire(K key, long expireAt) {
        expireMap.put(key, expireAt);
    }

    @Override
    public void refreshExpire(Collection<K> keyList) {
        if(CollectionUtil.isEmpty(keyList)) {
            return;
        }

        // 判断大小，小的作为外循环。一般都是过期的 keys 比较小。
        if(keyList.size() <= expireMap.size()) {
            for(K key : keyList) {
                Long expireAt = expireMap.get(key);
                expireKey(key, expireAt);
            }
        } else {
            for(Map.Entry<K, Long> entry : expireMap.entrySet()) {
                this.expireKey(entry.getKey(), entry.getValue());
            }
        }
    }

    @Override
    public Long expireTime(K key) {
        return expireMap.get(key);
    }

    /**
     * 过期处理 key
     * @param key key
     * @param expireAt 过期时间
     * @since 0.0.3
     */
    private void expireKey(final K key, final Long expireAt) {
        if(expireAt == null) {
            return;
        }

        long currentTime = System.currentTimeMillis();
        if(currentTime >= expireAt) {
            expireMap.remove(key);
            // 再移除缓存，后续可以通过惰性删除做补偿
            V removeValue = cache.remove(key);
            // 执行淘汰监听器
            ICacheRemoveListenerContext<K,V> removeListenerContext = CacheRemoveListenerContext.<K,V>newInstance().key(key).value(removeValue).type(CacheRemoveType.EXPIRE.code());
            for(ICacheRemoveListener<K,V> listener : cache.removeListeners()) {
                listener.listen(removeListenerContext);
            }
        }
    }

}

```



类似于 redis，我们采用定时删除的方案，就有一个问题：可能数据清理的不及时。

那当我们查询时，可能获取到到是脏数据。

于是就有一些人就想了，当我们关心某些数据时，才对数据做对应的删除判断操作，这样压力会小很多。

算是一种折中方案。

一般就是各种查询方法，比如我们获取 key 对应的值时

```java
    @Override
    @CacheInterceptor(evict = true)
    @SuppressWarnings("unchecked")
    public V get(Object key) {
        //1. 刷新所有过期信息
        K genericKey = (K) key;
        this.expire.refreshExpire(Collections.singletonList(genericKey));

        return map.get(key);
    }
```

我们在获取之前，先做一次数据的刷新。



实现原理也非常简单，就是一个循环，然后作删除即可。

这里加了一个小的优化：选择数量少的作为外循环。

循环集合的时间复杂度是 O(n), map.get() 的时间复杂度是 O(1);

```java

```
