---
layout: post
title: 在 Ubuntu 12.04 中修改屏幕 DPI
tags:
    - DPI
    - Xserver
    - Gnome
    - Ubuntu
    - Firefox
comments: true
---

至少需要修改 Xserver 和 Gnome 的配置，有些软件使用了自己的参数，还要另外修改。默认的 DPI 是 96， 这里以修改为 110 为例。

## Xserver
编辑 `/etc/lightdm/lightdm.conf`, 在 `[SeatDefaults]` 中加入

{% highlight ini %}
xserver=X -dpi 110
{% endhighlight %}

修改后，重启 lightdm 来使用新的配置。

## Gnome
Gnome 3 中的 DPI 被写死在 96 了。但是另外有一个选项 text-scaling-factor 来控制缩放。计算出 `110 / 96 = 1.458`, 在文本终端执行

```
gsettings set org.gnome.desktop.interface text-scaling-factor 1.458
```

执行完应该马上就能看到效果了。

## 其它软件

### Firefox

Firefox 的 `about:config` 中有两个相关的选项：`layout.css.dpi` 和 `layout.css.devPixelsPerPx`, 前者控制实际单位（如：英寸 in, 厘米 cm, 毫米 mm, 磅 pt）和屏幕像素之间的转换，后者控制抽象单位 px 和屏幕像素的转换，所以这两个选项应该分别设置为 `110` 和 `1.458`.

## 参考
* [How to find and change the screen DPI? - Ask Ubuntu](http://askubuntu.com/questions/197828/how-to-find-and-change-the-screen-dpi)
* [Firefox tweaks - ArchWiki](https://wiki.archlinux.org/index.php/firefox_tweaks#Configure_the_DPI_value)
* [Layout.css.dpi - MozillaZine Knowledge Base](http://kb.mozillazine.org/Layout.css.dpi)
