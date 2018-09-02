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

wildcard 可以用来进行后缀与前缀匹配。例如，"\*/a" 可以用来匹配任何以 "a" 结尾的后缀，也就是 XML 文档中任何 `<a>` 元素，但是不包含任何嵌套在 `<a>` 中的元素。"a/\*"  将会匹配任何 `<a>` 开头的元素，即任何嵌套在 `<a>` 中的元素。

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

为了允许多个动作相互协作，在调用 begin 与 end 方法时会包含解析上下文，作为第一个参数传递。解析上下文包含对象栈，对象映射，错误列表以及 Joran 调用动作时的一个引用。请查看 [`InterpretationContext`](https://logback.qos.ch/xref/ch/qos/logback/core/joran/spi/InterpretationContext.html) 类中详细的字段列表。

动作可以通过对象栈获取，入栈，出栈操作，或者通过对象映射来放置、获取 key 来进行协作。动作可以在解析上下文的 `StatusManager` 上通过添加错误项来报告任何错误条件。

### Hello world

这个章节中的第一个例子将会展示使用 Joran 所需要最小条件。这个例子由一个名为 [`HelloWorldAction`](https://logback.qos.ch/xref/chapters/onJoran/helloWorld/HelloWorldAction.html) 的动作组成。在调用它的 `begin()` 方法时会在控制台打印 "Hello World"。配置文件由解析器负责解析。为了实现本章的目的，我们实现了一个非常的简单的配置器 [`SimpleConfigurator`](https://logback.qos.ch/xref/chapters/onJoran/SimpleConfigurator.html)。[`HelloWorld`](https://logback.qos.ch/xref/chapters/onJoran/helloWorld/HelloWorld.html) 应用会将下面这些结合在一起：

-   创建一个规则与 `Context` 的映射
-   创建一个与 *hello-world* 模式相关的解析规则，以及对应的 `HelloWorldAction` 实例
-   创建一个 `SimpleConfigutator`，解析之前提到的规则映射。
-   调用配置器的 `doConfigure` 方法，解析 XML 文件
-   最后，将会收集上下文中的所有转态信息。如果有的话，将会打印

*hello.xml* 包含一个 \<hello-world\> 元素，没有任何其它的内置元素。详细的内容请查看 *logback-examples/src/main/java/chapters/onJoran/helloWorld/* 文件夹中的内容。

通过 *hello.xml* 运行 HelloWorld 应用将会在控制台输出 "Hello World"。

```java
java chapters.onJoran.helloWorld.HelloWorld src/main/java/chapters/onJoran/helloWorld/hello.xml
```

强烈推荐你在规则存储中添加新的规则，更改 XML 配置 (hello.xml)，以及添加新的动作。

### 动作相互合作

*logback-examples/src/main/java/joran/calculator/* 文件夹包含了几个动作，为了完成简单的计算，它们通过共同的对象栈相互合作。

*calculator1.xml* 文件包含一个 `computation` 元素，内嵌了一个 `literal` 元素。如下：

>   Example: calculator1.xml

```xml
<computation name="total">
  <literal value="3"/>
</computation>
```

在应用 [`Calculator1`](https://logback.qos.ch/xref/chapters/onJoran/calculator/Calculator1.html) 中，我们声明了各种解析规则 (模式与动作) 基于 XML 文档的内容一起合作来计算一个结果。

通过 *calculator1.xml* 运行 `Calculator`：

```jav
java chapters.onJoran.calculator.Calculator1 src/main/java/chapters/onJoran/calculator/calculator1.xml
```

将会输出：

```java
The computation named [total] resulted in the value 3
```

解析 *calculator1.xml* 文档包含如下的步骤：

-   开始事件对应的 \<computation\>  元素转换为当前模式 "/computation"。因为在  [`Calculator1`](https://logback.qos.ch/xref/chapters/onJoran/calculator/Calculator1.html) 中我们为 "/computation" 模式关联了一个 [`ComputationAction1`](https://logback.qos.ch/xref/chapters/onJoran/calculator/ComputationAction1.html) 实例。`ComputationAction1` 实例中的 `begin()` 方法将会被调用
-   开始事件对应的 \<litera\> 元素转换为当前模式 "/computation/literal"。为 "/computation/literal" 关联了一个 [`LiteralAction`](https://logback.qos.ch/xref/chapters/onJoran/calculator/LiteralAction.html) 实例。`LiteralAction` 实例中的 `begin()` 方法将会被调用。
-   同样的，结束事件对应的 \<computation\> 元素将会触发 `ComputationAction1` 实例中的 `end()` 方法的调用。

有意思的是动作相互合作的方式。`LiteralAction` 读取到一个字面值，并将其放到对象栈中，由 `InterpretationContext` 来维护。一旦完成，其它的动作可以获取该值或者对其进行更改。`ComputationAction1` 类的 `end()` 方法从栈中获取值，并打印。

下一个例子中， *calculator2.xml* 有点复杂，但是更加有趣。

>   有趣个鸡儿

>   Example：*calculator2.xml*

```xml
<computation name="toto">
  <literal value="7"/>
  <literal value="3"/>
  <add/>
  <literal value="3"/>
  <multiply/>
</computation>
```

在之前的例子中，为了响应 \<literal\> 元素，[`LiteralAction`](https://logback.qos.ch/xref/chapters/onJoran/calculator/LiteralAction.html) 实例会将 value 属性对应的整数放到解析上下文中的栈顶。在这个例子中，也就是 *calculator2.xml*，这个值是 7 与 3。为了响应 \<add\> 元素。[`AddAction`](https://logback.qos.ch/xref/chapters/onJoran/calculator/AddAction.html) 实例将会获取之前放进去的两个整数，计算它们的和，然后再放进去。如，在解析上下文栈顶的就是 10 (= 7 + 3)。下个 literal 元素将会让 LiteralAction 将会将整数 3 放入栈顶。为了响应 \<multiply\> 元素，[`MultiplyAction`](https://logback.qos.ch/xref/chapters/onJoran/calculator/MultiplyAction.html) 将会获取之前放入的两个整数 10 与 3，然后计算它们的乘积。它会将结果 30 放入栈顶。最后，为了响应结束事件对应的 \</computation\> 标签，ComputationAction1 将会打印堆栈顶部的结果，因此，运行：

```java
java chapters.onJoran.calculator.Calculator1 src/main/java/chapters/onJoran/calculator/calculator2.xml
```

将会输出：

```java
The computation named [toto] resulted in the value 30
```

### 默认动作

目前定义的队则都是被显示的动作调用。因为当前元素相关的模式/动作能够在规则存储中被找到。但是，在高度可以扩展的系统中，组件的数量跟类型都会非常多，因此对所有的模式都关联一个具体的动作将会变得十分的蛋疼。

同时，甚至在高扩展的系统中，可以看到重复的规则关联了不同的部分。如果我们可以识别这些规则，那么我们就可以在编译时处理由子组件组成的未知组件。例如，Apache Ant 有能力在编译时处理包含未知标签的任务，仅仅通过检查组件中的方法名是不是以 *add* 开头，像 `addFile` 或者 `addClassPath` 之类的。当 Ant 在任务内遇到一个内置的标签，它仅仅实例化一个匹配了任务类 add 方法的签名的对象，并且将结果对象附加到父级上。

Joran 通过默认动作的形式来提供类似的功能。Joran 保留了一系列的默认动作，如果当前模式没有具体的模式可以匹配时，它们将会被应用。但是，应用默认的动作可能并不总是合适的。在执行默认的动作之前，Joran 会询问指定的动作当前的情况是否合适。只有动作返回肯定的回答，Joran 的配置器才会调用默认的动作。注意，这个额外的步骤可能支持多个默认的动作，如果在给定的情况下，没有合适的默认动作，也可能一个都不支持。

你可以创建并注册一个自定义的默认动作。见下一个示例。该示例位于 *logback-examples/src/main/java/chapters/onJoran/implicit* 文件夹下。

[`PrintMe`](https://logback.qos.ch/xref/chapters/onJoran/implicit/PrintMe.html) 应用将一个 [`NOPAction`](https://logback.qos.ch/xref/chapters/onJoran/implicit/NOPAction.html) 实例与 "\*/foo" 模式相关联，也就是与名字叫做 "foo" 的任何元素。正如它的名字所示， `NOPAction` 的 `begin()` 与 `end()` 方法都为空。`PrintMe` 应用仍然会在它的默认动作列表注册一个 [PrintMeImplicitAction](https://logback.qos.ch/xref/chapters/onJoran/implicit/PrintMeImplicitAction.html) 的实例。`PrintMeImplicitAction` 对任何 *printme* 属性为 true 的元素有效。参见 `PrintMeImplicitAction` 的 `isApplicable()` 方法。`PrintMeImplicitAction` 的 `begin()` 方法会在控制台打印当前元素的名字。

*implicit1.xml* XML 文档说明了默认动作是如何起作用的。

> Example: *implicit1.xml* 

```xml
<foo>
  <xyz printme="true">
    <abc printme="true"/>
  </xyz>

  <xyz/>

  <foo printme="true"/>

</foo>
```

运行：

```java
java chapters.onJoran.implicit.PrintMe src/main/java/chapters/onJoran/implicit/implicit1.xml
```

输出：

```java
Element [xyz] asked to be printed.
Element [abc] asked to be printed.
20:33:43,750 |-ERROR in c.q.l.c.joran.spi.Interpreter@10:9 - no applicable action for [xyz], current pattern is [[foo][xyz]]
```

给定一个 `NOPAction` 实例与 "*\/foo*" 实例相关联，`NOPAction` 的 `begin()` 与 `end()` 方法在 \<foo\> 元素上被调用。`PrintMeImplicitAction` 不会在任何 \<foo\> 元素上触发。对于其它的元素，因为没有明确的动作可以匹配，所以 `PrintMeImplicitAction` 的 `isApplicable()` 方法被调用。它只有在 *printme* 属性设置为 true 的时候才会返回 true。也就是第一个 \<xyz\> 元素 (不是第二个) 与 \<abc\> 元素。第十行的第二个 \<xyz\> 元素，没有可用的动作，所以生成了一个内部的错误信息。这个信息通过 `PrintMe` 的最后一行代码 `StatusPrinter.print` 来进行输出。这也解释了上面的输出。

### 在实践中使用默认动作

logback-classic 与 logback-access 各自的 Joran 配置器只包含两个默认的动作，叫做 [`NestedBasicPropertyIA`](https://logback.qos.ch/xref/ch/qos/logback/core/joran/action/NestedBasicPropertyIA.html) 与 [`NestedComplexPropertyIA`](https://logback.qos.ch/xref/ch/qos/logback/core/joran/action/NestedComplexPropertyIA.html)。

`NestedBasicPropertyIA` 适用于任何属性的类型为原始类型 (或者  equivalent object type in the `java.lang` 包中的对象类型 )，枚举类，或者其它遵循 "valuesOf" 约定的类型。这些属性被称之为*基本* 或者*简单* 属性。如果一个类它包含一个名为 `valueOf()` 的静态方法，接受一个 `java.lang.String` 作为参数并且返回相关类型的实例，那么就说这个类遵循 "valueOf" 约定。目前，[`Level`](https://logback.qos.ch/xref/ch/qos/logback/classic/Level.html)，[`Duration`](https://logback.qos.ch/xref/ch/qos/logback/core/util/Duration.html)，以及 [`FileSize`](https://logback.qos.ch/xref/ch/qos/logback/core/util/FileSize.html) 类遵循这个约定。

`NestedComplexPropertyIA` 适用于 `NestedBasicPropertyIA` 不适用的情况，并且如果对象栈顶部的对象具有当前属性名的 set 与 add 方法，那么该属性名相当于当前元素的名称。这些属性可以反过来包含其它的组件。因此，这些属性可以说是有点*复杂*。由于复杂属性的存在，[`NestedComplexPropertyIA`](https://logback.qos.ch/xref/ch/qos/logback/core/joran/action/NestedComplexPropertyIA.html) 将会为内部组件实例化一个合适的类，并且通过使用父组件以及内部元素名的 set/add 方法将其附加到父组件上 (在对象栈的顶部)。相应的类通过当前 (内置) 元素的 *class* 属性来指定。但是，如果没有指定 *class* 属性，那么将会根据是否满足以下其中之一的条件来使用默认的类名。

1. 父对象属性指定的类有内部的规则相关联
2. set 方法包含一个指定类的 @DefaultClass 属性
3. set 方法的参数类型是一个含有共有构造方法的实体类

#### 默认类映射

在 logback-classic，有一些内部的规则将父 类/属性 名映射为默认的类。这些规则如下表所示：

| 父类                                           | 属性名    | 默认类                                              |
| ---------------------------------------------- | --------- | --------------------------------------------------- |
| ch.qos.logback.core.AppenderBase               | encoder   | ch.qos.logback.classic.encoder.PatternLayoutEncoder |
| ch.qos.logback.core.UnsynchronizedAppenderBase | encoder   | ch.qos.logback.classic.encoder.PatternLayoutEncoder |
| ch.qos.logback.core.AppenderBase               | layout    | ch.qos.logback.classic.PatternLayout                |
| ch.qos.logback.core.UnsynchronizedAppenderBase | layout    | ch.qos.logback.classic.PatternLayout                |
| ch.qos.logback.core.filter.EvaluatorFilter     | evaluator | ch.qos.logback.classic.boolex.JaninoEventEvaluator  |

在以后的版本中，这个列表可能会发生改变。最新的规则，请查看 logback-classic 中 [JoranConfigurator](https://logback.qos.ch/xref/ch/qos/logback/classic/joran/JoranConfigurator.html) 中的 `addDefaultNestedComponentRegistryRules` 方法。

#### 属性集

除了单个简单属性以及单个复杂属性外，logback 的默认动作支持属性集，它们可以是简单的或者复杂的。指定的属性通过 "add" 方法，而不是 set 方法。

### 动态添加新规则

Joran 包含一个允许 Joran 在解析 XML 文档的过程中，动态解析新规则的动作。示例代码，查看 *logback-examples/src/main/java/chapters/onJoran/newRule/* 文件夹。在这个包中，[`NewRuleCalculator`](https://logback.qos.ch/xref/chapters/onJoran/newRule/NewRuleCalculator.html) 仅仅设置两个规则，一个用来处理最顶层的元素，一个用来学习新规则。下面是 `NewRuleCalculator` 中相关的代码：

```java
ruleMap.put(new Pattern("*/computation"), new ComputationAction1());
ruleStore.addRule(new Pattern("/computation/newRule"), new NewRuleAction());
```

[`NewRuleAction`](https://logback.qos.ch/xref/ch/qos/logback/core/joran/action/NewRuleAction.html) 是 logback-core 的一部分，跟其它的动作非常的类似。它有 `begin()` 与 `end()` 方法，在每次解析器找到一个 *newRule*  元素时被调用。当被调用时，`begin()` 方法会去寻找 *pattern* 与 *actionClass* 属性。然后实例化相应的动作类，并将模式/动作作为一条新规则添加到 Joran 的规则存储中。

下面是如何在 xml 文件添加一条新规则：

```xml
<newRule pattern="*/computation/literal"
          actionClass="chapters.onJoran.calculator.LiteralAction"/>
```

使用 newRule 声明，我们可以看到 `NewRuleCalculator` 表现出跟之前看到的 `Calculator1` 同样的结果。包括计算在内，我们可以按照如下的方式进行表示：

> Example: *newRule.xml*

```xml
<computation name="toto">
  <newRule pattern="*/computation/literal" 
            actionClass="chapters.onJoran.calculator.LiteralAction"/>
  <newRule pattern="*/computation/add" 
            actionClass="chapters.onJoran.calculator.AddAction"/>
  <newRule pattern="*/computation/multiply" 
            actionClass="chapters.onJoran.calculator.MultiplyAction"/>

  <computation>
    <literal value="7"/>
    <literal value="3"/>
    <add/>
  </computation>   
 
  <literal value="3"/>
  <multiply/>
</computation>
```

```java
java chapters.onJoran.newRule.NewRuleCalculator src/main/java/chapters/onJoran/newRule/newRule.xml
```

> 作者原文中的命令写了两个 java

输出：

```java
The computation named [toto] resulted in the value 30
```

跟[之前](https://github.com/Volong/logback-chinese-manual/blob/master/11%E7%AC%AC%E5%8D%81%E4%B8%80%E7%AB%A0%EF%BC%9AJoran.md#%E5%8A%A8%E4%BD%9C%E7%9B%B8%E4%BA%92%E5%90%88%E4%BD%9C)的结果一致。

> 运行原文中的示例并不会输出坐着所说的结果，需要去掉 \<computation\> 中的 \<computation\> 标签才可以