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

### 配置自定义 layout

