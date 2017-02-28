## 前言
本文使用Spark SQL的场景是这样的：在公司里面，数据分析师都用Zeppelin的JDBC Interpreter去连接Spark Thrift Server，如果不经过任何配置，很容易就导致某个用户的某个大SQL占用了所有资源，导致其他用户的任务没发执行。

本文的Spark是基于Yarn。
## SQL Parser
## 作业调度
### 配置详解
依据阅读清单3，Spark作业调度的方式可以这样细分：

+ FIFO
+ FAIR
  - 默认配置：只有一个名为default的队列(pool)，所有的任务`默认`提交到default队列，
  - 定制：通过`spark.scheduler.allocation.file`，指定配置文件的路径

在不做任何配置的情况下，default队列的调度模式是FIFO，所以如果你只在配置文件(spark-defaults.conf)中将调度模式改成FAIR，从结果上而言你用的还是FIFO。

使用Spark Thrift Server非常需要公平调度，一种简单的办法是将default队列的调度模式定制为FAIR，这种方法保证了每条SQL的执行都是“公平”的。另一种方法是在SQL执行前指定队列，只要任意指定队列名，调度器就会动态生成一个FIFO的默认配置队列。比如将用户名指定为队列名，那么就可以保证用户之间是公平的。

FAIR模式下配置多队列，每个队列都可以配置调度模式(schedulingMode)、权重(weight)和minShare。minShare可以保证队列的基础资源。要完全理解这几个参数含义，当然还是要看源码，而这个部分的源码可读性非常好。([Spark源码传送门](https://github.com/apache/spark/blob/v2.1.0/core/src/main/scala/org/apache/spark/scheduler/SchedulingAlgorithm.scala))

### 指定任务的队列
对于多队列的模式，在一般的Spark任务中：
``` scala
// Assuming sc is your SparkContext variable
sc.setLocalProperty("spark.scheduler.pool", "pool1")
```

在Spark Thrift Server的JDBC Session中，可以
```
0: jdbc:hive2://localhost:10000> set spark.sql.thriftserver.scheduler.pool=test;
+----------------------------------------+--------+--+
|                  key                   | value  |
+----------------------------------------+--------+--+
| spark.sql.thriftserver.scheduler.pool  | test   |
+----------------------------------------+--------+--+
1 row selected (2.559 seconds)
```

## Reading List
1. [搭建Spark源码阅读和调试环境](http://zhihu.com/question/24869894/answer/97339151)
2. [Spark 2.0 Tuning Guide](http://www.slideshare.net/jcmia1/apache-spark-20-tuning-guide)
3. [Spark官方文档：Job Scheduling](https://spark.apache.org/docs/latest/job-scheduling.html)
4. [Mastering Apache Spark 2](https://jaceklaskowski.gitbooks.io/mastering-apache-spark/)
5. [Spark调度（一）：Task调度算法，FIFO还是FAIR](http://www.datastart.cn/tech/2016/07/11/spark-scheduler.html)
6. https://github.com/antlr/antlr4/blob/master/doc/tree-matching.md
