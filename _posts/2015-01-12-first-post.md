---
layout: post
title:  "一次Nginx 负载不均调查过程"
date:   2015-12-14 22:31:05
author: sunrainchy
categories: Nginx
image: true
---


最近在做在对产品进行极限压力测试的时候发现一个奇怪的问题
用异步压力并发超过10（只是十万个并发去请求，QPS 大约在 7W 左右）的时候Nginx 表现异常，只有一个进程在不断的accept 连接，已经把单核CPU打满，而其余进程没事干，如下：
<div class="post-img">
<img class="img-responsive img-post" src=" {{site.baseurl}}/img/ngx_top_20151214.png"/>
</div>
pstack 一下看看这个最忙的进程在干啥：
<div class="post-img">
<img class="img-responsive img-post" src=" {{site.baseurl}}/img/nginx-pstack-20151214.png"/>
</div>
看样子好像是一直在accept 连接，close 连接

   为什么这个样子，猜想可能是Nginx负载均衡那里出问题了，于是查了查资料翻了翻Nginx源码，源码看看负载均衡那块。Nginx accept 会在进程间上一个锁，保同一时间只会有一个进程去accept连接，在epoll_wait 返回之后会将事件分成两类，分别放到两个post队列中，接下来Nginx 优先处理accept事件，好让ngx_accept_mutex 锁尽快释放让其他进程能尽快拿到这把锁去accept连接，开始怀疑是这块问题，在压力大的时候这里锁会一直hold 住直到所有的accept 事件被处理完（epoll 的 ready事件队列中也没有accept事件），翻了源码，这样写的：

<div class="post-img">
<img class="img-responsive img-post" src=" {{site.baseurl}}/img/ngx-accept-code-20151214.png"/>
</div>

259 行，每次处理完accept 事件后会去释放ngx_accept_mutex, 那按照道理是不会造成ngx_mutex_lock hold 住的情况的，一次epoll_wait 返回处理完所有Accept事件后一定会把ngx_mutex_lock 释放掉。

看来不是这个原因，又回到Nginx现象中去，一个进程一直在accept连接，其他进程根本就没有Accept连接的机会，于是想到Nginx 配置中的multi_accept 配置项，翻开一看，multi_accept on;  注掉这项配置重启Nginx，发现问题不见了，Nginx负载均衡了，原来是这个东西搞的鬼。于是又翻翻源码看了下Nginx是如何accept 连接的<br>
 void
 ngx_event_accept(ngx_event_t *ev)
Accept 调用处在一个循环里面：
```
	do {
          // More code here
  	      s = accept(lc->fd, (struct sockaddr *) sa, &socklen);
          // More code here
 	} while (ev->available);
```
看看 ev->available 是什么鬼<br>
在代码里grep multi_accept :<br>
```
./event/ngx_event_accept.c:        ev->available = ecf->multi_accept;
```
果然是这个东西搞得鬼。

原因总结：
   Nginx配置文件中配置了multi_accept on; 含义是：Nginx在已经得到一个新连接的通知时,接收尽可能更多的连接,这样在Nginx Accept 过程中一直有新的Accept 请求到来，导致单进程把单核CPU打满也接收不完Accept，
导致进程一直hang在这里Accept新连接，不释放ngx_accept_mutex 造成负载不均。
解决方法：<br>
1、关掉 multi_accept on;<br>
2、在 void ngx_event_accept(ngx_event_t *ev)
   中每用Accept循环中加一计数器，一次性Accept连接数超过一定值的时候就停止新Accept连接，break出去。

