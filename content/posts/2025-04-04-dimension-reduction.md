---
author: "Chao.G"
title: "Dimensionality-Reduction Techniques for Approximate Nearest Neighbor Search"
date: "2025-04-04"
draft: false
tags: ["ann"]
categories: ["Papers"]
---

今天介绍一篇介绍降维的[survey文章](http://sites.computer.org/debull/A24sept/p63.pdf)。高维向量，尤其是最新的大模型产生的向量，维度正在越来越高，正在从几百维增加到上千维，那么降维就会是提高效率的一个很重要的手段。论文使用了常见的六种降维的手段做了测试，并提出一些未来的研究方向。

## 常见的降维算法

首先可以介绍一个Johnson-Lindenstrauss引理：塞下N个向量，只需要O(logN)维的空间，并将相对距离的误差控制在一定范围内。这是一个非常有用的结论，点明了降维的可行性，可以看这个具体的[数学解读](https://zhuanlan.zhihu.com/p/413581747)。

- PCA(Principal Components Analysis)：是一种经典的线性变换方法，其核心思想是为高维向量空间中的数据选择一组新的向量基底。具体而言，该方法以数据协方差矩阵的特征向量作为新的基底，并将具有较大特征值（即较高方差）的维度排列在新向量坐标的前端位置。
- DWT(Discrete Wavelet Transform)：通过对向量进行分层小波变换分解，将主要波动成分置于变换结果的前端位置。该变换能保持向量在时域和频域中的距离特性。与主成分分析（PCA）相比，DWT并非线性变换，无需在内存中存储变换矩阵，从而避免了额外的内存开销。
- ADSampling(https://arxiv.org/pdf/2303.09855)：采用随机生成的方阵作为变换矩阵，其中每个元素均采样自标准高斯分布。其随机投影对距离偏差具有特定形式的概率误差bound。在此方法中，可通过部分维度上的距离计算结果，以一定的（概率性）置信度估计精确距离值。计算维度越多，置信度越高。因此，ADSampling提供的是概率性精度保证，而非确定性保证（如PCA和DWT）。
- PM-LSH：PM-LSH通过计算向量与一组随机向量（即哈希函数族）的内积，直接降低数据维度。
- SEANet: 一种深度学习的方法，以降维前后的距离偏移作为loss function训练。
- OPQ：OPQ是Product Quantization的升级版，PQ通过切分向量维度，每个子向量通过聚类做编码。

## 如何集成降维到ANN查询

文章提出了两种方式

### In-Place Transformation

如下图所示，在建索引阶段对数据做预处理，而这些变化可以保证distance-preserving，把重要的维度集中到向量靠前的维度。在查询的时候，使用向量靠前的维度计算距离作为下界剪枝，如果不能early stop那就会把全维度都计算。所以这一类方法直接处理了原始数据，然后取前面的维度来提高效率。

![inplace](/assets/dimension-reduction-inplace.png)

### Out-of-Place Transformation

另一种方式把降维后的向量作为一个额外的存储，仍然保有原始向量数据。通过降维后的向量计算estimated distance，看能否剪枝来决定是否要计算原始向量的距离。

![outofplace](/assets/dimension-reduction-outofplace.png)

## 实验

从结果来看，一些观察是对于特别高维度并且简单的数据，比如Trevi数据集，DWT和PM-LSH的效果比较好，主要源于它们非常高效的预处理，其余的方法需要在高recall下才能有效果。在较低维度的数据集上，OPQ，PCA和ADSampling没有展示出超过20%的优势，有时候还会有副作用。

![r1](/assets/dimension-reduction-r1.png)

具体来看，降维后的向量距离和原向量距离的误差，会发现PCA会比DWT要准不少，但是DWT可以增加维度来提高精度，可以发现深度学习方法SEANet可以提供很准确的距离估计。

![r2](/assets/dimension-reduction-r2.png)

对于未来的研究方向，可以去思考降维和量化算法的结合，深度学习做降维的可能性，以及如何选择最合适的算法，需要同时考虑精度以及性能两个方面。

## Reference

- http://sites.computer.org/debull/A24sept/p63.pdf
- https://arxiv.org/pdf/2303.09855
- https://zhuanlan.zhihu.com/p/413581747
- https://zhuanlan.zhihu.com/p/416924689