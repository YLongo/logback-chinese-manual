Joran 代表寒冷的西北风，常常猛烈的吹在日列瓦湖上。位于西欧中部的日列瓦湖，表面上看起来比其它许多欧洲的湖泊都要小。但是它的平均深度有 153 米，异常的深。并且，它是西欧最大的淡水湖。

正如前几章所示，logback 基于 Joran，一个成熟的，灵活的并且强大的配置框架。logback 提供的许多的功能，只能基于 Joran 来实现。这章将专注于 Joran 的基本设计以及一些明显的特征。

Joran 实际上是一个通用的配置系统，能够被独立用于日志记录。为了强调这一点，我们需要说明的是 logback-core 模块没有 logger 的概念。所以，本章的大多数示例与 logger，appender，layout 无关。

本章节中的示例可以在 *LOGBACK_HOME/logback-examples/src/main/java/chapters/onJoran/* 文件夹下被找到。

要安装 Joran，只需要[下载](https://logback.qos.ch/download.html)，然后将 *logback-core-1.3.0-alpha4.jar* 放到类路径下。

## 历史回顾

反射是 Java 语言一个强大的特性，使得声明式的配置软件系统变成可能。例如，EJB 许多重要的属性都被配置在 *ejb.xml* 文件中。尽管 EJB 是用 Java 编写的，但是它们的许多属性都是通过  *ejb.xml* 来指定的。类似的，logback 也可以通过指定的 XML 格式的配置文件来进行设置。JDK 1.5 中的注解在 EJB 3.0 被大量使用用来替换之前 XML 文件中的许多指令。Joran 也会充分利用注解，但是使用的范围少的多。由于 logback 配置的动态特性 (相比 EJB)，Joran 使用注解相当有限。

在 logback 它爹 log4j 中， `DOMConfigurator` 类是 log4j 1.2.x 以及以后的版本的一部分。也能够解析 XML 的配置文件。`DOMConfigurator` 的编写方式强迫开发人员在配置文件的结构每次发生改变时，需要重新调整代码。调整的代码需要重新编译并重新部署。同样重要的是，`DOMConfigurator` 的代码由循环组成，用于解析子元素，包含了许多 if/else 语句。这样的代码散发着冗余以及重复的味道。 [commons-digester](http://jakarta.apache.org/commons/digester/) 告诉我们可以通过模式匹配规则来解析 XML 文件。在解析的时候，解析器会应用匹配了指定模式的规则。规则类通常比较小，并且具有专业性。因此，理解与维护相对简单。

有了 `DOMConfigurator` 的经验，我们开始开发 `Joran`，一个在 logback 中使用的、强大的配置框架。Joran 受到了 commons-digester 项目很大的启发。但是，它使用了一个稍微不同术语。在 commons-digester 中，规则可以看做由模式和规则组成，如同 `Digester.addRule(String pattern, Rule rule)` 方法展示的一样。我们发现一个不必要的困惑是规则包含自身，但是不是递归，而是有不同的含义。在 Joran 中，规则由模式以及动作组成。当相应的模式被匹配时，会调用一个动作。模式与动作的这种关系是 Joran 的核心。值得注意的是，可以使用简单的模式来处理复杂的匹配，或者更确切的是说是使用精确匹配以及通配符匹配。

### SAX 还是 DOM ?

由于 SAX API 是基于事件的结构，所以基于 SAX 的工具不能很好的处理前向引用，也就是引用元素被定义晚于当前元素被处理。循环引用元素也有同样的问题。通常，DOM API 允许用户在所有的元素上进行搜索，并且可以向前跳转。

这种额外的灵活性导致我们在开始的时候选择 DOM API 作为 Joran 的解析器。在经过了一些实验之后，我们发现当解析规则通过模式以及动作表达时，在解析 DOM 树时处理相隔较远的元素没有意义。*Joran 只需要 XML 文档中连续且深度优先顺序的元素。*

而且，在发生错误时，SAX API 提供元素的位置信息可以让 Joran 去展示精确的行号与列号。位置信息在识别解析问题时非常方便。

### 非目标

考虑到它的高度动态特性，Joran API 没有打算去解析包含几千个元素的 XML 文档。

### 模式 (Pattern)

Joran 的模式本质上就是一个字符串。有两种形式的模式：*exact* 与 *wildcard*。模式 "a/b" 可以用来匹配嵌套在 `<a>` 元素中的 `<b>` 元素。因为 *exact* 匹配的设置，"a/b" 模式不会匹配其它的元素。

wildcard 可以用来进行后缀与前缀匹配。例如，"\*/a" 可以用来匹配任何以 "a" 结尾的后缀，也就是 XML 文档中任何 `<a>` 元素，但是不包含任何嵌套在 `<a>` 中的元素。"a/\*"  将会匹配任何 `<a>` 开头的元素，但是不包括任何嵌套在 `<a>` 中的元素。

### 动作

正如之前提到的，Joran 解析规则由相关联的模式组成。动作继承了 [`Action`](https://logback.qos.ch/xref/ch/qos/logback/core/joran/action/Action.html) 类，包含了如下的抽象方法。为了简单起见，其它的方法被隐藏了。

```java
package ch.qos.logback.core.joran.action;

import org.xml.sax.Attributes;
import org.xml.sax.Locator;
import ch.qos.logback.core.joran.spi.InterpretationContext;

public abstract class Action extends ContextAwareBase {
  /**
   * 当解析器遇到一个元素匹配
   * {@link ch.qos.logback.core.joran.spi.Pattern Pattern}.
   */
  public abstract void begin(InterpretationContext ic, String name,
      Attributes attributes) throws ActionException;

  /**
   * 传递包含元素的 body (作为字符串) 参数
   */
  public void body(InterpretationContext ic, String body)
      throws ActionException {
    // NOP
  }

  /*
   * 当解析器遇到最后一个元素匹配
   * {@link ch.qos.logback.core.joran.spi.Pattern Pattern}.
   */
  public abstract void end(InterpretationContext ic, String name)
      throws ActionException;
}
```

所以，每个动作必须实现 `begin()` 与 `end()` 方法。`body()` 方法的实现是可选的，因为 `Action` 提供了一个空的实现。

### 规则存储

如前面提到的，根据匹配模式调用动作是 Joran 的核心概念。规则跟模式与动作相关联。规则被存储在 [RuleStore](https://logback.qos.ch/xref/ch/qos/logback/core/joran/spi/RuleStore.html) 中。

根据之前提到的，Joran 建立在 SAX API 上。当 XML 文档被解析时，每个元素会生成对应 start、body、end 的事件。当 Joran 的配置器接收到这些事件时，它会根据*当前模式*去规则存储中查找对应的动作。例如，元素 *B* 的 start、body、end 事件的当前模式为 "A/B"，内嵌在一个顶级元素 *A* 中。当 Joran 接收并处理 SAX 事件时，它会自动维护当前模式的数据结构。

当有几个规则匹配到当前模式时，精确匹配会比后缀匹配优先，后缀匹配会比前缀匹配优先。对于详细的实现细节，请查看 [SimpleRuleStore](https://logback.qos.ch/xref/ch/qos/logback/core/joran/spi/SimpleRuleStore.html) 类。

### 解析上下文



