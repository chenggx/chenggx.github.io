---
title: Laravel学习笔记
date: 2019-06-18 23:11:32
author: chenggx
img: http://static.xiangdangnian.net.cn/blog/laravel.jpg
top: false
categories: php
tags:
 - laravel
 - php
---

## 第一步：了解几个常用名称

> 代码基于 Laravel-5.8 部分内容给其他版本不一致，但整体逻辑是没有太大差别的。

> Service Container、Service Provider、Facades、Contracts。当第一次使用 Laravel 框架的时候被这几个词直接搞蒙圈了，完全不知道是什么意思，尤其是在看完文档之后，真的想说 WTF 。不过不知道也不会太影响使用。（😅有个朋友说过，不管用着在 NB 的框架，依然可以写出屎一样的代码）

### Service Container
中文解释：服务容器 （😂 是不是看完这个翻译更不知道是啥了）

我的理解：一个装着各种各样的服务实例的容器。这些服务就是 Laravel 中的各种模块，如 Redis、Route 等等。

实现一个简单的容器：
```php
class Container
{
  protected $binds;

  protected $instances;

  public function bind($abstract, $concrete)
  {
    //Todo: 向 container 添加一种对象的的生产方式

    //$abstract: 第一个参数 $abstract, 一般为一个字符串(有时候也会是一个接口), 当你需要 make 这个类的对象的时候, 传入这个字符串(或者接口), 这样make 就知道制造什么样的对象了
    //$concrete: 第二个参数 $concrete, 一般为一个 Closure 或者 一个单例对象, 用于说明制造这个对象的方式

    if ($concrete instanceof Closure) {
      $this->binds[$abstract] = $concrete;
    } else {
      $this->instances[$abstract] = $concrete;
    }
  }

  public function make($abstract, $parameters = [])
  {
    //Todo: 生产一种对象

    //$abstract: 在bind方法中已经介绍过
    //$parameters: 生产这种对象所需要的参数

    if (isset($this->instances[$abstract])) {
      return $this->instances[$abstract];
    }

    array_unshift($parameters, $this);

    return call_user_func_array($this->binds[$abstract], $parameters);
  }
}
```
#### 为什么理解 IOC Container 对于理解 Laravel 架构是如此的重要？

> 因为在 Laravel 中，你所能使用到的 Laravel 的特性和功能几乎全部是由 IOC Container 实现的。

```php
Cache::get('key'); 
Route::get('/', 'HomeController@index');
```
上面的 Cache、Route 都是通过把各自类的实现 bind 到 Container 中，然后 Container make 出的一个实例。
那么它们是在哪里进行的 bind 呢？没错就是 Service Provider 中。

### Service Provider

在 Laravel 中有两种方式来使用 IOC Container：
1. 通过 Service Provider
2. 不通过 Service Provider
> 一般情况都是使用第一种方式。

#### 不通过 Service Provider 来使用 IOC Container

实例：

```php
//web.php


//创建一个类
class Apple
{
    public $id = 123;
}

Route::get('/', function () {

  //使用 app() 辅助方法将 Apple 类 bind 到 IOC Container 中  
  App::bind('apple',function(){
      return new Apple();
  });

  //使用 App() 辅助方法将 Apple 实例 make 出来
  $apple = App::make('apple');
  
  //使用 Apple 类中的属性
  return $apple->id;
});
```

#### 通过 Service Provider 来使用 IOC Container
> 原因：我们知道，有时候我们的类、模块会有需要其他类和组件的情况，为了保证初始化阶段不会出现所需要的模块和组件没有注册的情况，laravel 将注册和初始化行为进行拆分，注册的时候就只能注册，初始化的时候就是初始化。拆分后的产物就是现在的 服务提供者。
可以想象这样一个场景，你要绑定 3 个类 A B C 到 IOC Container 中。 A，B，C 都是非常复杂的类。在 bind A 时，引用了一个类 B 的实例，那么想要获得类 B 的实例，就需要 B 已经被 bind，只有这样，我们的 IOC Container 才有能力 make 出一个 B 的实例。 而在 bind B 时，恰好又需要 C 的实例.
如果是这样的逻辑，那么在 bind A B C 时，就必须手动的严格安排 bind 的次序，而且这只是 3 个类的情况，如果有几十个类的话，人工已经无法完成了.
而这时就需要 Service Provider 的作用了。
>> 引用自——[从 1 行代码开始，带你系统性地理解 Service Container 核心概念
](https://learnku.com/laravel/t/3361/starting-with-the-1-line-of-code-with-a-systematic-understanding-of-the-core-concepts-of-service-container#e9b441)

```php
<?php
// 1. 创建一个 Apple 类 app/Test/Apple.php

namespace App\Test;

class Apple
{
    public $id = 'agg';
}

//2. 使用 php artisan make:provider AppleProvider 生成 provider

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use App\Test\Apple;

class AppleProvider extends ServiceProvider
{
    /**
     * Register services.
     *
     * @return void
     */
    public function register()
    {
        $this->app->singleton('apple',function($app){
            return new Apple();
        });
    }

    /**
     * Bootstrap services.
     *
     * @return void
     */
    public function boot()
    {}
}

// 3. 在 config/app.php 中注册 APPServiceProvider
...
'providers' => [
  ...
    App\Providers\AppleProvider::class,
  ...
    ],
...

// 4. 最终在 route/web.php 中使用

Route::get('/', function () {
   var_dump(app('apple'));
   var_dump(app('apple'));
});
/*
输出结果，因为使用了单例的模式进行绑定所以输出的两个实例的编号是一致的
object(App\Test\Apple)#206 (1) { ["id"]=> string(3) "agg" } 
object(App\Test\Apple)#206 (1) { ["id"]=> string(3) "agg" }
*/
```

### Facades

```php
Route::get('/', 'HomeController@index');
```
> 想必大家在第一使用 Laravel 的时候一定会注意到这行代码。在使用 phpStorm 之类的 IDE 工具是无法跳转到对应的类文件。那么这是为啥呢？

你是无法找到对 Route 类的声明的，是因为使用了别名。别名是 PHP 的一个特性（ [class_alias](https://www.php.net/manual/zh/function.class-alias.php) 方法 ）。

别名在哪里设置呢——app/config/app.php
```php
'aliases' => [
  ...
  'Route' => Illuminate\Support\Facades\Route::class,
  ...
```

查看 *vendor/laravel/framework/src/Illuminate/Support/Facades/Route.php* 文件....哎？怎么只有这么点东西，get 呢？
```php
namespace Illuminate\Support\Facades;

/*
   通过这行注释可以找到真实的 Route 类在什么地方
 * @see \Illuminate\Routing\Router
 */
 
class Route extends Facade
{
    /**
     * Get the registered name of the component.
     *
     * @return string
     */
    protected static function getFacadeAccessor()
    {
        return 'router';
    }
}
```

> 首先要知道 Facade 的作用：使用一个简单的语法，让你从 Laravel 的 IOC Container 中方便的 make 出你想要的对象。

#### Facade 实现原理（make）

1. 首先 Route 继承自 ``` Illuminate\Support\Facades ```。
2. ``` Illuminate\Support\Facades ``` 里也没有 get 静态方法。
3. 使用了 [__callStatic()](https://www.php.net/manual/zh/language.oop5.overloading.php#object.callstatic) 魔术方法。
4. 在 Route 中调用静态的 get 方法，实际上 Illuminate\Support\Facades 中的 ``` __callStatic ``` 魔术方法被调用。
5. 在 [__callStatic()](https://www.php.net/manual/zh/language.oop5.overloading.php#object.callstatic) 方法中调用了静态的 getFaRoot 方法。
6. 在 ``` getFacadeRoot ``` 方法中调用了静态的 ``` resolveFacadeInstance ``` 方法。
7. ``` resolveFacadeInstance ``` 方法需要传递一个参数，该参数通过静态 ``` getFacadeAccessor ``` 方法获得。
8. ``` getFacadeAccessor``` 该方法在 Route 类中实现了 ``` getFacadeAccessor ```并返回 route。
9. ``` resolveFacadeInstance``` 方法接收 route 字符串参数。首先判断是否为对象，当然不是啦，$name 是字符串。 然后判断该 ``` resolvedInstance ``` 数组中是否存在 route 相关信息，因为我们的程序是第一次运行当然也是没有的。 最后返回 ``` static::$app['route'] ```,同时把结果保存到 ``` resolvedInstance``` 数组中。
10. $app 其实就是前面说的 Application 类的实例对象，这个类是一个 IOC Container，实例化过程在 Laravel 最开始的时候。Facade 初始化的时候也让自己有了一个 $app,这个就是 Application 类的实例化对象。
11. 其实此时 $app 中并没有 'route' 属性，那么为什么可以调用呢？因为 Application 继承了 Container, 而 Container 又继承了 ArrayAccess 类,并实现了 offsetGet 方法。 该方法的内容为 ``` return $this->make($key);``` 这里就很明显了，直接make 出了一个 route 实例。
12. 最终相当于 ```$instance = static::getFacadeRoot(); ``` 与 ``` $instance = $app->make('router'); ``` 是相等的。

```php
    //Illuminate\Support\Facades\Facades.php;

	//以上内容省略
	....
	
    public static function getFacadeRoot()
    {
        return static::resolveFacadeInstance(static::getFacadeAccessor());
    }
    
     //该方法被子类复写
    protected static function getFacadeAccessor()
    {
        throw new RuntimeException('Facade does not implement getFacadeAccessor method.');
    }

    protected static function resolveFacadeInstance($name)
    {
        if (is_object($name)) {
            return $name;
        }

        if (isset(static::$resolvedInstance[$name])) {
            return static::$resolvedInstance[$name];
        }

        return static::$resolvedInstance[$name] = static::$app[$name];
    }
   
   //以下内容省略 
	....
	
    public static function __callStatic($method, $args)
    {
        $instance = static::getFacadeRoot();

        if (! $instance) {
            throw new RuntimeException('A facade root has not been set.');
        }

        return $instance->$method(...$args);
    }
```


#### Facade 实现原理（bind）

> 在最开始就了解到，既然要 make，必须首先 bind。而且最好的方式是通过 serviceProvider 来 bind 类。 而且不论是 make 还是 bind 都需要一个 key，用来在容器中保存和查找这个类。上面讲的是使用 route 关键字进行 make 的过程。那么我们可以肯定在之前一定有一个使用 route 进行 bind 的操作。下面就进行 bind 的讲解。

1. 首先 ServiceProvider 需要在 ``` config/app.php ``` 进行注册，我们在文件中找到对应 route 相关的内容，就是 ``` App\Providers\RouteServiceProvider::class, ```。
