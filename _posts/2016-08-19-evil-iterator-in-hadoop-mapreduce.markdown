---
layout: post
title: Hadoop MapReduce 里的迭代器是个大坑
tags:
    - Hadoop
    - MapReduce
    - Java
    - mutability
comments: true
---

上周踩了 Hadoop MapReduce 的一个大坑，花了整整两天踩查出来。

Hadoop MapReduce 里的 `reduce` 函数是这样定义的：

{% highlight java %}
public void reducer(KEYIN key, Iterable<VALUEIN> values, org.apache.hadoop.mapreduce.Reducer.Context context);
{% endhighlight %}

即，在 mapper 的输出中，具有相同 `key` 的项会被传递到对 `reduce` 函数的同一次调用中。

相同的 `key` 只要串一个进来就可以了，而问题就出现在值的传递上。函数签名里的 `Iterable<VALUEIN>` 很容易让人理解为这是一个产生 `VALUEIN` 类型的对象的迭代器。但实际上，在每次迭代的时候，MapReduce 并没有构造新对象，而是在原有 `Writable` 对象上调用 `readFields(DataInput)`, 用新值覆盖掉旧值。

在我的业务代码中，我在 Reducer 里对 values 做了语义相当于 `SELECT value, COUNT(1) GROUP BY value` 的操作，因此要把 `VALUEIN` 类型的对象缓存到一个 `Map<VALUEIN, Long>` 中进行计数，这里由于每次迭代时迭代器返回的都是（修改过的）同一个对象，我直接把 value 存到 `Map` 里的写法自然就出问题了。

这个问题查了这么久，主要还是由于完全没去考虑 `Iterable<VALUEIN>` 会每次都返回同一个对象的问题，毕竟迭代器本来的语义就是从一个容器中遍历对象；业务代码里也大量用了反射，导致我一直在到处检查是不是在业务代码处理过程中不小心把 value 对象修改了，于是加了一堆单元测试、改代码把存储业务数据的 bean 改成 immutable 对象，但仍然没有结果。到最后才把业务代码全部注释掉就留下用 `HashMap` 计数的代码，然后立即从 map 里把数据读出来检查，发现还有问题，才想到真正的原因。

Java 没有常量这个特性还是很坑啊，要是在 C++ 里面，这种对象肯定是用常量引用的方式传递给 reducer, 继承 reducer 实现 reduce 函数时，看到传进来的是常量引用，自然就知道要把要把它复制一份保存了。
