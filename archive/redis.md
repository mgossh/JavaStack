# Redis知识点总结

redis思维导图

## 数据结构
### 基础
* 字符串
* 散列
* 列表
* 集合
* 有序集合

### 扩展
* HyperLogLog
* 位图
* 地理坐标
* 流

## 发布与订阅
发布-订阅模式实现消息队列的缺陷

两个层面考虑：

* 性能：订阅者处理消息不够快，导致消息堆积，从而使输出缓冲区变大，可能导致redis速度变慢(旧版，新版使用了配置来规避)
* 稳定性：断网导致消息丢失

替代方案：使用列表来实现

## 事务
```
MULTI
...
EXEC
```
事务利弊
* 不支持事务回滚

## 持久化
redis持久化的两种方式

* `RDB`：快照存储
* `AOF`：追加文件

参考：

### RDB

#### 定义

 快照存储方式，即在指定的时间节点将redis内存中的数据进行快照存储。redis默认持久化方式，快照文件以.rdb后缀保存在磁盘。
 
#### 触发方式

* 手动触发，手动执行`save`和`bgsave`命令
   * `save`:会堵塞服务主线程，不推荐(废弃，杜绝生产环境使用)
   * `bgsave`：fork一个后台子进程，以异步方式执行数据备份操作。
* 自动触发
	* 配置文件 “save m n”,当m秒内发生n次变化时，触发bgsave
	* 从节点执行全量复制。主节点执行bgsave命令生成rdb发送给从节点
	* shutdown时，自动执行快照备份

#### 具体过程

bgsave命令操作流程：

1. 触发bgsave，判断当前redis父进程是否正在执行save，如果是，直接返回
2. 父进程执行fork操作创建子进程，该操作期间服务堵塞，直到fork成功
3. 子进程根据当前内存数据创建rdb临时文件，完成后对原文件进行原子覆盖
4. 子进程发送信号给父进程表示完成，父进程更新统计信息。

#### 优点

* 适合备份，全量复制。RDB 是一个非常紧凑（compact）的文件，它保存了 Redis 在某个时间点上的数据集。 这种文件非常适合用于进行备份： 比如说，你可以在最近的 24 小时内，每小时备份一次 RDB 文件，并且在每个月的每一天，也备份一个 RDB 文件。 这样的话，即使遇上问题，也可以随时将数据集还原到不同的版本。
* RDB 非常适用于灾难恢复（disaster recovery）：它只有一个文件，并且内容都非常紧凑，可以（在加密后）将它传送到别的数据中心，或者亚马逊 S3 中。
* RDB 可以最大化 Redis 的性能：父进程在保存 RDB 文件时唯一要做的就是 fork 出一个子进程，然后这个子进程就会处理接下来的所有保存工作，父进程无须执行任何磁盘 I/O 操作。
* 恢复数据速度比AOF快。

#### 缺点

* 实时性不好，有一定程度的数据丢失。如果你需要尽量避免在服务器故障时丢失数据，那么 RDB 不适合你。虽然 Redis 允许你设置不同的保存点（save point）来控制保存 RDB 文件的频率，但是，因为RDB文件需要保存整个数据集的状态，所以它并不是一个轻松的操作。因此你可能会至少 5 分钟才保存一次 RDB 文件。在这种情况下，一旦发生故障停机，你就可能会丢失好几分钟的数据。
* 每次保存 RDB 的时候，Redis 都要 fork() 出一个子进程，并由子进程来进行实际的持久化工作。 在数据集比较庞大时， fork() 可能会非常耗时，造成服务器在某某毫秒内停止处理客户端； 如果数据集非常巨大，并且 CPU 时间非常紧张的话，那么这种停止时间甚至可能会长达整整一秒。 虽然 AOF 重写也需要进行 fork() ，但无论 AOF 重写的执行间隔有多长，数据的耐久性都不会有任何损失。

#### 相关配置

```
# bgsave自动触发的条件；如果没有save m n配置，相当于自动的RDB持久化关闭，不过此时仍可以通过其他方式触发
save m n

# 当bgsave出现错误时，Redis是否停止执行写命令；设置为yes，则当硬盘出现问题时，可以及时发现，避免数据的大量丢失；设置为no，则Redis无视bgsave的错误继续执行写命令，当对Redis服务器的系统(尤其是硬盘)使用了监控时，该选项考虑设置为no。默认yes
stop-writes-on-bgsave-error yes

# 是否开启RDB文件压缩，默认yes
rdbcompression yes

# 是否开启RDB文件的校验，在写入文件和读取文件时都起作用,关闭checksum在写入文件和启动文件时大约能带来10%的性能提升，但是数据损坏时无法发现
rdbchecksum yes

# RDB文件名
dbfilename dump.rdb

# RDB文件和AOF文件所在目录
dir ./
```

### AOF

#### 定义

追加文件方式，记录每次对服务器的写操作,当服务器重启的时候会重新执行这些命令来恢复原始的数据。相比RDB,实时性好,主流的持久化方式。

当同时开启RDB和AOF时，优先执行AOF中的命令来执行数据恢复

#### 触发方式

redis默认开启RDB持久化，如果需要使用AOF,需要在配置文件中开启配置：

```
appendonly yes
```

开启后，redis的每一条写命令，都会记录在AOF文件中(类似日志文件)



#### 具体过程



#### 优点



#### 缺点

#### 相关配置
```
# 是否开启AOF,默认no
appendonly no

# AOF文件名，文件路径桶RDB文件路径一致
appendfilename "appendonly.aof"

# 性能和实时性的均衡考虑，采用每秒刷新AOF缓存区命令到文件
appendfsync everysec

# 在重写AOF文件时，是否禁止fsync,开启后能够减轻重写时磁盘和cpu的负载，但是可能会丢失AOF重写期间的数据。
# 均衡考虑，默认关闭。
no-appendfsync-on-rewrite no

# 自动触发重写AOF条件
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# redis使用aof恢复时，忽略最后一条可能存在问题的指令(一般是断电或意外关机导致的)
aof-load-truncated yes

# 混合持久化方案，默认关闭。4.0新增
aof-use-rdb-preamble no

```


### 持久化监控
```
info persistence
```

### 持久化选型

性能和容灾之间的博弈


### 参考资料
* [https://redis.io/topics/persistence](https://redis.io/topics/persistence)
* [Redis持久化](https://www.cnblogs.com/kismetv/p/9137897.html)

## 复制



## 集群

* 客户端分片
* 代理：Twemproxy

## 哨兵




## 内存回收机制--数据过期和淘汰策略

### 过期策略
为了防止一次性清理大量过期Key导致Redis服务受影响，Redis只在空闲时清理过期Key

* 定时过期

````
每个设置过期时间的key都需要创建一个定时器，到过期时间就会立即清除。该策略可以立即清除过期的数据，对内存很友好；但是会占用大量的CPU资源去处理过期的数据，从而影响缓存的响应时间和吞吐量。
````
* 惰性过期

```
只有当访问一个key时，才会判断该key是否已过期，过期则清除。该策略可以最大化地节省CPU资源，却对内存非常不友好。极端情况可能出现大量的过期key没有再次被访问，从而不会被清除，占用大量内存。
```
* 定期过期

```
每隔一定的时间，会扫描一定数量的数据库的expires字典中一定数量的key，并清除其中已过期的key。该策略是前两者的一个折中方案。通过调整定时扫描的时间间隔和每次扫描的限定耗时，可以在不同情况下使得CPU和内存资源达到最优的平衡效果。
```
Redis中同时使用了惰性过期和定期过期两种过期策略。

### 淘汰策略
redis是一个高性能的内存NoSQL数据库，当内存增长到一定程度时，就会执行数据淘汰，redis提供了6种数据淘汰策略：

* `volatile-lru`：利用 LRU 算法移除设置过过期时间的key。
* `volatile-random`：随机移除设置过过期时间的key。
* `volatile-ttl`：移除即将过期的key，根据最近过期时间来删除（辅以 TTL）
* `allkeys-lru`：利用 LRU 算法移除key。
* `allkeys-random`：随机移除任何key。
* `noeviction`：不移除任何key，只是返回一个写错误。

根据业务数据不同，使用不同的淘汰策略，使用INFO输出来监控缓存命中和错过次数，一般经验规则：

1. 如果数据呈现幂律分布，也就是一部分数据访问频率高，一部分数据访问频率低，则使用allkeys-lru
2. 如果数据呈现平等分布，也就是所有的数据访问频率都相同，则使用allkeys-random


### 缓存清理配置项：

* `maxmemory`：redis最大缓存配置，允许用户使用的最大缓存，当redis数据上升到最大值时，就会根据maxmemory-policy执行淘汰策略。
* `maxmemory-policy`：淘汰策略，提供6种机制
* `maxmemory-samples`：随机抽取淘汰样本数，默认设置为5。

## redis应用
### 实现消息队列
* List的`lpush`/`rpop`命令或者堵塞命令`blpop`/`brpop`
* `发布`-`订阅`模式

### 粉丝签到
利用数据结构`位图`记录每日签到记录，


## 运维
业界成熟方案

搜狐视频Redis私有云平台：[cachecloud](https://github.com/sohutv/cachecloud)

## 附录
1. [redis配置文件详解]()

## 参考资料
* [官网](https://redis.io/)
* [Github](https://github.com/antirez/redis)
* [Redis中文命令参考](http://redisdoc.com/)
* [Redis命令参考](https://redis.io/commands)
* [面试中关于Redis的问题看这篇就够了](https://juejin.im/post/5ad6e4066fb9a028d82c4b66)
* [Redis缓存穿透、缓存雪崩、redis并发问题分析](https://juejin.im/post/5b961172f265da0ab7198f4d)
* [如何高效深入的阅读Redis的源码](https://www.zhihu.com/question/28677076)
* [阿里云 Redis 开发规范](https://www.infoq.cn/article/K7dB5AFKI9mr5Ugbs_px)
* [redis运维-cachecloud](https://github.com/sohutv/cachecloud)
* [阿里数据库技术专家-怀听](https://yq.aliyun.com/users/lc72jj6ue3h36?spm=a2c4e.11153940.blogrightarea257459.2.42626027fjnkwZ)
* [Redis持久化方案该如何选型](https://zhuanlan.zhihu.com/p/39412293)
