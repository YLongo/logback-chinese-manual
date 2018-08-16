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

[`LevelFilter`](https://logback.qos.ch/xref/ch/qos/logback/classic/filter/LevelFilter.html) 基于级别来过滤日志事件。如果事件的级别与配置的级别相等，过滤器会根据配置的 `onMatch` 与 `onMismatch` 属性，接受或者拒绝事件。如下是一个简单的示例：

>   Example: *levelFilterConfig.xml*

```xml
<configuration>
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <filter class="ch.qos.logback.classic.filter.LevelFilter">
      <level>INFO</level>
      <onMatch>ACCEPT</onMatch>
      <onMismatch>DENY</onMismatch>
    </filter>
    <encoder>
      <pattern>
        %-4relative [%thread] %-5level %logger{30} - %msg%n
      </pattern>
    </encoder>
  </appender>
  <root level="DEBUG">
    <appender-ref ref="CONSOLE" />
  </root>
</configuration>
```

### ThresholdFilter

[`ThresholdFilter`](https://logback.qos.ch/xref/ch/qos/logback/classic/filter/ThresholdFilter.html) 基于给定的临界值来过滤事件。如果事件的级别等于或高于给定的临界值，当调用 `decide()` 时，`ThresholdFilter` 将会返回 NEUTRAL。但是事件的级别低于临界值将会被拒绝。下面是一个简单的例子：

>   Example: *thresholdFilterConfig.xml*

```xml
<configuration>
  <appender name="CONSOLE"
    class="ch.qos.logback.core.ConsoleAppender">
    <!-- deny all events with a level below INFO, that is TRACE and DEBUG -->
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
      <level>INFO</level>
    </filter>
    <encoder>
      <pattern>
        %-4relative [%thread] %-5level %logger{30} - %msg%n
      </pattern>
    </encoder>
  </appender>
  <root level="DEBUG">
    <appender-ref ref="CONSOLE" />
  </root>
</configuration>
```

## EvaluatorFilter

[`EvaluatorFilter`](https://logback.qos.ch/xref/ch/qos/logback/core/filter/EvaluatorFilter.html) 是一个通用的过滤器，它封装了一个 `EventEvaluator`。顾名思义，[`EventEvaluator`](https://logback.qos.ch/xref/ch/qos/logback/core/boolex/EventEvaluator.html) 根据给定的标准来评估给定的事件是否符合标准。在 match 和 mismatch 的情况下，`EvaluatorFilter` 将会返回 `onMatch` 或 `onMismatch` 指定的值。

注意 `EventEvaluator` 是一个抽象类。你可以通过继承 `EventEvaluator` 来实现自己事件评估逻辑。

### GEventEvaluator

[GEventEvaluator](https://logback.qos.ch/xref/ch/qos/logback/classic/boolex/GEventEvaluator.html) 是 [`EventEvaluator`](https://logback.qos.ch/xref/ch/qos/logback/core/boolex/EventEvaluator.html) 具体的实现，它采用 Groovy 表达式作为评估的标准。我们把 Groovy 表达式称为 "Groovy 评估表达式"。Groogy 评估表达式是目前为止进行事件过滤最灵活的方式。`GEventEvaluator` 需要 Groovy 运行环境。参考[相关部分](https://logback.qos.ch/setup.html#groovy)在类路径下添加 Groovy 运行环境。

评估表达式在解析配置文件期间被动态编译。作为用户，不需要考虑实际的情况。但是，你需要确保你的 Groovy 表达式是有效的。

评估表达式作用于当前的日志事件。logback 会自动将 [ILoggingEvent](https://logback.qos.ch/apidocs/ch/qos/logback/classic/spi/ILoggingEvent.html) 类型的日志事件作为变量插入，引用到 'event' 或者它的简称 'e'。TRACE, DEBUG, INFO, WARN 以及 ERROR 也能够被导入到表达式的范围中。所以，"event.level == DEBUG" 与 "e.level == DEBUG" 是等价的。只有当当前日志事件的级别为 DEBUG 时，Groovy 表达式才会返回 `true`。对于其它的级别比较操作，应该通过 `toInt()` 操作将 level 字段转变为整型。

下面是一个比较复杂的例子：

```xml

```







