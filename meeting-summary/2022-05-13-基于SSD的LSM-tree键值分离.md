# 2022.05.13 分享纪要

- 分享人：李林峰
- 关键词：SSD,LSM-tree,KV Separate,compaction
- 分享PPT: [基于SSD的LSM-tree键值分离](./slides/2022-05-13-基于SSD的LSM-tree键值分离.pdf)

## 分享内容： 基于SSD的LSM-tree键值分离

## 1. 背景

### 1.1 存储结构

1. 通常情况下，数据库存储引擎处理更新采用两种方法：就地更新（in-place updates）和异地更新（out-of-place updates）。就地更新结构（比如B+树）是直接将新的数据覆盖到原有的位置，这样虽然会带来好的查询性能，但是这样做导致随机IO，会极大降低写性能，并且多次更新和删除会严重导致磁盘页面碎片化问题，从而降低了空间利用率。

2. 相反，异地更新结构（比如LSM树）则是将更新数据存储在新的位置，而不是覆盖原有的旧数据。这样做利用了顺序IO，从而提高了写性能。但是这样做牺牲了读性能。

### 1.2 LSM

LSM-tree被广泛应用，一个重要的原因就是因为在以前，机械硬盘应用广泛，这种硬盘他的顺序写性能远远好于其随机写性能，所以为了提升系统的一个整体的吞吐量，所以选用LSM Tree ，采用异地更新来实现快速写入。但LSM-tree存在严重的读写放大

### 1.3 SSD

1. SSD随机和顺序性能之间的差异远没有硬盘那么大,执行大量顺序I/O以减少后来的随机I/O可能会不必要地浪费带宽

2. SSD具有很大程度的内部并行性

3. SSD可能会通过重复写入而磨损，LSM tree中的高写入放大会显著降低设备寿命

4. SSD特有的写入擦除循环和昂贵的垃圾回收，过量的随机写有害

5. SSD是由许多闪存颗粒组成，又细分为多个Block，每个Block又包含多个Page。由于固有的特征，它的最小读写单位是Page、

6. 同时已经有数据的page不能直接覆盖，必须将它进行擦除操作才能写入，但是由于擦除机制的影响，是以Block为最小单位。

7. 对于随机写入，因为写入的单位是page，如果你随机写入的数据很小，也依然要占用一个page，这样就有了好几倍的写放大，降低性能的同时会损耗SSD的宝贵寿命。

8. GC过程，他会将其中的数据迁移到空白块中，之后再擦除这两个物理块。整个垃圾回收写入和读取了大约两倍Block的数据量。频繁的随机I/O会创造很多无效的Page，而随着Block中无效Page的增加，就会垃圾回收，而垃圾回收的I/O放大很严重，对SSD的性能影响也很大。

### 1.4 KV分离

1. Key 放在 LSM 树里面，Value 放在分离开的 Value log 中

2. compaction 时，只会对 Key 进行排序和重写，不影响Value log，显著降低了写放大

3. Value 分离后，LSM-Tree 本身大幅减小，可以减少查询时从磁盘读取的次数，并且可以更好的利用cache

4. Value log需要定期进行GC，开销较大

## 2 KV分离策略

### 2.1 [wisckey](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf)

1. value放在value-log 文件中

2. 不写WAL，以 <vLog-offset, value-size> 形式插入 LSM 树中

3. Compaction:只需将无效key从 LSM 树中移除，不需要修改vLog

4. 范围查询:利用内部线程池并行读取 value 数据。

5. Value log：循环队列，tail至head之间部分包含所有value

6. GC：依次读取一个键值对块，返回 LSM 中检查value有效性，将有效值写回vLog 的head

7. GC完成后将新的 value 地址和 tail 指针地址写回到 LSM-Tree 中

缺点：

1. 严格的 gc 顺序，从 head -> tail，造成很多不必要的写放大

2. GC时通过LSM-tree 判断value 有效性，查询开销不容忽视。

3. GC完成后还需要将最新的value指针更新到LSM-tree，导致大量不必要的数据重定位

4. 对于小value的范围查询性能较差

### 2.2 [HashKV](https://www.usenix.org/system/files/conference/atc18/atc18-chan.pdf)

**部分KV分离:**

1. 小value放在LSM中，大value写入value log

2. 小value对整个系统的平均写放大都影响很小

3. 改善它的一个查询性能

4. 减小GC开销

**减小写放大方法：**

1. 对key进行hash将value放在不同partition

2. 对value进行冷热分离，最后一次插入以来至少更新过一次的KV对视为热数据

**Garbage Collection**

1. 以 segment group 为单位，选择写入量最大的segment

2. 检查KV有效性：不需要查找 LSM-tree

3. 构造临时内存哈希表(按键索引)来缓冲在段组中找到的有效KV对的地址

4. Cold data log进行GC，与wisckey一致

5. 更新 LSM-tree中的地址信息

### 2.3 [TerarkDB](https://github.com/bytedance/terarkdb)

1. value 需要在前台被写入 WAL

2. 部分KV分离：大 value写入 v-SST 中

3. V-SST 按 key 排序,LSM tree仅存储<key, fileno>

**GC：**

1. GC 无需更新LSM tree，在 MANIFEST 中记录 v-SST 之间的依赖关系，降低了 GC 对用户前台流量的影响

2. compaction 过程中顺便更新value 位置信息

### 2.4  [TItan](https://github.com/tikv/titan)

1. 部分KV分离，存储形式 ：<key, <fileno, offset>>

2. Blob file中的value有序

**传统GC**

1. 使用BlobFileSizeProperties和 discardable size来收集 GC 所需信息

2. 每个BlobFile在内存中维护一个 discardable size 变量

3. 需要去LSM-tree查询value有效性，GC完成后需要写回SST

**Level Merge**

1. Compaction的同时进行GC

2. 省去了有效性检查和更新SSTable

3. 造成写放大，仅对 LSM-tree 中 Compaction 到最后两层数据对应的 BlobFile 进行 Level Merge

### 2.5 [NovKV](https://storageconference.us/2020/Papers/15.NovKV.pdf)

**设计亮点：**

1. Vstore分为Vtable和Svtable，新写入的数据写入Vtable

2. GC的时候，利用DropKeys来检查kV的有效性

3. KV被读取的时候再更新GC后的epoch，file offset，item size 

4. VTable：顺序写入

​       SVTable：便于查询

5. 在key的尾部添加一个sequence number

6. Compaction：创建一个map

​       map key  ：file number； 

​       map Value : DropKeys组成的vector

GC流程：

1. 选择valid rate最小的VTable或者SVTable进行GC

2. 从后往前读取Drop keys,并将其全部插入hash set

3. 遍历KV Iterm，与hash set做对比，将valid value写入新的SVTable（memtable）

4. 将memtable刷盘（SSTable），并在末尾添加一个invalid Dfooter

5. GC后的结果当这个KV被读取的时再更新（通过epoch判断）

### 2.6 [UniKV](https://www.cse.cuhk.edu.hk/~pclee/www/pubs/icde20.pdf)

**优化点：采用分层体系结构实现差异化数据索引**

**保证 read/write/scan/Scalability 的高效**

1. UnsortedStore：内存刷回的数据，无序SortedStore：Unsorted Store合并后的有序的数据

2. 内存 HASH 索引来支持快速的读写

3. SortedStore采用KV分离

**Hash** **Indexing** 

1. 轻量级的两级 HASH：cuckoo hashing+linked hashing

2. 索引项:<keyTag, SSTableID, pointer>

3. 写入：从h_1一直到h_n，直到找到空的bucket，否则生成一个overflow entry添加到对应的桶 h_n (key)%N 后

读取：从h_n一直到h_1

**KV分离**

1. 元信息：<partition,logNumber,offset,length>

2. 动态范围分区:根据Key大小进行拆分，改善范围查询

3. GC：需要遍历Sortedstore，判断有效性。之后还需要将元信息写回sortedstore、

## 3. 总结

**KV分离改进措施：**

1. 部分KV分离，在不影响系统整体写放大的情况下，减少GC开销，提高范围查询性能（HashKV）

2. 对key进行hash、或按key的大小将value放在不同partition，减小写放大（HashKV，UniKV）

3. 冷热数据分离，减小冷数据GC造成的写放大（HashKV）

4. 通过 MANIFEST 记录GC前后文件之间的依赖关系，推迟至compaction 过程中更新value 位置信息（TerarkDB）

5. 在最后1~2 level，Compaction的同时进行GC，省去了有效性检查和更新SSTable（Titan）

6. Compaction时将Dropkey写入VStore，GC时利用DropKeys来检查kV的有效性（NovKV）

7. KV被读取时再更新SSTable（NovKV）

