---
layout: post
title: 在 Nginx 中拒绝代理请求
tags:
    - nginx
    - http
    - proxy
    - security
comments: true
---

在网络上会遇到很多针对开发代理的扫描，这样的请求大概是

```
GET http://www.baidu.com/ HTTP/1.1
Host: www.baidu.com

```

除非特意去配置，nginx 一般都不会成为这样的正向代理。但是在收到这种请求时，nginx 很可能就把自己的主页 index.html 发送出去了。这样虽然没有什么安全问题，但是却会因为大量的代理扫描而浪费带宽和流量。

HTTP/1.1 要求所有服务器（即使不是代理服务器），处理 absolute URI 请求；虽然 HTTP/1.1 客户端只会向代理服务器发送请求时才使用 absolute URI. Nginx 遵守了这个规范，所以上面的请求会被识别为一个正常的 HTTP 请求，于是 nginx 就按照主机名去匹配各个 `server {}` 里面的 `server_name`. 这时，一般是找不到匹配的。

在没有匹配到虚拟主机时，就会 fallback 到 default\_host 上。在没有指定 default\_host 时，在配置文件中出现的第一个 `server {}` 就是 default\_host. 于是你的网站首页就被返回了。

要解决这个问题，就应该另外指定一个 default\_host, 并在其中拒绝掉所有请求。即，在原来的第一个 `server {}` 小节之前，加入

```
server {
    deny all;
}
```

一个完整的 `/etc/nginx/conf.d/default` 大概如下：

```
server {
    # reject unknown host
    deny all;
}

server {
    server_name www.example.com;

    root /var/www/html/;
    index index.html;
}
```
