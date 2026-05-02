---
layout: laravel
title: laravel通过关联模型实现无限极分类
date: 2020-01-15 19:28:59
author: chenggx
img: http://static.xiangdangnian.net.cn/blog/kuala-lumpur-1820944_1280.jpg
top: false
categories: mysql
tags: 
  - php
  - laravel
---


# Laravel 通过关联模型实现无限极分类

> 这个内容是在 Laravel-China 论坛上看到的，怕以后不好找，这里记录一下。原文地址为：[https://learnku.com/articles/14068/simple-practice-of-laravel-infinite-class-classification](https://learnku.com/articles/14068/simple-practice-of-laravel-infinite-class-classification)

## 数据库结构

这里使用省市区结构

```php

class CreateAreasTable extends Migration
{
    .
    .
    .

    public function up()
    {
        Schema::create('areas', function (Blueprint $table) {
            $table->unsignedInteger('id');
            $table->string('name')->comment('城市名称');
            $table->unsignedInteger('pid')->default(0)->comment('父级id');
        });
    }
    .
    .
    .
}

```

## Area 模型增加关联方法

```php

class Area extends Model
{
    .
    .
    .

    public function childArea() {
        return $this->hasMany(Area::class,'pid','id');
    }

    public function allChildArea()
    {
        return $this->childArea()->with('allChildArea');
    }
    
    .
    .
    .

}

```

## 测试

> 通过如下代码可以得到所有地区的无限极分类结构。更改条件可以查看某个地区及其子地区的无限极分类结构

```php

$res = Area::with('allChildArea')->where('pid',0)->get();
return $res;

```