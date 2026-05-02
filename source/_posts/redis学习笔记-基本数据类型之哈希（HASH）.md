---
layout: reids
title: redis学习笔记-----基本数据类型之哈希（HASH）
date: 2019-12-12 20:27:41
author: chenggx
img: http://static.xiangdangnian.net.cn/blog/9c41d85e67935e1404a8037ea49b1ba3473fe13f.jpg
top: false
categories: redis
tags:
 - redis
---

# 哈希（HASH）

> 哈希就像是一个微缩版的 redis。由键值对组成。一般讲数据库中的记录取出直接放入 redis 中使用。

## HSET、HMSET

- 设置单个key
- 一次设置多个key

> 经过测试当前版本5.0.4，hset 也可以一次设置多个 key

### hset 设置多个 key
```
> hset user:1 name jack email jack@qq.com
(integer) 2

> hget user:1 name
"jack"

> hget user:1 email
"jack@qq.com"
```

### hset 设置单个 key
```
> hset user:2 name tom email tom@qq.com
(integer) 2

> hget user:2 name
"tom"

> hget user:2 email
"tom@qq.com"
```

## HGET、HMGET

- hget 获取单个 key 的值
- hmget 一次获取多个 key 的值

```
> hmget user:2 name email
1) "tom"
2) "tom@qq.com"
```

## HEDL 

- 删除一个或多个指定的key

```
> hdel user:2 name
(integer) 1

> hget user:2 name
(nil)

> hget user:2 email
"tom@qq.com"
```

## HSETNX

- 设置指定key的值，如果可以已经存在，则不进行任何操作

```
> hsetnx user:2 email 2@qq.com
(integer) 0

> hsetnx user:2 name tome
(integer) 1

> hmget user:2 name email
1) "tome"
2) "tom@qq.com"
```

## HVALS

-返回所有格值

```
> hvals user:2
1) "tom@qq.com"
2) "tome"
```

## HKEYS 

- 返回所有的键

```
> hkeys user:2
1) "email"
2) "name"
```

## HGETALL 

- 返回所有的键和值
- 返回值中，每个字段名的下一个就是他的值

```
> hgetall user:2
1) "email"
2) "tom@qq.com"
3) "name"
4) "tome"
```

## HEXISTS 

-  检查指定键是否存在

```
> hexists user:2 class
(integer) 0

> hexists user:2 email
(integer) 1
```

## HINCRBY 

- 对哈希中指定的键进行自增操作，如果对应的键不存在，则创建。如果存在，直接新增
- 所操作的键的值必须为整型

```
> hincrby user:2 old 1
(integer) 1

> hgetall user:2
1) "email"
2) "tom@qq.com"
3) "name"
4) "tome"
5) "sex"
6) "man"
7) "old"
8) "1"

> hincrby user:2 old 2
(integer) 3


> hgetall user:2
1) "email"
2) "tom@qq.com"
3) "name"
4) "tome"
5) "sex"
6) "man"
7) "old"
8) "3"

> hincrby user:2 name 1
(error) ERR hash value is not an integer
```

## HINCRBYFLOAT
- 同上面的 hincrby，只不过支持的值是 float 类型

## HLEN

- 返回指定哈希所包含的字段数量（key 的数量）

```
> hlen user:2
(integer) 4

> hgetall user:2
1) "email"
2) "tom@qq.com"
3) "name"
4) "tome"
5) "sex"
6) "man"
7) "old"
8) "3"
```

## HSTRLEN 

- 返回指定 key 的 value 的字符串长度
- 如果 value 或 哈希不存在，则返回0

```
> hstrlen user:2 name
(integer) 4

> hget user:2 name
"tome"
```