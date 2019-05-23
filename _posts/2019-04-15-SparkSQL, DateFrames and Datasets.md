## Spark SQL, DataFrames and Datasets

SparkSQL提供了查询结构化数据以及计算结果等信息的接口。
有几种方式可以和Spark SQL交互，包括SQL和Dataset API.

### 概述
Spark SQL是Spark处理结构化数据的一个模块。Spark SQL提供了查询结构化数据及计算结果的接口，

Datasets和DataFrames

Datasets是一个分布式数据集合，提供了RDD的优点(强类型化、可以用lambda函数)与Spark SQL执行引擎的优点

一个DataFrame是一个`Dataset`组成的指定列，DataFrames可以从大量的source中构造出来(比如文本文件、Hive、外部数据库、或者已存在的RDDs)

在Scala API中，`DataFrame`仅仅是一个`Dataset[Row]`类型的别名

### 入门

#### 起始点: SparkSession
Spark中所有功能的入口点是SparkSession类, 创建一个SparkSession,使用`SparkSession.builder()`就可以了
```scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession
    .builder()
    .appName("Spark SQL basic example")
    .config("spark.some.config.option", "some-value")
    .getOrCreate()
// 用于一些隐式的转换，比如从RDDs转换成DataFrames
import spark.implicits._
```

#### 创建DataFrames
在一个`SparkSession`中，可以从一个已经存在的RDD、Hive表或者从Spark数据源中创建一个DataFrame
```scala
val df = spark.read.json("xx.json")
df.show
```

#### 无类型的Dataset操作
**DataFrames在Scala中，仅仅是多个`Row`的Dataset**
```scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession
    .builder()
    .appName("Spark SQL basic example")
    .config("spark.some.config.option", "some-value")
    .getOrCreate()
// 用于一些隐式的转换，比如从RDDs转换成DataFrames
import spark.implicits._

val df = spark.read.json("xx.json")
df.printSchema()
df.select("name").show()
df.select($"name", $"age" + 1).show()
df.filter($"age" > 20).show()
df.groupBy("age").count().show()
```

#### 编程式的执行SQL查询
```scala
df.createOrReplaceTempView("people")
val sqlDF = spark.sql("select * from people")
sqlDF.show()
```

#### 全局临时视图
Spark SQL中的临时视图是session级别的，会随着session的结束而消失，如果想要一个临时视图在所有的session中可以相互传递且可用，知道spark应用程序退出为止，
可以创建一个全局的临时视图。
```scala
df.createGlobalTempView("people")
spark.sql("select * from global_temp.people").show()

// 新的session中依然存在
spark.newSession().sql("select * from global_temp.people").show()
```

#### 创建Datasets
Datasets与RDDs类似, 不同的是，并不是使用Java序列化或者Kryo来序列化，而是使用一种特定的编码器来序列化对象，以便在网路中处理和传送
```scala
case class Person(name: String, age: Long)

val caseClassDS = Seq(Person("Andy", 32)).toDS()
caseClassDS.show

val primitiveDS = Seq(1, 2, 3).toDS()
primitiveDS.map(_+1).collect

// 也可以通过提供一个类来讲DataFrames转换为Dataset
val peopleDS = spark.read.json("people.json").as[Person]
peopleDS.show()
```

#### RDD的互操作性 RDD转换为DataFrame/Dataset
Spark SQL支持两种方法将已存在的RDD转为Dataset
+ 反射推断一个包含指定的对象类型的RDD的Schema
+ 构造一个Schema，然后应用到一个因存在的RDD的编程接口，用于类型未知的RDD

##### 反射推断Schema
```scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession
    .builder()
    .appName("Spark SQL basic example")
    .config("spark.some.config.option", "some-value")
    .getOrCreate()
// 用于一些隐式的转换，比如从RDDs转换成DataFrames
import spark.implicits._

// 定义table的schema
case class Person(name: String, age: Long)

val peopleDF = spark.sparkContext
    .textFile("/usr/local/spark/examples/src/main/resources/people.txt")
    .map(_.split(" "))
    .map(attr => Person(attr(0), attr(1).trim.toInt))
    .toDF()
peopleDF.createOrReplaceTempView("people")
val teenagersDF = spark.sql("select * from people where age between 13 and 19")

teenagersDF.map("Name: " + _(0)).show()

teenagersDF.map("Name: " + _.getAs[String]("name")).show()

// 未提前定义的encoders for Dataset[Map[K, V]]
implicit val mapEncoder = org.apache.spark.sql.Encoders.kryo[Map[String, Any]]
teenagersDF.map(t => t.getValuesMap[Any](List("name", "age"))).collect
```

##### 以编程方式指定Schema
当case class不能够在执行之前被定义，可以通过如下方式来创建DataFrame
1. 从原始的RDD创建RDD的`Row`(行)
2. 创建Schema表示一个`StructType`匹配RDD中的`Row`的结构
3. 通过`SparkSession.createDataFrame`应用Schema到RDD的`Rows`
```scala
import org.apache.spark.sql.types._
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.Row

val spark = SparkSession
    .builder()
    .appName("Spark SQL basic example")
    .config("spark.some.config.option", "some-value")
    .getOrCreate()

val peopleRDD = spark.sparkContext.textFile("/usr/local/spark/examples/src/main/resources/people.txt")
val schemaString = "name age"
val fields = schemaString.split(" ")
    .map(StructField(_, StringType, true))
val schema = StructType(fields)

val rowRDD = peopleRDD
    .map(_.split(", "))
    .map(attr => Row(attr(0), attr(1).trim))

val peopleDF = spark.createDataFrame(rowRDD, schema)

peopleDF.createOrReplaceTempView("people")

val results = spark.sql("select name from people")

```

#### Aggregations(聚合)
略

### DataSource(数据源)
Spark SQL通过支持DataFrame接口对各种数据源进行操作，可以使用`relational transformations(关系转换)`操作，也可以创建`temporary view`(临时视图)。
将DataFrame注册为临时视图允许了对数据运行SQL查询。

#### 通用 加载/保存 功能
默认数据源`parquet`
```scala
val usersDF = spark.read.load("/usr/local/spark/examples/src/main/resources/users.parquet")
usersDF.select("name", "favorite_color").write.save("namesAndFavColors.parquet")
```

##### 手动指定选项
`spark.read.format('json').load("**.json")`
当然，也可以使用短名称
`json`, `parquet`, `jdbc`, `orc`, `libsvm`, `csv`, `text`等

##### 直接在文件上运行SQL
不用读取API将文件加载到DataFrame, 可以直接使用SQL查询
```scala
val sqlDF = spark.sql("select * from parquet.`/usr/local/spark/examples/src/main/resources/users.parquet`")
```

##### 保存模式
保存操作可以选择使用`SaveMode`，指定如果现有数据存在时的操作。
+ SaveMode.ErrorIfExists
+ SaveMode.Append
+ SaveMode.Overwrite
+ SaveMode.Ignore： 类似于CREATE TABLE IF NOT EXIST

##### 保存到持久表
DataFrames可以使用`saveAsTable`命令作为持久表保存到Hive MetaStore中，Spark会创建默认的local Hive metastore
`saveAsTable`将实现DataFrame的内容，并在创建一个指向Hive metastore中数据的指针，即使Spark重新启动，持久表仍然存在

可以通过使用表名，用`SparkSession.table("tableName")`来创建一个持久表的DataFrame

对于文件的数据源，比如text, parquet, json等，可以通过`path`选项指定自定义表路径，例如
`df.write.option("path", "/some/path").saveAsTable("t")`
如果指定了自定义表路径，即使当表被drop时，自定义表路径也不会删除，表数据仍然存在
如果没有指定，会使用仓库目录下的默认表路径，则当表被删除时，默认的表路径也会被删除

##### 分桶，排序和分区
对于基于文件的数据源，可以对输出进行bucket(分桶)和sort(排序)或者partition(分区)
bucket和sort仅适用于持久表
`peopleDF.write.bucketBy(42, "name").sortBy("age").saveAsTable("people_bucketed)`

partition可以和`save`以及`saveAsTable`一起使用
`usersDF.write.partitionBy("favorite_color").format("parquet").save("namesPartByColor.parquet")`

#### Parquet文件
Parquet是一种很多数据处理系统都支持的柱状格式。

##### 以编程方式加载数据
加载parquet数据
```scala
import spark.implicits._
val peopleDF = spark.read.json("xx.json")
peopleDF.write.parquet("people.parquet")

val parquetFileDF = spark.read.parquet("people.parquet")
parquetFileDF.createOrReplaceTempView("parquetFile")
val namesDF = spark.sql("select name from parquetFile where age between 13 and 19")
namesDF.map(attr => "Name: " + attr(0)).show

```

##### 分区发现
表分区是一种在Hive中常见的优化手段。在一个分区表中，数据按照分区列值编码被存储在不同的分区目录中。
所有内建的文件数据源(text, csv, json, orc, parquet)都支持自动发现和推断分区信息

从Spark1.6.0其，分区发现仅仅在给定的的目录下面进行。用户需要制定分区发现开始的基本路径，
比如`path/to/table/gender=male`是用户将`basePath`设置为`path/to/table`, `gender`就将是一个分区列

##### 模式合并
和PB、Avro和Thrift一样，Parquet也支持模式演进(schema evolution),用户可以从一个简单的schema开始，根据需要逐渐的向schema添加更多的列。
使用这种方式，可以使用不同但相互兼容的多个parquet文件。Parquet数据源现在支持自动发现这种模式并且合并所有这些文件的schema

由于schema合并是一个相对昂贵的操作，因此1.5.0以后就默认关闭了该功能。可以通过如下设置开启:
1. 读取parquet文件时，设置数据源的option `mergeSchema`为`true`
2. 或者设置全局的SQL option: `spark.sql.parquet.mergeSchema`为`true`

```scala
import spart.implicits._
val spuaresDF = spark.sparkContext.makeRDD(1 to 5).map(i => (i, i*i)).toDF("value", "square")
spuaresDF.write.parquet("data/test_table/key=1")

val cubesDF = spark.sparkContext.makeRDD(6 to 10).map(i => (i, i * i * i)).toDF("value", "cube")
cubesDF.write.parquet("data/test_table/key=2")

// 读取partition table
val mergedDF = spark.read.option("mergeSchema", "true").parquet("data/test_table")
mergedDF.printSchema()

// 最终schema中包含3列
// --value
// --square
// -- cube
```

##### Hive metastore Parquet Table转换
略

##### 配置
parquet的一些配置可以通过使用`SparkSession`的`setConf`来完成

#### JSON数据集
SparkSQL可以自动推断jason数据集(Dataset)的schema, 并将其作为`Dataset[ROW]`加载。
通过在`Dataset[String]`或者Json文件上使用`SparkSession.read.json()`来完成
```scala
import scala.implicits._

val path = "xx.json"
val peopleDF = spark.read.json(path)

// 查看自动推断出来的schema
peopleDF.printSchema()
peopleDF.createOrReplaceTempView("people")

// 从Dataset[String]上构建JSON DataFrame
val otherPeopleDataset = spark.createDataset(
"""{"name":"Yin", "address":{"city":"Columbus", "state":"Ohio"}}""" :: Nil
)
val otherPeople = spark.read.json(otherPeopleDataset)
otherPeople.show

```

#### Hive表
略

#### JDBC连接其他数据库
Spark SQL还可以使用JDBC从其他数据库读取数据，结果也会作为DataFrame返回。
这种方法要优先于直接使用`JdbcRDD`, 这是因为返回的结果是DataFrame, 可以直接的被Spark SQL处理以及与其他数据源join。

在Spark Shell中要使用JDBC必须在class path中添加JDBC driver
`bin/spark-shell --driver-class-path mysql-connector-java-5.1.40-bin.jar --jars mysql-connector-java-5.1.40-bin.jar`

```scala
// load
val jdbcDF = spark.read.jdbc
    .option("url", "jdbc:mysql://localhost:8889/spark")
    .option("user", "username")
    .option("password", "password")
    .load()
    
val connectionProperties = new Properties()
connectionProperties.put("user", "userName")
connectionProperties.put("password", "password")
val jdbcDF2 = spark.read.jdbc("jdbc:mysql://localhost:8889/spark", connectionProperties)

// save
jdbcDF.write
    .format("jdbc")
    .option("url", "jdbc:mysql://localhost:8889/spark")
    .option("user", "username")
    .option("password", "password")
    .save()

```

#### Apache Avro数据源
略

### 性能调优
对于一些工作负载，可以通过caching data in memory或者是调试一些实验性的选项来提高系统的效率

#### Caching Data in Memory
Spark SQL可以通过调用`spark.catalog.cacheTable("tableName")`或`dataFrame.cache()`来使用内存中的列格式来缓存表

Spark SQL将只扫描所需要的列，并将自动调整压缩以最小化内存使用量和GC压力。
调用`spark.catalog.uncacheTable("tableName")`从内存中删除该表

内存缓存的配置可以使用`SparkSession`上的`setConf`方法或使用SQL运行`set key=value`命令
+ spark.sql.inMemoryColumnarStorage.compressed: 默认true, Spark SQL将根据数据的统计信息为每个列自动选择一个压缩编解码器
+ spark.sql.inMemoryColumnarStorage.batchSize: 默认10000, 控制批量的柱状缓存大小，更大的数值可以提高内存利用率和压缩率，
但是在缓存数据时会冒出OOM(out of memory)风险

#### Broadcast Hint for SQL Queries
略

### 分布式SQL引擎
通过JDBC/ODBC或命令行, Spark SQL可以作为一种分布式的查询引擎。在这种模式下，
终端用户或应用程序可以直接通过Spark SQL来运行SQL 查询，并不需要任何代码

#### 运行Thrift JDBC/ODBC服务器
略

#### 运行Spark SQL Cli
`.bin/sparl-sql`