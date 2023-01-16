---
title: "Redis教程(二)---redis常用命令"
description: "redis常用命令汇总"
date: 2020-01-05 22:00:00
draft: false
tags: ["Redis"]
categories: ["Redis"]
---

redis常用命令汇总，包括`Key`、`String`、`Hash`、`List`、`Set`、`ZSet`、发布订阅等待。

<!--more-->

## 0. Key

基本操作

```sh
DEL key
EXISTS key
EXPIRE key seconds
#模糊查询所有匹配的key
KEYS pattern
#移除key的过期时间
PERSIST key
#以秒为单位，返回给定 key 的剩余生存时间(TTL, time to live)。
TTL key
#从当前数据库中随机返回一个key
RANDOMKEY
#仅当 newkey 不存在时，将 key 改名为 newkey 。
RENAMENX key newkey
#返回key的类型
TYPE key
```





## 1. String

作为常规的key-value缓存应用

一个键最大能存储512MB

常用的大概有`get`、`set`、`inc`等

```sh
# 可以在设置的同时指定过期时间
SET key value (expire)
#SET+Expire
SETEX key second value
GET key
#设置新值返回旧值
GETSET key value
STRLEN key

#返回 key 中字符串值的子字符
GETRANGE key start end
#key不存在才设置 分布式锁
SETNX key value
#所有key都不存在才设置
MSETNX key value[key value...]
#批量get set
MGET key1[key2...]
MSET key1[key2...]
#在原来的值上加或减 key不存在则会初始化为0之后执行 会返回命令执行后的值 只能在integer上操作 
INCR key
INCRBY key inc
INCRBYFLOAT key inc #增加浮点数
DECR key
DECRBY key inc
#字符串末尾追加数据
APPEEND key value
```

## 2. Hash

主要用来存储对象信息。

> 比如存储用户信息
>
> key为用户名或id
>
> field则为各种字段(name,age)
>
> value就是对应的信息(illusory,23)

命令都差不多 主要多了个`HLen`、`HKeys`、`HVals`、`HScan`等

常用的`hset`、`hget`、`hgetall`、`HINCRBY`、`hscan`等

```sh
HSET key field value
HSETNX key field value

HGET key field
HGETALL key field

HMSET key field value[field value...]
HMGET key field[field...]

HDEL key field[field...]
HEXISTS key field

HINCRBY key field inc
HINCRBYFLOAT key field inc
# 获取hash表中字段的数量
HLEN key
# 获取hash表中所有字段
HKEYS key
# 获取hash表中所有值
HVALS key

# 扫描整个hash表 直接用keys命令可能产生阻塞
# cursor填0则表示第一次遍历 从头开始
# MATCH pattern可以模糊匹配
# count 可以看成是每次返回的数量 虽然不完全是这样
HSCAN key cursor [MATCH pattern][Count count]
```



## 3. List

主要用于各种列表数据，比如`关注列表`、`粉丝列表`、`最新消息列表` 等

也可以做各种数据收集的工作。

> 比如统计数据按时间戳以分钟数区别写入redis
>
> 然后后续任务从redis中读取写入到db

当然list能做的zset也能做 但是同样的数据zset占用的空间会比list多很多很多。

常用命令` lpush`、`rpush`、`lpop`、`rpop`、`lrange` 等

```sh
#将值插入到表头(最左边)
#list不存在时会新建
LPUSH key value1[value2]
#list不存在时什么都不做
LPUSHX key value

#将值插入到表尾(最右边)
#list不存在时会新建
RPUSH key value1[value2]
#list不存在时什么都不做
RPUSHX key value

# 弹出表中元素 L表头 R表尾
# 非阻塞
LPOP key
RPOP key

# 指定多个表 弹出第一个非空表中的元素
# 阻塞 B=blocking L=left R=right
BLPOP key1[key2] timeout
BRPOP key1[key2] timeout

#将source表尾最后一个元素弹出并写入destination 作为表头第一个元素
RPOPPUSH source destination
BRPOPPUSH source destination timeout
#返回表中下标为index的元素
LINDEX key index
#在表key的元素pivot前|后插入元素value
# key或者pivot不存在则不执行任何操作
LINSERT key BEFORE|AFTER pivot value
#获取列表长度
LLEN key
#返回列表指定范围内的元素
LRANGE key start stop

#根据参数 count 的值，移除列表中与参数 value 相等的元素。
#count > 0 : 从表头开始向表尾搜索，移除与 value 相等的元素，数量为 count 。
#count < 0 : 从表尾开始向表头搜索，移除与 value 相等的元素，数量为 count 的绝对值。
#count = 0 : 移除表中所有与 value 相等的值。
LREM key count value

#通过索引设置列表元素的值
LSET key index value
#对一个列表进行修剪(trim)，只保留指定区间内的元素，区间外元素都将被删除
LTRIM key start stop
```

## 4. Set

无序集合，元素唯一(自动去重)，可以方便的去交、并、差集， 以非常方便的实现如共同关注、共同喜好、二度好友等功能 。

常用命令` sadd`、`spop`、`smembers`、`sunion  `

```sh
SADD key member1[member2]
SREM key member1 [member2]
#获取集合的成员数
SCARD key

#返回给定所有集合的差集
SDIFF key1[key2]
#将差集存到指定集合
SDIFFSTORE destination key1 [key2]
#交集
SINTER key1 [key2]
SINTERSTORE destination key1 [key2]
#并集
SUNION key1 [key2]
SUNIONSTORE destination key1 [key2]

#判断元素是否是集合的成员
SISMEMBER key member
#返回集合中所有成员
SMEMBERS key
#将 member 元素从 source 集合移动到 destination 集合
SMOVE source destination member
#随机移除并返回集合中的一个元素
SPOP key
#返回集合中一个或多个随机数
SRANDMEMBER key [count]
#迭代集合中的元素
SSCAN key cursor [MATCH pattern] [COUNT count]
```



## 5. Sorted Set

 sorted set的使用场景与set类似，区别是set不是自动有序的，而sorted set可以通过用户额外提供一个优先级(score)的参数来为成员排序，并且是插入有序的，即自动排序。 

>  比如用来存成绩,自动根据score排序。
>
>  很方便的取指定分数的成员或者指定名次的成员

常用命令` zadd`、`zrange`、`zrem`、`zcard `

```sh
ZADD key score1 member1 [score2 member2]

ZCARD key
#计算在有序集合中指定区间分数的成员数
ZCOUNT key min max

#有序集合中对指定成员的分数加上增量 increment
ZINCRBY key increment member

#计算给定的一个或多个有序集的交集并将结果集存储在新的有序集合 key 中
ZINTERSTORE destination numkeys key [key ...]

#返回集合中指定成员之间的成员数量 根据名字排序 A-Z a-z这样
#eg ZLEXCOUNT  scoreSet [a [f 返回a到f之间的成员数 如果有的话
#用来存电话号码吧。。 ZLEXCOUNT phoneSet [133 [134 返回133-144号段的记录
#通过[ (来设置开闭区间
ZLEXCOUNT key min max


# 根据索引返回区间内的成员
ZRANGE key start stop [WITHSCORES]
#同ZLEXCOUNT根据名字排序 返回区间内的成员 必须要集合中所有成员分数相同时结果才准确
ZRANGEBYLEX key min max [LIMIT offset count]
# 通过分数排序 返回min到max分的成员
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT]

#返回指定成员的索引
ZRANK key member
#移除指定成员
ZREM key member [member ...]
#通过名字排序移除
ZREMRANGEBYLEX key min max
#通过索引，分数从高到低
ZREMRANGEBYRANK key start stop

#Rev=reverse 颠倒了排序方式 其他是一样的
#通过索引返回有序集中指定区间内的成员，按照分数从高到低排
ZREVRANGE key start stop [WITHSCORES]
#通过分数返回有序集中指定区间内的成员，按照分数从高到低排
ZREVRANGEBYSCORE key max min [WITHSCORES]
#返回指定成员的排序 按分数值递减(从大到小)排序
ZREVRANK key member
#返回成员的分数
ZSCORE key member
#迭代有序集合中的元素（包括元素成员和元素分值）
ZSCAN key cursor [MATCH pattern] [COUNT count]
```



## 6. 发布订阅

Redis 发布订阅(pub/sub)是一种消息通信模式：发送者(pub)发送消息，订阅者(sub)接收消息。

```sh
#订阅给定的一个或多个频道的信息。
SUBSCRIBE channel [channel ...]
#指退订给定的频道。
UNSUBSCRIBE [channel [channel ...]]
#将信息发送到指定的频道。
PUBLISH channel message

#订阅一个或多个符合给定模式的频道
PSUBSCRIBE pattern [pattern ...]
#退订所有给定模式的频道。
PUNSUBSCRIBE [pattern [pattern ...]]
#查看订阅与发布系统状态
PUBSUB subcommand [argument [argument ...]]
```



## 7. 文档

`https://redis.io/commands`

`http://doc.redisfans.com/key/scan.html`

