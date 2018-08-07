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
| **c**{*length*}<br />**lo**{*length*}<br />**logger**{*length*} | 输出 logger 的名字作为日志事件的来源。转换字符接收一个作为它的第一个也是为一个参数。转换器的简写算法将会缩短 logger 的名字，但是通过不会丢失重要的信息。设置 length 的值为 0 是一个例外。它将会导致转换字符返回 logger 名字中最右边的点右边的字符。下面的表格提供了一个示例：<br /><table><tr><th>转换说明符</th><th>logger的名字</th><th>结果</th></tr><tr><td>%logger</td><td>mainPackage.sub.sample.Bar</td><td>mainPackage.sub.sample.Bar</td></tr><tr><td>%logger{0}</td><td>mainPackage.sub.sample.Bar</td><td>Bar</td></tr><tr><td>%logger{5}</td><td>mainPackage.sub.sample.Bar</td><td>m.s.s.Bar</td></tr><tr><td>%logger{10}</td><td>mainPackage.sub.sample.Bar</td><td>m.s.s.Bar</td></tr><tr><td>%logger{15}</td><td>mainPackage.sub.sample.Bar</td><td>m.s.sample.Bar</td></tr><tr><td>%logger{16}</td><td>mainPackage.sub.sample.Bar</td><td>m.sub.sample.Bar</td></tr><tr><td>%logger{26}</td><td>mainPackage.sub.sample.Bar</td><td>mainPackage.sub.sample.Bar</td></tr></table>logger 名字最右边的部分永远不会被简写，即使它的长度比 *length* 的值要大。其它的部分可能会被缩短为一个字符，但是永不会被移除。<br /> |
| **C**{*length*}  <br />**class**{*length*}                   | 输出发出日志请求的类的全限定名称。<br />跟 *%logger%* 转换符一样，它也可以接收一个整型的可选参数去缩短类名。0 表示特殊含义，在打印类名时将不会输出包的前缀名。默认表示打印类的全限定名。<br />生成调用者类的信息并不是特别快。因此，应该避免使用，除非执行速度不是问题。 |
| **contextName**<br />**cn**                                  | 输出日志事件附加到的 logger 上下文的名字。                   |
| **d**{*pattern*}  **date**{*pattern*}  **d**{*pattern*, *timezone*}  **date**{*pattern*, *timezone*} | 用于输出日志事件的日期。日期转换符允许接收一个字符串作为参数。字符串的语法与 [SimpleDateFormat](https://docs.oracle.com/javase/8/docs/api/java/text/SimpleDateFormat.html) 中的格式完全兼容。<br />你可以指定 "ISO8601" 来表示将日期格式为 ISO8601 类型。如果没有指定日期格式，那么 %date 转换字符默认为 [ISO860 类型](https://en.wikipedia.org/wiki/ISO_8601)。<br />这里有一个例子。它假设当前时间为 2006.10.20 星期五，作者刚刚吃完饭准备写这篇文档。<br /><table><tr><th>转换模式</th><th>结果</th></tr><tr><td>%d</td><td>2006-10-20 14:06:49,812</td></tr><tr><td>%date</td><td>2006-10-20 14:06:49,812</td></tr><tr><td>%date{ISO8601}</td><td>2006-10-20 14:06:49,812</td></tr><tr><td>%date{HH:mm:ss.SSS}</td><td>14:06:49.812</td></tr><tr><td>%date{dd MMM yyyy;HH:mm:ss.SSS}</td><td>20 oct. 2006;14:06:49.812</td></tr></table><br />第二个参数用于指定时区。例如， '%date{HH:mm:ss.SSS, Australia/Perth}' 将会打印世界上最孤立的城市，澳大利亚佩斯所在时区的日期。如果没有指定时区参数，则默认使用 Java 平台所在主机的时区。如果指定的时区不能识别或者拼写错误，则 [TimeZone.getTimeZone(String)](http://docs.oracle.com/javase/6/docs/api/java/util/TimeZone.html#getTimeZone(java.lang.String)) 方法会指定时区为 GMT。<br />`常见错误：` 对于 `HH:mm:ss,SSS` 模式，逗号会被解析为分隔符，所以最终会被解析为 `HH:mm:ss`，`SSS` 会被当作时区。如果你想在日期模式中使用逗号，那么你可以这样使用，%date{**"**HH:mm:ss,SSS**"**}  用双引号将日期模式包裹起来。 |
| **F / file**                                                 | 输出发出日志请求的 Java 源文件名。<br />由于生成文件的信息不是特别快，因此，应该避免使用，除非速度不是问题。 |
| **caller{depth}**<br />**caller{depthStart..depthEnd}**<br />**caller{depth, evaluator-1, ... evaluator-n}**<br />**caller{depthStart..depthEnd, evaluator-1, ... evaluator-n}** | 输出生成日志的调用者所在的位置信息。<br />位置信息依赖 JVM 的实现，但是通常由调用方法的全限定名以及调用者的来源组成。以及由圆括号括起来的文件名与行号。<br />*caller* 转换符还可以接收一个整形的参数，用来配置展示信息的深度。<br />例如，**%caller{2}** 会展示如下的信息：<br /><pre style="background-color:#eaecef">0    [main] DEBUG - logging statement<br />Caller+0   at mainPackage.sub.sample.Bar.sampleMethodName(Bar.java:22)<br />Caller+1   at mainPackage.sub.sample.Bar.createLoggingRequest(Bar.java:17)</pre> **%caller{3}**  会展示如下信息：<br /><pre><span style="background-color:#eaecef">16   [main] DEBUG - logging statement <br />Caller+0   at mainPackage.sub.sample.Bar.sampleMethodName(Bar.java:22) <br />Caller+1   at mainPackage.sub.sample.Bar.createLoggingRequest(Bar.java:17) <br />Caller+2   at mainPackage.ConfigTester.main(ConfigTester.java:38) </span></pre><br />*caller* 转换符还可以接收一个范围用来展示深度在这个范围内的信息。<br />例如，**%caller{1..2}** 会展示如下信息：<br /><pre>[main] DEBUG - logging statement <br />Caller+0   at mainPackage.sub.sample.Bar.createLoggingRequest(Bar.java:17)</pre><br />转换字符还可以接收一个 evaluator，在计算调用者数据之前通过指定的标准对日志事件进行测验。例如，**%caller{3, CALLER_DISPLAY_EVAL}** 会在 *CALLER_DISPLAY_EVAL* 返回一个肯定的答案，才会显示三行堆栈信息。<br />将在下面详细叙述 evaluator。 |
| **L / line**                                                 | 输出发出日志请求所在的行号。<br />生成行号不是特别快。因此，不建议使用，除非生成速度不是问题。 |
| **m / msg / message**                                        | 输出与日志事件相关联的，由应用程序提供的日志信息。           |
| **M / method**                                               | 输出发出日志请求的方法名。<br />生成方法名不是特别快，因此，应该避免使用，除非生成速度不是问题。 |
| **n**                                                        | 输出平台所依赖的行分割字符。<br />转换字符提供了像 "\n" 或 "\r\n" 一样的转换效果。因此指定行分隔符它是首选的指定方式。 |
| **p / le / level**                                           | 输出日志事件的级别。                                         |
| **r / relative**                                             | 输出应用程序启动到创建日志事件所花费的毫秒数                 |
| **t / thread**                                               | 输出生成日志事件的线程名。                                   |
| **X**{*key:-defaultVal*}<br />  **mdc**{*key:-defaultVal*}   | 输出生成日志事件的线程的 MDC (mapped diagnostic context)。<br />如果 **MDC** 转换字符后面跟着用花括号括起来的 kye，例 **%MDC{userid}**，那么 'userid' 所对应 MDC 的值将会输出。如果该值为 null，那么通过 :- 指定的[默认值](https://github.com/Volong/logback-chinese-manual/blob/master/03%E7%AC%AC%E4%B8%89%E7%AB%A0%EF%BC%9Alogback%20%E7%9A%84%E9%85%8D%E7%BD%AE.md#%E5%8F%98%E9%87%8F%E7%9A%84%E9%BB%98%E8%AE%A4%E5%80%BC) 将会输出。如果没有指定默认值，那么将会输出空字符串。<br />如果没有指定的 key，那么 MDC 的整个内容将会以 "key1=val1, key2=val2" 的格式输出。<br />查详情请见 [第八章](https://logback.qos.ch/manual/mdc.html) |
| **ex**{*depth*}  **exception**{*depth*}  **throwable**{*depth*}   **ex**{depth, evaluator-1, ..., evaluator-n}  **exception**{depth, evaluator-1, ..., evaluator-n}  **throwable**{depth, evaluator-1, ..., evaluator-n} | 输出日志事件相关的堆栈信息，默认情况下会输出全部的堆栈信息。<br /> *throwable* 转换词可以接收如下的参数：<br /><ul><li>*short*：输出堆栈信息的第一行<br></li><li>*full*：输出全部的堆栈信息</li><li>任意整数：输出指定行数的堆栈信息</li></ul><br />下面是一些示例：<br /><table><thead><th>转换模式</th><th>结果</th></thead><tbody><tr><td>%ex</td><td><pre>mainPackage.foo.bar.TestException: Houston we have a problem<br />  at mainPackage.foo.bar.TestThrower.fire(TestThrower.java:22)<br />  at mainPackage.foo.bar.TestThrower.readyToLaunch(TestThrower.java:17)<br />  at mainPackage.ExceptionLauncher.main(ExceptionLauncher.java:38)</pre></td></tr><tr><td>%ex{short}</td><td><pre><br />mainPackage.foo.bar.TestException: Houston we have a problem<br />  at mainPackage.foo.bar.TestThrower.fire(TestThrower.java:22)</pre></td></tr><tr><td>%ex{full}</td><td><pre><br />mainPackage.foo.bar.TestException: Houston we have a problem<br />  at mainPackage.foo.bar.TestThrower.fire(TestThrower.java:22)<br />  at mainPackage.foo.bar.TestThrower.readyToLaunch(TestThrower.java:17)<br />  at mainPackage.ExceptionLauncher.main(ExceptionLauncher.java:38)</pre></td></tr><tr><td>%ex{2}</td><td><pre>mainPackage.foo.bar.TestException: Houston we have a problem<br />  at mainPackage.foo.bar.TestThrower.fire(TestThrower.java:22)<br />  at mainPackage.foo.bar.TestThrower.readyToLaunch(TestThrower.java:17)</pre></td></tr></tbody></table><br />在输出前，转换字符还可以使用给定的标准再次检验日志事件。例如，使用 **%ex{full, EX_DISPLAY_EVAL}**，只有 *EX_DISPLAY_EVAL* 返回一个否定的答案，才会输出全部的堆栈信息。evaluator 在接下来的文档中将会进一步叙述。<br />如果你没有指定 %throwable 或者其它跟 throwable 相关的转换字符，那么 `PatternLayout` 会在最后一个转换字符加上这个。因为堆栈信息非常的重要。如果你不想展示堆栈信息，那么可以使用 %nopex (作者原文为 $nopex) 可以替代 %throwable。详情见 %nopex。 |
| **xEx**{*depth*}  <br />**xException**{*depth*}  **xThrowable**{*depth*}   <br />**xEx**{depth, evaluator-1, ..., evaluator-n}  <br />**xException**{depth, evaluator-1, ..., evaluator-n} <br />**xThrowable**{depth, evaluator-1, ..., evaluator-n} | 跟 %throwable 类似，只不过多了类的包信息。<br />在每个堆栈信息的末尾，多了包含 jar 文件的字符串，后面再加上具体的实现版本。这项创造性的技术是来自 [James Strachan](http://macstrac.blogspot.com/2008/09/better-stack-traces-in-java-with-log4j.html) 的建议。如果该信息不确定，那么类的包信息前面会有一个波浪号 (~)。<br />下面是一个例子：<br /><pre>java.lang.NullPointerException<br />  at com.xyz.Wombat(Wombat.java:57) ~[wombat-1.3.jar:1.3]<br />  at  com.xyz.Wombat(Wombat.java:76) ~[wombat-1.3.jar:1.3]<br />  at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.5.0_06]<br />  at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:39) ~[na:1.5.0_06]<br />  at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25) &#126;[na:1.5.0_06]<br />  at java.lang.reflect.Method.invoke(Method.java:585) &#126;[na:1.5.0_06]<br />  at org.junit.internal.runners.TestMethod.invoke(TestMethod.java:59) [junit-4.4.jar:na]<br />  at org.junit.internal.runners.MethodRoadie.runTestMethod(MethodRoadie.java:98) [junit-4.4.jar:na]<br />  ...etc </pre>logback 努力的去确保类的包信息正确的展示，即使是在复杂的类加载层次中。但是，一个不能保证信息的绝对正确，那么在这些数据的前面将会多一个波浪符 (\~)。因此，从理论上来说，打印的类的包信息跟真实的类的包信息是有区别的。在上面的例子中，类 Wombat 的包信息前面有一个波浪符，在实际的情况中，它真实包可能为  [wombat.jar:1.7]。<br />但是请注意潜在的性能损耗，计算[包信息默认是禁止的](https://github.com/Volong/logback-chinese-manual/blob/master/03%E7%AC%AC%E4%B8%89%E7%AB%A0%EF%BC%9Alogback%20%E7%9A%84%E9%85%8D%E7%BD%AE.md#%E5%9C%A8%E5%A0%86%E6%A0%88%E4%B8%AD%E5%B1%95%E7%A4%BA%E5%8C%85%E6%95%B0%E6%8D%AE)。当启用了计算包信息，那么 `PatternLayout` 将会自动认为在字符串模式的末尾 %xThrowable 替代了 %throwable。<br />根据用户的[反馈](https://jira.qos.ch/browse/LOGBACK-324)，Netbeans 会阻止包信息的打印。 |
| **nopex**<br />**nopexception**                              | 这个转换字符不会输出任何数据，因此，它可以用来有效忽略异常信息。<br />%nopex 转换字符允许用户重写 `PatternLayout` 内部的安全机制，该机制将会在没有指定其它处理异常的转换字符时，默认添加 %xThrowable。 |
| **marker**                                                   | 输出与日志请求相关的标签。<br />一旦标签包含子标签，那么转换器将会根据下面的格式展示父标签与子标签。<br />*parentName [child1, child2]* |
| **property{key}**                                            | 输出属性 *key* 所对应的值。相关定义参见 [定义变量](https://github.com/Volong/logback-chinese-manual/blob/master/03%E7%AC%AC%E4%B8%89%E7%AB%A0%EF%BC%9Alogback%20%E7%9A%84%E9%85%8D%E7%BD%AE.md#%E5%8F%98%E9%87%8F%E6%9B%BF%E6%8D%A2) 以及[作用域](https://github.com/Volong/logback-chinese-manual/blob/master/03%E7%AC%AC%E4%B8%89%E7%AB%A0%EF%BC%9Alogback%20%E7%9A%84%E9%85%8D%E7%BD%AE.md#%E4%BD%9C%E7%94%A8%E5%9F%9F)。如果 key 在 logger context 中没有找到，那么将会去系统属性中找。<br />*key* 没有默认值，如果缺失，则会展示 " Property_HAS_NO_KEY" 的错误信息。 |
| **replace(p){r, t}**                                         | 在子模式 'p' 产生的字符中，将所有出现正则表达式 'r' 的地方替换为 't'。例如，"%replace(%msg){'\s', ''}" 将会移除事件消息中所有空格。<br />模式 'p' 可以是任意复杂的甚至由多个转换字符组成。例如，"%replace(%logger %msg){'\.', '/'}" 将会替换 logger 以及消息中所有的点为斜杆。 |
| **rEx**{*depth*}  **rootException**{*depth*}   **rEx**{depth, evaluator-1, ..., evaluator-n}  **rootException**{depth, evaluator-1, ..., evaluator-n} | 输出与日志事件相关的堆栈信息，根异常将会首先输出，而是标准的"根异常最后输出"。下面是一个输出例子：<br /><pre>java.lang.NullPointerException<br />  at com.xyz.Wombat(Wombat.java:57) ~[wombat-1.3.jar:1.3]<br />  at com.xyz.Wombat(Wombat.java:76) ~[wombat-1.3.jar:1.3]<br />Wrapped by: org.springframework.BeanCreationException: Error creating bean with name 'wombat': <br />  at org.springframework.AbstractBeanFactory.getBean(AbstractBeanFactory.java:248) [spring-2.0.jar:2.0]<br />  at org.springframework.AbstractBeanFactory.getBean(AbstractBeanFactory.java:170) [spring-2.0.jar:2.0]<br />  at org.apache.catalina.StandardContext.listenerStart(StandardContext.java:3934) [tomcat-6.0.26.jar:6.0.26]</pre>%rootException 跟 %xException 类似，也允许一些可选的参数，包括深度以及 evaluator。它也会输出包信息。简单来说，%rootException 跟 %xException 非常的类似，仅仅是异常输出的顺序完全相反。<br />  %rootException 的作者 Tomasz Nurkiewicz 在他的博客说明了他所作的贡献 ["Logging exceptions root cause first"](http://nurkiewicz.blogspot.com/2011/09/logging-exceptions-root-cause-first.html)。 |

#### % 有特殊的含义

在给定的转换模式上下文中，% 有特殊的含义。如果作为字面量，需要进行转义。例如，"%d %p \% %m%n"。

#### 转换字符对字面量的限制

在大多数的情况下，字面量包括空格或者其它的分隔符，所以它们不会与转换字符混淆。例如，"%level [%thread] - %message%n" 包含字面量字符 " [" 与 "] - "。但是，如果一个转换字符后面紧跟着一个字面量，那么 logback 的模式解析器将会错误的认为这个字面量也是转换字符的一部分。例如，"%date**%nHello**" 将会被解析成两个转换字符 %date 与 %nHello，但是 %nHello 不是一个转换字符，所以 logback 将会输出 %PARSER_ERROR[nHello]。如果你想要区分 %n 跟 Hello，可以通过给 %n 传递一个空参数。例如，"%date**%n{}**Hello" 将会被解析为 %date %n 再紧跟着一个字符串 "Hello"。

## 格式修改器

默认情况下，相关信息按照原样输出。但是，在格式修改器的帮助下，可以对每个数据字段进行对齐，以及更改最大最小宽度。

可选的格式修改器放在百分号跟转换字符之间。

第一个可选的格式修改器是*左对齐标志*，也就是减号 (-) 字符。接下来的是*最小字段宽度修改器*，它是一个十进制常量，表示输出至少多少个字符。如果字段包含很少的数据，它会选择填充左边或者右边，直到满足最小宽度。默认是填充左边 (右对齐)，但是你可以通过左对齐标志来对右边进行填充。填充字符为空格。如果字段的数据大于最小字段的宽度，会自动扩容去容纳所有的数据。字段的数据永远不会被截断。

这个行为可以通过使用*最大字段宽度修改器*来改变，它通过一个点后面跟着一个十进制常量来指定。如果字段的数据长度大于最大字段的宽度，那么会从数据字段的开头移除多余的字符。举个🌰，如果最大字段的宽度是 8，数据长度是十个字符的长度，那么开头的两个字符将会被丢弃。这个行为跟 C 语言中 printf 函数从后面开始截断的行为相违背。

如果想从后面开始截断，可以在点后面增加一个减号。如果是这样的话，最大字段宽度是 8，数据长度是十个字符的长度，那么最后两个字符将会被丢弃。

下面是各种格式修改器的例子：

| 格式修改器    | 左对齐 | 最小宽度 | 最大宽度 | 备注                                                         |
| ------------- | ------ | -------- | -------- | ------------------------------------------------------------ |
| %20logger     | false  | 20       | none     | 如果 logger 的名字小于 20 个字符的长度，那么会在左边填充空格 |
| %-20logger    | true   | 20       | none     | 如果 logger 的名字小于 20 个字符的长度，那么会在右边填充空格 |
| %.30logger    | NA     | none     | 30       | 如果 logger 的名字大于 30 个字符的长度，那么从前面开始截断   |
| %20.30logger  | false  | 20       | 30       | 如果 logger 的名字大于 20 个字符的长度，那么会从左边填充空格。但是如果 logger 的名字大于 30 字符，将会从前面开始截断 |
| %-20.30logger | true   | 20       | 30       | 如果 logger 的名字小于 20 个字符的长度，那么从右边开始填充空格。但是如果 logger 的名字大于 30 个字符，将会从前面开始截断 |
| %.-30logger   | NA     | none     | 30       | 如果 logger 的名字大于 30 个字符的长度，那么从后面开始截断   |

下面的表格列出了格式修改器截断的例子。但是请注意综括号 "[]" 不是输出结果的一部分，它只是用来区分输出的长度。

| 格式修改器      | logger 的名字         | 结果                                                         |
| --------------- | --------------------- | ------------------------------------------------------------ |
| [%20.20logger]  | main.Name             | [&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;main.Name] |
| [%-20.20logger] | main.Name             | [main.Name&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;] |
| [%10.10logger]  | main.foo.foo.bar.Name | [o.bar.Name]                                                 |
| [%10.-10logger] | main.foo.foo.bar.Name | [main.foo.f]                                                 |

### 只输出日志等级的一个字符

除了可以输出 TRACE, DEBUG, WARN, INFO 或者 ERROR 来表示日志等级之外，还是输出T, D, W, I 与 E 来进行表示。你可以[自定义转换器](https://logback.qos.ch/manual/layouts.html#customConversionSpecifier) 或者利用刚才讨论的格式修改器来缩短日志级别为一个字符。这个转换说明符可能为 "%.-1level "。

## 转换字符的选项

一个转换字符后面可以跟一个选项。它们通过综括号来声明。我们之前已经看到了一些可能的选项。例如之前的 MDC 转换说明符 *%mdc{someKey}*。

一个转换说明符可能有多个可选项。一个转换说明符可以充分利用我们即将介绍到的 evaluator，可以添加多个 evaluator 的名字到可选列表。如下：

```xml
<pattern>%-4relative [%thread] %-5level - %msg%n \
  %caller{2, DISP_CALLER_EVAL, OTHER_EVAL_NAME, THIRD_EVAL_NAME}</pattern>
```

如果这些选项中包含了一些特殊字符，例如花括号，空格，逗号。你可以使用单引号或者双引号来包裹它们。例如：

```xml
<pattern>%-5level - %replace(%msg){'\d{14,16}', 'XXXX'}%n</pattern>
```





