# Spring Boot with Scala
## 前言
本文提供一个示例项目，如果你觉得代码更加亲切，请移步至[Spring Boot with Scala](https://github.com/sadhen/spring-boot-with-scala)。

## Bootstrap
第一步，我们从零开始构建一个五脏俱全的工程。注意，是工程，而不是代码片段。通常来说，一个工程需要开发、构建、测试、部署、监控等环节。

### 开发
大多数程序员并不聪明也不勤奋。一般而言，从零开始写一个工程实际上非常吃力。所以很多框架都会提供脚手架工程。Spring Boot官网就提供了[start.spring.io](http://start.spring.io)，快速生成脚手架工程。可惜并没有提供Scala相关的脚手架工程。故而，本文的示例项目实际上也是一个脚手架项目。

本文将同时采用Maven和SBT两套构建工具。

### 构建与部署
本节主要讨论构建工具。在阅读清单2中已经有比较详细的介绍了，这里讨论一些值得注意的细节和改进。

使用`spring-boot-maven-plugin`打包的话实际上会生成一个executable jar，我们执行`java -jar target/spring-boot-with-scala-1.0.jar`便可以运行。然而这并不美好。当我们在服务器上运行这个jar时，最好能有`start.sh`和`stop.sh`脚本管理程序的启动和中止。我们把事先编写好的启动和中止脚本放在bin目录下，然后用`maven-jar-plugin`，`maven-dependency-plugin`和`maven-assembly-plugin`生成`spring-boot-with-scala-1.0-bin.tar.gz`，然后部署到服务器上。

`start.sh`脚本中提供了一种覆盖jar中的application.properites的方法，即使用`spring.config.location`，在本文的项目中将这个配置文件的默认地址设置为`conf/application.properties`，如果没有这个文件，就使用jar中的配置。


### 日志
引入`spring-boot-start-logging`后，Spring Boot会使用slf4j-api和logback作为应用日志框架。Java的日志系统非常混乱，建议阅读材料10，理一下思路。

slf4j很好地解决了日志的性能问题。在处理日志时，我们希望字符串的拼接是lazy的。使用Java 8可以这样解决问题：`logger.debug(() -> "hello " + getValue())`。然而略显啰嗦，slf4j提供了这种方式：`logger.info("hello {}", name)`，不失其优雅。

因为我们是用Scala，所以推荐使用[log4s](https://github.com/Log4s/log4s)。这样就可以愉快地使用Scala的字符串插值特性，而不失其性能。如官网所言：

> Log4s goes even further in that it uses macros to manipulate the execution so that the string interpolations are not even performed unless the logger is enabled. It does this by inspecting the structure of the argument that you pass into the logger.

材料4讲解了Spring1.5.x动态修改日志级别的新特性，本文的示例工程中提供了`loglevel.sh`脚本：

``` bash
bin/loglevel.sh com.sadhen  # 显示package com.sadhen的日志级别
bin/loglevel.sh com.sadhen DEBUG # 将package com.sadhen的日志级别设置为DEBUG
```


## TDD
## Scala
前面讲的比较多还是Spring Boot本身，那么为什么要Scala呢？已经有很多比较Java和Scala文章了，这里不赘述。阅读清单中3和6都值得一看。下面简单谈一谈那些尤为重要的Scala特性。

### 惰性求值
在程序语言理论中，**惰性求值**是一种求值策略，它将表达式的求值计算延迟到实际用到这个值的时刻，以避免重复计算（翻译自维基百科，见阅读清单8）。

用Java举个例子：

``` java
class LazyDemo {
  String lazyString = null;

  private String initialize() {
    // 比较耗时的初始化
    String result = ...
    return result;
  }

  void fun1() {
    // 一些和lazyString无关的代码
    if (lazyString == null) {
        lazyString = initialize()
    }
    // 下面开始需要使用lazyString
  }

  void fun2() {
    // 一些和lazyString无关的代码
    if (lazyString == null) {
        lazyString = initialize(lazyString)
    }
    // 下面开始需要使用lazyString
  }
}
```
维基百科里面提到惰性求值的优点有三：

1. 提供了控制结构抽象化的能力
2. 提供了定义无穷数据结构的能力
3. 在对复合表达式的求值过程中避免了无谓的计算和错误处理

对于1，实际上Scala中的`Try`和`future`是非常直观的例子，我们也可以自定义一些参数是代码块的函数，由于Scala允许最后一个参数体的`()`改写成`{}`，形式上其实非常有美感。对于2，在Scala中体现在Stream这个集合中。另外，`lazy`关键字也是至关重要的。比如下面这个例子：

``` java
// Java Code
Long ret = null;
if (noNeedToCalculateResult) {
  // branch 1
} else {
  ret = calculate();
  if (ret == -1) {
    // branch 2
  } else {
    // branch 3
  }
}

// Scala Code
lazy val ret = calculate()
if (noNeedToCalculateResult) {
  // branch 1
} else if (ret == -1) {
  // branch 2
} else {
  // branch 3
}
```
在Scala中使用lazy，便可以减少`if-else`分支结构的层次，使代码逻辑更加清晰可读。对于`LazyDemo`，改写成Scala，也简化了许多：

``` scala
class LazyDemo {
  lazy val lazyString: String = initialize

  def initialize: String  = ???

  def fun1() {
    // 一些和lazyString无关的代码
    // 下面开始需要使用lazyString
  }

  def fun2() {
    // 一些和lazyString无关的代码
    // 下面开始需要使用lazyString
  }
}
```
最初用于演示延迟计算的例子中，如果fun1和fun2并发执行，会带来严重的问题。涉及到双重检测锁(DCL)，请参考阅读清单7和9。

### 代码风格

## Reading List
1. [Building "Bootiful" Scala Web Applications with Spring Boot](https://github.com/shekhargulati/52-technologies-in-2016/tree/master/37-spring-boot-scala)
2. [Scala开发者的SpringBoot快速入门指南](http://afoo.me/posts/2015-07-21-scala-developers-springboot-guide.html)
3. [Scala with a human face](http://dimafeng.com/2016/01/02/scala-spring/)
4. [Spring Boot 1.5.x新特性：动态修改日志级别](http://blog.didispace.com/spring-boot-1-5-x-feature-1/)
5. [Spring Boot Actuator监控端点小结](http://blog.didispace.com/spring-boot-actuator-1/)
6. [Scala: The Good Parts by 扶墙老师](http://vdisk.weibo.com/s/BbtIfGVUtj4-)
7. [lazy变量与双重检测锁(DCL)](http://hongjiang.info/scala-lazy-and-dcl/)
8. [Lazy Evaluation](https://en.wikipedia.org/wiki/Lazy_evaluation)
9. [双重检查锁定与延迟初始化](http://www.infoq.com/cn/articles/double-checked-locking-with-delay-initialization)
10. [Java日志系统详解](http://ieye.iteye.com/blog/1924215)
