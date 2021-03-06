---
title: 一道有趣的概率题
date: 2022-05-10 12:44:13
tags: 
---

最近看到一道有趣的概率题，内容为
> 两个预言家，一个准确率90%，一个准确率30%，他们都预言了未日降临，那么末日降临的概率是多少呢?

<!-- more -->

我一眼看出这是条件概率题。我把它分享给朋友，几个朋友不熟悉概率论，脑瓜卡壳了（记得是高中知识，可能都忘了）。为了讲清楚“贝叶斯公式”，有了这篇水文。

## 背景知识

![](/img/bayes.svg)

**什么是条件概率?**
假设事件$A$、$B$，我们分别记事件$A$、$B$的概率为: $P(A)$ 、$P(B)$。
假如事件存在相关性，在已知事件$B$发生的条件下，$A$也发生的概率是多少呢？我们把它记为$P(A|B)$，称为条件概率。

**什么是贝叶斯公式?**
贝叶斯公式，也叫贝叶斯定理，用于描述多个**条件概率**之间的关系。

借助Venn图，可以很容易理解。
$$P(A|B) = \frac{P(A \cap B)}{P(B)}$$
$$P(A \cap B) = P(A|B)P(B) = P(B|A)P(A) $$


## 解决问题

回到问题。借用Venn图表示
![](/img/doom_venn.svg)
$A$对应「世界末日」，$B$对应「两预言家都预言末日降临」，故灾难发生的概率为

$$
P(A|B) = \frac {0.27} {0.27 + 0.07} = 0.794
$$

## 一点小思考

值得注意的一点是，如果没有第二位预言家的预言，末日降临的概率为0.9。第二位预言家的预言拉低了概率。
这是可以理解的，因为第二们的预言家准确率低，在数学上，相当于70%准确率预言末日不降临。

把这个模型对应到生活，如果所有人都说股市要涨，大概率要跌 :)。
