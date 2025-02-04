---
title: redisson的分布式锁实现
date: 2025-01-14 16:02:21
tags: redis
categories: 底层实现
---

Redisson 分布式锁通过 Redis 的setnx和 Lua 脚本实现，主要特点如下：

#### **加锁**

- **使用 `SETNX` 命令加锁**：
  
  - `SET key value NX PX timeout`
  - `NX` 确保只有当锁不存在时才创建。
  - `PX timeout` 设置锁的过期时间，避免死锁。
  - `value` 通常是一个唯一标识（如 UUID），标记当前客户端（线程）持有锁。

- **续锁机制**：
  
  - Redisson 内置了锁续期机制（**Watchdog**），会定期检查锁是否仍然持有。
  - 默认锁过期时间为 30 秒，但 Watchdog 会每隔 10 秒（默认值）续期，保持锁的有效性，直到客户端主动解锁或宕机。

#### **解锁**

- 使用 Lua 脚本保证解锁的原子性：
  
  - 检查锁的值是否与当前客户端（线程）的唯一标识匹配，匹配则删除锁。
  
  - 示例脚本：
    
    lua
    
    复制代码
    
    `if redis.call("GET", KEYS[1]) == ARGV[1] then     return redis.call("DEL", KEYS[1]) else     return 0 end`
