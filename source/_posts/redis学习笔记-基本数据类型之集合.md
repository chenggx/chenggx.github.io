---
layout: reids
title: redis学习笔记-----基本数据类型之集合
date: 2019-12-10 19:43:58
author: chenggx
img: http://static.xiangdangnian.net.cn/blog/59b775569da2a.jpg
top: false
categories: redis
tags:
 - redis
---

# 集合（Set）

> 与 list 类似，区别是 set 中的值是不重复的。

## SADD 
- 将一个或多个member元素加入到集合key当中，已经存在于集合的member元素将被忽略。
- 假如key不存在，则创建一个只包含member元素作成员的集合。
- 当key不是集合类型时，返回一个错误。

```
> sadd k1 v1 v2 v3
(integer) 3
```

## SMEMBERS 
- 返回集合key中的所有成员。

```
> smembers k1
1) "v2"
2) "v3"
3) "v1"
```

## SREM 
- 移除集合key中的一个或多个member元素，不存在的member元素会被忽略。
- 当key不是集合类型，返回一个错误。

```
> srem k1 v2 v1
(integer) 2

> smembers k1
1) "v3"
```

## SISMEMBER
- 判断 member 元素是否是集合 key 的成员。
- 返回 1，则 member 元素在集合中。返回 0，则不在集合中。

```
> sismember k1 v3
(integer) 1

> sismember k1 v2
(integer) 0
```

## SCARD
- 返回集合key的基数(集合中元素的数量)。


```
> scard k1
(integer) 1

> sadd k1 v4 v5
(integer) 2

> scard k1
(integer) 3
```

## SMOVE
- 将 member 元素从 source 集合移动到 destination 集合。
- 如果 source 集合不存在或不包含指定的 member 元素，则命令不执行任何操作，仅返回0。
- 当 destination 集合已经包含 member 元素时，则命令只是简单地将 source 集合中的 member 元素删除。
- 当source或destination不是集合类型时，返回一个错误。

```
> smembers k1
1) "v4"
2) "v5"
3) "v3"

> smembers k2
(empty list or set)

> smove k1 k2 v4
(integer) 1

> smembers k1
1) "v5"
2) "v3"

> smembers k2
1) "v4"
```

## SPOP
- 移除并返回集合中的一个随机元素。
- 被移除的随机元素。
- 当key不存在或key是空集时，返回nil。

```
> smembers k1
1) "v5"
2) "v3"

> spop k1
"v3"

> smembers k1
1) "v5"

> spop k1
"v5"

> smembers k1
(empty list or set)

> spop k1
(nil)
```

## SRANDMEMBER
- 返回集合中的一个随机元素。(不删除)
- 返回值为被选中的随机元素。 当 key 不存在或 key 是空集时，返回nil。

```
> sadd k1 v1 v2 v3
(integer) 3

> smembers k1
1) "v2"
2) "v3"
3) "v1"

> srandmember k1
"v3"

> smembers k1
1) "v2"
2) "v3"
3) "v1"
```

## SINTER

- 返回给定的集合的交集。
- 当给定集合当中有一个空集时，结果也为空集(根据集合运算定律)。


```
> smembers k1
1) "v2"
2) "v3"
3) "v1"

> smembers k2
1) "v4"
2) "v1"
3) "v2"
4) "v5"
5) "v6"

> sinter k1 k2
1) "v2"
2) "v1"
```

## SINTERSTORE

- 基本等同于 sinter 命令，但它将结果保存在一个新的集合里。
- 如果要保存的集合已经存在，则将其覆盖。
- 返回结果集合中成员的数量。


```
> smembers k1
1) "v2"
2) "v3"
3) "v1"

> smembers k2
1) "v4"
2) "v1"
3) "v2"
4) "v5"
5) "v6"

> sinterstore k3 k1 k2
(integer) 2

> smembers k3
1) "v2"
2) "v1"
```

## SUNION
- 返回给定的所有集合的并集。

```
> smembers k1
1) "v2"
2) "v3"
3) "v1"

> smembers k2
1) "v4"
2) "v1"
3) "v2"
4) "v5"
5) "v6"

> sunion k1 k2
1) "v4"
2) "v3"
3) "v1"
4) "v2"
5) "v5"
6) "v6"
```

## SUNIONSTORE
- 基本等同于 sunion 命令，但它将结果保存在一个新的集合里。
- 如果要保存的集合已经存在，则将其覆盖。
- 返回结果集合中成员的数量。

```
> smembers k1
1) "v2"
2) "v3"
3) "v1"

> smembers k2
1) "v4"
2) "v1"
3) "v2"
4) "v5"
5) "v6"

> sunionstore k4 k1 k2
(integer) 6

> smembers k4
1) "v4"
2) "v3"
3) "v1"
4) "v2"
5) "v5"
6) "v6"
```

## SDIFF
- 返回给定的所有集合的差集。

```
> smembers tom
1) "orange"
2) "banner"
3) "apple"

> smembers jack
1) "pants"
2) "t-shirt"
3) "apple"

> sdiff tom jack
1) "orange"
2) "banner"
```

SDIFFSTORE
- 基本等同于 sdiff 命令，但它将结果保存在一个新的集合里。
- 如果要保存的集合已经存在，则将其覆盖。
- 返回结果集合中成员的数量。

```
> smembers tom
1) "orange"
2) "banner"
3) "apple"

> smembers jack
1) "pants"
2) "t-shirt"
3) "apple"

> sdiffstore new tom jack
(integer) 2

> smembers new
1) "orange"
2) "banner"
```