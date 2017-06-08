@spark @sql
## 前言
本文使用Spark SQL的场景是这样的：在公司里面，数据分析师都用Zeppelin的JDBC Interpreter去连接Spark Thrift Server，如果不经过任何配置，很容易就导致某个用户的某个大SQL占用了所有资源，导致其他用户的任务没发执行。

本文尽可能从使用Spark SQL的角度讨论一些问题，如果你想读Spark SQL的源码分析，请移步至：阅读清单7和9。

## SQL Parser
Spark的SQL Parser是使用antlr4实现的，为了方便阅读源码，我们可以先用sbt，编译一下catalyst这部分源码，从`SqlBase.g4`生成相应的源码。

```
$ sbt
> project catalyst
> compile
```
编译完成之后，找到`spark/sql/catalyst/target/scala-2.11/src_managed/main/antlr4`，在Intellij Idea的右键菜单中标记此文件夹为源码文件夹。这样Spark源码中的一些类就能找到相应的代码。

### 使用Spark的SQL Parser
在实际工作中，我们往往需要使用Spark的SQL Parser解析SQL，得到我们需要的信息。比如，判断一个SQL是那种类型、抽取出SQL中所有的使用到的表。通过阅读6和Spark源码中相关的测试用例可以知道：

``` scala
package com.sadhen.boilerplate.spark.sql

import org.antlr.v4.runtime.tree.xpath.XPath
import org.antlr.v4.runtime.{ANTLRInputStream, CommonTokenStream, IntStream}
import org.apache.spark.sql.catalyst.parser.{SqlBaseLexer, SqlBaseParser}
import org.apache.spark.sql.catalyst.parser.SqlBaseParser._
import org.scalatest._
import collection.JavaConversions.collectionAsScalaIterable
/**
  * Created by rendong on 17/3/3.
  */
class ParserSpec extends FlatSpec with Matchers {

  class ANTLRNoCaseStringStream(input: String) extends ANTLRInputStream(input) {
    override def LA(i: Int): Int = {
      val la = super.LA(i)
      if (la == 0 || la == IntStream.EOF) la
      else Character.toUpperCase(la)
    }
  }

  private def buildContext[T](command: String)(toResult: SqlBaseParser => T): T = {
    val lexer = new SqlBaseLexer(new ANTLRNoCaseStringStream(command))
    val tokenStream = new CommonTokenStream(lexer)
    val parser = new SqlBaseParser(tokenStream)
    toResult(parser)
  }

  "it" should "recognize the type of the SQL" in {
    buildContext("create database hello") { parse =>
      parse.statement() match {
        case statement: CreateDatabaseContext =>
          println("this is a create database statement")
          println(s"the database is ${statement.identifier().getText}")
        case _ =>
          println("other statement")
      }
    }
  }

  "it" should "list all the db.tb" in {
    val sql = "select * from db1.tb1 where x in (select id from db2.tb2)"
    buildContext(sql) { parser =>
      parser.statement().children.foreach { tree =>
        val xpath = "//tableIdentifier"
        XPath.findAll(tree, xpath, parser).foreach { x =>
          println(x.getText)
        }
      }
    }
  }
}
```
上面的两个例子一个是利用了Spark所建立的语法树，另外一个是利用了antlr4提供的高级功能。在使用Spark SQL Parser之前，掌握一些antlr4的知识非常有必要。

Hive的SQL Parser也可以拿出来用，由于Hive使用的antlr版本比较老(antlr3)，并没有Spark SQL Parser那么易用。
## Catalyst
Databricks官方的博客(阅读清单8)说，Catalyst的可扩展设计目标有二，其一使得将新的优化技术引入到Spark SQL变得更简单，其二让开发者们自由地扩展优化器。那么，Spark SQL目前使用了哪些优化技术，我们（开发者）如何能在Catalyst的基础上扩展，应用特定优化呢？

### 流程
如图，依次是：

1. analyzing a logical plan to resolve references (`org.apache.spark.sql.catalyst.analysis`)
2. logical plan optimization (`org.apache.spark.sql.catalyst.optimizer`)
3. physical planning (`org.apache.spark.sql.execution`)
4. code generation to compile parts of the query to Java bytecode (`org.apache.spark.sql.catalyst.expressions.codegen`)

![](https://github.com/sadhen/12-technologies-in-2017/raw/master/04-Spark-SQL/assets/spark-catalyst.png)

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

## UDF

## MISC
### 编译打包
不要去编译整个工程，实际上我们可以只编译子模块：
``` bash
mvn compile -pl sql/hive-thriftserver -am
```

## Reading List
1. [搭建Spark源码阅读和调试环境](http://zhihu.com/question/24869894/answer/97339151)
2. [Spark 2.0 Tuning Guide](http://www.slideshare.net/jcmia1/apache-spark-20-tuning-guide)
3. [Spark官方文档：Job Scheduling](https://spark.apache.org/docs/latest/job-scheduling.html)
4. [Mastering Apache Spark 2](https://jaceklaskowski.gitbooks.io/mastering-apache-spark/)
5. [Spark调度（一）：Task调度算法，FIFO还是FAIR](http://www.datastart.cn/tech/2016/07/11/spark-scheduler.html)
6. https://github.com/antlr/antlr4/blob/master/doc/tree-matching.md
7. https://mr-dai.github.io/spark.html
8. [Deep Dive into Spark SQL’s Catalyst Optimizer](https://databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html)
9. [盛利：Spark SQL 源码分析系列文章](http://blog.csdn.net/oopsoom/article/details/38257749)
10. [Spark性能优化指南——基础篇](http://tech.meituan.com/spark-tuning-basic.html)
11. [美团点评技术团队：Spark性能优化指南——高级篇](https://zhuanlan.zhihu.com/p/22024169)
