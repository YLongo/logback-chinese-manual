本章涉及到的内容为将 log4j 的组件，例如 appender 或者 layout 迁移到 logback-classic。

仅仅调用 log4j 客户端 API 的软件，也就是  `org.apache.log4j` 包中 `Logger` 或者 `Category` 类，可以通过 [SLF4J 迁移工具](https://www.slf4j.org/migrator.html)使用 SLF4J 来进行自动迁移。为了将 *log4j.property* 文件转换为同等的 logback 配置，你可以使用 [log4j.properties 转换器](https://logback.qos.ch/translator/)。

在某种程度上来说，log4j 与 logback-classic 密切相关。核心组件，logger，appender 以及 layout 在两个框架中都存在，并且目的一致。类似的，最重要的内部数据结构，叫做 `LoggingEvent`，在两个框架中非常相似，但是实现完全不同。最主要的是，在 logback-classic 中，`LoggingEvent` 实现了 `ILoggingEvent` 接口。迁移 log4j 组件到 logback-classic 最大的改变在于 `LoggingEvent` 类相关的实现不同。但是，请放心，这些变化是有限的。如果你尽了最大的努力仍然不能将 log4j 组件迁移到 logback-classic，你可以通过 [logback 开发邮件列表](https://logback.qos.ch/mailinglist.html)来进行提问。logback 的开发者应该可以提供指导。

### 迁移 log4j 的 layout

假设我们现在要迁移一个简单的，名叫 [TrivialLog4jLayout](https://logback.qos.ch/xref/chapters/migrationFromLog4j/TrivialLog4jLayout.html) 的 log4j layout，它将日志事件中的消息作为格式化消息返回。代码如下：

```java
package chapters.migrationFromLog4j;

import org.apache.log4j.Layout;
import org.apache.log4j.spi.LoggingEvent;

public class TrivialLog4jLayout extends Layout {

  public void activateOptions() {
    
  }

  public String format(LoggingEvent loggingEvent) {
    return loggingEvent.getRenderedMessage();
  }

  public boolean ignoresThrowable() {
    return true;
  }
}
```

等价的 logback-classic [TrivialLogbackLayout](https://logback.qos.ch/xref/chapters/migrationFromLog4j/TrivialLogbackLayout.html) 如下：

```java
package chapters.migrationFromLog4j;

import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.core.LayoutBase;

public class TrivialLogbackLayout extends LayoutBase<ILoggingEvent> {

  public String doLayout(ILoggingEvent loggingEvent) {
    return loggingEvent.getMessage();
  }
}
```

正如你所见，在 logback-classic layout 中，格式化的方法叫做 `doLayout`，而在 log4j 中叫 `format()`。因为在 logback-classic 中没有等价的方法，所以 `ignoresThrowable()` 方法则不需要。logback-classic layout 必须继承 `LayoutBase<ILoggingEvent>`  类。

`activateOptions()` 方法的优点值得进一步讨论。在 log4j 中，一个 layout 有它自己的 `activateOptions()` 方法，通过 log4j 的配置程序，也就是 `PropertyConfigurator` 与 `DOMConfigurator`，会在 layout 所有的选项都设置完之后调用。因此，layout 有机会去检查它的所有的选项是否一致，如果是，那么开始进行初始化。

在 logback-classic 中，layout 必须实现 [LifeCycle](https://logback.qos.ch/xref/ch/qos/logback/core/spi/LifeCycle.html) 接口，该接口包含了一个 `start()` 方法。这个 `start()` 方法相当 log4j 中的 `activateOptions()` 方法。

### 迁移 log4j 的 appender

迁移 appender 与迁移 layout 相当的类似。下面是有一个名为 [TrivialLog4jAppender](https://logback.qos.ch/xref/chapters/migrationFromLog4j/TrivialLog4jAppender.html) 的简单 appender，它会在控制台输出由它的 layout 返回的字符串。

```java
package chapters.migrationFromLog4j;

import org.apache.log4j.AppenderSkeleton;
import org.apache.log4j.spi.LoggingEvent;


public class TrivialLog4jAppender extends AppenderSkeleton {

  protected void append(LoggingEvent loggingevent) {
    String s = this.layout.format(loggingevent);
    System.out.println(s);
  }

  public void close() {
    // nothing to do
  }

  public boolean requiresLayout() {
    return true;
  }
}
```

在 logback-classic 中等价的写法为 [TrivialLogbackAppender](https://logback.qos.ch/xref/chapters/migrationFromLog4j/TrivialLogbackAppender.html)，如下：

```java
package chapters.migrationFromLog4j;

import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.core.AppenderBase;

public class TrivialLogbackAppender extends AppenderBase<ILoggingEvent> {

  @Override
  public void start() {
    if (this.layout == null) {
      addError("No layout set for the appender named [" + name + "].");
      return;
    }
    super.start();
  }

  @Override
  protected void append(ILoggingEvent loggingevent) {
    // AppenderBase.doAppend 只会在这个 appender 成功启动之后调用这个方法
    String s = this.layout.doLayout(loggingevent);
    System.out.println(s);
  }
}
```

比较这两个类，你会发现 `append()` 方法的内容没有改变。`requiresLayout` 方法在 logback 中没有用到，所以它可以被移除。在 logback 中，`stop()` 方法与 log4j 中的 `close()` 方法等价。然而，logback-classic 中的 `AppenderBase` 包含一个没有实现的 `stop` 方法，但是在这个简单的 appender 已经足够了。