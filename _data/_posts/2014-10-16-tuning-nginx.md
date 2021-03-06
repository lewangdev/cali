---
layout:     post
title:      "Nginx 性能调优「译」"
subtitle:   "来自 Nginx 官方博客"
date:   2014-10-16 15:27:00
author:     "Le"
header_img: "img/post-bg-2015.jpg"
tags:
    - nginx
    - tuning
    - Linux
    - performance
---

> 这是一篇译文，原文链接：[Tuning NGINX for Performance](http://nginx.com/blog/tuning-nginx/)

Nginx 为人熟知的是在负载均衡、静态缓存和 WEB 服务器等方面的高性能，目前世界上最繁忙的站点中大约有 40% 在使用 Nginx。绝大多数情况下，大多数默认的 Nginx 和 Linux 配置都可以工作得非常好，但也需要做一些优化以获得最好的性能。本文将讨论在优化系统时需要考虑的 Nginx 和 Linux 的部分配置。可配置的选项有很多，但是本文只涵盖推荐大多数用户调整的配置选项。本文没有涵盖的配置选项，只有那些对 Nginx 和 Linux 有了深入的理解的人或者获得了 Nginx 技术支持和专业的服务团队的推荐建议后，才可以考虑调整。Nginx 专业服务器团队已经为世界上一些最繁忙的站点通过优化 Nginx 获得了最高水平的性能，并且可以为任何需要获得自己系统最大产出的客户服务。

## 简介

---

本文假设读者对 Nginx 架构和配置的概念已有了基本的了解。Nginx 的文档内容将不会在本文中重复，但本文会提供各项配置简要的介绍和相关文档的链接。

在性能调优时，要遵循一个好的规则：一次只修改一个配置选项，如果这个修改没有在性能方面带来优化，那么要再改回默认值。

我们从 Linux 性能优化的讨论开始，因为 Linux 性能优化的一些值会影响到 Nginx 的一些配置。

## Linux 配置

---

尽管现代 Linux 内核（2.6+）在各种配置情况下都工作得很好，但也有一些配置是想要修改的。如果操作系统的配置设置的太低，那内核日志将会有错误信息，从而得知哪些配置需要调整。Linux 性能优化可能涉及的配置有很多，这里我们只讨论那些优化达到正常工作负载最有可能涉及到的那些配置。调整这些配置请参考详细的 Linux
文档。

### Backlog 队列

下面的配置选项与网络连接和其排队方法直接相关。如果连入率很高（译者注：客户端发起的连接很多）且系统性能配置不匹配，例如一些连接表现得有所停顿，那么修改下面得配置将可能有用。

- net.core.somaxconn: 设置等待 Nginx 接受的连接队列的大小。由于 Nginx 接受连接非常的快，这个值通常情况下不用设置得很大，但系统默认值可能比较小，所以对于流量比较大的站点，增大这个值是个不错的想法。如果这个值太小，在内核日志中应该会看到错误消息，那么就需要增大这个值，直到错误消失。注意：若将这个值设置为大于 512 的话，那么需要在 Nginx 配置中修改 listen 指令的 backlog 参数来匹配这个数字。

- net.core.netdev_max_backlog: 设置数据包在被发送到 CPU 前可被网卡缓存的速率。对于带宽很大的机器来说，这个值需要增大。可以查阅网卡关于这项设置的建议文档或者查看内核日志中此项设置相关的错误。


### 文件描述符

文件描述符是用于处理例如连接和打开的文件等的操作系统资源。Nginx 在一个连接中使用文件描述符可以达到两个，例如 Nginx 做代理，那文件描述符一个用于客户端连接，另外一个用于代理服务器，但如果开启 HTTP 保持连接，那这个比例将会很低。对于一个连接数量很大很大的系统来说，这个值可能需要调整：

- sys.fs.file_max: 文件描述符的系统级限制。

- nofile: 文件描述符用户级限制，可以在 /etc/security/limits.conf 文件中修改。


### 临时端口范围(Ephemeral ports)

当 Nginx 用作代理，每一个到后端服务器的连接都会短暂的、临时的使用一个端口。

- net.ipv4.ip_local_port_range: 设置端口启始范围。如果观察到端口耗尽，那么需要增大这个范围。通常的端口范围设置是 1024 到 65000。

- net.ipv4.tcp_fin_timeout: 设置端口停止使用后可再次被其它连接使用所需要的时间。通常默认值是 60 秒，但通常减少到 30 秒或者 15 秒都是安全的

## Nginx 配置

---

下面是一些影响系统性能的 Nginx 指令。前面已经说过，本文只讨论一些推荐给大多数人调整的指令。其它没有提到的任何指令，若没有 Nginx 团队的建议，推荐不要修改。


### 工作进程

Nginx 可以运行多个工作进程，每一个都可以处理大量的连接。通过下面的指令可以控制运行工作进程的数量和每个进程处理连接的数量：

- [worker_processes](http://nginx.org/r/worker_processes):控制 Nginx 运行工作进程的数量。多数情况下，每一个 CPU 核运行一个工作进程的方式可以很好的工作。可以通过设置指令的值为 "auto" 来达到这个效果。但也有需要增大这个数字的时候，例如工作进程需要做很多的磁盘 IO 操作。默认值是 1。

- [worker_connections](http://nginx.org/r/worker_connections):这个值表示每一个工作进程可以同时处理连接的最大数目。默认是 512，但是大多数系统可以处理一个大得多的数字。这个值应该被设置成多少依赖与服务器的大小和网络流量的特征。通过具体的测试可以找到具体的值。


### 连接持久化(Keepalives)

连接持久化可以在创建和关闭连接过程中降低 CPU 和网络开销，从而可对性能产生较大的影响。Nginx 会终止所有客户端连接以及和客户端连接分离开且独立的后端服务器连接。Nginx 支持客户端连接和后端服务器连接的持久化，可以通过下面的指令设置客户端连接持久化：

- [keepalive_requests](http://nginx.org/r/keepalive_requests):一个客户端使用一个持久化连接发送的请求数，默认值是 100. 这个值可以设置得很大，尤其在在做压力测试过程中，单一客户端发送大量请求得情况下会特别有用。

- [keepalive_timeout](http://nginx.org/r/keepalive_timeout): 连接一旦空闲后还会保持多长时间（也就是空闲多长时间以后被关闭）。

下面的指令可以设置后端服务器连接的持久化：

- [keepalive](http://nginx.org/r/keepalive): 为每一个工作进程开启的到后端服务器空闲持久化连接的数量。这个指令没有默认值。

为了启用 Nginx 到后端服务器的持久化连接，需要添加一下指令：

- proxy_http_version 1.1;
- proxy_set_header Connection "";


### 访问日志

记录每个请求的访问日志会占用 CPU 和 I/O 周期，启用访问日志缓冲可以减小影响。启动访问日志缓冲会使得 Nginx 缓存一些列日志记录到缓存中，然后在同一时间将他们写到文件中，而不是分开的执行每一个写操作。启用访问日志缓存功能需要在 Nginx 配置文件的 access_log 指令中使用 "buffer=size" 选项。这个是设置将要使用的缓存大小。也可以设置 "flush=time" 选项来告诉 Nginx
多长时间来将缓存中的记录写到磁盘日志文件中。如果配置了这两个选项，那当日志放不进缓存（缓存满了）或者缓存中日志记录比 flush 参数设置的时间还要老时，Nginx 将会写日志记录到日志文件中。当工作进程重新打开日志文件或者工作进程被关闭时，日志记录都会被写到日志文件中。当然，完全关闭访问日志也是可以的。 

### Sendfile

Sendfile 是一个可以在 Nginx 中使用的操作系统的特性。这个特性可以使 TCP 数据传输得更快：它通过在内核中从一个文件描述符拷贝数据到另一个文件描述符，通常可以达到零拷贝。
Nginx 可以利用它通过 socket 发送缓存的或者磁盘上的内容，并且不需要任何的用户空间的上下文切换，从而使得发送数据的过程非常的快并且使用更少的 CPU 开销。由于数据从不触及用户空间，在处理链中添加需要访问数据的过滤器是行不通的，所以无法使用任何会改变数据内容的 Nginx 过滤器，例如 gzip 过滤器。Sendfile 默认是不启用的。


### 限制(Limits)

Nginx 和 Nginx Plus 允许设置一些列的限制项，可用于控制被客户端访问的资源，所以这些设置项会对系统性能产生影响，并且也会影响用户体验和安全。下面是其中的一些指令：

- [limit_conn](http://nginx.org/r/limit_conn)/[limit_conn_zone](http://nginx.org/r/limit_conn_zone):这两个指令可用于限制 Nginx 允许的连接数目，例如限制单一的客户端 IP 地址。可以阻止单个客户端建立过多连接消耗过多资源的情况发生。

- [limit_rate](http://nginx.org/r/limit_rate): 限制一个客户端单个连接的带宽大小。可以防止由特定的客户端造成系统高负载的情况发生，从而保证所有客户端都可以获得好质量的服务。

- [limit_req](http://nginx.org/r/limit_req)/[limit_req_zone](http://nginx.org/r/limit_req_zone):
这两个指令可以限制被 Nginx 处理的请求速率。和 limit_rate 一起使用 可以防止由特性的客户端造成系统高负载情况的发生，从而保证所有客户端都可以获得好质量的服务。这些指令可以用于改善系统安全，尤其是在登录页面，可以通过设置限制请求速率值，这个值完全胜任一个人类用户但又回减慢程序用户访问来提高系统的安全。（大概意思就是通过这个设置来限制机器程序的访问, 如爬虫）

- [max_conns](http://nginx.org/r/upstream): 设置允许同时连接到后端服务器组中一台服务器的最大连接数。这个设置可以防止后端服务器过载。默认值是 0， 表示无限制。

- [queue](http://nginx.org/r/queue): 如果设置了 max_conns, 并且当一个请求由于没有可用的后端服务器或者后端服务器已到达 max_conns 上限而不能被处理时，此时发生什么就会交给 queue 指令来管理。这个指令设置存放到队列中请求数以及请求等待超时时间。如果这个指令没有设置，请求将不会出现排队的情形。

## 附加配置

---

值得一提的是，Nginx 有一些附加的特性可以用来提高 Web 应用的性能，尽管他们不是真的属于调优的范畴，但对性能的影响也是显著的。下面将讨论其中的两个特性。

### 缓存

有一个用于负载均衡一组后端 Web 应用的 Nginx 实例，通过启用这个实例的缓存配置，将会显著的增加到客户端的响应时间，同时也会显著的降低后端服务器的负载。缓存本身就是一个主题，这里不再详细讨论。需要了解更多的关于配置 Nginx 缓存的信息，可以参考 [Nginx 管理指南 - 缓存篇](http://nginx.com/resources/admin-guide/caching/)


### 压缩

压缩返回给客户端的数据可以大大减小返回数据的大小，从而需求更少的带宽，但压缩的操作需要占用 CPU 资源。所以在减少带宽有价值时使用压缩才是最好的。值得注意的还有不要压缩已经压缩过的对象， 譬如 jpeg 图片等。需要了解更多的关于配置 Nginx 压缩的信息，可以参考[Nginx 管理指南 - 压缩篇](http://nginx.com/resources/admin-guide/compression-and-decompression/)



更多阅读:

- [Nginx 性能测试](http://pages.nginx.com/2014_04_Website_Whitepaper_BenchmarkingNGINX_12014_04_Website_Whitepaper_BenchmarkingNGINXLandingPage.html)
- [Nginx 文档](http://nginx.org/en/docs/)
- [Nginx 和 Nginx Plus 特性](http://nginx.com/products/feature-matrix/)
- [Nginx Plus 技术规范](http://nginx.com/products/technical-specs/)

