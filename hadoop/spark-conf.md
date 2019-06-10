spark 提交命令

spark-submit
===================================
shell
----------------------------------
```
spark-submit \
--master yarn  \
--class com.test.project.streaming \
--num-executors 3 \
--executor-memory 1g \
--driver-memory 1g \
--executor-cores 1 \
--queue default \
--conf spark.yarn.maxAppAttempts=4 \
--conf spark.yarn.am.attemptFailuresValidityInterval=1h \
--conf spark.yarn.max.executor.failures=24 \
--conf spark.yarn.executor.failuresValidityInterval=1h \
--conf spark.task.maxFailures=8 \
--name "My_streaming_test1" \
--deploy-mode cluster ./test.jar
```


livy batches post body 
------------------------------
```
{
	    "proxyUser": "test",
	    "file": "/user/test/jars/testanalysis.jar",
	    "name":"my-streaming-test",
	    "numExecutors":4,
	    "executorCores":1,
	    "executorMemory":"1g",
	    "driverMemory":"1g",
	    "conf": {"spark.yarn.maxAppAttempts" :4,"spark.yarn.am.attemptFailuresValidityInterval" :"1h"},
	    "className": "com.test.project.streaming"  
}
```
java code httpclient post
------------------------------
```
 Map<String, Object> params = new HashMap<>(9);
 params.put("file", "hdfs://"+JAR_PATH+"streamanalysis.jar");
 params.put("proxyUser", USER);
 params.put("name", applicationName);
 params.put("className", "com.test.project.streaming");
 params.put("executorMemory", "1g");
 params.put("executorCores", 1);
 params.put("numExecutors", 3);
 params.put("driverMemory", "1g");
 Properties prop = new Properties();
 prop.put("spark.yarn.maxAppAttempts", 4);
 prop.put("spark.yarn.am.attemptFailuresValidityInterval", "1h");
 prop.put("spark.yarn.max.executor.failures", 24);
 prop.put("spark.yarn.executor.failuresValidityInterval", "1h");
 params.put("conf", prop);
```
spark conf 配置
==============================
*num-executors
设置spark任务是需要多少个executor来执行，这个可以参照任务的大小和集群的规模来设置；想要最大化集群资源的话，一般是和集群的节点数保持一致，或者是集群节点数的整数倍（还需要综合考虑单节点的内存，core，能调用的资源的数量）
*executor-memory
单个excutor的能使用的内存的大小，executor-memory *num-executors 就是本次任务需要的内存，这个值，需要参考能调用的资源的大小和单节点本身的内存的大小，在满足这两个前提之下，尽可能的把这个值设置的大些；
*executor-cores   
单个executor 执行给的core的数量，不能超过单节点的core的总和;
单个core同一时间只能执行一个task，在不影响其他人作业，且不超过节点的core的上限的时候，这个值越大执行的效率越高;
*driver-memory
驱动程序内存
*queue
程序运行队列

*spark.default.parallelism 
该参数设置每个stage划分的task的数量，这个参数一定要设置。合理的设置，应该是num-executors*executor-cores的2-3倍，
每个core执行2-3个任务。要是不设置，默认的设置的task数量很少，要是不设置，之前设置的executor的数量等，
就没有起到作用，因为task很少 大部分的executor属于空闲状态。

参数调优建议：Spark作业的默认task数量为500~1000个较为合适。很多同学常犯的一个错误就是不去设置这个参数，那么此时就会导致Spark自己根据底层HDFS的block数量来设置task的数量，默认是一个HDFS block对应一个task。通常来说Spark默认设置的数量是偏少的(比如就几十个task)，如果task数量偏少的话，就会导致你前面设置好的Executor的参数都前功尽弃。试想一下无论你的Executor进程有多少个，内存和CPU有多大，但是task只有1个或者10个，那么90%的Executor进程可能根本就没有task执行，也就是白白浪费了资源！
因此Spark官网建议的设置原则是，设置该参数为num-executors * executor-cores的2~3倍较为合适，比如Executor的总CPU core数量为300个，那么设置1000个task是可以的，此时可以充分地利用Spark集群的资源。

*spark.storage.memoryFraction
 
默认占用executor 60%的内存 用于持久化rdd。这个值可以根据需要持久化的rdd的大小来进行设置，适当的增大\减小。参照下一个参数一起调整。

参数说明：该参数用于设置RDD持久化数据在Executor内存中能占的比例，默认是0.6。也就是说，默认Executor 60%的内存，可以用来保存持久化的RDD数据。根据你选择的不同的持久化策略，如果内存不够时，可能数据就不会持久化，或者数据会写入磁盘。

参数调优建议：如果Spark作业中，有较多的RDD持久化操作，该参数的值可以适当提高一些，保证持久化的数据能够容纳在内存中。避免内存不够缓存所有的数据，导致数据只能写入磁盘中，降低了性能。但是如果Spark作业中的shuffle类操作比较多，而持久化操作比较少，那么这个参数的值适当降低一些比较合适。此外，如果发现作业由于频繁的gc导致运行缓慢（通过spark web ui可以观察到作业的gc耗时），意味着task执行用户代码的内存不够用，那么同样建议调低这个参数的值。

*spark.shuffle.memoryFraction

当前stage 拉取上一个stage的task输出数据能使用的内存的大小，默认是20%。 
权衡，是shuffle操作多，还是需要cache的rdd的内存的诉求高，两个内存的占比和为80%
剩下的20%是用户执行代码所需要的内存，如果发现由于频繁的gc导致运行时间过长，可以适当的增大这个值，但是确保三个值得和是1；

参数说明：该参数用于设置shuffle过程中一个task拉取到上个stage的task的输出后，进行聚合操作时能够使用的Executor内存的比例，默认是0.2。也就是说，Executor默认只有20%的内存用来进行该操作。shuffle操作在进行聚合时，如果发现使用的内存超出了这个20%的限制，那么多余的数据就会溢写到磁盘文件中去，此时就会极大地降低性能。

参数调优建议：如果Spark作业中的RDD持久化操作较少，shuffle操作较多时，建议降低持久化操作的内存占比，提高shuffle操作的内存占比比例，避免shuffle过程中数据过多时内存不够用，必须溢写到磁盘上，降低了性能。此外，如果发现作业由于频繁的gc导致运行缓慢，意味着task执行用户代码的内存不够用，那么同样建议调低这个参数的值。
*conf
设置spark 配置
spark.cores.max 是指你的spark程序需要的总核数
spark.executor.cores 是指每个executor需要的核数
executor 数量 = spark.cores.max/spark.executor.cores
--conf spark.executor.memory=800m   是指每个executor需要的内存
--conf spark.executor.cores=2  是指每个executor需要的核数
--conf spark.cores.max   是指你的spark程序需要的总核数

--conf spark.eventLog.dir=hdfs://dbmtimehadoop/tmp/spark2 
--conf spark.eventLog.enabled=false
--conf "spark.executor.extraJavaOptions=-XX:+PrintGCDetails -XX:+PrintGCTimeStamps" 
conf 参数
https://blog.csdn.net/zhanglong_4444/article/details/85069105
https://blog.csdn.net/weixin_41008393/article/details/86503151

https://blog.csdn.net/samaritan_h/article/details/79501351
https://blog.csdn.net/u010003835/article/details/81033484


