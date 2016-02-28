---
layout: post
title: 机器学习：朴素贝叶斯分类器（一）
tags:
    - machine learning
    - naive bayes
    - classification
mathjax: true
comments: true
---

朴素贝叶斯 (naive bayes) 分类器是一种简单有效的概率分类器，基于先验概率。

朴素贝叶斯分类器把分类问题看成是在已知特征 $ (f_1 = F_1, f_2 = F_2, ..., f_n = F_n) $ 的情况下，求分类 $ c $ 的概率分布的问题。即：

$$
\operatorname{classify} (f_1, f_2, ..., f_n) = \underset{C}{\operatorname{argmax}}\, p(c = C | f_1, f_2, ..., f_n)
$$

根据贝叶斯定理，有：

$$
p(c| f_1, f_2, ..., f_n) = \frac{p(c) p(f_1, f_2, ..., f_n | c)}{p(f_1, f_2, ..., f_n)} 
$$

其中，分子是一个和类别 c 无关的常量，令

$$
Z = p(f_1, f_2, ..., f_n)
$$

则有，

$$
p(c| f_1, f_2, ..., f_n) = \frac{1}{Z} p(c) p(f_1, f_2, ..., f_n | c) 
$$

在这里，引入一个 naive 的假设：特征 $ (f_1, f_2, ..., f_n) $ 统计独立。那么，就有：

$$
p(f_1, f_2, ..., f_n | c) = \prod_{i=0}^{n} p(f_i | c)
$$

所以，

$$
p(c| f_1, f_2, ..., f_n) = \frac{1}{Z} p(c) \prod_{i=0}^{n} p(f_i | c)
$$

<p>
由上式可知，只要知道 $ p(c) $ 和 $ p(f_i | c) $, 就可以求出 $ p(c | f_1, f_2, ..., f_n) $. 又因为在分类问题中， $ c $ 的取值是有限的，所以也可以直接求得 $ \underset{C}{\operatorname{argmax}}\; p(c = C | f_1, f_2, ..., f_n) $.
</p>

<p>
先验概率 $ p(f_i | c) $, 在知道概率分布模型的情况下，可以利用训练数据，通过参数估计求出。而 $ p(c) $ 可以根据情况，通过训练数据求出；在无法确定的情况下，可以令
</p>

$$
p(c = C_1) = p(c = C_2) = ... = p(c = C_k) = \frac{1}{k}
$$
