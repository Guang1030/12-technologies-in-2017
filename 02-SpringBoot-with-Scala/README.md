@springboot @scala
# Spring Boot with Scala
## 前言
本文提供一个示例项目，如果你觉得代码更加亲切，请移步至[Spring Boot with Scala](https://github.com/sadhen/spring-boot-with-scala)。

## Bootstrap
第一步，我们从零开始构建一个五脏俱全的工程。注意，是工程，而不是代码片段。通常来说，一个工程需要开发、构建、测试、部署、监控等环节。

### 开发
大多数程序员并不聪明也不勤奋。一般而言，从零开始写一个工程实际上非常吃力。所以很多框架都会提供脚手架工程。Spring Boot官网就提供了[start.spring.io](http://start.spring.io)，快速生成脚手架工程。可惜并没有提供Scala相关的脚手架工程。故而，本文的示例项目实际上也是一个脚手架项目。

本文将只采用Maven作为构建工具。

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
测试的话，主要是用spring-boot-starter-test, JUnit 和 ScalaTest。在Maven中声明这些依赖时需要指定scope为test，以表明这些依赖只对测试classpath有效。

### 从Assert开始
我们可以混用ScalaTest和JUnit，使用了ScalaTest并不意味着不使用JUnit。就像学习Scala，并不能放弃深入学习Java。而是在比较、揣摩两者的差异时，学习如何写出一手高质量的代码。

ScalaTest的assert是一个宏，可以抛出非常可读的Error Message：
``` scala
import org.scalatest.Assertions._
assert(a == b || c >= d)
// Error message: 1 did not equal 2, and 3 was not greater than or equal to 4
assert(xs.exists(_ == 4))
// Error message: List(1, 2, 3) did not contain 4
```

如果你也恰巧读过Clean Code，是否还记得函数那一章讲到没有参数的函数是最好的，一个参数的函数不复杂，两个参数的函数就需要程序员在时候的时候注意参数的顺序了。三个及以上参数的函数就不太妙了。即使Intellij如此智能，程序员还是很容易犯错。至少，你在使用assertEquals的时候，每一次都需要等IDE的提示出来才能愉快自信的了解的每个参数的真正含义。

### 有依赖注入的类怎么测试
很简单:
```
@RunWith(classOf[SpringRunner])
@SpringBootTest
class SampleTest {
  @Autowired
  var sampleService: SampleService = _
  
  def testSampleService = ???
}
```

下面这个例子演示了如何测试Rest Controller，其实也很简单，主要是利用了spring-boot-starter-test里面提供的TestRestTemplate。其中有些json4s的语法或许你没有接触过，且看下文。
``` scala
@RunWith(classOf[SpringRunner])
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class HelloControllerTest {
  @Autowired
  var restTemplate: TestRestTemplate = _

  @Test
  def testHello(): Unit = {
    val body = restTemplate.getForObject("/api/hello", classOf[JsonNode])
    val expected = ("code" -> 0) ~
      ("data" -> ("hello" -> "中国") ~ ("year" -> 2017)) ~
      ("error" -> JNull)
    assert {
      fromJsonNode(body) == expected
    }
  }
}
```

但是这种测试有个弊端，由于需要初始化上下文，每次都需要等上好长一段时间。

### ScalaMock
ScalaMock就是用来解决上文提到的问题的。看代码：

``` scala
class HelloControllerSpec extends FlatSpec with Matchers with MockFactory {

  "/api/hello" should "be ok" in {
    val worldService = stub[WorldService]
    (worldService.getCountry _).when().returns("法国")

    val helloController = new HelloController(worldService)

    val expected = ("code" -> 0) ~
      ("data" -> ("hello" -> "法国") ~ ("year" -> 2017)) ~
      ("error" -> JNull)

    assert {
      fromJsonNode(helloController.hello) == expected
    }
  }
}
```
这个例子中，我们mock了一个WorldService，通过指定getCountry方法的返回值定义了worldService的行为模式。从而不需要初始化上下文就可以完成Rest Controller的测试。

### 后记
这里面还有很多话题没有提及，比如异步的单元测试等等。为什么单元测试那么重要呢，因为实际上单元测试就是第一手的最准确的文档。如果你要用一个开源的库，恰好他的文档写得不够详细，那么多数情况下你都可以从单元测试中获得你的答案。如果遍历它的单元测试都没有找到库的正确用法，或许是你的水平没有到家，更可能的是这个库写得并不好，建议**不要**去使用它。

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
使用ScalaStyle和scalafmt。scalafmt有IntelliJ的插件。如果是在公司，可以使用SonarQube配置一套团队公用的ScalaStyle配置。

### Utilities
这里简单谈一谈对一些工具库的选择。基本上我都会选择那些基于久经考验的相关Java库的封装。这些库一般都会提供一些Scala语言特性上的适配，然后提供一些比较友好的DSL。那么为什么不选择pure scala呢？通常情况下，那些pure scala的库会重度依赖Akka，Scalaz等著名的库，由于很多是新造的轮子，并没有经历时间的考验，其实非常buggy。如果你使用它们，你就得做好撸起袖管fork的准备。

#### Mybatis
因为大家都习惯用druid和mybatis的组合。所以这里我选择用mybatis。其实slick也非常好用，只不过没有和Spring Boot的集成。写Java的话，大家习惯用lombok，在Scala里面没法用。我们可以用@BeanProperty这个注释做到类似的效果(可惜没法用case class)。

``` scala
class SQLStatDO {
  @BeanProperty var id: Long = _
  @BeanProperty var user: String = _
  @BeanProperty var age: Int = _
  @BeanProperty var sex: String = _
}
```
Mybatis的Scala支持好久没有更新了，所以我不用。

#### json4s(JSON)
推荐使用json4s的jackson support。用好json4s，最好了解一下Scala的模式匹配和隐式转换这两个语言特性。

为什么选择json4s呢，因为json4s提供了非常友好的DSL。

在构建Rest Controller时，由于Scala的集合和case class不支持直接序列化，我们可以引入
``` xml
<dependency>
  <groupId>com.fasterxml.jackson.module</groupId>
  <artifactId>jackson-module-scala_${scala.compat.version}</artifactId>
  <version>2.8.7</version>
</dependency>
```
做一些配置:
``` scala
@Configuration
@EnableWebMvc
class WebConfig extends WebMvcConfigurerAdapter {
  override def configureMessageConverters(converters: util.List[HttpMessageConverter[_]]): Unit =
    converters.add(jackson2HttpMessageConverter())

  @Bean
  def jackson2HttpMessageConverter(): MappingJackson2HttpMessageConverter =
    new MappingJackson2HttpMessageConverter(objectMapper())

  @Bean
  def objectMapper(): ObjectMapper =
    new ObjectMapper() {
      setVisibility(PropertyAccessor.FIELD, Visibility.ANY)
      registerModule(DefaultScalaModule)
    }
}
```

配置完成之后，我们直接在Rest Controller里面返回普通的Scala对象就可以由jackson将其序列化。另外一种情况是我们自己构造的JValue，则需要转换成JsonNode才能被正确地序列化。对于Http Post中的值的解析，这里也简单举个例子。

``` scala
@RestController
@RequestMapping(value = Array("/api"))
class HelloController {
  implicit def jvalue2jsonnode(value: JValue): JsonNode = asJsonNode(value)
  @RequestMapping(value = Array("/hello"))
  def hello: JsonNode = {
    val world = "世界"
    val ret =
      ("code" -> 0) ~
        ("data" -> ("hello" -> world) ~ ("year" -> 2017)) ~
        ("error" -> null)

    asJsonNode(ret)
  }

  @RequestMapping(value = Array("/echo"), method = Array(RequestMethod.POST))
  def echo(@RequestBody body: JsonNode): JsonNode = {
    val json = fromJsonNode(body)
    (json \ "hello", json \ "year") match {
      case (JString(world), JInt(year)) =>
        val ret =
          ("code" -> 0) ~
            ("data" -> ("hello" -> world) ~ ("year" -> year)) ~
            ("error" -> null)
        asJsonNode(ret)
      case _ =>
        val ret =
          ("code" -> 1) ~
            ("data" -> null) ~
            ("error" -> "invalid post body")
        asJsonNode(ret)
    }
  }
}
```
用curl做一下简单测试:
``` bash
➜  ~ curl -d '{"hello": "世界", "year": 2017}' -H "Content-Type: application/json" -X POST  http://localhost:8080/api/echo
{"code":0,"data":{"hello":"世界","year":2017},"error":null}
➜  ~ curl http://localhost:8080/api/hello
{"code":0,"data":{"hello":"世界","year":2017},"error":null}
```

这里asJsonNode比较繁琐，可以用隐式转换让代码更加简洁。
``` scala
@RestController
@RequestMapping(value = Array("/api"))
class HelloController {
  implicit def autoAsJsonNode(value: JValue): JsonNode = asJsonNode(value)

  @RequestMapping(value = Array("/hello"))
  def hello: JsonNode = {
    val world: String = "世界"

    ("code" -> 0) ~
      ("data" -> ("hello" -> world) ~ ("year" -> 2017)) ~
      ("error" -> null)
  }

  @RequestMapping(value = Array("/echo"), method = Array(RequestMethod.POST))
  def echo(@RequestBody body: JsonNode): JsonNode = {
    val json = fromJsonNode(body)
    (json \ "hello", json \ "year") match {
      case (JString(world), JInt(year)) =>
        ("code" -> 0) ~
          ("data" -> ("hello" -> world) ~ ("year" -> year)) ~
          ("error" -> null)
      case _ =>
        ("code" -> 1) ~
          ("data" -> null) ~
          ("error" -> "invalid post body")
    }
  }
}
```

#### gigahorse(HTTP Client)
之前写过[http4s client的学习笔记](http://sadhen.com/blog/2016/11/27/http4s-client-intro.html)，因为官网的文档语焉不详，所以翻看了测试用例才知道http4s client怎么用。

当然，如果只是翻一下测试用例就能愉快的使用，倒是很好，只不过后来在用http4s的时候碰到一个HTTP 1.1的chunked响应相关的一个坑。鼓捣了很久发现搞不定。而且，http4s默认返回的结果是在scalaz的Task里面的，并不是Scala标准库里面的Future。scalaz又在能力范围之外，所以弃用。

其实，就我的需求很简单：
1. 这个Http Client支持的方法要完整，比如scalaj-http就不支持PATCH
2. 使用足够简单，直观，不需要在使用时引入AKKA。play-ws就是一个反例。
3. 支持返回Future
4. 在定义URI的时候能够使用直观的DSL，避免字符串拼接

gigahorse满足前三个条件，至于URI的DSL，用scala-uri解决。gigahorse背后是著名的AsyncHttpClient，其实现会比http4s完整很多，不至于会遇到各种bug。

在使用Http Client的时候会涉及到Client的生命周期管理，一般在SpringBoot中，我们可以实现DisposableBean中的方法，在对象销毁的时候关闭Client。

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
12. [使用用Spring Boot Actuator、Jolokia和Grafana实现准实时监控](http://blog.didispace.com/spring-boot-jolokia-grafana-monitor/)

## Related Project
+ https://github.com/NET-A-PORTER/scala-uri
+ https://github.com/eed3si9n/gigahorse
+ https://github.com/json4s/json4s
+ https://github.com/Log4s/log4s
+ https://github.com/scalatest/scalatest
+ https://github.com/paulbutcher/ScalaMock
+ https://github.com/scalameta/scalafmt
+ https://github.com/scalastyle/scalastyle
+ https://github.com/Sagacify/sonar-scala
+ https://github.com/FasterXML/jackson-module-scala
