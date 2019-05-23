#### 使用spark-submit启动applications

```sbtshell
./bin/spark-submit \
    --class <main-class> \
    --master <master-url> \
    --deploy-mode <deploy-mode> \
    --conf <key>=<value> \
    ...
    <application-jar> \
    [application-arguments]
```
+ --class: application的入口点
+ --master: 集群的master url, 比如spark://23.185.26.187:7077
+ --deploy-mode: 选择是否将driver部署在worker nodes(cluster)上或者作为一个外部client来运行(默认client)
+ --conf: 任何的spark configuration属性，`key=value`格式
+ application-jar: 打包好的application,
+ application-arguments: main方法的参数，可选

通常的部署模式是通过一台网关机器提交你的application, 这个网管机器和集群机器物理上相连。这时候，使用`client`模式更为合适，
这种模式下，driver是直接运行在一个充当集群`client`的`spark-submit`进程上的，输入和输出都会直接连接到console, 
因此，这种模式特别适合涉及到REPL的applications

相对的，如果你的application是通过一台远程机器提交到集群的，通常使用`cluster`模式，以此来缩减drivers和executors的延迟


如果使用Spark Standalone集群模式，并且使用`cluster`deploy-mode, 可是设置`--supervise`来确保发生非0错误时driver可以自动重启。
下面是一些常用选项的例子:
```sbtshell
# locally on 8 cores
./bin/spark-submit \   
    --class org.apache.spark.example.SparkPi \
    --master local[8] \
    /path/to/examples.jar \
    100
    
# on a Spark Standalone cluster in `client` delpoy mode
./bin/spark-submit \
    --class org.apache.spark.example.SparkPi \
    --master spark://207.184.161.138:7077 \
    --executor-memory 20G \
    --total-executor-cores 100 \
    /path/to/examples.jar \
    1000

# on a Spark Standalone cluster in `cluster` deploy mode with supervise
./bin/spark-submit \
    --class org.apache.spark.example.SparkPi \
    --master spark://207.184.161.138:7077 \
    --deploy-mode: cluster \
    --supervise \
    --executor-memory 20G \
    --total-executor-cores 100 \
    /path/to/examples.jar
    1000

# on a Mesos cluster in `cluster` deploy mode with supervise
./bin/spark-submit \
    --class org.apache.spark.example.SparkPi \
    --master mesos://207.184.161.138:7077 \
    --deploy-mode: cluster \
    --supervise \
    --executor-memory 20G \
    --total-executor-cores 100 \
    /path/to/examples.jar
    1000
```

#### 从文件加载配置
默认使用`conf/spark-defaults.conf`来加载配置。
直接显示的在`SparkConf`上的配置具有最高优先级，其次是传递给`spark-submit`的flag. 再其次是这个默认的文件


#### 高级依赖管理
