---
title: Redis HyperLogLog基数统计
date: 2018/3/15 20:46:25
category:
- redis
tag:
- redis
- HyperLogLog
comments: true  
---
# Redis HyperLogLog

Redis 在 2.8.9 版本添加了 HyperLogLog 结构。

Redis HyperLogLog 是用来做基数统计的算法，HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定的、并且是很小的。

在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基 数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。

但是，因为 HyperLogLog 只会根据输入元素来计算基数，而不会储存输入元素本身，所以 HyperLogLog 不能像集合那样，返回输入的各个元素。



## 什么是基数?

比如数据集 {1, 3, 5, 7, 5, 7, 8}， 那么这个数据集的基数集为 {1, 3, 5 ,7, 8}, 基数(不重复元素)为5。 基数估计就是在误差可接受的范围内，快速计算基数。

## 实例

以下实例演示了 HyperLogLog 的工作过程：

```
redis 127.0.0.1:6379> PFADD runoobkey "redis"

1) (integer) 1

redis 127.0.0.1:6379> PFADD runoobkey "mongodb"

1) (integer) 1

redis 127.0.0.1:6379> PFADD runoobkey "mysql"

1) (integer) 1

redis 127.0.0.1:6379> PFCOUNT runoobkey

(integer) 3
```

------

## Redis HyperLogLog 命令

下表列出了 redis HyperLogLog 的基本命令：

| 序号   | 命令及描述                                    |
| ---- | ---------------------------------------- |
| 1    | [PFADD key element [element ...\]](http://www.runoob.com/redis/hyperloglog-pfadd.html) 添加指定元素到 HyperLogLog 中。 |
| 2    | [PFCOUNT key [key ...\]](http://www.runoob.com/redis/hyperloglog-pfcount.html) 返回给定 HyperLogLog 的基数估算值。 |
| 3    | [PFMERGE destkey sourcekey [sourcekey ...\]](http://www.runoob.com/redis/hyperloglog-pfmerge.html) 将多个 HyperLogLog 合并为一个 HyperLogLog |

# Redis 发布订阅

Redis 发布订阅(pub/sub)是一种消息通信模式：发送者(pub)发送消息，订阅者(sub)接收消息。

Redis 客户端可以订阅任意数量的频道。

下图展示了频道 channel1 ， 以及订阅这个频道的三个客户端 —— client2 、 client5 和 client1 之间的关系：

当有新消息通过 PUBLISH 命令发送给频道 channel1 时， 这个消息就会被发送给订阅它的三个客户端：

------

## 实例

以下实例演示了发布订阅是如何工作的。在我们实例中我们创建了订阅频道名为 **redisChat**:

```
redis 127.0.0.1:6379> SUBSCRIBE redisChat

Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "redisChat"
3) (integer) 1
```

现在，我们先重新开启个 redis 客户端，然后在同一个频道 redisChat 发布两次消息，订阅者就能接收到消息。

```
redis 127.0.0.1:6379> PUBLISH redisChat "Redis is a great caching technique"

(integer) 1

redis 127.0.0.1:6379> PUBLISH redisChat "Learn redis by runoob.com"

(integer) 1

# 订阅者的客户端会显示如下消息
1) "message"
2) "redisChat"
3) "Redis is a great caching technique"
1) "message"
2) "redisChat"
3) "Learn redis by runoob.com"

```

------

## Redis 发布订阅命令

下表列出了 redis 发布订阅常用命令：

| 序号   | 命令及描述                                    |
| ---- | ---------------------------------------- |
| 1    | [PSUBSCRIBE pattern [pattern ...\]](http://www.runoob.com/redis/pub-sub-psubscribe.html) 订阅一个或多个符合给定模式的频道。 |
| 2    | [PUBSUB subcommand [argument [argument ...\]]](http://www.runoob.com/redis/pub-sub-pubsub.html) 查看订阅与发布系统状态。 |
| 3    | [PUBLISH channel message](http://www.runoob.com/redis/pub-sub-publish.html) 将信息发送到指定的频道。 |
| 4    | [PUNSUBSCRIBE [pattern [pattern ...\]]](http://www.runoob.com/redis/pub-sub-punsubscribe.html) 退订所有给定模式的频道。 |
| 5    | [SUBSCRIBE channel [channel ...\]](http://www.runoob.com/redis/pub-sub-subscribe.html) 订阅给定的一个或多个频道的信息。 |
| 6    | [UNSUBSCRIBE [channel [channel ...\]]](http://www.runoob.com/redis/pub-sub-unsubscribe.html) 指退订给定的频道。 |

# Redis 事务

Redis 事务可以一次执行多个命令， 并且带有以下两个重要的保证：

- 事务是一个单独的隔离操作：事务中的所有命令都会序列化、按顺序地执行。事务在执行的过程中，不会被其他客户端发送来的命令请求所打断。
- 事务是一个原子操作：事务中的命令要么全部被执行，要么全部都不执行。

一个事务从开始到执行会经历以下三个阶段：

- 开始事务。
- 命令入队。
- 执行事务。

------

## 实例

以下是一个事务的例子， 它先以 **MULTI** 开始一个事务， 然后将多个命令入队到事务中， 最后由 **EXEC** 命令触发事务， 一并执行事务中的所有命令：

```
redis 127.0.0.1:6379> MULTI
OK

redis 127.0.0.1:6379> SET book-name "Mastering C++ in 21 days"
QUEUED

redis 127.0.0.1:6379> GET book-name
QUEUED

redis 127.0.0.1:6379> SADD tag "C++" "Programming" "Mastering Series"
QUEUED

redis 127.0.0.1:6379> SMEMBERS tag
QUEUED

redis 127.0.0.1:6379> EXEC
1) OK
2) "Mastering C++ in 21 days"
3) (integer) 3
4) 1) "Mastering Series"
   2) "C++"
   3) "Programming"

```

------

## Redis 事务命令

下表列出了 redis 事务的相关命令：

| 序号   | 命令及描述                                    |
| ---- | ---------------------------------------- |
| 1    | [DISCARD](http://www.runoob.com/redis/transactions-discard.html) 取消事务，放弃执行事务块内的所有命令。 |
| 2    | [EXEC](http://www.runoob.com/redis/transactions-exec.html) 执行所有事务块内的命令。 |
| 3    | [MULTI](http://www.runoob.com/redis/transactions-multi.html) 标记一个事务块的开始。 |
| 4    | [UNWATCH](http://www.runoob.com/redis/transactions-unwatch.html) 取消 WATCH 命令对所有 key 的监视。 |
| 5    | [WATCH key [key ...\]](http://www.runoob.com/redis/transactions-watch.html) 监视一个(或多个) key ，如果在事务执行之前这个(或这些) key 被其他命令所改动，那么事务将被打断。 |

# Redis 脚本

Redis 脚本使用 Lua 解释器来执行脚本。 Reids 2.6 版本通过内嵌支持 Lua 环境。执行脚本的常用命令为 **EVAL**。

### 语法

Eval 命令的基本语法如下：

```
redis 127.0.0.1:6379> EVAL script numkeys key [key ...] arg [arg ...]

```

### 实例

以下实例演示了 redis 脚本工作过程：

```
redis 127.0.0.1:6379> EVAL "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}" 2 key1 key2 first second

1) "key1"
2) "key2"
3) "first"
4) "second"

```

------

## Redis 脚本命令

下表列出了 redis 脚本常用命令：

| 序号   | 命令及描述                                    |
| ---- | ---------------------------------------- |
| 1    | [EVAL script numkeys key [key ...\] arg [arg ...]](http://www.runoob.com/redis/scripting-eval.html) 执行 Lua 脚本。 |
| 2    | [EVALSHA sha1 numkeys key [key ...\] arg [arg ...]](http://www.runoob.com/redis/scripting-evalsha.html) 执行 Lua 脚本。 |
| 3    | [SCRIPT EXISTS script [script ...\]](http://www.runoob.com/redis/scripting-script-exists.html) 查看指定的脚本是否已经被保存在缓存当中。 |
| 4    | [SCRIPT FLUSH](http://www.runoob.com/redis/scripting-script-flush.html) 从脚本缓存中移除所有脚本。 |
| 5    | [SCRIPT KILL](http://www.runoob.com/redis/scripting-script-kill.html) 杀死当前正在运行的 Lua 脚本。 |
| 6    | [SCRIPT LOAD script](http://www.runoob.com/redis/scripting-script-load.html) 将脚本 script 添加到脚本缓存中，但并不立即执行这个脚本。 |

# Redis 连接

Redis 连接命令主要是用于连接 redis 服务。

### 实例

以下实例演示了客户端如何通过密码验证连接到 redis 服务，并检测服务是否在运行：

```
redis 127.0.0.1:6379> AUTH "password"
OK
redis 127.0.0.1:6379> PING
PONG

```

------

## Redis 连接命令

下表列出了 redis 连接的基本命令：

| 序号   | 命令及描述                                    |
| ---- | ---------------------------------------- |
| 1    | [AUTH password](http://www.runoob.com/redis/connection-auth.html) 验证密码是否正确 |
| 2    | [ECHO message](http://www.runoob.com/redis/connection-echo.html) 打印字符串 |
| 3    | [PING](http://www.runoob.com/redis/connection-ping.html) 查看服务是否运行 |
| 4    | [QUIT](http://www.runoob.com/redis/connection-quit.html) 关闭当前连接 |
| 5    | [SELECT index](http://www.runoob.com/redis/connection-select.html) 切换到指定的数据库 |

# Redis 服务器

Redis 服务器命令主要是用于管理 redis 服务。

### 实例

以下实例演示了如何获取 redis 服务器的统计信息：

```
redis 127.0.0.1:6379> INFO

# Server
redis_version:2.8.13
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:c2238b38b1edb0e2
redis_mode:standalone
os:Linux 3.5.0-48-generic x86_64
arch_bits:64
multiplexing_api:epoll
gcc_version:4.7.2
process_id:3856
run_id:0e61abd297771de3fe812a3c21027732ac9f41fe
tcp_port:6379
uptime_in_seconds:11554
uptime_in_days:0
hz:10
lru_clock:16651447
config_file:

# Clients
connected_clients:1
client-longest_output_list:0
client-biggest_input_buf:0
blocked_clients:0

# Memory
used_memory:589016
used_memory_human:575.21K
used_memory_rss:2461696
used_memory_peak:667312
used_memory_peak_human:651.67K
used_memory_lua:33792
mem_fragmentation_ratio:4.18
mem_allocator:jemalloc-3.6.0

# Persistence
loading:0
rdb_changes_since_last_save:3
rdb_bgsave_in_progress:0
rdb_last_save_time:1409158561
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:0
rdb_current_bgsave_time_sec:-1
aof_enabled:0
aof_rewrite_in_progress:0
aof_rewrite_scheduled:0
aof_last_rewrite_time_sec:-1
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok

# Stats
total_connections_received:24
total_commands_processed:294
instantaneous_ops_per_sec:0
rejected_connections:0
sync_full:0
sync_partial_ok:0
sync_partial_err:0
expired_keys:0
evicted_keys:0
keyspace_hits:41
keyspace_misses:82
pubsub_channels:0
pubsub_patterns:0
latest_fork_usec:264

# Replication
role:master
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0

# CPU
used_cpu_sys:10.49
used_cpu_user:4.96
used_cpu_sys_children:0.00
used_cpu_user_children:0.01

# Keyspace
db0:keys=94,expires=1,avg_ttl=41638810
db1:keys=1,expires=0,avg_ttl=0
db3:keys=1,expires=0,avg_ttl=0

```

------

## Redis 服务器命令

下表列出了 redis 服务器的相关命令:

| 序号   | 命令及描述                                    |
| ---- | ---------------------------------------- |
| 1    | [BGREWRITEAOF](http://www.runoob.com/redis/server-bgrewriteaof.html) 异步执行一个 AOF（AppendOnly File） 文件重写操作 |
| 2    | [BGSAVE](http://www.runoob.com/redis/server-bgsave.html) 在后台异步保存当前数据库的数据到磁盘 |
| 3    | [CLIENT KILL [ip:port\] [ID client-id]](http://www.runoob.com/redis/server-client-kill.html) 关闭客户端连接 |
| 4    | [CLIENT LIST](http://www.runoob.com/redis/server-client-list.html) 获取连接到服务器的客户端连接列表 |
| 5    | [CLIENT GETNAME](http://www.runoob.com/redis/server-client-getname.html) 获取连接的名称 |
| 6    | [CLIENT PAUSE timeout](http://www.runoob.com/redis/server-client-pause.html) 在指定时间内终止运行来自客户端的命令 |
| 7    | [CLIENT SETNAME connection-name](http://www.runoob.com/redis/server-client-setname.html) 设置当前连接的名称 |
| 8    | [CLUSTER SLOTS](http://www.runoob.com/redis/server-cluster-slots.html) 获取集群节点的映射数组 |
| 9    | [COMMAND](http://www.runoob.com/redis/server-command.html) 获取 Redis 命令详情数组 |
| 10   | [COMMAND COUNT](http://www.runoob.com/redis/server-command-count.html) 获取 Redis 命令总数 |
| 11   | [COMMAND GETKEYS](http://www.runoob.com/redis/server-command-getkeys.html) 获取给定命令的所有键 |
| 12   | [TIME](http://www.runoob.com/redis/server-time.html) 返回当前服务器时间 |
| 13   | [COMMAND INFO command-name [command-name ...\]](http://www.runoob.com/redis/server-command-info.html) 获取指定 Redis 命令描述的数组 |
| 14   | [CONFIG GET parameter](http://www.runoob.com/redis/server-config-get.html) 获取指定配置参数的值 |
| 15   | [CONFIG REWRITE](http://www.runoob.com/redis/server-config-rewrite.html) 对启动 Redis 服务器时所指定的 redis.conf 配置文件进行改写 |
| 16   | [CONFIG SET parameter value](http://www.runoob.com/redis/server-config-set.html) 修改 redis 配置参数，无需重启 |
| 17   | [CONFIG RESETSTAT](http://www.runoob.com/redis/server-config-resetstat.html) 重置 INFO 命令中的某些统计数据 |
| 18   | [DBSIZE](http://www.runoob.com/redis/server-dbsize.html) 返回当前数据库的 key 的数量 |
| 19   | [DEBUG OBJECT key](http://www.runoob.com/redis/server-debug-object.html) 获取 key 的调试信息 |
| 20   | [DEBUG SEGFAULT](http://www.runoob.com/redis/server-debug-segfault.html) 让 Redis 服务崩溃 |
| 21   | [FLUSHALL](http://www.runoob.com/redis/server-flushall.html) 删除所有数据库的所有key |
| 22   | [FLUSHDB](http://www.runoob.com/redis/server-flushdb.html) 删除当前数据库的所有key |
| 23   | [INFO [section\]](http://www.runoob.com/redis/server-info.html) 获取 Redis 服务器的各种信息和统计数值 |
| 24   | [LASTSAVE](http://www.runoob.com/redis/server-lastsave.html) 返回最近一次 Redis 成功将数据保存到磁盘上的时间，以 UNIX 时间戳格式表示 |
| 25   | [MONITOR](http://www.runoob.com/redis/server-monitor.html) 实时打印出 Redis 服务器接收到的命令，调试用 |
| 26   | [ROLE](http://www.runoob.com/redis/server-role.html) 返回主从实例所属的角色 |
| 27   | [SAVE](http://www.runoob.com/redis/server-save.html) 异步保存数据到硬盘 |
| 28   | [SHUTDOWN [NOSAVE\] [SAVE]](http://www.runoob.com/redis/server-shutdown.html) 异步保存数据到硬盘，并关闭服务器 |
| 29   | [SLAVEOF host port](http://www.runoob.com/redis/server-slaveof.html) 将当前服务器转变为指定服务器的从属服务器(slave server) |
| 30   | [SLOWLOG subcommand [argument\]](http://www.runoob.com/redis/server-showlog.html) 管理 redis 的慢日志 |
| 31   | [SYNC](http://www.runoob.com/redis/server-sync.html) 用于复制功能(replication)的内部命令 |