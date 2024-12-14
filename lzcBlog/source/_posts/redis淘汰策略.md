---
title: redis淘汰策略
date: 2024-10-15 14:20:22
tags: redis
categories: 八股文
---

在 Redis 中，缓存淘汰策略是内置的，用户可以通过配置来选择合适的策略。Redis 提供多种缓存淘汰策略，主要用于内存限制时控制数据的自动过期或删除。当 Redis 内存达到指定的上限时，会根据配置的策略自动淘汰一些数据。

Redis 中的主要缓存淘汰策略有以下几种：

### 1. Redis 支持的淘汰策略

- **noeviction**：当内存不足以容纳新写入数据时，新写入操作会报错。这是 Redis 默认策略，适用于不希望数据被自动淘汰的场景。
- **allkeys-lru**：对所有的键使用 LRU（最近最少使用）算法进行淘汰。
- **volatile-lru**：只对设置了过期时间的键使用 LRU 算法进行淘汰。
- **allkeys-random**：对所有的键随机淘汰。
- **volatile-random**：只对设置了过期时间的键随机淘汰。
- **volatile-ttl**：只对设置了过期时间的键，选择即将过期的键进行淘汰。

#### Redis 4.0 之后新增的策略

- **allkeys-lfu**：对所有键使用 LFU 算法进行淘汰，淘汰最少使用的键。
- **volatile-lfu**：只对设置了过期时间的键使用 LFU 算法进行淘汰。

### 2. 配置 Redis 缓存淘汰策略

Redis 的淘汰策略可以通过修改配置文件 `redis.conf` 或运行时使用命令行配置。

#### 配置方式一：修改 `redis.conf`

找到 Redis 配置文件 `redis.conf`，修改 `maxmemory-policy` 来设置淘汰策略。例如：

```bash
maxmemory-policy allkeys-lru
```

此外，可以设置 Redis 的最大内存使用量，超过该内存时 Redis 将开始执行淘汰策略：

```bash
maxmemory 256mb
```

#### 配置方式二：运行时设置

你可以通过 Redis CLI 动态设置淘汰策略和最大内存使用。执行以下命令：

```bash
CONFIG SET maxmemory 256mb
CONFIG SET maxmemory-policy allkeys-lru
```

### 3. TTL（过期时间）实现

Redis 提供了两种设置过期时间的方法：

- `EXPIRE key seconds`：为键设置一个存活时间（秒），过期后自动删除。
- `SET key value EX seconds`：在设置键的同时指定过期时间（秒）。

#### 示例：

```bash
SET user:1001 "John" EX 60  # 设置 key 为 user:1001，60秒后过期
EXPIRE user:1002 120         # 为现有 key user:1002 设置120秒的过期时间
```

### 4. Redis LRU 实现机制

Redis 的 LRU 并不是严格精确的 LRU，它使用了近似算法。默认情况下，Redis 通过采样的方式（默认采样 5 个键）来判断最近最少使用的键进行淘汰。你可以通过调整 `maxmemory-samples` 参数来控制采样数，采样数越大，LRU 越准确，但性能也会受到一定影响。

#### 示例：

在配置文件中设置采样数：

```bash
maxmemory-samples 10
```

运行时设置：

```bash
CONFIG SET maxmemory-samples 10
```

### 5. Redis 缓存淘汰策略实战

假设我们有一个 Redis 实例，用来缓存用户的会话信息，并且只想缓存最新活跃的会话用户，同时确保 Redis 的内存不会超过 128MB，并使用 LRU 算法淘汰不活跃的会话：

1. **设置 Redis 的最大内存和 LRU 策略**：
   
   ```bash
   CONFIG SET maxmemory 128mb
   CONFIG SET maxmemory-policy allkeys-lru
   ```

2. **向 Redis 写入会话数据，并设置过期时间**：
   
   ```bash
   SET session:user:1001 "session_data" EX 3600  # 会话信息1小时过期
   SET session:user:1002 "session_data" EX 3600
   ```

3. **监控淘汰情况**：
    当 Redis 内存达到 128MB 时，Redis 将根据 LRU 算法自动淘汰最近最少使用的会话数据，以腾出空间。

### 总结

在 Redis 中，缓存淘汰策略已经内置并且可以通过简单配置来启用。在选择合适的策略时，应该结合业务场景，例如如果业务中所有键都有过期时间，可以选择 `volatile-lru` 或 `volatile-ttl`，而如果没有过期时间且想控制整体缓存大小，可以选择 `allkeys-lru` 或 `allkeys-random`。

Redis 中的主要缓存淘汰策略有以下几种：

1. Redis 支持的淘汰策略
   noeviction：当内存不足以容纳新写入数据时，新写入操作会报错。这是 Redis 默认策略，适用于不希望数据被自动淘汰的场景。
   allkeys-lru：对所有的键使用 LRU（最近最少使用）算法进行淘汰。
   volatile-lru：只对设置了过期时间的键使用 LRU 算法进行淘汰。
   allkeys-random：对所有的键随机淘汰。
   volatile-random：只对设置了过期时间的键随机淘汰。
   volatile-ttl：只对设置了过期时间的键，选择即将过期的键进行淘汰。
   Redis 4.0 之后新增的策略
   allkeys-lfu：对所有键使用 LFU 算法进行淘汰，淘汰最少使用的键。
   volatile-lfu：只对设置了过期时间的键使用 LFU 算法进行淘汰。
2. 配置 Redis 缓存淘汰策略
   Redis 的淘汰策略可以通过修改配置文件 redis.conf 或运行时使用命令行配置。

配置方式一：修改 redis.conf
找到 Redis 配置文件 redis.conf，修改 maxmemory-policy 来设置淘汰策略。例如：

maxmemory-policy allkeys-lru
1
此外，可以设置 Redis 的最大内存使用量，超过该内存时 Redis 将开始执行淘汰策略：

maxmemory 256mb
1
配置方式二：运行时设置
你可以通过 Redis CLI 动态设置淘汰策略和最大内存使用。执行以下命令：

CONFIG SET maxmemory 256mb
CONFIG SET maxmemory-policy allkeys-lru
1
2
3. TTL（过期时间）实现
Redis 提供了两种设置过期时间的方法：

EXPIRE key seconds：为键设置一个存活时间（秒），过期后自动删除。
SET key value EX seconds：在设置键的同时指定过期时间（秒）。
示例：
SET user:1001 "John" EX 60  # 设置 key 为 user:1001，60秒后过期
EXPIRE user:1002 120         # 为现有 key user:1002 设置120秒的过期时间
1
2
4. Redis LRU 实现机制
Redis 的 LRU 并不是严格精确的 LRU，它使用了近似算法。默认情况下，Redis 通过采样的方式（默认采样 5 个键）来判断最近最少使用的键进行淘汰。你可以通过调整 maxmemory-samples 参数来控制采样数，采样数越大，LRU 越准确，但性能也会受到一定影响。

示例：
在配置文件中设置采样数：

maxmemory-samples 10
1
运行时设置：

CONFIG SET maxmemory-samples 10
1
5. Redis 缓存淘汰策略实战
假设我们有一个 Redis 实例，用来缓存用户的会话信息，并且只想缓存最新活跃的会话用户，同时确保 Redis 的内存不会超过 128MB，并使用 LRU 算法淘汰不活跃的会话：

设置 Redis 的最大内存和 LRU 策略：

CONFIG SET maxmemory 128mb
CONFIG SET maxmemory-policy allkeys-lru
1
2
向 Redis 写入会话数据，并设置过期时间：

SET session:user:1001 "session_data" EX 3600  # 会话信息1小时过期
SET session:user:1002 "session_data" EX 3600
1
2
监控淘汰情况：
当 Redis 内存达到 128MB 时，Redis 将根据 LRU 算法自动淘汰最近最少使用的会话数据，以腾出空间。

总结
在 Redis 中，缓存淘汰策略已经内置并且可以通过简单配置来启用。在选择合适的策略时，应该结合业务场景，例如如果业务中所有键都有过期时间，可以选择 volatile-lru 或 volatile-ttl，而如果没有过期时间且想控制整体缓存大小，可以选择 allkeys-lru 或 allkeys-random。
