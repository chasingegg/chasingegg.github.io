---
author: "Chao.G"
title: "An Empirical Evaluation of Columnar Storage Formats"
date: "2025-04-12"
draft: false
tags: ["database"]
categories: ["Papers"]
---

列存格式比如Parquet和ORC诞生在“遥远”的2010年代，那还是所谓的大数据的年代，提供了开放的数据格式，上面可以用不同的分析引擎做数据分析。显然这些文件格式是否还能适应于今天的场景是一个问号，现在和十几年前已经发生了很大的改变，首先是硬件的变化，存储介质是更快的SSD以及大量数据存储在云上高吞吐低成本的对象存储，IO和计算的trade off需要重新考虑。数据上会有更多的非结构化数据出现，对这类数据高效的存储和分析会有很大的不同。

论文列出了列存格式的一些设计思路和实现原则，也构造了数据对每个点进行实验验证性能，最重要的是作者给出的一些对于未来设计列存数据格式的建议。


## 列存格式的设计

![exp](/assets/columnar-layout.png)

### Format layout

Parquet和ORC都是PAX（Partition Attributes Across）的格式，即先把数据进行水平划分为若干个Row Group，在Row Group里再按照列划分，是一个hybrid的模式，既可以在scan某一列的时候充分向量化执行，对于要访问多个列属性的时候，可以在Row Group的粒度组织tuple。对于每个column chunk首先会做encoding，然后再用通用的block compression来进一步压缩大小。存储meta信息的是footer，包括文件级别的meta以及每个Row Group的一些统计信息比如min/max等。Parquet使用行数来分割Row Group，而ORC用的是存储大小。

### Encoding

对数据做编码可以减少存储和IO开销，常见的技术比如Dictionary Encoding，Run-Length Encoding，Bitpacking会被用到。这里简单解释一样这几种算法，Dictionary Encoding是把比较大的属性比如string用ID来表示，如果这个属性上的值的基数比较小的话，这样就可以减少这种大属性的存储。Run-Length Encoding是把连续重复的值比如AAAABB表示成4A2B。Bitpacking是把小数值用更小的位数来表示，比如数字3只需要两个bit就可以表示而不需要int32的存储。

Parquet会对各种类型的column chunk都做Dictionary Encoding，而ORC只会用到string类型。除了Dictionary Encoding之外，ORC会采用一个hybrid的方式来做编码，根据数据分布的特征来换不同的算法，这会让ORC可以有更大的机会做更极致的压缩，但是这会对decoding阶段有更重的overhead。

![exp](/assets/columnar-encoding.png)

### Compression

这里的压缩区别于上面的编码，是一些通用的压缩算法，比如zstd。这类算法把数据当成字节流来处理，可以直接作用在任意的文件格式上。

### Index and Filter

Parquet和ORC会用zone map和bloom filtering做数据剪枝。Zone map包含min/max值以及一些预定义range的行数，用于直接跳过某些zone，这个zone map会用到文件级别和Row Group级别，而最小的zone map在Parquet可以到一个Page，在ORC中是一个可配置的行数。

### Nested Data Model

对于嵌套数据类型比如Json的支持，Parquet的方式基于Dremel论文，把atomic field（嵌套结构中的叶子节点）作为一个单独列，每个列包含repetition level和definition level两个属性，repetition level代表重复次数，definion level用来表示是否是NULL。ORC的存储方式更加直观，对于每个field，对应一个bool列表示是否有值，然后对于可重复字段，记录重复次数。

ORC会为non-atomic field创建额外的列来存，在查询的时候可能会多读一些列。但是Parquet往往会产生更大的文件，因为non-atomic field的信息可能在多个atomic field会有重复。

![exp](/assets/columnar-nested.png)

## Benchmark and lessons learned

在设计benchmark的时候，作者采用了现实的数据集，通过分析这些数据集的特征，比如NDV（Non distinct value）ratio，Null Ratio，Value Range，Sortedness，Skewness，分类了几种workloads。测试的其实就是(filtered)scan的性能。

![exp](/assets/columnar-workloads.png)

对于上面的每个设计的点，都有对应的评估。这里简单列一些实验结果和作者的观察和讨论。

Encoding上Parquet的压缩率大多好于ORC，因为现实中的数据集往往NDV比较低，这时候做字典编码会比较有效，尤其是在float的数据上优势非常明显。并且因为Parquet没有像ORC用更多复杂的编码算法，在decoding上也有优势。其实这给人一种感觉是ORC玩了很多花活，但是可能在现实数据集里字典编码才是真正非常实用有效的，而在这个算法上ORC只把它在string类型上，就有点吃亏。

![exp](/assets/columnar-encoding-result.png)

Block compression的效果就很不明显，这是因为大部分数据已经经过编码，留给进一步压缩的空间已经很小，decoding还会引入比较大的overhead。测试下来会发现只有在很慢的云盘上，block compression能够有一定作用，大部分的快盘上已经是计算瓶颈了，这类压缩算法完全应该被弃用。但是注意到，现在更多的数据是在对象存储上，由于高延迟的读取，读文件时候需要好几轮比如footer length，footer，column chunks，即使通过多线程打满对象存储的带宽，IO开销仍然是比较大的。这时候需要重新设计文件格式可以把meta存在一起，不需要多次读取，并且选择合适的Row Group或文件大小去适应对象存储的读取粒度。

在Index和filter的支持上，zone map和bloom filter的作用主要是在低选择率的查询。未来的文件格式可以考虑更多Index和filter的数据结构。对于Nested数据类型，需要尽量减少存储和内存里格式转换的开销，因为Arrow已经成为内存格式的一个标准，因为ORC不像Parquet直接转换成Arrow格式，中间需要多转换一次，开销会大一些。

最后重点说下AI有关的workloads，这也是这几年数据领域非常火的方向，怎么做好AI时代的数据基建，有一些场景是需要重新考虑的。

### 宽表Projection

在机器学习feature的迭代过程中，很可能产生非常多的属性特征，反映在数据上就是一个大宽表。作者生成了不同列属性的几张宽表，然后取其中的10个属性，这时候会发现meta的解析会几乎随整张表的属性个数线性增长。主要是因为footer结构不能很好地支持随机读，需要完整读取meta，所以未来的文件格式需要考虑如何组织meta信息，可以随机访问某个列的meta数据，提供宽表场景下的projection。

### 向量Embedding

向量已经成为一个基础的数据类型，但是block compression不能提供很好的压缩率。这个点上我感觉就是和普通数据类型的encoding和block compression的关系可以对应起来，重点可能在encoding上，也就是常说的向量量化的技术，对数据压缩有很好的效果。

和向量检索ANN结合，搜索结果的ID去拉取实际的向量数据，这里主要就是随机读的能力。结果可以发现拉数据的时间还是很显著的，在SSD上ORC会更好，因为ORC有更加细粒度的zone map来减少读放大。而在对象存储s3上，这个结果就反过来了，因为zone map在ORC的存储里是在每个Row Group的footer，但是Parquet是在整个file的footer上，可以减少读s3的次数。

![exp](/assets/columnar-vector.png)


### 非结构化数据

另一类数据就是常见的图片，视频等非结构化数据。这类的数据很大，需要重新设计一个合适的Row Group大小，比较小的话更有利于多线程读取的优势。但是普通列就不是这样了，太少的数据会影响压缩的效果。这里作者提供了一个想法，对这些大blob数据需要和普通列分开存储，有不一样的物理layout，对外表现一个统一的接口。

![exp](/assets/columnar-unstructured.png)

## 总结

这篇文章重新evaluate了两种列式存储格式Parquet和ORC在不同workloads的细粒度的表现。里面有不少可以优化的地方，尤其是在现在AI的场景下，甚至重新设计列存格式是很有必要的，也是lance format的一个很好的切入点。Data for AI这一套infra其实从存储到计算都是可以有新玩家的出现，里面还是有着很多机会。

## Reference

- https://www.vldb.org/pvldb/vol17/p148-zeng.pdf
- https://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/36632.pdf
- https://mp.weixin.qq.com/s/1oL9cTmmkecG8qCU6LAgdw
- https://lancedb.github.io/lance/format.html