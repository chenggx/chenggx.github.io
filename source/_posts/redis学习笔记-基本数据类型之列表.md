---
layout: reids
title: redis学习笔记-----基本数据类型之列表
date: 2019-12-01 17:16:07
author: chenggx
img: http://static.xiangdangnian.net.cn/blog/long-exposure-photography-of-vehicle-lights-3171592%202.jpg
top: false
categories: redis
tags:
 - redis
---

# List 列表

## LPUSH、RPUSH、LRANGE

- lpush 可以将一个或多个值插入到列表的头部
- rpush 可以将一个或多个值插入到列表的头部
- lrange 从列表的头部查看列表，元素下标从 0 开始，-1 表示最后一个元素，-2 表示倒数第二个元素，以此类推

```
> rpush mylist A    //将 A 插入到 mylist 列表的尾部
(integer) 1

> rpush mylist B    //将 B 插入到 mylist 列表的尾部
(integer) 2

> lpush mylist first    //将 first 插入到 mylist 列表的头部
(integer) 3

> rpush mylist C D E    //将 C、D、E 插入到列表 mylist 的尾部
(integer) 6

> lrange mylist 0 -1    //从列表的头部查看列表的全部内容
1) "first"
2) "A"
3) "B"
4) "C"
5) "D"
6) "E"

> lrange mylist 0 2 //从列表的头部查看 mylist 列表，从 0 开始，到 2 结束（元素下标从 0 开始，-1 表示最后一个元素，-2 表示倒数第二个元素，以此类推）
1) "first"
2) "A"
3) "B"
```

## LPOP、RPOP

- lpop 移除并返回头部元素
- rpop 移除并返回尾部元素

```
> lrange mylist 0 -1
1) "first"
2) "A"
3) "B"
4) "C"
5) "D"
6) "E"

> lpop mylist       //移除头部元素，并返回该元素
"first"

> lrange mylist 0 -1
1) "A"
2) "B"
3) "C"
4) "D"
5) "E"

> rpop mylist   //移除尾部元素，并返回该元素
"E"

> lrange mylist 0 -1
1) "A"
2) "B"
3) "C"
4) "D"

> rpop abc  //当 abc 列表不存在时，返回 nil
(nil)
```

## LINDEX

- 返回列表指定下标的元素（下标从0开始，-1 为最后一个元素）

```
> lrange mylist 0 -1
1) "A"
2) "B"
3) "C"
4) "D"

> lindex mylist -1
"D"

> lindex mylist 2
"C"
```

## LTRIM

- 让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除。下标与之前介绍的写法都一致，这里不赘述

```
> lrange mylist 0 -1
1) "A"
2) "B"
3) "C"
4) "D"

> ltrim mylist 0 1
OK

> lrange mylist 0 -1
1) "A"
2) "B"
```