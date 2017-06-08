# json4s 简明指南
## Ammonite 简明指南
``` bash
curl -L -o $HOME/bin/amm https://git.io/vHr1V && chmod +x $HOME/bin/amm
```

`$HOME/bin/amm` 可以启动 REPL，下文的代码均可以在 amm 上跑。如果要引入依赖，可以在 https://index.scala-lang.org 找。

## 我选择 Jackson
## JSON String, JValue, Scala对象
### JSON String -> JValue
``` scala
import $ivy.`org.json4s::json4s-jackson:3.5.2`
import org.json4s.jackson.JsonMethods.parse
// 正常的例子
parse(""" { "numbers" : [1, 2, 3, 4] } """)
// null
parse(""" { "code": 0, "error": null } """)
```

### JValue & Scala对象 -> JSON String
``` scala
import $ivy.`org.json4s::json4s-jackson:3.5.2`
import org.json4s.jackson.JsonMethods.render
import org.json4s.jackson.JsonMethods.compact
import org.json4s.jackson.JsonMethods.pretty
import org.json4s.JsonDSL._

// primitive type 会被隐式转换成 JValue，比如这里的 List(1, 2, 3) 就是 JArray(List(JInt(1), JInt(2), JInt(3)))
compact(render(List(1, 2, 3)))

// JValue
val joe = ("name" -> "joe") ~ ("age" -> Some(35))
pretty(render(joe))
```

### JValue -> Scala对象
``` scala
import $ivy.`org.json4s::json4s-jackson:3.5.2`
import org.json4s._
import org.json4s.JsonDSL._

implicit val formats = DefaultFormats

case class Person(name: String, age: Option[Int])
val joe = ("name" -> "joe") ~ ("age" -> Some(35))
joe.extract[Person]
```

### Scala对象 -> JValue
``` scala
import $ivy.`org.json4s::json4s-jackson:3.5.2`
import org.json4s.Extraction
implicit val formats = DefaultFormats

case class Person(name: String, age: Option[Int])
Extraction.decompose(Person("joe", Some(11)))
```

### 序列化
``` scala
import $ivy.`org.json4s::json4s-jackson:3.5.2`
import org.json4s._
import org.json4s.jackson.Serialization
import org.json4s.jackson.Serialization.{read, write}
implicit val formats = Serialization.formats(NoTypeHints)

case class Person(firstName: String)
write(Person("joe"))

read[Person](""" {"firstName": "joe"} """)

case class Status(code: Int, error: String)
read[Status](""" {"code": 0, "error": null} """)
```

## Option 和 Either
+ Option可以用来表示可有可无的字段。Option是合理的。
+ Either可以用来表示恶心的需求。使用了Either说明api定义不规范。

## Play with JValue
### Snakize and Camelize
``` scala
import $ivy.`org.json4s::json4s-jackson:3.5.2`
import org.json4s.JsonDSL._

val camelJoe = ("firstName" -> "joe") ~ ("age" -> 23)
camelJoe.snakizeKeys
val snakeJoe = ("first_name" -> "joe") ~ ("age" -> 23)
snakeJoe.camelizeKeys
```

在序列化和反序列化的过程中，如果字段名的风格不一致，就需要通过JValue这种中间状态转换字段名的风格。

### XPath
如果只需要JSON中的部分字段，那么就用XPath和Pattern Matching，如果所有的字段都需要用到，那就用extract to case class的方式。

而for-comprehension的可读性比较差，不推荐使用。在这种场景下，for-comprehension的语义被复杂化了，个人觉得是易用性上的缺陷。如果你要用for-comprehension，你必须清楚地知道整个ast的结构，知道每个`<-`的不同含义。
