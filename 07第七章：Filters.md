在之前的章节中介绍的[方法打印以及基本选择规则](https://github.com/Volong/logback-chinese-manual/blob/master/02%E7%AC%AC%E4%BA%8C%E7%AB%A0%EF%BC%9A%E6%9E%B6%E6%9E%84.md#%E6%96%B9%E6%B3%95%E6%89%93%E5%8D%B0%E4%BB%A5%E5%8F%8A%E5%9F%BA%E6%9C%AC%E9%80%89%E6%8B%A9%E8%A7%84%E5%88%99)是 logback-classic 的核心。在这章中，将介绍其它的过滤方法。

logback 过滤器基于三元逻辑，允许它们组装或者链接在一起组成一个任意复杂的过滤策略。它们在很大程度上受到 Linux iptables 的启发。

## 在 logback-classic 中

在 logback-classic 中，有两种类型的过滤器，regular 过滤器以及 turbo 过滤器。

### Regular 过滤器

reqular 过滤器继承自 [`Filter`](https://logback.qos.ch/xref/ch/qos/logback/core/filter/Filter.html) 这个抽象类。本质上它由一个单一的 `decide()` 方法组成，接收一个 `ILoggingEvent` 实例作为参数。

过滤器通过一个有序列表进行管理，并且基于三元逻辑。每个过滤器的 `decide(ILoggingEvent event)` 被依次调用。这个方法返回 [`FilterReply`](https://logback.qos.ch/xref/ch/qos/logback/core/spi/FilterReply.html) 枚举值中的一个， `DENY`, `NEUTRAL` 或者 `ACCEPT`。如果 `decide()` 方法返回 `DENY`，那么日志事件会被丢弃掉，并且不会考虑后续的过滤器。如果返回的值是 `NEUTRAL`，那么才会考虑后续的过滤器。如果没有其它的过滤器了，那么日志事件会被正常处理。如果返回值是 `ACCEPT`，那么会跳过剩下的过滤器而直接被处理。

在 logback-classic 中，过滤器可以被直接添加到 `Appender` 实例上。通过将一个或者多个过滤器添加到 appender 上，你可以通过任意标准来过滤日志事件。例如，日志消息的内容，MDC 的内容，时间，或者日志事件的其它部分。

### 实现你自己的过滤器

创建一个自己的过滤器非常的简单。只需要继承 `Filter` 并且实现 `decide()` 方法就可以了。

如下所示的 SampleFilter 就是一个简单的例子。如果日志事件包含字符 "sample"， `decide` 方法返回 ACCEPT。对于其他的日志事件，则返回 NEUTRAL。

下面是关于将 `SampleFilter` 添加到 `ConsoleAppender` 上的配置示例：

>   Example: *SampleFilterConfig.xml*

```xml
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">

    <filter class="chapters.filters.SampleFilter" />

    <encoder>
      <pattern>
        %-4relative [%thread] %-5level %logger - %msg%n
      </pattern>
    </encoder>
  </appender>
        
  <root>
    <appender-ref ref="STDOUT" />
  </root>
</configuration>
```

在 logback 配置框架 Joran 的帮助下，为过滤器指定属性或者子组件也变得更加的简单。在过滤器类中添加相应的 set 方法，通过 `<filter>` 元素嵌套一个以属性命名的 xml 元素中指定属性的值。

通常情况下，过滤器的逻辑由两个正交的部分组成，match/mismatch 的检验以及基于 match/mismatch 的返回值。例如，对于给定的检验，消息等于 "foobar"，一个过滤器在 match 的情况下返回 ACCEPT，在 mismatch 的情况下返回 NEUTRAL。另一个过滤可能在 match 的情况下返回 NEUTRAL，在 mismatch 的情况下返回 DENY。

注意这种正交，logback 附带了一个 [`AbstractMatcherFilter`](https://logback.qos.ch/xref/ch/qos/logback/core/filter/AbstractMatcherFilter.html) 类，提供了一个有用的骨架用来指定在 match 与 mismatch 情况下的返回值，这两个属性名分别叫做 *OnMatch* 与 *OnMismatch*。logback 中大部分的 regular 过滤器都源于 `AbstractMatcherFilter`。

### LevelFilter



