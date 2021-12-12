# 基于LSM的KV存储写放大优化 - tiering

- 时间：2021.12.10
- 分享人：牟泽培
- 关键字：LSM、Compaction
- 分享PPT：[2021-12-10-LSM compaction 设计](./slides/2021-12-10-LSM compaction 设计.pdf)

## 分享内容

### 问题描述
compaction是LSM存储体系的核心，也从根本上影响LSM存储引擎的性能，包括读写性能，空间占用。  
因此，选择适当的压缩策略是至关重要的，同时也很困难，因为LSM compaction设计空间非常大，在文献中也没有正式定义。
因此，大多数基于lsm的引擎使用固定的压缩策略，这通常是由工程师亲自选择的，它决定如何以及何时压缩数据。  
本次分享主要基于[Constructing and Analyzing the LSM Compaction Design Space（VLDB2021）](http://vldb.org/pvldb/vol14/p2216-sarkar.pdf) 对LSM compaction做一个比较完整的定义，分析评估当前多种先进compaction策略的性能情况。算是对LSM compaction设计提供一个指南。


### 内容

#### 背景：

1.LSM简介、LSM 的核心compaction介绍，定义：触发器+数据布局+粒度+数据移动策略。  
2.RocksDB Universal-compaction 详解，单level/多level，主要参考[universal-compaction](https://github.com/facebook/rocksdb/wiki/Universal-Compaction)  
3.当今最先进的数种compaction策略的组成形式和相关的性能分析，包括读/写/空间放大，读写性能，compaction性能。  
4.不同的工作负载对compaction策略的影响，包括不同的写入/查询分布，更新/删除 比例。  

## FAQ

1.Q:关于RocksDB的universal compaction问题，在数据量较大的情况下Tiering策略写放大优势不复存在？  
A: universal compaction的tiering和一般的tiering差距较大，有可能涉及多个level的compaction，在数据量大的情况下IO代价高比较正常，这是它的设计理念之一，核心目的是提供更低的空间放大。  
详情可参见官方文档[universal-compaction](https://github.com/facebook/rocksdb/wiki/Universal-Compaction)  
