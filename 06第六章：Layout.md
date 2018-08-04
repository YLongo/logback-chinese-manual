## 什么是 layout？

layout 是 logback 的组件，负责将日志事件转换为字符串。[`Layout`](https://logback.qos.ch/xref/ch/qos/logback/core/Layout.html) 接口中的 `format()` 方法接受一个表示日志事件的对象 (任何类型) 并返回一个字符串。`Layout` 接口的概要如下：

```java
public interface Layout<E> extends ContextAware, LifeCycle {

  String doLayout(E event);
  String getFileHeader();
  String getPresentationHeader();
  String getFileFooter();
  String getPresentationFooter();
  String getContentType();
}
```

这个接口相对简单，但是它可以满足大部分的格式化需求。

## Logback-classic

logback-classic 仅仅用来处理 [`ch.qos.logback.classic.spi.ILoggingEvent`](https://logback.qos.ch/xref/ch/qos/logback/classic/spi/ILoggingEvent.html) 类型的日志事件。我们将在这个部分说明这个事实。

## 定制 Layout

让我们为 logback-classic  模块实现一个简单但是实用的功能，打印应用启动所耗费的时间，日志事件的级别，被综括号包裹的调用者线程，logger 名，破折号后面跟日志信息，以及新起一行。

类似下面的输出：

```java
10489 DEBUG [main] com.marsupial.Pouch - Hello world.
```

下面是一种可能的实现：

>   Example: MySampleLayout.java

```java
package chapters.layouts;

import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.core.LayoutBase;

public class MySampleLayout extends LayoutBase<ILoggingEvent> {

  public String doLayout(ILoggingEvent event) {
    StringBuffer sbuf = new StringBuffer(128);
    sbuf.append(event.getTimeStamp() - event.getLoggingContextVO.getBirthTime());
    sbuf.append(" ");
    sbuf.append(event.getLevel());
    sbuf.append(" [");
    sbuf.append(event.getThreadName());
    sbuf.append("] ");
    sbuf.append(event.getLoggerName();
    sbuf.append(" - ");
    sbuf.append(event.getFormattedMessage());
    sbuf.append(CoreConstants.LINE_SEP);
    return sbuf.toString();
  }
}
```

`MySampleLayout` 继承自 [`LayoutBase`](https://logback.qos.ch/xref/ch/qos/logback/core/LayoutBase.html)。这个类管理所有 layout 实例的状态信息，例如：layout 是否启动或者停止，头部，尾部以及内容类型数据。它让开发者通过自己 `Layout` 集中在日志具体的格式化上。`LayoutBase` 类是通用的。在它的类声明上，`MySampleLayout` 继承 `LayoutBase<ILoggingEvent>`。

在上面这个例子中，`doLayout` 方法忽略了日志事件中任何可能的异常。在实际应用中，你可能需要打印异常信息。

### 配置自定义的 layout

配置自定义的 layout 跟其它的组件一样的配置。根据之前提到的，`FileAppender` 及其子类期望一个 encoder。为了去满足这个需求，我们将一个包裹了我们自己定义的 `MySampleLayout` 的 `LayoutWrappingEncoder` 的实例传递给 `FileAppender`。下面是配置示例：

>   Example: *sampleLayoutConfig.xml*

```xml
<configuration>

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
      <layout class="chapters.layouts.MySampleLayout" />
    </encoder>
  </appender>

  <root level="DEBUG">
    <appender-ref ref="STDOUT" />
  </root>
</configuration>
```

[`chapters.layouts.SampleLogging`](https://logback.qos.ch/xref/chapters/layouts/SampleLogging.html) 这个简单的应用通过第一个参数接收配置文件，然后打印了一个 debug 信息，接着打印了 error 信息。

在 *logback-examples* 文件夹下通过以下命令来运行：

```bash
java chapters.layouts.SampleLogging src/main/java/chapters/layouts/sampleLayoutConfig.xml
```

将会输出：

```java
0 DEBUG [main] chapters.layouts.SampleLogging - Everything's going well
0 ERROR [main] chapters.layouts.SampleLogging - maybe not quite...
```

这种足够简单。读者应该会发现，在 [`MySampleLayout2.java`](https://logback.qos.ch/xref/chapters/layouts/MySampleLayout2.html) 中，我们自定义的 layout 做了一点点的修改。正如本手册一直提到的，为 layout 或者其它 logback 的组件添加一个属性，跟为这个属性添加一个 set 方法一样简单。

[`MySampleLayout2`](https://logback.qos.ch/xref/chapters/layouts/MySampleLayout2.html) 类包含了两个属性。第一个是可以将一个前缀添加到输出的日志中。第二个属性可以用来选择是否展示发送日志请求的线程名。

下面是 [`MySampleLayout2`](https://logback.qos.ch/xref/chapters/layouts/MySampleLayout2.html) 类：

```java
package chapters.layouts;

import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.core.LayoutBase;

public class MySampleLayout2 extends LayoutBase<ILoggingEvent> {

  String prefix = null;
  boolean printThreadName = true;

  public void setPrefix(String prefix) {
    this.prefix = prefix;
  }

  public void setPrintThreadName(boolean printThreadName) {
    this.printThreadName = printThreadName;
  }

  public String doLayout(ILoggingEvent event) {
    StringBuffer sbuf = new StringBuffer(128);
    if (prefix != null) {
      sbuf.append(prefix + ": ");
    }
    sbuf.append(event.getTimeStamp() - event.getLoggerContextVO().getBirthTime());
    sbuf.append(" ");
    sbuf.append(event.getLevel());
    if (printThreadName) {
      sbuf.append(" [");
      sbuf.append(event.getThreadName());
      sbuf.append("] ");
    } else {
      sbuf.append(" ");
    }
    sbuf.append(event.getLoggerName());
    sbuf.append(" - ");
    sbuf.append(event.getFormattedMessage());
    sbuf.append(LINE_SEP);
    return sbuf.toString();
  }
}
```

添加相应的 set 方法就可以开启属性的配置。`PrintThreadName` 属性是 `boolean` 而不是 `String` 类型。关于配置 logback 的详细信息请参见[第三章：logback 的配置](https://github.com/Volong/logback-chinese-manual/blob/master/03%E7%AC%AC%E4%B8%89%E7%AB%A0%EF%BC%9Alogback%20%E7%9A%84%E9%85%8D%E7%BD%AE.md)。[第十一章](https://logback.qos.ch/manual/onJoran.html)将会提供更详细的内容。下面是关于 `MySampleLayout2` 的相关配置：

```xml
<configuration>

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
      <layout class="chapters.layouts.MySampleLayout2"> 
        <prefix>MyPrefix</prefix>
        <printThreadName>false</printThreadName>
      </layout>
    </encoder>
  </appender>

  <root level="DEBUG">
    <appender-ref ref="STDOUT" />
  </root>
</configuration>
```

## PatternLayout

logback 配备了一个更加灵活的 layout 叫做 [`PatternLayout`](https://logback.qos.ch/xref/ch/qos/logback/classic/PatternLayout.html)。跟所有的 layout 一样，`PatternLayout` 接收一个日志事件并返回一个字符串。但是，可以通过调整 `PatternLayout` 的转换模式来进行定制。

`PatternLayout` 中的转换模式与 C 语言中 `printf()` 方法中的转换模式密切相关。转换模式由字面量与格式控制表达式也叫*转换说明符*组成。你可以在转换模式中自由的插入字面量。每一个转换说明符由一个百分号开始 '%'，后面跟随可选的*格式修改器*，以及用综括号括起来的转换字符与可选的参数。转换字符需要转换的字段。如：logger 的名字，日志级别，日期以及线程名。格式修改器控制字段的宽度，间距以及左右对齐。

正如我们已经在其它地方提到过的，`FileAppender` 及其子类需要一个 encoder。因为，当将 `FileAppender` 及其子类与 `PatternLayout` 结合使用时，`PatternLayout` 必须用 encoder 包裹起来。鉴于 `FileAppender/PatternLayout` 结合使用很常见，因此 logback 单独设计了一个名叫 `PatternLayoutEncoder` 的 encoder，包裹了一个 `PatternLayout`，因此它可以被当作一个 encoder。下面是通过代码配置 `ConsoleAppender` 与 `PatternLayoutEncoder` 使用的例子：

>   Example: [PatternSample.java](https://logback.qos.ch/xref/chapters/layouts/PatternSample.html)

```java
package chapters.layouts;

import org.slf4j.LoggerFactory;

import ch.qos.logback.classic.Logger;
import ch.qos.logback.classic.LoggerContext;
import ch.qos.logback.classic.encoder.PatternLayoutEncoder;
import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.core.ConsoleAppender;

public class PatternSample {

  static public void main(String[] args) throws Exception {
    Logger rootLogger = (Logger)LoggerFactory.getLogger(Logger.ROOT_LOGGER_NAME);
    LoggerContext loggerContext = rootLogger.getLoggerContext();
    // we are not interested in auto-configuration
    loggerContext.reset();

    PatternLayoutEncoder encoder = new PatternLayoutEncoder();
    encoder.setContext(loggerContext);
    encoder.setPattern("%-5level [%thread]: %message%n");
    encoder.start();

    ConsoleAppender<ILoggingEvent> appender = new ConsoleAppender<ILoggingEvent>();
    appender.setContext(loggerContext);
    appender.setEncoder(encoder); 
    appender.start();

    rootLogger.addAppender(appender);

    rootLogger.debug("Message 1"); 
    rootLogger.warn("Message 2");
  } 
}
```

在上面这个例子中，转换模式被设置为 "**%-5level [%thread]: %message%n** "，关于 logback 中简短的转换字符将会很快给出。运行 `PatternSample`：

```bash
java java chapters.layouts.PatternSample
```

将会输出如下信息：

```java
DEBUG [main]: Message 1 
WARN  [main]: Message 2
```

在转换模式 **"%-5level [%thread]: %message%n"** 中，字面量与转换说明符之间没有明显的分隔符。当对转换模式进行解析的时候，`PatternLayout` 有能力对字面量 (空格符，方括号，冒号) 和 转换说明符进行区分。在上面的例子中，转换说明符 %-5level 表示日志事件的级别的字符应该向左对齐，保持五个字符的宽度。具体的转换格式将会在下面介绍。

在 `PatternLayout` 中，括号用于对转换模式进行分组。**'(' 与 ')' 有特殊的含义，因此如果想用作字面量，需要进行特殊的转义**。圆括号的特殊含义将在[下面](https://logback.qos.ch/manual/layouts.html#Parentheses) 进行详细的介绍。

之前提到过，特定的转换模式可以通过花括号指定可选的参数。一个简单的可选转换模式可以是 %logger{10}。在这里 "logger" 就是转换字符，10 就是可选参数。可选参将在[下面](https://logback.qos.ch/manual/layouts.html#cwOptions)详细介绍。

转换字符与它们的可选参数在下面的表格中进行详细叙述。当多个转换字符在同一个单元格中被列出来，它们被当作别名来考虑。

| 转换字符                                                     | 效果                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| **c**{*length*}<br />**lo**{*length*}<br />**logger**{*length*} | <table border="1">	<tr>		<th>Heading</th>		<th>Another Heading</th>	</tr>	<tr>		<td>row 1, cell 1</td>		<td>row 1, cell 2</td>	</tr>	<tr>		<td>row 2, cell 1</td>	<td>row 2, cell 2</td>	</tr></table> |
| **C**{*length*}  **class**{*length*}                         | 输出发出日志请求的类的全限定名称。<br />跟 *%logger%* 转换符一样，它也可以接收一个整型的可选参数去缩短类名。0 表示特殊含义，在打印类名时将不会输出包的前缀名。默认表示打印类的全限定名。<br />生成调用者类的信息并不是特别快。因此，应该避免使用，除非执行速度不是问题。 |
| **contextName** **cn**                                       | 输出日志事件附加到的 logger 上下文的名字。                   |
| **d**{*pattern*}  **date**{*pattern*}  **d**{*pattern*, *timezone*}  **date**{*pattern*, *timezone*} | 用于输出日志事件的日期。日期转换符允许接收一个字符串作为参数。字符串的语法与 [SimpleDateFormat](https://docs.oracle.com/javase/8/docs/api/java/text/SimpleDateFormat.html) 中的格式完全兼容。<br />你可以指定 "ISO8601" 来表示将日期格式为 ISO8601 类型。如果没有指定日期格式，那么 %date 转换字符默认为 [ISO860 类型](https://en.wikipedia.org/wiki/ISO_8601)。<br />这里有一个例子。它假设当前时间为 2006.10.20 星期五，作者刚刚吃完饭准备写这篇文档。<br />**转换模式**                                                       \| **结果**<br />%d                                                                 \| 2006-10-20 14:06:49,812 <br />%date                                                           \| 2006-10-20 14:06:49,812 <br />%date{ISO8601}                                         \| 2006-10-20 14:06:49,812 <br />%date{HH:mm:ss.SSS}                              \| 14:06:49.812 <br />%date{dd MMM yyyy;HH:mm:ss.SSS}    \| 20 oct. 2006;14:06:49.812 <br />第二个参数用于指定时区。例如， '%date{HH:mm:ss.SSS, Australia/Perth} 将会打印澳大利亚  佩斯所在时区的日期。 |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |



