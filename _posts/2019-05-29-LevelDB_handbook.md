## 基本概念

levelDB是典型的LSM（Log-Structed Merged-Tree)树实现，LSM树的核心就是放弃部分读的性能，换取最大的写入性能

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
leveldb的一次写入操作并不是将数据直接刷到磁盘文件，而是首先写到内存，memtable就是一个在内存中进行数据组织和维护的结构。
在memtable中，所有数据按照用户定义的排序方法排序后按序存储，等到容量达到阈值时（默认4MB），将其转换成一个不可更改的memtable,
与此同时创建一个新的memtable, 供用户继续读写，memtable底层采用跳跃表结构

#### immutable memtable
和memtable结构定义完全一样，区别是immutable memtable是只读的，当一个immutable memtable被创建时，
leveldb后台压缩进程会利用其中的内容，创建一个sstable，持久化到磁盘中

#### log
leveldb的写操作是先先内存。为了造成用户写入发生丢失，leveldb在写内存之前，会将所有的写操作写到日志文件，log文件，
当以下异常情况发生时，均可以通过日志文件恢复：
1. 写log期间进程异常
2. 写log完成，写内存未完成
3. write动作完成后，进程异常
4. immutable memtable持久化过程中进程异常
5. 其他压缩异常

每次日志的写操作都是一次顺序写，因此效率高

#### sstable
虽然先写内存，但是内存中数据不可无限增长，而且日志中记录的写入操作越多，异常恢复的时间就越长，
因此内存中数据到达一定容量时，就需要持久化到磁盘中。leveldb的数据主要是通过sstable进行存储

虽然在内存中，数据是按序排列的，但是多个memtable持久化到磁盘后，对应的不同sstable之间是会存在交集的，
因此在读操作时，需要对所有的sstable进行遍历，影响读效率，因此，leveldb后台进程会定期整理这些sstable文件，
这个过程称为compaction。随着compaction的进行，sstable在逻辑上被分成若干层，由内存数据直接dump出来的文件称为
level0层，后期整合的称为level i层

#### manifest
leveldb中有个版本的概念，一个版本中主要记录了每一层中所有文件的元数据，包括
1. 文件大小
2. 最大key值
3. 最小key值

以goleveldb为例，一个文件元数据包含最大最小key，文件大小信息
```go
type tFile struct {
    fd storage.FileDesc
    seekLeft int32
    size int64
    imin, imax internalKey
}
```
一个版本信息主要维护了每一层所有文件的元数据
```go
type version struct {
    s *session
    levels []tFiles
    
    cLevel int
    cScore float64
    
    cSeek unsafe.Pointer
    
    closing bool
    ref int
    released bool
}
```
每次compaction完成后，leveldb都会创建一个新的version，创建规则：
`versionNew = versionOld + versionEdit`。
其中versionEdit指代的是基于旧版本的基础上变化的内容
 
 而manifest文件就是用来记录这些versionEdit信息的，一个versionEdit数据，会被编码成一条记录，写入manifest文件。
 
 每条记录包括：
 1. 新增的sst文件
 2. 删除的sst文件
 3. 当前compaction的下标
 4. 日志文件编号
 5. 操作seqNumber
 
 #### current
 记录当前的manifest文件名。
 每次leveldb启动时，都会创建一个新的manifest文件
 
 
 ## 读写操作
 
 ### 写操作
 整体流程如下：
 ![avatar](/assets/images/leveldb_写操作整体流程.jpeg)
 
 leveldb的一次写入分为2部分
 1. 将写操作写入日志, 保障用户的写入不会丢失
 2. 将写操作应用到内存数据库中
 
 #### 写类型
 leveldb对外提供的写入接口有：1. Put, 2. Delete。
 本质对应同一种操作，Delete会被转换成一个value为空的Put
 
 #### batch结构
 批量的原子的完成数据库更新操作
 无论是Put，Delete还是批量操作，底层都会为这些操作创建一个batch实例作为一个数据库操作的最小执行单元，batch结构如下：
 ![avatar](/assets/images/leveldb_batch结构.jpeg)
 
 在batch中，每一条数据集都按照上图格式编码，编码后的第一位是操作类型(更新or删除),然后是key的长度，key的内容，value的长度，value的内容。
 
 batch中会维护一个size值，用于表示其包含的数据量的大小。为所有数据项key与value长度的累加，以及每条数据项额外的8个字节，这8个字节用于存储一条数据项额外的信息。
 
 #### key值编码
 数据项从batch中写入到内存数据库中时，都需要一个key的转换，所有的数据项的key都是进过特殊编码的，这种格式称为internalKey.
 
 internalKey在用户key基础上，尾部追加8字节，用于存储：1.操作对应的sequence number(7字节), 2.操作类型(1字节)
 
 其中，每一个操作都被赋予一个sequence number, leveldb每执行一次操作就会给这个number累加一次。
 又由于leveldb中，一个更新或者删除，并非直接更新元数据，而是采取append的方式。
 因此，对应一个key，会有多个版本的数据记录，而最大sequence number的记录就是最新的
 
 此外，leveldb的snapshot也是基于这个sequence number实现，即每一个sequence number代表着数据库的一个版本
 
 #### 合并写
 leveldb面对并发写入时，leveldb在同一个时刻，只允许一个写入操作将内容写入到日志以及内存数据库中，而在写入进程较多时，
 为了减少日志文件的小写入，leveldb会将一些小写入合并成一个大写入。流程如下：
 1. 第一个获取到写锁的写操作
    + 如果当前写操作的数据量未超过合并上限，而且有其他写操作被pending，将其他写操作的内容合并到自身
    + 如果当前写操作数据流量超过上限，或者无其他pending的写操作，将所有内容统一写入日志文件，并写入到内存数据库中
    + 通知每一个被合并的写操作最终的写入结果，释放或者移交写锁
    
2. 其他写操作
    + 等待获取写锁或者被合并
    + 如果被合并了，判断是否合并成功，若成功，等待最终写入结果；否则，直接从上个占有锁的写操作中接过写锁进行写入
    + 若未被合并，继续等待
 
#### 原子性
leveldb的任意一个写操作，无论包含了多少次写，其原子性都是由日志文件实现的。
一个写操作中的所有内容会以一个日志中的一条记录作为最小单位写入，以此来保证原子性


### 读操作
leveldb提供2种进行读取数据的接口：
1. Get
2. 先创建一个snapshot, 基于该snapshot调用Get

2种方式都是基于快照进行读取的

#### 快照
leveldb中，用于一个整型数来代表一个数据库的状态。
用户对同一个key的若干次修改，是以维护多条数据项的方式进行存储的，直到compaction时才会合并成一条记录，每条数据项都会赋予一个序列号，序列号越大，数据越新。

因此，每一个序列号，其实就代表着leveldb的一个状态。

当用户主动或者被动创建一个快照时，leveldb会以当前最新的序列号对其赋值。

在获取到一个快照之后，leveldb会为本次查询的key构建一个internalKey，其中的seq字段就是快照对应的seq，通过这种方式过滤掉所有seq大于快照号的数据项。

#### 读取
 ![avatar](/assets/images/leveldb_读取流程.jpeg)
 
leveldb读取分为3步：
1. 在memory db中查找指定的key，若找到，结束查找
2. 在immutable memory db中查找，若找到，结束查找
3. 按底层(0层)至高层的顺序在level i层的sstable文件中查找指定的key, 若找不到，返回不存在指定数据
 
在每一层sstable中查找时，都是按序依次查找sstable的；
0层文件比较特殊，key可能重合，因此在0层中，文件编号大的sstable优先查找，因为编号较大的总是最新的数据。
非0层，一层中的key不会重合，可以借助sstable元数据的最大最小key，可以快速定位，每一层只需要查找一个sstable文件的内容
 
 
## 日志
防止在写入内存数据库时因为进程异常等情况发生丢失，在写内存前会将写操作内容写入日志文件

![avatar](/assets/images/leveldb_写日志流程.jpeg)

### 日志结构

![avatar](/assets/images/leveldb_日志结构.jpeg)

在leveldb中，有2个memory db，以及对应的2份日志文件，其中一个memory db是可读写的，当这个db的数据量超过预定上限时，转换成一个不可写的memory db, 
同时，与之对应的日志文件变成一份frozen log。新生成的immutable memory db由后台的minor compaction进程将其转换成一个sstable文件进行持久化，
持久化完成后，对应的frozen log被删除。

为了增加读取效率，日志按照block划分，每个block大小32kb，每个block包含若干个完整的chunk。
一条日志记录包含一个或多个chunk，每个chunk包含：
1. 7字节大小的header，前4字节是校验码，接着2字节的长度，接着1字节的chunk类型
2. data数据

其中chunk类型：full、first、middle、last。

### 日志内容
日志的内容(即上面结构中的data数据)为**写入的batch编码后的信息**，格式如下：

![avatar](/assets/images/leveldb_日志内容格式.jpeg)

一条日志记录包含：
1. Header
    + 当前db的sequence number
    + 本次日志记录中包含的put/del操作的个数
2. batch data

### 日志写
![avatar](/assets/images/leveldb_日志写流程.jpeg)

在leveldb内部，实现了一个journal的writer，首先调用Next函数获取一个singleWriter，这个singleWriter的作用就是写一条journal记录。

singleWriter开始写入时，标志着第一个chunk开始写入，在写入过程中，不断判断writer中buffer的大小，如果超过32KB，将chunk开始到现在做为一个完整的chunk，
并为其计算header之后，将整个block写入文件，充值buffer，开始新的chunk写入。

如果一条journal较大，可能会分成几个chunk存储在若干个block中。

### 日志读
![avatar](/assets/images/leveldb_日志读流程.jpeg)

按block(32KB)读取, 避免频繁IO读取。

每次读取一条日志，reader调用Next返回一个singleReader, 每次调用Read函数就返回一个chunk的数据，每次读取一个chunk，检查校验码、数据类型、长度是否正确，
若错误，丢弃整个chunk数据。循环直到堵到一个last的chunk，表示整条日志都已经读取完成，返回。

## 内存数据库
leveldb的内存数据库用来维护**有序**的key-value对。底层采用跳跃表实现。

### 跳跃表

#### 概述

绝大多数读写操作时间复杂的O(logN),有着与平衡树媲美的操作效率，但是实现起来简单得多

平衡树(红黑树为代表)是一种非常复杂的数据结构，为了维持平衡，每次插入都可能会涉及到复杂的节点旋转等操作。

### 内存数据库
介绍leveldb是如何利用跳表来构建一个高效内存数据库。

#### 键值编码
内存数据库中，key称为internalKey, 由3部分组成：
1. 用户定义的key
2. 序列号
3. 类型，更新or删除

#### 键值比较
系统默认比较：字典序比较用户定义的key；按照用户key升序排列，若用户key一致，按照序列号降序排列。
也可以用户自定义

#### 数组组织
以goleveldb为例，内存数据库定义如下：
```go
type DB struct {
    cmp comparer.BasicComparer
    rnd *rand.Rand
    
    mu sync.RWMutex
    kvData []byte
    // Node data:
    // [0]: KV offset
    // [1]: Key length
    // [2]: Value length
    // [3]: Height
    // [4...height]: Next nodes
    nodedData []int
    prevNode [tMaxHeight]int
    maxHeight int
    n int
    kvSize int
}
```
kvData用来存储每一条数据项的key-value数据；
nodeData用来存储每个跳跃表节点的链接信息。



## sstable

### 概述
当leveldb达到checkpoint点(memtable中数据量超过了阈值)，会将当前memtable冻结成一个immutable memtable, 并且创建一个新的memtable供系统继续使用

immutable memtable会在后台进行一次minor compaction, 持久化到磁盘中

leveldb设计Minor Compaction的目的是为了：
1. 有效降低内存使用率
2. 避免日志文件过大，异常恢复时间过长

当memory db的数据被持久化到文件中时，leveldb以一定的规则进行文件组织，这种文件格式称为sstable

### SStable文件格式

#### 物理结构
为了提高读写效率，一个sstable文件按照固定大小进行块划分，默认每个块大小4KB，每个块中，除了存储数据，还存储2个额外的辅助字段
1. 压缩类型：leveldb默认使用Snappy压缩
2. CRC校验码

#### 逻辑结构
leveldb在逻辑上根据功能的不同，将sstable划分为：
1. data block: 用来存储key-value对, n个
2. filter block：用来存储一些过滤器相关的数据(bloom filter)，若是用户不指定使用过滤器，该block不会存储任何内容
3. meta index block: 用来存储filter block的索引信息（即filter block在该sstable文件的偏移量以及数据长度)
4. index block: 存储每个data block的索引信息
5. footer: 用来存储meta index block以及index block的索引信息

### data block结构
data block存储的是key-value键值对，其中一个data block中的数据部分按逻辑进行如下划分：

![avatar](/assets/images/leveldb_datablock结构.jpeg)

第一部分用来存储key-value数据，由于sstable中所有key-value都是严格按序存储，为了节省存储空间，并不会为每一对key-value键值对存储完整的key信息，
**而是存储与上一个key非共享的部分**，避免key重复内容的存储

每间隔若干个key-value对（默认16个），将为该条记录重新存储一个完整的key，重复该过程，每个重新存储完整key的点称为restart point

```
设计restart point的目的是在读取sstable时，加速查找的过程。
由于每个restart point存储的都是完整的key值，因此可以先利用restart point进行键值比较，快速定位目标数据所在区域；
当确定所在区域时，再依次对区间内数据逐项比较key
```

每个entry数据项的个数如下：
+ 与前一条记录key共享部分的长度
+ 与前一条记录key不共享部分的长度
+ value的长度
+ 与前一条记录key不共享的内容
+ value内容

一个data block数据段的末尾会添加每一个restart point的值（偏移量），以及所有restart point的个数

### filter block结构
为了加快sstable中数据查询的效率，在直接查询data block中的内容之前，
先根据filter block中的过滤器判断指定的data block中是否有需要查询的数据，如果不存在，则无需对这个data block进行查找

filter block存储的是data block数据的一些过滤信息，这些过滤信息一般指代布隆过滤器的数据。

![avatar](/assets/images/leveldb_filterblock结构.jpeg)

这个过滤数据一般指代布隆过滤器的数据，主要分为2部分：
1. 过滤数据
2. 索引数据

`filter i offset`表示第i个filter data在整个filter block中的偏移量；
`filter offset's offset`表示filter block索引数据在filter block中的偏移量

在读取filter block中内容时，可以先读取`filter offset's offset`，然后依次读取`filter i offset`, 根据这些offset可以分别读出`filter data`

Base Lg(1 byte)：默认为11，表示每2kb的数据，创建一个新的过滤器来存放过滤数据。

一个sstable只有一个filter block，存储了所有block的filter数据。
具体来说， `filter data k`包含了所有起始位置处于`[base*k, base*(k+1)]`范围内的block的key的集合的filter数据。

### meta index block结构
用来存储filter block在整个sstable中的索引，
只存储一条记录：
key: "filter" + 过滤器名字组成的常量字符串
value: filter block在sstable中索引信息序列化后的内容：1.在sstable中的偏移量 2.长度

### index block结构
用于存储所有data block的索引信息。每一条索引信息代表一个data block的索引信息，包括：
1. `data block i`的最大key值
2. `data block i`起始地址在sstable中的偏移量
3. `data block i`的大小

其中，`data block i`的最大key值还是index block中该条记录的key值，可以依次比较index block记录的key值来快速定位在哪个data block中。

### footer结构
固定48字节，存储meta index block和index block在sstable中的索引信息。
尾部还会存储一个magic word, 内容为"http://code.google.com/p/leveldb/" sha1哈希后的前8个字节


### 读写操作

#### 写操作
sstable的写操作通常发生在：
+ memory db将内容持久化到磁盘中时，会创建一个sstable进行写入 minor compaction
+ leveldb后台进行文件compaction时，会将若干个sstable文件的内容重新组织，输出到若干个新的sstable中 major compaction

```go
type tWriter struct {
    t *tPos
    
    fd storage.FileDesc // 文件描述符
    w storage.Writer    // 文件系统Writer
    tw *table.Writer
    
    first, last []byte
}
```
一次sstable写入为：一次不断利用迭代器读取需要写入的数据，不断调用tableWriter的`Append`函数，直到所有有效数据读取完毕，
然后为sstable文件附上元数据

其中sstable文件的元数据包括：
1. 文件编码
2. 文件大小
3. 最大key值
4. 最小key值

Append函数是理解整个sstable写入过程的关键

```go
type Writer struct {
    writer io.Writer
    blockSize int // 默认4KB
    
    dataBlock blockWriter       // data块writer
    indexBlock blockWriter      // indexBlock块writer
    filterBlock filterWriter    // filter块writer
    pendingBH blockHandle
    offset uint64
    nEntries int // key-value键值对个数
}
```

##### Append
一次Append函数的主要逻辑如下：
1. 若本次写入为新dataBlock的第一次写入，则将上一个dataBlock的索引信息(pendingBH)写入
2. 将key-value数据写入data block
3. 将过滤信息写入filter block
4. 若data block中的数据超过预定上限(默认4kb)，标志本次写入结束，将内容刷新到磁盘文件中

若一个data block中的数据超过了固定上限，需要将相关数据写入到磁盘：
1. 封装dataBlock, 记录restart point的个数
2. 若dataBlock需要压缩，进行压缩
3. 计算checksum
4. 封装dataBlock索引信息（偏移量、长度）
5. 将dataBlock的buffer中的数据写入磁盘
6. 利用这段时间里维护的过滤信息生成过滤数据，放入filterBlock对应的buffer中

当迭代器取出所有数据并完成写入后，调用tableWriter的Close函数完成最后的收尾工作：
1. 若buffer中仍有未写入的数据，封装成一个dataBlock写入
2. 将filterBlock内容写入磁盘文件
3. 将filterBlock的索引信息写入metaIndexBlock中，写入磁盘文件
4. 写入indexBlock数据
5. 写入footer数据

至此，所有的数据已经被写入一个sstable中了，由于一个sstable是作为一个memory db或者compaction的结果原子性落地的，
因此在写入后，还会进行更复杂的版本更新


#### 读操作

![avatar](/assets/images/leveldb_sstable读操作流程.jpeg)

大致流程为：
1. 判断"文件句柄"cache中是否有指定sstable文件的句柄，若存在，直接使用cache中的句柄；否则，打开sstable文件，按规则读取该文件的元数据，将句柄存储到cache中
2. 利用index block快速定位数据项，得到该数据项有可能存在的**2个**data block
3. 利用index block中的索引信息，首先打开第一个可能的data block
4. 利用filter block中的过滤信息，判断指定的数据项是否在该data block中，若存在，创建一个迭代器对data block中的数据遍历；
若不存在，结束在这个data block中的查找
5. 打开第二个可能的block data, 继续步骤4
6. 若5中还未找到，返回`Not Found`错误

##### 缓存
leveldb中，使用cache来缓存2类数据
1. sstable文件句柄及其元数据
2. data block中的数据

在打开文件前，首先判断能否在cache中命中sstable的文件句柄，避免重复读取的开销。

##### 元数据读取
在打开sstable文件后，需要读取必要的元数据，才能访问sstable中的数据。
1. 读取文件的最后48字节，即footer数据
2. 读取footer维护的meta index block, index block两个部分的索引信息
3. 利用meta index block索引信息读取该部分内容
4. 遍历meta index block，查看是否存在有用的filter block索引信息

##### 数据项快速定位
sstable中存在多个data block, 依次遍历是不可取的，可以利用有序性已经在index block中维护的索引信息快速定位目标数据项可能存在的data block

index block每一项都是一个键值对，每个键值对表示一个data block的索引信息，**key为该data block中数据项最大的key**。
**value为该data block的索引信息(偏移量、长度)**

因此，仅需依次比较index block中这些索引即可。

```
和data block一样，index block中的索引信息也采用了key值截取，即第二个索引信息的key并不是存储完整的key，而是与前一个key不共享的部分。
区别在于data block的划分粒度默认是16，而index block是2；
因此，index block中连续2条索引信息会被认为是一个最小的比较单元，这就会导致最终目标数据项的范围区间是某**2个**data block
```

##### 过滤data block
若sstable存有每一个data block的过滤数据，可以利用这些过滤数据对data block数据进行判定，确定目标数据是否不存在与data block中

##### 查找data block
data block中所有数据项都是按序排序的，data block以16条记录为一个查找单元，若entry1的key小于目标key，则下一个比较的就是entry17。这样极大的提升了整体的查询效率


### 文件特点

#### 只读性
sstable为compaction结果原子性产生，在其余时间都是只读的

#### 完整性
一个sstable, 其辅助数据
+ 索引数据
+ 过滤数据
都直接位于同一个文件中，当需要辅助数据时，无需额外的磁盘读取。需要删除时，无需额外的删除辅助数据

#### 并发访问友好性
由于sstable文件只读。不存在读写冲突。
leveldb采用引用计数维护每个文件的引用情况，当一个文件计数值大于0时，对文件的删除动作为等到文件被释放时才进行。

## 缓存系统
leveldb使用了一种基于LRU的缓存机制，用于缓存：
+ 已打开的sstable文件句柄和其元数据
+ sstable中的data block内容

### Cache结构
由2部分组成：
+ Hash table: 用来存储数据，基于Dynamic-Sized Nonblocking Hash Table实现，rehash过程中不阻塞其他并发的读写请求。
+ LRU：维护数据项的新旧信息：

![avatar](/assets/images/leveldb_cache结构.jpeg)

#### Dynamic-Sized Nonblocking Has Table
一个hash表通常由若干个bucket组成，每一个bucket会存储若干条被散列至此的数据项，当hash表进行resize时，需要从旧bucket中的数据独处，重写散列至一个新的bucket里。

由于一个哈希bucket的容量是有限的（一般不大于32个），因此在哈希bucket中进行插入、查找的时间都是常量级别的

当cache中维护的数据量太大时，会发生哈希表扩张，以下两种情况：
+ 整个cache中，数据项的个数超过预定的阈值（默认bucket 16个，每个bucket默认32个，16*32）
+ cache中出现了数据不平衡，某些bucket的数据量超过了32个，当这种散列不平衡累计超过128个时，就需要扩张

一次扩张过程如下：
1. 计算新哈希表的bucket个数(扩大一倍)
2. 创建一个新的空哈希表，将旧的哈希表转换成一个"过渡期"的哈希表，每个bucket都被冻结
3. 后台利用被冻结的哈希bucket信息对新的哈希表进行构建

整个构建过程中，哈希表继续提供读写功能。


#### LRU
leveldb中，LRU使用一个双向循环链表实现
```go
type lruNode struct {
    n *Node
    h *Handle
    ban bool
    next, prev *lruNode
}
```

#### 缓存数据
leveldb利用上述的cache结构来缓存数据，其中：
1. cache: 来缓存已经被打开的sstable文件句柄以及元数据(默认上限500）
2. bcache: 来缓存被读过的sstable中的dataBlock数据(默认上限8MB)

## 布隆过滤器
略

## compaction
是leveldb最复杂的过程之一，**也是leveldb的性能瓶颈之一**。本质是一种内部数据重新整合机制，同样也是一种平衡读写速率的有效手段

### Compaction的作用

#### 数据持久化
一次内存数据的持久化过程，在leveldb中称为"Minor Compaction",
一次minor compaction的产出是一个0层的sstable文件，包含了所有的内存数据，是可能存在overlap的

#### 提高读写效率
写效率很高，只需要一次顺序的文件写，和一个事件复杂度O(logN)的内存操作(有序的插入memtable的跳跃表)

而读操作就较为复杂，首先要1~2次的复杂度O(logN)的查询，若在内存没有命中，按照数据新旧程度在0层文件一次查找遍历，
由于0层文件存在overlap，因此最坏可能要遍历所有文件

随着运行时间增长，0层文件数量越来越多，这样查询一个数据的时间也越来越长。因此，leveldb设计了一个"Major Compaction"的过程，
将0层中的文件合并为若干个没有数据重叠的1层文件。由于没有重叠，因此一次查找过程就可以进行优化。

#### 平衡读写差异
一次major compaction的过程涉及到大量的磁盘读写开销，是一个严重的性能瓶颈。
当用户写入速度始终大于major compaction速度时，将导致0层文件不断增加，读取效率不断下降。因此，leveldb规定：
+ 当0层文件超过`SlowdownTrigger`时，写入的速度减慢
+ 当0层文件超过`PauseTrigger`时，写入暂停，直至Major Compaction完成

#### 整理数据
leveldb每一条数据项都有一个版本信息，major compaction过程会对不同版本的数据项合并

### Compaction过程
compaction分2类：
+ minor compaction
+ major compaction

#### Minor Compaction
将一个内存数据库中数据持久化到一个磁盘文件中。
每次minor compaction之后，都会生成一个新的sstable文件，这也意味着leveldb的版本状态发生了变化，会进行一个版本的更替。

minor compaction的时效性要求很高，要求尽可能快的完成，否则会阻塞正常的写入。因此优先级高于major compaction


#### Major Compaction
Major compaction的过程比较复杂。

![avatar](/assets/images/leveldb_major_compaction过程.jpeg)

##### 条件
触发leveldb进行major compaction的条件：
+ 当0层文件超过预定的上限（默认4个）
+ 当level i层的文件总大小超过(10^i)MB, 降低compaction的IO开销
//todo @019-06-25
+ 当某个文件无效读取的次数过多：以上2个机制可以保证随着合并的进行，数据是严格下沉的，但是仍然存在问题：
假设0层文件合并完成后，1层文件同时达到上限，也需要合并，最坏可能0~n层都需要合并。

其中一种优化机制是：
+ source层的文件个数只有1个
+ source层文件与source+1层文件没有重叠
+ source层文件与source+2层的文件重叠部分不超过10个文件
满足这几个条件时，可以将source层文件直接移到source+1层。

为了避免存在的巨大合并开销，leveldb还引入了第三个机制：错峰合并
+ 一个文件的一次查询开销为10ms，若某个文件查询次数过多，且查询在该文件中不命中，那么就可以视为无效的查询开销，这种文件就可以错峰合并
+ 对于1个1mb的文件，对其合并的开销为25ms，因此当一个1MB的文件无效查询超过25次，便可以对其合并


##### 采样探测
每个sstable文件元数据中，还有一个额外字段seekLeft, 默认为文件的大小/16KB。
leveldb在正常的数据访问时，会顺带进行采样探测，记录本次访问的第一个sstable文件，若命中，不做任何处理，若不命中，对seekLeft减一

直到一个文件的seekLeft减少为0，会触发对该文件的错峰合并

##### 过程
整个compaction可以简单的分为以下几步：
1. 寻找合适的输入文件
2. 根据key重叠情况扩大输入文件集合
3. 多路合并
4. 积分计算


## 版本控制
leveldb每次生成sstable文件，或者删除sstable文件，都会从一个版本升级成另一个版本。

### Manifest
manifest文件专用于记录版本信息，采用了增量式的存储方式，记录每一个版本相较于上一个版本的变化

一个manifest文件中，包含多条Session Record, 一个Session Record记录了上一个版本至该版本的变化情况，主要包括：
1. 新增了哪些sstable文件
2. 删除了哪些sstable文件（由于compaction导致)
3. 最新的日志文件标号等

借助这个manifest文件，leveldb启动时，可以根据一个初始的版本状态，不断的应用这些版本改动，恢复到最近一次使用的状态、

一个Manifest文件格式如下：
![avatar](/assets/images/leveldb_manifest文件格式.jpeg)

一个Manifest文件中包含若干条Session Record, 其中第一条Session Record记载了当时leveldb的全量版本信息。其余Session Record记录每次更迭的变化情况

一个Session Record可能包含以下字段：
+ Comparer名称
+ 最新的日志文件编号
+ 下一个可用的文件编号
+ 数据库已经持久化数据项的最大sequence number
+ 新增的文件信息
+ 删除的文件信息
+ compaction记录信息

### Commit
一次版本升级过程如下：
![avatar](/assets/images/leveldb_版本升级过程.jpeg)

1. 新建一个session record, 记录状态变更信息
2. 如果本次版本更新是由于minor compaction或者日志replay导致生成了一个sstable文件，
则在session record中记录新增的文件信息、最新的日志编号、数据库sequence number以及下一个可用的文件编号
3. 如果本次版本更新是由于major compaction, 则在session record中记录新增、删除的文件信息，以及下一个可用的文件编号
4. 利用当前版本信息和session record信息，创建一个全新版本信息，新的版本信息更改的内容为：1)每一层的文件信息 2）每一层积分信息
5. 将session record持久化
6. 若这是数据库启动后第一条session record, 则新建一个manifest文件，将完整的版本信息全局记录进session record， 同时更改current文件，
将其指向新建的manifest
7. 若数据库中已经创建了manifest文件，则将该条session record进行序列化后直接作为一条记录写入即可
8. 将当前的版本设置为刚创建的版本

### Recover
数据库每次启动时，都会有一个recover过程，就是利用manifest信息重新构建一个最新的version

### Current
每次启动，都会新建一个manifest文件，leveldb中会存在多个manifest文件，因此需要一个current文件来指示当前系统使用的是哪个manifest文件

### 异常处理
manifest文件丢失, leveldb提供`Recover`接口进行版本信息恢复，

### 多版本并发控制
1. sstable文件时只读的，每次compaction都只是对若干个sstable进行多路合并后创建新的文件，故不会影响在某个sstable文件读操作的正确性
2. sstable都是具有版本信息的，即每次compaction完成之后，都会生新版本的sstable， 因此可以保障读写操作都可以针对于相应版本文件进行，解决了读写冲突
3. compaction生成的文件只有等合并完成之后才会写入数据库元数据，不会污染用户正常的读操作
4. 采用引用计数来控制删除行为，当compaction完成后试图去删除某个sstable文件时，会根据该文件的引用计数来适当的延迟删除


