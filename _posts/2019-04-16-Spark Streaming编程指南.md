## Spark Streaming编程指南

### 概述
Spark Streaming是Spark Core Api的扩展，支持弹性的，高吞吐的，容错的实时数据流处理。
数据可以从多种数据源获取，比如Kafka, Flume, 以及TCP sockets等，也可以通过`map`, `reduce`, `join`, `window`等高级函数组成的复杂算法处理。
最终处理后的数据可以输出到文件系统，数据库或者实时仪表盘中。

Spark Streaming提供了一个叫`DStream`的高级抽象，表示一个连续的数据流。**在内部，一个DStream是用一系列的RDDs来表示的**

### 入门示例
```scala
import org.apache.spark._
import org.apache.spark.streaming._

// 创建一个具有2个`工作线程`，并且批次间隔(batch interval)为1s的本地Streaming Context
val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")
val ssc = new StreamingContext(conf, Seconds(1))


// 根据这个context, 可以创建代表从TCP来的数据流的一个DStream
val lines = ssc.socketTextStream("localhost", 9999)
// 这个离散数据流的每一条记录都是一行文本,我们将其拆分成单词
val words = lines.flatMap(_.split(" "))
// 计算每一个batch中每一个单词的数量
val pairs = words.map((_, 1))
val wordCounts = pairs.reduceByKey(_+_)
// 打印数据流
wordCounts.print(10)

// 开始计算
ssc.start()
// 等待计算被中断
ssc.awaitTermination()

```

### 基础概念

#### 依赖
sbt中添加依赖
`libraryDependencies += "org.apache.spark" % "spark-streaming_2.11" % 2.4.0`

针对不同的数据源，比如kafka、Flume等，需要添加对应的依赖
```scala
libraryDependencies += "org.apache.spark" % "spark-streaming-kafka_2.11" % 2.4.0
libraryDependencies += "org.apache.spark" % "spark-streaming-flume_2.11" % 2.4.0
```

#### 初始化Streaming Context
初始化一个Spark Streaming程序，必须先创建`StreamingContext`对象，它是所有Spark Streaming功能的主入口点
```scala
import org.apache.spark._
import org.apache.spark.streaming._

/**
* master: Spark, Mesos或者YARN集群url, 或者是local[*]的本地模式
*/
val conf = new SparkConf().setAppName("appName").setMaster("master")
// 这里在内部其实也创建了一个SparkContext, 因为它是所有Spark功能的出发点，可以通过`ssc.sparkContext`访问
// batch interval的设置需要根据应用需要而设置，见`性能调优指南`章节
val ssc = new StreamingContext(conf, Seconds(1))

// 也可以从一个现有的SparkContext对象来创建
val sc = new SparkContext(conf)
val ssc2 = new StreamingContext(sc, Seconds(1))
```
在定义了一个StreamingContext之后，必须执行以下操作:
1. 通过创建输入DStreams来定义输入源
2. 通过应用转换和输出操作DStreams定义计算流
3. 开始接收输入并使用`streamingContext.start()`来开始处理数据
4. 使用`awaitTermination()`等待处理被终止
5. 使用`stop()`来手动停止处理

记住重要的几点:
+ 一个context启动后，就不会有新的数据流的计算可以被添加到其上
+ 一个context已经停止，不会被重新启动
+ 同一时间内在JVM中只有一个StreamingContext可以被激活
+ 在StreamingContext上的stop()也同样停止了SparkContext。可以设置stop(stopSparkContext=false)来仅仅停止StreamingContext
+ 这样，SparkContext就可以重复利用

#### DStream
DStream的每一个RDD都包含了一个特定interval的数据
**在内部，一个DStream被表示为一系列连续的RDDs，因此，应用于DStream的任何操作在底层，都会转换为对RDD的操作**

#### Input DStream和Receiver
输入DStream是代表输入数据是从流的源数据接收到的流的DStream，每一个Input DStream与一个Receiver相关联(file stream除外)，
从数据源获取数据，并且存储到Spark内存中用于后续处理

Spark提供了两种内置的streaming source
+ 基础数据源: StreamingContext API中可以直接使用的资源，比如文件，socket等
+ 高级数据源: 比如Kafka、Flume等，通过额外的utility class来使用

如果想要在Streaming应用中并行接收多个数据流，可以创建多个input DStreams, 这样就会创建多个Receivers来同时接收多个数据流。
但是，一个Spark的worker/executor是一个长期运行的任务(task)，
因此它将占用分配给Spark Streaming应用的所有核中的一个核，因此，一个Spark Streaming应用程序需要分配足够的核来处理所接收的数据，以及来运行接收器

注意几点:
+ 在本地运行一个Spark Streaming时，不要用"local"或者"local[1]"作为master参数的值。
这意味着只有1个线程用于运行本地任务，如果正在使用1个基于接收器的input DStream(例如sockets, Kafka, Flume等)，那么这个单独的线程，将用于运行接收器。
这样，就没有任何线程来处理接收到的数据。
+ 将逻辑扩展到集群上，分配给Spark Streaming的内核数必须大于接收器的数量

#### 基础数据源

##### File Stream
从文件中读取数据吗，在任何与HDFS API兼容的文件系统中，可以这样创建一个DStream
简单文件:`steamingContext.fileStream[KeyClass, ValueClass, InputFormatClass](dataDirectory)`
普通文件:`streamingContext.textFileStream(dataDirectory)`

###### 目录是如何被监控的
Spark Streaming将监控`dataDirectory`目录中任何新建的文件（不支持嵌套)
+ 简单的目录可以被监控，比如`hdfs://namenode:8040/logs/`, 任何在目录下新创建的文件都将会被发现
+ 可以使用POSIX的glob模式，比如`hdfs://namenode:8040/logs/2017/*`, 此时，DStream将会包含所有匹配这个模式的文件
+ 所有的文件格式必须相同
+ 基于文件的修改时间，而不是创建时间，来讲其作为interval的一部分
+ 一旦文件被创建，文件内部的修改将被忽略
+ 目录下的文件越多，发现改变的时间就将越长
+ 当使用模糊匹配目录时，将一个目录命名修改为匹配这个模式时，这个目录将会被加入监控。
+ 可以通过调用`FileSystem.setTimes()`来修改时间戳来讲这些文件放入下一个window, 即使文件的内容并没有任何改变

##### Using Object Stores as a source of data

##### Streams based on Custom Receivers
DStreams可以使用自定义的`receiver`接收到的数据来创建。略

##### RDDs队列
用于测试Spark Streaming，可以使用`streamingContext.queueStream(queueOfRDDs)`来创建一个基于RDD的队列的DStream，
每个进入队列的RDD都被视为DStream中的一个批次数据

##### 高级数据源
Kafka, Flume等，必须通过依赖单独的类库实现创建DStream的功能。

#### DStream上的转换
与RDD类似，DStreams支持很多在RDD中可用的transformation算子。
其中，讨论下部分转换:

##### updateStateByKey操作
`updateStateByKey`允许维护任意状态，同时不断更新信息，可用通过两步来使用
1. 定义state-state，可用是任何的数据类型
2. 定义state update function(状态更新函数), 使用函数指定如何使用先前的状态来和输入流中的新的数据更新到当前的状态

在每个batch中，Spark会使用状态更新函数为所有的key更新状态, 无论在这个batch中是否有新的数据。
比如说，想要保持在文本数据流中看到的每个单词的计数，那么可以如下定义状态更新函数
```scala
val updateFunction = (newValues: Seq[Int], state: Option[Int]) => {
  val newCount = XXX
  Some(newCount)
}
// 这可以应用在一个包含word的DStream上, 即上面代码中的pair DStream上
val runningCount = pairs.updateStateByKey[Int](updateFunction)
```
update函数将会被每个单词调用，`newValues`拥有一系列的1(来自pairs (word, 1)), 而`state`变量中拥有之前的计数次数

注意，使用`updateStateByKey`需要配置checkpoint目录。

##### transform操作
`transform`操作允许在DStream运行任何RDD-to-RDD的函数。能够被用来应用任何未在DStream API提供的RDD操作。
例如，连接数据流中的每个批次(batch)和另外一个数据集的功能，并没有暴露在DStream API中。然而，却可以通过利用`transform`方法做到。
举例说明，可以通过将输入数据流与预先计算的垃圾邮件信息进行实时数据清理，然后根据它进行过滤
```scala
val spamInfoRDD = ssc.sparkContext.newAPIHadoopRDD(...) // RDD contains spam information
val cleanedDStream = wordCounts.transform({rdd => 
  rdd.join(spamInfoRDD).filter(...) // join data stream with spam information to clean data
  ... 
})
```
注意，这个函数在每个batch interval都会被调用，这就允许了可以执行随着时间变动的RDD操作，
也就是说，RDD操作，partition数量，广播变量等，都可以在batches之间发生变化

##### 窗口(window)操作
任何window操作都需要设置如下两个参数:
1. window length: duration of window，窗口的大小
2. sliding interval: 每次滑动的长度
Spark Streaming允许在一个滑动窗口的数据上应用transformation。
比如，要来统计过去30s的词频，间隔时间是10s，那么就可以如下设置参数:
`val windowWordCount = pairs.reduceByKeyAndWindow(_+_, Seconds(30), Seconds(10))`

一些常见的window操作如下:
+ window(windowLength, slideInterval): 返回基于windowed的批次的DStream的新DStream
+ countByWindow(windowLength, slideInterval): 返回一个滑动窗口的流数据中元素的个数
+ reduceByWindow(func, windowLength, slideInterval): 返回一个单元素的流，通过使用func函数聚合滑动窗口里的所有元素，
func必须是符合结合律和交换律，这样可以保证在并行计算中正确执行
+ reduceByKeyAndWindow(func, windowLength, slideInterval[, numTasks]): 当在一个键值对(K, V)DStream上调用该函数，返回一个新的(K, V)键值对DStream，
其中，每个滑动窗口的key的元素都通过func函数进行聚合
+ reduceByKeyAndWindow(func, invFunc, windowLength, slideInterval[, numTasks]): 提供发向reduce函数，加速计算过程
+ countByValueAndWindow(windowLength, slideInterval[, numTasks]): 当在一个键值对(K, V)DStream上调用该函数，返回一个新的(K, Long)键值对DStream，
其中，每个key的值是滑动窗口中这个key的出现的频次

##### join操作
可以很轻松的在Spark Streaming中执行不同类型的join
```scala
val stream1: DStream[String, String] = ...
val stream2: DStream[String, String] = ...
val joinedStream = stream1.join(stream2)
```
此外，在stream的window(窗口)上进行join也是常用的操作:
```scala
val windowStream1 = stream1.window(Seconds(20))
val windowStream2 = stream2.window(Minutes(1))
val joinedStream = windowStream1.join(windowStream2)
```
也可以将window stream和Dataset join
```scala
val dataset: RDD[String, String] = ...
val windowStream = stream.window(Seconds(20))
val joinedStream = windowStream.transform({rdd => rdd.join(dataset)})
```

#### DStream的输出操作
输出操作将DStream的数据推送到外部系统，如数据库或文件系统
+ print()： 在Driver node中打印每个批次数据的前10个元素，用于开发和debug
+ saveAsTextFile(prefix[, suffix]): 保存DStream成文件
+ saveAsObjectFile(prefix[, suffix])：保存成序列化的Java对象的序列化文件(SequenceFile)
+ saveAsHadoopFile(prefix[, suffix]): 保存成Hadoop文件
+ foreachRDD(func): 对从流中生成的每个RDD应用func函数。此功能应将每个RDD中的数据推送到外部系统

##### 使用foreachRDD设计模式
`DStream.foreachRDD`是一个强大的原语，允许将数据发送到外部系统，但是需要避免以下常见的错误
```scala
dstream.foreachRDD { rdd => 
  val conn = createNewConnection()  // 在Driver上执行
  rdd.foreach { record => 
    conn.send(record)   // 在worker上执行
  }
}
```
**这是错误的！！！**。因为这需要将连接对象`conn`序列化并从driver发送到worker。这种连接对象很少能够跨机器进行转移，
因此可能会显示序列化错误(连接对象不允许序列化)、初始化错误(连接对象需要在worker中初始化)等问题。

正确的方法，就是在worker中创建连接对象。

但是，这又会导致另一个常见的**错误**：**为每个记录创建一个连接**
```scala
dstream.foreachRDD { rdd => 
  rdd.foreach { record => 
    val conn = createNewConnection()
    conn.send(record)   // 在worker上执行
    conn.close()
  }
}
```
**这也是错误的！！！**。因为创建连接对象通常需要时间和资源的开销，并会显著的降低系统的吞吐量。

正确的方法，就是使用`rdd.foreachPartition`。在每个RDD partition中创建一个连接对象，并在其中使用这个连接中发送所有记录
```scala
dstream.foreachRDD { rdd => 
  rdd.foreachPartition { partitionOfRecords => 
    val conn = createNewConnection()
    partitionOfRecords.foreach(record => conn.send(record))
    conn.close()
  }
}
```
这样，就可以在多个记录上分摊创建连接的开销

最后，可以通过跨多个 RDD/批次 重用连接对象来进一步优化。可以维护连接对象的静态池，这些连接对爱选哪个可以在多个发送数据到外部的batch中复用，进一步降低消耗
```scala
dstream.foreachRDD { rdd => 
  rdd.foreachPartition { partitionOfRecords => 
    val conn = ConnectionPool.getConnection()
    partitionOfRecords.foreach(record => conn.send(record))
    ConnectionPool.returnConnection(conn) // 将连接放回连接池实现重用
  }
}
``` 
需要注意的几点：
+ 和RDDs一样，DStream的计算也是惰性的，要等到输出操作的时候才开始计算。
明确的说，DStream输出操作内部的RDD actions会强行开始处理接收到的数据。因此，如果你的程序没有任何输出操作，或者在`dstream.foreachRDD()`中没有任何RDD的action.
那么任何计算都不会被执行


#### DataFrame和SQL操作
可以很轻松的在流数据上使用DataFrames And SQL操作。
可以直接使用StreamingContext内部的的SparkContext来创建一个SparkSession
```scala
// 在Steaming程序中的DataFrame操作
val words: DStream[String] = ...
words.foreachRDD { rdd => 
  val spark = SparkSession.builer.config(rdd.sparkContext.getConf).getOrCreate()
  import spark.implicits._
  
  val wordsDataFrame = rdd.toDF("word")
  
  wordsDataFrame.createOrReplaceTempView("words")
  
  val wordCountsDataFrame = spark.sql("select word, count(*) as total from words group by word")
  wordCountsDataFrame.show()
}

```

#### MLlib操作
略

#### 缓存/持久性
和RDD类似，DStream允许开发人员将流的数据保存在内存中，在DStream上使用`persist()`方法会自动将该DStream的每个RDD保留在内存中。

+ 对于窗口操作和基于状态的操作，这些都是隐含的被保存在内存中，不需要手动调用`persist()`
+ 对于通过网络接收数据(Kafka, Flume等)的输入流，默认持久级别被设置为 *将数据复制到两个节点进行容错*

其中，与RDD不同的是，DStream的默认持久级别将数据序列化在内存中。更多信息见`性能调优`章节

#### Checkpointing
spark streaming程序必须可以7*24小时运行，因此必须对与应用逻辑无关的故障(比如JVM崩溃等)具有弹性。因此需要`checkpoint`足够的信息来到容错存储系统，以便从故障中恢复、
checkpoint有两种类型的数据
+ Metadata checkpointing: 将定义streaming计算的信息保存到容错存储(如HDFS)中，用于从运行streaming应用程序的driver节点的故障中恢复。元数据包括
    + Configuration: 用于创建流应用程序的配置
    + DStream operations: 定义streaming应用程序的DStream操作集
    + Incomplete batches: job在排队未完成的批次
+ Data checkpointing: 将生产的RDD保存到可靠存储。这在一个将多个批次之间的数据进行组合的`状态`变换中是必须的。在这种转换中，生成的RDD依赖于先前批次的RDD。
这将导致依赖的长度随时间而增加，为了避免故障恢复时间随着时间无限增加，有状态转换的中间RDD会定期checkpoint到可靠的存储(比如HDFS)

总而言之，**Metadata checkpointing主要用于从driver故障中恢复，而如果使用了stateful transformations, 
Data/RDD checkpointing就是必须的**

##### 何时启用checkpoint
对于以下任一要求的应用程序，必须启用checkpoint
+ 使用了状态转换: 比如`updateStateByKey`或`reduceByKeyAndWindow`，必须提供checkpoint目录以允许定期的Data/RDD Checkpointing
+ 从运行应用程序的driver的故障中恢复-用Metadata checkpoints来恢复

配置checkpoint
可以在一个可容错的可靠的文件系统作设置一个目录用于保存checkpointing信息,比如HDFS, S3等。
使用`streamingContext.checkpoint(checkpointDirectory)`来实现。

另外，如果希望程序可以从driver错误中恢复，那就必须重新改写程序，添加以下操作:
+ 当程序第一次启动时，创建一个新的`StreamingContext`, 设置所有的流，然后调用`start()`
+ 当程序从错误中重新启动时，从checkpoint directory的checkpoint data中重新创建一个`StreamingContext`

```scala
def funcToCreateContext(): StreamingContext = {
  val ssc = new StreamingContext(...)
  val lines = ssc.socketTextStream(...)
  ...
  ssc.checkpoint(checkpointDirectory)
  ssc
}
val context = StreamingContext.getOrCreate(checkpointDirectory, funcToCreateContext _)

context. ...

context.start()
context.awaitTermination()
```

为了使用`StreamingContext.getOrCreate`, 必须要使得Driver Program可以在失败的时候自动重启，这可以通过部署来设置，详见`部署`(Deploying Applications)章节

注意到RDD的checkpointing保存到可靠存储的代价，这可能会导致增加每个批次(batch)的处理时间。因此，checkpointing的间隔必须小心设置。
对于一个小的batch(比如1 second), 每个batch都checkpointing的话会严重缩减吞吐量。相仿，太少的checkpointing会导致任务大小增长，会导致很多额外的不利影响。

对于stateful transformations这种需要RDD checkpointing的操作，默认的interval就是batch interval的倍数，最少10秒。
可以通过`dstream.checkpoint(checkpointInterval)`来设置,通常情况下，5~10倍的滑动窗口大小是一个很好的设置

#### 累加器，广播变量和Checkpoints
在Spark Streaming中，无法从checkpoint恢复累计器和广播变量，如果启用了checkpointing并使用累加器和广播变量，必须为累加器和广播变量创建延迟实例化的单例实例，
以便在driver重启失败后重新实例化

```scala
import org.apache.spark._
import org.apache.spark.broadcast.Broadcast
import org.apache.spark.util.LongAccumulator

object WordBlackList {
  @volatile private var instance: Broadcast[Seq[String]] = null
  def getInstance(sc: SparkContext): Broadcast[Seq[String]] = {
    if (instance == null) {
      synchronized {
        if (instance == null) {
          val wordBlackList = Seq("a", "b", "c")
          instance = sc.broadcast(wordBlackList)
        }
      }
    }
    instance
  }
}

object DroppedWordsCount {
  @volatile private var instance: LongAccumulator = null
  def getInstance(sc: SparkContext): LongAccumulator = {
        if (instance == null) {
          synchronized {
            if (instance == null) {
              instance = sc.longAccumulator("wordsInBlackListCounter")
            }
          }
        }
        instance
  }
}

wordCounts.foreachRDD { (rdd: RDD[(String, Int)], time: Time) => 
  val blackList = WordBlackList.getInstance(rdd.sparkContext)
  val droppedWordsCount = DroppedWordsCount.getInstance(rdd.sparkContext) 
  val counts = rdd.filter(...).collection.mkString("[", ", ", "]")
}

```

#### 应用程序部署 Deploying Applications
这里讨论如何部署一个Spark Streaming应用

##### 要求
+ 集群和一个集群管理器
+ 打包应用程序: 编译为JAR，如果使用`spark-submit`启动应用程序，在Jar包里不需要提供Spark和Spark Streaming.
然后，如果使用了任何一种高级数据源(比如Kafka, Flume等), 那么需要打包额外artifact和所有依赖。
+ 为executor/worker配置足够的内存
+ 配置checkpointing: 如果应用需要，设置一个支持Hadoop API的可容错的存储目录作为checkpointing目录，然后修改程序，使其发送错误时从checkpointing中恢复数据
+ 配置应用程序driver的自动重启，故障时重新拉起：不同的cluster manager有不同的实现
    + Spark Standalone: Driver自己占用一个worker node运行，cluster manage用来监控driver, 当driver发生非0错误时重新启动
    + YARN: 类似的自动重启机制
    + Mesos: Marathons
+ 配置预写日志(write-ahead logs): 如果设置开启，所有来自Receiver的数据都会被写入checkpointing目录的一个预写日志，用来防止Driver恢复过程中的数据丢失，实现`0数据丢失`。
配置`spark.streaming.receiver.writeAheadlog.enable`为`true`。但是，这是以降低单个receiver的吞吐量为代价的。
推荐的做法是，在Spark中，在设置`writeAheadlog`为`true`时，设置`复制接收到的数据`为`false`。
可以通过设置输入存储级别为`StorageLevel.MEMORY_AND_DISK_SER`。
+ 设置最大接收速率：如果集群的资源不够，即处理数据的速度赶不上接收数据的速度，那么可以给receivers设置一个最大接收速度(terms/second).

在Spark1.5以后，可以设置`spark.streaming.backpressure.enabled`为`true`来让Spark Streaming自动计算速率限制

#### 监控应用程序
Spark Web UI额外显示一个Streaming的选项卡，显示running receivers的统计信息

web UI中两个指标比较重要：
+ Processing Time(处理时间): 处理每batch(批次)数据的时间
+ Scheduling Delay(调度延迟): batch在队列中等待之前批次处理完成的时间

如果processing time始终超过batch interval(批次间隔时间), 或者queueing delay不断增加，
就表示系统无法及时处理批次，这种情况下，就要考虑**减少批处理时间**

### 性能调优
想要在集群上的Spark Streaming应用中获得最佳性能，需要一些列的调优，从高层次上，需要考虑两件事情:
1. 通过有效利用集群资源，减少每批数据的处理时间
2. 设置正确的batch size, 以便数据的处理时间可以跟上接收速度

#### 减少批处理时间
Spark中可以进行一些优化来最小化每批处理时间，参考`调优指南`章节

##### 数据接收中的并行级别
通过网络接收数据，需要反序列化数据并存储在Spark中，如果数据接收是系统的瓶颈，那么考虑下并行化接收数据。

注意每个input DStream创建接收单个数据流的单个接收器。因此，可以通过创建多个input DStreams来实现接收多个数据流。
并配置它们以从source接收数据流的不同分区。

比如一个Kafka input DStream同时接收两个topic的数据，可以将输入数据划分成2个Kafka Input stream, 这样就可以创建2个receivers,允许数据并行的被接收。

```scala
val numStreams = 5
val kafkaStreams = (1 to numStreams).map({i => KafkaUtils.createStream(...)})
val unifiedStream = streamingContext.union(kafkaStreams)
```

另外一个可以考虑的参数是`receiver's block interval`(接收器的块间隔)，由`spark.streaming.blockInterval`设置。
对于大部分receivers，接收到的数据在存储进Spark内存之前，会联合成多个blocks，每个批次(batch)中block的数量transformation这些数据所用的tasks的数量。

批次中每个receiver的task的数量大约等于`batch interval / block interval`。如果这个数据太小，那么就可能导致没有足够的core来处理数据。
在给定的batch interval下，想要增加task的数量，那么就要减少block interval.

然而，推荐最小的block interval不要小于50ms. 否则task的过多的启动开销会是个问题。

也可以明确的重新分配(repartition)输入数据流，这会在进一步处理之前将接收到的批次数据分发的集群中指定数量的机器上。
`inputStream.repartition(<number of patitions>)`，这会将收到的batches of data重新分配到集群中指定数量的机器上，用于接下来进一步处理。

##### 数据处理中的并行级别
对于分布式的reduce操作，比如`reduceByKey`和`reduceByKeyAndWindow`，默认的并行task数量是由`spark.default.parellelism`来控制的。
通过改变这个设置来更改数据处理的并行度


##### 数据序列化
可以调整序列化的格式来减小数据序列化的开销，对于Streaming应用来说，有2种数据是被序列化的：
+ Input data: 默认的，通过Receivers接收到的数据会存储在executor的内存中，存储级别设置为`StorageLevel.MEMORY_AND_DISK_SER_2`, 
意思是数据序列化成字节，来减少GC的开销，并且复制来用于容错。当内存不足时，数据会溢出到磁盘。这种序列化显然是有开销的，
每个Receiver必须先反序列化接收到的数据，然后用Spark的序列化格式重新序列化数据
+ Streaming操作中持久化的RDDs：比如在window操作会持久化数据到内存，因为它们会被多次处理。然而，不同于Spark Core默认的`StorageLevel.MEMORY_ONLY`,
持久化的RDDs默认使用`StorageLevel.MEMORY_ONLY_SER`来最小化GC开销。

在这两种数据序列化中，都可以使用`Kryo`序列化来减少CPU核内存的消耗。

如果你的应用batch interval很小(比如几秒钟), 而且没有使用任何window操作，那么完全可以尝试在持久化数据时**禁止序列化**。

##### 任务启动消耗
如果task启动的频率过高，比如每秒超过50个, 那么将任务发送给子节点(slaves)的开销就会非常大，可以通过以下方法来减少开销:
+ 运行模式: Standalone模式或者粗粒度(coarse-grained)的Mesos模式将比细粒度(fine-grained)Mesos模式获得更好的task启动时间


#### 设置正确的批次间隔
batch interval的设置对给定集群处理数据的速率有较大的影响。

一个好的设置争取的batch interval的方法就是先从一个保守的值(比如5~10秒)和较低的数据速率来测试。为了验证系统是否跟得上数据的速率，
可以检查每个批次处理的end-to-end delay(端到端延迟), 如果这个延迟和batch size大小差不多, 那么系统就是稳定的。
如果这个延迟持续的增加，就说明系统跟不上。尝试增加data rate或者缩减batch size，来达到一个稳定的配置

#### 内存调优
Spark Streaming程序所需要的集群内存大小依赖于使用的transformations的类型，例如，你想要使用一个窗口操作过去10分钟的数据，那么你的集群内存就必须要可以存储10分钟的数据量。
或者你要使用`updateStateByKey`来更新大量的key, 那么所需的内存必然会很高。
相反，如果你仅仅是做简单的filter或者map操作，所需的内存就较少。

总的来说，通过Receivers接收到的数据会以`StorageLevel.MEMORY_AND_DISK_SER_2`的级别进行存储，内存不够时数据会溢出到磁盘。这样会大大影响程序的效率，
因此，建议给streaming应用分配足够的内存空间。

另外一个内存调优的考量就是垃圾回收，作为需要低延迟的streaming应用，显然大量的暂停来执行JVM GC是不现实的

下面有一些参数可以来调整内存使用和垃圾回收的开销:
+ DStreams的持久化级别: 相比较于反序列化，开启序列化更有益于缩减内存使用和GC开销；开启Kryo序列化有利于进一步优化；
更多的优化可以通过使用以CPU时间为消耗的数据压缩来进行
+ 清除旧数据: 一般情况下，DStreams转换操作生成的所有数据和持久化RDDs都会自动的被清理
+ CMS垃圾回收器: 建议使用concurrent mark-and-sweep GC. 通过在driver中设置(在`spark-submit`时设置`--driver-java-option`）
以及在executor中设置(使用`spark.executor.extraJavaOptions`)来使用CMS GC
+ 其他: 为了更进一步缩减GC开销，可以考虑下几个策略：
    + 使用`OFF_HEAP`级别来持久化RDDs
    + 使用更小heap size的executor


**几个需要注意的点:**
1. 一个DStream和一个receiver相关联，一个receiver在executor上运行，占用一个核，因此，在receivers占位了之后，需要有足够的核来处理数据。
设置`spark.cores.max`时需要考虑receiver的位置。
2. 当数据从stream source接收到时，receiver会创建多个blocks. 每个batch interval会产生大约`batchInterval/blockInterval`个block.
这些blocks会通过当前executor的BlockManager分发到其他executors的BlockManager. 在此之后，在Driver上运行的Network Input Track会被告知这些blocks的位置，
用于后续处理
3. 在Driver中，每个batch interval会为这些blocks创建一个RDD，这些blocks都是RDD的分区(partitions), 在Spark中，每个partition上分配一个task
4. 除非是非本地调度，否则，在blocks上的map task会在executor that has the blocks irrespective of block interval中执行.
也就是说更大的blockInterval有更大的blocks。设置更高的`spark.locality.wait`可以增加block在本地处理的机会。尽量保证较大的block在本地被处理
5. 除了依赖batchInterval和blockInterval这两个参数，你也可以直接指定partition的数量`inputDStream.repartition(n)`.
这会将RDD的数据重新随机分配到n个partition中。每个RDD的操作会被Driver的job scheduler调度为一个job，任何时刻只有1个job是活跃的，其他的都在排队。
6. 如果你有两个DStream, 那么就有2个RDDs形成，就会有2个job被创建，然后依次的被调度执行。为了避免这种情况，你可以`union`这两个DStream，
这样就会被当做是一个job
7. 如果batch处理时间大于batch interval, 那么receiver的内存会很快被填满并且抛出异常，设置receiver的rate来解决`saprk.streaming.receiver.maxRate`.


#### 容错语义
略