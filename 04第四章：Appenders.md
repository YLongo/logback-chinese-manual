

# 第四章：Appenders

## 什么是 Appender

logback 将写入日志事件的任务委托给一个名为 appender 的组件。Appender 必须实现 [`ch.qos.logback.core.Appender`](https://logback.qos.ch/xref/ch/qos/logback/core/Appender.html) 接口。该接口的方法如下：

```java
package ch.qos.logback.core;
  
import ch.qos.logback.core.spi.ContextAware;
import ch.qos.logback.core.spi.FilterAttachable;
import ch.qos.logback.core.spi.LifeCycle;
  

public interface Appender<E> extends LifeCycle, ContextAware, FilterAttachable {

	public String getName();
  	public void setName(String name);
  	void doAppend(E event);
}
```

`doAppender()` 方法接收一个泛型参数 *E* 作为唯一的参数。*E* 的实际参数类型取决于 logback 模块。在 logback-classic 模块里面，*E* 的类型是 [ILoggingEvent](https://logback.qos.ch/apidocs/ch/qos/logback/classic/spi/ILoggingEvent.html) 。在 logback-access 模块里面，*E* 的类型是 [AccessEvent](https://logback.qos.ch/apidocs/ch/qos/logback/access/spi/AccessEvent.html)。`doAppend()` 是 logback 框架里面最重要的模块。它的责任是将日志事件进行格式化，然后输出到对应的设备上。

Appender 都是实体类，这样可以确保它们通过名字被引用。`Appender` 接口继承了  `FilterAttachable` 接口。使得一个或多个过滤器可以附加到 appender 实例上。

Appender 最基本的责任是将日志事件进行输出。然而，它们可以委托 `Layout` 或者 `Encoder` 对象来对日志事件进行格式化。每一个 layout/encoder 有且只与一个 appender 相关联。例如，`SocketAppender` 仅仅序列化日志事件，然后再通过线路传输。

## AppenderBase

[`ch.qos.logback.core.AppenderBase`](https://logback.qos.ch/xref/ch/qos/logback/core/AppenderBase.html)  是一个抽象类，实现了 `Appender` 接口。它提供了基本方法供所有 appender 使用。例如：获取或设置名称的方法、激活状态、布局以及过滤器。它是 logback 中所有 appender 的父类。尽管是一个抽象类，但是 `AppenderBase` 还实现了 `Append` 接口中 `doAppend()` 方法。可能附上源码的摘要来讨论 `AppenderBase` 是最清楚的方式。

```java
public synchronized void doAppend(E eventObject) {

  // prevent re-entry.
  if (guard) {
    return;
  }

  try {
    guard = true;

    if (!this.started) {
      if (statusRepeatCount++ < ALLOWED_REPEATS) {
        addStatus(new WarnStatus(
            "Attempted to append to non started appender [" + name + "].",this));
      }
      return;
    }

    if (getFilterChainDecision(eventObject) == FilterReply.DENY) {
      return;
    }
    
    // ok, we now invoke the derived class's implementation of append
    this.append(eventObject);

  } finally {
    guard = false;
  }
}
```

`doAppend()` 的实现是 synchronized 的。不同的线程通过同一个 appender 打印日志是线程安全的。当一个线程 *T* 正在执行 `doAppend()` 方法，接下来其它的线程调用将会被阻塞直到线程 *T* 离开 `doAppend()` 方法，这样可以确保 *T* 对 appender 的访问具有独占性。

因为这种同步并不总是适合的，所以 logback 提供了 [`ch.qos.logback.core.UnsynchronizedAppenderBase`](https://logback.qos.ch/xref/ch/qos/logback/core/UnsynchronizedAppenderBase.html) 类，0跟 [AppenderBase](https://logback.qos.ch/xref/ch/qos/logback/core/AppenderBase.html) 类十分的相似。为了简单起见，接下来的内容我们只讨论 `UnsynchronizedAppenderBase`。

首先，`doAppend()` 方法会去检查 `guard` 是不是为 true。如果是，它会立即退出。如果 `guard` 不是为 true，下一步将它设置为 true。`guard` 会确保 `doAppend()` 方法不会被自身递归调用。想象一下这样的一个组件，被在 `append()` 方法之外的地方被调用，用于打印日志。它的调用可能被直接指向一个完全相同的刚刚调用过它的一个  appender，导致无限循环和堆栈溢出。

> 注：``UnsynchronizedAppenderBase` 类中有一个 `guard` 变量
>
> `private ThreadLocal<Boolean> guard = new ThreadLocal<Boolean>();`

接下来，我们会检查 `started` 是否为 true。如果不是，`doAppend()` 会发出一个警告并返回。换句话说，一旦 appender 被关闭，就不能对它进行写入。`Appender` 对象实现了 `LifeCycle` 接口，也就是说它实现了 `start()`、`stop()`、`isStarted` 方法。在设置完一个 appender 的所有属性之后，logback 的配置框架 - Joran，将会给 appender 发一个信号去激活它的属性。根据它的类型，一个 appender 可能会启动失败，如果某些特定的属性丢失或者由于各种属性之间的冲突。例如，创建文件依赖截断模式，`FileAppender` 不能对 `File` 选项的值起作用，直到这个值确定下来。明确的激活步骤可以确保一个 appender 在知道它们的值之后再作用于其属性。

如果 appender 不能被被启动或者被停止，logback 内部状态管理系统将会发出一条警告信息。在几次尝试之后，为了避免被同一条警告信息淹没内部状态系统，`doAppend()` 会停止发出这些警告。

接下来 `if` 语句检查过滤器的结果，根据过滤器链的结果，事件可以被拒绝或者被接受。如果缺少过滤器链，事件默认会被接受。

接下来 `doAppend()` 调用 `append()` 的实现方法。这个方法是实际的执行者，用来将事件附加到合适的设备上。

最后，`guard` 被释放，后续对 `append()` 的调用得以执行。

在手册的其余部分，我们保留 `option` 与 `property`，用于 JavaBean 通过 get 和 set 方法动态推断出来的任何属性。

# Logback-core

Logback-core 为 logback 其他模块的构建奠定了基础。一般来说，logback-core 的组件需要一些定制，尽管很少。但是，在接下来的几个部分，我们描述了一种可以开箱即用的 appender。

## OutputStreamAppender

[`OutputStreamAppender`](https://logback.qos.ch/xref/ch/qos/logback/core/OutputStreamAppender.html) 将事件附加到 `java.io.OutputStream` 上。这个类提供了其它 appender 构建的基础服务。用户通常不会直接实例一个 OutputStreamAppender 实例。因为一般来说 `java.io.OutputStream` 类型不能方便的转为 String。因为在配置文件中没有方法去直接指定一个 `OutputStream` 目标对象。简单来说，你不能通过配置文件配置一个 `OutputStreamAppender`。但是这并不意味着 `OutputStreamAppender` 缺少配置属性。这些属性描述如下：

|     属性名     |                            属性值                            |                             描述                             |
| :------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|     ecoder     | [`Encoder`](https://logback.qos.ch/xref/ch/qos/logback/core/encoder/Encoder.html) | 决定通过哪种方式将事件写入 `OutputStreamAppender`，Encoder 将会在单独的章节介绍 |
| immediateFlush |                           boolean                            | `immediateFlush` 的默认值为 true。立即刷新输出流可以确保日志事件被立即写入，并且可以保证一旦你的应用没有正确关闭 appender，日志事件也不会丢失。从另一方面来说，设置这个属性为 false，有可能会使日志的吞吐量翻两番(视情况而定)。但是，设置为 false，当应用退出的时候没有正确关闭 appender，会导致日志事件没有被写入磁盘，可能会丢失。 |

`OutputStreamAppender` 是其他三个 appender 的父类，分别是 `ConsoleAppender`、`FileAppender` 以及 `RollingFileAppender`。`FileAppender` 又是 `RollingFileAppender` 的父类。下面的类图展示 `OutputStreamAppender` 与子类之间的关系：

![appenderClassDiagram](images/appenderClassDiagram.jpg)

## ConsoleAppender

[`ConsoleAppender`](https://logback.qos.ch/xref/ch/qos/logback/core/ConsoleAppender.html) 就跟名字显示的一样，是将日志事件附加到控制台，跟进一步说就是通过 *System.out* 或者 *System.err* 来进行输出。默认通过前者。`ConsoleAppender` 通过用户指定的 encoder，格式化日志事件。Encoder 会在接下来的章节讨论。*System.out* 与 *System.err* 两者都是 `java.io.PrintStream` 类型。因此，它们被包装在可以进行 I/O 缓存操作的 `OutputStreamWriter` 中。

|  属性名   |                             类型                             |                             描述                             |
| :-------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|  encoder  | [`Encoder`](https://logback.qos.ch/xref/ch/qos/logback/core/encoder/Encoder.html) |               见  `OutputStreamAppender` 属性                |
|  target   |                            String                            |      *System.out* 或 *System.err*。默认为 *System.out*       |
| withJansi |                           boolean                            | `withJansi` 的默认值为 `false`。设置 `withJansi` 为 `true` 可以激活 [Jansi](http://jansi.fusesource.org/) 在 windows 使用 ANSI 彩色代码。在 windows 上如果设置为 true，你应该将 ` org.fusesource.jansi:jansi:1.9` 这个 jar 包放到 classpath 下。基于 Unix 实现的操作系统，像 Linux、Max OS X 都默认支持 ANSI 才彩色代码。 |

*Example: ConsoleAppender configuraion (logback-Console.xml)*

```xml
<configuration>
	
	<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender" >
		<!-- encoder 默认使用 ch.qos.logback.classic.encoder.PatternLayoutEncoder -->
		<encoder>
			<pattern>%-4relative [%thread] %-5level %logger{35} - %msg %n</pattern>
		</encoder>	
	</appender>
	
	<root level="DEBUG">
		<appender-ref ref="STDOUT" />
	</root>
</configuration>
```

通过以下命令执行上面的配置文件：

```shell
java chapters.appenders.ConfigurationTester logback-Console.xml
```

## FileAppender

[`FileAppender`](https://logback.qos.ch/xref/ch/qos/logback/core/FileAppender.html) 是 `OutputStreamAppender` 的子类，将日志事件输出到文件中。通过 `file` 来指定目标文件。如果该文件存在，根据 `append` 的值，要么将日志追加到文件中，要么该文件被截断。

| 属性名  |                             类型                             |                             描述                             |
| :-----: | :----------------------------------------------------------: | :----------------------------------------------------------: |
| append  |                           boolean                            | 如果为 `true`，日志事件会被追加到文件中，否则的话，文件会被截断。默认为 `true` |
| encoder | [`Encoder`](https://logback.qos.ch/xref/ch/qos/logback/core/encoder/Encoder.html) |             参见  `OutputStreamAppender` 的属性              |
|  file   |                            String                            | 要写入文件的名称。如果文件不存在，则新建。在 windows 平台上，用户经常忘记对反斜杠进行转义。例如，*c:\temp\test.log* 不会被正确解析，因为 *'\t'* 是一个转义字符，会被解析为一个 *tab* 字符 (\u0009)。正确的值应该像：*c:/temp/test.log* 或者 *c:\\\temp\\\test.log*。没有默认值。 |
| prudent |                           boolean                            | 在严格模式下，`FileAppender` 会将日志安全的写入指定文件。即使在不同的 JVM 或者不同的主机上运行 `FileAppender` 实例。默认的值为 `false`。<br />严格模式可以与 `RollingFileAppender` 结合使用。<br />严格模式也意味着 `append` 属性被自动设置为 `true`。<br />严格模式依赖排他文件锁。实验证明，文件锁大概是写入日志事件成本的 3 倍。在严格模式关闭的情况下，往一台"普通"电脑的硬盘上将一个日志事件写入文件，大概需要耗费 10 微秒。但是在开启的情况下，大概需要 30 微秒。也就是说在关闭的情况下可以一秒钟写入 100'000 个日志事件，但是在开启的情况下，一秒钟只能写入33'000 个日志事件。<br />严格模式可以在所有 JVM 写入同一个文件时，有效的序列化 I/O 操作。因此，随着竞相访问同一个文件的 JVM 数量上升，将会延迟每一个 I/O 操作。只要总共的 I/O 操作大约为每秒 20 个日志请求，对性能的影响可以被忽略。但是，如果应用每秒产生了 100 个以上的 I/O 操作，性能会受到明显的影响，应该避免使用严格模式。<br />`网络文件锁` 当日志文件位于网络文件系统上时，严谨模式的成本会更高。同样重要的是，网络文件系统的文件锁带有很强的偏向性，当前获得锁的进程在释放锁之后会立马又重新获得。因此，当一个进程独占日志文件，将会导致其它进程饥饿死锁。<br />严格模式的影响严重依赖网速以及操作系统实现的细节。我们提供了一个小型应用  [FileLockSimulator](https://gist.github.com/2794241) 用于在你的环境中模拟严格模式。 |

`立即刷新` 默认情况下，每一个日志事件都会被立即刷新到底层的输出流。默认方法更加的安全，因为日志事件在你的应用没有正确关闭 appender 的情况下不会丢失。但是，要想显著的增加日志的吞吐率，你可以将 `immediateFlush` 设置为 `false`。

下面是 `FileAppender` 的配置示例：

> Example: logback-fileAppender.xml 

```xml
<configuration>
	<appender name="FILE" class="ch.qos.logback.core.FileAppender">
		<file>testFile.log</file>
<!-- 		将 immediateFlush 设置为 false 可以获得更高的日志吞吐量 -->
		<immediateFlush>true</immediateFlush>
<!-- 		默认为 ch.qos.logback.classic.encoder.PatternLayoutEncoder -->
		<encoder>
			<pattern>%-4relative [%thread] %-5level %logger{35} - %msg%n</pattern>
		</encoder>
	</appender>
	
	<root level="DEBUG">
		<appender-ref ref="FILE" />
	</root>
</configuration>
```

在 *logback-examples* 的文件夹下，运行以下命令：

```bash
java chapters.appenders.ConfigurationTester logback-fileAppender.xml
```

> 要指定配置文件的具体路径
>
> 也可以直接在 eclipse 里面 Run Application 时设置 Arguments

### 文件唯一命名 (使用时间戳)

在应用的开发阶段或者短期应用中，例如：批处理程序，在每个应用启动的时候创建一个新的日志文件。通过 `<timestamp>` 元素可以轻易做到这点。

> Example: logback-timestamp.xml 

```xml
<configuration>
<!-- 	通过 "bySecond" 将时间格式化成 "yyyyMMdd'T'HHmmss" 的形式插入到 logger 的上下文中 
		这个值对后续的配置也适用
-->
	<timestamp key="bySecond" datePattern="yyyyMMdd'T'HHmmss" />
	
	<appender name="FILE" class="ch.qos.logback.core.FileAppender">
<!-- 		利用之前创建的 timestamp 来创建唯一的文件 -->
		<file>log-${bySecond}.txt</file>
		<encoder>
			<pattern>%logger{35} - %msg%n</pattern>
		</encoder>
	</appender>
	
	<root level="DEBUG">
		<appender-ref ref="FILE" />
	</root>
</configuration>
```

`timestamp` 元素需要两个强制的属性 *key* 跟 *datePattern* 以及可选的属性 *timeReference*。*key* 属性的值是来区分哪个 timestamp 元素，并且在后续的配置中可以通过[TODO 变量替换](变量替换)来使用。*datePattern* 属性用于将当前时间格式化成字符串。日期格式必须遵循 [SimpleDateFormat](https://docs.oracle.com/javase/8/docs/api/java/text/SimpleDateFormat.html) 中的规范。*timeReference* 表示时间戳引用哪个时间。默认为解析配置文件的时间，也就是当前时间。但是，在一些特定的情况下，可以设置为上下文初始化的时间。通过 设置 *timeReference* 的值为 `contextBirth`。

通过一下命令来测试 `<timestamp>` 元素：

```bash
java chapters.appenders.ConfigurationTester logback-timestamp.xml
```

> 译者注：需要指定具体配置文件的具体路径，也可以通过 eclipse 来运行。后续不再重复此注意事项。

设置 *timeReference* 的值为 "contextBirth" 的例子如下：

> Example: logback-timestamp-contextBirth.xml

```xml
<configuration>
	<timestamp key="bySecond" datePattern="yyyyMMdd'T'HHmmss" timeReference="contextBirth"/>
	...
</configuration>
```

## RollingFileAppender

[`RollingFileAppender`](https://logback.qos.ch/xref/ch/qos/logback/core/rolling/RollingFileAppender.html) 继承自`FileAppender`，具有轮转日志文件的功能。例如，`RollingFileAppender` 将日志输出到 *log.txt* 文件，在满足了特定的条件之后，将日志输出到另外一个文件。

与 `RollingFileAppender` 进行交互的有两个重要的子组件。第一个是 `RollingPolicy`，它负责日志轮转的功能。另一个是 `TriggeringPolicy`，它负责日志轮转的时机。所以 `RollingPolicy` 负责发生什么，`TriggeringPolicy` 负责什么时候发生。

为了让 `RollingFileAppender` 生效，必须同时设置 `RollingPolicy` 与 `TriggeringPolicy`。但是，如果 `RollingPolicy` 也实现了 `TriggeringPolicy` 接口，那么只需要设置前一个就好了。

`RollingFileAppender` 的属性如下所示：

|      属性名      |                             类型                             |                             描述                             |
| :--------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|       file       |                            String                            |                     参见 `FileAppender`                      |
|      append      |                           boolean                            |                     参见 `FileAppender`                      |
|     encoder      | [`Encoder`](https://logback.qos.ch/xref/ch/qos/logback/core/encoder/Encoder.html) |                 参见  `OutputStreamAppender`                 |
|  rollingPolicy   |                        RollingPolicy                         | 当轮转发生时，指定 `RollingFileAppender` 的行为。下面将会详细说明 |
| triggeringPolicy |                       TriggeringPolicy                       | 告诉 `RollingFileAppender` 什么时候发生轮转行为。下面将会详细说明 |
|     prudent      |                           boolean                            | [`FixedWindowRollingPolicy`](#FixedWindowRollingPolicy) 不支持该属性。<br />`RollingFileAppender` 在使用严格模式时要与  [`TimeBasedRollingPolicy](#TimeBasedRollingPolicy) 结合使用，但是有两个限制：<br />1. 在严格模式下，也不支持也不允许文件压缩（我们不能让一个 JVM 在写入文件时，另一个 JVM 在压缩该文件）<br />2. 不能对 `FileAppender` 的 `file` 属性进行设置。实际上，大多数的操作系统不允许在有进程操作文件的情况下对文件改名。<br />其它的参考 `FileAppender` |

### Rolling policy 简介

[`RollingPolicy`](https://logback.qos.ch/xref/ch/qos/logback/core/rolling/RollingPolicy.html) 负责轮转的方式为：移动文件以及对文件改名。

`RollingPolicy` 接口如下：

```java
package ch.qos.logback.core.rolling;  

import ch.qos.logback.core.FileAppender;
import ch.qos.logback.core.spi.LifeCycle;

public interface RollingPolicy extends LifeCycle {

    public void rollover() throws RolloverFailure;
    public String getActiveFileName();
    public CompressionMode getCompressionMode();
    public void setParent(FileAppender appender);
}
```

`rollover` 方法负责对日志文件进行归档。`getActiveFileName()` 方法负责获取当前日志文件的名字。`getCompressionMode` 方法决定采取哪种压缩模式。通过 `setParent` 方法引用父类。

#### TimeBasedRollingPolicy

[`TimeBasedRollingPolicy`](https://logback.qos.ch/xref/ch/qos/logback/core/rolling/TimeBasedRollingPolicy.html) 是最常用的轮转策略。它是基于时间来定义轮转策略。例如按天或者按月。`TimeBasedRollingPolicy` 既负责轮转的行为，也负责触发轮转。实际上，`TimeBasedRollingPolicy` 同时实现了 `RollingPolicy` 与 `TriggeringPolicy` 接口。

`TimeBasedRollingPolicy` 的配置需要一个强制的属性 `fileNamePattern` 以及其它的可选属性。

|       属性名        |  类型   | 描述                                                         |
| :-----------------: | :-----: | :----------------------------------------------------------- |
|   fileNamePattern   | String  | 该属性定义了轮转时的属性名。它的值应该由文件名加上一个 *%d* 的占位符。*%d* 应该包含  java.text.SimpleDateFormat 中规定的日期格式。如果省略掉这个日期格式，那么就默认为 *yyyy-MM-dd*。轮转周期是通过  **fileNamePattern** 推断出来的。<br /> <br />注意：可以选择对 `RollingFileAppender`（`TimeBasedRollingPolicy` 的父类）中的 `file` 属性进行设置，也可以忽略。通过设置 `FileAppender` 的 `file` 属性，你可以将当前活动日志的路径与归档日志的路径分隔开来。当前日志永远会是通过 `file` 指定的文件。它的名字不会随着时间的推移而发生变化。但是，如果你选择忽略 `file` 属性，当前活动日志在每个周期内将会根据 `fileNamePattern` 的值变化。稍后的例子将会说明这一点。<br />*%d{}* 中的日期格式将会遵循  java.text.SimpleDateFormat 中的约定。斜杆 '/' 或者反斜杠 '\\' 都会被解析成目录分隔符。<br /><br />**指定多个 %d**<br /><br />可以指定多个 %d，但是只能有一个是主要的，用于推断轮转周期。其它的 %d 占位符必须通过 'aux' 标记为辅助的。见下面的示例：<br />多个 %d 占位符允许你在文件夹中去管理归档文件，这个跟轮转周期不同。如下所示：通过年月来管理日志文件夹，但是轮转周期是在每天晚上零点。<br />/var/log/%d{yyyy/MM, aux}/myapplication.%d{yyyy-MM-dd}.log<br /><br />**TimeZone**<br /><br />在某些情况下，你可能想要根据时区而不是主机的时钟来轮转日志。你可以通过如下方式来指定一个时区，例如：<br />aFloder/test.%d{yyyy-MM-dd-HH, UTC}.log<br />如果指定的 timezone 不能被识别或者拼写错误，将会根据  [TimeZone.getTimeZone(String)](http://docs.oracle.com/javase/6/docs/api/java/util/TimeZone.html#getTimeZone(java.lang.String)) 方法指定为 GMT。 |
|     maxHistory      |   int   | 这个可选的属性用来控制最多保留多少数量的归档文件，将会异步删除旧的文件。比如，你指定按月轮转，指定 maxHistory = 6，那么 6 个月内的归档文件将会保留在文件夹内，大于 6 个月的将会被删除。注意：当旧的归档文件被移除时，当初用来保存这些日志归档文件的文件夹也会在适当的时候被移除。 |
|    totalSizeCap     |   int   | 这个可选属性用来控制所有归档文件总的大小。当达到这个大小后，旧的归档文件将会被异步的删除。使用这个属性时还需要设置 maxHistory 属性。而且，maxHistory 将会被作为第一条件，该属性作为第二条件。 |
| cleanHistoryOnStart | boolean | 如果设置为 true，那么在 appender 启动的时候，归档文件将会被删除。默认的值为 false。<br />归档文件的删除通常在轮转期间执行。但是，有些应用的存活时间可能等不到轮转触发。对于这种短期应用，可以通过设置该属性为 true，在 appender 启动的时候执行删除操作。 |

下面是关于 `fileNamePattern` 的介绍。

| fileNamePattern                            | 轮转周期                                                     | 示例                                                         |
| ------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| */wombat/foo.%d*                           | 每天轮转（晚上零点）。由于省略了指定 %d 的日期格式，所以默认为 *yyyy-MM-dd* | 没有设置 `file` 属性：在 2006.11.23 这一天的日志都会输出到 */wombat/foo.2006-11-23* 这个文件。晚上零点以后，日志将会输出到 *wombat/foo.2016-11-24* 这个文件。<br />设置 `file` 的值为 */wombat/foo.txt*：在 2016.11.23 这一天的日志将会输出到 */wombat/foo.txt* 这个文件。在晚上零点的时候，*foo.txt* 将会被改名为 */wombat/foo.2016-11-23*。然后将创建一个新的 *foo.txt*，11.24 号这一天的日志将会输出到这个新的文件中。 |
| */wombat/%d{yyyy/MM}/foo.txt*              | 每个月开始的时候轮转                                         | 没有设置 `file` 属性：在 2016.10 这一个月中的日志将会输出到 */wombat/2006/10/foo.txt*。在 10.31 晚上凌晨以后，11 月份的日志将会被输出到 */wombat/2006/11/foo.txt*。<br />设置 `file` 的值为 */wombat/foo.txt*：在 2016.10，这个月份的日志都会输出到 */wombat/foo.txt*。在 10.31 晚上零点的时候，*/wombat/foo.txt* 将会被重命名为 */wombat/2006/10/foo.txt*，并会创建一个新的文件 */wombat/foo.txt* ，11 月份的日志将会输出到这个文件。依此类推。 |
| */wombat/foo.%d{yyyy-ww}.log*              | 每周的第一天（取决于时区）                                   | 每次轮转发生在每周的第一天，其它的跟上一个例子类似           |
| */wombat/foo%d{yyyy-MM-dd_HH}.log*         | 每小时轮转                                                   | 跟之前的例子类似                                             |
| */wombat/foo%d{yyyy-MM-dd_HH-mm}.log*      | 每分钟轮转                                                   | 跟之前的例子类似                                             |
| */wombat/foo%d{yyyy-MM-dd_HH-mm, UTC}.log* | 每分钟轮转                                                   | 跟之前的例子类似，不过时间格式是 UTC                         |
| */foo/%d{yyyy-MM, aux}/%d.log*             | 每天轮转。归档文件在包含年月的文件夹下                       | 第一个 %d 被辅助标记。第二个 %d 为主要标记，但是日期格式省略了。因此，轮转周期为每天（由第二个 %d 控制），文件夹的名字依赖年与月。例如，在 2016.11 的时候，所有的归档文件都会在 */foo/2006-11/* 文件夹下，如：*/foo/2006-11/2006-11-14.log* |

任何斜杆或者反斜杠够会被当作文件夹分隔符。任何必要的文件夹都会在有需要的时候创建。你可以轻松的将日志文件放在单独的文件夹中。

`TimeBasedRollingPolicy` 支持文件自动压缩。如果 `fileNamePattern` 以 *.gz* 或者 *.zip* 结尾，将会启动这个特性。

| fileNamePattern     | 轮转周期                                             | 示例                                                         |
| ------------------- | ---------------------------------------------------- | ------------------------------------------------------------ |
| */wombat/foo.%d.gz* | 每天轮转（晚上零点），自动将归档文件压缩成 GZIP 格式 | `file` 属性没有设置：在 2009.11.23，日志将会被输出到 */wombat/foo.2009-11-23* 这个文件。但是，在晚上零点的时候，文件将会被压缩成 */wombat/foo.2009-11-23.gz*。在 11.24，这一天的日志将会被直接输出到 */wombat/folder/foo.2009-11-24* 这个文件。<br />`file` 属性的值设置为 */wombat/foo.txt*：在 2009.11.23，日志将会被输出到 */wombat/foo.txt* 这个文件。在晚上零点的时候，该文件会被压缩成 */wombat/foo.2009-11-23.gz*。并会创建一个新的 */wombat/foo.txt* 文件，11.24 这一天的日志将会被输出到该文件。依此类推。 |

 `fileNamePattern` 有两个目的。logback 通过该属性可以进行周期性的轮转并且得到每个归档文件的名字。注意，两种跟不同的 pattern 可能会有相同的轮转周期。*yyyy-MM* 与 *yyyy@MM* 同样都是按月轮转，但是归档文件最终的名字不一样。

通过设置 `file` 属性，你可以将活动日志文件的路径与归档文件的路径分隔开来。日志将会一直输出到通过 `file` 属性指定的文件中，并且不会随着时间而改变。但是，如果你选择忽略 `file` 属性，活动日志的名字将会根据 `fileNamePattern` 的值在每个周期内变化。不设置 `file` 属性的时候，如果在轮转期间存在外部文件句柄引用日志文件，将会避免[命名错误](https://logback.qos.ch/codes.html#renamingError)。

`maxHistory` 控制归档文件保留的最大数目，并删除旧的文件。例如，如果你指定按月轮转，并设定 `maxHistory` 的值为 6，那么 6 个月之内的归档文件都会被保留，大于 6 个月的文件将会被删除。注意，当旧的文件被移除时，为文件归档而创建的文件夹在适当的时候也会被移除。

由于各种技术原因，轮转并不是时间驱动的，而是依赖日志事件。例如，在 2002.03.08，假设 `fileNamePattern` 的值为 *yyyy-MM-dd*（按天轮转），在晚上零点之后，没有日志事件到来，假设在 23 分 47 秒之后，第一个到达的日志事件将会触发轮转。也就是说轮转实际发生在 03.09 00:23'47 AM 而不是 0:00 AM。因此，依赖日志事件的到达速度，所以轮转可能会有延迟。但是，不管延迟的情况是什么样，一定周期内生成的日志事件将会被输出到指定的文件中，从这个角度来看，轮转算法始终都会是正确的。

下面是 `RollingFileAppender` 与 `TimeBaseRollingPolicy` 结合使用的例子：

> *Example: logback-RollingTimeBased.xml*

```xml
<configuration>
	<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
		<file>logFile.log</file>
		<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
<!-- 			按天轮转 -->
			<fileNamePattern>logFile.%d{yyyy-MM-dd}.log</fileNamePattern>
<!-- 			保存 30 天的历史记录，最大大小为 30GB -->
			<maxHistory>30</maxHistory>
			<totalSizeCap>3GB</totalSizeCap>
		</rollingPolicy>
		
		<encoder>
			<pattern>%-4relative [%thread] %-5level %logger{35} - %msg%n</pattern>
		</encoder>
	</appender>
	
	<root level="DEBUG">
		<appender-ref ref="FILE" />
	</root>
</configuration>
```

下面是在 `prudent` 模式下（严格模式）`RollingFileAppender` 与 `TimeBasedRollingPolicy` 的结合使用的例子：

> Example: logback-PrudentTimeBasedRolling.xml 

```xml
<configuration>
	<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
		<!-- 支持多个 JVM 同时写一个文件 -->
		<prudent>true</prudent>
		<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
			<fileNamePattern>logFile.%d{yyyy-MM-dd}.log</fileNamePattern>
			<maxHistory>30</maxHistory>
			<totalSizeCap>3GB</totalSizeCap>
		</rollingPolicy>
		
		<encoder>
			<pattern>%-4relative [%thread] %-5level %logger{35} - %msg%n</pattern>
		</encoder>
	</appender>

	<root level="DEBUG">
		<appender-ref ref="FILE" />
	</root>
</configuration>
```

### 基于大小以及时间的轮转策略

有时你希望按时轮转，但同时又想限制每个日志文件的大小。特别是如果后期处理工具需要对日志进行大小限制。为了满足这个需求，logback 配备了 `SizeAndTimeBasedRollingPolicy`。

注意，`TimeBasedRollingPolicy` 可以限制归档文件总的大小。所以如果你想要这个限制，你可以通过设置 `totalSizeCap` 来达到这个目的。

下面的示例展示了基于时间及大小的配置：

> *Example: logback-sizeAndTime.xml*

```xml
<configuration debug="true">
	<appender name="ROLLING" class="ch.qos.logback.core.rolling.RollingFileAppender">
		<file>mylog.txt</file>
		<rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
<!-- 			按天轮转 -->
			<fileNamePattern>mylog-%d{yyyy-MM-dd}.%i.txt</fileNamePattern>
			<maxFileSize>100MB</maxFileSize>
			<maxHistory>60</maxHistory>
			<totalSizeCap>20GB</totalSizeCap>
		</rollingPolicy>
		
		<encoder>
			<pattern>%msg%n</pattern>
		</encoder>
	</appender>
	
	<root level="DEBUG">
		<appender-ref ref="ROLLING" />
	</root>
</configuration>
```

注意，除了 %d 之外还有 %i。这两个占位符都是强制要求的。在当前时间还没有到达周期轮转之前，日志文件达到了 `maxFileSize` 指定的大小，会进行归档，递增索引从 0 开始。

基于大小与时间的文件归档支持删除旧的归档文件。你需要指定 `maxHistory` 属性的值来保存几个周期的日志。当你的应用停止或者启动的时候，日志将会继续向正确的位置输出。即当前周期内索引最大的。

在 1.17 版本前，这个文档会提及一个叫 `SizeAndTimeBasedFNATP` 的组件。但是 `SizeAndTimeBasedFNATP` 组件只提供一个最简单的配置。我们不再提供关于 `SizeAndTimeBasedFNATP` 的文档。尽管这样，早期的配置文件使用 `SizeAndTimeBasedFNATP` 依然会运行的很好。 实际上，`SizeAndTimeBasedRollingPolicy` 是使用 `SizeAndTimeBasedFNATP` 实现的。

#### FixedWindowRollingPolicy

在轮转时，[`FixedWindowRollingPolicy`](https://logback.qos.ch/xref/ch/qos/logback/core/rolling/FixedWindowRollingPolicy.html) 根据固定窗口算法重命名文件，具体描述如下：

`filaNamePattern` 表示归档文件的名字。这个属性是必须的，而且必须包含一个表示整形的占位符 *i%*。

`FixedWindowRollingPolicy` 的可用属性如下：

| 属性名          | 类型   | 描述                                                         |
| --------------- | ------ | ------------------------------------------------------------ |
| minIndex        | int    | 表示窗口索引的下界                                           |
| maxIndex        | int    | 表示窗口索引的上界                                           |
| fileNamePattern | String | `FixedWindowRollingPolicy` 在重命名日志文件时将会根据这个属性来命名。它必须包含一个 *i%* 的占位符，该占位符指明了窗口索引的值应该插入的位置。<br /><br />例如，当该属性的值为 *MyLogFile%i.log*，最小与最大的值分别为 *1* 和 *3*。将会产生的归档文件为 *MyLogFile1.log*，*MyLogFile2.log*，*MyLogFile3.log*。<br /><br />文件压缩的方式也是通过该属性来指定。例如，设置该属性的值为 *MyLogFile%i.log.zip*，那么归档文件将会被压缩成 *zip* 格式。也可以选择压缩成 *gz* 格式。 |

由于窗口固定算法需要跟窗口大小一样的的重命名次数，因此强烈不推荐太大的窗口大小。当用户指定一个较大值时，当前的实现会将窗口大小自动减少为 20。

让我们通过一个例子来了解下固定窗口算法。假设 `minIndex` 的值为 *1*，`maxIndex` 的值为 *3*。`fileNamePattern` 的值为 *foo%i.log*，`file` 属性的值为 *foo.log*。



| 轮转数目 | 当前输出文件 | 归档日志文件                 | 描述                                                         |
| -------- | ------------ | ---------------------------- | ------------------------------------------------------------ |
| 0        | foo.log      | -                            | 还没有到轮转周期，logbak 将日志输出初始文件                  |
| 1        | foo.log      | foo1.log                     | 第一次轮转，*foo.log* 被重命名为 *foo1.log*。一个新的 *foo.log* 文件将会被创建并成为当前输出文件 |
| 2        | foo.log      | foo1.log，foo2.log           | 第二次轮转，*foo1.log* 被重命名为 *foo2.log*。*foo.log* 被重命名为 *foo1.log*。一个新的 *foo.log* 被创建并成为当前输出文件 |
| 3        | foo.log      | foo1.log，foo2.log，foo3.log | 第三次轮转，*foo2.log* 被重命名为 *foo3.log*。*foo1.log* 被命名为 *foo2.log*。*foo.log* 被重命名为 *foo1.log*。一个新的 *foo.log* 被创建并成为当前输出文件 |
| 4        | foo.log      | foo1.log，foo2.log，foo3.log | 在这次以及后续的轮转中，将会删除 *foo3.log* 文件，其它文件的重命名操作跟之前的步骤一样。在本次以及以后的轮转中，将会一直只有三个归档文件以及一个活跃的日志文件 |

下面的给出了 `RollingFileAppender` 配合 `FixedWindowRollingPolicy` 使用的例子。注意，`file` 属性是强制的，即使它包含了一些跟 `fileNamePattern` 属性相同的信息。

> Example：logback-RollingFixedWindow.xml 

```xml
<configuration>
	<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
		<file>test.log</file>
		<rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
			<fileNamePattern>tests.%i.log.zip</fileNamePattern>
			<minIndex>1</minIndex>
			<maxIndex>3</maxIndex>
		</rollingPolicy>
		
		<triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
			<maxFileSize>5MB</maxFileSize>
		</triggeringPolicy>
		
		<encoder>
			<pattern>%-4relative [%thread] %-5level %logger{35} - %msg%n</pattern>
		</encoder>
	</appender>
	
	<root level="DEBUG">
		<appender-ref ref="FILE" />
	</root>
</configuration>
```

## 触发策略简介

[`TriggeringPolicy`](https://logback.qos.ch/xref/ch/qos/logback/core/rolling/TriggeringPolicy.html) 的实现用于通知 `RollingFileAppender` 何时轮转。

`TriggeringPolicy` 接口仅仅只包含了一个方法。

```java
package ch.qos.logback.core.rolling;

import java.io.File;
import ch.qos.logback.core.spi.LifeCycle;

public interface TriggeringPolicy<E> extends LifeCycle {

  public boolean isTriggeringEvent(final File activeFile, final <E> event);
}
```

`isTriggeringEvent()` 方法接收当前活动的文件以及当前的日志事件作为参数。基于这些参数，通过具体的实现来决定轮转是不是应该发生。

`TimeBasedRollingPolicy` 是使用最广泛的触发策略。也可以用作轮转策略来使用。

#### SizeBasedTriggeringPolicy

[`SizeBasedTriggeringPolicy`](https://logback.qos.ch/xref/ch/qos/logback/core/rolling/SizeBasedTriggeringPolicy.html) 观察当前活动文件的大小，如果已经大于了指定的值，它会给 `RollingFileAppender` 发一个信号触发对当前活动文件的轮转。

`SizeBasedTriggeringPolicy` 只接收 `maxFileSize` 这一个参数，它的默认值是 10 MB。

`maxFileSize` 可以为字节，千字节，兆字节，千兆字节，通过在数值后面指定一个后缀 *KB*，*MB* 或者 *GB*。例如，*5000000*，*5000KB*，*5MB* 以及 *2GB* 都是有效的，前三个是一样的。

```xml
<configuration>
	<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
		<file>test.log</file>
		<rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
			<fileNamePattern>test.%i.log.zip</fileNamePattern>
			<minIndex>1</minIndex>
			<maxIndex>3</maxIndex>
		</rollingPolicy>

		<triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
			<maxFileSize>5MB</maxFileSize>
		</triggeringPolicy>
		<encoder>
			<pattern>%-4relative [%thread] %-5level %logger{35} - %msg%n
			</pattern>
		</encoder>
	</appender>

	<root level="DEBUG">
		<appender-ref ref="FILE" />
	</root>
</configuration>
```

## Logback Classic

虽然日志事件在 logback-core 是通用的，但是在 logback-classic 中，它们永远是 `ILoggingEvent` 的实例。logback-classic 只不过是用来处理 `ILoggingEvent` 实例的专门处理管道。

### SocketAppender and SSLSocketAppender

到目前为止，所介绍的 appender 只能将日志输出到本地资源。相反的是，[`SocketAppender`](https://logback.qos.ch/xref/ch/qos/logback/classic/net/SocketAppender.html) 被设计成可以将 `ILoggingEvent` 实例序列化再传输到远端机器。当使用 `SocketAppender` 时，日志事件将以明文发送，使用 [`SSLSocketAppender`](https://logback.qos.ch/xref/ch/qos/logback/classic/net/SSLSocketAppender.html) 时，日志事件将通过安全的通道传输。

序列化事件的实际类型为 [`LoggingEventVO`](https://logback.qos.ch/xref/ch/qos/logback/classic/spi/LoggingEventVO.html)，它实现了 `ILoggingEvent` 接口。就日志事件而言，远程日志是非侵入式的。在接收到日志事件并反序列化之后，日记事件就像在本地生成的一样。多个 `SocketAppender` 实例运行在不同的机器上，直接将它们的日志通过固定的格式输出到中央日志服务器上。`SocketAppender` 不会关联 layout，因为它是序列化日志事件到远程服务器上。`SocketAppender` 在 *Transmission Control Protocol (TCP)* 层上运行，该层提供了可靠，有序，流式控制以及段对端的八位字节流。所以，如果远程服务器是可以到达的，那么日志事件最终都会到达那里。相反，如果远程服务器挂掉或者不可达到，那么日志事件会被丢弃。如果服务器重新恢复，那么日志事件的传输将会继续进行。这种重连是通过一个连接线程池周期性的尝试连接来进行的。

日志事件由本地 TCP 实现自动缓冲。也就是说如果连接到服务器的速度很慢，但是比客户端产生日志事件的速度要快，那么客户端不会受到网速的影响。但是，如果网络连接速度比日志产生速度要慢，那么客户端只能以网络速度进行处理。特别在极端情况下，连接到服务器的网络挂掉了，客户端最终会被阻塞。如果网络连接恢复了，但是服务器挂掉了，客户端不会阻塞，尽管因为服务器挂掉了，日志事件丢失。

尽管 `SocketAppender` 不再依赖任何的 logger，在当前线程的连接下也不会被垃圾收集。但是连接线程只有在服务器挂掉的情况下才存在，为了避免出现垃圾收集问题，你需要明确的关闭 `SocketAppender`。生命周期长的应用会创建/销毁许多 `SocketAppender` 实例，应该注意到这个垃圾回收问题。大部分的应用可以忽略这个问题。如果 JVM 在 `SocketAppender` 关闭之前退出，无论是显式的退出或者是通过后续的垃圾回收，都可能会导致未传输的数据在管道中被丢失。这是基于 windows 系统常见的问题。为了避免数据丢失，通常的做法是调用 `close()` 方法去关闭 `SocketAppender`，或者在应用退出之前调用 `LoggerContext` 的 `stop()` 方法。

远程服务器通过 `remoteHost` 与 `port` 属性来标识。`SocketAppender` 属性罗列在下表当中。`SSLSocketAppender` 支持额外的一些属性，将在[第十五章](https://logback.qos.ch/manual/usingSSL.html)进行讨论。

| 属性名            | 类型                                                         | 描述                                                         |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| includeCallerData | boolean                                                      | 如果为 true，那么调用者的信息也会被发送到服务端。默认为 false。 |
| port              | int                                                          | 远程服务器的端口号                                           |
| reconnectionDelay | [Duration](https://logback.qos.ch/apidocs/ch/qos/logback/core/util/Duration.html) | 接受一个表示持续时间的字符串，例如 "10 seconds" 代表每次重连的间隔时间。默认值为 30 秒。将这个值设置为 0，将会关闭重连机制。注意，如果连接服务器成功，则不会出现连接线程池。 |
| queueSize         | int                                                          | 接受一个整数（大于0）代表有多少个日志事件传输到服务端。当这个值为 1 时，会同步传输日志事件到远端。当这个值大于 1 时，将设队列中还有剩余的空间，那么一个新的事件将会入队。队列的长度大于 1 可以提高性能，消除网络引起的延时。<br /><br />参见 `eventDelayLimist` 属性。 |
| eventDelayLimit   | [Duration](https://logback.qos.ch/apidocs/ch/qos/logback/core/util/Duration.html) | 接受一个表示持续时间的字符串，例如："10 seconds"。它表示一旦当前队列已满（例如已经包含了 `queueSieze` 个事件），在丢弃事件之前的等待时间。在远端服务器一直缓慢接收事件时，这种情况就会发生。默认值为 100 毫秒。 |
| remoteHost        | String                                                       | 服务器的主机名                                               |
| ssl               | SSLConfiguration                                             | 只支持 ` SSLSocketAppender`，这个属性提供了 SSL 配置供 appender 使用，将在[第十五章](https://logback.qos.ch/manual/usingSSL.html)进行讨论 |

#### 日志服务器的选择

logback classic 给服务器提供了两个选择接收来自 `SocketAppender` 与 `SSLSocketAppender` 的日志事件。

- `ServerSocketReceiver` 与跟它相对应的具有 SSL 功能的 `SSLServerSocketReceiver` 都是接收组件。可以通过配置应用中的 *logback.xml* 文件来接收远程 socket appender 的日志事件。查看[第十四章](https://logback.qos.ch/manual/receivers.html)获取更多的信息。
- `SimpleSocketServer` 与具有 SSL 功能的 `SimpleSSLSocketServer` 都提供了一个简单的 java 应用程序，该应用程序被设计成可配置，并且可以在命令行界面运行。这些应用仅仅等待来自 `SocketAppender` 或 `SSLSocketAppender` 的日志事件，每个被接收的日志事件按照本地服务器策略进行打印。下面给出一个简单的例子。

#### 使用 SimpleSocketServer

`SimpleSocketServer` 应用接收两个命令行参数：*port* 与 *configFile*。*port* 监听的端口，*configFile* 表示 XML 格式的配置文件。

在 *logback-examples/* 文件夹下，通过一下命令来启动 `SimpleSocketServer`：

```bash
java ch.qos.logback.classic.net.SimpleSocketServer 6000 \
  src/main/java/chapters/appenders/socket/server1.xml
```

6000 为监听的端口，在 *server1.xml* 中，`ConsoleAppender` 与 `RollingFileAppender` 被添加到 root logger 上。

在启动了 `SimpleSocketServer` 之后，你可以在多个客户端上使用 *SocketAppender* 来发送日志事件。这个手册中相关的示例包含了两个这样的客户端：*chapters.appenders.SocketClient1* 与 *chapters.appenders.SocketClient2*。两个客户端都会等待用户在控制台输入字符。输入的字符会被包裹在 debug 级别的日志事件中，然后发送到远程服务器。这个两个客户端不同的地方在于 `SocketAppender` 的配置。`SocketClient1` 通过编码来配置，`SocketClient2` 需要获取一个配置文件。

`SimpleSocketServer` 在本地机器上启动之后，通过以下命令去连接：

```bash
java chapters.appenders.socket.SocketClient1 localhost 6000
```

你输入的每一行字符都会出现在之前启动 `SimpleSocketServer` 的控制台上。如果你停止或者重启 `SimpleSocketServer`，客户端会重连新的服务实例，但是在断开连接时的日志时间会丢失（不能取消）。

与 `SocketClient1` 不同的是，`SocketClient2` 需要获取一个 XML 格式的配置文件来进行配置。配置文件 *client1.xml* 如下所示，它创建了一个 `SocketAppender` 附加到 root logger 上。

> Example: client1.xml 

```xml
<configuration>
          
  <appender name="SOCKET" class="ch.qos.logback.classic.net.SocketAppender">
    <remoteHost>${host}</remoteHost>
    <port>${port}</port>
    <reconnectionDelay>10000</reconnectionDelay>
    <includeCallerData>${includeCallerData}</includeCallerData>
  </appender>

  <root level="DEBUG">
    <appender-ref ref="SOCKET" />
  </root>  

</configuration>
```

注意配置文件中的 `remoteHost`，`port`，`includeCallerData` 属性的值并没有直接给出，而是通过占位符来代替。这些值可以通过系统属性来指定：

```bash
java -Dhost=localhost -Dport=6000 -DincludeCallerData=false \
  chapters.appenders.socket.SocketClient2 src/main/java/chapters/appenders/socket/client1.xml
```

这个命令应该会得到跟之前 `SocketClient1` 这个例子类似的结果。

让我再重复强调一遍，日志事件的序列化没有侵入性。反序列化出来的日志事件像其它的日志事件一样携带同样的信息。可以像操作本地日志事件一样操作它，除了序列化日志事件默认不会包含调用者的信息。下面通过一个例子来说明，首先通过一下命令启动 `SimpleSocketServer` ：

```bash
 java ch.qos.logback.classic.net.SimpleSocketServer 6000 \
  src/main/java/chapters/appenders/socket/server2.xml
```

配置文件 server2.xml 创建了一个可以输出调用者文件名以及行号的 `ConsoleAppender`。如果你像之前通过配置文件 *client1.xml* 来运行 `SocketClient2`，你将会在两个括号之间看到两个问号，而不是调用者的文件名以及行号：

```java
2006-11-06 17:37:30,968 DEBUG [Thread-0] [?:?] chapters.appenders.socket.SocketClient2 - Hi
```

可以通过设置 `includeCallerData` 的值为 true，来改变 `SocketAppender` 的输出信息。使用如下命令：

```bash
java -Dhost=localhost -Dport=6000 -DincludeCallerData=true \
  chapters.appenders.socket.SocketClient2 src/main/java/chapters/appenders/socket/client1.xml
```

因为反序列化出来的事件可以像本地日志时间一样被处理，所以它们甚至可以被传送到第二个服务器做进一步的处理。在练习的时候，你可以设置两台服务器，第一台服务器接受客户端的日志事件，然后转发到第二台服务器。

#### 使用 SimpleSSLSocketServer

`SimpleSSLSocketServer` 跟之前使用的 `SimpleSocketServer` 一样，在命令行接收两个参数：*port*，*configFile*。此外，你必须为日志服务器的 X.509 认证通过系统属性提供位置与密码信息。

在 *logback-examples/* 文件夹下，通过如下命令来启动 `SimpleSSLSocketServer`：

```
java -Djavax.net.ssl.keyStore=src/main/java/chapters/appenders/socket/ssl/keystore.jks \
    -Djavax.net.ssl.keyStorePassword=changeit \
    ch.qos.logback.classic.net.SimpleSSLSocketServer 6000 \
    src/main/java/chapters/appenders/socket/ssl/server.xml
```

`SimpleSSLSocketServer` 示例使用 X.509 认证，非常适合测试以及实验。**在生产环境使用 `SimpleSSLSocketServer` 之前，你应该为你的日志服务器获取一个 X.509 认证来标识你的服务器**。详情请参考[第十五章](https://logback.qos.ch/manual/usingSSL.html)。

因为服务器的配置文件在根元素上指定了 `debug="true"`，所以你将会在服务器启动的过程中看到 SSL 的配置信息。这在验证本地安全策略是否被正确实现时非常有效。

`SimpleSSLSocketServer` 启动的时候，你可以使用 `SSLSocketAppender` 来连接服务器。下面这个例子展示所需要的配置信息：

```xml
<configuration debug="true">
          
  <appender name="SOCKET" class="ch.qos.logback.classic.net.SSLSocketAppender">
    <remoteHost>${host}</remoteHost>
    <port>${port}</port>
    <reconnectionDelay>10000</reconnectionDelay>
    <ssl>
      <trustStore>
        <location>${truststore}</location>
        <password>${password}</password>
      </trustStore>
    </ssl>
  </appender>

  <root level="DEBUG">
    <appender-ref ref="SOCKET" />
  </root>  

</configuration>
```

注意，跟之前一样，`remoteHost`、`port` 的值通过占位符来指定。另外，注意一下 `ssl` 属性，它包含了一个内置属性 `trustStore`，通过这个属性来指定信用商店的位置以及密码。这个配置非常的必要，因为我们使用自签名的证书。见[第十五章](https://logback.qos.ch/manual/usingSSL.html)来查看更多使用 `SSLSocketAppender` 关于 SSL 的配置信息。

我们使用这个配置来运行一个客户端实例，在命令行通过系统属性指定占位符的值：

```bash
java -Dhost=localhost -Dport=6000 \
    -Dtruststore=file:src/main/java/chapters/appenders/socket/ssl/truststore.jks \
    -Dpassword=changeit \
    chapters.appenders.socket.SocketClient2 src/main/java/chapters/appenders/socket/ssl/client.xml
```

跟之前的示例一样，在客户端提示的时候输入一些信息，然后这些信息将会传送到服务端（使用安全的通道），并展示在控制台。

注意， *truststore* 的值是通过在命令行使用系统属性指定了一个标识信用商店的文件路径。你也可以使用 classpath 路径，具体参考[第十五章](https://logback.qos.ch/manual/usingSSL.html)。

我们可以看到跟服务器启动时类似的信息，因为我们在客户端的配置文件的根元素中添加了 `debug="true"`，客户端启动的时候，详细打印了 SSL 的配置信息来帮助我们验证本地策略的一致性。

### ServerSocketAppender and SSLServerSocketAppender

我们之前讨论过的 `SocketAppender` 组件（以及具有 SSL 能力的副本）被设计成允许应用程序通过网络连接远程日志服务器，以便传输日志事件。在某些情况下，通过应用程序初始化一个对日志服务器的连接可能不方便或者不可行。在这些情况下，logback 提供了 [`ServerSocketAppender`](https://logback.qos.ch/xref/ch/qos/logback/classic/net/server/ServerSocketAppender)。

`ServerSocketAppender` 不会初始化一个到日志服务器的连接，而是被动的监听 TCP 端口，等待客户端的连接。日志事件被传输给这个 appender，然后再分发给每个连接的客户端。如果没有客户端连接，日志事件会被马上丢弃。

除了基本的 `ServerSocketAppender` 之外，logback 还提供了 [`SSLServerSocketAppender`](https://logback.qos.ch/xref/ch/qos/logback/classic/net/server/SSLServerSocketAppender)，它通过一个安全，加密的通道传输日志事件到每个连接的客户端。而且，具有 SSL 功能的 appender 完全支持基于证书的双向认证，只有认证通过的客户端才可以连接到这个 appender 去接收日志事件。

对日志事件进行编码再传输的方法与 `SocketAppender` 完全一致，每个日志事件都是 `ILoggingEvent` 的实例。只不过连接的起始方向是相反的。虽然 `SocketAppender` 充当了一个主动的角色去连接日志服务器，`ServerSocketAppender` 是被动的监听即将到来的日志事件。

`ServerSocketAppender` 的子类型只提供给 logback 的接收组件。关于接收组件的信息，请查看[第十四章](https://logback.qos.ch/manual/receivers.html)。

`ServerSocketAppender` 支持如下的配置属性：

| 属性名            | 类型             | 描述                                                         |
| ----------------- | ---------------- | ------------------------------------------------------------ |
| address           | String           | appender 监听的本地网络接口地址。如果没有指定，则监听所有的网络接口 |
| includeCallerData | boolean          | 如果为 true，调用者的信息将会发送给远程服务器。为 false，则不发送。 |
| port              | int              | appender 监听的端口                                          |
| ssl               | SSLConfiguration | 仅仅支持 `SSLServerSocketAppender`。具体参见[第十五章](https://logback.qos.ch/manual/usingSSL.html) |

以下是关于 `ServerSocketAppender` 的配置：

```xml
<configuration debug="true">
    <appender name="SERVER" class="ch.qos.logback.classic.net.server.ServerSocketAppender">
        <port>${port}</port>
        <includeCallerData>${includeCallerData}</includeCallerData>
    </appender>
    
    <root level="debug">
        <appender-ref ref="SERVER" />
    </root>
</configuration>
```

注意这个配置跟之前 `SocketAppender` 的配置只有 *class* 这个属性不同。`remoteHost` 缺失表示 appender 被动的等待远程主机的连接，而不是新开一个到远程日志服务器的连接。

以下是关于 `SSLServerSocketAppender` 的配置：

```xml
<configuration debug="true">
  <appender name="SERVER" 
    class="ch.qos.logback.classic.net.server.SSLServerSocketAppender">
    <port>${port}</port>
    <includeCallerData>${includeCallerData}</includeCallerData>
    <ssl>
      <keyStore>
        <location>${keystore}</location>
        <password>${password}</password>
      </keyStore>
    </ssl>
  </appender>

  <root level="debug">
    <appender-ref ref="SERVER" />
  </root>  

</configuration>
```

这个配置跟上一个配置主要的不同在于 appender 的 *class* 属性为 `SSLServerSocketAppender` 类型，包含一个嵌套的 `ssl` 元素。在这个例子中，为 appender 配置了 X.509 认证的 keyStore。

> 具体的 SSL 配置参见第十五章，这句话翻的我想吐了

因为 `ServerSocketAppender` 的子类是专门为接收组件设计的，所以我们将对这个的阐述推迟到 [第十四章](https://logback.qos.ch/manual/receivers.html) 介绍。

### SMTPAppender

[`SMTPAppender`](https://logback.qos.ch/xref/ch/qos/logback/classic/net/SMTPAppender.html) 收集日志事件到一个或多个固定大小的缓冲区，当用户指定的事件发生时，将从缓冲区中取出适当的内容进行发送。SMTP 邮件是异步发送的。默认情况下，当日志的级别为 ERROR 时，邮件发送将会被触发。而且默认的情况下，所有事件都使用同一个缓冲区。

`SMTPAppender` 的属性如下表所示：

| 属性名              | 类型                                                         | 描述                                                         |
| ------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| smtpHost            | String                                                       | SMTP 服务器的主机名。强制性的。                              |
| smtpPort            | int                                                          | SMPT 服务监听的端口。默认为 25.                              |
| to                  | String                                                       | 接收者的邮件地址。触发事件发送给接收者。多个收件人可以使用逗号(,)分隔，或者使用多个 `<to>` 元素来指定。 |
| from                | String                                                       | `SMTPAppender` 使用的发件人，格式遵循[邮件通用格式](http://en.wikipedia.org/wiki/Email_address)，如果你想要包含发送者的名字，使用这种格式 " Adam Smith &lt;smith@moral.org&gt;"，那么邮件将会显示收件人为 " Adam Smith \<smith@moral.org\>" |
| subject             | String                                                       | 邮件的主题。它可以是通过 [PatternLayout](https://logback.qos.ch/manual/layouts.html#ClassicPatternLayout) 转换后的有效值。关于 Layout 将在接下来的章节讨论。<br />邮件应该有一个主题行，对应触发的邮件信息。<br />假设 `subject` 的值为："Log: %Logger - %msg"，触发事件的 logger 名为 "com.foo.Bar"，并且日志信息为 "Hello world"。那么发出的邮件信息将会有一个名为 "Log: com.foo.Bar - Hello World" 的主题行。<br />默认情况下，这个属性的值为 "%logger{20} - %m" |
| discriminator       | [Discriminator](https://logback.qos.ch/xref/ch/qos/logback/core/sift/Discriminator.html) | 在 Discriminator 的帮助下，`SMTPAppender` 根据 discriminator 返回的值可以将不同日志事件分散到不同的缓冲区中。默认的  discriminator 将返回同一个值，所以所有的事件都使用同一个缓冲区。 |
| evaluator           | [IEvaluator](https://logback.qos.ch/xref/ch/qos/logback/classic/boolex/IEvaluator.html) | 通过创建一个新的  <EventEvaluator/> 元素来声明此选项。通过 `class` 属性指定 class 的名字表示用户希望通过哪个类来满足 `SMTPAppender` 的 `Evaluator` 的需要。<br />如果没有指定此选项，当触发一个大于等于 *ERROR* 级别的事件时，`SMTPAppender` 将会被分配一个 [OnErrorEvaluator](https://logback.qos.ch/xref/ch/qos/logback/classic/boolex/OnErrorEvaluator.html) 的实例。<br />logback 配备了几个其它的  evaluator，分别叫  [`OnMarkerEvaluator`](https://logback.qos.ch/xref/ch/qos/logback/classic/boolex/OnMarkerEvaluator.html) （将在下面讨论），一个相对强大的 evaluator 叫  [`JaninoEventEvaluator`](https://logback.qos.ch/xref/ch/qos/logback/classic/boolex/JaninoEventEvaluator.html)（在[其它章节](https://logback.qos.ch/manual/filters.html#evalutatorFilter)讨论） 以及最近版本才有的一个更加强大的 evaluator 叫  [`GEventEvaluator`](https://logback.qos.ch/manual/filters.html#GEventEvaluator)。 |
| cyclicBufferTracker | [`CyclicBufferTracker`](https://logback.qos.ch/xref/ch/qos/logback/core/spi/CyclicBufferTracker.html) | 从名字可以看出，是一个  `CyclicBufferTracker` 的实例追踪循环缓冲区。它基于 discriminator 返回的 key （见上）。<br />如果你不想指定一个  cyclicBufferTracker，那么将会自动创建一个 [CyclicBufferTracker](https://logback.qos.ch/xref/ch/qos/logback/core/spi/CyclicBufferTracker.html) 的实例。默认的，这个实例用来保留事件的循环缓冲区的大小为 256。你需要改变 `bufferSize` 选项的大小（见下面） |
| username            | String                                                       | 默认为 null                                                  |
| password            | String                                                       | 默认为 null                                                  |
| STARTTLS            | boolean                                                      | 如果为 true，那么 appender 将会发送 STARTTLS 命令（如果服务器支持）将连接变成 SSL 连接。注意，连接初始的时候是为加密的。默认为 false。 |
| SSL                 | boolean                                                      | 如果为 true，将通过 SSL 连接服务器。默认为 false。           |
| charsetEncoding     | String                                                       | 邮件信息将会通过 [charset]((https://docs.oracle.com/javase/8/docs/api/java/nio/charset/Charset.html)]https://docs.oracle.com/javase/8/docs/api/java/nio/charset/Charset.html) 进行编码。默认编码为 "UTF-8" |
| localhost           | String                                                       | 一旦  SMTP 客户端的主机名没有配置正确，例如客户端的 hostname 不是全限定的，那么服务端会拒绝客户端发送的 HELO/EHLO 命令。为了解决这个问题，你可以将 `localhost` 的值设置为客户端主机的全限定名。详情见  [com.sun.mail.smtp](http://javamail.kenai.com/nonav/javadocs/com/sun/mail/smtp/package-summary.html) 包文档中的 "mail.smtp.localhost" 属性。(这个网站已经关闭了...) |
| asynchronousSending | boolean                                                      | 决定邮件传输是否是异步进行。默认为 'true'。但是，在某些特定的情况下，异步发送不怎么合适。例如，当发生一个严重错误时，你的应用使用 `SMTPAppender` 去发送一个警告，然后退出。但是相关线程可能没有时间去发送警告邮件。在这种情况下，你可以设置该属性的值为 'false'。 |
| includeCallerData   | boolean                                                      | 默认为 false。如果 `asynchronousSending` 的值为 true，并且你希望在日志中看到调用者的信息，你可以设置该属性的值为 true |
| sessionViaJNDI      | boolean                                                      | `SMTPAppender` 基于 `javax.mail.Session` 来发送邮件信息。默认情况下，该属性的值为 false，所以需要用户指定相关属性通过 `SMTPAppender` 来构建 `javax.mail.Session` 实例。如果设置为 true，`javax.mail.Session` 实例将会通过 JNDI 来获取。参见 `jndiLocation` 属性。<br />通过 JNDI 获取 `Session` 实例可以减少需要配置的数量，使你的应用减少重复([dryer](http://en.wikipedia.org/wiki/Don't_repeat_yourself))的工作。更多关于在 Tomcat 配置 JNDI 的信息请参考  [JNDI Resources How-to](http://tomcat.apache.org/tomcat-6.0-doc/jndi-resources-howto.html#JavaMail_Sessions)。<br />`注意`：通过 JNDI 获取 `Session` 的时候请移除 web 应用下  *WEB-INF/lib* 文件夹下的 *mail.jar* 与 *activation.jar*。 |
| jndiLocation        | String                                                       | JNDI 中放置  javax.mail.Session 的地方。默认为：" java:comp/env/mail/Session " |

SMTPAppender 仅仅只在它的循环缓存区中保留最后 256 个日志事件，当缓存区快要满的时候丢掉旧的日志事件。因此，通过 SMTPAppender 发送任何邮件包含的日志事件都不会超过 256 个。这在保留内存需求的限制，还提供了数量可观的应用上下文。

`SMTPAppender` 基于 JavaMail API。在 JavaMail 1.4 版本做过测试。JavaMail 需要 JavaBeans Activation Framework 包。你可以去它们各自的网站下载 [JavaMail API](http://java.sun.com/products/javamail/) 与 [JavaBeans Activation Framework](http://java.sun.com/beans/glasgow/jaf.html)。在运行下面的示例之前先确保将这两个 jar 包放在 classpath 下。

[`chapters.appenders.mail.EMail`](https://logback.qos.ch/xref/chapters/appenders/mail/EMail.html) 应用会生成多个日志信息，随后再生成一个错误日志信息。它接收两个参数，第一参数是整形，表示需要生成多少个日志事件。第二个参数表示 logback 的配置文件。*Email* 最后生成一个错误日志，将会触发发送邮件信息。

下面是一个 `Email` 应用的简单配置信息：

> Example: mail1 

```xml
<configuration>
    <appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">
        <smtpHost>SMTP 服务器的地址</smtpHost>
        <to>收件人1</to>
        <to>收件人2</to>
        <from>发件人</from>
        <subject>TESTING: %logger{20} - %m</subject>
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>%date %-5level %logger{35} - %message%n</pattern>
        </layout>
    </appender>
    
    <root level="DEBUG">
        <appender-ref ref="EMAIL" />
    </root>
</configuration>
```

在使用以上配置测试 `chapters.appenders.mail.Email` 之前，你需要设置 `smtpHost`，`to`，`from` 的值。一旦你正确设置了配置文件的值，你可以通过一下命令来执行：

```bash
java chapters.appenders.mail.EMail 100 src/main/java/chapters/appenders/mail/mail1.xml
```

收件者收到的邮件是经过 `PatternLayout` 格式化后的 100 条日志。下图展示的就是 Mozilla Thunderbird 邮件客户端接收到的邮件。

![smtpAppender1](images/smtpAppender1.jpg)

下个例子配置文件 *mail2.xml* 中的 `smtpHost`，`to`，`from` 属性的值通过占位符来代替。下面是 `mail2.xml` 配置中的一部分：

```xml
<appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">
  <smtpHost>${smtpHost}</smtpHost>
  <to>${to}</to>
  <from>${from}</from>
  <layout class="ch.qos.logback.classic.html.HTMLLayout"/>
</appender>
```

你可以通过命令行来传递参数：

```bash
java -Dfrom=source@xyz.com -Dto=recipient@xyz.com -DsmtpHost=some_smtp_host \
  chapters.appenders.mail.EMail 10000 src/main/java/chapters/appenders/mail/mail2.xml
```

根据你的环境替换合适的值。

注意在新的例子中，`PatternLayout` 被 `HTMLLayout` 替代，将日志格式化为 HTML 表格。你可以更改行与列的顺序，以及表格的 CSS 样式。查看 [HTMLLayout](https://logback.qos.ch/manual/layouts.html#ClassicHTMLLayout) 文档更详细的信息。

由于给定的循环缓冲区的大小为 256，收件人可以看到经过 256 条经过格式化的日志显示在 HTML 表格中。注意，`chapters.appenders.mail.Email` 应用生成了 10'000 条日志，但是只有最新的 256 条日志会显示在邮件中。

![smtpAppender2](images/smtpAppender2.jpg)

像 Mozilla Thunderbird, Eudora or MS Outlook 这些邮件客户端，提供了非常好的 CSS 样式来支持 HTML 邮件。但是，有时候会自动将 HTML 格式变成文本格式。如果想在 Thunderbird 查看 HTML 格式的邮件，需要通过 "View→Message Body As→Original HTML" 选项来进行设置。Yahoo 邮箱对 HTML 邮件有非常好的 CSS 样式支持。另一方面对 Gmail 来说，虽然它支持基本 HTML 表结构，但是它会忽略内部的 CSS 样式。Gmail 支持内联的 CSS 样式，但是由于内联的 CSS 会使输出结果变得庞大，所以 `HTMLLayout` 不会使用内联的 CSS 样式。

### 定制缓冲区大小

默认情况下，`SMTPAppender` 会输出最新的 256 条日志信息。下面一个例子设置了不同缓冲区大小。

> Example: customBufferSize.xml

```xml
<configuration>
	<appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">
		<smtpHost>${smtpHost}</smtpHost>
		<to>${to}</to>
		<from>${from}</from>
		<subject>%logger{20} - %m</subject>
		<layout class="ch.qos.logback.classic.html.HTMLLayout" />

		<cyclicBufferTracker
			class="ch.qos.logback.core.spi.CyclicBufferTracker">
			<!-- 每封邮件只包含一条日志 -->
			<bufferSize>1</bufferSize>
		</cyclicBufferTracker>
	</appender>

	<root level="DEBUG">
		<appender-ref ref="EMAIL" />
	</root>
</configuration>    
```

### 触发事件

如果 **evaluator** 属性没有设置，当日志事件的级别为 ERROR 时，`SMTPAppender` 会创建一个 [OnErrorEvaluator](https://logback.qos.ch/xref/ch/qos/logback/classic/boolex/OnErrorEvaluator.html) 实例来触发发送邮件。发送错误时发送邮件是比较合理的，`EventEvaluator` 提供了不同的实现来重写默认的方法。

`SMTPAppender` 提交的每一个日志事件通过 evaluator 都会调用 `evaluate()` 方法来决定这个事件是否需要发送邮件，或者仅仅是替换缓冲区中的内容。当 evaluator 给了一个确定的答案，那么将会发送邮件。`SMTPAppender` 有且只有一个 evaluator 实例。这个实例能够管理它自己的内部状态。为了对这个进行说明，`CounterBasedEvaluator` 实现了一个 evaluator，当第 1024 个日志事件到来时才会触发邮件发送。

> Example: [CounterBasedEvaluator.java](https://logback.qos.ch/xref/chapters/appenders/mail/CounterBasedEvaluator.html)

```java
package chapters.appenders.mail;

import ch.qos.logback.core.boolex.EvaluationException;
import ch.qos.logback.core.boolex.EventEvaluator;
import ch.qos.logback.core.spi.ContextAwareBase;

public class CounterBasedEvaluator extends ContextAwareBase implements EventEvaluator {

  static int LIMIT = 1024;
  int counter = 0;
  String name;

  public boolean evaluate(Object event) throws NullPointerException,
      EvaluationException {
    counter++;

    if (counter == LIMIT) {
      counter = 0;

      return true;
    } else {
      return false;
    }
  }

  public String getName() {
    return name;
  }

  public void setName(String name) {
    this.name = name;
  }
}
```

这个类继承了 `ContextAwareBase` 以及实现了 `EventEvaluator` 接口。这可以让用户专注在 `EventEvaluator` 的核心功能上，让基类提供基础的功能。

设置 `SMTPAppender` 的 `evaluator` 选项来指定用户自定义的 evaluator。下面的例子将 `SMTPAppender` 附加在 root logger 上，并使用 `CounterBasedEvaluator` 作为它事件的 evaluator。

> Example: mail3.xml 

```xml
<configuration>
  <appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">
    <evaluator class="chapters.appenders.mail.CounterBasedEvaluator" />
    <smtpHost>${smtpHost}</smtpHost>
    <to>${to}</to>
    <from>${from}</from>
    <subject>%logger{20} - %m</subject>

    <layout class="ch.qos.logback.classic.html.HTMLLayout"/>
  </appender>

  <root level="DEBUG">
    <appender-ref ref="EMAIL" />
  </root>  
</configuration>
```

### 基于标记(Marker)触发

虽然通过默认通过错误级别的日志来触发邮件的发送是合理的，但是可能会导致太多的邮件充斥用户的邮箱。logback 自带了另外一个触发策略，叫做 [OnMarkerEvaluator](https://logback.qos.ch/xref/ch/qos/logback/classic/boolex/OnMarkerEvaluator.html)。它基于标记来触发。其实就是通过用户指定的标记来触发。下面这个例子说明这个一点：

[Marked_EMail](https://logback.qos.ch/xref/chapters/appenders/mail/Marked_EMail.html) 应用包含了几个日志语句，都是 ERROR 级别的。但是只有其中的一条被标记。下面的是相关的代码：

```java
Marker notifyAdmin = MarkerFactory.getMarker("NOTIFY_ADMIN");
logger.error(notifyAdmin,
  "This is a serious an error requiring the admin's attention",
   new Exception("Just testing"));
```

下面的配置文件，表示当存在日志事件被标记为 NOTIFY_ADMIN 或者 TRANSACTION_FAILURE 将会触发邮件发送。

> Example: mailWithMarker.xml

```xml
<configuration>
  <appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">
    <evaluator class="ch.qos.logback.classic.boolex.OnMarkerEvaluator">
      <marker>NOTIFY_ADMIN</marker>
      <!-- 你可以指定多个标记 -->
      <marker>TRANSACTION_FAILURE</marker>
    </evaluator>
    <smtpHost>${smtpHost}</smtpHost>
    <to>${to}</to>
    <from>${from}</from>
    <layout class="ch.qos.logback.classic.html.HTMLLayout"/>
  </appender>

  <root>
    <level value ="debug"/>
    <appender-ref ref="EMAIL" />
  </root>  
</configuration>
```

通过以下命令执行：

```bash
java -Dfrom=source@xyz.com -Dto=recipient@xyz.com -DsmtpHost=some_smtp_host \
  chapters.appenders.mail.Marked_EMail src/main/java/chapters/appenders/mail/mailWithMarker.xml
```

#### 基于标记触发的 JaninoEventEvaluator

除了使用核心的 `OnMarkerEvaluator`，我们还可以是使用更加通用的 [`JaninoEventEvaluator`](https://logback.qos.ch/manual/filters.html#JaninoEventEvaluator)，甚至更加强大的 [`GEventEvaluator`](https://logback.qos.ch/manual/filters.html#GEventEvaluator)。例如，下面的配置文件使用 `JaninoEventEvaluator` 而不是 `OnMarkerEvaluator`，但是它们的含义是一样的。

> Example: mailWithMarker_Janino.xml 

```xml
<configuration>
  <appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">
    <evaluator class="ch.qos.logback.classic.boolex.JaninoEventEvaluator">
      <expression>
        (marker != null) &&
        (marker.contains("NOTIFY_ADMIN") || marker.contains("TRANSACTION_FAILURE"))
      </expression>
    </evaluator>    
    ... 跟之前一样
  </appender>
</configuration>
```

#### 基于标记触发的 GEventEvaluator

通过使用 [GEventEvaluator](https://logback.qos.ch/manual/filters.html#GEventEvaluator) 来实现一个相同的 evaluator。

> Example: mailWithMarker_GEvent.xml 

```xml
<configuration>
  <appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">
    <evaluator class="ch.qos.logback.classic.boolex.GEventEvaluator">
      <expression>
        e.marker?.contains("NOTIFY_ADMIN") || e.marker?.contains("TRANSACTION_FAILURE")
      </expression>
    </evaluator>    
    ... 跟之前一样
  </appender>
</configuration>
```

因为日志事件可能没有 marker，所以 marke 的值可能为 null。可以使用 Groovy 的安全解引用操作符，也就是 . ? 操作符。

### STARTTLS/SSL 认证

`SMTPAppender` 支持通过用户名/密码以及 STARTTLS，SSL 协议进行认证。STARTTLS  跟 SSL 的不同之处在于，STARTTLS 初始化连接不是加密的，仅仅只有在客户端发出 STARTTLS 命令的时候将连接变为 SSL。在 SSL 模式下，连接在一开始就是被加密的。

### Gmail 的 SMTPAppender 配置 (SSL)

下面的例子是 Gmail SSL 协议的 `SMTPAppender` 配置：

> Example: gmailSSL.xml 

```xml
<configuration>
  <appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">
    <smtpHost>smtp.gmail.com</smtpHost>
    <smtpPort>465</smtpPort>
    <SSL>true</SSL>
    <username>YOUR_USERNAME@gmail.com</username>
    <password>YOUR_GMAIL_PASSWORD</password>

    <to>EMAIL-DESTINATION</to>
    <to>ANOTHER_EMAIL_DESTINATION</to> <!-- additional destinations are possible -->
    <from>YOUR_USERNAME@gmail.com</from>
    <subject>TESTING: %logger{20} - %m</subject>
    <layout class="ch.qos.logback.classic.PatternLayout">
      <pattern>%date %-5level %logger{35} - %message%n</pattern>
    </layout>       
  </appender>

  <root level="DEBUG">
    <appender-ref ref="EMAIL" />
  </root>  
</configuration>
```

> 译者注：Gmail 的配置我没有测试成功，一直发送不了邮件。最后还是配置的 163 邮箱

### Gmail 的 SMTPAppender 配置 (STARTTLS)

下面的例子是 Gmail STARTTLS 协议的 `SMTPAppender` 配置：

> Example: gmailSTARTTLS.xml 

```xml
<configuration>   
  <appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">
    <smtpHost>smtp.gmail.com</smtpHost>
    <smtpPort>587</smtpPort>
    <STARTTLS>true</STARTTLS>
    <username>YOUR_USERNAME@gmail.com</username>
    <password>YOUR_GMAIL_xPASSWORD</password>
    
    <to>EMAIL-DESTINATION</to>
    <to>ANOTHER_EMAIL_DESTINATION</to> <!-- additional destinations are possible -->
    <from>YOUR_USERNAME@gmail.com</from>
    <subject>TESTING: %logger{20} - %m</subject>
    <layout class="ch.qos.logback.classic.PatternLayout">
      <pattern>%date %-5level %logger - %message%n</pattern>
    </layout>       
  </appender>

  <root level="DEBUG">
    <appender-ref ref="EMAIL" />
  </root>  
</configuration>
```

### SMTPAppender 与 MDCDiscriminator

之前提到过，指定一个 discriminator 而不是使用默认的。根据指定的 discriminator，`SMTPAppender` 会生成关于特定用户，用户 session，客户端 IP 地址的邮件信息。

下面是 [MDCBasedDiscriminator](https://logback.qos.ch/xref/ch/qos/logback/classic/sift/MDCBasedDiscriminator.html) 与一个名叫 req.remoteHost 的 MDC key 结合使用的一个例子。假定该 key 已经包含了远程主机的 IP 地址。在 web 应用中，你可以使用 [MDCInsertingServletFilter](https://logback.qos.ch/manual/mdc.html#mis) 去填充 MDC 的值。

> Example: mailWithMDCBasedDiscriminator.xml 

```xml
<configuration>   
  <appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">
    <smtpHost>ADDRESS-OF-YOUR-SMTP-HOST</smtpHost>
    <to>EMAIL-DESTINATION</to>
    <from>SENDER-EMAIL</from>

    <discriminator class="ch.qos.logback.classic.sift.MDCBasedDiscriminator">
      <key>req.remoteHost</key>
      <defaultValue>default</defaultValue>
    </discriminator>

    <subject>${HOSTNAME} -- %X{req.remoteHost} %msg"</subject>
    <layout class="ch.qos.logback.classic.html.HTMLLayout">
      <pattern>%date%level%thread%X{req.remoteHost}%X{req.requestURL}%logger%msg</pattern>
    </layout>
  </appender>

  <root>
    <level level="DEBUG"/>
    <appender-ref ref="EMAIL" />
  </root>  
</configuration>
```

所以，`SMTPAppender` 发送的每一封邮件都有一个独特的远程主机，这非常利于定位问题。

#### 在繁忙的应用中进行缓冲区管理

在内部，discriminator 返回每个不同的值都会创建一个新的循环缓冲区。但是，会维护一个 `maxNumberOfBuffers` 变量 (默认为 64)。当缓冲区的数量超过 `maxNumberOfBuffers` 时，最近最少更新的缓冲区会被丢弃。第二个安全策略是，最近 30 分钟没有被更新的缓冲区会被丢弃。

在每分钟有大量事务的机器上，设置一个较小值的 `maxNumberOfBuffers`，将会导致邮件中的日志数量变得特别小。实际上，在存在大量事务的情况下，连续生成同一个 discriminator 值会导致多个缓冲区会与同一个事务相关联，因为缓冲区会被 kill 掉再重建。即使在繁忙的系统中，循环缓冲区的最大数量也会被 `maxNumberOfBuffers` 所限制。

为了避免这种悠悠球效应(摇摆不定)，当日志时间被标记为 "FINALIZE_SESSION " 时， `SMTPAppender` 会释放与给定 discriminator 值相关联的缓冲区。这将会导致在每个事务快要结束的时候，将会丢弃适当的缓冲区。你可以在避免 OOM 的前提下，增加 `maxNumberOfBuffers` 的值到 512 或 1024。

这里有三个完全不同但是又互补的机制一起管理循环缓冲区。它们可以确保即使在繁忙的系统中，也会让相关的缓冲区存活。

### DBAppender

[`DBAppender`](https://logback.qos.ch/xref/ch/qos/logback/classic/db/DBAppender.html) 以一种独立于 JAVA 语言的方式将日志事件插入到三张数据库表中。

这三张表分别为：*logging_event*, *logging_event_property* 与 *logging_event_exception*。在使用 DBAppender 之前，它们必须存在。logback 自带 SQL 脚本来创建表。这些脚本在 *logback-classic/src/main/java/ch/qos/logback/classic/db/script* 文件夹下。每一种最流行的数据库都有一个对应的脚本。如果没有你指定的数据库脚本，参考已经存在的例子，可以很简单的写一个。如果你把他们发给我们，我们很乐意在将来的版本中发布这些脚本。

如果你的 JDBC 驱动支持 JDBC 3.0 specification 规范中的 `getGeneratedKeys` 方法，并且你也创建了上述所需要色数据库表，那么不再需要额外的步骤。否则的话，你必须选择一个适合你数据库的 `SQLDialect`。目前 logback 支持的数据库方言有 H2, HSQL, MS SQL Server, MySQL, Oracle, PostgreSQL, SQLLite and Sybase。

下面的表格总结了数据库类型，以及它们是否支持 `getGeneratedKeys()` 方法：

| RDBMS                | 测试版本 | JDBC 驱动的测试版本         | 是否支持 `getGeneratedKeys()` | logback 是否提供对应的方言 |
| -------------------- | -------- | --------------------------- | ----------------------------- | -------------------------- |
| DB2                  | untested | untested                    | unknown                       | NO                         |
| H2                   | 1.2.132  | -                           | unknown                       | YES                        |
| HSQL                 | 1.8.0.7  | -                           | NO                            | YES                        |
| Microsoft SQL Server | 2005     | 2.0.1008.2 (sqljdbc.jar)    | YES                           | YES                        |
| MySQL                | 5.0.22   | 5.0.8 (mysql-connector.jar) | YES                           | YES                        |
| PostgreSQL           | 8.x      | 8.4-701.jdbc4               | NO                            | YES                        |
| Oracle               | 10g      | 10.2.0.1 (ojdbc14.jar)      | YES                           | YES                        |
| SQLLite              | 3.7.4    | -                           | unknown                       | YES                        |
| Sybase SQLAnywhere   | 10.0.1   | -                           | unknown                       | YES                        |

在 "标准的" PC 电脑上，经过测试，将单个的日志事件写入数据库需要花费大约 10 毫秒。如果使用线程池，大约只需要花费大约 1 毫秒。大部分的 JDBC 驱动都支持连接池。

> 不是很懂这个 "标准的" PC 电脑，到底是台什么电脑

根据连接数据库的工具以及数据库自身，配置 logback 去使用 `DBAppender` 可以通过几种不同的方式去实现。我们很快就会发现，配置 `DBAppender` 的主要问题在于如何设置 `ConnectionSource` 实例。

一旦为数据库配置了 `DBAppender`，日志事件就会被发送到指定的数据库中。根据之前说的，logback 使用三张表来存储日志数据。

*logging_event* 表包含了以下字段：

| Field                 | Type       | Description                                                  |
| --------------------- | ---------- | ------------------------------------------------------------ |
| **timestamp**         | `big int`  | 日志事件的创建时间                                           |
| **formatted_message** | `text`     | 经过 `org.slf4j.impl.MessageFormatter` 格式化后的消息        |
| **logger_name**       | `varchar`  | 发出日志的 logger 名                                         |
| **level_string**      | `varchar`  | 日志事件的级别                                               |
| **reference_flag**    | `smallint` | 用来表示是否是异常或者与 MDC 属性相关联。它的值通过 `ch.qos.logback.classic.db.DBHelper` 计算得到。日志时间包含 `MDC` 或者 `Context` 时，它的值为 *1*。包含异常时，它的值为 *2*。包含两者，则值为 *3*。 |
| **caller_filename**   | `varchar`  | 发出日志请求的文件名                                         |
| **caller_class**      | `varchar`  | 发出日志请求的类                                             |
| **caller_method**     | `varchar`  | 发出日志请求的方法                                           |
| **caller_line**       | `char`     | 发出日志请求所在的行                                         |
| **event_id**          | `int`      | 日志事件在数据库的 id                                        |

*logging_event_property* 表用于存储 `MDC` 或者 `Context` 中的 key 与 value。它包含如下字段：

| Field            | Type      | Description         |
| ---------------- | --------- | ------------------- |
| **event_id**     | `int`     | 日志事件的数据库 id |
| **mapped_key**   | `varchar` | `MDC` 属性的 key    |
| **mapped_value** | `text`    | `MDC` 属性的 value  |

*logging_event_exception* 表包含如下字段：

| Field          | Type       | Description         |
| -------------- | ---------- | ------------------- |
| **event_id**   | `int`      | 日志事件的数据库 id |
| **i**          | `smallint` | 堆栈所在的行        |
| **trace_line** | `varchar`  | 相对应的堆栈信息    |

下面给出一个更加直观的示例，截图上面的信息是 MySQL 数据中 `DBAppender` 提供的内容。

表 *logging_event*：

![dbAppenderLE](images/dbAppenderLE.gif)

表 *logging_event_exception*：

![dbAppenderLEException](images/dbAppenderLEException.gif)

表 *logging_event_property*：

![dbAppenderLEProperty](images/dbAppenderLEProperty.gif)

#### ConnectionSource

`ConnectionSource` 接口提供了一种可插拔式的方式为需要使用 `java.sql.Connection` 的 logback 类获取 JDBC 连接。`ConnectionSource` 目前有三种实现，分别为：`DataSourceConnectionSource，` `DriverManagerConnectionSource` 与 `JNDIConnectionSource`。

第一个例子我们使用 `DriverManagerConnectionSource` 与 MySQL database，如下所示：

> Example: append-toMySQL-with-driverManager.xml 

```xml
<configuration>

  <appender name="DB" class="ch.qos.logback.classic.db.DBAppender">
    <connectionSource class="ch.qos.logback.core.db.DriverManagerConnectionSource">
      <driverClass>com.mysql.jdbc.Driver</driverClass>
      <url>jdbc:mysql://host_name:3306/datebase_name</url>
      <user>username</user>
      <password>password</password>
    </connectionSource>
  </appender>
  
  <root level="DEBUG" >
    <appender-ref ref="DB" />
  </root>
</configuration>
```

必须正确配置 JDBC 驱动，这里使用 `com.mysql.jdbc.Driver`。`url` 必须以 *jdbc:mysql://* 开头。

[`DriverManagerConnectionSource`](https://logback.qos.ch/xref/ch/qos/logback/core/db/DriverManagerConnectionSource.html) 实现了 `ConnectionSource` 接口，通过基于 URL 的传统 JDBC 方式来获取连接。

这个类为每一个调用 `getConnection()` 的方法都新建一个 `Connection` 连接。推荐你使用本地支持的连接池的 JDBC 驱动，或者创建你自己实现的 `ConnectionSource`，基于你已经使用的任何连接池机制。如果你可以使用支持 `javax.sql.DataSource` 的 JNDI 实现，例如，在 J2EE 应用服务中，参见下面的 [`JNDIConnectionSource`](https://logback.qos.ch/manual/appenders.html#JNDIConnectionSource)。

> Example: append-with-datasource.xml 

```xml
<configuration  debug="true">

  <appender name="DB" class="ch.qos.logback.classic.db.DBAppender">
     <connectionSource class="ch.qos.logback.core.db.DataSourceConnectionSource">
       
       <dataSource class="${dataSourceClass}">
         <!-- Joran 不能替换不是属性的变量。因此我们不能像其它变量一样声明接下来的变量
         -->
         <param name="${url-key:-url}" value="${url_value}"/>
         <serverName>${serverName}</serverName>
         <databaseName>${databaseName}</databaseName>
       </dataSource>
       
       <user>${user}</user>
       <password>${password}</password>
     </connectionSource>
  </appender>

  <root level="INFO">
    <appender-ref ref="DB" />
  </root>  
</configuration>
```

在这个例子中，我们大量使用了变量替换。当需要把一些连接的细节集中在一个配置文件中，并且通过 logback 与其它框架共享时非常方便。

#### JNDIConnectionSource

[`JNDIConnectionSource`](https://logback.qos.ch/xref/ch/qos/logback/core/db/JNDIConnectionSource.html) 是 logback 自带的，`ConnectionSource` 的另一种实现。从名字可以看出来，它通过 JNDI   获取 `javax.sql.DataSource`，然后再获取 `java.sql.Connection` 实例。`JNDIConnectionSource` 主要设计用在 J2EE 应用服务器以及应用服务器客户端中，这里假设应用服务器支持远程获取 `javax.sql.DataSource`。因为可以利用连接池或者其它应用服务器所提供的好处。更加重要的是，你的应用不需要做重复的工作，因为不需要在 *logback.xml* 中定义一个 `DataSource`。

例如，下面的是关于 Tomcat 的一个配置片段，它是基于 PostgreSQL。当然，上面提到其它数据也可以。

```xml
<Context docBase="/path/to/app.war" path="/myapp">
  ...
  <Resource name="jdbc/logging"
               auth="Container"
               type="javax.sql.DataSource"
               username="..."
               password="..."
               driverClassName="org.postgresql.Driver"
               url="jdbc:postgresql://localhost/..."
               maxActive="8"
               maxIdle="4"/>
  ...
</Context>
```

一旦 `DataSource` 在你的 J2EE 服务中定义了，你可以轻松的在 logback 配置文件中引用。如下所示：

> Example: append-via-jndi.xml 

```xml
<configuration debug="true">
  <appender name="DB" class="ch.qos.logback.classic.db.DBAppender">
    <connectionSource class="ch.qos.logback.core.db.JNDIConnectionSource">
      <!-- please note the "java:comp/env/" prefix -->
      <jndiLocation>java:comp/env/jdbc/logging</jndiLocation>
    </connectionSource>
  </appender>
  <root level="INFO">
    <appender-ref ref="DB" />
  </root>  
</configuration>
```

这个类通过午餐构造函数获取一个 `javax.naming.InitialContext`。在 J2EE 环境通常可以行得通。但是在 J2EE 环境之外，你需要根据 JNDI 提供者的文档提供一个 *jndi.properties* 属性文件。

#### 连接池

日志事件可以很快的被创建。为了让日志事件都能被插入到数据库，推荐 `DBAppender` 使用连接池配置。

经过实验发现，使用连接池，可以让 `DBAppender` 有大幅的性能提升。下面的配置文件，将日志事件发送给 MySQL，没有使用连接池。

> Example: append-toMySQL-with-datasource.xml 

```xml
<configuration>

  <appender name="DB" class="ch.qos.logback.classic.db.DBAppender">
    <connectionSource class="ch.qos.logback.core.db.DataSourceConnectionSource">
      <dataSource class="com.mysql.jdbc.jdbc2.optional.MysqlDataSource">
        <serverName>${serverName}</serverName>
        <port>${port$</port>
        <databaseName>${dbName}</databaseName>
        <user>${user}</user>
        <password>${pass}</password>
      </dataSource>
    </connectionSource>
  </appender>
    
  <root level="DEBUG">
    <appender-ref ref="DB" />
  </root>
</configuration>
```

在这个配置文件中，发送 500 个日志事件到 MySQL 数据库，需要高达 5 秒的时间，相当每条请求需要 10 毫秒。在大型的应用中，这个数字是不能够被接受的。

`DBAppender` 连接池需要使用一个专业的外部库。下一个例子中使用 [c3p0](http://sourceforge.net/projects/c3p0)。为了使用 c2p0，你需要下载并将 *c3p0-VERSION.jar* 放在类路径下。

> Example: append-toMySQL-with-datasource-and-pooling.xml 

```xml
<configuration>

  <appender name="DB" class="ch.qos.logback.classic.db.DBAppender">
    <connectionSource
      class="ch.qos.logback.core.db.DataSourceConnectionSource">
      <dataSource
        class="com.mchange.v2.c3p0.ComboPooledDataSource">
        <driverClass>com.mysql.jdbc.Driver</driverClass>
        <jdbcUrl>jdbc:mysql://${serverName}:${port}/${dbName}</jdbcUrl>
        <user>${user}</user>
        <password>${password}</password>
      </dataSource>
    </connectionSource>
  </appender>

  <root level="DEBUG">
    <appender-ref ref="DB" />
  </root>
</configuration>
```

使用这个新的配置，发送 500 条日志事件到 MySQL 数据库大约需要 0.5 秒，大约 1 毫秒一条请求，性能提升了十倍。

### SyslogAppender

syslog 协议非常的简单：syslog 发送者将信息发送给 syslog 接收者。接收者通常叫做 *syslog 守护线程* 或者 *syslog 服务器*。logback 可以把消息发送给远程的 syslog 守护线程。通过 [`SyslogAppender`](https://logback.qos.ch/xref/ch/qos/logback/classic/net/SyslogAppender.html) 可以实现。

下面是 SyslogAppender 的属性：

| 属性名                | 类型      | 描述                                                         |
| --------------------- | --------- | ------------------------------------------------------------ |
| **syslogHost**        | `String`  | syslog 服务器的主机名                                        |
| **port**              | `String`  | 用来 syslog 服务器的端口号。默认端口为 514，这个情况下，不需要修改。 |
| **facility**          | `String`  | 用来确定消息的来源。<br />它的值必须为 *KERN, USER, MAIL, DAEMON, AUTH, SYSLOG, LPR, NEWS, UUCP, CRON, AUTHPRIV, FTP, NTP, AUDIT, ALERT, CLOCK, LOCAL0, LOCAL1, LOCAL2, LOCAL3, LOCAL4, LOCAL5, LOCAL6, LOCAL7* 其中之一。大小写不敏感 |
| **suffixPattern**     | `String`  | 该属性指定发送到 syslog 服务器的消息非标准部分的格式。默认情况下，它的值为 *[%thread] %logger %msg*。`PatternLayout` 可以使用的任何值都是正确的 suffixPattern 值。 |
| **stackTracePattern** | `String`  | 该属性允许定制出现在每个堆栈行前面的字符。默认的值为 "\t"，即制表符。`PatternLayout` 可以使用的任何值都是正确的  stackTracePattern 值。 |
| **throwableExcluded** | `boolean` | 设置该属性的值为 `true`，会导致堆栈信息被忽略。默认为 `false`，所以堆栈信息可以被发送给 syslog 服务器。 |

日志事件的 syslog 严重程度是根据日志事件的级别转换来的。*DEBUG* 被转换为 *7*，*INFO* 被转换为 *6*，*WARN* 被转换为 *4*，*ERROR* 被转换为 *3*。

因为 syslog 请求的格式非常严格，所以 `SyslogAppender` 没有任何 layout。但是使用 `suffixPattern` 可以让用户展示他想展示的信息。

下面为 `SyslogAppender` 的配置示例：

> Example: logback-syslog.xml 

```xml
<configuration>

  <appender name="SYSLOG" class="ch.qos.logback.classic.net.SyslogAppender">
    <syslogHost>remote_home</syslogHost>
    <facility>AUTH</facility>
    <suffixPattern>[%thread] %logger %msg</suffixPattern>
  </appender>

  <root level="DEBUG">
    <appender-ref ref="SYSLOG" />
  </root>
</configuration>
```

在对这个配置进行测试的时候，应该先验证远程 syslog 守护线程是否能够接收外部的资源。根据以往的经验，syslog 守护进程通常会拒绝来自网络的请求。

### SiftingAppender

如名字所示，`SiftingAppender` 根据给定的运行时属性分离或者过滤日志。例如，`SiftingAppender` 可以根据用户的 session 分离日志，因此不同的用户的日志会有不同的日志文件，一个用户一个日志文件。

| 属性名               | 类型                                                         | 描述                                                         |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **timeout**          | [Duration](https://logback.qos.ch/apidocs/ch/qos/logback/core/util/Duration.html) | 一个内置的 appender 如果在指定 `timeout` 时间内没有被访问，则被认为是过时的。一个过时的 appender 会被关闭，并且不会被 `SiftingAppende` 所引用。默认值为 30 分钟 |
| **maxAppenderCount** | `integer`                                                    | `SiftingAppender` 可以创建并且跟踪内置 appender 的最大数量。默认值为 Integer.MAX_VALUE |

`SiftingAppender` 通过动态创建来实现这个。`SiftingAppender` 通过配置文件中指定的模板 (通过闭合的 `<sift>` 元素，见下面的例子) 来创建内置的 appender。`SiftingAppender` 负责管理子 appender 的生命周期。例如，`SiftingAppender` 会自动关闭并移除任何过时的 appender。在指定的 `timeout` 时间内没有被访问过的内置 appender，被认为是过时的。

在处理一个日志事件时，`SiftingAppender` 会委托一个子 appender 去进行处理。选择的标准是通过 discriminator 在运行时计算。用户也可以通过 [Discriminator](https://logback.qos.ch/xref/ch/qos/logback/core/sift/Discriminator.html) 来指定一个选择标准。让我们通过一个示例来学习一下。

#### 示例

[SiftExample](https://logback.qos.ch/xref/chapters/appenders/sift/SiftExample.html) 应用通过打印日志来表明应用已经启动。通过 MDC 设置键 "userid" 对应的值为 "Alice"，并打印了一条日志信息。下面是主要的代码：

```java
logger.debug("Application started");
MDC.put("userid", "Alice");
logger.debug("Alice says hello"); 
```

`SiftingAppender` 在配置文件中使用模板的示例如下：

> Example: byUserid.xml 

```xml
<configuration>
	
	<property name="FILE_NAME" value="FILE" />

	<appender name="SIFT"
		class="ch.qos.logback.classic.sift.SiftingAppender">
		<!-- 在缺少 class 属性的情况下，默认的 discriminator 类型为 			                             ch.qos.logback.classic.sift.MDCBasedDiscriminator -->
		<discriminator>
			<key>userid</key>
			<defaultValue>unknown</defaultValue>
		</discriminator>
		<sift>
			<appender name="FILE-${userid}"
				class="ch.qos.logback.core.FileAppender">
				<file>${userid}_${FILE_NAME}.log</file>
				<append>false</append>
				<layout class="ch.qos.logback.classic.PatternLayout">
					<pattern>%d [%thread] %level %mdc %logger{35} - %msg%n</pattern>
				</layout>
			</appender>
		</sift>
	</appender>

	<root level="DEBUG">
		<appender-ref ref="SIFT" />
	</root>
</configuration>
```

在没有 class 属性的情况下，默认的 discriminator 类型为 [MDCBasedDiscriminator](https://logback.qos.ch/xref/ch/qos/logback/classic/sift/MDCBasedDiscriminator.html)。discriminator 的的值为 MDC 的 key 所对应的值。但是，如果 MDC 的值为 null，那么 `defaultValue` 的将为 discriminator 的值。

`SiftingAppender` 的独特之处在于它有能力去引用以及配置子 appender。在上面的例子中，`SiftingAppender` 会创建多个 `FileAppender` 实例。每个 `FileAppender` 实例通过 MDC 的 key 所对应的值来标识。每当 MDC 的 key "userid" 被分配一个新值时，一个新的 `FileAppender` 将会被构建。`SiftingAppender` 可以追踪它所创建的 appender。appender 在 30 分钟之内没有被使用将会被自动关闭并丢弃。

`导出变量` 有不同 appender 实例是不够的。每一个实例都必须输出到一个唯一的资源中。为了做到这种区分，在 appender 模板中，key 被传递给 discriminator。在上面的例子中是 "userid"，它将被导出并变成一个[变量](https://github.com/Volong/logback-chinese-manual/blob/master/03%E7%AC%AC%E4%B8%89%E7%AB%A0%EF%BC%9Alogback%20%E7%9A%84%E9%85%8D%E7%BD%AE.md#%E5%8F%98%E9%87%8F%E6%9B%BF%E6%8D%A2)。因此，该变量可以通过给定的子 appender 来区分具体的资源。

在上面的示例中，使用 "byUserid.xml" 来运行 `SiftExample`，将会创建两个不同的日志文件，"unknown.log" 与 "Alice.log"。

`本地变量` 在版本 1.0.12 中，配置文件中局部变量的属性也可以应用到内置的 appender 中。而且，你可以在 `<sift>` 元素中[定义变量](https://github.com/Volong/logback-chinese-manual/blob/master/03%E7%AC%AC%E4%B8%89%E7%AB%A0%EF%BC%9Alogback%20%E7%9A%84%E9%85%8D%E7%BD%AE.md#%E5%8F%98%E9%87%8F%E7%9A%84%E5%AE%9A%E4%B9%89)以及[动态定义属性](https://github.com/Volong/logback-chinese-manual/blob/master/03%E7%AC%AC%E4%B8%89%E7%AB%A0%EF%BC%9Alogback%20%E7%9A%84%E9%85%8D%E7%BD%AE.md#%E5%8A%A8%E6%80%81%E5%AE%9A%E4%B9%89%E5%B1%9E%E6%80%A7)。或者在 `<sift>` 元素之外定义变量，在里面使用也是支持的。

#### 获取正确的 `timeout` 

对于特定类型的应用，正确的获取 `timeout` 参数非常困难。如果 `timeout` 过小，一个新的内置 appender 在创建几秒钟之后就被移除了。这种现象被称为 "制造垃圾"。如果 `timeout` 的值过大，那么 appender 会快速接连的被创建，可能会耗尽资源。同理，设置 `maxAppenderCount` 的值太低会产生垃圾。

在大多数情况下，在代码中显示的指出不需要再创建内置的 appender。需要在代码中标记日志事件为 [FINALIZE_SESSION](https://logback.qos.ch/apidocs/ch/qos/logback/classic/ClassicConstants.html#FINALIZE_SESSION_MARKER)。无论什么时候 SiftingAppender 看到日志事件标记为 `FINALIZE_SESSION`，它将会终结相关的子 appender。在生命周期快结束时，内置的 appender 将会留存几秒钟来处理之后到来的日志事件，然后再关闭。

```java
import org.slf4j.Logger;
import static ch.qos.logback.classic.ClassicConstants.FINALIZE_SESSION_MARKER;

  void job(String jobId) {
   
    MDC.put("jobId", jobId);
    logger.info("Starting job.");

    ... do whather the job needs to do
    
    // 将导致内置 appender 结束生命周期。但是会留存几秒钟
    logger.info(FINALIZE_SESSION_MARKER, "About to end the job");

    try {
      .. perform clean up
    } catch(Exception e);  
      // 被留存的 appender 处理，但是不会再创建新的 appender
      logger.error("unexpected error while cleaning up", e);
    }
  }
```

### AsyncAppender

AsyncAppender 异步的打印 [ILoggingEvent](https://logback.qos.ch/apidocs/ch/qos/logback/classic/spi/ILoggingEvent.html)。它仅仅是作为一个事件调度器的存在，因此必须调用其它的 appender 来完成操作。

`默认满了 80% 会丢弃数据` AsyncAppender 使用 [BlockingQueue](http://docs.oracle.com/javase/1.5.0/docs/api/java/util/concurrent/BlockingQueue.html) 来缓存日志时间。AsyncAppender 会创建一个工作线程去队列的头部获取数据，并将日志事件调度给附加再 AsyncAppender 上的 appender。在默认的情况下，在队列被占用了 80% 的情况下，AsyncAppender 会丢弃掉级别为 TRACE，DEBUG，INFO 的日志事件。这个策略虽然会丢失掉日志，但是对性能有利。

`停止/重新部署应用` 当停止或者重新部署应用时，`AsyncAppender` 必须被终止，为了停止并召回工作线程，以及刷新队列中的日志事件。通过[终止上下文](https://github.com/Volong/logback-chinese-manual/blob/master/03%E7%AC%AC%E4%B8%89%E7%AB%A0%EF%BC%9Alogback%20%E7%9A%84%E9%85%8D%E7%BD%AE.md#%E5%81%9C%E6%AD%A2-logback-classic)可以达到这个目标，并会关闭所有的 appender，包含所有 `AsyncAppender` 实例。`AsyncAppender` 将等待工作线程指定的 `maxFlushTime` 时间来刷新队列。如果你发现在关闭 `LoggerContext` 期间，队列中的日志事件被丢弃了，那么你需要去增加 `maxFlushTime`。指定 `maxFlushTime` 的值为 0 将会强制 `AsyncAppender` 等待所有日志事件被刷新才会从 stop() 方法中返回。

`停止后再清理` 取决于 JVM 的停止模式，工作线程在处理队列中的事件时被打断会导致日志事件在队列中被阻塞。一般发生在 `LoggerContext` 没有被完全停止，或者 JVM 在典型控制流之外终止。为了避免工作队列在这些情况下被打断。将一个 shutdown hook 在 JVM 运行时插入，在 JVM 开始准备停止的时候可以[正确的关闭 LoggerContext](https://github.com/Volong/logback-chinese-manual/blob/master/03%E7%AC%AC%E4%B8%89%E7%AB%A0%EF%BC%9Alogback%20%E7%9A%84%E9%85%8D%E7%BD%AE.md#%E5%81%9C%E6%AD%A2-logback-classic)。当其它的 shutdown hook 尝试记录事件时，shutdown hook 可以作为首选的方法来完全关闭 logback。

>   最后一句话没有搞懂是什么意思

如下是 AsyncAppender 的一些属性：

| 属性名              | 类型      | 描述                                                         |
| ------------------- | --------- | ------------------------------------------------------------ |
| queueSize           | `int`     | 队列的最大容量，默认为 256                                   |
| discardingThreshold | `int`     | 默认，当队列还剩余 20% 的容量时，会丢弃级别为 TRACE, DEBUG 与 INFO 的日志，仅仅只保留 WARN 与 ERROR 级别的日志。想要保留所有的事件，可以设置为 0 |
| includeCallerData   | `boolean` | 获取调用者的数据相对来说比较昂贵。为了提高性能，默认情况下不会获取调用者的信息。默认情况下，只有像线程名或者 [MDC](https://logback.qos.ch/manual/mdc.html) 这种"便宜"的数据会被复制。设置为 true 时，appender 会包含调用者的信息 |
| maxFlushTime        | `int`     | 根据所引用 appender 队列的深度以及延迟， `AsyncAppender` 可能会耗费长时间去刷新队列。当 `LoggerContext` 被停止时， `AsyncAppender stop` 方法会等待工作线程指定的时间来完成。使用 maxFlushTime 来指定最大的刷新时间，单位为毫秒。在指定时间内没有被处理完的事件将会被丢弃。这个属性的值的含义与  [Thread.join(long)](http://docs.oracle.com/javase/7/docs/api/java/lang/Thread.html#join(long)) 相同 |
| neverBlock          | `boolean` | 默认为 false，在队列满的时候 appender 会阻塞而不是丢弃信息。设置为 true，appender 不会阻塞你的应用而会将消息丢弃 |

默认情况下，事件队列的最大 容量为 256。如果队列被填满，那么新的日志事件将被阻塞，直到工作线程有机会去调度日志事件。当队列的容量没有处在最大容量的时候，应用线程能够再次开始记录日志。所以，在 AsyncAppender 在缓冲区的容量满了或者快满的情况下，异步日志记录变成了伪异步。但这也不是什么坏的事。虽然在 appender 缓冲区的压力减少之前，会稍微花点时间去处理日志事件，但是这个设计可以让应用继续保持运行。

为应用的最大吞吐量优化 appender 的事件队列依赖以下几个因素。以下的任何一个因素都有可能导致伪同步的发生：

- 大量的应用线程
- 每个应用调用大量的日志事件
- 每个日志事件有大量的数据
- 高延迟的子 appender

为了保持事物继续下去，增加队列的大小通常有用，但是是以应用可用的堆为代价。

`丢弃行为` 根据之前的讨论，为了减少阻塞。默认情况下，当剩余容量少于 20% 的时候，`AsyncAppender` 会丢掉 TRACE, DEBUG 以及 INFO 级别的日志，保留 WARN 与 ERROR 级别的日志。该策略可以确保非阻塞的处理日志事件 (因此具有高性能)。通过设置 `discardingThreshold` 的值为 0 可以阻止丢弃日志事件。

> Example: logback-async.xml 

```xml
<configuration>
  <appender name="FILE" class="ch.qos.logback.core.FileAppender">
    <file>myapp.log</file>
    <encoder>
      <pattern>%logger{35} - %msg%n</pattern>
    </encoder>
  </appender>

  <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="FILE" />
  </appender>

  <root level="DEBUG">
    <appender-ref ref="ASYNC" />
  </root>
</configuration>
```

### 编写你自己的 Appender

通过继承 AppenderBase 可以编写你自己的 appender。它支持处理过滤器，状态信息，以及其它大多数 appender 共享的功能。子类仅仅只需要实现 `append(Object eventObject)` 方法。

接下来列出来的 `CountingConsoleAppender`，限制了输出到控制台的日志事件的数量。当日志事件的数量达到上限时，它会退出。它使用 `PatternLayoutEncoder` 来格式化日志事件，还可以接收一个 `limit` 的参数。因此，除了 `append(Object eventObject)` 方法之后，还需要一些其它的方法。正如下面展示的，这些参数都是通过 logback 多种配置机制来自动处理的。

> Example: *CountingConsoleAppender.java* 

```java
package chapters.appenders;

import java.io.IOException;

import ch.qos.logback.classic.encoder.PatternLayoutEncoder;
import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.core.AppenderBase;


public class CountingConsoleAppender extends AppenderBase<ILoggingEvent> {
  static int DEFAULT_LIMIT = 10;
  int counter = 0;
  int limit = DEFAULT_LIMIT;
  
  PatternLayoutEncoder encoder;
  
  public void setLimit(int limit) {
    this.limit = limit;
  }

  public int getLimit() {
    return limit;
  }
  
  @Override
  public void start() {
    if (this.encoder == null) {
      addError("No encoder set for the appender named ["+ name +"].");
      return;
    }
    
    try {
      encoder.init(System.out);
    } catch (IOException e) {
    }
    super.start();
  }

  public void append(ILoggingEvent event) {
    if (counter >= limit) {
      return;
    }
    // 通过我们自己的 layout 来格式化日志事件
    try {
      this.encoder.doEncode(event);
    } catch (IOException e) {
    }

    // 准备下一个事件
    counter++;
  }

  public PatternLayoutEncoder getEncoder() {
    return encoder;
  }

  public void setEncoder(PatternLayoutEncoder encoder) {
    this.encoder = encoder;
  }
}
```

`start()` 方法会检查是否有 `PatternLayoutEncoder` 存在，如果不存在，appender 将会启动失败并会发出错误信息。

这个定制的 appender 说明了两点：

- 所有的属性遵循 JavaBean 的 setter/getter 转换，由 logback 透明的处理。在 logback 配置期间，`start()` 方法会被自动调用，用来验证 appender 的各种属性是否设置与一致
-  `AppenderBase.doAppend()` 方法会调用它子类的所有 append() 方法。实际上输出操作发生在 `append()` 方法中。而且，在这个方法中，appender 通过调用它们的 layout 来格式日志事件。

[`CountingConsoleAppender`](https://logback.qos.ch/xref/chapters/appenders/CountingConsoleAppender.html) 可以像其它的 appender 一样配置。详情见 *countingConsole.xml*。

## Logback Access

大部分的 appender 都可以在 logback-classic 中找到，同样的在 logback-access 中也可以找到。它们的工作本质上与在 logback-classic 中表现的是一样的。在接下来的部分，我们将讨论它们的用法。

### SocketAppender 与 SSLSocketAppender

[`SocketAppender`](https://logback.qos.ch/xref/ch/qos/logback/access/net/SocketAppender.html) 被委托将序列化的 `AccessEvent` 对象记录到远程实体上去。远程日志事件对 access event 来说是非侵入式的。在接收到并序列化之后，日志事件就像在本地被生成一样。

[`SSLSocketAppender`](https://logback.qos.ch/xref/ch/qos/logback/access/net/SSLSocketAppender.html) 扩展了 `SocketAppender`，通过 SSL 传输日志到远程实体上。

access 的 `SocketAppender` 属性跟 classic 中的 `SocketAppender` 属性一样。

### ServerSocketAppender 与 SSLServerSocketAppender

跟 `SocketAppender` 一样，[`ServerSocketAppender`](https://logback.qos.ch/xref/ch/qos/logback/access/net/server/ServerSocketAppender.html) 被委托传输序列化后的 `AccessEvent` 对象到远程实体上。但是，使用 `ServerSocketAppender` 时，appender 充当一个服务器的角色，被动的监听 TCP 端口，等待客户端的连接。传送到 appender 的日志事件将被分发给所有连接的客户端。

[`SSLServerSocketAppender`](https://logback.qos.ch/xref/ch/qos/logback/access/net/server/SSLServerSocketAppender.html) 拓展了 `ServerSocketAppender`，通过 SSL 传输日志到远程实体上。

access 的 `ServerSocketAppender` 属性跟 classic 中的 `ServerSocketAppender` 属性一样。

### SMTPAppender

access 中的 [`SMTPAppender`](https://logback.qos.ch/xref/ch/qos/logback/access/net/SMTPAppender.html) 工作的机制跟 classic 中的一样。但是 `evaluator` 属性完全不同。默认情况下，`SMTPAppender` 使用一个 `URLEvaluator` 对象。这个 evaluator 包含了一个 url 列表，用来检查当前请求的 url。当其中一个页面给 `URLEvaluator` 进行请求时，`SMTPAppender` 会发送一封邮件。

下面是在 access 环境下的一个例子：

> Example: logback-smtp.xml 

```xml
<appender name="SMTP"
  class="ch.qos.logback.access.net.SMTPAppender">
  <layout class="ch.qos.logback.access.html.HTMLLayout">
    <pattern>%h%l%u%t%r%s%b</pattern>
  </layout>
    
  <Evaluator class="ch.qos.logback.access.net.URLEvaluator">
    <URL>url1.jsp</URL>
    <URL>directory/url2.html</URL>
  </Evaluator>
  <from>sender_email@host.com</from>
  <smtpHost>mail.domain.com</smtpHost>
  <to>recipient_email@host.com</to>
</appender>
```

在某些特定的流程中，用户选择的页面是一个重要的步骤，那么将会触发邮件的发送。例如，当一个这样的页面被访问时，之前被访问过的页面会包含在邮件中被发送，还会包含任何用户想要的信息。

### DBAppender

[`DBAppender`](https://logback.qos.ch/xref/ch/qos/logback/access/db/DBAppender.html) 用来将 access 事件插入到数据库。

`DBAppender` 用到两张表：*access_event* 以及 *access_event_header*。在使用 `DBAppender` 之前，它们必须存在。logback 内置了 SQL 脚本用来创建表格。它们在 *logback-access/src/main/java/ch/qos/logback/access/db/script* 文件夹中。大部分流行的数据库都有一个对应的脚本。如果你使用的数据库不存在一个这样的脚本，那么你可以根据已经存在例子，很轻易的就可以写一个。我们鼓励你将一个这样的脚本提交到这个项目中。

表 *access_event* 包含的字段如下：

| 字段           | 类型      | 描述                                              |
| -------------- | --------- | ------------------------------------------------- |
| **timestamp**  | `big int` | access 时间创建的时间                             |
| **requestURI** | `varchar` | 请求的 URI                                        |
| **requestURL** | `varchar` | 请求的 URL。由请求方法，请求 URI 以及请求协议组成 |
| **remoteHost** | `varchar` | 远程主机的名字                                    |
| **remoteUser** | `varchar` | 远程用户的名字                                    |
| **remoteAddr** | `varchar` | 远程 IP 地址                                      |
| **protocol**   | `varchar` | 请求协议。例如 *HTTP* 或 *HTTPS*                  |
| **method**     | `varchar` | 请求方法。通常为 *GET* 或 *POST*                  |
| **serverName** | `varchar` | 发出请求的服务器的名字                            |
| **event_id**   | `int`     | access 事件的数据库 id                            |

表 *access_event_header* 包含了每个请求头。字段如下：

| 字段             | 类型    | 描述                                                         |
| ---------------- | ------- | ------------------------------------------------------------ |
| **event_id**     | int     | 相对应 access 事件的数据库 id                                |
| **header_key**   | varchar | 请求头的名字，例如 *User-Agent*                              |
| **header_value** | varchar | 请求头的值，例如  *Mozilla/5.0 (Windows; U; Windows NT 5.1; fr; rv:1.8.1) Gecko/20061010 Firefox/2.0* |

classic 中的 `DBAppender` 属性在 access 的 `DBAppender` 中一样有效。后者提供了另一个选项，如下：

| 属性名            | 类型    | 描述                                             |
| ----------------- | ------- | ------------------------------------------------ |
| **insertHeaders** | boolean | 告诉 `DBAppender` 用所有请求的请求头来填充数据库 |

下面是一个使用 `DBAppender` 的例子：

> Example: *logback-DB.xml*

```xml
<configuration>

  <appender name="DB" class="ch.qos.logback.access.db.DBAppender">
    <connectionSource class="ch.qos.logback.core.db.DriverManagerConnectionSource">
      <driverClass>com.mysql.jdbc.Driver</driverClass>
      <url>jdbc:mysql://localhost:3306/logbackdb</url>
      <user>logback</user>
      <password>logback</password>
    </connectionSource>
    <insertHeaders>true</insertHeaders>
  </appender>

  <appender-ref ref="DB" />
</configuration>
```

### SiftingAppender

logback-access 中的 SiftingAppender 跟 logback-classic 中的 SiftingAppender 非常相似。主要的不同在于 logback-access 默认的 discriminator 名字叫 [AccessEventDiscriminator](https://logback.qos.ch/xref/ch/qos/logback/access/sift/AccessEventDiscriminator.html)，而不是基于 MDC。从名字可以看出，AccessEventDiscriminator 在 AccessEvent 中使用一个指定的字段来选择一个内置的 appender。如果它的值为 null，那么将使用 `defaultValue` 指定的值。

指定的 AccessEvent 可以是 COOKIE, REQUEST_ATTRIBUTE, SESSION_ATTRIBUTE, REMOTE_ADDRESS, LOCAL_PORT, REQUEST_URI 其中的一种。注意，前三个字段中必须指定 `AdditionalKey`。

下面是配置示例：

> Example: access-siftingFile.xml 

```xml
<configuration>
  <appender name="SIFTING" class="ch.qos.logback.access.sift.SiftingAppender">
    <Discriminator class="ch.qos.logback.access.sift.AccessEventDiscriminator">
      <Key>id</Key>
      <FieldName>SESSION_ATTRIBUTE</FieldName>
      <AdditionalKey>username</AdditionalKey>
      <defaultValue>NA</defaultValue>
    </Discriminator>
    <sift>
       <appender name="ch.qos.logback:logback-site:jar:1.3.0-alpha4" class="ch.qos.logback.core.FileAppender">
        <file>byUser/ch.qos.logback:logback-site:jar:1.3.0-alpha4.log</file>
        <layout class="ch.qos.logback.access.PatternLayout">
          <pattern>%h %l %u %t \"%r\" %s %b</pattern>
        </layout>
      </appender>
    </sift>
  </appender>
  <appender-ref ref="SIFTING" />
</configuration>
```

在上面的配置文件中，`SiftingAppender` 内置了一个 `FileAppender` 实例。名为 "id" 的键被作为一个变量用于内置的 `FileAppender` 实例。默认的 discriminator，名叫 AccessEventDiscriminator，会在每个 `AccessEvent` 中查找一个 "username" 的 session 属性。如果没有，那么将使用默认值 "NA"。因此，如果一个名叫 "username" 的 session 属性包含了用户每条日志的用户名，那么以用户名命名的日志文件将会在 *byUser/* 文件夹下，日志文件包含了该用户产生的所有 access 日志。