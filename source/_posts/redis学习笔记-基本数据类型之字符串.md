---
layout: redis
title: 1、Redis 学习笔记————基本数据类型之字符串
date: 2019-11-27 19:24:33
author: chenggx
img: http://static.xiangdangnian.net.cn/blog/b65c86ab1cbdae10bfb99911def7ad59.jpg
top: false
categories: redis
tags: 
  - redis
---


## 字符串（string）

### SET、GET |基本操作 

- 使用 set 关键字设置
- 使用 get 关键字获取字符串
- 值可以是任何类型的字符串（包括二进制，例如图片），值不能超过512 MB

```
> set key value
OK
> get key
"value"
```

### APPEND |追加值

- append 命令，如果 key 存在，则在 value 后追加值，不存在，则先创建一个 value 为空字符串的 key，然后在追加。

```
> append haha 123
(integer) 3

> get haha
"123"

> append haha 456
(integer) 6

> get haha
"123456"
```


### MSET |一次存储或获取多个 key

```
> mset a 10 b 20 c 30
OK

> mget a b c
1) "10"
2) "20"
3) "30"
```

### GETSET |获取原值并赋予新值

- 获取 key 原有的值，并赋予新值

```
> set num 100
OK
> getset num 50
"100"
> get num
"50"
```

### EXISTS、DEL |删除键、检查键是否存在

- 删除成功返回 1，失败返回 0
- 存在返回 1，不存在返回 0

```
> set abc 123
OK

> exists def
(integer) 0

> exists abc
(integer) 1

> del def
(integer) 0

> del abc
(integer) 1
```

### TYPE |查看数据类型操作命令

- 查询的键存在返回相应的数据类型，不存在返回 none

```
> set hello word
OK

> type hello
string

> type word
none
```

### SETEX、TTL、PSETEX、PTTL |设置过期时间（1 秒 = 1000 毫秒）

- 使用 setex 设置以秒为单位的过期时间
- 使用 psetex 设置以毫秒为单位的过期时间
- 使用 ttl 获取以秒为单位的剩余时间
- 使用 pttl 获取以毫秒为单位的剩余时间
> **使用 ttl 和 pttl 查询键的过期时间，键不存在返回 -2，没有设置过期时间返回 -1，其他情况返回以秒为单位的剩余时间**

```
// 秒为单位
> setex hello 10 11111
OK

> ttl hello
(integer) 6

> ttl hello
(integer) 4

> ttl hello
(integer) 3

> get hello
(nil)
```
```
//毫秒为单位
> psetex test 10000 1111
OK

> pttl test
(integer) 8455

> pttl test
(integer) 7430

> pttl test
(integer) 6733

> get test
(nil)
```

### INCR、INCRBY、DECR、DECRBY |原子递增递减

- set 一个值为整型的字符串，可以使用 incr 操作命令自增
- 可以使用 incrby 操作命令指定步长自增
- incr 是原子操作。也就是说客户端 1 和客户端 2 同时读出10，他们俩都对其加 1 操作，最终的值一定是12。
- 相应的有 decr 和 decrby 操作命令进行递减操作

```
> set count 100
OK

> incr count
(integer) 101
> incr count
(integer) 102

> incrby count 100
(integer) 202
```

### SETNX |不存在则设置，存在则不进行任何操作

- set 命令在执行的时候，如果 key 已经存在，则新值会覆盖旧值。
- setnx 命令，如果 key 存在，则不做任何操作，否则等同于 set。

```
> set test 123
OK

> set test 456
OK

> get test
"456"

> setnx test aaa
(integer) 0

> get test
"456"
```

### STRLEN | 计算 value 的长度

```
> set xixi abc123
OK

> strlen xixi
(integer) 6
```
