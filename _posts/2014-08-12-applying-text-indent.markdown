---
layout: post
title: 优雅地使用 text-indent
tags:
    - CSS
    - text-indent
comments: true
---

在中文的排版中，一般都要在正文段落中使用 2 em 的首行缩进。现在要用 CSS 来实现这个需求，于是我开始写：

{% highlight css %}
p {
    text-indent: 2em;
}
{% endhighlight %}

然后刷新下页面，发现页面上好多不相关元素都向右偏移了 2em. 原来，`<p>` 是个很基本的元素，在好多地方都用。而我要的是**正文**部分缩进。好在这个博客模板上，正文部分是加了 `class="post"` 的。于是现在的 CSS 变成：

{% highlight css %}
.post p {
    text-indent: 2em;
}
{% endhighlight %}

嗯，看起来不错，继续写多点东西试试。很快我发现，在遇到英文时候，这个 2em 的缩进实在太大了。英文本来没有缩进的习惯，中文的 2em 又可能相当于英文的 4、5 个字符了。这个问题我是用微调缩进量解决的。在我使用 Arial, 文泉驿微米黑这两个字体的情况下，这个值我调到了 1.8em 可以得到较好的效果。

这个版本看起来很完美了，用了好久，直到有一天，我遇到了这样的内容：

{% highlight html %}
<li>
    <p>Item A</p>
    <p>Description of Item A</p>
</li>
{% endhighlight %}

然后，我就发现 Item A 和小圆点之间好大一块空白。为了解决这个问题，必须把这种情况排除掉。为了在 CSS 中准确选择出这种元素，需要用到**伪类** (pseudo-classes).

类 (class) 是作者在写 html 时明确指定的，而伪类就可以是认为浏览器在后台默默分好的。浏览器会把所有未访问过的 `<link>` 放进伪类 `:link`, 把 each element that is the first child of another 放进伪类 :first-child.

> -- 等等，为什么在这里突然用英文了呢？

> -- 因为我实在不知道怎么用中文准确清楚地表达这个概念了。。。

嗯，需要排除的就是每个 `<li>` 中的第一个 `<p>`. 于是，CSS 该这样写：

{% highlight css %}
.post li > p:first-child {
    text-indent: 0;
}
{% endhighlight %}

注意，选择器不是 `.post li:first-child`. `:first-child` 是一个伪类，而不是 javascript 里面的 `node.firstChild`.

最后用的完整 CSS 就是：

{% highlight css %}
.post p {
    text-indent: 1.8em;
}

.post li > p:first-child {
    text-indent: 0;
}
{% endhighlight %}

