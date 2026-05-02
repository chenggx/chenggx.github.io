---
layout: redis
title: redis学习笔记-----基本数据类型之有序集合
date: 2019-12-26 20:44:19
author: chenggx
img: http://static.xiangdangnian.net.cn/blog/5cf317e5f53f34ab0b141e3ca0b3f2eff04f92ab.jpg
top: false
categories: redis
tags: 
  - redis
---

# 有序集合（sort sets）

- 与集合类型，区别是集合不能字段排序，而有序集合可以设置额外的优先级(score)参数来为成员排序。
- 使用场景：当你需要一个有序并且不重复的集合列表的时候，使用有序集合。 

## ZADD 
- 将一个或多个元素以及其 score 值加入到 key 中。
- 如果某个元素已经在有序集合中，则更新该元素的 score 值。
- score 值可以是整数或双精度浮点数。

```
127.0.0.1:6379> zadd language 100 php 90 java 80 js
(integer) 3
```

## ZCARD

- 返回有序集合 key 的值得个数
- 如果 key 不存在，则返回 0

```
127.0.0.1:6379> zcard language
(integer) 3

127.0.0.1:6379> zcard xxxxx
(integer) 0
```

## ZCOUNT 

- 返回有序集合 key 中，score 的值在 min 和 max 直接的成员个数。（包含 min 和 max）
- 如果在统计时，不需要包含某个 score时 ，则添加一个 ( 即可。
```
127.0.0.1:6379> zcount language 80 100
(integer) 3

127.0.0.1:6379> zcount language 80 (100
(integer) 2
```

## ZSCORE
 
- 返回有序集合 key 中，成员的score值

```
127.0.0.1:6379> zscore language php
"100"
```

## ZRANGE

- 格式：zrange key start stop [withscores]
- 根据索引返回元素
- withscores 参数可以连同元素的 score 一起返回。
- start 为 0，stop 为 -1，即可返回整个有序集合。

```
127.0.0.1:6379> zrange language 0 2
1) "js"
2) "java"
3) "php"

127.0.0.1:6379> zrange language 0 2 withscores
1) "js"
2) "80"
3) "java"
4) "90"
5) "php"
6) "100"
```

## ZREVRANGE

- 同 zrange 基本一致，区别是 zrevrange 是反者来的。

```
127.0.0.1:6379> zrevrange language 0 2
1) "php"
2) "java"
3) "js"

127.0.0.1:6379> zrevrange language 0 2 withscores
1) "php"
2) "100"
3) "java"
4) "90"
5) "js"
6) "80"
```

## ZRANGEBYSCORE

- 返回有序集合 key 中，score 的值在 min 和 max 直接的成员。（包含 min 和 max）
- 如果不需要包含某个 score时 ，则添加一个 ( 即可。

```
127.0.0.1:6379> zrangebyscore language 80 100
1) "js"
2) "java"
3) "php"

127.0.0.1:6379> zrangebyscore language 80 100 withscores
1) "js"
2) "80"
3) "java"
4) "90"
5) "php"
6) "100"

127.0.0.1:6379> zrangebyscore language 80 (100
1) "js"
2) "java"
```

## ZRANK

- 返回有序集合 key 中成员 member 在集合中的排名序号。（排名按 score 值从小到大的顺序）。
- 排名序号从 0 开始。(即 序号为 0 的 score 值最小)

```
127.0.0.1:6379> zrank language php
(integer) 2

127.0.0.1:6379> zrank language js
(integer) 0

127.0.0.1:6379> zrank language java
(integer) 1
```

## ZREVRANK 
- 同上面的 zrank。区别是 zrevrank 的排序是从大到小

```
127.0.0.1:6379> zrevrank language php
(integer) 0

127.0.0.1:6379> zrevrank language js
(integer) 2

127.0.0.1:6379> zrevrank language java
(integer) 1
```

## ZINCRBY

- 格式：zincrby key increment member
- 向有序集合 key 中的成员 member 的 score 值进行增加。
- 如果 key 中不存在 member，则在 key 中创建一个 member，并且该 member 的 score 为 设置的值。

```
127.0.0.1:6379> zincrby language 100 php
"200"

127.0.0.1:6379> zincrby language 100 python
"100"

```

## ZREM

- 从集合中删除指定 member。

 ```
 127.0.0.1:6379> zrem language python
(integer) 1

127.0.0.1:6379> zrange language 0 -1 withscores
1) "js"
2) "80"
3) "java"
4) "90"
5) "php"
6) "200"
 ```