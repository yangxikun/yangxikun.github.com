---
layout: post
title: "推荐系统学习"
description: ""
category: WEB智能算法
tags: [WEB智能算法]
---
{% include JB/setup %}

*参考自网上许多资料，维基百科，[IBM Developer三篇文章](http://www.ibm.com/developerworks/cn/web/1103_zhaoct_recommstudy1/index.html)，《WEB智能算法》*

#### 数据归一化
- - -
* [常用的归一化方法](http://in.sdo.com/?p=1889)
* [归一化方法](http://baike.baidu.com/view/4154516.htm)

<!--more-->
#### 欧式距离
- - -
n维空间两点之间的距离，设![M_1(x_1,x_2,...,x_n)](/assets/img/201403060101.png)和![M_2(y_1,y_2,...,y_n)](/assets/img/201403060102.png)为n维空间中的两点，则它们的距离（即向量的摸）为：

![d(M_1,M_2) = \sqrt{\sum (x_i-y_i)^2}](/assets/img/201403060103.png)

#### 朴素相似度
- - -
相似度介于0~1之间，0表示用户完全不相同，1表示用户爱好一致，CommonItems的值为两用户对相同物品都有评分的数量，MaxCommonItems的值为两用户对物品有评分的数量的最大者。

![sim(M_1,M_2) = \frac{beta}{beta+d(M_1,M_2)}](/assets/img/201403060104.png)

![sim(M_1,M_2) = 1 - \tanh (\frac{d(M_1,M_2)}{\sqrt{CommonItems}})](/assets/img/201403060105.png)

![sim(M_1,M_2) = (1 - \tanh (\frac{d(M_1,M_2)}{\sqrt{CommonItems}})) \cdot \frac{CommonItems}{MaxCommonItems}](/assets/img/201403060106.png)

#### 余弦相似度 (Cosine Similarity)
- - -
相当于计算两向量夹角的余弦值。余弦相似度通常用于两个向量的夹角小于90°之内，因此余弦相似度的值为0到1之间。

![sim(M_1,M_2)=\frac{\sum{x_iy_i}}{\sqrt {\sum x_i^2} \sqrt {\sum y_i^2}}](/assets/img/201403060107.png)

#### 修正的余弦相似度 (Adjusted Cosine Similarity)
- - -
余弦相似度能判断两组数据的趋势是否相似，但存在一个缺陷：假设内容评分为1-5，用户A对某两个内容的评分为(1,2)，用户B对同样两个内容的评分为(4,5)，那么使用余弦相似度计算得到：0.978，因此就出现了修正的余弦相似度，即所有维度上的数值都减去该维度的均值。

![sim(M_1,M_2)=\frac{\sum{(x_i-\overline{M_1})(y_i-\overline{M_2})}}{\sqrt {\sum (x_i-\overline{M_1})^2} \sqrt {\sum (y_i-\overline{M_2})^2}}](/assets/img/201403060108.png)

那么使用修正的余弦相似度来计算的话为：0.316，显然该值才符合实际。

#### 皮尔逊相关系数
- - -
将两组数据首先做Z分数处理之后，然后两组数据的乘积和除以样本数。

Z分数一般代表正态分布中，数据偏离中心点的距离，等于变量减掉平均数再除以标准差。

标准差则等于变量减掉平均数后的平方和，再除以样本数，最后再开方。

![r_{M_1M_2} = \frac{\sum z(x_i)z(y_i)}{n} = \frac{\sum (x_i-\overline{M_1})(y_i-\overline{M_2})}{n\cdot s(M_1)s(M_2)}](/assets/img/201403060109.png)

![r_{M_1M_2} = \frac{\sum (x_i-\overline{M_1})(y_i-\overline{M_2})}{n\cdot \sqrt {\frac{1}{n}\sum (x_i-\overline{M_1})}\sqrt {\frac{1}{n}\sum {(y_i-\overline{M_2})}}}](/assets/img/201403060110.png)

#### Jaccard index
- - -
用于比较两个样本集合的相似性和差异性。

相似性：
![J(A,B) = \frac{|A \cap B|}{|A \cup B|}](/assets/img/201403060111.png)

差异性：
![d_j(A,B) = 1-J(A,B) = \frac{|A \cup B|-|A \cap B|}{|A \cup B|}](/assets/img/201403060112.png)

如果是两个向量：

![f(M_1,M_2)=\frac{\sum{x_iy_i}}{\sum x_i^2 + \sum y_i^2 - \sum{x_iy_i}}](/assets/img/201403060113.png)

#### 相似邻居的计算
- - -
* 固定数量的邻居：K-neighborhoods 或者 Fix-size neighborhoods
* 基于相似度门槛的邻居：Threshold-based neighborhoods

相似用户计算：比较用户A和用户B对相同条目的评分。

相似条目计算：找到对条目A和条目B，都有评分的所有用户，然后计算条目A和条目B的相似度。

![neighborhoods](/assets/img/201403060114.gif)

#### 基于用户的 CF（User CF）
- - -
实现方法一：通过评估某一用户对某一条目的可能评分，我们需要找到与他相似的用户（user邻居）对这个条目的评分，然后把每个邻居的评分乘以其自身的权重，最后加起来求加权平均分，这样就得到某一用户对某一条目的预测评分了，然后将预测评分高的物品推讲给用户。

实现方法二：找出某一用户的邻居后，将该邻居喜爱但用户还未接触到的物品推荐给用户。

![uc](/assets/img/201403060115.gif)

#### 基于条目的 CF（Item CF）
- - -
实现方法一：通过评估某一用户对某一条目的可能评分，我们需要计算该条目与用户已经评分过的条目（item邻居）的相似度，然后把每一个邻居的评分乘以其自身的权重，最后加起来求加权平均分。这样就得到某一用户对某一条目的预测评分了，然后将预测评分高的物品推讲给用户。

实现方法二：找出用户喜爱的物品的邻居，将相似度高的推荐给用户。

![ic](/assets/img/201403060116.gif)

#### 基于内容的推荐
- - -
[基于内容的推荐（Content-based Recommendations）](http://www.cnblogs.com/breezedeus/archive/2012/04/10/2440488.html)