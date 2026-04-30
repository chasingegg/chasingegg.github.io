---
author: "Chao.G"
title: "Extended RabitQ SIGMOD 2025"
date: "2026-03-29"
draft: false
tags: ["ann"]
categories: ["Papers"]
---

今天分享下RabitQ的后续[扩展工作](https://arxiv.org/pdf/2409.09913)，上一次对于RabitQ的分享文章已经过去一年多，但是当时还没想到这会是近年来向量检索领域的最大突破，可以说没有之一。不得不说在计算机领域里面很多人还是走一个比较朴实的路线，出于一些比较直观的想法来做算法，而数学出身的人在这种时候就有更大的机会去发现一些不太直观的东西，但是当把事情讲明白之后，其实你会觉得某种程度上这也是很直观的...比如加一个random ratation去“抹掉”不同维度之间的重要性差别，使之在空间中呈现出较为均匀分布的状态，自然就做量化变得容易多了，非常美而简单的思路。这几年以来ANN这个领域论文已经是井喷的状态，现在所谓的SIGMOD，VLDB早已不是什么稀罕事了，已经成为了烂大街的状态，但是你的SIGMOD和他的SIGMOD也许并不是一个级别的工作，而这时候需要从业者有一些判断能力，目前看起来AI可以帮你快速地把论文理解了，但这是一个好文还是水文是需要人的经验的。

## Recap

RabitQ原文是提出了做binary量化的方案，但是binary量化对于精度的影响还是比较大的，所以这个后续工作就是为了做多bit的量化，来达到存储和精度的balance。先来回顾下RabitQ做binary量化的思路。

### 归一化

$$o := \frac{o_r - c}{\|o_r - c\|}, \quad q := \frac{q_r - c}{\|q_r - c\|}$$

其中 $c$ 为质心向量，$o_r$ 和 $q_r$ 分别为原始数据向量和查询向量。把欧式距离分成如下的公式，问题转化成归一化后向量的inner product。

$$\|o_r - q_r\|^2 = \|(o_r - c) - (q_r - c)\|^2 \tag{1}$$

$$= \|o_r - c\|^2 + \|q_r - c\|^2 - 2 \cdot \|o_r - c\| \cdot \|q_r - c\| \cdot \langle q, o \rangle \tag{2}$$

### 码本构造

$$C_r := \{Px \mid x \in C\}, \quad \text{where } C := \left\{+\frac{1}{\sqrt{D}}, -\frac{1}{\sqrt{D}}\right\}^D \tag{3}$$

其中 $P$ 是随机正交矩阵（Johnson-Lindenstrauss 变换）。而实际上我们只需要存储这个矩阵即可，只需要根据向量做完矩阵旋转之后的每个维度的正负号就可以确定量化后的表示。

### 无偏估计器

$\frac{\langle \bar{o}_0, q \rangle}{\langle \bar{o}_0, o \rangle}$ 是 $\langle o, q \rangle$ 的**无偏估计器**。

以至少 $1 - \exp(-c_0 \epsilon_0^2)$ 的概率，误差界为：

$$\left| \frac{\langle \bar{o}_0, q \rangle}{\langle \bar{o}_0, o \rangle} - \langle o, q \rangle \right| \leq \sqrt{\frac{1 - \langle \bar{o}_0, o \rangle^2}{\langle \bar{o}_0, o \rangle^2}} \cdot \frac{\epsilon_0}{\sqrt{D-1}} \tag{4}$$

其中 $c_0$ 是常数，$\epsilon_0$ 是控制失败概率的参数。

### 内积计算

最后就是如何高效地做$\langle q, \bar{o}_0 \rangle$的计算。
$$\langle q, \bar{o}_0 \rangle = \left\langle q, P\left(\frac{2}{\sqrt{D}}\bar{x}_b - \frac{1}{\sqrt{D}}1_D\right) \right\rangle \tag{5}$$
$$= \frac{2}{\sqrt{D}} \langle q', \bar{x}_b \rangle - \frac{1}{\sqrt{D}} \sum_{i=1}^{D} q'[i] \tag{6}$$

其中 $q' = P^{-1}q$，$q'[i]$ 表示向量 $q'$ 的第 $i$ 个分量。


## Extended RabitQ

而Extended RabitQ希望能做到在$D$维空间构造出可以表达$2^{B*D}$空间的codebook，而同时仍然可以保持binary量化的时候良好的无偏和error bound的性质，以及高效的计算。所以需要满足这个codebook仍然是由归一化后的向量通过正交矩阵旋转得到，然后在计算时可以直接通过量化后的表示直接进行计算，而不需要decompression。

### 码本构造

下图就是一个图示，

![extened-rbq-codebook](/assets/extended-rbq-codebook.png)

首先构造出均匀的网格集合，

$$\mathcal{G} := \left\{ -\frac{2^B - 1}{2} + u \;\middle|\; u = 0, 1, 2, 3, ..., 2^B - 1 \right\}^D \tag{7}$$

再做归一化和旋转，使之贴合在高维空间下的单位球面上。需要注意的是，这里为了前面提到的性质实际上对量化表达的范围是有损失的。观察图可以直观看到高维空间下某些网格点经过处理之后变成了同一个点。

$$\mathcal{G}_r := \left\{ P \frac{y}{\|y\|} \;\middle|\; y \in \mathcal{G} \right\} \tag{8}$$

### 量化后的表示

对于数据向量 $o$，寻找码本中最近的量化向量 $\bar{o}$：

$$\bar{y} = \arg\min_{y \in \mathcal{G}} \left\| P\frac{y}{\|y\|} - o \right\|^2 = \arg\min_{y \in \mathcal{G}} \left( 2 - 2\left\langle P\frac{y}{\|y\|}, o \right\rangle \right) \tag{9}$$

而找到这个量化向量就不像binary向量那样看个正负号那么容易了，如何高效地做这件事呢？这在作者的[blog](https://dev.to/gaoj0017/extended-rabitq-an-optimized-scalar-quantization-method-83m)也有一个直观的解释。对于图中数据点x，在均匀的网格点上最近的是A，但是在码本上最近的是B，而均匀的网格上去找到这个最近的点是非常容易的，类似于看正负号就能获得binary值，但是在码本上找到这个量化点很难。作者提供的方案是我们总可以找到rescaling factor t，把数据点x通过rescaling到某个地方，然后通过这个rescale的点找最近网格点的方式找到实际的量化点。而我们并不需要遍历rescaling factor，因为只需要rescaling导致最近的量化点发生变化，去到新的网格点的时候再去判断距离，然后对于每一个维度就是求$2^B-1$个距离的最小值即可找到量化点。

<div style="display: flex; gap: 10px; justify-content: center;">
  <img src="/assets/extended-rbq-rescaling1.png" alt="extened-rbq-rescaling1" style="width: 50%;">
  <img src="/assets/extended-rbq-rescaling2.png" alt="extened-rbq-rescaling2" style="width: 50%;">
</div>


### 内积计算

$$\langle \bar{o}, q \rangle = \left\langle P\frac{\bar{y}}{\|\bar{y}\|}, q \right\rangle = \left\langle \frac{\bar{y}}{\|\bar{y}\|}, P^{-1}q \right\rangle = \frac{1}{\|\bar{y}\|} \langle \bar{y}, q' \rangle \tag{11}$$

$$= \frac{1}{\|\bar{y}\|} \left( \langle \bar{y}_u, q' \rangle - \frac{2^B - 1}{2} \sum_{i=1}^{D} q'[i] \right) \tag{12}$$

其中$q' = P^{-1}q$，$\bar{y}_u = \bar{y} + \frac{2^B - 1}{2} \cdot 1_D$ 是无符号整数表示的量化码。除去可以预计算的部分以及针对query的一次性计算成本，仍然可以通过query向量和量化向量的内积来计算近似距离。

### 查询加速

论文中作者还提到了一个很实用的trick，可以把多bit的量化数据拆解成两个部分，1bit的部分等同于原始的RabitQ，把它用于做初筛，然后用error bound来决定是否要抽取后面的多bit数据来做精排，可以进一步地提升查询性能。

![extended-rbq-trick](/assets/extended-rbq-trick.png)


## 实验结果

首先论文比较了各种算法的误差，rabitq-pad是作者提出的一个baseline，通过在原向量后面补0，然后在这个扩充后的向量再做binary量化得到的相对于原始向量的多bit向量。

![extended-rbq-space-accuracy](/assets/extended-rbq-space-accuracy.png)

然后是recall-qps的曲线，优势明显

![extended-rbq-qps-recall](/assets/extended-rbq-qps-recall.png)

## 总结

如果说RabitQ第一篇文章给出了非常漂亮的理论证明，并且打开了部分落地的可能，那这一篇后续工作其实是把RabitQ工程落地的可能性变得非常高了，通过对于多bit不同比特数的调控，可以有足够的空间做到对存储，性能，recall的trade off。

而下一步能做的是一些data dependent的工作，所以RabitQ算是开辟了方向，它达到的效果是数学上的解释很漂亮，因此有很好的通用性，也因此更容易去做查询加速。而data dependent显然是继续灌水比较好的思路，在recall的角度上来说还会有进一步的压缩空间。

## Reference

- https://arxiv.org/pdf/2409.09913
- https://dev.to/gaoj0017/extended-rabitq-an-optimized-scalar-quantization-method-83m
