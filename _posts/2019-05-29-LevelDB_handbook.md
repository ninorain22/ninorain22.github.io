## 基本概念

levelDB是典型的LSM树实现，LSM树的核心就是放弃部分读的性能，换取最大的写入性能

简单点说，就是尽量减少随机写的次数，对于每次写入操作，不是直接将最新的数据写到磁盘，而是将其拆分成
1. 一次日志文件的顺序写
2. 一次内存中的数据插入
首先将数据更新在内存中，当内存数据达到一定的阈值时，将这部分数据刷到磁盘文件，因此获取极高的写性能

### 整体架构
![avatar](/assets/images/leveldb_整体架构.jpeg)

leveldb主要由以下几个重要的部件构成:
+ memtable
+ immutable memtable
+ log
+ sstable
+ manifest
+ current

#### memtable
@2019-05-29