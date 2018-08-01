## 什么是 encoder

encoder 将日志事件转换为字节数组，同时将字节数组写入到一个 `OutputStream` 中。encoder 在 logback 0.9.19 版本引进。在之前的版本中，大多数的 appender 依赖 layout 将日志事件转换为 string，然后再通过 `java.io.Writer` 写出。在之前的版本中，用户需要在 `FileAppender` 中内置一个 `PatternLayout`。在 0.9.19 之后的版本中，`FileAppender` 以及子类[需要一个 encoder 而不是 layout](https://logback.qos.ch/codes.html#layoutInsteadOfEncoder)。

为什么会有这个改变？

layout 将会在下一章节讨论，它只能将日志事件转换为成 string。而且，考虑到 layout 在日志事件写出时不能控制日志事件，不能将日志事件批量聚合。与之相反的是，encoder 不但可以完全控制字节写出时的格式，而且还可以控制这些字节什么时候被写出。

`PatternLayoutEncoder` 是目前真正唯一有用的 encoder。它仅仅包裹了一个 `PatternLayout` 就完成了大部分的工作。因此，除了不必要的复杂性，encoder 似乎不会有太多的用处。但是，我们希望一个全新的更加强大的 encoder 来改变这种印象。

## Encoder 接口

encoder 负责将日志事件转换为字节数组，并将字节数组写入到合适的 `OutputStream` 中。所以，encoder 可以完全控制将什么样的字节以及什么时候将字节写入到由 appender 维护的 `OutputStream` 中。下面是 [Encoder 接口:](https://logback.qos.ch/xref/ch/qos/logback/core/encoder/Encoder.html) 

```java
package ch.qos.logback.core.encoder;

public interface Encoder<E> extends ContextAware, LifeCycle {

   /**
   * This method is called when the owning appender starts or whenever output
   * needs to be directed to a new OutputStream, for instance as a result of a
   * rollover.
   */
  void init(OutputStream os) throws IOException;

  /**
   * Encode and write an event to the appropriate {@link OutputStream}.
   * Implementations are free to defer writing out of the encoded event and
   * instead write in batches.
   */
  void doEncode(E event) throws IOException;


  /**
   * This method is called prior to the closing of the underling
   * {@link OutputStream}. Implementations MUST not close the underlying
   * {@link OutputStream} which is the responsibility of the owning appender.
   */
  void close() throws IOException;
}
```

正如你所看见的，`Encoder` 接口仅仅包含几个方法，但是令人惊讶的是这些方法可以完成许多有用的事情。

## LayoutWrappingEncoder

直到 logback 的 0.9.19 版本，许多 appender 依赖 layout 实例去控制日志的格式化输出。因为基于 layout 接口存在了大量的代码，所以我们需要一种方式容 encoder 与 layout 进行交互。[LayoutWrappingEncoder](https://logback.qos.ch/xref/ch/qos/logback/core/encoder/LayoutWrappingEncoder.html) 就是 encoder 与 layout 之间的桥梁。它实现了 encoder 接口并且包裹了一个 layout，通过委托该 layout 将日志事件转换为字符串。

下面是 `LayoutWrappingEncoder` 的部分代码，说明了如何委托包裹的 layout 实例去完成工作。

```java
package ch.qos.logback.core.encoder;

public class LayoutWrappingEncoder<E> extends EncoderBase<E> {

  protected Layout<E> layout;
  private Charset charset;
 
   // encode a given event as a byte[]
   public byte[] encode(E event) {
     String txt = layout.doLayout(event);
     return convertToBytes(txt);
  }

  private byte[] convertToBytes(String s) {
    if (charset == null) {
      return s.getBytes();
    } else {
      return s.getBytes(charset);
    }
  } 
}
```

`doLayout()` 方法首先通过包裹的 layout 将日志事件转换为字符串。返回的字符串结果根据用户设定的字符编码 (charset) 再转换为字节数组。

## PatternLayoutEncoder

由于 `PatternLayout` 是最常用的 layout，logback 使用 `PatternLayoutEncoder` 来满足这种用法。它扩展了 `LayoutWrappingEncoder`，被限制用来包裹 `PatternLayout` 实例。

在 logback 0.9.19 版本，无论 `FileAppender` 还是其子类通过 `PatternLayout` 来进行配置，都必须使用 `PatternLayoutEncoder` 来代替。具体的解释参见：[layoutInsteadOfEncoder](https://logback.qos.ch/codes.html#layoutInsteadOfEncoder)。

#### immediateFlush 属性

在 `LOGBACK 1.2.0` 中, `immediateFlush` 属性是 appender 的一部分。

#### 用格式化字符串作为开头

为了帮助解析日志文件，logback 可以将格式化字符串插入到日志文件的顶部。这个功能默认是**关闭**的。可以为相关的 `PatternLayoutEncoder` 设置 `outputPatternAsHeader` 属性的值为 `true` 来开启这个功能。下面是示例：

```java
<appender name="FILE" class="ch.qos.logback.core.FileAppender"> 
  <file>foo.log</file>
  <encoder>
    <pattern>%d %-5level [%thread] %logger{0}: %msg%n</pattern>
    <outputPatternAsHeader>true</outputPatternAsHeader>
  </encoder> 
</appender>
```

将会在日志文件中输出类似下面的日志：

```java
#logback.classic pattern: %d [%thread] %-5level %logger{36} - %msg%n
2012-04-26 14:54:38,461 [main] DEBUG com.foo.App - Hello world
2012-04-26 14:54:38,461 [main] DEBUG com.foo.App - Hi again
...
```

以 "#logback.classic pattern" 开头的行就是新插入的行。









