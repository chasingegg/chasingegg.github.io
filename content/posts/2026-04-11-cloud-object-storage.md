---
author: "Chao.G"
title: "Exploiting Cloud Object Storage for High-Performance Analytics"
date: "2026-04-11"
draft: false
tags: ["database"]
categories: ["Papers"]
---

云原生这个词现在听起来已经是个老词了，现代的数据库系统向云端基础设施迁移，是一次根本性的架构演进。云原生的数据系统的市场份额已经全面超过了本地部署系统，这种基于云的范式是因为云提供的弹性，可扩展性，计算和存储分离的按需调配等一系列好处，而这里一个很重要的角色就是云对象存储，主流的云服务提供商比如aws，gcp，azure都提供了对象存储服务，极具成本优势，近乎无线扩展，且高度耐用。如何基于对象存储来做性能优化，就需要了解下对象存储的特性，今天这篇[论文](https://www.vldb.org/pvldb/vol16/p2769-durner.pdf)来自著名的TUM数据库组，作者充分调研以后，然后基于这些实验观察开发了一个开源的对象存储下载工具[AnyBlob](https://github.com/durner/AnyBlob)，以很低的CPU占用达到很高的吞吐，并集成到[Umbra](https://umbra-db.com/)数据库系统中，不依赖本地SSD做cache就可以达到很好的性能。

而今天这个分享，主要是截取文中一些对象存储相关特性的总结作为cheat sheet，这里的数据在现在看来还是基本不过时的。而文中提出的AnyBlob库即便可能性能真的不错，但真的要实际用大概率还得用原生的s3 client稳妥点，但是不得不吐槽一句，这client的成熟度倒真的完全不如s3本身的成熟度，但也只能耐着头皮用啊。

## 对象存储的收费

所有对象存储的收费是类似的，按照请求收费，而跟请求的大小无关。而这个成本与其他的存储相比是最划算的，比如EBS和NVME SSD，只有HDD的价格接近，但是HDD带宽非常受限，并且无法提供对象存储的持久性（比如s3可以提供11个9），相比对象存储还是差了一点。

![s3-paper-storage-cost](/assets/s3-paper-storage-cost.png)

## 延迟 

作者对不同大小的请求的延迟分布做了充分测试，延迟分成首字节到达延迟和全量数据接收完成延迟。当请求比较小的时候，比如1KB，首字节延迟占据统治地位，网络数据包的传输是瓶颈。而随着请求变大，带宽开始成为限制因素，请求大小从8MB到16MB时，延迟变成1.9倍，说明带宽利用率达到上限。所以这说明了请求大小其实是有一个最优值的。

![s3-paper-latency](/assets/s3-paper-latency.png)


## 吞吐

对象存储的吞吐基本就卡在网络带宽上，aws上的大实例可以达到10GB/s（注意这里是大B）以上，可以认为跟大实例的本地NVME盘的磁盘带宽接近。

## 最佳的请求size

根据前面的数据，最佳的请求size其实已经基本可以得出了，一方面size要求尽量大，因为计费是按照请求计算的，另一方面也不能是太大的请求，会造成读放大，即使是没有读放大，也可以通过减小单请求size增大并发度来提升读取的性能。在请求大小很小的时候，对象存储的费用比较高，请求比较大的时候，实例本身的费用才开始dominate。所以最佳的请求size在8-16MB左右。

![s3-paper-throughput](/assets/s3-paper-throughput.png)

## 加密

加密也是吃走CPU的罪魁祸首之一，作者测试下来使用https连接会比http多使用一倍的CPU。而在aws里，跨越region或者AZ的所有网络流量，都已经在云提供商的物理网络基础设施层面被强制且透明地进行了硬件级自动加密。所以在ec2实例和s3网关之间的网络数据传输本身就是安全的，不需要使用https协议。作者认为，只需要保证数据存储的安全性，比如放在s3上的时候不被第三方窃取，这也是很多合规的要求，只需要上传到对象存储前用AES等算法做静态加密，查询的时候再做解密，而在网络传输这一层，只需要http协议即可，这样CPU的overhead会小很多。

![s3-paper-encryption](/assets/s3-paper-encryption.png)

## 如何用满带宽

如何用并发的请求打满带宽，可以用下面这个公式简单计算出来，base latency可以认为是网络传输的固定开销，大概是~30ms，而传输数据的延迟在~20ms/MB，这样就可以计算出单位size的延迟，乘上带宽上限，就是并发请求数。论文里给出的例子是如果是8MB的请求大小，大约需要200-250的并发请求可以打满10GB/s的网络带宽。

$$requests = throughput \cdot \frac{baseLatency + size \cdot dataLatency}{size}$$

而AnyBlob就是通过异步IO避免开出数百个线程来做数据下载，减少了CPU的占用。

## 总结

这篇文章对于对象存储的特性做了很充分的实验，这对于想做基于对象存储的性能优化的人来说是一个很好的指南。但是需要指出的是，因为文章主要针对OLAP的workload，是以Scan为主，所以当现在网络带宽和本地NVME SSD盘带宽基本相当的前提下，不用任何cache可以达到类似的性能，但是读对象存储跟读盘的区别就在于CPU的使用上要高出很多，于是AnyBlob做了不少的优化。但是需要做随机点查的话，本地盘可以提供更加细粒度的KB级别的读取，对象存储更加适合MB级别的读取，这里会有读放大，并且对象存储的延迟比本地盘的延迟高出几个数量级。所以很多时候需要对文件layout有一个好的设计以及需要一个很好的cache系统，后面可以来调研一下。

## Reference

- https://www.vldb.org/pvldb/vol16/p2769-durner.pdf
- https://github.com/durner/AnyBlob
- https://umbra-db.com
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/data-protection.html#encryption-transit