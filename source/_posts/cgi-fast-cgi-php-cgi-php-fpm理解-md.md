---
title: CGI, Fast-CGI，PHP-CGI，PHP-FPM 几个概念的总结
date: 2019-01-24 21:43:13
author: chenggx
img: http://static.xiangdangnian.net.cn/blog/timg.jpeg
top: false 
categories: php
tags:
  - php
---

> 2020-2-6 更新

** 最近又看到一篇[文章](https://segmentfault.com/q/1010000000256516)，讲解的非常到位，在更正一下之前的结论  **

1. cgi、fast-cgi 都是协议，规定了 web server 需要传递那些数据给 php 解释器。他们的区别是 cgi 每次都需要加载配置文件来初始化，而 fast-cgi 会先启一个master，解析配置文件，初始化执行环境，然后再启动多个worker。当请求过来时，master会传递给一个worker，然后立即可以接受下一个请求。这样就避免了重复的劳动，效率自然是高。

2. php-cgi 相当于 php 的一个解释器是一个具体的程序，也就是系统中的进程。而 php-fpm 是一个实现了 fast-cgi 的具体程序，用来管理 php-cgi 进程。

----

> 2019-6-27 更新

** 最近又看到了类似的文章，发现之前记录的好像是不太对。现在也不知道哪个是正确的，就都先记下来吧。** 

## 结论
1. CGI、FastCGI 只是PHP的的运行方式。其他的还有APACHE2HANDLER、CLI 方式。
2. PHP-FPM（FastCGI Process Manager） 只是 FastCGI 的进程管理器。
3. PHP-CGI 只是 Fast-CGI 的子进程。


### 参考链接
[php 的4种常见运行方式](https://www.jb51.net/article/62554.htm)



------


> 在开始学php的时候有几个概念一直没有弄明白，最近查了些资料，特此补充记录一下，以防忘记。有不对的地方还请高手指点。

## 结论
- CGI：WEB 服务器与 WEB 应用程序之间交换数据的一种协议。
- FastCGI：同 CGI 一样，也是一种协议，只是在效率上比 CGI 好一些。
- PHP-CGI：fastCGI 协议的一种实现。(也就是 php 可执行目录下的php-cgi程序)。他有2个问题。
    - 更改配置文件后无法平滑重启。
    - 无法动态调整进制多少。
- spawn-fcgi：解决了 php-cgi 出现的问题。但器仅仅是一个进程管理器。
- PHP-FPM：实现了 Fast-CGI 协议并且之前平滑重启，同时还带有进程管理功能。

### 参考资料

[CGI 协议内容](https://www.ietf.org/rfc/rfc3875)

[Fast-CGI 协议内容](http://andylin02.iteye.com/blog/648412)

[从CGI到FastCGI到PHP-FPM](http://yongxiong.leanote.com/post/%E4%BB%8ECGI%E5%88%B0FastCGI%E5%88%B0PHP-FPM)

[CGI、FastCGI和PHP-FPM关系图解](https://www.awaimai.com/371.html)
