---
layout: post
title: Debian 上通过 uwsgi 部署 flask 应用
tags:
    - flask
    - uwsgi
    - debian
    - nginx
comments: true
---

一直很喜欢 python 优美的语法和丰富的库，但是在 Web 方面，python 部署起来就比 php 麻烦很多了。即使是和 Java 相比，上手时也比 Tomcat 困难。也许是 python 在这方面的选择太多，没有一个特别突出的方案，导致这方面的资料比较少吧。

这次部署的项目是一个 flask 应用，使用了 virtualenv, 使用 redis 来保存全局数据。应用的前端是纯 ajax 的，flask 只在 /api/ 下提供几个 RESTful API.

应用的测试版本目录结构如下：

```
myapp
 |-- venv/
 |-- static/
 |     |---- index.html
 |     |---- style.css
 |     |---- myapp.js
 |
 |-- myapp.py
```

在部署前，第一步要做的是迁移 virtualenv. Virtualenv 是绑定在绝对路径上的，由于现在要把 myapp 目录移走，并且要用 `www-data` 用户来运行，放在 `~bill/myapp/venv/` 的环境肯定是用不了的。

但其实  virtualenv 基本上是没发可靠迁移的，所以这里用的方法是，重新建一个一样的 virtualenv.

首先把当前的环境导出：

{% highlight sh %}
. myapp/venv/bin/activate
pip freeze > requirements.txt
deactivate
{% endhighlight %}

然后，把整个项目目录复制到目的地：

{% highlight sh %}
rm -r myapp/venv
sudo cp -r myapp /opt/
{% endhighlight %}

接着，重建 venv:

{% highlight sh %}
sudo chown -R bill:bill /opt/myapp  # 为了避免使用 root 权限来运行下面的命令
virtualenv /opt/myapp/venv
. /opt/myapp/venv/bin/activate
pip install -r /opt/myapp/requirements.txt
deactivate
sudo chown -R root:root /opt/myapp
{% endhighlight %}

然后就要选择部署方案了，在各种眼花缭乱的部署选项中，我选了用 nginx + uwsgi 的方案。至于原因嘛，就是这种方案感觉和自己熟悉 Apache + Tomcat 方案最像，可能会比较好上手。

先把相关的包装上：

{% highlight sh %}
sudo apt-get install nginx uwsgi uwsgi-plugin-python
{% endhighlight %}

这里简单介绍一下 uwsgi. Uwsgi 是一个用 C 写的 application container, 后端支持多种接口，当然 python 的 wsgi 是少不了的，前端支持 http, http-socket, 还有 uwsgi 的专用协议。

下面是最困难的一步，配置 uwsgi. Uwsgi 是没有自己的 init script 的。所以每个发行版打包的时候都是自己处理。官方的文档也就只有直接手工运行时的例子。首先介绍一下 Debian 的 init script 的处理。 Debian 中，uwsgi 的配置文件保存在 /etc/uwsgi. Init script 会加载 /etc/uwsgi/apps-enabled 下的配置，对里面的每一个配置文件，都启动一个 uwsgi daemon. 这个就是典型的 Debian 配置的目录结构了，在 /etc/uwsgi/apps-available 下建立配置文件，然后把需要启动的符号链接到 /etc/uwsgi/apps-enabled 下。

下面就开始写配置文件了。按照 [flask 的文档](http://flask.pocoo.org/docs/0.10/deploying/uwsgi/)，启动 uwsgi 应该使用以下命令：

{% highlight sh %}
uwsgi -s /tmp/myapp.sock --module myapp --callable app
{% endhighlight %}

其中， `--module myapp` 指明了要加载的 python 模块，而 `--callable app` 则是指明模块中 `Flask` 实例的变量名。显然，这个命令有一个前提条件：为了能找到模块 `myapp.py`, 当前目录 `pwd` 必须是 `myapp.py` 所在目录。

于是开始查 `man uwsgi`. 发现了一个参数

>        --chdir
>              chdir to specified directory before apps loading

就是它了。然后还有一个问题，就是上面这个命令不涉及 virtualenv. 再查文档，发现了 `--virtualenv` 参数。最后，为了安全，还要加上 `--uid www-data --gid www-data` 来让 uwsgi 放弃 root 权限。本来 uwsgi 的文档里是推荐单独建立一个 `uwsgi`  用户的，这里我就偷懒一下了。

最后，把这些参数转化成 .ini 配置文件。

{% highlight ini %}
[uwsgi]
uid = www-data
gid = www-data
plugin = python
socket = /tmp/myapp.sock
chdir = /opt/myapp
virtualenv = /opt/myapp/venv
module = myapp
callable = app
{% endhighlight %}

执行 `sudo service uwsgi start` 启动服务。下面就要配置 nginx 做前面的反向代理。这个应用是纯 ajax 的，只有在 /api/ 下的 RESTful API 是需要走 flask. 其他的文件都可以由 nginx 直接提供。我希望最后的网站是这个结构:

```
 /
 |--- index.html
 |--- style.css
 |--- myapp.js
 |--- api/
       |--- ...
       ...
```

所以应该让 nginx 把 `/api` 下的请求传递给 uwsgi, 其他请求直接从 `/opt/myapp/static` 中提供。 Nginx 的资料齐全，这里就不细说了，最后的配置如下：

```
server {
    location / {
        root /opt/myapp/static;
    }

    location /api/ {
        include uwsgi_params;
        uwsgi_pass unix:/tmp/myapp.sock;
    }
}
```

放在 `/etc/nginx/sites-availabe/`, 然后符号链接到 `/etc/nginx/sites-enabled/`. Reload nginx 后，应该就能够正常工作了，如果出现问题，首先看 nginx 的 log. 如果是 500, 502 之类的错误，再去看 uwsgi 的 log.
