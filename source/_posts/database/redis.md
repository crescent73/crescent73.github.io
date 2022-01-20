---
title: redis
date: 2021-11-5 9:23:00
categories:
- database
tags:
- database
- redis
---
# Redis

## 1. NoSQL数据库简介
### 1.1 NoSQL数据库概述
NoSQL(NoSQL = Not Only SQL )，意即“不仅仅是SQL”，泛指非关系型的数据库。

NoSQL不依赖业务逻辑方式存储，而以简单的 key-value模式存储。因此大大的增加了数据库的扩展能力。

特点：
+ 不遵循SQL标准
+ 不支持ACID(原子性、一致性、隔离性、持久性)
+ 远超于SQL的性能

### 1.2 NoSQL适用场景
+ 对数据高并发的读写
+ 海量数据的读写
+ 对数据高可扩展性的

### 1.3 NoSQL不适用场景
+ 需要事务支持
+ 基于sql的结构化查询存储，处理复杂的关系,需要即席查询。
+ (用不着sql的和用了sql 也不行的情况，请考虑用NoSql ) 

### 1.4 几种常用的NoSQL数据库

Memcache
+ 很早出现的NoSql数据库
+ 数据都在内存中，一般不持久化
+ 支持简单的key-value模式，支持类型单一\
+ 一般是作为缓存数据库辅助持久化的数据库

Reids
+ 几乎覆盖了Memcache的绝大部分功能
+ 数据都在内存中，**支持持久化**，主要用作备份恢复
+ 除了支持简单的key-value模式，还支持**多种数据结构**的存储,比如list、 set、 hash、 zset等
+ 一般是作为**缓存数据库**辅助持久化的数据库

MongoDB
+ 高性能、开源、模式自由(schema free)的**文档型数据库**
+ 数据都在内存中，如果内存不足，把不常用的数据保存到硬盘
+ 虽然是key-value模式,但是对value(尤其是json）提供了丰富的查询功能
+ 支持二进制数据及大型对象
+ 可以根据数据的特点**替代RDBMS** ，成为独立的数据库。或者配合RDBMS，存储特定的数据。

### 1.5 行式数据库和列式数据库
+ 行式数据库
+ 列式数据库：HBase、Cassandra

### 1.6 图关系数据库
+ 图关系数据库：Neo4j

## 2 Redis概述
+ Redis是一个开源的**key-value**存储系统。|
+ 和Memcached类似，它支持存储的value类型相对更多，包括string(字符串).list(链表)、set(集合)、zset(sorted set--有序集合)和hash(哈希类型)
+ 这些数据类型都支持push/pop、add/remove及取交集并集和差集及更丰富的操作，而且这些操作都是原子性的。
+ 在此基础上，Redis支持各种不同方式的排序。
+ 与memcached一样，为了保证效率，数据都是缓存在内存中。·
+ 区别的是Redis 会周期性的把更新的数据写入磁盘或者把修改操作写入追加的记录文件。

### 2.1 应用场景

#### 2.1.1 配合关系型数据库做高速缓存
+ 高频次，热门访问的数据，降低数据库IO
+ 分布式架构，做session共享

#### 2.1.2 多样的数据结构存储持久化数据
+ 最新N个数据 --> 通过依List实现按自然时间排序的数据
+ 排行榜，Top N --> 利用ZSet（有序集合)
+ 时效性的数据，比如手机验证码 --> Expire过期
+ 计数器，秒杀 --> 原子性，自增方法INCR、DECR
+ 去除大量数据中的重复数据 --> 利用Set集合
+ 构建队列 --> 利用list集合
+ 发布订阅消息系统 --> pub/sub模式

### 2.2 Redis安装
#### 2.2.1 安装地址
官网：https://redis.io/download

#### 2.2.2 安装目录
redis默认安装目录：
+ redis-benchmark:性能测试工具，可以在自己本子运行，看看自己本子性能如何
+ redis-check-aof :修复有问题的AOF文件，rdb和aof后面讲·
+ redis-check-dump:修复有问题的 dump.rdb文件·
+ redis-sentinel : Redis集群使用·
+ redis-server : Redis服务器启动命令
+ redis-cli :客户端，操作入口

### 2.2.3 前台启动（不推荐）
直接在命令行输入`redis-server`。命令行窗口不能关闭，否则服务器停止

### 2.2.4 后台启动（推荐）
将redis设置为后台启动后，`redis-server redis.conf` 就能实现后台启动
关闭：`redis-cli shutdown`
redis服务控制：
```
sudo service redis start # 启动redis
sudo service redis restart # 重启redis
sudo service redis stop # 关闭redis
sudo systemctl status redis # 查看redis运行状态
```

### 2.2.5 redis配置
修改redis配置：sudo vi /etc/redis/redis.conf

```
daemonize yes # 支持后台启动
bind 0.0.0.0  # 支持远程连接
requirepass 123456 # 需要密码
protected-mode yes # 支持远程访问
```

### 2.2.6 redis相关知识介绍
+ 默认端口6379(merz)
+ 默认16个数据库，类似数组下标从o开始，初始默认使用0号库
+ 使用命令select <dbid>来切换数据库。如: select 8 
+ 统一密码管理，所有库同样密码。·
+ dbsize查看当前数据库的 key的数量。
+ flushdb清空当前库
+ flushall通杀全部库。

串行 vs 多线程+锁 ( memcached ) vs 单线程+多路IO复用(Redis).
(与Memcache三点不同:支持多数据类型，支持持久化，单线程+多路IO复用)

Redis是单线程+多路IO复用技术

多路复用是指使用一个线程来检查多个文件描述符( Socket )的就绪状态，比如调用select和poll函数，传入多个文件描述符，如果有一个文件描述符就绪，则返回，否则阻塞直到超时。得到就绪状态后进行真正的操作可以在同一个线程里执行，也可以启动线程执行(比如使用线程池)

## 3. 常用的五大数据类型
常用的五大数据类型：String、List、Set、Hash、Zset

### 3.1 Redis键（key）

+ set key value，给key设置value
+ keys * 查看当前库所有kev(匹配: keys *1)
+ exists key判断某个key是否存在
+ type key查看你的 key是什么类型
+ del key 删除指定的key数据
+ unlink key根据value选择非阻塞删除（仅将keys 从 keyspace元数据中删除，真正的删除会在后续异步操作。
+ expire key 10，10秒钟:为给定的 key设置过期时间
+ ttl key查看还有多少秒过期，-1表示永不过期，-2表示已过期
+ select 命令切换数据库
+ dbsize 查看当前数据库的key的数量
+ flushdb 清空当前库
+ flushall 清空所有库

### 3.2 Redis字符串（String）
#### 3.2.1 简介
String是Redis最基本的类型，你可以理解成与Memcached一模一样的类型，一个key对应一个value。

String类型是二进制安全的。意味着Redis的string可以包含任何数据。比如jpg图片或者序列化的对象。

String类型是Redis最基本的数据类型，一个Redis中字符串value最多可以是512M

#### 3.2.2 常用命令
+ set key value，设置相同的key的值会覆盖
+ get key 查询对应键值
+ append key value 将给定的value追加到原值的末尾
+ strlen key 获得值的长度
+ setnx key value 只有在key不存在时设置key的值
+ incr key 将key中存储的数字值增1，只能对数字值操作，如果为空，新增值为1
+ decr key 将key中储存的数字值减1，只能对数字值操作，如果为空，新增值为-1
+ incrby/decrby key 步长，将key中存储的数字值增减，自定义步长。

注意：incr/decr操作是原子性操作

所谓原子操作是指不会被线程调度机制打断的操作。
这种操作一旦开始，就一直运行到结束，中间不会有任何context switch(切换到另一个线程)。
1. 在单线程中，能够在单条指令中完成的操作都可以认为是"原子操作”，因为中断只能发生于指令之间。
2. 在多线程中，不能被其它进程（线程)打断的操作就叫原子操作。

Redis单命令的原子性主要得益于Redis 的单线程。

+ mset.…...key1 value1 key2 value2 ...，同时设置一个或多个key-value对。
+ mget ..key1 key2 key3 ...，同时获取一个或多个value
+ msetnx key1 value1 key2 value2.....，同时设置一个或多个key-value对，当且仅当所有给定key都不存在。

注意：原子性，**有一个失败则都失败**

+ getrange key <起始位置><结束位置>，获得值的范围，类似java中的 substring，前包，后包
+ setrange  key <起始位置> value ，用 value 覆写 key 所储存的字符串值，从<起始位置>开始(索引从0开始).
+ setex  key <过期时间> value ，设置键值的同时，设置过期时间，单位秒。
+ getset  key value，以新换旧，设置了新值同时获得旧值。

#### 3.2.2 数据结构
String的数据结构为简单动态字符串(Simple Dynamic String,缩写SDS)。是可以修改的字符串，内部结构实现上类似于Java的 ArrayList,采用预分配冗余空间的方式来减少内存的频繁分配..

### 3.3 Redis列表（List）
#### 3.3.1 简介
单键多值

Redis列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素到列表的头部(左边）或者尾部(右边)。

它的底层实际是个**双向链表**，**对两端的操作性能很高**，通过索引下标的操作中间的节点性能会较差。
![1636077133618.png](https://s2.loli.net/2022/01/19/Vg4WTwukBOPrl7G.png)

#### 3.3.2 常用命令
+ lpush/rpush key value1 value2 value3 ....从左边/右边插入一个或多个值。
+ lpop/rpop key 从左边/右边吐出一个值。**值在键在，值光键亡**。
+ rpoplpush key1 key2 从 key1 列表右边吐出一个值，插到 key2 列表左边。
+ lrange  key start stop 按照索引下标获得元素(从左到右)，lrange mylist 0 -1 0左边第一个，-1右边第一个，(0-1表示获取所有)
+ lindex key index 按照索引下标获得元素(从左到右)
+ llen key 获得列表长度
+ linsert key before value newvalue ，在 value 的后面插入 newvalue 插入值
+ lrem  key n value ，从左边删除n个value(从左到右)
+ lset key index value 将列表key下标为index的值替换成value

#### 3.3.3 数据结构
List的数据结构为快速链表quickListL

首先在列表元素较少的情况下会使用一块连续的内存存储，这个结构是ziplist，也即是压缩列表。
它将所有的元素紧挨着一起存储，分配的是一块连续的内存。

当数据量比较多的时候才会改成quicklist。
因为普通的链表需要的附加指针空间太大，会比较浪费空间。比如这个列表里存的只是int类型的数据，结构上还需要两个额外的指针prev和next。
![1636078213627.png](https://s2.loli.net/2022/01/19/NhDM3VL7y2Iwjd6.png)
Redis将链表和ziplist结合起来组成了quicklist。也就是将多个ziplist使用双向指针串起来使用。这样既满足了快速的插入删除性能，又不会出现太大的空间冗余。

### 3.4 Redis集合（Set）
#### 3.4.1 简介
Redis set对外提供的功能与list类似是一个列表的功能，特殊之处在于set是可以自动排重的，当你需要存储一个列表数据，又不希望出现重复数据时，set是一个很好的选择，并且set提供了判断某个成员是否在一个set集合内的重要接口，这个也是list所不能提供的。

Redis的set是string类型的无序集合。它**底层其实是一个value为null的hash表**，所以添加，删除，查找的**复杂度都是O(1)**。
一个算法，随着数据的增加，执行时间的长短，如果是O(1)，数据增加，查找数据的时间不变。

#### 3.4.2 常用命令
+ sadd key value1 value2  ..... ，将一个或多个member元素加入到集合key中，已经存在的member元素将被忽略
+ smembers key 取出该集合的所有值。
+ sismember key value 判断集合 key 是否为含有该 value 值，有1，没有0
+ scard key 返回该集合的元素个数
+ srem key value1 value2 ....删除集合中的某个元素。
+ spop key **随机**从该集合中吐出一个值。
+ srandmember key n 随机从该集合中取出n个值。不会从集合中删除。
+ smove source destination value把集合中一个值从一个集合移动到另一个集合
+ sinter key1 key2 返回两个集合的交集元素。
+ sunion key1 key2 返回两个集合的并集元素。
+ sdiff key1 key2 返回两个集合的差集元素(key1中的，不包含key2中的)

#### 3.4.3 数据结构
set数据结构是dict字典，字典是用哈希表实现的。

Java中 HashSet的内部实现使用的是HashMap，只不过所有的value都指向同一个对象。Redis的set结构也是一样，它的内部也使用hash结构，所有的value都指向同一个内部值。

### 3.5 Redis哈希（Hash）
#### 3.5.1 简介
Redis hash是一个键值对集合: 
Redis hash是一个string类型的 field和value的映射表，hash特别适合用于存储对象。
类似Java里面的Map<String.Object>

#### 3.5.2 常用命令
+ hset key field value 给 key 集合中的 field 键赋值 value
+ hget key1 field 从 key1 集合 field 取出value 
+ hmset key1 field1 value1 field2 value2 ...批量设置hash 的值
+ hexists key1 field 查看哈希表key中，给定域field是否存在。
+ hkeys key 列出该hash集合的所有field
+ hvals key 列出该hash 集合的所有valuel
+ hincrby key field increment 为哈希表key中的域field 的值加上增量1 -1
+ hsetnx key field value 将哈希表key中的域field的值设置为value，当且仅当域field不存在.

#### 3.5.3 数据结构
Hash类型对应的数据结构是两种: ziplist（压缩列表)，hashtable (哈希表)。当field-value长度较短且个数较少时，使用ziplist，否则使用hashtable。

### 3.6 Redis有序集合（Zset）

#### 3.6.1 简介

Redis有序集合zset与普通集合set非常相似，是一个没有重复元素的字符串集合。

不同之处是有序集合的每个成员都关联了一个**评分(score)** ,这个评分( score )被用来按照从最低分到最高分的方式排序集合中的成员。**集合的成员是唯一的，但是评分可以是重复的**。

因为元素是有序的，所以你也可以很快的根据评分(score)或者次序(position)来获取一个范围的元素。

访问有序集合的中间元素也是非常快的,因此你能够使用有序集合作为一个没有重复成员的智能列表。

#### 3.6.2 常用命令
+ zadd key score1 value1 score2 value2 ...将一个或多个member元素及其score值加入到有序集key当中。
+ vzrange key start stop [WITHSCORES]，返回有序集 key中，下标在 start stop 之间的元素。带WITHSCORES，可以让分数一起和值返回到结果集。
+ zrangebyscore key min max [withscores] [limit offset count]，返回有序集 key中，所有score值介于min和max 之间(包括等于min或max )的成员。有序集成员按score值递增(从小到大)次序排列。
+ zrevrangebyscore key max min [withscores] [limit offset count]，同上，改为从大到小排列。
+ zincrby key increment value 为元素的score加上增量
+ zrem key value 删除该集合下，指定值的元素
+ zcount key min max 统计该集合，分数区间内的元素个数
+ zrank key value 返回该值在集合中的排名，从0开始。

#### 3.6.3 数据结构
SortedSet(zset)是Redis提供的一个非常特别的数据结构，一方面它等价于Java的数据结构Map<String，Double>，可以给每一个元素value赋予一个权重score，另一方面它又类似于TreeSet，内部的元素会按照权重score进行排序，可以得到每个元素的名次，还可以通过score的范围来获取元素的列表。

zset 底层使用了两个数据结构.
1. hash, hash的作用就是关联元素value和权重score，保障元素value的唯—性，可以通过元素value找到相应的score值。
2. 跳跃表，跳跃表的目的在于给元素value排序，根据score的范围获取元素列表。


## 4. Redis的发布和订阅
### 4.1 发布和订阅
Redis 发布订阅(pub/sub)是一种消息通信模式∶发送者(pub)发送消息，订阅者(sub)接收消息。

Redis客户端可以订阅任意数量的频道。
### 4.2 命令行实现发布订阅
1. 打开客户端订阅channel1：`SUBSCRIBE channel`
2. 打开另一个客户端，给channel1发布消息hello：`publish channel1 hello`
3. 打开第一个客户端可以看到发送的消息

## 5. Redis6新数据类型
### 5.1 BiyMaps
现代计算机用二进制(位）作为信息的基础单位，1个字节等于8位，例如“abc"字符串是由3个字节组成，但实际在计算机存储时将其用二进制表示，“abc”分别对应的 ASCII 码分别是 97、98、99，对应的二进制分别是01100001、01100010和01100011。

合理地使用操作位能够有效地提高内存使用率和开发效率。

Redis提供了Bitmaps这个“数据类型”可以实现对位的操作:。
1. Bitmaps 本身不是一种数据类型，实际上它就是字符串 (key-value),但是它可以对字符串的位进行操作。
2. Bitmaps单独提供了一套命令，所以在 Redis 中使用Bitmaps 和使用字符串的方法不太相同。可以把 Bitmaps想象成一个以位为单位的数组，数组的每个单元只能存储0和1，数组的下标在Bitmaps 中叫做偏移量。

在第一次初始化 Bitmaps时，假如偏移量非常大，那么整个初始化过程执行会比较慢，可能会造成 Redis 的阻塞。

### 5.2 HyperLogLog
Redis HyperLogLog是用来做基数统计的算法，HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定的、并且是很小的。

在Redis里面，每个 HyperLogLog键只需要花费12KB内存，就可以计算接近2^64个不同元素的基数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。

但是，因为 HyperLogLog只会根据输入元素来计算基数，而不会储存输入元素本身，所以 HyperLogLog不能像集合那样，返回输入的各个元素。

### 5.3 Geospatial
Redis 3.2中增加了对GEO类型的支持。GEO ，Geographic，地理信息的缩写。该类型，就是元素的⒉维坐标，在地图上就是经纬度。redis 基于该类型，提供了经纬度设置，查询，范围查询，距离查询，经纬度Hash 等常见操作。

## 6. Java操作Redis
直接操作：Jedis
spring boot操作：spring-boot-starter-data-redis