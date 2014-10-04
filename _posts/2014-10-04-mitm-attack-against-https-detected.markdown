---
layout: post
title: 访问 Bing 的时候被中间人攻击了
tags:
    - MITM
    - HTTPS
    - SSL
    - TLS
    - 证书
comments: true
---

今天在打开 `https://cn.bing.com` 遇到中间人攻击了。伪造证书的是一个 hotmal.com 的自签名证书。

![MITM Attack against Bing Detected]({{ site.url }}/images/2014-10-04-mitm-attack-against-bing-detected.png)

也必须顺便吐槽一下 Safari 对证书错误的处理，只有一个很不明确的提醒，还把页面内容显示出来了。这要是在 Firefox 或者 Google Chrome 上直接就大红叉禁止访问了。
