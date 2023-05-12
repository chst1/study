---
title: LambdaMart算法原理与实现
date: 2021-12-11 16:31:59
description: LambdaMart算法作为经典的搜索排序算法，至今依然被各大互联网公司使用，在工作中使用到该算法，这里进行详细学习。
tags:
    排序
categories:
    机器学习
mathjax:
    true
---

# NDCG评估指标

NDCG用于评估搜索返回的结果相关性指标。其由CG转换到DCG再发展出nDCG。下面依次介绍。

## 累计增益(CG)

CG，cumulative gain，是DCG的前身，只考虑到了相关性的关联程度，没有考虑到位置的因素。它是一个搜素结果相关性分数的总和。指定位置P上的CG为：
$$
CG_P=\sum_{i=1}^P rel_i
$$
其中$rel_i$为第$i$个位置上的文档和query的相关性。

该度量方法存在问题，因为其无法区分排序顺序，只能判断召回的数据是否相关，比如一个query应该召回三个文档$a,b,c$。其相关性依次降低，但如果召回的实际顺序是$c,b,a$时，该评估指标和最优情况下的指标值是一样的。

为解决上述问题，引入了DCG。

## 折损累计增益(DCG)

相比于CG，DCG在CG的每一个结果上增加了一个折损值。目的就是为了让排名越靠前的结果越能影响最后的结果。假设排序越往后，价值越低。到第i个位置的时候，它的价值是 $\frac{1}{log_2{(i+1)}}$，那么第i个结果产生的效益就是 $rel_i * \frac{1}{log_2{(i+1)}}$，因此：
$$
DCG_P\sum_i^P\frac{rel_i}{log_2{(i+1)}}
$$
因为是乘以一个折损值，因此该值可以取别的函数，还有一个更加常用的公式，如下：
$$
DCG_P\sum_i^P\frac{2^{rel_i}-1}{log_2{(i+1)}}
$$
DCG解决了同一个query下排序的指标，但是对于不同query其召回的文档数量是不一致的，DCG采用的是累加的方式，这对应召回文档更多的query是更加友好的，但不符合实际使用，因此NDCG被提出以改进DCG。

## 归一化折损累计增益(NDCG)

由于DCG是一个累加值，无法对不同搜索结果进行比较，因此NDCC对每个query的结果进行了归一化操作，其方法是对DCG值除以IDCG(理想情况下最大DCG值)：
$$
nDCG_P=\frac{DCG_p}{IDCG_p}
$$

$$
IDCG_P = \sum_{i=1}^{\vert RET \vert}\frac{2^{rel_i}-1}{log_2{(i+1)}}
$$

其中$\vert REL \vert$表示，结果按照相关性从大到小的顺序排序，取前P个结构组合成的集合，即按照最优的方式对结果进行排序是的DCG值。

# 排序学习LTR

**排序学习（Learning to Rank，LTR）**，也称**机器排序学习（Machine-learned Ranking，MLR)** ，就是使用机器学习的技术解决排序问题。

传统的检索模型所考虑的因素并不多，主要是利用词频、逆文档频率和文档长度、文档重要度这几个因子来人工拟合排序公式，且其中大多数模型都包含参数，也就需要通过不断的实验确定最佳的参数组合，以此来形成相关性打分。

LTR 则是基于特征，通过机器学习算法训练来学习到最佳的拟合公式，相比传统的排序方法。

排序学习的模型通常分为**单点法（Pointwise Approach）**、**配对法（Pairwise Approach）**和**列表法（Listwise** **Approach）**三大类，三种方法并不是特定的算法，而是排序学习模型的设计思路，主要区别体现在损失函数（Loss Function）、以及相应的标签标注方式和优化方法的不同。

## 单点法

单点法排序学习模型的每一个训练样本都仅仅是某一个查询关键字和某一个文档的配对。它们之间是否相关，与其他文档和其他查询关键字都没有关系。

## 配对法

配对法的基本思路是对样本进行两两比较，构建偏序文档对，从比较中学习排序，因为对于一个查询关键字来说，最重要的其实不是针对某一个文档的相关性是否估计得准确，而是要能够正确估计一组文档之间的 “相对关系”。

## 列表法

相对于尝试学习每一个样本是否相关或者两个文档的相对比较关系，列表法排序学习的基本思路是尝试直接优化像 NDCG（Normalized Discounted Cumulative Gain）这样的指标，从而能够学习到最佳排序结果。

# LambdaMART算法

LambdaMART 的提出先后经由 RankNet、LambdaRank 逐步演化而来。

## RankNet

RankNet 的核心是**提出了一种概率损失函数来学习 Ranking Function**，并应用 Ranking Function 对文档进行排序。

RankNet网络将输入query的特征向量$x\in R^n$,映射为一个实数$f(x) \in R$。具体的，给定两个query下的两个文档$U_i$和$U_j$，其特征值分别为$x_i$，$x_j$，经过RankNet进行前向计算得到相应的对应分数$s_i = f(x_i) $,$s_j=f(x_j)$。使用$U_i \triangleright U_j$表示$U_i$比$U_j$排序更靠前（如对某个 query 来说，𝑈𝑖 被标记为 “good”，𝑈𝑗 被标记为 “bad”）。RankNet把排序问题转化成比较一个$(U_i,U_j)$文档对的排序问题，首先计算每个文档得分，定义使用下面的公式计算根据模型得分获取的$U_i$比$U_j$排序更靠前的概率：
$$
P_{ij} \equiv P(U_i \triangleright U_j) \equiv \frac{1}{1+e^{-\sigma(s_i-s_j)}}
$$
这个概率就是深度学习中常用的sigmoid函数，由于参数$\sigma$只影响函数形状，对最终结果影响不大，因此通常使用$\sigma=1$进行简化。

可以看到RankNet预测的分数$s_i>s_j$且差距越大时，$P_{ij}$越接近1，即$U_i$排序就应该在$U_j$前概率越大，反之则$U_j$在$U_i$之前概率越大。

RankNet 证明了如果知道一个待排序文档的排列中相邻两个文档之间的排序概率，则通过推导可以算出每两个文档之间的排序概率。因此对于一个待排序文档序列，只需计算相邻文档之间的排序概率，不需要计算所有 pair，减少计算量。

### 真实的相关性概率

对于特定query，定义$S_{ij} \in {0,\pm 1}$为文档$U_i$和文档$U_j$被标记的标签之间的关联，即：
$$
S_{ij} =
\begin{cases}
1,\qquad U_i比U_j更相关 \\
0,\qquad U_i和U_j相关性一致 \\
-1,\qquad U_j比U_i更相关
\end{cases}
$$
因此可以定义文档$U_i$比文档$U_j$更相关的真实概率为：
$$
\overline{P_{ij}} = \frac{1}{1+S_{ij}}
$$

### 损失函数

对于一个排序，RankNet 从各个 doc 的相对关系来评价排序结果的好坏，排序的效果越好，那么有错误相对关系的 pair 就越少。所谓错误的相对关系即如果根据模型输出 𝑈𝑖 排在 𝑈𝑗 前面，但真实 label 为 𝑈𝑖 的相关性小于 𝑈𝑗，那么就记一个错误 pair，**RankNet 本质上就是以错误的 pair 最少为优化目标，也就是说 RankNet 的目的是优化逆序对数。而在抽象成损失函数时，RankNet 实际上是引入了概率的思想：不是直接判断 𝑈𝑖 排在 𝑈𝑗 前面，而是说 𝑈𝑖 以一定的概率 P 排在 𝑈𝑗 前面，即是以预测概率与真实概率的差距最小作为优化目标。**最后，**RankNet 使用交叉熵（Cross Entropy）作为损失函数**，来衡量 $P_{ij}$ 对 $\overline{P_{ij}}$ 的拟合程度：
$$
L = - \overline{P_{ij}}logP_{ij} - (1-\overline{P_{ij}})log(1-P_{ij})
$$
化简后：
$$
L_{ij} = \frac{(1-S_{ij}) \sigma (s_i-s_j)}{2}+log(1+e^{-\sigma(s_i-s_j)})
$$
当$S_{ij}=1$时：
$$
L_{ij} = log(1+e^{-\sigma(s_i-s_j)})
$$
当$S_{ij}=-1$时：
$$
L_{ij} = \sigma (s_i-s_j) + log(1+e^{-\sigma(s_i-s_j)})  = log(1+e^{-\sigma(s_j-s_i)})
$$
从损失函数可以看出，如果对文档$U_i$和$U_j$的打分可以正确拟合标签时，则$L_{ij}$趋于0，否则趋于线性函数。具体的，假如$S_{ij}$=1,则$U_i$应该比$U_j$排序高：

如果$s_i > s_j$则拟合的分数可以正确排序文档：
$$
\underset{si-sj \rightarrow \infty}{lim} L_{ij}  =  \underset{si-sj \rightarrow \infty}{lim}log(1+e^{-\sigma(s_i-s_j)}) = log1=0
$$
如果$s_i<s_j$，则拟合的分数不能正确排序文档：
$$
\underset{si-sj \rightarrow -\infty}{lim} L_{ij}  =  \underset{si-sj \rightarrow -\infty}{lim}log(1+e^{-\sigma(s_i-s_j)})=log(e^{-\sigma(s_i-s_j)}) = -\sigma(s_i-s_j)
$$
下图展示了当$S_{ij}$分别取1，0，-1时，损失函数以$s_i-s_j$为变量的取值示意图：

[![5WQ0JI.png](https://z3.ax1x.com/2021/10/24/5WQ0JI.png)](https://imgtu.com/i/5WQ0JI)

该损失函数具有如下特点：

1. 当两个相关性不同的文档算出来的模型分数相同时，即 𝑠𝑖=𝑠𝑗，此时的损失函数的值为 log2，依然大于 0，仍会对这对 pair 做惩罚，使他们的排序位置区分开。
2. 损失函数是一个类线性函数，可以有效减少异常样本数据对模型的影响，因此具有鲁棒性。

Ranknet 最终目标是训练出一个打分函数$s=f(x,w)$，使得所有 pair 的排序概率估计的损失最小，即：
$$
L = \sum_{(i,j) \in I} L_{ij}
$$
其中，𝐼 表示所有在同一 query 下，且具有不同相关性判断的 doc pair，每个 pair 有且仅有一次。

通常该打分函数只要光滑可导即可。

### 参数更新

RankNet 采用神经网络模型优化损失函数，也就是后向传播过程，采用梯度下降法求解并更新参数：
$$
w_k = w_k - \eta \frac{\partial L} {\partial w_k}
$$
其中$\eta$为学习率。RankNet 是一种方法框架，因此这里的 𝑤𝑘 可以是 NN、LR、GBDT 等算法的权重。

对RankNet的梯度$\frac{\partial L} {\partial w_k}$进行因式分解，有：
$$
\frac{\partial L} {\partial w_k} = \sum_{(i,j)\in I}\frac{\partial L_{ij}} {\partial w_k}=\sum_{(i,j)\in I}[\frac{\partial L_{ij}} {\partial s_i} \frac{\partial s_i}{\partial w_k} + \frac{\partial L_{ij}} {\partial s_j} \frac{\partial s_j}{\partial w_k}]
$$
其中：
$$
\frac{\partial L_{ij}} {\partial s_i} = \\ \frac{\partial \{ \frac{(1-S_{ij}) \sigma (s_i-s_j)}{2}+log(1+e^{-\sigma(s_i-s_j)})\}}{\partial s_i} = \\ \sigma[\frac{(1-S_{ij})}{2}-\frac 1 {1+e^{\sigma(s_i-s_j)}}] =\\ -\frac{\partial L_{ij}} {\partial s_j}
$$
$s_i$和$s_j$对$w_k$的偏导数可以根据实际使用的模型求的。

结果排序最终由模型得分$s_i$确定，其梯度很关键，定义：
$$
\lambda_{ij} \overset{def}{=} \frac{\partial L_{ij}} {\partial s_i} = -\frac{\partial L_{ij}} {\partial s_j} =\\ \sigma[\frac{(1-S_{ij})}{2}-\frac 1 {1+e^{\sigma(s_i-s_j)}}]
$$
对于有序对$(i,j)$,有$S_{ij}= 1$，于是化简：
$$
\lambda_{ij} \overset{def}=-\frac 1 {1+e^{\sigma(s_i-s_j)}}
$$
此时$\lambda_{ij}$始终为负数，使用梯度更新时，将会使$s_i$增大，因此，对于文档$U_i$来说，真实的相关性在其后的文档将促进其分数增加。即给其一个向上推的力。

由于：
$$
\frac{\partial L_{ij}} {\partial s_i}  = -\frac{\partial L_{ij}} {\partial s_j}
$$
对于$s_j$来说，其导数均为正数，则在更新模型时使$s_j$变小，因此对应任意文档来说，真实的相关性在其前的文档都将促使其分数减小，即给其一个向下推的力。

这个$lambda_{ij}$即为接下来介绍的LambdaRank 和 LambdaMART 中的 Lambda，称为 Lambda 梯度。

## LambdaRank

RankNet 本质上就是以错误的 pair 最少为优化目标，也就是说 RankNet 的直接目的就是优化逆序对数（pairwise error），这种方式一定程度上能够解决一些排序问题，但它并不完美。

逆序对数（pairwise error）表示一个排列中，抽查任意两个 item，一共有 $C_2^n$ 种可能的组合，如果这两个 item 的之间的相对排序错误，逆序对数量增加 1。

**逆序对数的实质就是插入排序过程中要移动元素的次数。直观理解为要想把某个元素移动到最优排序位置，需要移动多少次，两个元素就是二者移动次数的和。**

例如，对某个 Query，和它相关的文章有两个，记为 (𝑄,[𝐷1,𝐷2]) 。

1. 如果模型 𝑓(⋅) 对此 Query 返回的 n 条结果中， 𝐷1,𝐷2 分别位于前两位，那么 pairwise error 就为 0；
2. 如果模型 𝑓(⋅) 对此 Query 返回的 n 条结果中， 𝐷1,𝐷2 分别位于第 1 位和第 n 位，那么 pairwise error 为 n-2；
3. 如果模型 𝑓(⋅) 对此 Query 返回的 n 条结果中， 𝐷1,𝐷2 分别位于第 2 和第 3 位，那么 pair-wise error 为 2；

假设RankNet经过两轮迭代实现下图所示的顺序优化：

[![5WNCo8.png](https://z3.ax1x.com/2021/10/24/5WNCo8.png)](https://imgtu.com/i/5WNCo8)

第一轮的时候逆序对数为 13，第二轮为 3 + 8 = 11，逆序对数从 13 优化到 11，损失确实是减小了，如果用 AUC 作为评价指标，也可以获得指标的提升。但实际上，我们不难发现，优化逆序对数并没有考虑位置的权重，这与我们实际希望的排序目标不一致。下一轮迭代，RankNet 为了获得更大的逆序对数的减小，会按照黑色箭头那样的趋势，排名靠前的文档优化力度会减弱，更多的重心是把后一个文档往前排，这与我们搜索排序目标是不一致的，我们更希望出现红色箭头的趋势，优化的重点放在排名靠前的文档，尽可能地先让它排在最优位置。所以我们需要一个能够考虑位置权重的优化指标。

**RankNet 以优化逆序对数为目标，并没有考虑位置的权重，这种优化方式对 AUC 这类评价指标比较友好，但实际的排序结果与现实的排序需求不一致，现实中的排序需求更加注重头部的相关度，排序评价指标选用 NDCG 这一类的指标才更加符合实际需求。而 RankNet 这种以优化逆序对数为目的的交叉熵损失，并不能直接或者间接优化 NDCG 这样的指标。**

**对于绝大多数的优化过程来说，目标函数很多时候仅仅是为了推导梯度而存在的。而如果我们直接就得到了梯度，那自然就不需要目标函数了。**

微软学者经过分析，就直接把 RankNet 最后得到的 Lambda 梯度拿来作为 LambdaRank 的梯度来用了，这也是 LambdaRank 中 Lambda 的含义。这样我们便知道了 LambdaRank 其实是一个经验算法，它不是通过显示定义损失函数再求梯度的方式对排序问题进行求解，而是分析排序问题需要的梯度的物理意义，直接定义梯度，即 Lambda 梯度。有了梯度，就不用关心损失函数是否连续、是否可微了，所以，微软学者直接把 NDCG 这个更完善的评价指标与 Lambda 梯度结合了起来，就形成了 LambdaRank。

首先来分析一下lambda梯度的意义：LambdaRank 中的 Lambd 其实就是 RankNet 中的梯度 $\lambda_{ij}$， $\lambda_{ij}$可以看成是 𝑈𝑖 和 𝑈𝑗 中间的作用力，代表下一次迭代优化的方向和强度。如果 𝑈𝑖⊳𝑈𝑗，则 𝑈𝑗 会给予 𝑈𝑖 向上的大小为  $\lambda_{ij}$ 的推动力，而对应地 𝑈𝑖 会给予 𝑈𝑗 向下的大小为  $\lambda_{ij}$ 的推动力。对应上面的那张图，Lambda 的物理意义可以理解为图中箭头，即优化趋势。

因此，直接使用$\vert \triangle_{NDCG} \vert $乘以$\lambda_{ij}$，就可以得到LambdaRank的Lambda，即：
$$
\lambda_{i,j} \overset{def}=\frac{\partial L{(s_i-s_j)}}{\partial s_i} =-\frac 1 {1+e^{\sigma(s_i-s_j)}} \vert \triangle_{NDCG} \vert
$$
其中$\vert \triangle_{NDCG} \vert $为$U_i$和$U_j$交换排序位置得到的NDCG差值的**绝对值**。通过前文的描述我们可以知道，NDCG指标考虑了排序位置的影响，$\vert \triangle_{NDCG} \vert $值为在全部召回队列中仅考虑i和j两个文档时，互换位置对最终DNCG分数的影响。

对于$U_i \rhd U_j$来说，如果返回的分数排序后$U_i$对应$index_1$,$U_j$对应$index_j$。则：
$$
 \triangle_{NDCG}   = \frac 1 {IDCG} [ (\sum_{k \in I, k \notin {i,j} } \frac{rel_k}{log_2(index_k+1)})+ \frac{rel_i}{log_2{(index_j+1)}}+\frac{rel_j}{log_2{(index_i+1)}} - \\ (\sum_{k \in I, k \notin {i,j} } \frac{rel_k}{log_2(index_k+1)})+ \frac{rel_i}{log_2{(index_j+1)}} - \frac{rel_i}{log_2{(index_i+1)}}-\frac{rel_j}{log_2{(index_j+1)}}]=\\
\frac 1 {IDCG} (rel_i-rel_j)(\frac 1 {log(index_j+1)}-\frac 1 {log(index_i+1)})
$$
从上试我们可以得知，大两个文档的真实分数差越大时，结果的$\vert \triangle_{NDCG} \vert $值越大，这样就起到了对于排名高并且相关性高的文档更快地向上推动，而排名低而且相关性较低的文档较慢地向上推动。

对于特定$U_i$，累加其他所有排序项的影响，得到：
$$
\lambda_i = \sum_{(i,j)\in I,i \rhd j}\lambda_{ij} + \sum_{(k,i)\in I,k \rhd i}\lambda_{ki}=\sum_{(i,j)\in I,i \rhd j}\lambda_{ij} - \sum_{(k,i)\in I,k \rhd i}\lambda_{ik}
$$
即，对于每个文档，分别计算与排在其后的所有文档的$\lambda_{ij}$和与排在其前的所有文档的$\lambda_{ki}$,对这些值进行求和。

从之前的分析我们知道，排在其后的文档给予当前文档一个向上推动的力，排在其后的文档给予其向下推动的力。因此为了保证相关性更高的文档排序更高(对相关性低的类似，让其排序更靠后)，因此我们直接拟合$\lambda_i$即可。

如果将$\lambda_i$看做一个导数，那么我们可以推出其效用函数为：
$$
C_{ij} = log(1+e^{-\sigma(si-sj)})\vert \triangle_{NDCG} \vert
$$

## LambdaMART算法

LambdaRank 重新定义了梯度，赋予了梯度新的物理意义，因此，所有可以使用梯度下降法求解的模型都可以使用这个梯度，基于决策树的 MART 就是其中一种，将梯度 Lambda 和 MART 结合就是大名鼎鼎的 LambdaMART。

MART即为GBDT，LambdaMART 只是在 GBDT 的过程中做了一个很小的修改。原始 GBDT 的原理是直接在函数空间对函数进行求解模型，结果由许多棵树组成，每棵树的拟合目标是损失函数的梯度，而在 LambdaMART 中这个梯度就换成了 Lambda 梯度，这样就使得 GBDT 并不是直接优化二分分类问题，而是一个改装了的二分分类问题，也就是在优化的时候优先考虑能够进一步改进 NDCG 的方向。

LambdaMART整体流程如下：

[![5fujCq.png](https://z3.ax1x.com/2021/10/24/5fujCq.png)](https://imgtu.com/i/5fujCq)

其执行逻辑如下：

0. 隐含步骤：使用当前模型$F(x)$的输出$s$计算效用函数$C$。这里实际并不会执行，因为我们已经直接找到了其梯度，不需要通过效用函数来计算梯度。这里是提醒读者，每轮循环优化的目标，依然是最小化效用函数。

1. 每棵树的训练都会先遍历所有的数据，计算每个pair（同一个query下的）互换位置导致的指标变化$\vert \triangle_{NDCG} \vert $以及lambda值，即：
   $$
   \lambda_{i,j} =-\frac 1 {1+e^{\sigma(s_i-s_j)}} \vert \triangle_{NDCG} \vert
   $$
   之后计算每个文档的Lambda：
   $$
   \lambda_i =\sum_{(i,j)\in I,i \rhd j}\lambda_{ij} - \sum_{(k,i)\in I,k \rhd i}\lambda_{ik}
   $$
   通过lambda梯度我们推出效用函数为：
   $$
   C(s_i-sj) = \sum_{i,j\in I}{log(1+e^{-\sigma(si-sj)})\vert \triangle_{NDCG} \vert}
   $$
   由此可得：
   $$
   \frac{\partial C} {\partial s_i} = \sum_{i,j \in I} \frac{-\sigma\vert \triangle_{NDCG} \vert }{1+e^{\sigma (s_i -s_j)}}
   $$
   定义：
   $$
   \rho_{ij} = \frac 1 {1+e^{\sigma (s_i -s_j)}}=\frac{-\lambda_{ij}}{\sigma\vert \triangle_{NDCG} \vert }
   $$
   则：
   $$
   y_i = \lambda_i =\frac{\partial C} {\partial s_i} = \sum_{i,j \in I} {-\sigma}\vert \triangle_{NDCG} \vert \rho_{ij}
   $$
   计算每个$ y_i$对$\lambda_i$的倒数，用于后续使用牛顿法求解叶子节点的数值：
   $$
   w_i =\frac{\partial y_i} {\partial s_i} = \frac{\partial^2 C} {\partial s_i^2} = \sum_{i,j\in I}\sigma^2 \vert \triangle_{NDCG} \vert \rho_{ij}(1-\rho_{ij}) = \sum_{i,j \in I} \sigma^2 (-\lambda_{ij}) (1+\frac{\lambda_{ij}}{\sigma \vert \triangle_{NDCG} \vert})
   $$

2. 使用$(X, \lambda)$,拟合一个决策回归树，作为本轮循环生成的弱分类器。<font color=red>注意，这里的$\lambda$是上一步的$-y_i$，即负梯度</font>。

3. 对第二步生成的回归树，计算每个叶子节点的数值，采用牛顿迭代法求解。

   牛顿法是对于一个函数，使用泰勒展开，忽略高阶项的一个迭代优化方式。例如对于任意$f(X)$,在任意点$x_n$进行泰勒展示为：
   $$
   f(x) = f(x_n) + \frac{f^{(1)}(x_n)}{1}(x-x_n) + \frac{f^{(2)}(x_n)}{2}(x-x_n)^2 + \sum_{k=3}^{\infty}\frac{f^{(k)}(x_n)}{k}(x-x_n)^k \approx \\ f(x_n) + \frac{f^{(1)}(x_n)}{1}(x-x_n) + \frac{f^{(2)}(x_n)}{2}(x-x_n)^2
   $$
   牛顿法忽略高阶项，之后如果要使得函数$f(x)$最小，则找到$f(x)$梯度为0，即：
   $$
   f^{(1)}(x_n) + f^{(2)}(x_n)(x-x_n) = 0 => \\
   x = x_n - \frac{f^{(1)}(x_n)}{f^{(2)}(x_n)}
   $$
   GBDT对的每个叶节点的计算为： 
   $$
   c_{lk} = arg \ \underset{c}{min} \sum_{x_i \in R_{lk}}L(y_i, f_{t-1}(x_i)+c)
   $$
   

   对应到lambdaMART算法，当前的$x_i$对应为输入$x_i$。函数$f_{t-1}(x)$对应于当前输出的$s_i$分数。$L$对应于效用函数$C$。这里$s_i$的实际是当前已经训练的弱模型的加和即:$s_i=F_{k-1}(x_i)$，其中$k$-1为当前训练的回归树数量。

   利用牛顿法，获取最小化效用函数时输入为：
   $$
   f_{t-1}(x_i)+c = -\frac{\lambda_i}{w_i}
   $$
   此时$s_i-s_j = f_{t-1}(x_i)$，因此：
   $$
   c_{lk} = -\frac{\lambda_i}{w_i}
   $$
   对应上图$\gamma_{lk}$的计算：
   $$
   \gamma_{lk} = \frac{\sum_{x_i in R_{lK}} y_i}{\sum_{x_i \in R_{lk}}w_i} = \frac{\sum_{x_i in R_{lK}} \frac{\partial C} {\partial s_i}}{\sum_{x_i \in R_{lk}}\frac{\partial^{(2)} C} {\partial s_i^2}} = \frac{\sum_{x_i in R_{lK}} \lambda_i}{\sum_{x_i \in R_{lk}}w_i}
   $$
   这里缺失一个负号，我理解是上图有错误，在使用回归树拟合的时候，拟合的是负梯度，即$y_i = -\lambda_i$，对于二阶导，应该使用原$\lambda$的二阶导，这样相除就存在一个负号，这样求出的$\gamma_{lk}$才是上图中的形式。

4. 模型更新

   使用牛顿法更新模型，将第k颗树增加到模型中。模型更新如下：
   $$
   F_k(x_i) = F_{k-1} + \eta \sum_l \gamma_{lk}I(x_i \in R_{lk})
   $$
   其中$\eta$为学习率。

 



# 程序执行逻辑

```python
def train(train_data, valid_data, model_save_path):
    with open(train_data) as trainfile, \
            open(valid_data) as valifile:
        TX, Ty, sample_weight, Tqids, _ = pyltr.data.letor.read_dataset(trainfile, one_indexed=True, has_weight=True)
        VX, Vy, _, Vqids, _ = pyltr.data.letor.read_dataset(valifile, one_indexed=True, has_weight=False)
    # old_k is 4 
    metric_train = pyltr.metrics.NDCG(k=20)
    metric_valid = pyltr_note.metrics.NDCG(k=4)
    monitor = pyltr.models.monitors.ValidationMonitor(
            VX, Vy, Vqids, metric=metric_valid, stop_after=250)
    
    print "load data finished ....."
    print 'begin to train ......'


#pjz
    model = pyltr.models.LambdaMART(
        metric=metric_train,
        n_estimators=1000,
        learning_rate=0.01,
        max_features=0.5,
        query_subsample=0.75,
        max_leaf_nodes=20,
        min_samples_leaf=64,
        verbose=1,
    )
    
    model.fit(TX, Ty, sample_weight, Tqids, monitor=monitor)
    save_model(model, model_save_path)

    gen_info(model.estimators_)
    print 'output tree info at trees.info'
    
    print sum(model.feature_importances_)
    print model.feature_importances_
    print np.argsort(model.feature_importances_)
    
    id2featrue = {}
    for line in open('../dicts/featrue2id'):
        line = line.strip()
        featrue,id = line.split()
        id = int(id)
        id2featrue[id] = featrue
    for i in np.argsort(model.feature_importances_):
        print id2featrue[i],model.feature_importances_[i]
```

## 读取数据

`pyltr.data.letor.read_dataset`逻辑：

```c
/*
source: string or iterable 可迭代读取的数据。可以是打开的文件描述符
has_targets: bool 是否存在y（即标签）,如果有的化，则必须放到每一行的第一个
one_indexed: bool 参数x是否是按照索引开始的（也就是是从1开始还是从0开始）
missing: float 某个纬度的数据缺失时的默认值
has_weight: bool 是否每个数据有不同的权重，如果有的话，权重必须在每行的第二个值
*/

def read_dataset(source, has_targets=True, one_indexed=True, missing=0.0, has_weight=False):
    """Parses a LETOR dataset from `source`.

    Parameters
    ----------
    source : string or iterable of lines
        String, file, or other file-like object to parse.
    has_targets : bool, optional
        See `iter_lines`.
    one_indexed : bool, optional
        See `iter_lines`.
    missing : float, optional
        See `iter_lines`.

    Returns
    -------
    X : array of arrays of floats
        Feature matrix (see `iter_lines`).
    y : array of floats
        Target vector (see `iter_lines`).
    qids : array of objects
        Query id vector (see `iter_lines`).
    comments : array of strs
        Comment vector (see `iter_lines`).

    """
    if isinstance(source, sklearn.externals.six.string_types):
        source = source.splitlines()

    max_width = 0
    xs, ys, weights, qids, comments = [], [], [], [], []
    // 处理数据
    it = iter_lines(source, has_targets=has_targets,
                    one_indexed=one_indexed, missing=missing, has_weight=has_weight)
    # 添加每一行数据到各种数组中
    for x, y, weight, qid, comment in it:
        xs.append(x)
        ys.append(y)
        weights.append(weight)
        qids.append(qid)
        comments.append(comment)
        # 记录最大的x纬度
        max_width = max(max_width, len(x))
    # 根据最大的x纬度，对整体的x进行填充，保证所有x纬度一致
    assert max_width > 0
    X = np.ndarray((len(xs), max_width), dtype=np.float64)
    X.fill(missing)
    for i, x in enumerate(xs):
        X[i, :len(x)] = x
    ys = np.array(ys) if has_targets else None
    qids = np.array(qids)
    comments = np.array(comments)
    weights = np.array(weights)

    return (X, ys, weights, qids, comments)
```



```python
# 参数与上述的函数一致
def iter_lines(lines, has_targets=True, one_indexed=True, missing=0.0, has_weight=False):
    """Transforms an iterator of lines to an iterator of LETOR rows.

    Each row is represented by a (x, y, qid, comment) tuple.

    Parameters
    ----------
    lines : iterable of lines
        Lines to parse.
    has_targets : bool, optional
        Whether the file contains targets. If True, will expect the first token
        of every line to be a real representing the sample's target (i.e.
        score). If False, will use -1 as a placeholder for all targets.
    one_indexed : bool, optional
        Whether feature ids are one-indexed. If True, will subtract 1 from each
        feature id.
    missing : float, optional
        Placeholder to use if a feature value is not provided for a sample.
    has_weight : bool, optional
        Whether has sample weight, If False, all samples' weight will be assigned 1.0, otherwise
        use the value read from data

    Yields
    ------
    x : array of floats
        Feature vector of the sample.
    y : float
        Target value (score) of the sample, or -1 if no target was parsed.
    weight : float
        sample  weight
    qid : object
        Query id of the sample. This is currently guaranteed to be a string.
    comment : str
        Comment accompanying the sample.

    """
    # 遍历每一行
    for line in lines:
        # 去除末尾的空格，并按照#切割为两部分，data为要获取的数据，comment为注释，即数据对应的辅助信息
        data, _, comment = line.rstrip().partition('#')
        # 按照空格切割数据
        toks = data.split()

        num_features = 0
        # 默认x存在8个纬度，先使用missing即缺失值进行填充
        x = np.repeat(missing, 8)
        y = -1.0
        weight = 1.0
        # 有标签y时，第一个值为权重
        if has_targets:
            y = float(toks[0])
            toks = toks[1:]
        # 有权重时，第二个值为权重
        if has_weight:
            weight = float(toks[0])
            toks = toks[1:]
        # 获取qid，使用qid:切割
        qid = _parse_qid_tok(toks[0])
        # 剩余的toks[1:]全部为x，变量每一个参数
        for tok in toks[1:]:
            # 获取对应的纬度和值
            fid, _, val = tok.partition(':')
            fid = int(fid)
            val = float(val)
            # 如果使用索引开始，则需要将fid减去1（即文件中从1开始）
            if one_indexed:
                fid -= 1
            assert fid >= 0
            # 如果当前x长度小于其索引，则说明预分配的空间不足
            while len(x) <= fid:
                orig = len(x)
                # 将x扩大到两倍，扩大部分也使用missing填充
                x.resize(len(x) * 2)
                x[orig:orig * 2] = missing
            # 设置对应x向量值
            x[fid] = val
            # 记录实际x的纬度
            num_features = max(fid + 1, num_features)
        # 根据纬度对X进行resize
        assert num_features > 0
        x.resize(num_features)

        yield (x, y, weight, qid, comment)
```

根据上述代码，可以直到，pyltr读取数据的格式如下：

```c
y weight qid:value x_1:value_1 x_2:value_2 ... x_n:value_n#commment
```

其中y和weight非必须（训练应该必须有y）qid的value用于划分pair，x_i为输入的x向量，如果某个向量没有，可以不在文件中存在，使用missing填充。comment是这条数据对应的额外信息，如query和对应的doc。

注意qid必须要是同一个的连续存放。



# lambdaMart

## 初始化参数

| 参数              | 类型                            | 含义                                                         | 默认值 |
| ----------------- | ------------------------------- | ------------------------------------------------------------ | ------ |
| metric            | object                          | 模型要最大化的度量。                                         |        |
| learning_rate     | float                           | 学习率                                                       | 0.1    |
| n_estimators      | int                             | 提升阶段数量，即简单模型的数量（决策树）                     | 100    |
| max_depth         | int                             | 单个回归估计器的最大深度。最大深度限制了树中节点数量，通过调整该参数来优化性能。如果`max_leaf_nodes`不为空则忽略该参数。 | 3      |
| min_samples_split | int                             | 拆分内部节点所需的最小样本数。                               | 2      |
| min_samples_leaf  | int                             | 叶节点所需的最小样本数。                                     | 1      |
| subsample         | float                           | 用于拟合单个基础学习器的样本比例。                           | 1.0    |
| query_subsample   | float                           | 用于拟合单个基础学习器的查询比例。（即qid数量）              | 1.0    |
| max_features      | int，float，string，none        | 内部节点生成子节点时考虑的特征数量。（在存在大量特征时加速）。int时，表示要考虑的数量。float时，表示所有节点的百分比，auto时为sqrt(n_features)，sqrt和auto一样，log2为($log_2{n\_features}$)，none时为所有特征均要考虑。选择“max_features < n_features”会导致方差减少和偏差增加。注意：在找到至少一个节点样本的有效分区之前，分割的搜索不会停止，即使它检查已经超过 ``max_features`` 个特征。 | none   |
| max_leaf_nodes    | int/none                        | 生成树的最大叶节点数量。如果是none，则表示无限制。不为none时将忽略参数`max_depth` | none   |
| verbose           | int                             | 启用详细输出。 如果为 1，则它会不时打印进度和性能（树越多频率越低）。 如果大于 1，则它会打印每棵树的进度和性能。 | 0      |
| warm_start        | bool                            | 当设置为 ``True`` 时，重用之前调用 fit 的集成模型并向集成添加更多估计量，否则，只需删除之前的解决方案。 | false  |
| random_state      | int，RandomState instance，none | 如果是 int，random_state 是随机数生成器使用的种子； 如果是 RandomState 实例，random_state 是随机数生成器； 如果没有，随机数生成器是 `np.random` 使用的 RandomState 实例。 | none   |

上述是初始化LambdaMART的参数，用于定义如下生成模型。除了这些参数以外，还包含如下的内部属性：

| 属性                 | 类型                                               | 含义                                                         |
| -------------------- | -------------------------------------------------- | ------------------------------------------------------------ |
| feature_importances_ | n_features纬数组                                   | 特征重要性（越高，特征越重要）。                             |
| oob_improvement_     | n_estimators纬数组                                 | 袋外样本相对于前一次迭代的损失（= 偏差）改进。``oob_improvement_[0]`` 是对 ``init`` 估计器的第一阶段损失的改进。 |
| train_score_         | n_estimators纬数组                                 | 第 i 个分数 ``train_score_[i]`` 是模型在迭代 ``i`` 时在袋内样本上的偏差（= 损失）。如果 ``subsample == 1`` 这就是训练数据的偏差。 |
| estimators_          | [n_estimators, 1]的ndarray存储训练完成的决策树集合 | 拟合子估计量的集合。 对于二元分类，``loss_.K`` 为 1，否则为 n_classes。 |
| estimators_fitted_   | int                                                | 实际拟合的子估计量的数量。 这可能与 n_estimators 在提前停止、修剪等情况下不同。 |

### metric

进行训练效果的度量，这里使用的是NDCG，其定义如下：

```python
class NDCG(Metric):
    def __init__(self, k=10, gain_type='exp2'):
        super(NDCG, self).__init__()
        self.k = k
        self.gain_type = gain_type
        self._dcg = DCG(k=k, gain_type=gain_type)
        self._ideals = {}
```

其中k表示只计算排序前k的数据的NDCG。`gain_type`表示度量NDCG是，每个文档的加权值，可以选择：

exp2表示使用的算分函数为：
$$
DCG_P\sum_i^P\frac{2^{rel_i}-1}{log_2{(i+1)}}
$$
identity表示使用的函数为：
$$
DCG_P\sum_i^P\frac{x}{log_2{(i+1)}}
$$


## 执行fit（拟合）函数

```python

    def fit(self, X, y, sample_weight, qids, monitor=None):
        """Fit lambdamart onto a dataset.

        Parameters
        ----------

        X : array_like, shape = [n_samples, n_features]
            Training vectors, where n_samples is the number of samples
            and n_features is the number of features.
        y : array_like, shape = [n_samples]
            Target values (integers in classification, real numbers in
            regression)
            For classification, labels must correspond to classes.
        sample_weight : array_like, shape = [n_samples]
            The weight of the samples.
        qids : array_like, shape = [n_samples]
            Query ids for each sample. Samples must be grouped by query such
            that all queries with the same qid appear in one contiguous block.
        monitor : callable, optional
            The monitor is called after each iteration with the current
            iteration, a reference to the estimator and the local variables of
            ``_fit_stages`` as keyword arguments ``callable(i, self,
            locals())``. If the callable returns ``True`` the fitting procedure
            is stopped. The monitor can be used for various things such as
            computing held-out estimates, early stopping, model introspecting,
            and snapshoting.

        """
        // 如果初始化时参数warm_start为false，则将其之前可能存在的训练属性全部清空
        if not self.warm_start:
            self._clear_state()
        
        // 检验输入数据
        X, y = sklearn.utils.check_X_y(X, y, dtype=sklearn.tree._tree.DTYPE)
        n_samples, self.n_features = X.shape

        sklearn.utils.check_consistent_length(X, y, qids)
        if y.dtype.kind == 'O':
            y = y.astype(np.float64)

        random_state = sklearn.utils.check_random_state(self.random_state)
        self._check_params()
        
        // 分配内部属性空间。
        if not self._is_initialized():
            self._init_state()
            begin_at_stage = 0
            y_pred = np.zeros(y.shape[0])
        else:
            if self.n_estimators < self.estimators_.shape[0]:
                raise ValueError('n_estimators=%d must be larger or equal to '
                                 'estimators_.shape[0]=%d when '
                                 'warm_start==True'
                                 % (self.n_estimators,
                                    self.estimators_.shape[0]))
            begin_at_stage = self.estimators_.shape[0]
            self.estimators_fitted_ = begin_at_stage
            self.estimators_.resize((self.n_estimators, 1))
            self.train_score_.resize(self.n_estimators)
            if self.query_subsample < 1.0:
                self.oob_improvement_.resize(self.n_estimators)
            y_pred = self.predict(X)
        
        // 执行fit
        n_stages = self._fit_stages(X, y, sample_weight, qids, y_pred,
                                    random_state, begin_at_stage, monitor)

        // 如果实际生成的树数量小于初始化参数n_estimators，则将相应预分配的内部属性变更为实际大小
        if n_stages < self.estimators_.shape[0]:
            self.trim(n_stages)

        return self
```



```python
    def _fit_stages(self, X, y, sample_weight, qids, y_pred, random_state,
                    begin_at_stage=0, monitor=None):
        // 样本数量
        n_samples = X.shape[0]
        // 每次训练是否使用全部样本
        do_subsample = self.subsample < 1.0
        #sample_weight = np.ones(n_samples, dtype=np.float64)
        // 获取qid数量（unique；多少个query）
        n_queries = check_qids(qids)
        // 根据qid生成query组，其中元素为qid，在样本中起始下标，末尾下标，下标列表
        query_groups = np.array([(qid, a, b, np.arange(a, b))
                                 for qid, a, b in get_groups(qids)],
                                dtype=np.object)
        // 之前计算的qid数量应该和query组数量一致
        assert n_queries == len(query_groups)
        // 每次训练是否使用全部query
        do_query_oob = self.query_subsample < 1.0
        // query的mask，当不使用全部query时，使用该数组和query_groups获取所有要参加训练的样本，初始化全部为1，表示所有query均参与训练
        query_mask = np.ones(n_queries, dtype=np.bool)
        // query数组下标，在不使用全部quey时，需要进行打散，选择部分query组
        query_idx = np.arange(n_queries)
        // 选择全部query组中的比例
        q_inbag = max(1, int(self.query_subsample * n_queries))

        // 初始化输出
        if self.verbose:
            verbose_reporter = _VerboseReporter(self.verbose)
            verbose_reporter.init(self, begin_at_stage)
        
        // 执行每一个基础训练
        for i in range(begin_at_stage, self.n_estimators):
            // 如果不使用全部的query组
            if do_query_oob:
                // 将索引随机化
                random_state.shuffle(query_idx)
                // 重新初始化query的mask
                query_mask = np.zeros(n_queries, dtype=np.bool)
                // 选择指定部分的query进行训练
                query_mask[query_idx[:q_inbag]] = 1
            // 实际需要参与本轮训练的query组
            query_groups_to_use = query_groups[query_mask]
            // 实际需要参与训练的样本mask
            sample_mask = np.zeros(n_samples, dtype=np.bool)
            // 变量每一个query组
            for qid, a, b, sidx in query_groups_to_use:
                // 需要使用的样本下标
                sidx_to_use = sidx
                // 如果不使用全部样本，与query类似，对query组中的样本进行随机采样，但感觉这里有问题，因为取的最大值是b-1乘以采样系数，但每个样本中样本数量应该是b-a，但是如果使用b-a，也无法保证训练使用的样本数量是指定的比例，因为还有可能只是使用了部分的query。
                if do_subsample:
                    query_samples_inbag = max(
                        1, int(self.subsample * (b - 1)))
                    random_state.shuffle(sidx)
                    sidx_to_use = sidx[:query_samples_inbag]
                sample_mask[sidx_to_use] = 1


            // 记录未参与训练的样本的当前训练分数
            if do_query_oob:
                old_oob_total_score = 0.0
                for qid, a, b, _ in query_groups[~query_mask]:
                    old_oob_total_score += self.metric.evaluate_preds(
                        qid, y[a:b], y_pred[a:b])

            // 进行训练，并获取新的样本预估
            y_pred = self._fit_stage(i, X, y, qids, y_pred, sample_weight,
                                     sample_mask, query_groups_to_use,
                                     random_state)
            
            // 计算参与训练后，训练样本评分和未参与训练样本评分
            train_total_score, oob_total_score = 0.0, 0.0
            for qidx, (qid, a, b, _) in enumerate(query_groups):
                score = self.metric.evaluate_preds(
                    qid, y[a:b], y_pred[a:b])
                if query_mask[qidx]:
                    train_total_score += score
                else:
                    oob_total_score += score
            // 记录当前循环的参与训练的样本的分数
            self.train_score_[i] = train_total_score / q_inbag
            // 如果不是使用全部样本进行训练，则记录本轮训练对未参与训练样本的分数提升
            if do_query_oob:
                if q_inbag < n_queries:
                    self.oob_improvement_[i] = \
                        ((oob_total_score - old_oob_total_score) /
                         (n_queries - q_inbag))

            # 判断是否终止训练
            early_stop = False
            monitor_output = None
            if monitor is not None:
                monitor_output = monitor(i, self, locals())
                if monitor_output is True:
                    early_stop = True
            # 输出当前训练进度日志`	1qnm  Zxdsdwertyul;xcv
            if self.verbose > 0:
                verbose_reporter.update(i, self, monitor_output)

            if early_stop:
                break

        return i + 1
```

执行的训练函数

```python
    def _fit_stage(self, i, X, y, qids, y_pred, sample_weight, sample_mask,
                   query_groups, random_state):
        """Fit another tree to the boosting model."""
        assert sample_mask.dtype == np.bool

        n_samples = X.shape[0]

        all_lambdas = np.zeros(n_samples)
        all_deltas = np.zeros(n_samples)
        #根据query组计算lambda和lambda对模型的梯度
        for qid, a, b, _ in query_groups:
            lambdas, deltas = self._calc_lambdas_deltas(qid, y[a:b],
                                                        y_pred[a:b])
            all_lambdas[a:b] = lambdas
            all_deltas[a:b] = deltas
 
        # 构建回归决策树
        tree = sklearn.tree.DecisionTreeRegressor(
            criterion='friedman_mse',
            splitter='best',
            presort=True,
            max_depth=self.max_depth,
            min_samples_split=self.min_samples_split,
            min_samples_leaf=self.min_samples_leaf,
            min_weight_fraction_leaf=0.0,
            max_features=self.max_features,
            max_leaf_nodes=self.max_leaf_nodes,
            random_state=random_state)
        # 根据采样，确定模型权重，对于权重为0的数据，相当于没有参与训练
        if self.subsample < 1.0 or self.query_subsample < 1.0:
            sample_weight = sample_weight * sample_mask.astype(np.float64)
        # 回归树训练，拟合新的lambda
        tree.fit(X, all_lambdas, sample_weight=sample_weight,
                 check_input=False)
        # 更新模型
        self._update_terminal_regions(tree.tree_, X, y, all_lambdas,
                                      all_deltas, y_pred, sample_mask)
        self.estimators_[i, 0] = tree
        self.estimators_fitted_ = i + 1

#        print 'sample_weight:', sample_weight
        return y_pred
```

## 获取lambda及相对于当前模型的梯度

这里计算算法介绍中的$\lambda_i$和$w_i$。

其执行逻辑为：

```python
    # 输入为qid，y为样本真实label，y_pred为模型训练的输出label，y, y_pred均为一维数组
    def _calc_lambdas_deltas(self, qid, y, y_pred):
        ns = y.shape[0]
        '''
        获取按照预估的分数每个文档的排名（从0开始）
        def get_sorted_y_positions(y, y_pred, check=True):
            if check:
                y = sklearn.utils.validation.column_or_1d(y)
                y_pred = sklearn.utils.validation.column_or_1d(y_pred)
                sklearn.utils.validation.check_consistent_length(y, y_pred)
            # 对索引按照二维数组排序，先按照-y_pred再按照y排序
            return np.lexsort((y, -y_pred))
        '''
        positions = get_sorted_y_positions(y, y_pred, check=False)
        # 按照预估的排序对样本真实的相关性进行重排
        actual = y[positions]
        # 获取交换任意两个文档其 NDCG差值
        swap_deltas = self.metric.calc_swap_deltas(qid, actual)
        # 获取模型关注的文档数量（即query召回的前k个）
        max_k = self.metric.max_k()
        # 如果未设置数量，或者数量超过了query下召回的文档数量，则设置为召回文档数量
        if max_k is None or ns < max_k:
            max_k = ns

        lambdas = np.zeros(ns)
        deltas = np.zeros(ns)

        # 遍历每个需要考虑的排序前K个文档，计算与其他文档的lambda和梯度
        for i in range(max_k):
            for j in range(i + 1, ns):
                # 对于相关性一致则跳过
                if actual[i] == actual[j]:
                    continue
                # 获取文档位置互换的NDCG差值，这里并未取绝对值，需要后续判断
                delta_metric = swap_deltas[i, j]
                if delta_metric == 0.0:
                    continue
                // 获取文档在数据中实际位置
                a, b = positions[i], positions[j]
                # invariant: y_pred[a] >= y_pred[b]
                # 对于真实的相关性 i < j处理
                if actual[i] < actual[j]:
                    # 这时delta_metric应该大于0
                    assert delta_metric > 0.0
                    # 计算 1/(1+e^(sj - si)),注意scipy.special.expit = 1/(1+exp(-x)),即本身存在一个负号，因此输入时是si-sj
                    logistic = scipy.special.expit(y_pred[a] - y_pred[b])
                    # 计算 1/(1+e^(sj - si))* NDGC，因为delta_metric是正数，因此直接计算
                    l = logistic * delta_metric
                    # 对于a来说，其是真实相关性较差的文档，因此减去lambda
                    lambdas[a] -= l
                    # 对应b来说，其是真实相关性较好的文档，因此加上lambda
                    lambdas[b] += l
                else:
                    assert delta_metric < 0.0
                    logistic = scipy.special.expit(y_pred[b] - y_pred[a])
                    l = logistic * -delta_metric
                    lambdas[a] += l
                    lambdas[b] -= l
                # 计算梯度
                gradient = (1 - logistic) * l
                # 更新梯度
                deltas[a] += gradient
                deltas[b] += gradient

        return lambdas, deltas
```

这里的lambda是负梯度，deltas是正常的lambda对$s_i$的导数。

## 计算互换位置后NDCG值

```python
    def calc_swap_deltas(self, qid, targets):
        # 根据qid获取IDCG值
        ideal = self._get_ideal(qid, targets)
        if ideal < _EPS:
            return np.zeros((len(targets), len(targets)))
        # 计算替换后的NDCG差值
        return self._dcg.calc_swap_deltas(
            qid, targets, coeff=1.0 / ideal)
      
      
      def _get_ideal(self, qid, targets):
        # 如果_ideals中已经存在则不用计算直接返回
        ideal = self._ideals.get(qid)
        if ideal is not None:
            return ideal
        # 按照真实的文档相关性对文档进行排序（由大到小）
        sorted_targets = np.sort(targets)[::-1]
        # 计算IDCG值
        ideal = self._dcg.evaluate(qid, sorted_targets)
        # 写入_ideals
        self._ideals[qid] = ideal
        return ideal
 
class DCG(Metric):
      def evaluate(self, qid, targets):
        # 计算每个文档得分，相加
        # _gain_fn可选，直接使用x还是(2.0 ** x) - 1.0，通过初始化时指定gain_fn参数：identity：x；exp2：(2.0 ** x) - 1.0
        # _get_discount为对应位置的log(i+2)，位置从0开始
        return sum(self._gain_fn(t) * self._get_discount(i)
                   for i, t in enumerate(targets) if i < self.k)
      
      def _get_discount(self, i):
        if i >= self.k:
            return 0.0
        while i >= len(self._discounts):
            self._grow_discounts()
        return self._discounts[i]
      
      def _grow_discounts(self):
        self._discounts = self._make_discounts(len(self._discounts) * 2)
        
      def _make_discounts(self, n):
        return np.array([1.0 / np.log2(i + 2.0) for i in range(n)])
      
      
      def calc_swap_deltas(self, qid, targets, coeff=1.0):
        n_targets = len(targets)
        deltas = np.zeros((n_targets, n_targets))
        # 只查看关注的前k个文档与其他文档交换顺序的差值，使用上文介绍的方法来计算差值，这里并未取绝对值
        for i in range(min(n_targets, self.k)):
            for j in range(i + 1, n_targets):
                deltas[i, j] = coeff * \
                    (self._gain_fn(targets[i]) - self._gain_fn(targets[j])) * \
                    (self._get_discount(j) - self._get_discount(i))
```

## 更新模型

```python
    def _update_terminal_regions(self, tree, X, y, lambdas, deltas, y_pred,
                                 sample_mask):
        # 获取每个样本进过前向算法最终输出的叶子节点
        terminal_regions = tree.apply(X)
        masked_terminal_regions = terminal_regions.copy()
        # 对于未参与训练的样本，设置其输出的叶子节点为-1，即剔除
        masked_terminal_regions[~sample_mask] = -1
        # 遍历模型的每个叶子节点
        for leaf in np.where(tree.children_left ==
                             sklearn.tree._tree.TREE_LEAF)[0]:
            # 获取所有输出为该节点的样本
            terminal_region = np.where(masked_terminal_regions == leaf)
            # 计算其lambda的和
            suml = np.sum(lambdas[terminal_region])
            # 计算其梯度的和
            sumd = np.sum(deltas[terminal_region])
            # 设置叶子节点的输出值
            tree.value[leaf, 0, 0] = 0.0 if abs(sumd) < 1e-300 else (suml / sumd)
        # 更新模型的预测结果
        y_pred += tree.value[terminal_regions, 0, 0] * self.learning_rate
```

注意，这里

## 预测

训练完成后，或者验证时，执行预测的函数如下：

```python
def predict(self, X):
        # 校验输入
        X = sklearn.utils.validation.check_array(
            X, dtype=sklearn.tree._tree.DTYPE, order='C')
        score = np.zeros((X.shape[0], 1))
        # 确定实际回归树数量
        estimators = self.estimators_
        if self.estimators_fitted_ < len(estimators):
            estimators = estimators[:self.estimators_fitted_]
        # 执行预测
        sklearn.ensemble._gradient_boosting.predict_stages(
            estimators, X, self.learning_rate, score)

        return score.ravel()
```



## 终止训练判断

```python
class ValidationMonitor(object):
    """Monitor for early stopping via validation set."""
    def __init__(self, X, y, qids, metric, stop_after=100,
                 trim_on_stop=True):
        if len(X) == 0:
            raise ValueError("A validation set can not be empty!")
        self.X, self.y = sklearn.utils.check_X_y(
            X, y, dtype=sklearn.tree._tree.DTYPE)
        self.qids = qids
        self.metric = metric
        self.stop_after = stop_after
        self.trim_on_stop = trim_on_stop

        sklearn.utils.check_consistent_length(self.X, self.y, self.qids)
        check_qids(qids)

        self._query_groups = list(get_groups(self.qids))
        self._y_pred = None
        self._prev_iter = -1
        self._iter_scores = []
        self._best_score = None
        self._best_score_i = None
```

其中参数`trim_on_stop`表示是否允许提前终止，`stop_after`表示当前训练的轮数，与效果最好的一轮之间差的轮数如果超过该值则停止训练（`trim_on_stop`为true时）。

具体终止判断的逻辑如下：

```python
    # 返回true表示终止训练
    def __call__(self, i, model, localvars):
        """Returns True if the model should stop early.

        Otherwise, returns a status string.

        """
        # 当前轮数
        assert i == self._prev_iter + 1
        self._prev_iter = i
        # 如果是加法模型，则对预测值进行累加
        if isinstance(model, AdditiveModel):
            if self._y_pred is None:
                self._y_pred = model.predict(self.X)
            else:
                self._y_pred += model.iter_y_delta(i, self.X)
            y_pred = self._y_pred
        else:
            y_pred = model.predict(self.X)
        # 计算每个query下文档的NDCG值，并加和
        score = 0.0
        for qid, a, b in self._query_groups:
            sorted_y = get_sorted_y(self.y[a:b], y_pred[a:b])
            score += self.metric.evaluate(qid, sorted_y)
        # 输出当前轮总的NDCG值
        print "monitor score: ", score
        # 计算平均每个query的NDCG值
        score /= len(self._query_groups)
        # 记录最好的轮次和其分数
        if self._best_score is None or score > self._best_score:
            self._best_score = score
            self._best_score_i = i
        # 记录当前轮次与最好轮次之间差距
        since = i - self._best_score_i
        # 如果允许提前终止，并且达到终止条件，则终止
        if self.trim_on_stop and \
                (since >= self.stop_after or i + 1 == model.n_estimators):
            # 将模型按照实际回归树数量缩小
            model.trim(self._best_score_i + 1)
            return True
        # 如果未终止，则返回当前训练状态 C为query的平均NDCG，B为最好的分数，S为举例之前的最好分数的轮次
        return 'C:{:12.4f} B:{:12.4f} S:{:3d}'.format(
            score, self._best_score, since)
```

## 日志输出

日志输出类如下：

```python
class _VerboseReporter(object):
    """Reports verbose output to stdout.

    If ``verbose==1`` output is printed once in a while (when iteration mod
    verbose_mod is zero).; if larger than 1 then output is printed for
    each update.

    """
    def __init__(self, verbose):
        self.verbose = verbose

    def init(self, est, begin_at_stage=0):
        # header fields and line format str
        header_fields = ['Iter', 'Train score']
        verbose_fmt = ['{iter:>5d}', '{train_score:>12.4f}']
        # do oob?
        if est.query_subsample < 1:
            header_fields.append('OOB Improve')
            verbose_fmt.append('{oob_impr:>12.4f}')
        header_fields.append('Remaining')
        verbose_fmt.append('{remaining_time:>12s}')
        header_fields.append('Monitor Output')
        verbose_fmt.append('{monitor_output:>40s}')

        # print the header line
        print(('%5s ' + '%12s ' *
               (len(header_fields) - 2) + '%40s ') % tuple(header_fields))

        self.verbose_fmt = ' '.join(verbose_fmt)
        # plot verbose info each time i % verbose_mod == 0
        self.verbose_mod = 1
        self.start_time = time.time()
        self.begin_at_stage = begin_at_stage

    def update(self, j, est, monitor_output):
        """Update reporter with new iteration. """
        if monitor_output is True:
            print('Early termination at iteration ', j)
            return
        do_query_oob = est.query_subsample < 1
        # we need to take into account if we fit additional estimators.
        i = j - self.begin_at_stage  # iteration relative to the start iter
        if self.verbose > 1 or (i + 1) % self.verbose_mod == 0:
            oob_impr = est.oob_improvement_[j] if do_query_oob else 0
            remaining_time = ((est.n_estimators - (j + 1)) *
                              (time.time() - self.start_time) / float(i + 1))
            if remaining_time > 60:
                remaining_time = '{0:.2f}m'.format(remaining_time / 60.0)
            else:
                remaining_time = '{0:.2f}s'.format(remaining_time)
            if monitor_output is None:
                monitor_output = ''
            print(self.verbose_fmt.format(iter=j + 1,
                                          train_score=est.train_score_[j],
                                          oob_impr=oob_impr,
                                          remaining_time=remaining_time,
                                          monitor_output=monitor_output))
            if i + 1 >= 10:
                self.verbose_mod = 5
            if i + 1 >= 50:
                self.verbose_mod = 10
            if i + 1 >= 100:
                self.verbose_mod = 20
            if i + 1 >= 500:
                self.verbose_mod = 50
            if i + 1 >= 1000:
                self.verbose_mod = 100
```

输出信息如下：

[![5qj7Z9.png](https://z3.ax1x.com/2021/10/28/5qj7Z9.png)](https://imgtu.com/i/5qj7Z9)

其中输出的每个字段含义如下：

| 字段        | 含义                                                         |
| ----------- | ------------------------------------------------------------ |
| Iter        | 迭代轮数                                                     |
| train score | 平均的NDCG分数（越高约好）                                   |
| OOB Improve | 每次与上一步迭代提升的train_score，即delta train_score；     |
| Remaining   | 根据当前训练轮数和花费时间预估训练完成时间                   |
| C           | 验证集的平均NDGC                                             |
| B           | 历次迭代中验证集中最高的平均NDGC                             |
| S           | 迭达到当前步数，距离验证集最高的平均ndgc迭代步数的差(i-best_socre_i) |

# 存储模型并输出相关效果

使用pickle包存储模型。

```python
def save_model(model, file):
    save_file = open(file, "wb")
    pickle.dump(model, save_file)
    save_file.close()
```

输出每个因子的重要性。

```python
id2featrue = {}
    for line in open('../dicts/featrue2id'):
        line = line.strip()
        featrue,id = line.split()
        id = int(id)
        id2featrue[id] = featrue
    for i in np.argsort(model.feature_importances_):
        print id2featrue[i],model.feature_importances_[i]
```

其中中存储了featureid到featurename的映射关系：

```
ctr_norm 0
title_oral_tight_term_hit_count_norm 1
cqr_norm 2
pcqr 3
likeRank 4
title_oral_offset_value 5
title_oral_boost_term_hit_count 6
title_oral_offset_value_norm 7
tag_match_score 8
cqr_ 9
title_oral_levenshtein_distance_norm 10
```

# 模型转换

gbrank只支持gbrank模型，因此需要将训练完成的lambdaMART模型进行一下转换。

修改`notes/bin/trans_model_format.sh`中的输入和输出模型即可。

执行对应脚本即获得可以直接在vsrank中使用的模型。
