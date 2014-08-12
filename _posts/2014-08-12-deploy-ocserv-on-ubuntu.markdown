---
layout: post
title: 在 Ubuntu 14.04 上部署 ocserv
tags:
    - OpenConnect
    - AnyConnect
    - ocserv
comments: true
---

## 背景

[OpenConnect VPN Server](http://www.infradead.org/ocserv/)(ocserv) 是一个 SSL VPN 服务器，使用 Cisco AnyConnect 的协议。

### 协议简述

Cisco AnyConnect 的协议是基于 HTTPS 和 DTLS 的，服务器需要监听一个 TCP 端口和一个 UDP 端口。TCP 端口用于传输握手、控制信息，并作为备用数据通道，由 TLS 保护；UDP 端口是数据通道，由 DTLS 保护。客户端的身份认证可以使用用户名/口令或者数字证书。下面简要描述使用用户名/口令认证时的握手过程。

在客户端，用户需要指定 VPN 服务的 URL. 客户端会与 URL 指定的主机/端口建立 TLS 连接，然后 HTTP GET 指定的 URL. 服务器会返回一个 XML, 要求认证。我们可以通过浏览器直接打开这个 URL 来查看这个 XML. 下面是 ocserv 返回的 XML.

{% highlight xml %}
<config-auth client="vpn" type="auth-request">
    <version who="sg">0.1(1)</version>
    <auth id="main"><message>Please enter your username</message>
        <form method="post" action="/auth">
            <input type="text" name="username" label="Username:"/>
        </form>
    </auth>
</config-auth>
{% endhighlight %}

这时客户端会提示输入用户名，然后这个用户名会按照 form 元素中的描述，被 POST 到 /auth. 接下来服务器会用类似的方式要求口令。在验证通过后，客户端会发出 HTTP CONNECT 请求，服务器回应 200 CONNECTED, 并在 header 中给出分配的 IP, DNS, route, MTU 等参数和数据通道的 UDP 端口号，以及用于保护数据通道的 DTLS session ID, cipher suite.

接下来再通过 session resumption 建立起受保护的 DTLS 通道，用于传输 VPN 数据。

## 安装

### 下载源代码

{% highlight sh %}
wget ftp://ftp.infradead.org/pub/ocserv/ocserv-0.8.2.tar.xz
tar -xf ocserv-0.8.2.tar.xz
cd ocserv-0.8.2/
{% endhighlight %}

### 安装编译依赖

{% highlight sh %}
aptitude install libgnutls28-dev libwrap0-dev libpam0g-dev libseccomp-dev libreadline-dev linnl-route-3-dev libprotobuf-c0-dev libhttp-parser-dev libpcl1-dev libopts25-dev autogen
{% endhighlight %}

### 编译安装

{% highlight sh %}
./configure && make && make install
{% endhighlight %}

## 配置

在源码包中有例子 doc/sample.config, 可以在这个文件的基础上修改。

{% highlight sh %}
mkdir /etc/ocserv
cp doc/sample.config /etc/ocserv/config
{% endhighlight %}

在这个文件中，关键的参数有：

* auth

    认证方法，可取值 "certificate", "pam" 或者 "plain[ocpasswd文件路径]". 在这里我们使用 "plain[/etc/ocserv/ocpasswd]".

* use-seccomp

    布尔值 (true or false), 是否使用 seccomp 隔离 worker 进程。

* max-clients 和 max-same-clients

    限制连接数

* dpd

    心跳包间隔

* server-cert 和 server-key

    服务器证书和证书私钥的路径。

    生成私钥和自签名证书：

    {% highlight sh %}
certtool --gererate-privkey > /etc/ocserv/server-key.pem
# 注意：要使用 0600 权限保护私钥。
chmod 0600 /etc/ocserv/server-key.pem
certtool --gererate-self-signed --load-privkey /etc/ocserv/server-key.pem --outfile /etc/ocserv/server-cert.pem
    {% endhighlight %}

* ipv4-network 和 ipv4-netmask

    指定分配给客户端的地址池范围。

* dns

    推送给客户端的 DNS 服务器，可以选择 Google Public DNS 或者 OpenDNS.

* route

    推送给客户端的路由信息。如果要作为默认路由，不指定即可。

接下来就要建立 ocpasswd 文件了。

{% highlight sh %}
touch /etc/ocserv/ocpasswd
chmod 0600 /etc/ocserv/ocpasswd
ocpasswd -c /etc/ocserv/ocpasswd USERNAME
{% endhighlight %}

输入口令，建立用户。然后就可以用 ocserv -f -c /etc/ocserv/config 来运行了。不过，就算用 -f 指定了前台运行，好像错误信息也只会发送到 syslog.

如果一切正常，就可以把 ocserv 指定成自动启动了。不过 ocserv 没有提供 initscript, 我就自己用 sketelon 改了一个，放在了 [Gist](https://gist.github.com/BillLeecn/cc0e4bd5e69a3109a899).

把这个文件放置到 /etc/init.d/, 然后使用 update-rc.d 创建符号链接。

{% highlight sh %}
update-rc.d ocserv defaults
{% endhighlight %}

## 防火墙设置

我使用防火墙规则如下，就不具体说了，细说就够另写一篇了。
{% highlight sh %}
# 打开 TCP 和 UDP 端口
iptables -t filter -A INPUT -p tcp -m tcp --dport 443 -j ACCEPT
iptables -t filter -A INPUT -p udp -m udp --dport 443 -j ACCEPT
# 客户端隔离
iptables -t filter -A FORWARD -s 192.168.69.0/24 -d 192.168.69.0/24 -j DROP
# 允许转发 VPN 子网的包
iptables -t filter -A FORWARD -d 192.168.69.0/24 -j ACCEPT
iptables -t filter -A FORWARD -s 192.168.69.0/24 -j ACCEPT

# 对 VPN 子网做 NAT
iptables -t nat -A POSTROUTING -s 192.168.69.0/24 -j MASQUERADE
{% endhighlight %}

