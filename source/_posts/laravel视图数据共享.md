---
title: laravel 视图数据共享
date: 2020-09-13 22:22:22
author: chenggx
top: false
categories: laravel
img: http://static.xiangdangnian.net.cn/blog/2020/09/14/21-30-35-d84e9c675dc1332aa70a59158eef496d-91c21f.jpg
tags:
  - laravel
---


在我们做网站的时候有些数据是每个视图页面都需要的（导航、侧边栏等内容），但如果我们在每个视图的控制器里面都写向视图传递数据的操作则会显得代码比较冗余。那么在 laravel 中我们一般可以使用 viewShare 和 viewComposer 的方式来进行视图页面数据的共享。

## viewShare

### 首先需要在 AppServiceProvider 中的 boot 方法中定义需要共享的数据。

app/Providers/AppServiceProvider.php

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Facades\View;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     *
     * @return void
     */
    public function register()
    {
        //
    }

    /**
     * Bootstrap any application services.
     *
     * @return void
     */
    public function boot()
    {
        View::share('sitename','xxx的网站');
    }
}
```

### 定义两个路由文件

routes/web.php

```php
<?php

use Illuminate\Support\Facades\Route;

Route::get('/product',function(){
    return view('product');
});

Route::get('/blog',function(){
    return view('blog');
});
```

### 在 blog 和 product 视图页使用共享的数据

resources/views/blog.blade.php

```php
{{$sitename}}

文章页
```

resources/views/product.blade.php

```php
{{$sitename}}

产品页
```

### 验证效果

![](http://static.xiangdangnian.net.cn/20200914105715.png)
![](http://static.xiangdangnian.net.cn/20200914105738.png)


## viewComposer

### 设置 composer 

有三种方式设置
- 使用新的provider

- 在 AppServiceProvider 中的 boot 方法使用基于 viewComposer 的闭包

- 在 AppServiceProvider 中的 boot 方法使用 viewComposer 成器

  #### 使用新的provider

  app/Providers/MenuComposerProvider.php

```php
<?php

namespace App\Providers;

use App\Http\View\Composers\MenuComposer;
use Illuminate\Support\Facades\View;
use Illuminate\Support\ServiceProvider;

class MenuComposerProvider extends ServiceProvider
{
    public function register()
    {
    }

    public function boot()
    {
        View::composer('menu',MenuComposer::class);
    }
}
```



#### 在 AppServiceProvider 中的 boot 方法使用基于 viewComposer 的闭包(逻辑比较简单的话使用该方法)

app/Providers/MenuComposerProvider.php

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Facades\View;

class AppServiceProvider extends ServiceProvider
{
    public function register()
    {
    }
  
    public function boot()
    {
        // 使用基于合成器的闭包
        View::composer('menu', function ($view) {
                $view->with('list',['首页', '文章', '产品']);
            }
        );
    }
}
```



#### 在 AppServiceProvider 中的 boot 方法使用 viewComposer 

app/Providers/MenuComposerProvider.php

```php
<?php

namespace App\Providers;

use App\Http\View\Composers\MenuComposer;
use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Facades\View;

class AppServiceProvider extends ServiceProvider
{
    public function register()
    {
    }

    public function boot()
    {
        View::composer('menu',MenuComposer::class);
    }
}
```



### 注册 Provider （后两种方式不用注册）

config/app.php

```php
...
  'providers' => [
  ...
        App\Providers\MenuComposerProvider::class
    ],
...
```



### 创建 MenuComposer

app/Http/View/Composers/MenuComposer.php

```php
<?php


namespace App\Http\View\Composers;


use Illuminate\View\View;

class MenuComposer
{
    public function compose(View $view)
    {
        $view->with('list', ['首页', '文章', '产品']);
    }
}
```

### 创建 menu 视图文件

resources/views/menu.blade.php

```php
<ul>
    @foreach($list as $menu)
        <li>{{$menu}}</li>
    @endforeach
</ul>
```



### 设置路由

routes/web.php

```php
<?php

use Illuminate\Support\Facades\Route;

Route::get('/product',function(){
    return view('product');
});

Route::get('/blog',function(){
    return view('blog');
});
```



### 在 product、blog 视图文件中引入 menu 视图

resources/views/product.blade.php

```php
@include('menu')
产品页
```

resources/views/blog.blade.php

```php
@include('menu')

文章页
```

### 查看效果

![](http://static.xiangdangnian.net.cn/20200914143225.png)

![](http://static.xiangdangnian.net.cn/20200914143247.png)


![程序员的艺术人生](http://static.xiangdangnian.net.cn/4vB3ZRunrw8VzSW6cLpJyaibKfYTNCU0.png)
