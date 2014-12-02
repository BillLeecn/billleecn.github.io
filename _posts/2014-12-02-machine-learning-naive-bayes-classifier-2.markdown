---
layout: post
title: 机器学习：朴素贝叶斯分类器（二）
tags:
    - machine learning
    - naive bayes
    - classification
mathjax: true
comments: true
---

## 回顾

在[上一篇][naive-bayes-1]中，得到了朴素贝叶斯分类器的表达式

<div>\[
\begin{align}
\operatorname{classify} (f_1, f_2, ..., f_N) &= \underset{C}{\operatorname{argmax}}\, p(c = C | f_1, f_2, ..., f_N) \\
&= \underset{C}{\operatorname{argmax}}\, \frac{1}{Z} p(c) \prod_{i=0}^{N} p(f_i | c) \\
&= \underset{C}{\operatorname{argmax}}\, p(c) \prod_{i=0}^{N} p(f_i | c)
\end{align}
\]</div>

## 应用

朴素贝叶斯分类器广泛应用于垃圾邮件的检测中，这里就以垃圾邮件的检测作为例子。

这是一个二元分类问题：

\\[
c = \begin{cases}
0, & \text{正常邮件}\\\\
1, & \text{垃圾邮件}\\\\
\end{cases}
\\]

假设用以下特征来识别垃圾邮件：

* $ f_1 $: 单词 sale 在正文中出现的次数
* $ f_2 $: 单词 drug 在正文中出现的次数
* $ f_3 $: 邮件头部是否有 DKIM 签名

训练数据如下：

| 特征 | 正常邮件 | 垃圾邮件 |
|-----|--------|---------|
| $ f_1=0 $ | 81 | 1 |
| $ f_1=1 $ | 9 | 5 |
| $ f_1=2 $ | 0 | 3 |
| $ f_1=4 $ | 0 | 1 |
| 邮件总数 | 90 | 10 |
| $ f_2=0 $ | 90 | 8 |
| $ f_2=4 $ | 0 | 1 |
| $ f_2=5 $ | 0 | 1 |
| 邮件总数 | 90 | 10 |
| $ f_3=0 $ | 3 | 8 |
| $ f_3=1 $ | 87 | 2 |
| 邮件总数 | 90 | 10 |

在这里，用最大似然估计的方法来估计参数，即

\\[\begin{align}
p(c=C) &= \frac{M_C}{M} \\\\
p(f_i=F_{i} | c=C) &= \frac{M_{F_i \cap C}}{M_C}
\end{align}\\]

其中，$ M $ 为样本总数，$ M_C $ 为属于分类 $ C $ 的样本数，$ M_{F_i \cap C} $ 为属于分类 $ C $, 且特征 $ f_i = F_i $ 的样本数。例如：

\\[\begin{alignat}{1}
p(c=0) &= \frac{90}{100} &= 0.9 \\\\
p(c=1) &= \frac{10}{100} &= 0.1 \\\\
p(f_1=0 | c=0) &= \frac{81}{90} &= 0.9 \\\\
p(f_1=0 | c=1) &= \frac{5}{10} &= 0.5 \\\\
p(f_1=1 | c=0) &= \frac{9}{90} &= 0.1 \\\\
p(f_1=1 | c=1) &= \frac{5}{10} &= 0.5 \\\\
p(f_1=2 | c=0) &= \frac{0}{90} &= 0 \\\\
p(f_1=2 | c=1) &= \frac{3}{10} &= 0.3 \\\\
p(f_1=3 | c=0) &= \frac{0}{90} &= 0 \\\\
p(f_1=3 | c=1) &= \frac{0}{10} &= 0 \\\\
p(f_1=4 | c=0) &= \frac{0}{90} &= 0 \\\\
p(f_1=4 | c=1) &= \frac{1}{10} &= 0.1 \\\\
...
\end{alignat}
\\]

现在又一封新邮件， 特征为 $ (f_1=1, f_2=0, f_3=0) $. 我们就可以求得

<div>\[
\begin{align}
&p(c=0 | f_1=1, f_2=1, f_3=0) \\
=\: & p(c=0) p(f_1=1|c=0) p(f_2=0|c=0) p(f_3=0|c=0) \\
=\: & 0.9 \times 0.1 \times 1 \times 0.033 \\
=\: & 0.00297 \\
&p(c=1 | f_1=1, f_2=1, f_3=0) \\
=\: & p(c=1) p(f_1=1|c=1) p(f_2=0|c=1) p(f_3=0|c=1) \\
=\: & 0.1 \times 0.5 \times 0.8 \times 0.033 \\
=\: & 0.00132 \\
\end{align}
\]</div>

因为

\\[ \max(p(c=0 | f_1, f_2, f_3), p(c=1 | f_1, f_2, f_3)) = p(c=0 | f_1, f_2, f_3) \\]

所以判定

\\[ \operatorname{classify}(f_1=1, f_2=0, f_3=0) = 0 \\]

即这封新邮件不是垃圾邮件。

## 实现细节

在这里，有一个问题：在使用最大似然估计时，先验概率 $ p(f_i | c) $ 中会出现等于 0 的项。一旦某个特征 $ p(f_i | c) = 0 $, 那么整个 $ \prod_{i=0}\^{N} p(f_i | c) $ 都将为 0. 这就使得其它特征都失去了作用。

为了避免这个问题，需要对 $ p(f_i | c) $ 做 [Laplace Smoothing][wikipedia-additive-smoothing].

\\[
\hat{p}(f_i=F_i | c=C) = \frac{M_{F_i \cap C} + 1}{M + N}
\\]

其中， $ N $ 是特征的数量。

在实际的应用中，特征的数量 $ n $ 可能非常大，可能有几百个、几千个。由于 $ 0 \le p(f_i|c) \le 1 $, 在计算机中计算 $ \prod_{i=0}\^{N} p(f_i | c) $ 很可能发生浮点数下溢出。为了解决这个问题，当特征的取值为正数时，常常对其取对数。由于 $ \log(ab) = \log(a) + \log(b) $, 这个变换可以解决下溢出的问题。又因为对数函数是单调函数，这个变换不影响其最值点。因此，在实际实现中，常用的方法是

<div>\[
\operatorname{classify}(f_1, f_2, ..., f_N) = \underset{C}{\operatorname{argmax}}\left(\log p(c) + \sum_{i=0}^{N} \log(p(f_i | c))\right)
\]</div>

这种用来处理特征为离散值的分类器，称为多项朴素贝叶斯分类器 (Multinomial Naive Bayes Classifier). 这种分类器被广泛应用在文本的分类中，如：识别垃圾邮件，判定新闻类别。

## 参考资料

* Christopher D. Manning, Prabhakar Raghavan and Hinrich Schütze, 2008, *[Introduction to Information Retrieval](http://nlp.stanford.edu/IR-book/)*, Cambridge University Press. *[Naive Bayes text classification](http://nlp.stanford.edu/IR-book/html/htmledition/naive-bayes-text-classification-1.html)*

[naive-bayes-1]: /2014/11/machine-learning-naive-bayes-classifier-1/, "机器学习：朴素贝叶斯分类器（一）"
[wikipedia-additive-smoothing]: //en.wikipedia.org/wiki/Additive_smoothing, "Additive smoothing"
